---
name: fullstack-security-hardening
description: Comprehensive security review and hardening skill for a React + Node.js + GitHub Actions + Supabase + Vercel stack, covering both development and production environments. Use this whenever the user asks to audit, harden, review, or "make secure" any part of this stack — CI/CD pipelines, secrets, dependencies, authentication, Supabase RLS/database, Vercel deployment config, API/backend code, or compliance posture (NCA ECC, SAMA CSF, ISO 27001/27002). Also trigger for "security checklist", "pre-launch security review", "incident response plan", "supply chain security", "pentest prep", or requests referencing Saudi regulatory cybersecurity requirements. Grounded in real incidents (tj-actions/changed-files 2025, Shai-Hulud npm worm, Supabase RLS breaches, Coinbase supply-chain compromise) — pulls in specific reference files rather than giving generic advice.
---

# Full-Stack Security Hardening (React / Node.js / GitHub Actions / Supabase / Vercel)

## Purpose

Give a concrete, evidence-based security posture across the whole SDLC — code, CI/CD, dependencies, runtime, database, and deployment — for a stack of: **React (frontend)**, **Node.js (backend/API)**, **GitHub Actions (CI/CD)**, **Supabase (Postgres/BaaS)**, **Vercel (hosting/edge)**. This is not generic security advice — every recommendation below is either (a) a control that closes a specific documented attack pattern, or (b) mapped to NCA ECC-2:2024, SAMA CSF, or ISO 27001:2022/27002:2022.

This skill assumes the person is a developer or security-conscious founder, not a security novice — use precise terminology, but explain WHY a control matters (the failure mode it prevents), not just what to configure.

## How to use this skill

1. **Scope the ask first.** Don't dump the entire checklist unprompted. Figure out if the person wants: (a) a full audit of an existing repo/project, (b) hardening guidance before first production launch, (c) a specific incident/vulnerability class (e.g. "is my Supabase safe?"), or (d) compliance mapping (NCA/SAMA/ISO). If ambiguous and the answer isn't inferable from context, ask one clarifying question via `ask_user_input_v0` (e.g., dev vs. prod focus, which frameworks matter, whether they want an audit of actual code/config or a reference checklist).
2. **If actual code/config is available** (uploaded files, a repo, `.github/workflows/*.yml`, `package.json`, Supabase migrations, `vercel.json`), inspect it directly with `view`/`bash_tool` rather than only giving generic advice — grep for the concrete anti-patterns listed in the reference files (hardcoded secrets, unpinned actions, `service_role` key usage, `SELECT *` policies, wildcard CORS, etc.).
3. **Pull in only the relevant reference file(s)** below rather than loading everything — each is self-contained and organized so you can jump straight to the relevant section.
4. **Prioritize by exploitability, not by checklist completeness.** Lead with what's actively being exploited in the wild right now on this exact stack (npm worm propagation, unpinned GitHub Actions, missing Supabase RLS, exposed service-role keys) before secondary hardening.
5. **When producing a deliverable** (audit report, checklist, remediation plan), prefer a markdown artifact or `.docx` per the standard file-creation rules — don't dump a 500-line checklist inline in chat unless the person explicitly wants a quick answer.
6. For anything touching actual exploit code, working proof-of-concept payloads, or step-by-step attack instructions against a system the person doesn't own — decline per standard policy; this skill is for defense (hardening, detection, incident response), not offense.

## Reference files (load the ones relevant to the request)

