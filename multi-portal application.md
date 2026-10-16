Building a multi-portal SaaS where <b>Stores are strictly isolated, Organizations group those stores, and Customers are global across the entire platform</b> requires a <b>Hierarchical Multi-Tenant + Cross-Tenant Identity Architecture.</b>

If you start with microservices for this, you will down drown in distributed data synchronization. The gold-standard production blueprint for Go and React at this stage is a <b>Modular Monolith Backend paired</b> with a <b>Turborepo React Monorepo </b> and <b>PostgreSQL with Row-Level Security (RLS).</b>

## 1. The Core Architectural Challenge: Store Isolation vs Global Customers
Your hardest engineering problem is that a Store's data (products, employees, invoices) must never leak to another store, yet a single Customer can log in and see invoices from Store A (Org1) and Store B (Org2) in one unified dashboard.

To solve this cleanly in PostgreSQL without duplicating databases:

1. Logical Isolation with <span style="color:gray">org_id</span> and <span style="color:gray">store_id</span> :Every store-specific table (<span style="color:gray">products, store_employees, store_roles, invoices</span> ) lives in a single shared PostgresSQL database and carries bot <span style="color:gray">org_id</span> and <span style="color:gray">store_id</span> (UUIDs) with composite indexes.
2. PostgreSQL Row-Level Securite (RLS)
	Instead of relying solely on developers remembering to write WHERE store_id = $1 on every single SQL query, enable Postgres RLS on all store-scoped tables. When a request hits your Go backend for a specific store, your middleware wraps the database call in a transaction and executes SELECT set_config('app.current_store_ . if a developer writes a buggy query without a WHERE clause, Postgres automatically blocks cross-store data leaks at the database engine level).
3. How they link: When a store creates an invoice for john@example.com , it links to store_customers. When John logs into the Global Customer Portal using his verified email/phone, your backend queries invoices joined via global_customer_id . He sees a unified feed of invoices across all stores, can click Accept or Reject, and his expense tracker aggregates his spend categorized by store_id .
## 2. Backend Architecture: Golang Modular Monolith
Do not split this into 3 separate backend servers yet. Build on Go binary structured by domain modules, exposing 3 distinct router groups with separate authentication middlewares.

#### Recommended Go Stack
- Router / HTTP: Chi (go-chi/chi) or Echo - lightweight, idiomatic, and supports cleanly mounting sub-routes per portal.
- Database layer: sqlc + pgx/v5 (compiles raw SQL into type-safe Go code with zero ORM overhead). Avoid heavy ORMs like GORM for complex multi-tenant reporting and RLS setups.
- Authorization (RBAC): Casbin (using Domains where each store_id acts as a security domain for custom store roles) or a clean permission-bitmask table scoped per store.
- Background Jobs & Webhooks: River (Postgres-backed queue for Go) or Asynq (Redis-backed) - essential for sending invoice emails, computing subscription usage limits, and handling Stripe webhooks reliably.

#### Go Directory Blueprint
```text
/cmd
	/api/main.go  # Single entry point
/internal
	/middleware   # JWT verification, Store-Context injectore, RLS tx wrapper
	/platform     # Side 1: Super Admin (Plans, Subscriptions, Global Metrics)
	/organization # Side 2a: Org Owner (Billing, Creating Stores, Org-level analytics)
	/store        # Side 2b: Store Context (Products, Employees, Custom Roles, Invoices)
	/customer     # Side 3: Global Customer (Cross-store Invoice Feed, Accept/Reject, Expenses)
	/shared/db    # sqlc generated models, migrations (golang-migrate or Goose)
```
#### API Route Separation
By namespace-isolating your API routes, you guarantee a Customer token can never accidentally hit a Store Admin endpoint:

- /api/va/admin/* - Requires SuperAdminJWT
- /api/v1/orgs/{orgID}/* - Requires OrgUserJWT + checks org ownership
- /api/v1/stores/{storeID}/* - Requires OrgUserJWT + verifies user has an active employee/owner role in {storeID} + sets Postgres RLS context
- /api/v1/customer/* - Requires GlobalCustomerJWT

## 3. Frontend Architecture: React + Turborepo Monorepo
Because your 3 portals have completely different navigation layouts, user jurneys, and security levels, do not cram them into a single React app with giant if(user.role=== ...) statements. Build 3 separate Vite + React SPAs inside a single Turborepo monorepo.

How the Monorepo Works
```text
/apps
	/super-admin     # Deployed to: admin.yourdomain.com
	/org-portal      # Deployed to: app.yourdomain.com (Includes Store Switcher dropdown)
	/customer-portal # Deploy to: my.yourdomain.com
	/ui
	/api-client
```

#### Why this is the best frontend setup:
1. Zero Bundle Bloat & High Security: A customer logging into my.youradmin.com never downloads the JavaScript bundle containing your Super Admin billing controls or Store inventory management forms.
2. Code Reuse: When you design the Invoice Preview Component in /packages/ui , both the Store Portal (app.) and the Customer Portal (my. ) import the exact same component.
3. Recommended React Stack: Vite (build tool), TanStact Router or React Router (routing), TanStack Query (React Query) (caching API states and invalidating data when a customer accepts and invoice), and Tailwind CSS + shadcn/ui
## 4. Production Deployment: Buleprint (Easiest & Most Scalable)
To move this specific stack to production without hiring a DevOps team:

| Component       | Production Service         | Architectural Role                                                                                                                                   |
| --------------- | -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Go Backend API  | Railway, Render, or Fly.io | Connects to your GitHub repo, builds your Go Dockerfile , and deploys with zero-downtime health checks.                                              |
| PostgreSQL DB   | Neon or Supabase           | Managed Postgres with built-in connection pooling (PgBouncer) and automated daily backups.                                                           |
| 3 React Portals | Cloudflare Pages or Vercel | Connect the same monorepo 3 times, pointing each subdomain (admin. , app. , my.) to its respective /apps/* build folder.                             |
| File Storeage   | Cloudflare R2 or AWS S3    | Stores generated Invoice PDFs and product images (never store files or the Go server's local disk).                                                  |
| Billing Engine  | Stripe Billing             | Maps to your Super Admin plans. Store the stripe_subscription_on the organizations table and enforce store/ employee count limits via Go middleware. |
## 5. Step-by-Step Execution Order
Do not build all three portals simultaneously. Follow the dependency chain:
1. Phase 1: Database Schema & Multi-Tenant Foundation (Week 1)
	Write your SQL migrations for organizations, stores, users and global_customers . Build the Go JWT authentication middleware and the store_switching context validator.
2. Phase 2: Org & Store Portal - Core Loop (Weeks 2-4)
	Build app.yourdomain.com first. Implement Org signup, creating a Store, adding Store Employee/Roles, adding Products and generating an Invoice.
3. Phase 3: Global Customer Portal (Weeks-5-6)
	Build my.yourdomain.com . Create the customer login flow, link their email to existing store invoices, build the Accept/Reject state machine (which triggers a notification back to the Store), and add the SQL aggregation query (SUM(amount) GROUP BY store_id, month) for their expense tracker.
4. Phase 4: Super Admin & Subscription Enforcement (Week 7)
	Build admin.yourdomain.com . Add plan tiers (e.g., Basic: Max 2 Stores, Pro: Max 10 Stores), hook up your payment gateway webhooks, and build the global tenant oversight dashboard.
