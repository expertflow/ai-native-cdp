# 🔍 Deep Dive: T1-1 — Release-Ready Definition of Done Enforcement

**Owner:** Haroon + stream tech leads  
**Week:** June 8, 2026  
**Target:** Draft objective DoD criteria + identify first CI gate  
**Status:** Revised — incorporates May 2026 process update + stakeholder clarifications

---

## 1. The Current Process (May 2026 Update)

The Release-Ready flow has been **inverted**. Previously: merge to `develop` first, then RMT test. **Now: RMT test first, then merge to `develop`.

### 1.1 End-to-End Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    NEW RELEASE-READY WORKFLOW                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  MICROSERVICE REPO                    CIM-SOLUTION (Skeleton)               │
│  (e.g. cim-backend)                   (Helm charts + deployment configs)    │
│                                                                             │
│  feature/CIM-1234-xxx                 feature/CIM-1234-xxx                  │
│       │                                     │                               │
│       │  1. Dev + QA on dev server          │  1. Update image tags/config  │
│       │  2. Code review + Trivy             │                               │
│       │  3. Upgrade scripts ready             │                               │
│       │                                     │                               │
│       │       ┌─────────────────────────────┘                               │
│       │       │  4. Open MR: cim-solution feature → RC branch               │
│       │       │     (triggers CI: package charts → deploy to RMT)           │
│       │       │                                                             │
│       │       ▼  5. Haroon/Junaid manually deploy to RMT                    │
│       │          (QA runs feature + targeted regression on RMT)             │
│       │          (Parallel testing only if different microservices)         │
│       │                                                                     │
│       │       ▲  6. RMT testing PASSED                                      │
│       │       │                                                             │
│       └──►────┘  7. Merge microservice feature → develop                    │
│                  8. Merge cim-solution MR → RC branch                       │
│                  9. Mark Jira → Release-Ready                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 The Two-MR Requirement (Three-Repos for MongoDB Features)

A feature is **not** Release-Ready until **all required** merge requests are merged:

| MR | Repository | Target Branch | When It Opens | When It Merges | Required For |
|---|---|---|---|---|---|
| **MR-A** | Microservice repo (e.g. `cim-backend`) | `develop` | After dev/QA complete | **After** RMT testing passes | **All** features |
| **MR-B** | `cim-solution` (skeleton) | `Release-Candidate` | To trigger RMT deployment | **After** RMT testing passes | **All** features |
| **MR-C** | `transflux` (Airflow DAGs) | `develop` or release branch | When MongoDB schema changes are needed | **After** RMT testing passes | **MongoDB features only** |

> **Critical implication:** Features touching MongoDB require coordination across **three repositories** before they can be marked Release-Ready. This is a coordination risk that the current DoD does not explicitly address.

### 1.3 Component Collision Rules for RMT Parallel Testing

| Scenario | Same Component? | Allowed? |
|---|---|---|
| Feature A touches `cim-backend` + `agent-manager` | Feature B touches `conversation-manager` | Different microservices | ✅ **Yes** |
| Feature A touches `cim-backend` | Feature B also touches `cim-backend` | Same microservice | ❌ **No** |
| Feature A adds a new `cim-backend` API | Feature B consumes that API | Dependency (A must complete first) | ❌ **No** |

**Current state:** Haroon/Junaid manually manage the RMT queue and component reservations. No automated system tracks which microservices are under test.

---

## 2. What the Updated DoD Says

From the May 2026 Change Request:

