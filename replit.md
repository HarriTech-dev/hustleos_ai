# HustleOS AI

A production-ready SaaS platform for African entrepreneurs and SMEs. Manage invoices, inventory, customers, and get AI-powered business advice — built for African markets with Paystack & Flutterwave support.

## Run & Operate

- `pnpm --filter @workspace/api-server run dev` — run the API server (port 8080, proxied at /api)
- `pnpm --filter @workspace/hustle-os run dev` — run the frontend (port 26039, proxied at /)
- `pnpm run typecheck` — full typecheck across all packages
- `pnpm run build` — typecheck + build all packages
- `pnpm --filter @workspace/api-spec run codegen` — regenerate API hooks and Zod schemas from the OpenAPI spec
- `pnpm --filter @workspace/db run push` — push DB schema changes (dev only)
- Required env: `DATABASE_URL` — Postgres connection string

## Stack

- pnpm workspaces, Node.js 24, TypeScript 5.9
- Frontend: React + Vite + TailwindCSS + Shadcn UI (wouter routing)
- API: Express 5 (port 8080)
- DB: PostgreSQL + Drizzle ORM
- Auth: Supabase Auth (JWT verified server-side)
- AI: OpenAI GPT-4o-mini
- Payments: Paystack + Flutterwave
- Storage: Cloudinary
- Validation: Zod (`zod/v4`), `drizzle-zod`
- API codegen: Orval (from OpenAPI spec)
- Build: esbuild (CJS bundle)

## Where things live

- `artifacts/hustle-os/` — React+Vite frontend
- `artifacts/api-server/` — Express API server
- `lib/db/` — Drizzle ORM schema + migrations
- `lib/api-spec/openapi.yaml` — OpenAPI contract (source of truth)
- `lib/db/src/schema/` — DB table definitions
- `artifacts/api-server/src/routes/` — API route handlers
- `artifacts/hustle-os/src/pages/` — Page components
- `artifacts/hustle-os/src/contexts/AuthContext.tsx` — Supabase auth context
- `artifacts/hustle-os/.env` — VITE_ prefixed env vars for frontend

## Architecture decisions

- Single-owner tenancy: `tenantId = userId` (each Supabase user owns their own tenant)
- Auth middleware upserts user record on first login; no separate signup flow needed
- Supabase handles auth; API server verifies JWT with Supabase service role key
- All monetary values stored as text in DB (exact decimal), converted to Number in API responses
- Plans endpoint is public (no auth); all other endpoints require Bearer token

## Product

- **Dashboard** — revenue stats, invoice summary, inventory overview, recent activity
- **Invoices** — create, send, mark paid; printable detail page; auto-deducts inventory on creation
- **Inventory** — track stock, set low-stock alerts, adjust quantities; item detail with stock history
- **Customers** — CRM with spend history, invoice history, balance tracking, WhatsApp CRM link
- **Transactions** — income/expense tracking with NLP natural language entry
- **Debtors** — outstanding balance view across all customers
- **Reports** — monthly business report with PDF download + CSV/Excel export
- **Forecasting** — 6-month historical trends + 3-month AI revenue forecast with Recharts
- **Marketplace** — buy/sell products & services; WhatsApp-based contact; view tracking
- **AI Advisor** — GPT-4o-mini powered advisor with African market context
- **AI Marketing Generator** — generate marketing copy (Facebook, WhatsApp, Instagram, flyers, etc.)
- **CRM** — WhatsApp leads & orders management with status tracking
- **Notifications** — in-app notification center with auto-generated alerts (low stock, paid, lead, team)
- **Audit Logs** — action history across the business account with filters; auto-logged on all CRUD
- **Team** — invite and manage team members with roles
- **Billing** — Free (₦0), Pro (₦3,000/mo), Premium (₦8,000/mo), Enterprise (custom); Paystack + Flutterwave
- **Settings** — profile, business details, dark mode (light/dark/system)
- **PWA** — manifest.json + Apple/Android meta tags for installability

## User preferences

_Populate as you build — explicit user instructions worth remembering across sessions._

## Gotchas

- `req.tenantId` and `req.userId` are typed `string | undefined` on `AuthenticatedRequest`; always cast with `as string` after `requireAuth` middleware runs
- `req.params.id` is typed `string | string[]` in Express 5; always cast with `as string`
- Drizzle `and()` with undefined conditions: use `.filter(Boolean)` array pattern
- The `SUPABASE_URL` secret has `/rest/v1/` appended — strip it for the Supabase client SDK (`VITE_SUPABASE_URL` in `.env` has the correct base URL)
- `pnpm --filter @workspace/db run push` must be run after any schema changes
- After lib changes, run `pnpm run typecheck:libs` before leaf artifact checks

## Pointers

- See the `pnpm-workspace` skill for workspace structure, TypeScript setup, and package details