| File | Covers |
|---|---|
| `references/cicd-supply-chain.md` | GitHub Actions hardening, npm/dependency supply-chain defense, secrets in CI, artifact integrity — grounded in tj-actions/changed-files (CVE-2025-30066), reviewdog/action-setup (CVE-2025-30154), Shai-Hulud npm worm, Nx/s1ngularity attack |
| `references/supabase-security.md` | Row Level Security, anon vs service_role key, Postgres hardening, Storage buckets, Edge Functions, Realtime, Auth/JWT config, MCP/AI-assistant risk — grounded in CVE-2025-48757 and the broader RLS-misconfiguration epidemic |
| `references/vercel-nextjs-security.md` | Vercel project/env-var hardening, preview-deployment exposure, Next.js/React app security (SSRF via server actions, middleware auth bypass, image-optimization SSRF history), edge config, DDoS/WAF |
| `references/appsec-node-react.md` | OWASP Top 10 mapped to Node.js/Express/React specifics: authN/authZ, input validation, XSS/CSRF/CSP, JWT pitfalls, dependency management, secrets management, logging without leaking secrets |
| `references/compliance-frameworks.md` | NCA ECC-2:2024 (5 domains/114+ controls), SAMA CSF (4 domains), ISO/IEC 27001:2022 + 27002:2022 Annex A controls — mapped to concrete technical controls in this stack, with a control-to-implementation crosswalk |
| `references/incident-response-playbook.md` | Dev vs. prod incident response procedures, secret-rotation runbook, historical incident case studies (2020–2026) with root cause and the specific control that would have prevented each one |

## The eight failure modes that cause the most real breaches on this exact stack

Lead with these — they are the highest-frequency, highest-impact issues actually observed in production incidents on React/Node/GitHub Actions/Supabase/Vercel stacks, not theoretical risks:

1. **Supabase tables shipped with RLS disabled** (Postgres default: OFF). Anyone holding the public `anon` key — which is meant to be public — can read/write every row. This is the single largest source of Supabase breaches (CVE-2025-48757 alone hit 170+ apps; industry scans put ~80%+ of Supabase critical findings in this bucket). → `references/supabase-security.md`
2. **`service_role` key leaked to the client or to an AI coding assistant/MCP server.** It bypasses RLS entirely — full database god-mode. → `references/supabase-security.md`
3. **Unpinned GitHub Actions** (`uses: org/action@v4` or `@main` instead of a commit SHA). A tag can be silently retargeted by a compromised maintainer account, as happened to tj-actions/changed-files (23,000+ repos, CVE-2025-30066) and reviewdog/action-setup (CVE-2025-30154). → `references/cicd-supply-chain.md`
4. **npm dependency/maintainer-account compromise propagating through `postinstall`/`preinstall` scripts** — the Shai-Hulud worm (Sept–Nov 2025) self-replicated across 500+ packages this way, and the Nx/s1ngularity attack used the same vector. → `references/cicd-supply-chain.md`
5. **Secrets committed to git history or printed in CI logs.** Even after deletion, git history retains them; the tj-actions incident exposed secrets purely through log output, not exfiltration. → `references/cicd-supply-chain.md` and `references/appsec-node-react.md`
6. **Vercel preview deployments left publicly accessible with production-like env vars**, or environment variables scoped to the wrong environment (Preview/Dev secrets reachable in a public preview URL). → `references/vercel-nextjs-security.md`
7. **Broken access control in custom Node.js/Express API routes** (IDOR, missing ownership checks) — still OWASP's #1 category by prevalence, and common even when Supabase RLS is used correctly for direct table access but bypassed by custom backend logic that uses `service_role`. → `references/appsec-node-react.md`
8. **No incident response plan or secret-rotation runbook**, meaning that when (not if) one of the above happens, response is improvised under pressure. → `references/incident-response-playbook.md`

## Working style for this skill

- Cite the *why* (attack pattern / real incident) alongside every *what* (control) — this is what makes the guidance stick versus a generic checklist.
- When mapping to NCA/SAMA/ISO, give the control ID crosswalk, not just "this satisfies compliance" — compliance reviewers need the specific control reference.
- Distinguish clearly between **platform-managed security** (what Supabase/Vercel/GitHub secure by default) and **shared-responsibility items** (what the user must configure) — most real incidents on this stack are shared-responsibility failures, not platform vulnerabilities.
- Keep dev-environment and prod-environment guidance distinct where they diverge (e.g., MFA enforcement, debug endpoints, verbose error messages, seed data, `.env` handling).