| Step | Criterion | Validation Type |
|------|-----------|-----------------|
| 1 | Feature developed and tested on team's dev server | Self-reported / Jira |
| 2 | All relevant docs updated (deployment, config, API, feature) | Self-reported |
| 3 | Dev completes dev-testing | Self-reported |
| 4 | Deploy on dev-server for review | Self-reported |
| 5 | QA verifies feature on dev server | Jira status (QA-Passed) |
| 6 | Load test performed **at feature level** if needed | Self-reported |
| 7 | TechLead passes Code Review | GitLab MR approval |
| 8 | Trivy scan passed in CI pipeline | GitLab CI gate |
| 9 | **Upgrade scripting and configs supplied for the CI job** | File presence (see §3) |
| 10 | **Create MR for release candidate branch** (cim-solution) | GitLab MR |
| 11 | **Feature deployed and tested on the RMT environment** | RMT test results (manual) |
| 12 | **Merge feature branch into `develop`** | GitLab MR merged |
| 13 | Mark Jira → Release-Ready | Jira status |

---

## 3. Upgrade Script Landscape (Clarified)

The "upgrade scripting" in the new DoD refers to **database migration/upgrade scripts**. The location depends on the database type:

| Database | Script Repository | Location | Format | Ownership |
|----------|------------------|----------|--------|-----------|
| **Reporting DB (MySQL / MSSQL)** | `cim-solution` | `kubernetes/pre-deployment/reportingConnector/dbScripts/dbupdate/` | `.sql` | Reporting team |
| **MongoDB** | `transflux` (separate repo) | Airflow DAGs | Python DAGs | Data platform team |
| **PostgreSQL** (license manager) | `cim-solution` | `kubernetes/pre-deployment/licensemanager/` | `.sql` | Platform team |
| **Other component DBs** | Component-specific | Varies | `.sql`, `.js` | Component team |

### Existing Script Examples in cim-solution

```
cim-solution/
├── kubernetes/scripts/mongo/
│   ├── 4.4-4.5_upgrade.js
│   └── 4.0-4.2_upgrade.js
├── kubernetes/pre-deployment/reportingConnector/dbScripts/dbupdate/
│   ├── historical_reports_db_update_script_MYSQL_4.4.10_to_4.7.sql
│   └── historical_reports_db_update_script_MSSQL.sql.sql
└── kubernetes/pre-deployment/licensemanager/
    └── licensemanager.sql
```

> **Note:** The cim-solution `.gitlab-ci.yml` currently has **no validation** that upgrade scripts are present when DB-related changes are made. The Helm pipeline packages charts and auto-bumps versions, but does not check for migration script completeness.

---

## 4. Why Enforcement Is Still Failing

The process changed, but the **enforcement mechanism** did not.

### Problem A: No Gate Between "RMT Tested" and "Release-Ready"
- RMT testing is entirely manual — QA tests, then manually updates Jira
- There is **no artifact or CI check** that proves RMT testing passed
- A feature could theoretically be marked Release-Ready without ever touching RMT

### Problem B: The Multi-MR Model Has No Linkage
- MR-A (microservice → develop), MR-B (cim-solution → RC), and MR-C (transflux → develop) are **independent** in GitLab
- Nothing prevents merging MR-A **before** MR-B is tested
- Nothing prevents marking Jira Release-Ready if only one or two MRs are merged
- The "all MRs accepted" rule exists on paper but not in tooling

### Problem C: Upgrade Script Presence Is Not Validated
- The DoD requires upgrade scripts "for the CI job to merge this feature"
- But the CI job that merges MR-B (cim-solution) does not check for:
  - SQL files in `dbScripts/dbupdate/` when reporting values change
  - Airflow DAG changes in `transflux` when MongoDB schema changes
- This is a **latent requirement with no implementation**

### Problem D: RMT Deployment Success ≠ RMT Test Pass
- CI can package Helm charts successfully
- But "chart packaged" is not the same as "deployed and tested on RMT"
- RMT deployment is **manual** (Haroon/Junaid execute it)
- There is **no automated smoke test** post-RMT-deploy
- Feature testing on RMT is entirely manual QA

### Problem E: No Automated Component Reservation
- RMT parallel testing is allowed for **different microservices**
- But there is **no system** tracking which microservices are currently under test
- Haroon/Junaid manually manage the queue via tribal knowledge
- A team could open a cim-solution MR for a microservice already under test by another team

---

## 5. What "Good" Looks Like: Three Layers of Enforcement

