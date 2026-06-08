# Priority List: CI/CD, Test Automation & Delivery Process

> **Status:** Active — Working Backlog (Round 2 — updated June 2026)
> **Produced by:** Problem Inventory analysis + stakeholder corrections (June 2026)
> **Sources:** `problem-inventory-cicd-test-automation.md`, `docs/cicd_objectives_gaps.md`, `team_input/Road to CD.md`, Confluence retro pages, direct stakeholder input (Haroon — RMT, Junaid — RMT)
> **Last updated:** 2026-06-04

---

## Strategic Themes

Two goals run through all tiers:

1. **Increase release velocity and predictability** — reduce the time and uncertainty between a feature being "dev complete" and it being in production.
2. **Reduce cognitive load on stream-aligned teams** — teams should be able to build and ship with confidence, without needing tribal knowledge, manual coordination, or waiting on other humans to unblock them.

These themes are not in conflict. Most of the high-priority items serve both simultaneously.

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

**Start week:** June 8, 2026 — target: draft DoD criteria and identify first CI gate to enforce

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

**Sequencing note:** Nabeel's 4th and final priority in current stack — after OM version management (T1-6), Reporting/Data Platform packaging (T2-9), and Integration Testing guide (T2-3).

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

**Start week:** June 8, 2026 — target: Slack webhook wired to RC build completion event

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

**Sequencing note:** Nabeel's 3rd priority after OM version management (T1-6) and Reporting/Data Platform packaging (T2-9).

**DRI suggestion:** Nabeel Ahmad  
**Effort:** Low to start (guide) — Medium to implement

---

### T2-4 · Vulnerability Management for Supported Releases
**Cluster:** F4

**Problem:** Trivy scans run for the current release in flight. Customers on older supported versions (e.g. CX5.0.x while CX5.1 is current) have no security posture visibility. No support window definition exists. No SLA for CVE fixes in older releases. Abdul Moeed's action item is open.

**Why it's Tier 2:** Customer trust issue with enterprise clients. Sales blocker for regulated industries. No immediate incident today, but the exposure grows with each release.

**DRI suggestion:** Abdul Moeed + RMT  
**Effort:** Medium — scheduled pipeline + support window policy decision

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

**Problem:** VAPT is not automated or scheduled. OWASP, Burp Suite, and Nmap scans are manual at release time. Automated dependency updates (Renovate or equivalent) are not implemented — Trivy alert volume is inflated by fixable dependency debt.

**Why it's Tier 2:** Security feedback is currently end-of-cycle. Earlier detection is cheaper. Automated dependency updates directly reduce Trivy noise.

**Phased approach:**
1. Decide Renovate vs Snyk and implement automated dependency update MRs (reduces Trivy alert volume immediately)
2. Schedule Trivy across supported releases (addresses T2-4 overlap)
3. Automate VAPT cadence — schedule and publish reports

**DRI suggestion:** DevOps / RMT  
**Effort:** Medium

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

---

## ⚪ Tier 4 — Horizon (Strategic / Long Lead Time)

These are real and worth tracking, but they depend on Tier 1–2 progress or require a strategic decision that isn't ready yet.

| # | Challenge | Dependency / Blocker |
|---|-----------|----------------------|
| T4-2 | WCAG accessibility compliance (Customer Widget + Agent Desk) | Customer ask growing; no owner or target level defined yet |
| T4-3 | Continuous Deployment — business feedback loop | Requires a "friendly customer" programme and feedback mechanism design |
| T4-4 | Customer-facing support lifecycle policy | FNB/Andreas request; needs product and management alignment |
| T4-5 | Cisco deployment ownership and reliability | Low frequency; needs an owner assigned, then a process |

---

## Summary View

| Tier | Count | Defining characteristic |
|------|-------|------------------------|
| 🔴 Tier 1 | 5 | Root causes and active risks; fix first |
| 🟠 Tier 2 | 9 | High compounding cost; next 1–2 quarters |
| 🟡 Tier 3 | 10 | Real value; plan this quarter, sequence carefully |
| ⚪ Tier 4 | 4 | Strategic horizon; track but don't act yet |

---

## Cognitive Load Index

The following items have the highest direct impact on stream-aligned team friction — independent of release outcome:

| Item | Cognitive load problem it removes |
|------|----------------------------------|
| T1-1 (DoD enforcement) | "Am I done?" becomes an objective answer, not a negotiation |
| T1-4 (pre-release announcement) | "What do I build against?" has one clear answer, always |
| T2-2 (deployment guides validated) | Teams own their guide; RMT stops being the place where gaps are discovered |
| T2-5 (target lock) | "Will my target change?" is bounded by a known protocol |
| T3-4 (IaC pre-release self-serve) | Teams can spin up their own env without waiting for RMT |
| T3-2 (feature flags) | Teams can merge incomplete work safely; no "is this ready to ship?" gate on merge |
| T3-7 (TBD adoption) | One branching model across all teams; no context switching |
