# Problem Inventory: CI/CD, Test Automation & Delivery Process

> **Status:** Complete — Shared Understanding Phase  
> **Sources:** `docs/How_We_Work.md`, `team_input/Road to CD.md`, `team_input/QA Automation (BDD + Playwright)..md`, `docs/cicd_objectives_gaps.md`, direct stakeholder input  
> **Goal:** Complete inventory before moving to solution design

---

## 1. Architectural Delivery Mismatch

### 1.1 Microservices Delivered as a Monolith
- **Problem:** The solution is architected as multiple microservices but is *delivered and tested as a single monolithic unit*.
- **Impact:** Teams cannot independently deploy, validate, or release their own services. Every change — regardless of scope — must pass through the full monolithic integration cycle, inflating lead time and risk.
- **Evidence:** Skeleton repository aggregates all components; Helm charts are manually coordinated across all services; RMT integration testing validates the entire stack, not just the changed component.
- **Source:** Direct stakeholder input

### 1.2 No Strategy for Decomposing CX into Smaller Releasable Packages
- **Problem:** A recurring action item exists to apply SAFe Release on Demand to decompose CX into smaller, independently releasable packages, but no strategy or candidate boundaries have been defined.
- **Impact:** The monolithic release cycle continues indefinitely. Even components that could be decoupled (CXAGENT, CXCHAN, CXVOICE, CXBI, CXPLAT) ship together, forcing every team into the same cadence and risk pool.
- **Evidence:** CI/CD Objectives Gap 8 — two Google Docs and one Confluence page exist on this topic but no decisions made; Object Model (OM) dependency is a known blocker to independent release.
- **Source:** `docs/cicd_objectives_gaps.md` (Gap 8), `team_input/Road to CD.md`

---

## 2. Delayed Feedback Loop (Stream-Aligned Quality Gaps)

### 2.1 Feature Testing Is Inadequate at the Team Level
- **Problem:** Stream-aligned teams do not perform thorough feature testing before handing off to RMT. Defects that should be caught during feature development escape to the release candidate.
- **Impact:** Issues discovered late in RMT are exponentially more expensive to fix. They delay the entire release and block other teams.
- **Evidence:**
  - How We Work §3.3: *"Teams are not consistently following the Release-Ready criteria before handing off to RMT"* — flagged as a known issue
  - Road to CD notes: feature development testing was not performed properly, resulting in multiple issues discovered during release testing
- **Source:** `docs/How_We_Work.md`, `team_input/Road to CD.md`, direct stakeholder input

### 2.2 Release-Ready Definition of Done Is Not Enforced
- **Problem:** The Release-Ready DoD exists on paper but is not enforced by tech leads or the CI pipeline. Teams self-report readiness without objective validation.
- **Impact:** Incomplete features (missing docs, untested upgrade paths, unverified BAT) flow into the release pipeline.
- **Evidence:**
  - How We Work §3.3: *"Tech leads in each stream team are the first escalation point for this"* — implying reactive, not preventive, enforcement
  - BAT was reinstated because it had stopped being practiced; teams "were not aware it was required"
- **Source:** `docs/How_We_Work.md`

### 2.3 Environment Inconsistency
- **Problem:** Stream team dev environments differ materially from RMT. A feature that "works on dev" may fail on RMT due to configuration drift, data differences, or missing integration surfaces.
- **Impact:** Even when teams DO test properly, the results are not trustworthy. False confidence leads to escaped defects.
- **Evidence:** Direct stakeholder feedback — "environment inconsistencies: works on dev, not on RMT; works on one RMT, not another RMT"
- **Source:** Direct stakeholder input

### 2.4 Narrow Feature Testing Scope by Design
- **Problem:** Stream-aligned QA performs targeted regression only in areas directly affected by the change. Full regression is explicitly deferred to RMT.
- **Impact:** This design is sound *only if* feature testing is robust. If feature testing is weak (as evidence shows), the system is architected to let defects leak.
- **Evidence:** How We Work §2.4: *"Full regression is NOT performed at this stage — that is RMT's responsibility at release level"*
- **Source:** `docs/How_We_Work.md`

---

## 3. RMT Bottleneck & Release Pipeline