### Layer 1: Pre-RMT Gates (Catch Before Deploying)

**Goal:** Don't waste RMT time with obviously incomplete features.

| Gate | Location | What It Checks | Current State |
|------|----------|---------------|---------------|
| **G1.1** | cim-solution MR pipeline | Image tags in `values.yaml` resolve to existing registry artifacts | ❌ Not implemented |
| **G1.2** | cim-solution MR pipeline | Helm lint is a **blocking** gate (remove `\|\| true`) | ⚠️ Partial — `helm lint` runs but never fails |
| **G1.3** | cim-solution MR pipeline | If reporting ConfigMap/values changed → SQL upgrade script exists in `dbScripts/dbupdate/` | ❌ Not implemented |
| **G1.4** | cim-solution MR pipeline | If MongoDB-related values changed → transflux MR exists or DAG change is linked | ❌ Not implemented |
| **G1.5** | Microservice MR pipeline | Trivy passed, code review approved, CI green | ✅ Already enforced |
| **G1.6** | Jira workflow | `QA-Passed` status set before cim-solution MR can open | ❌ Not implemented |

### Layer 2: RMT Gates (Validate During Testing)

**Goal:** Prove the feature works in the integration environment.

| Gate | Location | What It Checks | Current State |
|------|----------|---------------|---------------|
| **G2.1** | Manual today / CI after T1-5 | Feature successfully deployed to RMT | ❌ Manual (Haroon/Junaid) |
| **G2.2** | QA process | QA records test results in structured Jira comment or custom field | ❌ Unstructured / manual |
| **G2.3** | Post-deploy (after T1-5) | Automated smoke test runs on RMT after Helm deploy | ❌ Not implemented |
| **G2.4** | RMT queue | Component reservation check — microservice not already under test | ❌ Manual only |

### Layer 3: Post-RMT Gates (Block Merge Until Proven)

**Goal:** No merge to `develop` or `RC` without proof of RMT pass.

| Gate | Location | What It Checks | Current State |
|------|----------|---------------|---------------|
| **G3.1** | GitLab MR-A (microservice → develop) | MR cannot merge until: MR-B merged AND Jira has `RMT-Passed` | ❌ Not implemented |
| **G3.2** | GitLab MR-B (cim-solution → RC) | MR cannot merge until: Jira has `RMT-Passed` | ❌ Not implemented |
| **G3.3** | GitLab MR-C (transflux → develop) | MR cannot merge until: Jira has `RMT-Passed` (for MongoDB features) | ❌ Not implemented |
| **G3.4** | Jira workflow | `Release-Ready` transition blocked until: all required MRs merged, docs linked, BAT/waiver present | ❌ Not implemented |

---

## 6. T1-1 Status: Paused — ScriptRunner Approach Rejected

> **Decision (June 2026):** After reviewing the Jira + GitLab integration options and ScriptRunner licensing requirements with Nabeel, the team has decided **not** to pursue a Jira workflow validator approach for Release-Ready DoD enforcement at this time. The effort and licensing cost outweigh the immediate value. T1-1 is **paused** — not abandoned — and will be revisited once the CI/CD pipeline is more mature.
>
> **Rationale:** The Release-Ready DoD is a process gate. Before we can enforce it with automation, the underlying pipeline (deploy → test → report) needs to exist. Building the enforcement layer before the validation layer creates a gate with nothing behind it. The team agreed to invest in the pipeline first, then add enforcement.

### What Would Un-Pause T1-1

| Condition | When |
|-----------|------|
| Automated regression runs reliably on every RMT deploy | After this initiative (§7 below) is stable |
| CI can prove "RMT tested" with an artifact | When `RMT-Regression-Passed` is trustworthy |
| Teams trust the automated result over manual QA sign-off | After 2–3 sprints of stable regression runs |

---

## 7. New Initiative: Post-RMT Playwright Regression Pipeline

