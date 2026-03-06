# ULTRA-LEAN E-COMMERCE PLATFORM ARCHITECTURE SPECIFICATION

Version: 1.2 (Final Consolidated)
Intent: Multi-tenant, Self-hostable, Minimal Infrastructure Cost

================================================================================
1. EXECUTIVE SUMMARY
================================================================================
Goal: Create a multi-tenant e-commerce SaaS that can run on free/low-cost 
infrastructure while remaining self-hostable by customers.

Core Philosophy:
- Single Codebase, Multi-Tenant SaaS (No per-store deployments)
- Serverless / Edge-first architecture (cost ≈ $0 when traffic is $0)
- Prefer providers with generous free tiers and low bandwidth costs
- Avoid infrastructure that requires persistent servers
- Maintain abstraction layers so the system can be self-hosted via Docker

================================================================================
2. HIGH-LEVEL ARCHITECTURE
================================================================================

[User Browser] 
      │
      ▼
[Cloudflare CDN / Edge Cache] 
      │
      ▼
[Next.js Frontend (Cloudflare Pages)] 
      │
      ├── Middleware (detect store by hostname)
      │
      ├── Auth Layer ──► [Postgres (Neon)] (Users / Sessions)
      │
      ├── Edge API (Cloudflare Workers) 
      │        │
      │        └── [Postgres (Neon)] (Products, Orders, Customers)
      │
      ├── Image Upload ──► [Object Storage (R2)]
      │
      ├── Email ──► [Resend]
      │
      └── Payments ──► [Stripe Connect]

Key design idea:
- Next.js handles UI + routing
- Workers handle API logic
- Postgres stores all persistent data

================================================================================
3. TECHNOLOGY STACK (LOWEST-COST FOCUSED)
================================================================================

| Component       | Technology                | Reasoning                        | Free Tier Limit                  |
| --------------- | ------------------------- | -------------------------------- | -------------------------------- |
| Frontend        | Next.js (App Router)      | Standard modern React framework  | N/A                              |
| Hosting         | Cloudflare Pages          | Free tier, global CDN            | 100k requests/day                |
| Edge API        | Cloudflare Workers        | Serverless backend logic at edge | 100k requests/day                |
| Database        | Neon Serverless Postgres  | Auto-scaling Postgres            | 0.5 GB Storage                   |
| ORM             | Drizzle ORM               | Lightweight, serverless-friendly | N/A                              |
| Authentication  | Auth.js                   | Open-source, no vendor lock-in   | N/A                              |
| File Storage    | Cloudflare R2             | Object storage, zero egress fees | 10 GB Storage / month            |
| Image Delivery  | Cloudflare Image Resizing | Dynamic resizing at CDN edge     | 100k requests/month              |
| Email           | Resend                    | Simple transactional email API   | 3,000 emails / month             |
| Payments        | Stripe Connect            | Marketplace-style payouts        | No monthly fee (2.9% + 30¢)      |
| Search          | Postgres Full Text        | Avoid external search services   | Included in DB                   |
| Background Jobs | Inngest / Trigger.dev     | Serverless workflows             | 10k steps/month (Free Tier)      |

================================================================================
4. MULTI-TENANCY STRATEGY
================================================================================

A. Domain Routing (Middleware)
   One Next.js deployment serves all stores.

   Examples:
   - store1.platform.com
   - store2.platform.com
   - custom-domain.com

   Middleware detects hostname and rewrites internally.

   Example middleware.ts:
   -----------------------------------------------------------------------------
   import { NextResponse } from "next/server";
   import type { NextRequest } from "next/server";

   export function middleware(req: NextRequest) {
     const host = req.headers.get("host");
     const subdomain = host?.split(".")[0];

     if (subdomain && subdomain !== "www" && subdomain !== "platform") {
       const url = req.nextUrl.clone();
       url.pathname = `/stores/${subdomain}${url.pathname}`;
       return NextResponse.rewrite(url);
     }
   }
   -----------------------------------------------------------------------------

B. Database Isolation (Row-Level Security)
   Every table contains a store_id column.

   Example Tables:
   - products
   - orders
   - customers
   - inventory

   Example SQL:
   -----------------------------------------------------------------------------
   ALTER TABLE products ADD COLUMN store_id UUID NOT NULL;
   ALTER TABLE products ENABLE ROW LEVEL SECURITY;

   CREATE POLICY store_isolation ON products
     USING (store_id = current_setting('app.current_store_id')::uuid);
   -----------------------------------------------------------------------------

   This ensures stores can never access each other's data.

================================================================================
5. DATA MODEL PHILOSOPHY
================================================================================

Keep schema minimal. Core tables only:

| Table          | Purpose                      |
| -------------- | ---------------------------- |
| stores         | Tenant configuration         |
| users          | Admin/Customer accounts      |
| products       | Product catalog              |
| product_images | Image references (R2 keys)   |
| customers      | Customer profiles            |
| orders         | Order headers                |
| order_items    | Order line items             |
| carts          | Active carts (logged-in)     |