### 3.1 Single RMT Instance Serializes All Validation
- **Problem:** All teams must queue to test on a single shared RMT incremental build instance. Only one feature can be tested at a time.
- **Impact:** Work piles up instead of flowing. Teams wait days or weeks for RMT access. Release velocity is capped by RMT capacity, not development capacity.
- **Evidence:**
  - Road to CD #1: *"Team B can't start until Team A is done on RMT. Work is piling up instead of flowing"*
  - Stakeholder clarification: queue is a symptom, but the real issue is that defects found at RMT create rework loops
- **Source:** `team_input/Road to CD.md`, direct stakeholder input

### 3.2 RMT Operations Are Entirely Manual
- **Problem:** Building, deploying, and testing on RMT are all manual processes. Every step depends on specific people (Haroon, Junaid) being available.
- **Impact:** Slow, error-prone, and non-scalable. No ability to run parallel validation tracks.
- **Evidence:** Road to CD #2: *"Everything on RMT is manual"*
- **Source:** `team_input/Road to CD.md`

### 3.3 No Automated RMT Change Notification
- **Problem:** When the RMT instance is updated (new build deployed, config changed), teams are not automatically notified.
- **Impact:** Teams test against stale environments or miss critical changes, leading to false results and wasted effort.
- **Evidence:** Road to CD #3: *"When something changes, nobody gets notified or updated automatically"*
- **Source:** `team_input/Road to CD.md`

### 3.4 Helm Chart Pipeline Is Manual
- **Problem:** Helm charts for release candidates require manual editing of values and tags. Coordination between GitLab and GitHub chart sources is also manual.
- **Impact:** Human error in version/tag management. Release packaging is slow and requires dedicated RMT resources.
- **Evidence:** How We Work §3.2: *"Manually edit chart values and tags for each release candidate... RMT confirms which applies per release component"*
- **Source:** `docs/How_We_Work.md`

### 3.5 Dual GitLab/GitHub Chart Source of Truth
- **Problem:** Helm chart sources exist in both GitLab and GitHub with no clear ownership or automated reconciliation.
- **Impact:** Risk of deploying wrong chart versions. Confusion about which source applies to which component.
- **Evidence:** How We Work §3.2 and Open Action Items
- **Source:** `docs/How_We_Work.md`

---

## 4. Test Automation Gaps

### 4.1 Regression Automation Coverage Is Too Low
- **Problem:** Automated regression coverage is approximately 40%. The majority of regression testing is manual.
- **Impact:** Full regression cycles are slow and resource-intensive. They cannot be run frequently, which means defects accumulate between runs.
- **Evidence:** How We Work Open Action Items: *"Increase regression automation coverage beyond 40%"*
- **Source:** `docs/How_We_Work.md`

### 4.2 Manual Regression Testing Blocks Releases
- **Problem:** Full regression is executed manually at the release stage. This is the final gate before production.
- **Impact:** Any defect found triggers a full or partial re-test, compounding delays. The release timeline is unpredictable.
- **Evidence:** Testing Responsibility Matrix shows "Full regression" executed by RMT + QA at release testing stage
- **Source:** `docs/How_We_Work.md`

### 4.3 No Integration Testing Strategy for Java/Node
- **Problem:** There is no standardized integration testing guide or framework for the Java and Node.js services.
- **Impact:** Teams cannot write reliable integration tests. Contract boundaries between microservices are unverified.
- **Evidence:** Road to CD Open Action Items: *"Nabeel — Put together an integration testing guide for Java and Node"* — Status: No Progress
- **Source:** `team_input/Road to CD.md`

### 4.4 QA Automation Framework in Early Stage
- **Problem:** A Playwright + BDD automation framework is being designed but not yet production-ready. Key blockers exist:
  - `data-testid` attributes are not consistently implemented in the Angular frontend
  - No dedicated automation repository is live
  - AI-assisted workflow (Claude + Cursor) is experimental
- **Impact:** UI automation cannot be written reliably. The existing Angular CSS classes are dynamically generated and change after every deployment.
- **Evidence:** `team_input/QA Automation (BDD + Playwright)..md`
- **Source:** `team_input/QA Automation (BDD + Playwright)..md`

### 4.5 Test Data Management Is Undefined
- **Problem:** No evidence of a strategy for test data provisioning, seeding, or cleanup across environments.
- **Impact:** Tests may fail due to data conflicts or stale state. Reproducibility is low.
- **Evidence:** Not explicitly mentioned in any document — inferred gap.
- **Source:** Inferred