> **Scope:** This is **not** a DoD enforcement gate. It is a **new CI/CD pipeline stage** that completes the RMT deploy-test loop: package charts → deploy to RMT → **run automated regression** → publish results. It directly replaces manual QA regression on RMT with an automated, repeatable, fast-feedback mechanism.

### Why Now?

| Reason | Explanation |
|--------|-------------|
| **Completes the CI/CD flow** | Today: package → deploy → *manual test*. Target: package → deploy → **automated test** |
| **No new licenses** | Uses existing GitLab CI + Playwright — no ScriptRunner or Forge app required |
| **Direct pain-point relief** | Eliminates hours of manual regression on RMT; QA validates automatically |
| **Fast feedback** | Regression results in minutes, not hours/days of waiting for manual QA |
| **Enables future DoD enforcement** | Once this produces a trusted `RMT-Regression-Passed` artifact, T1-1 can be un-paused |

### Pipeline Design

Extend the cim-solution MR pipeline (MR-B: `feature/* → Release-Candidate`) with a post-deploy stage:

```
cim-solution MR opened
        │
        ▼
┌───────────────────────────────────────┐
│  CI Stage 1: Package & Validate       │
│  • helm lint (blocking)               │
│  • image tag resolution check         │
│  • upgrade script presence (conditional)│
└───────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────┐
│  CI Stage 2: Deploy to RMT (T1-5)     │
│  • Native GitLab agent → Helm upgrade │
│  • Wait for rollout completion        │
└───────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────┐
│  CI Stage 3: Playwright Regression    │  ◄── NEW
│  • Trigger regression suite vs RMT    │
│  • Target: @regression tag            │
│  • Publish HTML + JUnit report        │
│  • Update Jira: RMT-Test-Result       │
└───────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────┐
│  CI Stage 4: Notify                   │
│  • Slack notification to stream team  │
│  • Update RC build status page (T1-4) │
│  • Publish test report link           │
└───────────────────────────────────────┘
```

**What the regression suite validates:**
1. Core chat flows (agent login, webchat routing, conversation lifecycle)
2. Supervisor flows (monitoring, whisper, barge-in)
3. Key integration surfaces (Keycloak auth, conversation manager APIs)

> **Scope note:** Umar Ikhlaq's Playwright framework (`qa-automation-playwright`) currently targets **70% chat use-case coverage**. This pipeline runs the `@regression` tag from that repo. Coverage expansion to 85%+ is tracked as T3-1.

**Result publishing (lightweight, no Jira integration):**
- Test results published as GitLab CI artifacts:
  - HTML report (browsable)
  - JUnit XML (`artifacts:reports:junit` for MR test result visualization)
- Slack notification to stream team channel with pass/fail summary + artifact link
- No Jira fields, no service accounts, no external API calls — everything lives in GitLab CI + Slack

**What this replaces:**
- Manual QA regression on RMT (hours, human-dependent, done ad-hoc)
- "Did someone test this on RMT?" tribal knowledge

**What this does NOT do (intentionally out of scope):**
- Block MR-A (microservice → `develop`) or MR-B (cim-solution → RC) merges
- Enforce Release-Ready DoD criteria
- Replace manual feature-level QA testing

### Pre-Requisites & Dependencies

| Prerequisite | Status | Owner | Notes |
|-------------|--------|-------|-------|
| T1-5 RMT deploy stage | ✅ **Ready** | Umar Naveed | Demo complete; runner already reaches RMT |
| Playwright regression suite (`qa-automation-playwright`) | ✅ 70% coverage | Umar Ikhlaq | Runtime: ~10 min for full suite |
| Playwright `BASE_URL` from env var | 🔄 5-min check | Umar Ikhlaq | `process.env.BASE_URL` in config |
| Test user credentials as GitLab CI variables | 🔄 Add to project settings | Haroon / RMT | Masked variables: `TEST_USER`, `TEST_PASS` |
| T1-4 Slack webhook | 🔄 In progress | Haroon / RMT | Reuse for regression result notification |

