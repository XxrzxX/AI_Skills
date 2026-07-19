# Compliance Crosswalk: NCA ECC, SAMA CSF, ISO/IEC 27001 & 27002

## Scope note

These are Saudi regulatory/national frameworks (NCA, SAMA) plus the international standard (ISO 27001/27002) most commonly required alongside them. They apply *mandatorily* to specific sectors — but the underlying controls are good practice for any production stack. Below: what each framework actually requires structurally, then a crosswalk to concrete controls on this exact stack (React/Node/GitHub Actions/Supabase/Vercel).

**Important caveat to state to the user:** none of this constitutes a compliance certification or legal determination of applicability — recommend a qualified GRC/compliance advisor and (for regulated entities) direct engagement with the NCA/SAMA for authoritative scoping and audit. This reference explains structure and gives an engineering-level starting point, not a certification.

## NCA Essential Cybersecurity Controls (ECC)

- Published by Saudi Arabia's National Cybersecurity Authority. Current baseline version is **ECC-2:2024** (updated from ECC-1:2018).
- **Mandatory for:** government entities, and private-sector organizations owning/operating/hosting Critical National Infrastructure (CNI). Recommended as best practice more broadly.
- **Structure:** 5 main domains, ~29 subdomains, 114+ controls, built on four pillars: Strategy, People, Process, Technology. Explicitly designed with reference to ISO 27001, NIST CSF, and NIST SP 800-53.
- **The 5 domains:**
  1. **Cybersecurity Governance** — strategy, policies, roles/responsibilities (including a mandated full-time Saudi cybersecurity professional in the CISO-equivalent role for in-scope entities), risk management, compliance with laws/regulations, periodic audit, cybersecurity in HR (screening, awareness training, termination processes), cybersecurity in project/change management.
  2. **Cybersecurity Defense** — asset management, identity and access management (including MFA for remote access — directly relevant to any admin access to Vercel/Supabase/GitHub from outside a corporate network), email protection, network security, mobile device security, cryptography, backup/recovery, vulnerability management, penetration testing, application/system security (secure coding standards, security requirements checklist for development), physical security, event log/monitoring management.
  3. **Cybersecurity Resilience** — business continuity management, cybersecurity incident and threat management (aligned with a documented incident-response plan — see `references/incident-response-playbook.md`).
  4. **Third-Party and Cloud Computing Cybersecurity** — this is the domain most directly implicating Supabase and Vercel as third-party/cloud providers: contractual cybersecurity obligations with vendors, risk assessment of cloud services before adoption, and (a related, separate NCA document) the **Cloud Cybersecurity Controls (CCC)** which splits responsibilities explicitly between Cloud Service Provider (CSP) and Cloud Service Tenant (CST) obligations — relevant when evaluating Supabase/Vercel's shared-responsibility model.
  5. **Industrial Control Systems (ICS/OT) Cybersecurity** — not typically applicable to a standard web SaaS stack; relevant only if the product interfaces with industrial/OT systems.

### NCA ECC → this stack, practical crosswalk

| ECC Domain/theme | Concrete control on this stack |
|---|---|
| Identity & Access Management (MFA for remote access, privileged access mgmt) | MFA enforced on GitHub org, Vercel team, Supabase dashboard, and any admin panel; separate privileged (`service_role`) credentials from day-to-day accounts |
| Secure coding / application security requirements | The full `references/appsec-node-react.md` checklist; security requirements gate in PR review/CI |
| Vulnerability management & pentesting | SCA scanning in CI (`references/cicd-supply-chain.md`), periodic external pentest, Supabase Security Advisor run pre-launch |
| Backup and recovery | Supabase automated backups configured and *tested* (restore drills, not just backup existence) |
| Event log/monitoring | Centralized logging per `references/appsec-node-react.md` A09; retained per policy |
| Third-party/cloud cybersecurity | Documented risk assessment of Supabase/Vercel/GitHub as processors; review their SOC 2/compliance attestations; contractual data-processing terms |
| Cybersecurity incident management | Documented, tested IR plan — `references/incident-response-playbook.md` |

## SAMA Cyber Security Framework (CSF)

- Published by the Saudi Central Bank (SAMA), first issued **May 2017**, mandatory for SAMA-regulated **Member Organizations**: banks, insurance companies, finance companies, fintech/payment institutions, credit bureaus, and financial market infrastructure operators.
- **Principle-based / risk-based**, not a rigid checklist — each subdomain states a principle, an objective, and control considerations; organizations tailor implementation to context, with a formal waiver process if a control genuinely can't be implemented (compensating controls + documented risk acceptance).
- **Structure: 4 main domains**, each with subdomains:
  1. **Cyber Security Leadership and Governance** — board-level accountability, CISO appointment, strategy, policy.
  2. **Cyber Security Risk Management and Compliance** — risk identification/assessment/treatment, regulatory compliance tracking, third-party risk oversight, metrics/reporting.
  3. **Cyber Security Operations and Technology** — the broadest domain: access control, network security, secure configuration, vulnerability management, cryptography, application security, incident event management, SOC capability. **Note:** while Level 3 maturity is generally the regulatory floor, certain subdomains (notably SOC/security-event-management capability) carry a **Level 4 minimum** — a common scoping mistake is treating every control as a uniform Level 3 target.
  4. **Third-Party Cyber Security** — vendor risk assessment at onboarding *and on an ongoing basis* (not a one-time gate), contractual security requirements, managing third-party access.