### 4.6 SonarQube Coverage Targets Are Inconsistent and Undocumented
- **Problem:** Two SonarQube instances run with different coverage targets (Developer: 90% on new code; Community: 80% on new code). The two-instance setup, quality gate definitions, and technical debt remediation process are not documented in the master pipeline.
- **Impact:** Teams don't know which instance applies to them or what the gate is. Coverage baselines for the overall codebase are unknown. Technical debt is not systematically planned.
- **Evidence:** CI/CD Objectives Gap 6 — action items for implementing quality gates, generating scan reports, and planning technical debt are open.
- **Source:** `docs/cicd_objectives_gaps.md` (Gap 6)

### 4.7 No Load Test Environment or Performance Baselines
- **Problem:** No load test environment has been defined. Performance baselines (concurrent agents, conversations, API response times) do not exist. It is unclear when load testing runs, with what tools, or what the pass/fail thresholds are.
- **Impact:** Performance regressions go undetected until customer deployment. Voice load testing is on the QA roadmap but lacks infrastructure.
- **Evidence:** CI/CD Objectives Gap 4 — Jehanzeb Riaz was to confirm scope but status is unknown. QA roadmap mentions JMeter but no master pipeline integration.
- **Source:** `docs/cicd_objectives_gaps.md` (Gap 4)

---

## 5. Infrastructure & Environment

### 5.1 No IaC for Voice (CXVOICE)
- **Problem:** CXVOICE environments must be set up manually. No Infrastructure-as-Code scripts exist.
- **Impact:** Environment creation is high-effort, slow, and non-reproducible. Voice components cannot be reliably deployed or tested in CI/CD.
- **Evidence:** How We Work §2.1: *"IaC NOT available. Environment must be set up manually."* — Status: Manual — high effort
- **Source:** `docs/How_We_Work.md`

### 5.2 BI/WFM Servers Not in IaC Pool
- **Problem:** CXBI/WFM servers are not managed by IaC. This team is also excluded from release planning.
- **Impact:** BI components cannot be automatically deployed or tested alongside the rest of the stack.
- **Evidence:** How We Work §2.1 and §8
- **Source:** `docs/How_We_Work.md`

### 5.3 CXCHAN Needs IaC Enablement
- **Problem:** Channels team has not been enabled on IaC setup and code review process.
- **Impact:** Same as above — deployment friction, environment inconsistency.
- **Evidence:** How We Work §2.1 and §8
- **Source:** `docs/How_We_Work.md`

### 5.4 GitLab Runner Performance Under Load
- **Problem:** GitLab runners slow down significantly when multiple pipelines run concurrently.
- **Impact:** CI feedback loops become longer. Developers wait for pipeline results, reducing productivity.
- **Evidence:** Direct stakeholder input
- **Source:** Direct stakeholder input

### 5.5 No Data Rollback Mechanism
- **Problem:** There is no reliable rollback for MongoDB and PostgreSQL. DB backups exist but restoring them risks data loss for transactions after the backup.
- **Impact:** If a release with DB migrations causes a critical issue, there is no safe recovery path. This is a **🔴 Critical** risk.
- **Evidence:** How We Work §7: *"No reliable rollback... High risk"* — flagged as critical gap
- **Source:** `docs/How_We_Work.md`

### 5.6 No Multi-Step Helm Upgrade via Pre/Post Hooks
- **Problem:** Helm upgrades that skip minor versions (e.g., CX5.0 → CX5.2) require intermediate migration steps (schema changes, config transformations). These are not orchestrated automatically.
- **Impact:** Upgrade failures and manual intervention are inevitable. Customers cannot safely skip versions. The upgrade path is a major source of release risk.
- **Evidence:** CI/CD Objectives Gap 5 — action item for Masood Farooq Malik is open.
- **Source:** `docs/cicd_objectives_gaps.md` (Gap 5)

### 5.7 No Load Test Environment Definition
- **Problem:** There is no defined load test environment topology, tooling, or ownership. It is unclear whether load testing runs on a dedicated cluster, a subset of RMT, or on-demand.
- **Impact:** Performance testing is either skipped or run on environments that do not represent production load patterns. Results are not comparable.
- **Evidence:** CI/CD Objectives Gap 4 — no environment definition, no baseline targets, no tool mandate in the master pipeline.
- **Source:** `docs/cicd_objectives_gaps.md` (Gap 4)

---

## 6. Team Enablement & Process Adherence