**What we DON'T need (deliberately excluded):**
- Jira custom fields — not needed for informational pipeline
- GitLab CI service account — no external API calls
- Headless mode verification — Playwright runs headless automatically when `CI=true`
- Network connectivity test — already proven by deployment pipeline

---

## 7. This Week's Deliverables

### Deliverable 1: Updated Objective DoD Criteria Document

| DoD Item | Objective Form | Can CI Enforce Today? | After T1-5? |
|----------|---------------|----------------------|-------------|
| Feature dev-tested on dev server | `Jira status = Dev-Complete` | Partial (self-reported) | Partial |
| QA-Passed on dev server | `Jira status = QA-Passed` + QA assignee ≠ developer | ✅ Jira API | ✅ Jira API |
| Code review passed | GitLab MR approval by Tech Lead | ✅ GitLab | ✅ GitLab |
| Trivy passed | GitLab CI job `security_scan` exit 0 | ✅ GitLab CI | ✅ GitLab CI |
| **Upgrade scripts present** | **DB-specific:**<br>• MySQL/MSSQL: file exists in `cim-solution/dbScripts/dbupdate/`<br>• MongoDB: transflux MR exists or DAG changed<br>• Version matches target release | ✅ **Can add to cim-solution CI** | ✅ cim-solution CI |
| cim-solution MR opened | MR exists: `feature/* → Release-Candidate` | ✅ GitLab API | ✅ GitLab API |
| transflux MR opened (conditional) | MR exists for MongoDB features | ✅ GitLab API | ✅ GitLab API |
| Image tags valid | Referenced `image:tag` exists in registry | ✅ CI job (can add) | ✅ CI job |
| **RMT deployed** | **Manual:** Haroon confirms deploy | ❌ Manual only | ✅ GitLab CD |
| **RMT tested** | **Automated:** Playwright regression passes on RMT deploy | ❌ Manual only | ✅ Playwright in CI (after T1-5) |
| **Parallel RMT safe** | No overlapping microservice in active RMT test | ❌ Manual (Haroon manages) | ⚠️ Needs component reservation logic |
| All required MRs merged | `merge_commit_sha` exists for MR-A, MR-B, (MR-C) | ✅ GitLab API | ✅ GitLab API |
| Docs updated | `Documentation Links` Jira field non-empty | Partial | Partial |
| BAT or waiver | `Jira status = BAT-Passed` OR comment `BAT-WAIVED:` | ✅ Jira API | ✅ Jira API |

### Deliverable 2: Playwright Regression Pipeline Stage — Specification

> **Not a gate.** This is an **informational pipeline stage** that produces a test result artifact. It does not block merges or enforce DoD. Blocking will be considered only after the suite is proven stable (see §6 for T1-1 un-pause conditions).

**Pipeline Stage:** `regression-test` — Post-RMT Playwright Regression + CI Result Artifact

**Specification:**
- **Trigger:** cim-solution MR-B (`feature/* → Release-Candidate`) pipeline reaches post-deploy stage
- **Execution logic:**
  1. Helm deploy to RMT completes successfully (T1-5 CD stage)
  2. Pipeline job checks out `qa-automation-playwright` repo at matching branch/tag
  3. Job runs Playwright regression suite against RMT environment URL:
     - `npx playwright test --grep @regression --reporter=junit,html`
  4. Test results published as GitLab CI artifacts:
     - HTML report (browsable)
     - JUnit XML (`artifacts:reports:junit`)
  5. Slack notification fires to stream team channel (T1-4 webhook) with pass/fail summary + artifact link
- **Non-blocking:** Stage failure does **not** block MR-B merge. The result is informational.
- **Audit trail:** All results in GitLab CI job logs + artifacts

**Rollout plan:**
- Week 1 (June 8–12): Document spec; confirm Playwright repo is CI-runnable (headless, env config)
- Week 2 (June 15–19): Add Jira custom fields; create GitLab CI service account
- Week 3 (June 22–26): Implement post-deploy stage in cim-solution pipeline (test project); pilot with CXAGENT team
- Week 4 (June 29–July 3): Validate stability on 3–5 real features; tune flaky tests; roll out to all stream teams

