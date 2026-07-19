# CI/CD & Software Supply-Chain Security (GitHub Actions, npm/Node ecosystem)

## Why this matters more than almost anything else on this stack

Modern breaches on JS/Node stacks increasingly don't come from a bug the team wrote — they come from something the team *trusted*: a GitHub Action, an npm package, a maintainer account. 2025–2026 turned this from theoretical to routine.

## Documented incidents (root cause → lesson)

### tj-actions/changed-files (CVE-2025-30066, March 2025)
- **What happened:** Attacker compromised a bot's GitHub Personal Access Token with push rights to the `tj-actions/changed-files` action (used by 23,000+ repos). They pushed a malicious commit, then retroactively rewrote *every existing version tag* (`v35`, `v44.5.1`, etc.) to point at it. Any workflow referencing the action by tag — even a "pinned" version tag — pulled the malicious code the next run.
- **Payload:** Dumped CI runner process memory, printed secrets (tokens, keys, credentials) to the workflow log in base64, exposed in plaintext on public repos.
- **Compounding factor:** Likely enabled by an earlier compromise of `reviewdog/action-setup` (CVE-2025-30154), a transitive dependency of the tj-actions CI — a supply chain attack that bootstrapped another supply chain attack.
- **Who was actually targeted:** Evidence suggests the campaign began as a targeted attack against Coinbase repos, then broadened indiscriminately once the attacker had access.
- **The one control that would have prevented it:** commit-SHA pinning. Repos pinning `tj-actions/changed-files@<full-40-char-SHA>` instead of `@v45` were unaffected unless they happened to update to a compromised SHA during the ~15-hour window.

### Shai-Hulud npm worm (Sept 2025, "2.0" wave Nov 2025, continuing into 2026)
- **What happened:** Self-replicating worm distributed via compromised npm packages (starting with `@ctrl/tinycolor` and ~500 others). Entry vector: phishing campaign spoofing npm's "update your MFA" emails, harvesting maintainer credentials and npm publish tokens.
- **Propagation:** Once a package was installed, a `postinstall` (v1) or `preinstall` (v2 — runs even earlier, widening blast radius to more CI pipelines and dev machines) script ran, which: stole cloud/CI credentials and npm tokens from the local environment, used the stolen npm token to inject itself into every *other* package the same maintainer owned and auto-publish new "infected" versions, exfiltrated secrets by creating a public GitHub repo named "Shai-Hulud" under the victim's account containing a base64 JSON dump, and attempted to flip private repos to public.
- **Why it's different from prior npm attacks:** Fully automated propagation — no human attacker action needed per victim. Traditional "one bad package" defenses (pin a known-good version) don't fully help because *trusted* packages you already depend on get weaponized after the fact.
- **Preceding/related:** The Nx "s1ngularity" attack (Aug 2025) used a compromised GitHub Actions workflow in the Nx repo to steal maintainer credentials, which fed directly into the broader campaign.
- **Controls that mitigate this:** lockfile integrity + `npm ci` (not `npm install`) in CI, disabling arbitrary lifecycle scripts by default, isolating CI credentials so a compromised dependency can't reach cloud/npm tokens, and rapid detection (SCA tooling that flags newly-published versions of already-installed packages).

## Hardening checklist — GitHub Actions

**Pin everything to a commit SHA, not a tag or branch.**
```yaml
# Bad — mutable, can be silently retargeted
- uses: tj-actions/changed-files@v45
# Bad — same risk
- uses: tj-actions/changed-files@main
# Good — immutable
- uses: tj-actions/changed-files@0a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t
```
Use Dependabot or Renovate with SHA-pinning support to keep pinned actions updated safely (they open a PR with the new SHA + the tag it corresponds to, so a human reviews the diff).

**Least-privilege `GITHUB_TOKEN` permissions.** Default to `permissions: {}` at the workflow level and grant only what each job needs:
```yaml
permissions:
  contents: read
jobs:
  build:
    permissions:
      contents: read
      # only add id-token: write if you need OIDC, packages: write if you publish, etc.
```