### 6.1 Three Teams Excluded from Release Planning
- **Problem:** CXCHAN (Ehtasham), CXBI/WFM (Umar), and Quality Management are consistently absent from Track 2 release planning.
- **Impact:** Their components are included in releases but their constraints, readiness, and dependencies are not accounted for during planning. Late-discovered scope issues are inevitable.
- **Evidence:** How We Work §1: *"Three teams are consistently absent from Track 2 planning... their absence is a known source of late-discovered scope issues"*
- **Source:** `docs/How_We_Work.md`

### 6.2 BI and Other Teams Not Following the Process
- **Problem:** BI and some other stream-aligned teams are not enabled on the standard process. Their participation is ad-hoc and human-driven.
- **Impact:** Process gaps create integration failures, missed handoffs, and undocumented changes.
- **Evidence:** Direct stakeholder input — *"BI and some other teams are not enabled and they do not follow the process. It's pretty much human-driven as of now"*
- **Source:** Direct stakeholder input

### 6.3 Jira Usage Is Inconsistent
- **Problem:** Teams do not consistently update Jira statuses or document configuration changes.
- **Impact:** Release planning lacks accurate visibility. RMT cannot reliably determine what is Release-Ready.
- **Evidence:** How We Work §8: *"Jira usage inconsistent — statuses not updated correctly"*
- **Source:** `docs/How_We_Work.md`

---

## 7. Security & Compliance

### 7.1 VAPT Process Not Automated
- **Problem:** Vulnerability Assessment and Penetration Testing (VAPT) is not automated or systematically scheduled for released and to-be-released versions.
- **Impact:** Security gaps are discovered reactively (or by customers), not proactively.
- **Evidence:** Road to CD June 1 meeting points: *"VAPT process on released and to-be-released: Announce the policy; Automate the scanning and fixing — security hotfix"*
- **Source:** `team_input/Road to CD.md`

### 7.2 Credential Leakage Risk in CI/CD
- **Problem:** Security enhancements for credentials leakage and similar risks are needed in the CI/CD pipeline.
- **Impact:** Secrets could be exposed in logs, artifacts, or version control.
- **Evidence:** Direct stakeholder input
- **Source:** Direct stakeholder input

### 7.3 Security Scanning Is Partial
- **Problem:** Trivy runs in CI but OWASP, SonarQube, Burp Suite, and Nmap scans are performed manually at release time.
- **Impact:** Security feedback is delayed to the end of the cycle. Issues found late are expensive to fix.
- **Evidence:** How We Work §3.1: Security Testing gate at release testing stage
- **Source:** `docs/How_We_Work.md`

### 7.4 No Vulnerability Management for Supported Releases
- **Problem:** Trivy scans run for the current release in flight, but there is no scheduled scan policy for older supported releases. Customers on older versions (e.g., CX5.0.x while CX5.1 is current) have no security posture visibility.
- **Impact:** CVEs in supported releases are discovered reactively or by customers. There is no defined SLA for fixing CVEs in older releases.
- **Evidence:** CI/CD Objectives Gap 2 — no scheduled scan policy, no publication channel defined, no support window defined, vulnerability fix process for public releases not formally defined (action item: Abdul Moeed — open).
- **Source:** `docs/cicd_objectives_gaps.md` (Gap 2)

### 7.5 No Automated Dependency Update Strategy
- **Problem:** Many CVEs flagged by Trivy originate from outdated dependencies, not from custom code. There is no automated dependency update process — neither a CI step (`mvn versions:use-latest-releases`, `npm-check-updates`) nor a bot (Renovate, Snyk).
- **Impact:** Dependency debt grows faster than teams can manually address it. Trivy alert volume is inflated by known, fixable issues.
- **Evidence:** CI/CD Objectives Gap 3 — two-phase approach defined but not implemented. Decision needed on Renovate vs Snyk.
- **Source:** `docs/cicd_objectives_gaps.md` (Gap 3)

### 7.6 No WCAG Accessibility Compliance
- **Problem:** Customer-facing accessibility compliance (WCAG) was implemented in Hybrid Chat but abandoned in 2021. It is not mentioned in the master pipeline or QA roadmap. Customers are now actively requesting it.
- **Impact:** Sales opportunities and enterprise deals can be blocked. Compliance debt grows as the UI evolves without accessibility considerations.
- **Evidence:** CI/CD Objectives Gap 1 — target level undefined, ownership unclear (QA vs dev), not a CI gate.
- **Source:** `docs/cicd_objectives_gaps.md` (Gap 1)

---

## 8. Component-Specific Deployment Challenges

