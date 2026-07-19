# Incident Response Playbook & Historical Case Studies

## Why every team on this stack needs a written IR plan before launch

Every incident documented below was made worse by improvisation under pressure — deciding *during* the incident who has authority to rotate a production database credential, whether to take the app offline, or how to communicate to users. Decide these things now, while calm.

## Minimum viable incident response plan

1. **Roles:** who can declare an incident, who has authority to rotate production secrets / take the app offline, who communicates externally (even if it's a one-person team, write down what *you* will do in each role so you don't skip a step under stress).
2. **Detection sources:** GitHub secret-scanning alerts, Supabase Security Advisor, Vercel deployment logs, dependency/SCA alerts, user reports, npm registry security advisories.
3. **Triage:** what's the blast radius — which secrets/data could be exposed, for how long, and to whom (public repo logs vs. private; anon key exposure vs. service_role exposure)?
4. **Containment:** the fastest safe action (revoke a token, disable a compromised GitHub Action reference, flip Supabase table RLS back on, roll back a deployment) — favor speed over root-cause certainty at this stage.
5. **Eradication & recovery:** rotate every credential that was potentially exposed (see rotation runbook below), patch the vulnerable dependency/config, redeploy.
6. **Notification:** determine legal/contractual notification obligations (data protection law, customer contracts, and — for regulated entities — NCA/SAMA incident reporting obligations, which can carry specific timelines) *before* an incident, not during.
7. **Post-incident review:** blameless retro — what control would have prevented or caught this sooner; add it to the backlog with an owner and date, not just "we'll be more careful."

## Secret-rotation runbook (generic — adapt per credential type)

When any secret is suspected exposed (found in a public repo, a CI log, a leaked `.env`, a compromised third-party action/package):

1. **Assume compromise immediately** — don't wait to confirm active exploitation before rotating; the cost of unnecessary rotation is far lower than the cost of a delayed one.
2. **Rotate at the source** (generate a new credential in Supabase/Vercel/GitHub/the third-party provider dashboard or API) — don't just remove it from the current codebase, since the old value remains valid until explicitly revoked.
3. **Update every consumer** of the credential (CI secrets, running deployments, other services) before or immediately as you revoke the old one, to avoid a self-inflicted outage.
4. **Purge from git history** if committed (`git filter-repo` or BFG Repo-Cleaner) — recognizing this doesn't undo any exposure that already occurred, but prevents ongoing exposure via repo browsing/cloning.
5. **Audit for use of the compromised credential during the exposure window** — Supabase/Vercel/GitHub audit logs, database query logs — to determine if it was actually used maliciously, informing your notification decision.
6. **Document the timeline** (when exposed, when detected, when rotated) — needed for both the post-incident review and any compliance/regulatory notification.

## Historical incident case studies (root cause → prevented-by)

### tj-actions/changed-files supply chain compromise (March 2025, CVE-2025-30066)
- **Root cause:** compromised bot PAT with push access; attacker retroactively rewrote existing Git tags to point to malicious code.
- **Impact:** CI secrets across 23,000+ repos printed to logs for ~15 hours.
- **Prevented by:** commit-SHA pinning of third-party GitHub Actions (see `references/cicd-supply-chain.md`).

### Shai-Hulud npm worm (September 2025, "2.0" wave November 2025)
- **Root cause:** phishing campaign compromised npm maintainer credentials/tokens; malicious `postinstall`/`preinstall` scripts self-propagated to every other package the compromised maintainer owned.
- **Impact:** 500+ packages compromised in the first wave; secrets exfiltrated to public attacker-created GitHub repos; some organizations' private repos flipped public.
- **Prevented/limited by:** disabling arbitrary lifecycle scripts in CI installs, SCA monitoring for newly-published versions of already-trusted dependencies, isolating CI credentials from install steps, npm 2FA on publish for maintainers.

### Supabase RLS-missing breaches (CVE-2025-48757, and the broader pattern including the 2025 "Moltbook" breach)
- **Root cause:** Postgres/Supabase tables shipped with Row Level Security disabled (the platform default), often in AI-generated scaffolding that produced functionally correct but security-incomplete code.
- **Impact:** full table contents (170+ apps in the CVE-2025-48757 disclosure; ~1.5M tokens and 35K emails in the Moltbook case) readable by anyone with the public anon key.
- **Prevented by:** RLS enabled on every table + explicit per-operation policies + pre-launch anon-key verification (see `references/supabase-security.md`).

### Next.js middleware authorization bypass (March 2025, CVE-2025-29927)
- **Root cause:** framework trusted an internal-only header (`x-middleware-subrequest`) even when supplied by an external client, allowing middleware-based auth checks to be skipped entirely.
- **Impact:** unauthenticated access to routes that relied solely on middleware for protection, on self-hosted (non-Vercel) deployments running vulnerable versions.
- **Prevented by:** framework patching (fast — always keep Next.js current) *and*, structurally, defense-in-depth (never rely on a single layer — re-check authorization at the data-access layer regardless of middleware).

### General lesson across all four: "the platform/tool is secure, but the default configuration or an unpinned dependency wasn't"
This is the throughline across nearly every incident above: none were a case of "the vendor's core product was broken" in the sense of an unpatchable design flaw — Supabase's Postgres engine, GitHub's Actions runtime, npm's registry, and Vercel's edge network are all sound. Every incident was a **shared-responsibility gap**: a default that assumed the developer would configure something (RLS), a trust assumption on a mutable reference (git tags) that the developer could have pinned, or a maintainer-side compromise (npm/GitHub Actions) that better credential hygiene and CI isolation would have contained. This is the argument for treating this skill's checklists as continuous practice, not a one-time launch gate — new tables, new third-party actions, new dependencies each reopen the same categories of gap.

## Dev-environment incident considerations (often overlooked)

- A compromised developer laptop/CI runner is an incident even if production wasn't touched — rotate any credential that machine had access to (local `.env`, cached CLI tokens for Vercel/Supabase/GitHub, SSH keys).
- Staging/preview environments with copies of production data turn a "just a dev breach" into a real data-exposure incident — this is why `references/vercel-nextjs-security.md` and `references/supabase-security.md` both recommend synthetic/anonymized data in non-prod environments.
- Local `.env` files leaked via screen-sharing, misconfigured dotfile sync, or accidental commits are a routine and under-reported incident category — treat with the same rotation urgency as a production leak.
