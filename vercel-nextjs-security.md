# Vercel Deployment & Next.js/React Security

## Documented incident: CVE-2025-29927 — Next.js Middleware Authorization Bypass

- **What happened:** Next.js middleware uses an internal header, `x-middleware-subrequest`, to prevent infinite loops when middleware triggers a subrequest. Versions from 11.1.4 through 15.2.2 trusted this header even when it arrived from an *external* client — an attacker could send a request with `x-middleware-subrequest: middleware:middleware:middleware:middleware:middleware` (or version-specific variants) and the framework would skip middleware execution entirely, including any auth/session checks implemented there.
- **Impact:** Any app using middleware as its *sole* authorization layer for protected routes (e.g., checking a session cookie in `middleware.ts` and redirecting unauthenticated users) could be bypassed with a single crafted header — no valid session needed. CVSS 9.1.
- **Important nuance for this stack specifically: Vercel-hosted deployments were not affected** — Vercel's edge network strips/normalizes this header before it reaches the app. **Self-hosted Next.js deployments (Docker, custom Node server, other platforms) were vulnerable** if running an affected version.
- **The generalizable lesson, independent of the patch:** *middleware should never be your only authorization layer.* Even patched, treat middleware as a fast-path UX optimization (redirect unauthenticated users to `/login` quickly) and always re-check authorization at the data-access layer (in the route handler / server action / database query itself). If middleware is bypassed by some future flaw, a properly defense-in-depth app degrades to "slow redirect didn't happen" rather than "full data exposure."

## Vercel platform-specific hardening

**Environment variable scoping.** Vercel lets you scope env vars to Production / Preview / Development independently — use this deliberately:
- Never put production database credentials, `service_role` keys, or payment-provider secret keys in a scope reachable by Preview deployments unless the preview environment is fully isolated (ideally: use separate Supabase/Stripe/etc. projects for preview vs. prod).
- Anything prefixed for client exposure (`NEXT_PUBLIC_*`) is bundled into client-side JS and downloadable by anyone visiting the site — audit these prefixes specifically for anything that shouldn't be public.

**Preview deployments are, by default, reachable by anyone with the URL.** Preview URLs are unguessable but not secret — if shared in a Slack channel, PR comment, or logged, they can leak.
- Enable **Vercel Deployment Protection** (password protection, Vercel Authentication/SSO, or trusted-IP allowlisting) for Preview (and, for sensitive projects, Production) deployments, especially on team/enterprise plans.
- Treat preview deployments as pre-production: if they connect to a database, it should be a preview/staging dataset (synthetic or anonymized), not a copy of live customer data.

**Don't rely on client-side "hiding" via build config.** Anything in a client bundle can be extracted regardless of minification/obfuscation — that includes API base URLs, feature flags, and (mistakenly included) keys.

**Vercel Firewall / WAF.** Use Vercel's built-in Web Application Firewall and rate limiting for public API routes, especially anything unauthenticated (signup, contact forms, webhooks) to blunt credential stuffing, scraping, and volumetric abuse.

**Serverless/Edge Function isolation.** Each Vercel Function execution is isolated, but a compromised dependency running inside a Function still has access to whatever environment variables are injected into that function's scope — apply the same "least secrets per surface" principle as you would to CI: don't inject secrets a given API route doesn't need.

**Webhooks (Stripe, GitHub, Supabase, etc.) hitting Vercel routes must verify signatures.** Never trust webhook payloads based on source IP or "it came from the right URL" — always verify the provider's HMAC/signature header before acting on the payload, and reject on failure.

## Next.js / React application-layer risks

**Server Actions and Route Handlers are backend code — treat them like it.** A common mistake porting from a traditional client/server split is treating a Server Action as "just a form submit" and skipping the authorization check that an equivalent Express route would obviously need. Every Server Action and Route Handler must independently verify: who is calling (auth), and whether they're allowed to do *this specific thing to this specific resource* (authz) — don't assume the UI only exposes the action to authorized users, since the endpoint is directly callable.

**Don't leak internal errors to the client.** Next.js's default error overlay in dev mode is verbose by design — ensure production builds return generic error messages and log full stack traces server-side only (see `references/appsec-node-react.md` for logging guidance).

**SSRF via user-controlled URLs.** Any feature that fetches a URL supplied by the user (image proxies, link previews, webhook test buttons, PDF-from-URL generators) is a classic SSRF vector that can be used to reach internal services, cloud metadata endpoints (e.g., `169.254.169.254`), or Supabase's internal network. Validate and allowlist schemes/hosts, and don't let the server fetch arbitrary user-supplied URLs without restriction. (Next.js's built-in Image Optimization endpoint has itself had historical SSRF-adjacent CVEs from unrestricted remote image domains — keep `next.config.js`'s `images.remotePatterns`/`domains` allowlist as narrow as possible, and keep Next.js patched.)

**Content Security Policy and standard security headers.** Set CSP, `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `Strict-Transport-Security`, and `Permissions-Policy` via `next.config.js` headers or middleware (in addition to, not instead of, data-layer authz per the CVE-2025-29927 lesson above). A solid CSP is one of the most effective mitigations against XSS impact even if an injection slips through.

**React-specific XSS traps:** `dangerouslySetInnerHTML` is exactly what it says — sanitize with a vetted library (e.g., DOMPurify) before ever passing user-generated HTML through it. Rendering user content as a URL (`href={userInput}`) without validating the scheme allows `javascript:` URI XSS.

**Keep Next.js and React on patched versions on a fixed cadence.** Given the CVE-2025-29927 pattern (framework-level auth bypass, not app-code bug), staying current on the framework is itself a security control, not just a feature-update nicety.

## Dev vs. prod distinctions for this layer

| | Development | Production |
|---|---|---|
| Error detail | Full stack traces useful for debugging (fine, local only) | Generic error messages to client; full detail to server-side logs/monitoring only |
| Preview protection | Can be looser for solo dev | Password/SSO-protect previews once a team or external stakeholders are involved |
| Env vars | Local `.env.local` (gitignored), never real production secrets | Scoped Vercel env vars per environment, rotated on any suspected exposure |
| CSP | Can be relaxed (`unsafe-eval` for HMR) | Strict CSP, no `unsafe-inline`/`unsafe-eval` without a specific justified exception |
| Debug/admin routes | Fine to expose locally | Must be authenticated + ideally IP-restricted or removed entirely from prod builds |
