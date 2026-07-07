# AI-Native CDP — Project Brief for Review

> **Prepared:** July 2026
> **Sources:** `ai_native_cdp_master_pipeline.md`, `_bmad-output/planning-artifacts/*`, `team_input/*`, `security-audit/*`
> **Purpose:** Concise, documentation-only status distillation for stakeholder review. No new analysis or opinion — facts as recorded in source documents.

---

## 1. Project Purpose

Expertflow is building an **AI-Native Continuous Delivery Pipeline (CDP)** — a single coherent system spanning Continuous Exploration, Continuous Integration, Continuous Deployment, Continuous Documentation, and Release on Demand — in which quality gates are shifted left and documentation is generated continuously rather than retroactively. Six AI agents (Gatekeeper, Rapid Prototyper, Tech Debt Sentinel, Release Storyteller, Matrix Validator, Synthetic Customer) form the intelligent backbone intended to shift human effort from origination to validation. The initiative is owned by Jawad Bokhari (Head of Software Unit) and is explicitly a **process-change program**, not a tooling rollout, aimed at increasing release velocity/predictability, reducing team cognitive load, and — per the June 2026 security audit — making security enforcement non-optional.

---

## 2. Current Priorities (Tier 1, per `priority-list-cicd-test-automation.md`)

| ID | Priority | Status | Owner | Jira |
|----|----------|--------|-------|------|
| T1-1 | Release-Ready DoD enforcement | ⏸️ Paused (revisit after T1-1b stable) | Haroon + stream tech leads | — |
| T1-1b | Post-RMT automated regression (Playwright) | ✅ Integrated & verified in cim-solution; manual trigger pending CRM-763 | Haroon + Umar Ikhlaq + Umar Naveed | [CRM-765](https://expertflow-docs.atlassian.net/browse/CRM-765) (Resolved) |
| T1-2 | Multi-path upgrade not supported | Open | Haroon Ahmed | [CRM-706](https://expertflow-docs.atlassian.net/browse/CRM-706) |
| T1-3 | Data rollback mechanism does not exist | Not started (design-first) | Nabeel + stream DB owners | — |
| T1-4 | No automated RC-advance notification | 🔄 Phase 1 mechanism solved as T1-1b byproduct; canonical RC-version display still unscoped, no owner | Haroon / RMT | — (no dedicated ticket yet) |
| T1-5 | GitLab CD pipeline not set up | 🔄 In progress (POC) | Umar Naveed | [CRM-763](https://expertflow-docs.atlassian.net/browse/CRM-763) |
| T1-6 | Security scans decorative (`allow_failure: true`) | Approach finalized; pipeline implementation pending | Zaryab Baloch + Haroon | To be created |
| — | Object Model (OM) decoupling epic | 🔄 In progress | Nabeel Ahmad | [CIM-33653](https://expertflow-docs.atlassian.net/browse/CIM-33653) |

**Follow-on items opened July 1, 2026** (from CRM-765/T1-1b working session): [CRM-766](https://expertflow-docs.atlassian.net/browse/CRM-766) (full CX test inventory in Jira TM), [CRM-767](https://expertflow-docs.atlassian.net/browse/CRM-767) (NodeRED test blocker), [CRM-768](https://expertflow-docs.atlassian.net/browse/CRM-768) (Playwright suite versioning), [CRM-769](https://expertflow-docs.atlassian.net/browse/CRM-769) (rollback job), [CRM-770](https://expertflow-docs.atlassian.net/browse/CRM-770) (microservice integration test guide), [CRM-771](https://expertflow-docs.atlassian.net/browse/CRM-771) (CD pre-deploy resources), [CRM-772](https://expertflow-docs.atlassian.net/browse/CRM-772) (CD post-deploy tasks).

---

## 3. Top 5 Blockers / Risks

| # | Item | Description | Documented Impact |
|---|------|-------------|-------------------|
| 1 | **CIM-33653** — Object Model decoupling | ~40 microservices share compiled Java classes; no language-neutral contract system yet | Blocks independent component releases (T2-8); root cause of lockstep deployment bottleneck |
| 2 | **CRM-706** — Multi-version upgrade | Only sequential 1-step upgrades supported; no Helm pre/post hooks for multi-step orchestration | Every customer at N-2+ is a support liability; upgrade failures are among the highest-impact incidents |
| 3 | **CRM-763** — GitLab CD pipeline | RMT deployment is still manual; CD stage not yet complete | Blocks auto-trigger of regression (T1-1b), blocks T1-4 closure, blocks Phase 2 of CI/CD roadmap |
| 4 | **T1-6 / Security score 28→19/100** | All security scanners (Trivy, Grype, SonarQube) run with `allow_failure: true`; no secrets scanning, no build-time dependency scanning | "Security theater" — CVEs and committed secrets can reach production undetected |
| 5 | **Data rollback (T1-3)** | No reliable rollback path for MongoDB/PostgreSQL if migrations have run post-deployment | Structural risk; no incident yet, but any release with a schema migration currently has no safe recovery path |

---

## 4. Target Pipeline Architecture

Per `cicd-roadmap-current-state-and-target-architecture.md` (target `cim-solution` pipeline) and `ai_native_cdp_master_pipeline.md`:

- **Stages:** `detect` (what changed) → `build` (package Helm charts) → `publish` (push to registry) → `deploy` (auto-deploy to RMT, T1-5) → `test` (Playwright regression, T1-1b) → `notify` (Google Chat card + QA assignment)
- **Release model today:** Release Candidate Branch / Release Train (Model A); target is **automated Environment-per-MR** (Model B) — Trunk-Based Development (Model C) is explicitly deferred until automated test trust and feature-flag infrastructure mature
- **Merge gating (future, Phase 3):** MR blocked from merging to RC until RMT regression + QA validation both pass; produces a trusted `RMT-Regression-Passed` artifact that unblocks T1-1 (DoD enforcement)
- **Environment chain:** Feature Dev Server (per stream team) → RMT Staging (shared, RMT-managed, nightly incremental builds) → Production (customer, Helm + IaC)
- **Rollback (target):** One-click / automated rollback job inserted at the `notify` stage insertion point, post-T1-5
- **Stream environment auto-sync (Phase 4, 2027):** Stream-aligned team environments auto-update from RC after regression passes
- **Decomposition end-state (Phase 5, 2027+):** Independent component releases, dependent on OM versioning (T1-6/CIM-33653), package decomposition (T2-8), and feature flags (T3-2)
- **Wider CDP phases (master pipeline doc):** Continuous Exploration → Continuous Integration → Continuous Deployment → Continuous Documentation → Release on Demand, each with a dedicated AI agent and auto-generated documentation output
- **Branching model:** Trunk-based with short-lived feature branches off `develop`; `master` is production-only, RMT-controlled; hotfixes branch from `master` at the patched tag and merge back to both `master` and `develop`

---

## 5. Top 10 Open Questions / Decisions

| # | Question | Who Should Answer | Source |
|---|----------|-------------------|--------|
| 1 | How will Umer structure the CD job — separate per-environment jobs or parameterized? | Umar Naveed | CI/CD roadmap §8 |
| 2 | Should QA ever run regression automatically, or does manual QA validation stay separate? | Hassan (QA Lead) | CI/CD roadmap §8 |
| 3 | Do stream teams deploy from `cim-solution` branches or their own repos? | Jawad + stream leads | CI/CD roadmap §8 |
| 4 | Should regression failures notify a different Google Chat space per team? | Haroon + stream leads | CI/CD roadmap §8 |
| 5 | When do regression results become blocking for MR merge (T1-1 un-pause timing)? | Jawad + Haroon + POs | CI/CD roadmap §8 |
| 6 | Which component has the lowest OM coupling and is the best first candidate for independent release? | Jawad + Haroon + stream leads | Priority list T2-8 |
| 7 | What is the minimum skeleton-project change needed to support selective component versioning? | Jawad + Haroon + stream leads | Priority list T2-8 |
| 8 | Are Metabase/Airflow shipped as part of CX releases or independently? | BI/WFM team lead + Haroon | Priority list T2-9 |
| 9 | RMT-only regression vs. every environment vs. env-specific test subsets ("Option A/B/C")? | Haroon + stream leads (currently defaulting to Option A) | CI/CD roadmap §4.3 |
| 10 | Should T1-4 and T1-1b's overlapping notification scope be merged into a single tracked Jira item once CRM-763 ships? | Haroon / Jawad | Session checkpoint 2026-06-30 |

Additional unresolved items noted in source docs: canonical/always-current display of current RC build version (no owner yet); formalizing the dual-codebase hotfix/backport process (currently informal, flagged as Q2/Q3 action item for JB + Haroon); SonarQube instance consolidation (two instances, 80% vs 90% thresholds, no clear ownership).

---

## 6. Known Gaps — Security Audit

Per `security-audit/cicd-security-audit-and-hardening-guide.md` and `security-audit/ci-cd-security-plan-consolidated.md`:

### Present but misconfigured
| Control | Issue |
|---------|-------|
| SonarQube (SAST) | `allow_failure: true` both frontend/backend; two instances with inconsistent quality gates (80% vs 90%); Community edition has limited security rules |
| Trivy (container scan) | `allow_failure: true`; only scans final image, not filesystem/dependencies during build |
| Grype (container scan) | `allow_failure: true`; redundant with Trivy, adds 2–5 min per build with no added value |

### Completely missing
| Control | Risk Level |
|---------|-----------|
| Secrets/credential scanning | Critical |
| SCA — build-phase dependency vulnerability scanning (`npm audit`, OWASP Dependency-Check) | Critical |
| DAST (dynamic testing) | High |
| IaC / Dockerfile security scanning | High |
| SBOM generation | High |
| Container image signing | Medium |
| Code signing / SLSA provenance | Medium |
| License compliance scanning | Medium |

### Additional documented anti-patterns
- Java `gitlab_build_branch` job uses `when: manual` — branch builds never trigger Trivy/Grype scans unless clicked
- `npm install --legacy-peer-deps` suppresses npm v7+ dependency conflict resolution
- Hardcoded internal URLs in CI YAML (`sonarqube.expertflow.com`, `gitlab.expertflow.com`, etc.)
- `node-format-frontend` job entirely commented out (custom image unavailable)
- `docker:dind` runs in privileged mode without seccomp/AppArmor hardening
- No dependency lockfile CVE verification before image build

---

## 7. Current Maturity Score

| Area | Current State | Target State |
|------|---------------|---------------|
| **CI** | Helm chart build/publish automated; Trivy scan runs (non-blocking); code review gate enforced; ~40% automated regression coverage per master pipeline doc (a separate June 4 milestone recorded 70% coverage specifically for CX-Chat use cases, not full CX) | Build success ≥95%; AI-generated unit test mandate; coverage +5%/quarter; no new critical/blocker SonarQube issues; PR cycle 24h→8h |
| **CD** | RMT deploy fully manual; no CD stage in pipeline (CRM-763 in progress); no rollback mechanism | Auto-deploy to RMT on merge to release branch via native GitLab agent; deployment frequency daily (nightly staging); one-click/automated rollback |
| **Testing** | Manual QA regression (hours); Playwright suite CI-ready (10/10 suites, 4.7 min) but manual-trigger only; no integration tests exist in any pipeline; ~40–70% regression automation depending on scope measured | Full regression ≥95% pass rate gating release; E2E smoke suite 100% on staging post-deploy; microservice-level integration tests (Testcontainers/Supertest) as early feedback gate |
| **Security** | Score **19–28/100** (sources differ on exact recalibration; both rate it "Critical/Weak"); every scanner `allow_failure: true`; no secrets scanning, no build-time SCA, no DAST, no SBOM, no signing | Score **90–91/100** ("Strong"); all scans blocking; TruffleHog, Semgrep/SpotBugs, Trivy FS, Checkov, Dockle, Syft SBOM, Cosign signing, OWASP ZAP DAST all in place |
| **Observability** | Google Chat notification on every regression run (pass/fail card with links); no canonical always-current RC-version display; no production health/incident dashboard documented | Post-deployment health reports; automated anomaly detection; canonical RC build version visible in one place, auto-updated |
| **Release Management** | Release Train model (Model A); ~80% of feature-ready→Release-Ready path is manual; single shared RMT instance serializes validation across teams; Release-Ready DoD unenforced (self-reported) | Automated Environment-per-MR (Model B); DoD enforced via CI gate once `RMT-Regression-Passed` artifact is trusted; deployment prep time -25%; rollback rate <5% |