**Never let untrusted PRs run with secrets.** Use `pull_request` (not `pull_request_target`) for anything that checks out and executes fork PR code. If you must use `pull_request_target` (e.g., to comment on PRs from forks), never check out and execute the fork's code inside that workflow — that combination is the classic "pwn request" pattern that lets any external contributor exfiltrate your secrets via a malicious PR.

**Don't echo secrets, and assume anything printed to a log can leak.** GitHub masks known secret values in logs, but only if referenced via `secrets.X` directly — a secret that's been transformed (base64, concatenated, reversed) will NOT be masked. Treat CI logs on public repos as a hostile disclosure surface.

**Use OIDC federation instead of long-lived cloud credentials in CI.** GitHub Actions supports OIDC tokens for AWS/GCP/Azure/Vercel — this avoids storing static cloud secrets in `Settings > Secrets` entirely, and the token is scoped to a specific repo/branch/workflow.

**Environment protection rules for prod deployments.** Use GitHub Environments with required reviewers for any workflow that deploys to production or has access to production secrets — this creates a manual gate even if the workflow file itself is compromised.

**Audit third-party Action usage regularly.** Run a periodic search across your org for Actions pinned to tags/branches (not SHAs), and maintain an allowlist of vetted Actions (GitHub Actions policies at the org level can enforce this).

**Harden the runner itself where possible.** Tools like StepSecurity's Harden-Runner can detect anomalous outbound network calls from a CI job (e.g., a compromised action trying to phone home) — this is what actually caught some Shai-Hulud-adjacent activity in the wild.

## Hardening checklist — npm / Node dependency supply chain

- **Use `npm ci` in CI/CD, never `npm install`.** `ci` installs exactly what's in the lockfile and fails on drift; `install` can silently update transitive deps.
- **Commit and enforce the lockfile** (`package-lock.json`); block PRs that modify it without an explicit dependency-update commit.
- **Disable lifecycle scripts for untrusted installs where feasible**: `npm config set ignore-scripts true` in CI for install steps that don't require native builds, or use `npm ci --ignore-scripts` and explicitly run only the scripts you need.
- **Pin exact versions for security-sensitive or high-blast-radius dependencies** (auth libraries, crypto libraries) rather than `^`/`~` ranges — you want to review before you get a new version, not accept it silently on next install.
- **Run SCA (software composition analysis) in CI** — `npm audit`, GitHub Dependabot alerts, Socket.dev, Snyk, or similar — but treat "newly published version of a package you already trust" as its own risk category, since Shai-Hulud-style attacks weaponize *already-approved* dependencies.
- **Enable npm 2FA + provenance for anything your org publishes**, and require it for maintainers.
- **Verify package provenance/attestation** where available (`npm audit signatures`) — confirms a package was actually built from its claimed source repo via a trusted CI pipeline rather than uploaded manually from a possibly-compromised developer machine.
- **Isolate CI credentials from build steps that run third-party code.** Don't inject cloud deploy credentials into the same job/step context where `npm ci` executes untrusted postinstall scripts if it can be split into separate jobs with scoped secrets.
- **Rotate npm tokens periodically and scope them** (publish-only tokens with 2FA-required, not full-access tokens, for CI publishing).

## Secrets in CI/CD generally

- Never hardcode secrets in workflow YAML, Dockerfiles, or source — use the platform's secret store (GitHub Actions secrets, Vercel env vars, Supabase Vault) and reference them.
- Rotate any secret that was ever exposed in a public log, git history, or a compromised third-party action — assume compromise, don't wait for proof of misuse (see `references/incident-response-playbook.md` for the rotation runbook).
- Scan git history for accidentally committed secrets with tools like `gitleaks` or `trufflehog` as a CI gate on every PR, not just once.
- Treat `.env` files as radioactive: never commit them (enforce via `.gitignore` + a pre-commit hook + a CI gate that fails if one is staged), and use `.env.example` with placeholder values for onboarding.
