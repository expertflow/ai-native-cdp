# Priority List: CI/CD, Test Automation & Delivery Process

> **Status:** Active — Security-Consolidated Edition (Round 3 — June 2026)  
> **Produced by:** Problem Inventory analysis + stakeholder corrections + security audit consolidation (June 2026)  
> **Sources:** `problem-inventory-cicd-test-automation.md`, `security-audit/problem-inventory-cicd-security.md`, `security-audit/ci-cd-security-plan-consolidated.md`, `docs/cicd_objectives_gaps.md`, `team_input/Road to CD.md`, Confluence retro pages, direct stakeholder input (Haroon — RMT, Junaid — RMT, Zaryab Baloch — Security Lead/PO)  
> **Last updated:** 2026-06-09  
> **Note:** This is a consolidated copy that integrates security audit findings into the master priority list. Original planning artifacts are preserved unchanged.  

---

## Strategic Themes

Two goals run through all tiers:

1. **Increase release velocity and predictability** — reduce the time and uncertainty between a feature being "dev complete" and it being in production.
2. **Reduce cognitive load on stream-aligned teams** — teams should be able to build and ship with confidence, without needing tribal knowledge, manual coordination, or waiting on other humans to unblock them.

A third goal has been added in this consolidated edition:

3. **Security is not optional** — every pipeline stage that scans for vulnerabilities must either pass or block. A green pipeline with failing security scans is a false promise.

These themes are not in conflict. Most of the high-priority items serve all three simultaneously.

---

## 🔴 Tier 1 — Critical (Address Now)

These are root causes or active risks that, if left unaddressed, prevent meaningful improvement in everything below them.

---

### T1-1 · Release-Ready Definition of Done Is Not Enforced
**Cluster:** A2

**Problem:** The Release-Ready DoD exists on paper. Teams self-report readiness. No CI gate or objective checkpoint makes it verifiable. Tech leads are the first escalation point — reactive, not preventive.

**Why it's Tier 1:** This is the single root cause behind the largest cluster of compounding problems: defects escaping to RMT, rework loops, unpredictable release timelines, and the need for full regression at the end of the cycle. Fixing this is the highest-leverage intervention in the entire inventory.

**What good looks like:**
- DoD criteria are objective and automatable (CI green, upgrade script present, deployment guide validated, no open blockers in Jira)
- The pipeline blocks an MR or a release-ready tag without those criteria being met
- Tech leads validate intent; the pipeline validates fact

**Cognitive load angle:** A clear, enforced DoD removes ambiguity for developers. They know exactly when they're done. No more "are we ready?" conversations — the system answers.

**DRI suggestion:** Haroon + stream tech leads  
**Effort:** Medium — process design + CI gate implementation

---

