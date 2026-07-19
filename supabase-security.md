# Supabase Security (Postgres, Auth, Storage, Edge Functions, Realtime, MCP)

## The core mental model

Supabase auto-generates a REST API (PostgREST) directly from your Postgres schema. Every table you create is reachable at `/rest/v1/<table>` the moment it exists. Security is **not** "hide the API key" — the `anon` key is *designed* to be public, embedded in frontend JS bundles. The actual security boundary is **Row Level Security (RLS)** policies on each table. If you understand nothing else about Supabase security, understand this one sentence: *the anon key is a guest pass, RLS is the bouncer, and Postgres ships with the bouncer off by default.*

## The headline incident: CVE-2025-48757 and the broader RLS epidemic

- Security researcher Matt Palmer disclosed (May 2025) that 170+ apps built with the AI app-builder Lovable (10.3% of a 1,645-app sample) shipped with Supabase tables fully readable via the public `anon` key — no authentication required — because RLS was never enabled on tables holding real user data.
- Industry-wide scans put **RLS misconfiguration behind roughly 80%+ of critical Supabase findings**, and one high-profile 2025 breach ("Moltbook") exposed ~1.5 million API auth tokens and ~35,000 emails within three days of launch through exactly this pattern.
- Root cause pattern is structural, not just carelessness: Postgres tables have RLS **disabled by default**, AI code-generation tools scaffold working CRUD code without adding policies (the app *works* in testing because the developer is usually testing as an authenticated user who "should" have access), and the failure is invisible until someone runs a `curl` against the anon key.
- **PostgREST itself has had CVEs too** — CVE-2022-35912 allowed authenticated users to escalate to superuser via the `db-pre-request` hook, bypassing RLS entirely at the middleware layer. Patched, but a reminder that RLS-policy correctness isn't the *only* layer that matters.

## Mandatory checklist before any table goes to production

1. **Enable RLS on every table in the `public` schema**, including lookup/reference tables — even "harmless" tables can leak schema/business info or be a pivot point.
   ```sql
   ALTER TABLE your_table ENABLE ROW LEVEL SECURITY;
   ```
2. **Turn on the project-level "Enable RLS on new tables" default** (Dashboard → Authentication → Policies) so you can't forget on the next table.
3. **Write explicit policies for every operation you allow** — `SELECT`, `INSERT`, `UPDATE`, `DELETE` are independent; enabling RLS with zero policies makes a table fully *inaccessible* (fails closed, which is good), but developers often then write one overly broad policy to "make it work."
4. **Never write a policy with `USING (true)` or `WITH CHECK (true)` for user-owned data.** That's RLS enabled in name only.
5. **Understand the two-clause split on `UPDATE` policies:** `USING` controls which existing rows a user can target; `WITH CHECK` validates the *new* row values after the update. Missing `WITH CHECK` lets a user update their own row into someone else's identity/ownership fields.
6. **`INSERT` policies need a matching `SELECT` policy** if you want the client to get the inserted row back (Postgres re-selects after insert to return it) — a common source of confusing "policy violation" errors that leads developers to loosen policies out of frustration rather than add the correct one.
7. **Test policies as each role**, not just as an authenticated owner: anonymous, authenticated-but-not-owner, and (if used) other custom roles. Supabase's Security Advisor (Dashboard) flags tables with RLS off or with `USING (true)`-style policies — run it before every launch.
8. **Prefer `(select auth.uid())` over bare `auth.uid()`** inside policies for performance at scale (the wrapped form lets Postgres cache it per-statement instead of re-evaluating per-row) — not a security issue per se, but a common footgun that leads teams to loosen policies "for performance" instead of fixing the query pattern.

## `service_role` key — the god-mode key

- Bypasses RLS entirely. Full read/write on every table, function, and storage object.
- **Must never** appear in frontend code, mobile app bundles, public repos, CI logs, or client-side environment variables (anything prefixed `NEXT_PUBLIC_`, `VITE_`, `REACT_APP_` is bundled into the client — never put `service_role` behind one of these prefixes).
- Use it only in trusted server-side contexts (Supabase Edge Functions, your own Node.js backend, server-only Next.js code) — and even there, scope down further where possible (see custom roles below).
- Supabase's 2025 key-format change: `sb_publishable_xxx` (replaces `anon`, safe to expose) and `sb_secret_xxx` (replaces `service_role`, never expose) — migrate to the new naming so the format itself signals intent in code review.
- **AI coding assistants / MCP servers are a new leak vector.** If a Supabase MCP server is configured with the `service_role` key so an AI assistant can query the database directly, a malicious or attacker-controlled *row value* in the database can act as a prompt-injection payload that manipulates the assistant into running destructive or exfiltrating SQL — because the assistant reads untrusted data and trusted instructions through the same channel. **Mitigation:** never give an AI assistant MCP access with the `service_role` key; if AI-assisted DB access is needed, create a dedicated read-only Postgres role scoped to non-sensitive tables/views and use that for MCP.

## Storage buckets

- Buckets are private by default but are frequently made public without realizing objects (including e.g. user-uploaded documents, generated PDFs) inherit no further access control once public — RLS-style policies exist for Storage too (`storage.objects` table policies); use them for anything containing PII.
- Signed URLs for private objects should have the shortest practical expiry.
- Validate file type/size server-side (or via Edge Function) before/at upload — client-side validation is not a security control.

## Edge Functions

- Treat as regular server-side code: validate all input, don't trust the JWT claims without verifying signature/expiry (the Supabase client libraries do this, but hand-rolled fetches to the function endpoint must verify explicitly).
- Store secrets used by Edge Functions in Supabase's secret manager (`supabase secrets set`), not hardcoded.
- Rate-limit or add auth checks on functions that call paid third-party APIs (email, SMS, LLMs) to prevent abuse/cost-exhaustion attacks.

## Auth / JWT configuration

- Rotate the JWT signing secret if ever exposed; understand this invalidates all existing sessions.
- Enforce MFA for any admin/staff Supabase accounts and, per NCA ECC / SAMA CSF expectations (see `references/compliance-frameworks.md`), for customer-facing accounts on sensitive apps.
- Configure appropriate session/JWT expiry — long-lived tokens increase the exposure window if leaked.
- Set a restrictive **Site URL** and **Redirect URLs allow-list** in Auth settings to prevent open-redirect abuse of the OAuth/magic-link flow.
- Review rate limits on auth endpoints (signup, password reset, OTP) to prevent enumeration and brute force — Supabase has built-in rate limiting but defaults may need tightening for a public-facing app.

## Realtime

- Realtime subscriptions respect RLS on the underlying table **only if you've enabled it correctly for Realtime** — verify that broadcast/presence channels aren't leaking data cross-tenant in a multi-tenant app; this is a common multi-tenancy isolation gap distinct from REST-API RLS.

## Multi-tenant isolation specifically

- If building multi-tenant SaaS on Supabase, isolation must be enforced at the RLS-policy level with an explicit tenant/organization ID check (`tenant_id = (select auth.jwt() ->> 'tenant_id')` or via a join to a membership table) — do not rely on the application layer alone to filter by tenant, since any direct REST/Realtime access bypasses app-layer filtering.

## Pre-launch verification (fast sanity check)

```bash
# Should return [] (empty) for any table containing non-public data.
# If it returns rows, RLS is missing or misconfigured on that table.
curl -X GET 'https://<project>.supabase.co/rest/v1/<table>?select=*' \
  -H "apikey: <anon_key>"
```
Run this against every table with the anon key before launch, and again periodically — schema changes can silently reintroduce the gap (e.g., a new table created via SQL editor, which defaults to RLS-off).
