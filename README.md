# Amit Furniture — A.F Public

**Build things with love and care.**
Official customer-facing website for Amit Furniture (owner: Amit Kumar Sharma, est. 2026), serving Siliguri, Darjeeling, Jalpaiguri and North Bengal.

## Repository inspection & architecture decision

The starting repository was a fresh Next.js 16 (App Router) + Tailwind v4 + Drizzle ORM template connected to a **local PostgreSQL** instance (`src/db/index.ts`, `DATABASE_URL`). Only a health route and empty schema existed.

**Documented substitution:** the specification proposes Supabase (Postgres/Auth/Storage). This environment provides local PostgreSQL via Drizzle instead, so the equivalent trust boundaries are implemented in the application backend:

- **Auth** → server-side email/password (scrypt) with HTTP-only session cookies, one-time email verification, password reset tokens, session invalidation on password change, rate limiting on sensitive endpoints.
- **Row Level Security** → every customer query is scoped by the authenticated session's user id inside the API route (ownership enforced in SQL predicates, never by UI filtering).
- **Storage policies** → uploads are stored outside `/public` in `private-uploads/` and served only through `/api/files/[id]` after an ownership check.
- **Business rules** (serial generation, atomic acceptance, order confirmation, one-week rule, completion balance check, review cooldown, moderation history) → enforced exclusively in trusted API routes with transactions and row locks.

On a Supabase deployment the same schema maps directly onto Postgres + RLS policies; the API layer remains the authority.

## Demo accounts (seeded)

| Role | Email | Password |
|---|---|---|
| Owner (A.F Admin console at `/admin`) | `owner@amitfurniture.example` | `Owner@2026` |
| Customer (verified, records in every stage) | `priya@example.com` | `Customer@2026` |

Seed: `npx drizzle-kit push` then `npx tsx scripts/seed.ts`.

## Key workflow guarantees

- Enquiry submission creates an internal reference only — **no quotation serial**.
- Serials come from the Postgres sequence `quotation_serial_seq`, assigned **atomically on acceptance** (`SELECT … FOR UPDATE` + guarded update; double clicks cannot duplicate).
- Accepted-quotation **PDF only after acceptance**; generated from DB records with pdf-lib; accepted records immutable.
- **No validity extension** — expiry is final; new quotation required.
- Acceptance ≠ project. **Project created only on A.F order confirmation**, which enforces the **one-week start rule** (7 calendar days, pending final operational confirmation) with a documented-exception escape hatch — never a silent decline.
- Additional work **Option A** continues the project; **Option B** pauses it until dual approval or A.F rejection; completed projects use follow-up requests.
- Payments/refunds are separate ledger entries; totals derived; **completion blocked while outstanding > 0**.
- Reviews require a **completed, owned project**; public username stored separately from legal name (and rejected if it matches name/email); likes enforce a **10-minute cooldown** server-side with a first-like notice.
- Account deletion disables access + anonymises personal data while retaining financial records and public reviews.

## Pending configuration (see `src/lib/config.ts`)

Business contact details, production domain, transactional email provider (verification/reset links are surfaced in a **clearly-labelled development mode** until then), final serial format, PDF legal wording, all legal policies, final upload limits, backup schedule/retention, audit retention, and calendar-day interpretation of the one-week rule.

## Structure

- `src/app` — public pages, auth pages, quote flow, `/account/*` dashboard, `/admin` owner console, `/api/*` trusted backend
- `src/db/schema.ts` — Drizzle schema (users, sessions, enquiries, items, files, messages, quotations, projects, milestones, updates, payments, reviews, likes, additional requests, account requests, audit log)
- `src/lib` — config placeholders, auth, business rules, finance derivation, uploads, public content data
- `scripts/seed.ts` — demo data + serial sequence

## Commands

```
npm run build         # production build
npx next typegen      # route types
npm exec tsc -- --noEmit
npx drizzle-kit push  # apply schema
npx tsx scripts/seed.ts
```