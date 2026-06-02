# CI/CD Objectives — Gap Analysis

**Source:** [CI/CD Objectives (Confluence)](https://expertflow-docs.atlassian.net/wiki/spaces/EF/pages/2000376/CI+CD+Objectives) and child pages  
**Reviewed by:** Jawad Bokhari  
**Date:** 2026-06-02  
**Purpose:** Items from the Confluence CI/CD objectives discussion that are not yet addressed in the master pipeline doc. Each gap below is a work item to discuss and resolve together.

---

## Gap 1 — WCAG Compliance

**What Confluence says:**  
Customer-facing accessibility compliance (WCAG) was implemented in Hybrid Chat but abandoned in 2021. Customers are now actively asking for it and it needs to be resumed for **Customer Widget** and **Agent Desk** at minimum.

**Why it matters:**  
This is a customer-driven obligation, not a nice-to-have. Without it, sales opportunities and enterprise deals can be blocked.

**What's missing from project docs:**  
No mention of accessibility compliance anywhere in the master pipeline or QA roadmap docs.

**Questions to resolve:**
- Which WCAG level is the target? (AA is the standard enterprise baseline)
- Is this a QA responsibility (automated axe-core / Playwright accessibility checks) or a dev responsibility (semantic HTML, ARIA)?
- Should it be a CI quality gate (blocking) or a reporting metric first?
- Who owns the initial audit of Customer Widget and Agent Desk?

---

## Gap 2 — Vulnerability Management Across All Supported Releases

**What Confluence says:**  
Two distinct objectives:
1. Publish vulnerability report for the **latest release** — automated Trivy scan, consolidated report to Confluence.
2. Publish vulnerability report for **all supported releases** — scheduled Trivy scan across every active CX version, automated reporting.

Plus: define and announce the **vulnerability fix process for public releases** (action item: Abdul Moeed — currently open).

**Why it matters:**  
The master pipeline covers Trivy scanning on RC merges and production releases but only for the current release in flight. Customers on older supported versions (e.g. CX5.0.x while CX5.1 is current) have no visibility into their security posture.

**What's missing from project docs:**  
- No scheduled scan policy for older supported releases
- No process for publishing vulnerability reports externally or to Confluence
- Vulnerability fix process for public releases not formally defined
- No definition of which release versions are "supported" at any given time

**Questions to resolve:**
- How many versions are actively supported concurrently? (Define the support window)
- What is the publication channel for vulnerability reports — Confluence, customer portal, both?
- What is the SLA for fixing a CVE found in an older supported release vs the current release?
- Who is the DRI for the vulnerability fix process? (Abdul Moeed was assigned but this is open)

---

## Gap 3 — Automated Dependency Updates

**What Confluence says:**  
Two-phase approach:
1. Short-term: Add a `Update Dependencies` CI step using `mvn versions:use-latest-releases` (Java) and `npx npm-check-updates -u` (Node).
2. Medium-term (Q3): Deploy an automated dependency update bot — **Renovate** (open source) or **Snyk** (paid) — that raises MRs automatically on a weekly/monthly schedule.

**Why it matters:**  
Many CVEs flagged by Trivy are in outdated dependencies, not code. Without automated updates, the dependency debt grows faster than teams can manually address it.

**What's missing from project docs:**  
The master pipeline mentions Trivy scanning (detect) but has no strategy for remediation automation (fix). This is the upstream step that reduces the Trivy alert volume.

**Questions to resolve:**
- Renovate (free, self-hosted) vs Snyk (paid, more opinionated) — decision needed
- Should the dependency update bot target only CVE-flagged deps or all outdated deps?
- How do MRs from the bot get reviewed — automated approval for patch versions? Tech Lead review for minor/major?
- Java (Maven) and Node components are clear candidates; what about Helm chart dependencies?

---

## Gap 4 — Load Test Environment

**What Confluence says:**  
Need a config guide for a load test environment using **stubs of a vanilla CX solution**. (Jehanzeb Riaz was to confirm scope — status unknown.)

**What QA Roadmap adds:**  
Voice load testing and JMeter are on the QA roadmap; individual learning goal for Q1 2026 for QA team members.

**Why it matters:**  
Without a defined load test environment, performance baselines cannot be established and voice/scale regressions go undetected before customer deployments.

**What's missing from project docs:**  
- No load test environment definition or topology
- No baseline performance targets (concurrent agents, concurrent conversations, API response times)
- No defined when load testing runs (pre-release gate? scheduled? on-demand?)
- No defined tools/framework in the master pipeline (JMeter referenced in QA Roadmap only)

**Questions to resolve:**
- Is the load test environment a separate VM/cluster or a subset of RMT?
- What is the minimal vanilla CX configuration that is representative enough for load testing?
- Which components must be included in scope first? (CXVOICE is the hardest — no IaC yet)
- What are the initial pass/fail thresholds for a load test gate?
- Who owns the environment setup — RMT (Haroon/Nabeel) or QA (Umar/Hassan)?

---

## Gap 5 — Multi-Step Helm Upgrade via Pre/Post Hooks

**What Confluence says:**  
Action item for Masood Farooq Malik: implement **multi-step upgrade through Helm pre/post hooks** — currently open.

**Why it matters:**  
Customers upgrading from CX5.0 to CX5.2 (skipping a minor) need intermediate migration steps (schema changes, config transformations) executed automatically. Without Helm hooks orchestrating this, upgrade failures and manual intervention are inevitable.

**What's missing from project docs:**  
The master pipeline documents the upgrade/downgrade goal at a high level (Helm-based automation) but does not address multi-version upgrade paths or how Helm lifecycle hooks are used to execute intermediate steps.

**Questions to resolve:**
- Which release-to-release transitions require intermediate steps? (Database schema migrations are the primary driver)
- Are the intermediate steps idempotent? Can a failed upgrade mid-way be safely re-run?
- Who defines the pre/post hook scripts — RMT or the stream team that owns the migration?
- Is there a test harness for upgrade path testing (upgrade N → N+2 in staging)?

---

## Gap 6 — SonarQube Coverage Targets: Developer vs Community

**What Confluence says:**  
Expertflow runs two SonarQube instances with different coverage targets:
- **SonarQube Developer:** 90% coverage on new code
- **SonarQube Community:** 80% coverage on new code

Action items (SonarQube Code Coverage Plan page):
- Implement quality gates per defined criteria
- Generate SonarQube scan report for overall codebase (both editions)
- Plan identified technical debt with relevant team (create Epics)

**What's missing from project docs:**  
The master pipeline references SonarQube as a gate (no new critical/blocker issues) but does not document the two-instance setup, the different coverage thresholds per edition, or the technical debt remediation planning process.

**Questions to resolve:**
- Which teams/projects are on Developer edition vs Community edition? Is there a migration plan?
- Are the quality gate definitions finalized for both instances? (Confluence links to a separate quality gates page — needs review)
- What is the current baseline coverage across the codebase? (This scan was an action item for Nabeel/Haroon — what was the result?)
- How does technical debt get planned — as dedicated Epics per team, or as a shared backlog?

---

## Gap 7 — Continuous Delivery vs Continuous Deployment: Strategic Distinction

**What Confluence says:**  
These are two distinct practices with different orientations:

| | Continuous Delivery | Continuous Deployment |
|---|---|---|
| Question | Are we building things **right**? | Are we building the **right** thing? |
| Focus | Technical | Business |
| Mechanism | Commit-to-release pipeline; fast flow | Roll out to in-house team or friendly customers for business feedback |
| Gate | Technical quality gates | Business usability and customer validation |

Continuous Deployment at Expertflow means getting releasable software in front of real users (internal business team or friendly customer) **before deciding to ship widely** — not relying on technical assumptions alone.

**What's missing from project docs:**  
The master pipeline defines the technical pipeline (Continuous Delivery) thoroughly but does not define the **Continuous Deployment** practice — who the "friendly customers" are, how business feedback is collected, what the feedback loop looks like, and how it feeds back into requirements.

**Questions to resolve:**
- Which internal business team can act as the first-recipient of new releases for business validation?
- Do we have a "friendly customer" programme? If not, how do we establish one?
- What is the feedback mechanism — structured interview, usage analytics, support tickets, dedicated session?
- How does business feedback connect back to the CE phase (Gatekeeper / backlog prioritization)?

---

## Gap 8 — Breaking CX into Smaller Releasable Packages

**What Confluence says:**  
A recurring action item: go through the SAFe **Release on Demand** framework to discuss decomposing CX into smaller, independently releasable packages. Two Google Docs and one Confluence page exist on this topic (not yet reviewed here).

**Why it matters:**  
The current CX release is a monolithic bundle. Smaller packages mean faster delivery of specific capabilities to customers, reduced upgrade risk, and more targeted testing.

**What's missing from project docs:**  
The master pipeline references Release on Demand in Phase 5 (RoD) but has no strategy for how CX would be decomposed — what the candidate packages are, which teams own them, and what the release cadence per package would be.

**Questions to resolve:**
- What are the natural package boundaries? (CXAGENT, CXCHAN, CXVOICE, CXBI, CXPLAT are logical candidates but they currently ship together)
- Which components have hard interdependencies that prevent independent release? (Object Model/OM is one known blocker — documented in Road to CD)
- What is the minimum change to the skeleton project to support selective component versioning?
- Is this a Q3/Q4 initiative or longer-horizon? (Depends on OM dependency resolution)

---

## Summary Table

| Gap | Urgency | Who Should Drive | Current Status |
|-----|---------|-----------------|----------------|
| WCAG Compliance | Medium (customer ask) | QA Lead + Dev Leads | Not started |
| Vulnerability mgmt all releases | High (customer trust) | RMT + Abdul Moeed | Action item open |
| Automated dependency updates | Medium | DevOps / RMT | Not started |
| Load test environment | Medium | RMT + QA | Scoping only |
| Multi-step Helm upgrade hooks | High (upgrade reliability) | Masood / RMT | Action item open |
| SonarQube Dev vs Community targets | Medium | Haroon + Nabeel | Scans pending |
| CD vs CI strategic distinction | Low (clarity/framing) | Jawad | Not documented |
| Smaller CX packages (OM dependency) | Low-Medium (strategic) | Jawad + Haroon + stream leads | Discussed only |