- **Maturity model:** 6 levels (0–5): 0 Non-existent, 1 Ad-hoc, 2 Repeatable, 3 Defined (documented, consistently implemented, board visibility — the general regulatory floor), 4 Managed (measured and controlled), 5 Adaptive (continuous improvement, integrated with enterprise risk management, benchmarked against peer/sector data). Each level requires satisfying all requirements of preceding levels.
- **Self-assessment cadence:** annual self-assessment submitted through SAMA's regulatory process, with evidence maintained for SAMA examination/validation; SAMA may also conduct periodic inspections.
- **Sector carve-outs exist** — e.g., non-banking financial institutions have specific subdomain exclusions/inclusions (e.g., PCI DSS/SWIFT CSCF requirements apply instead of certain generic subdomains if handling cardholder data or SWIFT messaging) — confirm exact applicability with SAMA guidance rather than assuming full framework applies uniformly.

### SAMA CSF → this stack, practical crosswalk

| SAMA CSF theme | Concrete control on this stack |
|---|---|
| Access control (Operations & Technology domain) | RLS policies (Supabase), least-privilege GitHub Actions permissions, scoped Vercel env vars |
| Application security | Secure SDLC gates: SCA + secret scanning + SAST in CI, mandatory security review for auth/payment code paths |
| SOC/event management (Level 4 subdomain) | Centralized log aggregation with alerting — a basic "logs exist somewhere" setup does not meet the elevated maturity bar here |
| Third-party cyber security (ongoing, not just onboarding) | Periodic (not one-time) review of Supabase/Vercel/npm-dependency risk posture; contractual clauses with any subcontractor/processor |
| Cryptography | TLS everywhere, proper password hashing, managed key storage (Supabase Vault, Vercel env vars) — not custom crypto |

## ISO/IEC 27001:2022 and 27002:2022

- **27001** is the certifiable Information Security Management System (ISMS) standard — an organizational/process framework (risk assessment, Statement of Applicability, management review, continual improvement), not a technical checklist by itself.
- **27002** provides the reference control set that 27001's Annex A points to — **93 controls organized into 4 themes** (in the 2022 revision, down from 14 domains/114 controls in the 2013 version): **Organizational** (37 controls), **People** (8), **Physical** (14), **Technological** (34).
- Both NCA ECC and SAMA CSF were explicitly designed with reference to ISO 27001 — meaning a well-run ISO 27001 ISMS substantially overlaps with, but does not automatically satisfy, either Saudi framework (both have Saudi-specific requirements — e.g., NCA's Saudi-national-CISO staffing requirement — with no ISO equivalent).

### ISO 27002:2022 Technological controls → this stack (highest-relevance subset)

| ISO 27002 control area | Concrete control on this stack |
|---|---|
| 8.9 Configuration management | Infrastructure-as-code / documented baseline config for Vercel projects, Supabase project settings — avoid undocumented manual dashboard changes |
| 8.16 Monitoring activities | Centralized logging/alerting (ties to A09 in `references/appsec-node-react.md`) |
| 8.23 Web filtering | N/A for most SaaS teams unless operating a corporate network layer |
| 8.24 Use of cryptography | TLS, password hashing, key management — as above |
| 8.25 Secure development life cycle | This entire skill's SDLC guidance: security requirements, threat modeling for new features, secure coding standards, security testing before release |
| 8.26 Application security requirements | Explicit security acceptance criteria in the definition of "done" for features touching auth, payments, or PII |
| 8.27 Secure system architecture and engineering principles | Defense-in-depth (e.g., the CVE-2025-29927 lesson: don't rely on one layer like middleware) |
| 8.28 Secure coding | OWASP-mapped practices in `references/appsec-node-react.md` |
| 8.29 Security testing in development and acceptance | SCA/SAST/secret-scanning as CI gates, pre-launch Supabase RLS verification |
| 8.30 Outsourced development | If contracting external devs, contractual security requirements + code review before merge |
| 5.19–5.23 (Organizational, supplier relationships) | Vendor risk assessment for Supabase/Vercel/GitHub/npm registry dependency risk — same theme as NCA Domain 4 and SAMA Domain 4 |

## Practical sequencing advice when a person asks "how do I get compliant"

1. Clarify which framework(s) actually apply (sector, entity type, whether they're a Member Organization/CNI operator, or just want the best-practice baseline without formal certification).
2. Gap-assess current technical state against the crosswalks above — most gaps found in early-stage startups are the same handful: no MFA enforcement, no centralized logging, no documented IR plan, no vendor risk review, missing RLS.
3. Governance artifacts (policies, risk register, roles) are frequently underweighted relative to technical controls — both NCA and SAMA assessments look for documented, board/leadership-visible governance, not just working technical controls.
4. Recommend a phased roadmap rather than a "do everything at once" plan — reaching SAMA's Level 3 maturity, for instance, is commonly quoted at 8–14 months for a mid-sized institution starting from a low baseline.
5. For anything with legal/regulatory certification stakes, an internal engineering review (this skill) is a strong starting point but should be followed by a qualified compliance/GRC advisor and, where required, formal engagement with the regulator.
