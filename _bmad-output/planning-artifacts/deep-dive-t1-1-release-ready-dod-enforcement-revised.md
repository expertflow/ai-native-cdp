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

## 6. Recommended First CI Gate: Jira `Release-Ready` Transition Validator

Given the current state (manual RMT deployment, no CI smoke tests yet), the **highest-leverage first gate** is a **Jira workflow validator** on the `Release-Ready` status transition.

### Why This Gate?

| Reason | Explanation |
|--------|-------------|
| **Process-native** | Fits the new two/three-MR model without changing team behavior |
| **High frequency** | Every feature hits this path |
| **Catches the core risk** | Prevents untested or incomplete features from being marked Release-Ready |
| **Works with manual RMT** | Does not require T1-5 (automated CD) to be complete first |
| **Single point of enforcement** | Jira is the system of record all teams already use |

### Gate Design

**Mechanism:** Add a Jira workflow validator on the transition to `Release-Ready` that checks:

1. **Microservice MR merged:** GitLab API query confirms MR-A (microservice repo → `develop`) is merged
2. **cim-solution MR merged:** GitLab API query confirms MR-B (cim-solution → `Release-Candidate`) is merged
3. **MRs reference same Jira ticket:** Both MR descriptions contain the Jira ticket ID (`CIM-XXX`)
4. **transflux MR merged (conditional):** If the Jira ticket has the label `mongodb-change`, confirm MR-C (transflux → `develop`) is merged
5. **QA-Passed:** Jira status = `QA-Passed`
6. **BAT or waiver:** Jira status = `BAT-Passed` OR a comment contains `BAT-WAIVED:` with PO name

**Error messages:**
- If MR-A not merged: *"Cannot mark Release-Ready: Microservice MR not yet merged. Complete RMT testing and merge all required MRs first."*
- If MR-B not merged: *"Cannot mark Release-Ready: cim-solution MR not yet merged."*
- If MR-C required but not merged: *"Cannot mark Release-Ready: MongoDB changes require transflux MR to be merged."*
- If BAT missing: *"Cannot mark Release-Ready: BAT not completed. Complete BAT or document a waiver in Jira."*

**Implementation options:**
- **Option A (preferred):** Jira ScriptRunner (Adaptavist) — most flexible, requires license
- **Option B:** Jira built-in workflow validator with JQL + GitLab integration (if enabled)
- **Option C:** Custom Jira app or webhook service

### Phase 2 Gate (After T1-5 Completes)

Once Umar Naveed completes T1-5 (GitLab CD Pipeline) and RMT deployment is automated:

Replace the manual Jira checks with CI-validated artifacts:
- Add a `RMT-Test-Result` Jira custom field
- CI job post-RMT-deploy updates the field: `Passed` or `Failed`
- Jira validator checks `RMT-Test-Result = Passed` instead of just checking MR merge status

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
| **RMT tested** | **Manual:** QA updates Jira | ❌ Manual only | ⚠️ Still manual QA; can add smoke test |
| **Parallel RMT safe** | No overlapping microservice in active RMT test | ❌ Manual (Haroon manages) | ⚠️ Needs component reservation logic |
| All required MRs merged | `merge_commit_sha` exists for MR-A, MR-B, (MR-C) | ✅ GitLab API | ✅ GitLab API |
| Docs updated | `Documentation Links` Jira field non-empty | Partial | Partial |
| BAT or waiver | `Jira status = BAT-Passed` OR comment `BAT-WAIVED:` | ✅ Jira API | ✅ Jira API |

### Deliverable 2: First Gate Specification

**Gate:** `G3.4` — Jira `Release-Ready` transition validator

**Specification:**
- **Trigger:** User attempts to transition Jira ticket to `Release-Ready`
- **Validation logic:**
  1. Query GitLab API for MRs referencing this Jira ticket (`CIM-XXX`)
  2. Confirm at least 2 MRs exist and are merged:
     - One to a microservice repo's `develop` branch (MR-A)
     - One to `cim-solution`'s `Release-Candidate` branch (MR-B)
  3. If Jira ticket has label `mongodb-change`, confirm a 3rd MR to `transflux` `develop` is merged (MR-C)
  4. Confirm Jira status = `QA-Passed`
  5. Confirm `BAT-Passed` status OR a Jira comment matching `BAT-WAIVED: .* by .*`
- **Blocking:** If any check fails, transition is blocked with specific error message
- **Audit log:** Log all validation attempts (pass/fail) for compliance tracking

**Rollout plan:**
- Week 1 (June 8–12): Design validator, confirm Jira admin access, document spec
- Week 2 (June 15–19): Implement validator in Jira test project, pilot with CXAGENT team
- Week 3 (June 22–26): Roll out to all stream teams, train tech leads

---

## 8. Open Questions & Risks (Final)

| Question | Status | Answer / Resolution |
|----------|--------|---------------------|
| What is "upgrade scripting for CI job"? | ✅ Answered | DB migration scripts: MySQL/MSSQL in `cim-solution/pre-deployment/reportingConnector/dbScripts/`, MongoDB DAGs in `transflux` (separate repo) |
| How does RMT deployment trigger? | ✅ Answered | **Manual today** (Haroon/Junaid). Will be automated by Umar Naveed's T1-5 (GitLab CD Pipeline) |
| Can two features be tested in parallel on RMT? | ✅ Answered | **Collision = same microservice**. Different microservices = parallel OK. |
| Where are MongoDB upgrade DAGs stored? | ✅ Answered | **Separate `transflux` repository** — NOT in cim-solution |
| Which GitLab project IDs are the microservice repos? | 🔄 Open | Haroon to provide canonical list; store in Jira app config |
| Does Jira have a GitLab integration enabled? | 🔄 Open | Check if MR status is already visible in Jira; if not, need ScriptRunner or custom webhook |
| Who can create Jira workflow validators? | 🔄 Open | Requires Jira admin or Adaptavist ScriptRunner. Identify admin; estimate 1–2 days to implement |
| What is the exact file path convention for DB upgrade scripts? | 🔄 Open | Naming pattern appears to be `*_db_update_script_{DBTYPE}_{from}_to_{to}.sql`. Confirm with Haroon/Nabeel |
| Are transflux DAGs versioned with the release? | 🔄 Open | Transflux is a separate repo — need to confirm branching model (does it mirror cim-solution release branches?) |
| How does Haroon track RMT component reservations today? | 🔄 Open | Document current manual process before designing automation |

---

## 9. Success Criteria for End of Week

- [ ] Updated objective DoD criteria document reflecting the new RMT-first + three-repo model
- [ ] Haroon + at least 2 stream tech leads have reviewed and agreed on the criteria
- [ ] First gate specified: Jira `Release-Ready` transition validator design documented
- [ ] GitLab-Jira integration status confirmed (enabled/disabled + MR visibility)
- [ ] Jira admin contacted for workflow validator feasibility (ScriptRunner availability)
- [ ] Canonical list of microservice GitLab project IDs requested from Haroon
- [ ] Rollout plan: pilot with one stream team (recommend CXAGENT — most mature CI)

---

*Document produced: June 8, 2026 | Revisions: Incorporates May 6, 2026 process update + stakeholder clarifications on upgrade scripts, RMT deployment, parallel testing, transflux repo location, and component definition.*