### T1-2 · Multi-Path Upgrade Not Supported
**Cluster:** C6, E6  
**Jira Epic:** [CRM-706 — Multi-version Upgrade](https://expertflow-docs.atlassian.net/browse/CRM-706) · Assigned: Haroon Ahmed · Status: Open

**Problem:** Only one-step sequential upgrades are supported (CX4.10 → 4.10.1 → 4.10.2 … → 4.10.5). Customers cannot jump versions safely. The epic covers ConfigMaps externalization, DB schema migrations across MongoDB/PostgreSQL/Reporting, environment variable management, secrets rotation, and a move to tenant-specific versioned configs in S3/DB. Helm pre/post hooks for multi-step upgrade orchestration are not yet implemented.

**Why it's Tier 1:** Every customer at N-2 or behind is a support liability on every upgrade. Upgrade failures at customer sites are among the highest-impact incidents.

**What good looks like:**
- Helm lifecycle hooks execute intermediate migration steps automatically when upgrading across versions
- Upgrade paths are tested in CI (upgrade N → N+2 in staging)
- Intermediate steps are idempotent — a failed mid-upgrade can be safely retried

**DRI:** Haroon Ahmed  
**Effort:** High — work is scoped in CRM-706; execution is the gap

---

### T1-3 · Data Rollback Mechanism Does Not Exist
**Cluster:** E5

**Problem:** MongoDB and PostgreSQL have no reliable rollback path. DB backups exist but restoring them loses transactions between backup and failure. Any release with a schema migration carries production risk with no safe recovery.

**Why it's Tier 1:** Not yet experienced as a major incident — but the risk is structural and the consequence of a first failure would be severe. The longer this goes unaddressed, the more migration history accumulates without a rollback story.

**What good looks like:**
- DB migrations are versioned and reversible (down migrations exist alongside up migrations)
- Pre-migration snapshots are automated as part of the release deployment hook
- A documented, tested rollback runbook exists and is exercised at least once per quarter

**Note:** This is an engineering design problem first, a tooling problem second. The first step is a design session on what "rollback" means per database and per migration type.

**DRI suggestion:** Nabeel + stream DB owners  
**Effort:** High — design-first, then incremental implementation per service

---

### T1-4 · No Automated Notification When the Incremental RC Build Advances

**Cluster:** B8

**Vocabulary note:** At Expertflow, the release lifecycle has three distinct stages: the **feature branch** (developer's working branch), the **Release Candidate (RC)** (the build under test at RMT), and the **production release** (shipped, tagged version). The RC is updated incrementally — each time RMT integrates a new release-ready feature, a new **incremental RC build** is produced. The term "pre-release" is retired; it was ambiguous.

**Problem:** When RMT advances the RC (integrates a new feature, deploys a new incremental build), stream-aligned teams are not automatically notified. Teams testing on RMT don't know the build has changed beneath them, leading to stale test results and wasted effort.

**Why it's Tier 1 (cognitive load):** Pure friction with a low-effort fix. The impact is felt on every incremental build cycle — which is every active release.

**What good looks like:**

- When RMT cuts a new incremental RC build, an automated notification fires to all stream teams (Slack + Jira comment or Confluence page update)
- The current RC build version is always visible in one canonical place, updated by CI automatically

**Phase 1 — Notification (current scope):**
Notification fires automatically when a new incremental RC build is deployed. Stream teams are informed and can manually update their environments.

**DRI:** Haroon / RMT  
**Effort:** Low — CI step + Slack webhook

**Phase 2 — Auto-update stream environments (follow-on after T1-5):**
Once the CD pipeline (T1-5 / CRM-763) is stable, stream-aligned team environments are automatically updated with the latest RC build following the notification — no manual action required from stream teams.

**DRI (Phase 2):** Haroon  
**Effort:** Low–Medium — depends on T1-5 CD pipeline being in place first; primarily configuration + pipeline extension

---

### T1-5 · GitLab CD Pipeline Not Set Up
**Jira:** [CRM-763 — Setup GitLab CD (Continuous Deployment) pipeline](https://expertflow-docs.atlassian.net/browse/CRM-763) · Assigned: Umar Naveed · Status: In Progress (running in parallel)

**Problem:** RMT currently uses GitLab CI to build and push container images, but deployment to the on-prem Kubernetes cluster is done manually. Every deployment requires a person to execute it, introducing delays and human error. There is no CD stage in the pipeline. CXVOICE and CXBI (reports, reporting server, data pipelines) are also not properly covered under CI/CD and remain a separate gap.

**Why it's Tier 1:** This is the foundational piece that unlocks the delivery improvement programme. Without automated CD, the entire incremental RC build cycle stays manual and error-prone. It runs in parallel with other Tier 1 items rather than blocking them.

**Decided approach:** Native GitLab agent (not ArgoCD or push-based deployment).

**Target workflow (scope of CRM-763):**
1. Stream-aligned team develops and tests a feature on their own team server
2. On merge to the release branch, the CD script automatically upgrades RMT's incremental build with that update — no manual deployment request

**Out of scope for CRM-763 — follow-on item (DRI: Haroon):**
Once CRM-763 is complete, the next step is: stream-aligned team environments are automatically updated with the latest RC build following a notification. This will be tracked separately.

**What good looks like:**
- GitLab CI pipeline extended with a CD stage using the native GitLab agent that deploys to the on-prem K8s cluster via Helm automatically
- Merge to release branch is the trigger — no manual deployment request required
- No manual deployment steps required; a stream team's release-ready feature triggers the RMT incremental build upgrade end-to-end

**Known gap — rollback not defined:** There is currently no rollback process for an incremental RC build upgrade that fails or causes regression. This is the next item to address once the CD pipeline is stable (candidate for T1-3 scope or a new item).

**Next actions:**
- Complete POC and document the native GitLab agent deployment model
- Align with T1-4 so the notification fires automatically from the CD pipeline post-deployment
- Define rollback procedure for failed incremental upgrades

**DRI:** Umar Naveed  
**Effort:** Medium — POC in progress; full workflow implementation follows

---

### T1-6 · Security Scans Are Decorative — `allow_failure: true` on Every Gate
**Cluster:** F1, F2, F3, F5 (consolidated security audit finding)  
**Jira:** *To be created — "Enforce Security Scans in CI/CD"*

**Problem:** Every vulnerability scanner in both Node.js and Java pipelines — Trivy (branch + merge), Grype (branch + merge), and SonarQube (frontend + backend) — runs with `allow_failure: true`. A CRITICAL CVE produces a red report that no one reads, and the image is still pushed to the registry and deployed. The SonarQube quality gate has two instances (local lab at 80%, live at 90%) with no clear ownership, and neither blocks the pipeline because `allow_failure: true` overrides it. This is the single highest-leverage security intervention in the entire pipeline.

**Why it's Tier 1:** Security scans provide a false sense of safety. The pipeline shows green while untested, vulnerable code reaches production. This undermines every other security control — Trivy, Grype, SonarQube, and any future scanners — because none of them can actually stop a bad build. It is also the prerequisite for *every* other security item: there is no point adding Checkov, TruffleHog, or OWASP ZAP if they too will be allowed to fail.

**What good looks like:**
- `allow_failure: false` on all security scan jobs: Trivy, Grype, SonarQube (both instances), and any future scanners
- A graduated enforcement policy: CRITICAL and HIGH CVEs block immediately; MEDIUM and LOW are warnings with a 30-day remediation SLA
- SonarQube instance consolidation: one canonical instance with 90% threshold, or explicit rules on when each applies
- A documented exception process: if a CVE has no fix and risk is accepted, the exception is recorded in Jira with Security Lead approval

**Risk of implementation:** Changing `allow_failure: true` → `false` will break existing pipelines until current CVEs are triaged and remediated. This must be done in a controlled rollout:
1. **Week 1:** Audit current scan results and create a CVE backlog (Trivy + Grype + SonarQube)
2. **Week 2:** Fix or triage all CRITICAL and HIGH findings (estimated 20-40 items based on current Trivy output)
3. **Week 3:** Enable `allow_failure: false` on all scan jobs
4. **Week 4:** Monitor and adjust thresholds

**Cognitive load angle:** Developers need to know that when CI is green, the build is genuinely safe — not just "scanned but ignored." This removes the ambiguity of "should I care about this Trivy report?"

**DRI:** Zaryab Baloch (Security Lead/PO) + Haroon (DevOps)  
**Effort:** Low (configuration change) — Medium (CVE triage and remediation)  
**Prerequisite for:** T2-4, T2-7, T2-10, T2-11, T2-12, T2-13, T3-11, T3-12, T3-13, T4-1, T4-6, T4-7

---

## 🟠 Tier 2 — High Urgency (Next 1–2 Quarters)

These have significant compounding cost if delayed but are not immediately blocking releases today.

---

### T2-1 · IaC and CD Gaps: CXVOICE, CXCHAN Not Enabled, CXBI Excluded
**Cluster:** E1, E2, E3, G5

**Problem:** CXVOICE has no IaC — every environment is a manual snowflake. CXCHAN is not enabled on the IaC and code review process. CXBI/WFM are excluded from IaC and release planning entirely. Several engineers (Bilal Alam, Ehtasham, WFM/BI team) are not trained on IaC deployment.

**Why it's Tier 2:** Until all components are deployable via IaC, automated testing and reliable release packaging cannot cover the full stack. Voice is the highest-risk gap — it's also the component hardest to reproduce in the lab.

**Phased approach:**
1. CXCHAN enablement (lower effort — code review process already exists for other teams)
2. CXBI/WFM onboarding to release planning and IaC
3. CXVOICE IaC and CD — full treatment required (see sub-items below)

**CXVOICE sub-items (to be addressed as part of phase 3):**
- Structural definition of CXVOICE components and how they are deployed (which services, which Helm charts, which namespaces)
- Release management mechanism per CXVOICE component (versioning, tagging, upgrade path)
- Configuration management — environment-specific configs, secrets, and ConfigMaps for voice components
- Automated testing for CXVOICE in the CD pipeline (smoke tests at minimum post-deploy)

**DRI suggestion:** Haroon (IaC) + respective stream tech leads  
**Effort:** Medium–High depending on component

---

### T2-2 · Deployment and Upgrade Guides Incomplete and Untested
**Cluster:** H4

**Problem:** Deployment, upgrade, TLS, and auth guides are handed to RMT without being validated end-to-end. Guides are outdated or missing steps. This has caused documented rework and delays in multiple releases (CX4.10, CX4.10.1).

**Why it's Tier 2:** Directly impacts RMT velocity on every release. Every release with an invalid guide is a support incident waiting to happen.

**What good looks like:**
- The Release-Ready DoD (T1-1) includes: "Deployment guide has been tested against a fresh environment by the feature team"
- An uninstallation guide exists and is maintained
- A guide validation step is part of stream-team testing, not RMT discovery

**DRI suggestion:** Stream tech leads (per component)  
**Effort:** Medium — process change + one-time guide audit

---

### T2-3 · Integration Testing: Define Scope and Produce a Guide
**Cluster:** D3

**Problem:** No integration tests are conducted today. No integration testing guide exists for Java or Node services. Contract boundaries between microservices are unverified until RMT.

**Why it's Tier 2 (not Tier 1):** The 70% regression automation already catches some of what integration tests would catch at the later stage. However, the retro evidence is clear that the class of defects escaping to RMT — Keycloak permission mismatches, async behavior issues, cross-component interference — is exactly what integration tests are designed to prevent.

**Recommended first step:** Nabeel produces the integration testing guide. This is a scoping exercise (which service boundaries are highest risk? what tooling fits — Testcontainers, Pact, Supertest?) before any implementation commitment.

**DRI suggestion:** Nabeel Ahmad  
**Effort:** Low to start (guide) — Medium to implement

---

### T2-4 · Vulnerability Management for Supported Releases
**Cluster:** F4

**Problem:** Trivy scans run for the current release in flight. Customers on older supported versions (e.g. CX5.0.x while CX5.1 is current) have no security posture visibility. No support window definition exists. No SLA for CVE fixes in older releases. Zaryab Baloch's action item is open.

**Why it's Tier 2:** Customer trust issue with enterprise clients. Sales blocker for regulated industries. No immediate incident today, but the exposure grows with each release.

**What good looks like:**
- A defined support window: e.g., N and N-1 are supported; N-2 receives critical CVE patches only
- Scheduled Trivy scans for every supported release branch, published to a security dashboard
- An SLA for CVE remediation: CRITICAL ≤ 7 days, HIGH ≤ 30 days, MEDIUM ≤ 90 days
- Enterprise customers (FNB, Cisco) can request an SBOM and current vulnerability report on demand

**DRI suggestion:** Zaryab Baloch + RMT  
**Effort:** Medium — scheduled pipeline + support window policy decision
**Depends on:** T1-6 (enforcing scans)

---

### T2-5 · Release Target Changes Mid-Development
**Cluster:** C8

**Problem:** Features are started against one release target and redirected to another after development has begun. Causes rework, branch management confusion, and unplanned sprint disruption.

**Why it's Tier 2:** Primarily a planning governance problem. The fix requires a decision protocol — not tooling — but the cognitive load on teams is high.

**What good looks like:**
- A release target lock date exists in every sprint; changes after lock require explicit approval and impact assessment
- The process for communicating target changes to affected teams is documented and followed

**DRI suggestion:** Jawad + POs  
**Effort:** Low — governance decision + process documentation

---

### T2-6 · Load Testing Not Wired into Release Pipeline
**Cluster:** D7

**Problem:** A load test environment exists on AWS. Performance baselines, pass/fail thresholds, and a cadence for running load tests against releases are not defined. Load testing does not happen on every release today.

**Why it's Tier 2:** The infrastructure investment is done. The remaining gap is defining when it runs, what it measures, and who owns the results. This is a governance + configuration task, not a platform build.

**What good looks like:**
- Load test suite runs on every RC (or at minimum every minor release)
- Baseline thresholds defined for: concurrent agents, concurrent conversations, API response times under load
- Results are published to Confluence automatically; regressions block promotion

**DRI suggestion:** Umar / Hassan (QA) + RMT  
**Effort:** Low–Medium — pipeline wiring + threshold definition

---

### T2-7 · VAPT and Security Scanning Automation
**Cluster:** F1, F3, F5

**Problem:** VAPT is not automated or scheduled. OWASP, Burp Suite, and Nmap scans are manual at release time. Automated dependency updates (Renovate or equivalent) are not implemented — Trivy alert volume is inflated by fixable dependency debt. The security scan ecosystem is fragmented: Trivy and Grype do the same job, secret scanning is absent, and build-time dependency checks are missing.

**Why it's Tier 2:** Security feedback is currently end-of-cycle. Earlier detection is cheaper. Automated dependency updates directly reduce Trivy noise. The audit found 6 gaps beyond what EF's plan addressed: Trivy FS scanning, Checkov IaC scanning, SBOM generation, image signing, DAST, and SLSA provenance.

**Phased approach:**
1. **Phase 2A (Weeks 2–3):** TruffleHog secret scanning, `npm audit` + OWASP Dependency-Check build-time scanning, Dockerfile scanning (Hadolint/Dockle), IaC scanning (Checkov), eliminate redundant Grype
2. **Phase 2B (Weeks 4–6):** Semgrep SAST (Node + Java), SpotBugs/FindSecBugs (Java), docker:dind hardening with seccomp, base image SHA pinning, Syft SBOM generation, detect-secrets pre-commit hook
3. **Phase 2C (Weeks 7–8):** Renovate/Snyk automated dependency updates, VAPT scheduling and reporting

**DRI suggestion:** Zaryab Baloch (Security Lead/PO) + Haroon (DevOps) + stream leads  
**Effort:** Medium  
**Depends on:** T1-6 (enforcing scans)

---

### T2-8 · CX Decomposition into Smaller Independently Releasable Packages
**Cluster:** G1, G2 (was T4-1 — elevated)

**Problem:** CX ships as a monolithic bundle. Stream-aligned teams cannot independently deploy, validate, or release their own services — every change must pass through the full monolithic integration cycle, inflating lead time, blast radius, and risk. CXAGENT, CXCHAN, CXVOICE, CXBI, and CXPLAT are logical package boundaries but currently ship together. The primary known blocker is the **Object Model (OM)** dependency, which requires manual updates across all components and prevents independent versioning.

**Why it's Tier 2 (elevated from T4):** This is the strategic capability that gives stream teams genuine release autonomy — shorter feedback loops, targeted testing per package, and reduced upgrade risk for customers. OM version management has been elevated to Tier 1 (see T1-2 / CRM-706 scope); once OM is automated, decomposition becomes actionable.

**Prerequisite:** OM versioning automation must be resolved first (part of T1-2 scope). Decomposition design can begin in parallel.

**What good looks like:**

- Natural package boundaries are defined and agreed (CXAGENT, CXCHAN, CXVOICE, CXBI, CXPLAT as candidates)
- The skeleton project supports selective component versioning
- At least one package (lowest OM coupling) can be released independently with its own CI/CD pipeline
- Hard interdependencies are documented; a roadmap exists to eliminate or automate each one

**Questions to resolve:**

- Which component has the lowest OM coupling and is the best first candidate for independent release?
- What is the minimum change to the skeleton project to support selective component versioning?
- Which components have hard interdependencies beyond OM that prevent independent release?

**DRI:** Jawad + Haroon + stream leads  
**Effort:** High — design-first (Q3), phased implementation per package

---

### T2-9 · Reporting and Data Platform (Metabase + Airflow) Not Packaged or Released
**Cluster:** G5 (elevated from T4-6)

**Problem:** Metabase (reporting) and Airflow (data pipelines) have no defined packaging, release, or deployment process. There is no IaC, no CD pipeline, and no release management mechanism for these components. Every deployment is ad-hoc. These are not currently covered under CI/CD at all.

**Why elevated to Tier 2:** Raised as an active concern in the June 1 2026 meeting. These components are customer-facing and business-critical — the absence of a release process creates unpredictable delivery and support risk. Waiting on T2-1 (CXBI onboarding) is no longer sufficient justification to defer.

**What good looks like:**
- Metabase and Airflow are packaged as versioned, deployable artifacts (Helm charts or equivalent)
- A defined release management mechanism exists per component (versioning, tagging, upgrade path)
- Data pipelines (Airflow DAGs) are version-controlled and deployed via CD, not manually
- Components are included in IaC and release planning alongside the rest of the CX stack

**First step:** Assign an owner to define packaging scope and answer: are these shipped as part of CX releases, or independently? That decision shapes everything else.

**DRI suggestion:** BI/WFM team lead + Haroon  
**Effort:** Medium — design and scoping first, then implementation

---

### T2-10 · Build-Time Dependency Scanning (npm audit + OWASP Dependency-Check)
**Cluster:** F3 (security audit finding)

**Problem:** `npm audit` never runs in the Node.js pipeline. The `node_artifacts_frontend` job uses `npm install --legacy-peer-deps`, which suppresses npm v7+ dependency conflict resolution. A vulnerable transitive dependency can slip in without any warning. The Java pipeline has no OWASP Dependency-Check or equivalent. Container scanning (Trivy/Grype) only sees the final image — dev dependencies and build tools are invisible.

**Why it's Tier 2:** Build-time scanning catches vulnerabilities in dependencies that never make it into the container image (webpack, babel, test frameworks, build plugins). These are attack surfaces for supply chain compromise. OWASP Dependency-Check is the industry standard for Java; `npm audit` is built into npm and costs nothing to run.

**What good looks like:**
- `npm audit` runs in every Node.js build, fails on CRITICAL and HIGH findings (configurable)
- OWASP Dependency-Check Maven plugin runs in every Java build, fails on CVSS ≥ 7.0
- Both tools are configured with `allow_failure: false` (enforced by T1-6)
- Suppression files are maintained for known false positives, reviewed quarterly

**DRI suggestion:** Node lead + Java lead + Haroon (DevOps)  
**Effort:** Low — plugin configuration per pipeline  
**Depends on:** T1-6 (enforcing scans)

---

### T2-11 · Secret Scanning (TruffleHog) and Pre-Commit Prevention
**Cluster:** F1, F2 (security audit finding)

**Problem:** Neither pipeline runs any secret detection. CI variables like `SONAR_PASS`, `SONAR_PASS_LIVE`, and `CI_PIPELINE_IID_TOKEN` are referenced in YAML. If any developer accidentally hardcodes these values, they would be committed and pushed with zero detection. No pre-commit hooks exist to catch secrets before they reach CI.

**Why it's Tier 2:** Committed secrets are the #1 cause of cloud breaches. TruffleHog scans the full Git history (including MRs) for secrets. A pre-commit hook (detect-secrets) stops secrets at the developer's machine before they reach Git. The combination provides defense in depth.

**What good looks like:**
- TruffleHog runs on every MR, scanning the entire branch history (not just the diff), with `allow_failure: false`
- detect-secrets pre-commit hook is installed repo-wide via `.pre-commit-config.yaml`
- A secrets response playbook exists: rotation procedure, incident notification, forensics steps
- Quarterly secret scan audits of the full repository history

**DRI suggestion:** Zaryab Baloch (Security Lead/PO) + Haroon (DevOps)  
**Effort:** Low — TruffleHog is a single CI job; pre-commit hook is repo configuration  
**Depends on:** T1-6 (enforcing scans)

---

### T2-12 · Dockerfile and Container Security Scanning
**Cluster:** F5 (security audit finding)

**Problem:** No tool validates Dockerfiles for security misconfigurations before build. Common issues like running as root, missing `.dockerignore`, using `ADD` instead of `COPY`, or missing `HEALTHCHECK` go undetected. Docker base images use mutable tags instead of SHA digests. `docker:dind` runs in privileged mode without seccomp or AppArmor profiles.

**Why it's Tier 2:** Misconfigured Dockerfiles are direct attack vectors. A `USER root` directive or an overly permissive `chmod` introduces vulnerabilities in every image. SHA digest pinning prevents tag-tampering attacks. docker:dind hardening prevents container escape.

**What good looks like:**
- Hadolint or Dockle runs on every Dockerfile change, fails on HIGH severity issues
- All `FROM` statements pin to SHA digest (not tag): `FROM node:18.14.1-alpine3.17@sha256:...`
- `docker:dind` service uses `security_opt` with custom seccomp profile and resource limits
- `USER` directive is present in every Dockerfile; no image runs as root by default

**DRI suggestion:** Haroon (DevOps) + stream leads  
**Effort:** Low–Medium — Dockerfile updates + CI job configuration  
**Depends on:** T1-6 (enforcing scans)

---

### T2-13 · Infrastructure-as-Code Security Scanning (Checkov)
**Cluster:** F5 (security audit finding)

**Problem:** Expertflow deploys to Kubernetes via Helm charts, but no security scanning exists for K8s manifests, Helm charts, or Terraform. Misconfigurations like privileged containers, missing resource limits, secrets in ConfigMaps, or excessive RBAC permissions go undetected.

**Why it's Tier 2:** A misconfigured Helm chart can deploy a container with `privileged: true`, mount the host filesystem, or expose a service without network policies. These are deployment-time vulnerabilities invisible to code scanning. Checkov is the industry standard for IaC security.

**What good looks like:**
- Checkov runs on every Helm chart and Terraform change, fails on CRITICAL and HIGH findings
- A custom Checkov policy baseline is defined for Expertflow's K8s environment
- Suppression file maintained for known false positives, reviewed quarterly
- Results published to a security dashboard alongside Trivy and SonarQube

**DRI suggestion:** Haroon (DevOps) + Zaryab Baloch (Security Lead/PO)  
**Effort:** Low–Medium — tool configuration + policy baseline definition  
**Depends on:** T1-6 (enforcing scans)

---

## 🟡 Tier 3 — Plan This Quarter (Meaningful ROI, Not Immediately Blocking)

| # | Challenge | Why Tier 3 | DRI |
|---|-----------|------------|-----|
| T3-1 | Regression coverage lift from 70% to 85%+ | 70% milestone demonstrated June 4 2026 (Playwright — Umar Ikhlaq). Next: define objectives and milestones for path to 85%+ | Umar Ikhlaq (Head of QA) |
| T3-2 | Feature flag framework production-ready | Enables TBD and safe merging of incomplete work; needs TBD adoption enforced first | Awais / DevOps |
| T3-3 | Release versioning policy enforcement | Process discipline before tooling; quick governance fix | Haroon + stream leads |
| T3-4 | IaC for pre-release versions (stream self-serve) | Reduces dependency on RMT for env setup; high cognitive load reduction | RMT |
| T3-5 | Three excluded teams brought into release planning (CXCHAN, CXBI, QM) | Late-discovered scope issues on every release; fix is inclusion, not tooling | Jawad |
| T3-6 | Jira usage consistency enforcement | Visibility problem; blocks accurate release planning | Stream tech leads |
| T3-7 | Trunk-Based Development adoption verified across all teams | TBD doc exists; actual adherence across non-Core teams is unclear | Awais + stream leads |
| T3-8 | Teams outside Core brought to same dev/code-review standards | Integration surprises traced to non-Core teams not following same practices | Haroon + stream leads |
| T3-9 | Formal rollback procedure documented | Currently ad-hoc via Google Chat; tribal knowledge | RMT |
| T3-10 | Hotfix and dual-codebase patch process formalized | Two related gaps: (1) hotfix must be release-ready for the latest production release; (2) stream-aligned teams are not consistently applying hotfix changes back to the latest develop branch — a gate is needed to enforce this. Reactive and unplanned today; needs a trigger protocol, gate, and effort accounting | Jawad + Haroon + POs |
| T3-11 | **Test runner coverage thresholds enforced (jest + Jacoco)** | SonarQube enforces 80%/90% but is bypassed by `allow_failure: true`. Test runner thresholds (jest `coverageThreshold`, Jacoco `minimum`) are the tripwire that catches coverage drops at build time, before SonarQube. Without them, a developer can delete all tests and the pipeline passes. | Node lead + Java lead |
| T3-12 | **Java branch build auto-triggered (`when: manual` removed)** | Java `gitlab_build_branch` uses `when: manual`, meaning branch images are never scanned unless manually triggered. If someone deploys a branch image directly, it bypasses all security scanning. This was an intentional cost-saving measure that creates a security gap. | Java lead + Haroon |
| T3-13 | **Code format job re-enabled in Node pipeline** | `node-format-frontend` is entirely commented out. Inconsistent formatting increases cognitive load during code review and makes security-relevant code harder to spot. | Node lead |

### T3-11 · Test Runner Coverage Thresholds (jest + Jacoco)
**Cluster:** D1, D2 (security audit finding)

**Problem:** `npm run coverage --u` and `mvn verify` run tests with coverage reporting but no minimum threshold. A developer can delete all tests and the pipeline still passes. SonarQube has 80%/90% thresholds but is unenforced due to `allow_failure: true`.

**Why it's Tier 3:** Coverage thresholds are a quality gate, not an immediate security risk. But untested code paths are unvalidated attack surfaces — injection points, authentication bypasses, and business logic errors often live in paths with no test coverage. This is a hygiene item that prevents regression.

**What good looks like:**
- jest `coverageThreshold` set to 70% global minimum (aligns with current SonarQube baseline)
- Jacoco `minimum` configured in `pom.xml` for line and branch coverage
- Both thresholds are enforced with `allow_failure: false` (via T1-6)
- Thresholds increase incrementally: 70% → 75% → 80% per quarter

**DRI suggestion:** Node lead + Java lead  
**Effort:** Low — configuration changes in `jest.config.js` and `pom.xml`  
**Depends on:** T1-6 (enforcing scans)

---

### T3-12 · Java Branch Build Auto-Triggered (`when: manual` Removed)
**Cluster:** F3 (security audit finding)

**Problem:** The Java pipeline's `gitlab_build_branch` job has `when: manual`. Branch builds never trigger Trivy/Grype scans unless manually clicked. A developer can push to a feature branch and merge via MR, but the branch image itself is never scanned.

**Why it's Tier 3:** The merge request path still runs scans. The risk is only if someone deploys the branch image directly (hot-fixes, ad-hoc testing, air-gapped environments). This is a lower-probability path but a complete security bypass when it happens.

**What good looks like:**
- `when: manual` removed from `gitlab_build_branch`; branch builds auto-trigger on push
- Branch scans use the same `allow_failure: false` policy as merge scans (via T1-6)
- If CI cost is a concern, branch scans run a lighter check (Trivy FS only, not image scan)

**DRI suggestion:** Java lead + Haroon  
**Effort:** Low — remove `when: manual`  
**Depends on:** T1-6 (enforcing scans)

---

### T3-13 · Code Format Job Re-Enabled in Node Pipeline
**Cluster:** H2 (security audit finding)

**Problem:** The `node-format-frontend` job is entirely commented out. It was disabled because the custom image (`gitlab.expertflow.com:9242/general/node:CSN-3623`) became unavailable. Without format enforcement, code style drift accumulates.

**Why it's Tier 3:** Formatting is not a security control. But inconsistent formatting increases cognitive load during code review. Security-relevant code (auth checks, input validation) is harder to spot and verify when buried in formatting noise. The absence of this gate also signals that quality enforcement is optional.

**What good looks like:**
- Format job re-enabled using a standard Node image (not a custom one)
- Prettier or ESLint `fix` runs in CI, fails on unformatted code
- Developers can auto-fix locally with `npm run format` to avoid CI failures

**DRI suggestion:** Node lead  
**Effort:** Low — replace custom image with standard Node image + ESLint/Prettier config  
**Depends on:** None

---

## ⚪ Tier 4 — Horizon (Strategic / Long Lead Time)

These are real and worth tracking, but they depend on Tier 1–2 progress or require a strategic decision that isn't ready yet.

| # | Challenge | Dependency / Blocker |
|---|-----------|----------------------|
| T4-1 | **SBOM generation (Syft/CycloneDX) per release** | T1-6 + T2-7. Required for enterprise customer trust and EU Cyber Resilience Act compliance (2027). No SBOM = no quick CVE response. |
| T4-2 | WCAG accessibility compliance (Customer Widget + Agent Desk) | Customer ask growing; no owner or target level defined yet |
| T4-3 | Continuous Deployment — business feedback loop | Requires a "friendly customer" programme and feedback mechanism design |
| T4-4 | Customer-facing support lifecycle policy | FNB/Andreas request; needs product and management alignment |
| T4-5 | Cisco deployment ownership and reliability | Low frequency; needs an owner assigned, then a process |
| T4-6 | **DAST (OWASP ZAP) against staging on every release** | T1-6 + T2-7. Runtime security validation: headers, CORS, auth bypass, exposed endpoints. SAST cannot catch runtime misconfigurations. |
| T4-7 | **Container image signing (Cosign) and SLSA Level 1 provenance** | T1-6 + T2-7. Cryptographic proof that images were built by the official CI pipeline. Prevents registry tampering and supply chain attacks. |

### T4-1 · SBOM Generation (Syft/CycloneDX)
**Cluster:** F4 (security audit finding)

**Problem:** No SBOM is generated for any release. When the next Log4j-equivalent CVE is announced, there is no way to quickly answer: "Do we use this component? In which version? In which services?"

**Why it's Tier 4:** Not immediately blocking. But enterprise customers (FNB, Cisco) increasingly require SBOMs. The EU Cyber Resilience Act mandates SBOMs for software products by 2027.

**What good looks like:**
- Syft generates CycloneDX SBOMs for every release (Node + Java + container images)
- SBOMs are published to a registry alongside images
- A searchable SBOM database allows querying by component name and version across all releases

**DRI suggestion:** Zaryab Baloch (Security Lead/PO) + Haroon  
**Effort:** Low–Medium — tool configuration + registry integration  
**Depends on:** T1-6, T2-7

---

### T4-6 · DAST (OWASP ZAP) Against Staging
**Cluster:** F5 (security audit finding)

**Problem:** No runtime security validation of deployed applications exists. SAST tools catch code-level issues but cannot detect runtime misconfigurations like missing security headers, exposed admin endpoints, CORS misconfigurations, or authentication bypass.

**Why it's Tier 4:** Requires a stable staging environment and a mature CI/CD pipeline. DAST is noisy and requires tuning. But the Angular frontend + Node/Java backends are exposed to the internet; missing CSP headers or exposed `/actuator` endpoints are exploitable.

**What good looks like:**
- OWASP ZAP baseline scan runs against staging on every release
- ZAP full scan runs before every major release
- Findings are triaged and tracked in Jira with SLAs
- False positive suppression file maintained

**DRI suggestion:** Zaryab Baloch (Security Lead/PO) + QA lead  
**Effort:** Medium — tool setup + environment configuration + tuning  
**Depends on:** T1-6, T2-7

---

### T4-7 · Container Image Signing (Cosign) and SLSA Level 1
**Cluster:** F4 (security audit finding)

**Problem:** Images pushed to the GitLab registry have no cryptographic signature. At deployment time, Kubernetes pulls the image without verifying its provenance. There is no attestation that an artifact was built by this specific pipeline, from this specific commit, with these specific dependencies.

**Why it's Tier 4:** Requires a stable signing infrastructure and key management. The risk is supply chain tampering — a compromised registry or MITM attacker could substitute a tampered image. SLSA Level 1 is the minimum bar for build provenance.

**What good looks like:**
- Cosign signs every image with a key stored in GitLab CI variables (or KMS)
- Kubernetes admission controller verifies image signatures before deployment
- SLSA Level 1 provenance generated for every build (build source, dependencies, builder identity)
- SLSA Level 2 (signed provenance) and Level 3 (hermetic builds) on the roadmap for 2027

**DRI suggestion:** Zaryab Baloch (Security Lead/PO) + Haroon  
**Effort:** Medium — key management + Cosign integration + admission controller  
**Depends on:** T1-6, T2-7

---

## Summary View

| Tier | Count | Defining characteristic |
|------|-------|------------------------|
| 🔴 Tier 1 | 6 | Root causes and active risks; fix first |
| 🟠 Tier 2 | 13 | High compounding cost; next 1–2 quarters |
| 🟡 Tier 3 | 13 | Real value; plan this quarter, sequence carefully |
| ⚪ Tier 4 | 7 | Strategic horizon; track but don't act yet |

---

## Security Items by Phase

The following table maps security items to the 3-phase execution plan from `ci-cd-security-plan-consolidated.md`:

| Phase | Timeline | Security Items | Target Score |
|-------|----------|----------------|--------------|
| **Phase 1: Stop the Bleeding** | Week 1 | T1-6 (enforce scans), T2-11 (TruffleHog), T2-10 (npm audit + OWASP DC), T3-12 (Java auto-build), T3-13 (format job) | 19 → 55 |
| **Phase 2: Baseline Security** | Weeks 2–3 | T2-7 (VAPT automation), T2-12 (Dockerfile scanning), T2-13 (Checkov), T3-11 (coverage thresholds), T2-4 (supported release scanning) | 55 → 78 |
| **Phase 3: Supply Chain & Runtime** | Months 2–3 | T4-1 (SBOM), T4-6 (DAST/ZAP), T4-7 (Cosign signing + SLSA) | 78 → 91 |

---

## Cognitive Load Index

The following items have the highest direct impact on stream-aligned team friction — independent of release outcome:

| Item | Cognitive load problem it removes |
|------|----------------------------------|
| T1-1 (DoD enforcement) | "Am I done?" becomes an objective answer, not a negotiation |
| T1-4 (pre-release announcement) | "What do I build against?" has one clear answer, always |
| T1-6 (security scan enforcement) | "Is this build safe?" is answered by the pipeline, not by hoping someone read the Trivy report |
| T2-2 (deployment guides validated) | Teams own their guide; RMT stops being the place where gaps are discovered |
| T2-5 (target lock) | "Will my target change?" is bounded by a known protocol |
| T2-11 (secret scanning) | "Did I accidentally commit a secret?" is caught before push, not after deployment |
| T3-4 (IaC pre-release self-serve) | Teams can spin up their own env without waiting for RMT |
| T3-2 (feature flags) | Teams can merge incomplete work safely; no "is this ready to ship?" gate on merge |
| T3-7 (TBD adoption) | One branching model across all teams; no context switching |
| T3-13 (format enforcement) | Code review focuses on logic, not formatting; security-relevant code is easier to spot |

---

## DRI Quick Reference

| Person | Items | Role |
|--------|-------|------|
| **Zaryab Baloch** | T1-6, T2-4, T2-7, T2-11, T4-1, T4-6, T4-7 | Security Lead / Product Owner |
| Haroon | T1-1, T1-4, T1-5, T2-1, T2-6, T2-7, T2-12, T2-13, T3-4, T3-8, T3-10, T4-1, T4-7 | RMT / DevOps |
| Nabeel | T1-3, T2-3 | Stream lead |
| Umar Naveed | T1-5, T2-6 | CD pipeline / QA |
| Jawad | T2-5, T2-8, T3-5, T3-10 | Program management |
| Umar Ikhlaq | T3-1 | Head of QA |
| Awais | T3-2, T3-7 | DevOps / TBD |
| Node lead | T2-10, T3-11, T3-13 | Node.js stream |
| Java lead | T2-10, T3-11, T3-12 | Java stream |
| Stream tech leads | T1-1, T2-1, T2-2, T3-3, T3-6, T3-8 | Per component |

---

## Notes for Discussion with Haroon / Nabeel / Jawad

1. **T1-6 is the security anchor.** Every other security item depends on it. Without `allow_failure: false`, adding TruffleHog, Checkov, or ZAP is theater. The rollout plan (4 weeks, with CVE triage first) is designed to minimize pipeline breakage.

2. **T2-4 and T2-7 DRI updated.** Original docs listed "Abdul Moeed" as Security Lead DRI. These have been updated to "Zaryab Baloch" in this consolidated version. Original planning artifacts are preserved unchanged.

3. **Security adds 7 new Tier 2 items, 3 new Tier 3 items, and 3 new Tier 4 items.** The total priority list grows from 28 to 39 items. This is a significant expansion but reflects the audit finding that security was under-represented in the original prioritization.

4. **Tier 1 now has 6 items (was 5).** T1-6 elevates security enforcement to the same criticality as DoD enforcement, multi-path upgrade, and CD pipeline setup.

5. **Phase 1 exit criteria:** `allow_failure: false` on all security jobs, TruffleHog, `npm audit`, OWASP Dependency-Check, Java auto-build, format job re-enabled. Estimated timeline: 1 week.

6. **Resource implications:** The security audit identified ~20 hours/week of Security Lead time for Phase 1–2. DevOps time is concentrated in Week 1 (T1-6 configuration) and Weeks 2–3 (tool integration). Stream lead time is front-loaded for language-specific items (T2-10, T3-11, T3-12, T3-13).