### 8.1 Voice Components Are Outside the Skeleton
- **Problem:** Voice components are not part of the `cim-solution` skeleton repository. Structural definition, release management, and automated testing for voice are undefined.
- **Impact:** Voice releases are manual and decoupled from the main release process. Risk of version skew.
- **Evidence:** Road to CD June 1 meeting points
- **Source:** `team_input/Road to CD.md`

### 8.2 Object Model (OM) Versioning Is Manual
- **Problem:** The Object Model dependency requires manual updates across components. This is time-consuming and error-prone.
- **Impact:** Release packaging is slowed. Version mismatches cause integration failures.
- **Evidence:** Road to CD June 1 meeting points: *"Need to either remove the OM dependency or automate the process"*
- **Source:** `team_input/Road to CD.md`

### 8.3 Reporting & Data Platform Packaging Undefined
- **Problem:** Metabase and Airflow (reporting and data platform) do not have a defined packaging and release mechanism.
- **Impact:** These components cannot be reliably included in automated releases.
- **Evidence:** Road to CD June 1 meeting points
- **Source:** `team_input/Road to CD.md`

### 8.4 Feature Flag Framework Is Not Finalized
- **Problem:** A feature flag framework is needed but not yet in production.
- **Impact:** Teams cannot decouple deployment from release. Every feature must be fully ready before merge, increasing batch size and risk.
- **Evidence:** Road to CD June 1 meeting points
- **Source:** `team_input/Road to CD.md`

---

## 9. Process & Governance Gaps

### 9.1 No Formal Rollback Procedure
- **Problem:** Rollback is initiated informally via Google Chat. No documented procedure exists.
- **Impact:** Incident response is ad-hoc, slow, and error-prone. The team relies on tribal knowledge.
- **Evidence:** How We Work §7.2–7.3
- **Source:** `docs/How_We_Work.md`

### 9.2 Dual-Codebase Patches Have No Process
- **Problem:** When Ops requires a feature on an older release, development must happen on both codebases. There is no formal process for deciding when this is triggered or how effort is accounted for.
- **Impact:** Unplanned, unbudgeted work. Sprint capacity is consumed reactively.
- **Evidence:** How We Work §5: *"This is a reactive, unplanned cost with no formal process today"*
- **Source:** `docs/How_We_Work.md`

### 9.3 Hot-Fix Process Gap
- **Problem:** Hot-fixes must be release-ready for the latest release as well, but this is not consistently enforced.
- **Impact:** Patches may introduce new issues because they skip validation gates.
- **Evidence:** Road to CD June 1 meeting points
- **Source:** `team_input/Road to CD.md`

### 9.4 Continuous Deployment Practice Is Undefined
- **Problem:** The master pipeline defines Continuous Delivery (technical commit-to-release pipeline) but does not define Continuous Deployment (business validation via internal or friendly-customer rollout before wide release).
- **Impact:** Business usability and customer validation are not systematically gathered before shipping. The "are we building the right thing?" question is answered by assumption, not evidence.
- **Evidence:** CI/CD Objectives Gap 7 — no friendly customer programme, no feedback mechanism, no connection back to CE/Gatekeeper phase.
- **Source:** `docs/cicd_objectives_gaps.md` (Gap 7)

---

## 10. Missing Information

The following sources have been identified but **not yet incorporated** into this inventory:

| Source | Status | Action Needed |
|--------|--------|---------------|
| Confluence: [CI/CD Objectives](https://expertflow-docs.atlassian.net/wiki/spaces/EF/pages/2000376/CI+CD+Objectives) | ✅ Incorporated via `docs/cicd_objectives_gaps.md` | None — complete |
| Confluence child pages of CI/CD Objectives | ✅ Incorporated via `docs/cicd_objectives_gaps.md` | None — complete |
| Confluence: [Release retro](https://expertflow-docs.atlassian.net/wiki/spaces/EF/pages/505511940) | 🔴 Not yet fetched | Fetch if relevant |
| Confluence: [Sessions with Awais on improving CI/CD](https://expertflow-docs.atlassian.net/wiki/spaces/EF/pages/1636827164) | 🔴 Not yet fetched | Fetch if relevant |

---

## Next Steps

1. **Review for completeness:** Validate whether any additional problem areas exist or whether any items above need correction.
2. **Prioritize:** Once validated, we can classify problems by urgency/impact to focus the solution design.
3. **Proceed to solution design:** Move to TR (Technical Research) to evaluate improvement strategies.