Each table contains:
- id (UUID)
- store_id (UUID)
- created_at (Timestamp)

This keeps multi-tenancy consistent.

================================================================================
6. MEDIA STORAGE ARCHITECTURE
================================================================================

Image Workflow:
1. Merchant uploads image
2. Image stored in R2
3. Served through CDN with dynamic resizing

Example URL:
- image.jpg?width=400

Edge automatically generates optimized versions.

Advantages:
- No need to pre-generate sizes
- Saves storage
- Faster storefront loading

================================================================================
7. CART STRATEGY
================================================================================

Avoid unnecessary database writes.

| User Type     | Storage Method        | Benefit                          |
| ------------- | --------------------- | -------------------------------- |
| Guest Users   | localStorage / Cookies| Zero DB load, faster interaction |
| Logged-in     | Database              | Persistence across devices       |

Sync strategy:
- Merge guest cart into DB cart on login.

================================================================================
8. SELF-HOSTABLE VS SAAS MODE
================================================================================

Architecture supports both modes.

A. SaaS Mode (Managed)
   - Hosting: Cloudflare Pages
   - API: Cloudflare Workers
   - Database: Neon Postgres
   - Storage: R2
   - Email: Resend

B. Self-Host Mode (Docker)
   - Hosting: Node container
   - Database: Postgres container
   - Storage: MinIO (S3 compatible)
   - Email: SMTP / Mailhog

Example abstraction pattern:
-----------------------------------------------------------------------------
export const storage =
  process.env.SELF_HOSTED === "true"
    ? new MinioProvider()
    : new R2Provider();
-----------------------------------------------------------------------------

This allows the same codebase to run anywhere.

================================================================================
9. COST BREAKDOWN (EARLY STAGE)
================================================================================

| Service        | Cost     | Limit Before Upgrade        |
| -------------- | -------- | --------------------------- |
| Hosting        | $0       | 100k requests/day           |
| Database       | $0       | 0.5 GB Storage              |
| Object Storage | $0       | First 10 GB                 |
| Email          | $0       | 3,000 emails/month          |
| Payments       | 2.9% + ¢ | Per transaction             |
| Domain         | ~$10/yr  | One main domain             |

Estimated monthly cost: $0–$10 for early MVP usage.

================================================================================
10. ENGINEERING CHALLENGES & SOLUTIONS
================================================================================

1. Database Connections
   Problem: Serverless functions can exhaust DB connections.
   Solution: Use Neon's connection pooler.
   Config: Connection string ?pgbouncer=true

2. Product Page Performance
   Problem: Every page hitting DB increases latency.
   Solution: Cache product pages at CDN layer.
   Config: Cache TTL = 60 seconds

3. Stripe Webhooks
   Problem: Payments require webhook verification.
   Solution: Handle webhooks via edge API endpoints.
   Security: Verify signature headers.

4. Rate Limiting
   Problem: Public APIs may be abused.
   Solution: Use CDN-level rate limiting rules.
   Tool: Cloudflare WAF / Rate Limit Rules

5. Search Scaling
   Problem: Full-text search slows for large catalogs.
   Solution: Add GIN indexes on tsvector columns.
   Limit: Move to dedicated search engine at >10k products.

6. Background Jobs
   Problem: Serverless functions timeout (10s-60s).
   Solution: Offload to Inngest/Trigger.dev.
   Use Case: Order confirmation emails, inventory updates.

================================================================================
11. IMPLEMENTATION ROADMAP
================================================================================

Phase 1: Core Platform
- Setup Next.js + Postgres
- Implement authentication
- Create store onboarding
- Build product CRUD

Phase 2: Storefront
- Implement domain routing middleware
- Build basic storefront theme
- Implement product pages

Phase 3: Commerce
- Cart system (Guest + DB)
- Checkout integration
- Stripe Connect onboarding
- Order creation

Phase 4: Media
- Image upload
- CDN image optimization

Phase 5: Self-Hosting
- Docker configuration
- Provider abstraction layer
- Documentation

================================================================================
12. SCALING PLAN (WHERE FREE TIER BREAKS)
================================================================================

First bottlenecks likely appear in:

| Resource        | Free Limit   | Upgrade Path                  | Cost After    |
| --------------- | ------------ | ----------------------------- | ------------- |
| Database Size   | 0.5 GB       | Neon Paid Tier                | $19/month     |
| Image Bandwidth | 10 GB        | Enforce compression / Paid R2 | $0.015/GB     |
| Email Volume    | 3,000/mo     | AWS SES                       | $0.10/1k      |
| Search          | 10k Products | Typesense / Algolia           | $29+/month    |

Critical rule:
- NEVER deploy per store.
- One deployment must serve all stores.

================================================================================
END OF SPECIFICATION
================================================================================