**When this becomes a gate (future):**
- After 2–3 sprints of stable runs with <5% flaky test rate
- After QA and stream leads confirm they trust the automated result over manual regression
- After T1-1 is un-paused and DoD enforcement is revisited

---

## 8. Open Questions & Risks (Final)

| Question | Status | Answer / Resolution |
|----------|--------|---------------------|
| What is "upgrade scripting for CI job"? | ✅ Answered | DB migration scripts: MySQL/MSSQL in `cim-solution/pre-deployment/reportingConnector/dbScripts/`, MongoDB DAGs in `transflux` (separate repo) |
| How does RMT deployment trigger? | ✅ Answered | **Manual today** (Haroon/Junaid). Will be automated by Umar Naveed's T1-5 (GitLab CD Pipeline) |
| Can two features be tested in parallel on RMT? | ✅ Answered | **Collision = same microservice**. Different microservices = parallel OK. |
| Where are MongoDB upgrade DAGs stored? | ✅ Answered | **Separate `transflux` repository** — NOT in cim-solution |
| Which GitLab project IDs are the microservice repos? | ✅ Resolved | Not needed for CI gate approach; pipeline operates within cim-solution + Playwright repos |
| Does Jira have a GitLab integration enabled? | ✅ Resolved | Not needed; pipeline uses REST API directly with service account |
| Who can create Jira workflow validators? | ✅ Resolved | **Not applicable** — approach changed to CI-driven; no Jira validators required |
| Can Playwright run headless against RMT from GitLab CI? | ✅ Confirmed | Playwright runs headless by default when `CI=true`. RMT reachability proven by deployment pipeline |
| How long does the full `@regression` suite take to run? | ✅ Confirmed | ~10 minutes for 70% coverage — acceptable for CI stage |
| Does the Playwright repo need CI-specific config? | ✅ Resolved | `BASE_URL`, credentials, queue name, customer name all read from env vars. Config complete. |
| What is the exact file path convention for DB upgrade scripts? | 🔄 Open | Naming pattern appears to be `*_db_update_script_{DBTYPE}_{from}_to_{to}.sql`. Confirm with Haroon/Nabeel |
| Are transflux DAGs versioned with the release? | 🔄 Open | Transflux is a separate repo — need to confirm branching model (does it mirror cim-solution release branches?) |
| How does Haroon track RMT component reservations today? | 🔄 Open | Document current manual process before designing automation |

---

## 9. Success Criteria for End of Week

- [ ] Updated objective DoD criteria document reflecting the new RMT-first + three-repo model
- [ ] Haroon + at least 2 stream tech leads have reviewed and agreed on the criteria
- [x] T1-1 approach decided: **Paused** — ScriptRunner rejected; DoD enforcement deferred
- [x] ScriptRunner approach rejected: licensing cost + effort not justified
- [x] T1-1 criteria document updated to reflect paused status
- [x] Playwright regression pipeline spec documented (§7)
- [x] Playwright repo confirmed to read `BASE_URL` from env var — verified in CI
- [x] Test user credentials added as masked GitLab CI variables
- [x] All 10 test suites pass in CI (4.7 minutes, headless, against mtt02)
- [ ] Merge `feature/ci-playwright-config` branch to `main`
- [ ] Integrate regression stage into cim-solution `.gitlab-ci.yml`
- [ ] Pilot with real RMT feature deploy (CXAGENT team)
- [ ] Validate stability over 3–5 real feature deployments

---

*Document produced: June 8, 2026 | Last updated: June 17, 2026*

*Revisions: Incorporates May 6, 2026 process update + stakeholder clarifications on upgrade scripts, RMT deployment, parallel testing, transflux repo location, and component definition. June 17 update: T1-1 paused (ScriptRunner rejected), T1-1b Playwright regression pipeline delivered — all 10 suites pass in CI (4.7 min).*
