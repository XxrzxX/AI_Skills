# Application Security: Node.js Backend + React Frontend (OWASP-mapped)

Mapped to OWASP Top 10 (2021, still current as the industry reference) with stack-specific implementation notes. Use this for custom backend logic — anything outside what Supabase's RLS/PostgREST auto-generates (see `references/supabase-security.md` for the DB-layer specifics).

## A01 — Broken Access Control (still #1 by prevalence, ~94% of tested apps have some form)

- **IDOR (Insecure Direct Object Reference):** any endpoint like `GET /api/orders/:id` must verify the *authenticated* user owns/can access that specific `:id` — not just that they're logged in. This is the most common real-world finding across Node APIs regardless of how good the framework is.
- Centralize authorization logic (a policy/permission-check function or middleware) rather than re-implementing ownership checks ad hoc in every route — inconsistency between routes is exactly how these gaps appear.
- If using Supabase for data but a custom Node backend with `service_role` for privileged operations, remember RLS is bypassed — your Node code is now the *only* access control for that path. Treat every `service_role`-backed endpoint as if RLS doesn't exist and re-implement equivalent checks explicitly.
- Don't rely on obscurity (unguessable IDs, hidden UI elements) as an access control — enforce server-side.

## A02 — Cryptographic Failures

- Enforce TLS everywhere (HSTS with `includeSubDomains` and a meaningful `max-age`); Vercel/Supabase handle TLS termination by default, but verify no plaintext HTTP fallback exists.
- Hash passwords with `bcrypt`/`argon2` (never roll your own, never MD5/SHA1/plain SHA256 without a proper KDF) — though if using Supabase Auth, this is handled for you; verify you're not also storing a duplicate/shadow credential store insecurely elsewhere.
- Don't log sensitive data (passwords, tokens, full card numbers, national ID numbers) — see logging section below.
- Use environment-managed secrets for encryption keys, never derive them from predictable values.

## A03 — Injection

- **SQL injection:** if using the Supabase JS client or an ORM (Prisma, Drizzle) with parameterized queries, you're largely protected — the risk reappears when raw SQL is constructed via string concatenation (e.g., inside a Postgres function, an Edge Function, or a custom Node query) with user input. Always parameterize.
- **NoSQL/query injection:** validate the *shape* of input, not just content, when using MongoDB-style query objects — a user submitting `{"$gt": ""}` where a string was expected is a classic operator-injection pattern in Express apps that JSON-parse request bodies directly into queries.
- **Command injection:** never pass user input to `child_process.exec` (string-based); use `execFile`/`spawn` with an argument array, which doesn't invoke a shell.
- **Prototype pollution:** a Node/JS-specific injection class — be cautious with deep-merge libraries and `JSON.parse` of untrusted input flowing into object-merge operations; keep dependencies patched (this class has produced real CVEs in popular npm packages).

## A04 — Insecure Design

- Rate limit and add friction (CAPTCHA, progressive delays) on: login, signup, password reset, OTP verification, and any endpoint that costs you money per call (email/SMS sends, LLM API calls) — abuse of unthrottled endpoints is a common cost-exhaustion / enumeration attack independent of any "bug."
- Design multi-tenant data models with tenant isolation as a first-class concern (see Supabase multi-tenancy note), not bolted on later.

## A05 — Security Misconfiguration

- Remove default/debug routes, verbose framework banners (`X-Powered-By: Express` — disable via `app.disable('x-powered-by')`), and stack traces from production responses.
- Set security headers explicitly (`helmet` middleware for Express is a fast baseline: CSP, `X-Frame-Options`, `X-Content-Type-Options`, etc.) — see also the Next.js-specific header guidance in `references/vercel-nextjs-security.md`.
- Lock down CORS: never `Access-Control-Allow-Origin: *` combined with `Access-Control-Allow-Credentials: true` (browsers block this combination, but misconfigured proxies can still create the equivalent risk) — allowlist specific origins.
- Review default configs on every managed service (Supabase, Vercel) rather than assuming secure defaults — as the Supabase RLS-off-by-default pattern demonstrates, "secure by default" is not a safe assumption for BaaS platforms generally.

## A06 — Vulnerable and Outdated Components

- See `references/cicd-supply-chain.md` for the full dependency-security treatment (SCA tooling, lockfile discipline, Shai-Hulud-style risk).
- Maintain an SBOM (Software Bill of Materials) for anything subject to compliance review — `npm sbom` or CycloneDX tooling can generate this from `package-lock.json`.

## A07 — Identification and Authentication Failures

- **JWT pitfalls specific to this stack:** never accept `alg: none`; verify the algorithm matches what you expect (don't let a client dictate `HS256` vs `RS256` — algorithm-confusion attacks exploit exactly this); verify signature, expiry, and issuer/audience claims on every request, not just on login.
- If issuing your own JWTs alongside Supabase Auth (uncommon, but happens in hybrid setups), don't reuse the Supabase JWT secret for anything else, and keep expiry short with refresh-token rotation.
- Enforce MFA for privileged/admin accounts unconditionally, and offer/encourage it for regular users — required for several NCA ECC and SAMA CSF control considerations for remote/customer-facing access (see `references/compliance-frameworks.md`).
- Invalidate sessions server-side on password change / suspected compromise — a JWT-only stateless design without a revocation mechanism (e.g., a `token_valid_after` timestamp checked against issued-at) can't do this cleanly; plan for it upfront.

## A08 — Software and Data Integrity Failures

- Verify integrity of anything auto-updated in your pipeline (this is the CI/CD supply-chain section's domain — see `references/cicd-supply-chain.md`).
- Avoid insecure deserialization of untrusted data (`eval`, `vm` module on user input, unsafe YAML/pickle-equivalent parsers) — parse, don't execute.

## A09 — Security Logging and Monitoring Failures

- Log authentication events (success/failure, MFA challenges), authorization failures, and administrative actions — but **never log secrets, tokens, passwords, or full PII** (mask/redact before logging; a logging pipeline is itself a data-exposure surface, as the tj-actions incident demonstrated at the CI-log layer).
- Centralize logs (don't rely on scattered `console.log` across serverless functions with no aggregation) and set alerting on anomalous patterns: spike in 401/403s, repeated failed logins from one source, unusual `service_role`/admin API usage.
- Retain logs long enough to support incident investigation per your compliance regime (see NCA/SAMA retention expectations in `references/compliance-frameworks.md`).

## A10 — Server-Side Request Forgery (SSRF)

- Covered in depth in `references/vercel-nextjs-security.md` for the Next.js angle; the general Node principle: any server-side `fetch`/`axios` call using a user-supplied URL or hostname needs an allowlist, DNS-rebinding protection, and a block on internal/link-local address ranges (`127.0.0.1`, `169.254.169.254`, `10.0.0.0/8`, etc.).

## Secrets management summary (cross-cutting)

- Local dev: `.env.local`, gitignored, never committed; use a secrets manager (1Password, Doppler, Vercel env vars pulled via CLI) rather than passing `.env` files around informally.
- CI: platform secret store only (GitHub Actions secrets, not hardcoded YAML).
- Runtime: Vercel env vars / Supabase Vault, scoped per environment (dev/preview/prod) and per least-privilege (a function only gets the secrets it needs).
- Rotation: have a documented, tested rotation procedure for every secret class *before* you need it under pressure (see `references/incident-response-playbook.md`).
- Detection: run secret-scanning (gitleaks/trufflehog) as a CI gate on every PR, and enable GitHub secret scanning + push protection at the repo/org level.
