# Threat Model

## Project Overview

HustleOS AI is a public-facing SaaS platform for SMEs that exposes an Express API backed by PostgreSQL plus a browser frontend for business management workflows. The documented primary production surfaces are `artifacts/api-server` and `artifacts/hustle-os`; there is also a newer `artifacts/hustle-os-nextjs` frontend that may be an alternate or in-progress production client, so both frontends remain in scope where they touch the same API or auth model. Users authenticate with Supabase Auth; the backend verifies bearer tokens server-side and performs tenant-scoped CRUD over invoices, inventory, customers, CRM, reports, AI features, subscriptions, team management, and file uploads. Production deployment is public on Replit autoscale. Per platform assumptions, TLS is handled by the platform and `NODE_ENV` is `production` in production.

## Assets

- **User accounts and bearer tokens** — Supabase identities, access tokens, team membership, and effective tenant context. Compromise permits impersonation and cross-tenant access.
- **Business records** — invoices, customers, inventory, CRM leads/orders, notifications, audit logs, reports, and forecasts. These contain sensitive commercial data and some personal/contact data.
- **Billing state** — subscription plans, active subscription rows, payment provider references, and webhook processing state. Integrity matters because billing unlocks product features.
- **Uploaded media and document links** — Cloudinary-hosted images/videos and shared invoice artifacts. Improper access or deletion could damage customer data or public assets.
- **Application secrets and third-party credentials** — database URL, Supabase service-role key, OpenAI key, Cloudinary credentials, Paystack/Flutterwave secrets, email provider keys.

## Trust Boundaries

- **Browser to API** — all frontend input is untrusted and must be authenticated, authorized, validated, and scoped server-side.
- **API to PostgreSQL** — the API has broad read/write access to tenant data; authorization bugs here become direct cross-tenant data exposure or tampering.
- **API to Supabase Auth** — authentication depends on correct server-side token verification and correct mapping from user identity to tenant/role.
- **API to external services** — billing, AI, email, and Cloudinary calls cross into third-party systems using server-held secrets.
- **Public to authenticated surfaces** — most `/api` routes are authenticated, but some onboarding, billing, webhook, and shared-invoice style endpoints are intentionally public and need extra scrutiny.
- **Production vs dev/demo artifacts** — `artifacts/api-server` is the primary server trust boundary. `artifacts/mockup-sandbox` is treated as dev-only per platform assumptions unless proven reachable. Video artifacts are treated as non-production. `artifacts/hustle-os-nextjs` is in scope only for auth/API interactions, not as a separate backend.

## Scan Anchors

- Primary backend entry points: `artifacts/api-server/src/index.ts`, `artifacts/api-server/src/app.ts`, and `artifacts/api-server/src/routes/*.ts`.
- Primary auth boundary: `artifacts/api-server/src/lib/auth.ts` (`requireAuth`) plus any route that omits it.
- Highest-risk areas: subscription checkout/webhooks/redirect verification, team/invite flows, invoice/public-share style flows, uploads, report/export generation, and AI endpoints that call external services.
- Public routes worth re-checking on every scan: `/healthz`, auth onboarding/reset endpoints, `/subscriptions/plans`, payment webhooks/redirect handlers, and any invoice/share tracking endpoints.
- Usually ignore unless production reachability is demonstrated: `artifacts/mockup-sandbox`, video artifacts, generated `dist/`, and `.next/` build output.

## Threat Categories

### Spoofing

The application relies on bearer tokens from Supabase and maps each authenticated principal to a tenant and role. Protected routes must require a valid bearer token, and public callback/webhook routes must authenticate the calling provider or otherwise prove the caller is authorized to act on the referenced tenant or resource.

### Tampering

Users can mutate invoices, inventory, customers, CRM records, team state, and billing state. The server must derive authorization and sensitive business state from trusted server-side data, not from client-controlled identifiers or parameters, and payment/webhook flows must not allow attacker-controlled parameters to activate plans or alter records for another tenant.

### Information Disclosure

Tenant business data is the core value of the product, so every record-returning endpoint must enforce tenant scoping server-side. Public endpoints, exports, logs, and generated documents must not reveal cross-tenant data, secrets, stack traces, or internal identifiers beyond what the feature intentionally shares.

### Denial of Service

The app exposes AI, upload, and export-like endpoints that can consume CPU, memory, bandwidth, or paid third-party quota. Public or weakly protected routes must have bounded body sizes and reasonable rate limits, and integrations should avoid attacker-triggered expensive work without sufficient gating.

### Elevation of Privilege

The most serious risks are broken tenant isolation, team-role bypasses, insecure public endpoints that mutate state, and payment/integration flows that let one user act on another tenant’s resources. All database writes and reads must be bound to the effective tenant from authenticated context unless the route is intentionally public and backed by a separate unguessable capability token or verified provider signature.
