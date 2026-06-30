# How We Work
## Expertflow CX — Release & Delivery Process

|              |                                                                                 |
| ------------ | ------------------------------------------------------------------------------- |
| **Status**   | Version 0.2 — Validated by Haroon Ahmed and Nabeel Ahmad, February 2026         |
| **Owners**   | Haroon Ahmed (RMT), Nabeel Ahmad (Platform/DevOps)                              |
| **Audience** | All stream-aligned teams, ProjectDev, Platform team                             |
| **Purpose**  | Single reference for how a feature moves from development to production release |

> **Document Status** — This document reflects how the release and delivery process actually works at Expertflow as of February 2026. It was authored collaboratively by JB, Haroon, and Nabeel. Sections marked ⚠️ identify known gaps or risks requiring further action.

---

## 1. Overview: Two Release Tracks

Expertflow runs two distinct release planning tracks. Both tracks share the same downstream packaging, testing, and deployment process — but they have different triggers, owners, and planning rules.

| | **Track 1: Customer-Driven** | **Track 2: Product-Driven** |
|---|---|---|
| **Trigger** | Customer commitment or Ops request | Stream-aligned teams have a good set of features ready |
| **Release Type** | Maintenance / patch (may include new features) | Minor or major release |
| **Planner** | Project Manager + Ops (Amir Ayub) | Haroon + stream-aligned tech leads / POs |
| **Scope Decision** | PM and Ops decide; stream teams may not be consulted | Collaborative — Haroon, Nabeel, and stream leads agree |
| **Cut-off Rule** | Shortened to meet customer deadline; scope reduced without stream team input | Agreed scope and cut-off date set collaboratively |
| **Go/No-Go Authority** | PM + Ops + Haroon | Haroon + Nabeel |

> ⚠️ **Known Gap** — Three teams are consistently absent from Track 2 planning: Ehtasham's channels team (CXCHAN), Umar Tanveer's BI/WFM team (CXBI), and Quality Management. Since all three contribute components to every release, their absence from planning is a known source of late-discovered scope issues.

---

## 2. Development Process (Stream-Aligned Teams)

This process applies to all stream-aligned teams: CXAGENT, CXCHAN, CXVOICE, CXBI, and CXPLAT. It describes how a feature moves from Dev-Ready to Release-Ready.

### 2.1 Environment Setup

Before beginning development, each team must have a working environment. IaC scripts are available for most teams to deploy the latest minor release.

| **Team / Component** | **Environment Setup** | **Status** |
|---|---|---|
| Most stream teams | IaC script available for latest minor release (e.g. CX5.0.0). Manual upgrade required to reach latest patch (e.g. CX5.0.0 → CX5.0.3). | Available |
| Voice (CXVOICE) | IaC NOT available. Environment must be set up manually. | ⚠️ Manual — high effort |
| Channels (CXCHAN) | Needs enablement on IaC setup. | Needs enablement |
| BI / WFM (CXBI) | Servers not yet in IaC pool. | Needs enablement |

### 2.2 Create a Feature Branch

- Spawn from the `develop` branch.
- Follow the branch naming convention (see Branching Strategy doc).
- Ensure skeleton project tags are updated accordingly.
- Feature branches must be short-lived — avoid long-running branches.

### 2.3 Develop and Daily Commits

- Commit to the feature branch at least once per day.
- Include the Jira ID in every commit message. Format: `CIM-101: commit message`
- Document all configuration changes in Jira comments:
  - Deprecated or new ConfigMap attributes
  - Database schema (DDL) or data (DML) changes
  - Dependencies on other components or minimum version requirements
- Once development is complete, do a 5–10 minute shoulder check with PO/QA to validate acceptance criteria.
- Change Jira status to **QA-Ready** when ready for feature testing.

### 2.4 Feature Testing

Performed by the stream-aligned QA resource on the development server.

- **Cycle 1:** Execute test cases, report issues as Story-bugs or Bugs.
- **Cycle 2:** Verify fixes, update test case status.
- **Cycle 3 (if needed):** Final cycle — remaining open issues move to next sprint/release.
- **Scope:** Feature testing covers the new feature plus targeted regression in areas affected by the change only.
- Full regression is **NOT** performed at this stage — that is RMT's responsibility at release level.

### 2.5 Code Review and Merging to Develop

- Once QA-Passed, developer creates a merge request from feature branch to `develop`.
- Merge request must be reviewed by at least one reviewer other than the feature developer.
- Tech Lead approves code review.
- CI pipeline runs automated checks including Trivy security scan — must pass before merge.
- On merge: create a tag on the `develop` branch with the feature branch name.
- Close the feature branch on merge.
- Merge the feature branch in the skeleton project into `develop` and create a feature tag.

### 2.6 Definition of Done: Release-Ready

A feature or bug is considered **Release-Ready** when ALL of the following are true:

- [ ] Feature developed and tested on the stream-aligned team's development server.
- [ ] All relevant documents updated: deployment guide, configuration, API docs, feature docs.
- [ ] QA has verified the feature against acceptance criteria (QA-Passed).
- [ ] Code review approved by Tech Lead.
- [ ] Trivy scan passed in CI pipeline.
- [ ] No open sub-tasks remain.
- [ ] Dev-tested for both fresh deployment and upgrade path.
- [ ] Upgrade and downgrade scripts supplied.
- [ ] Feature branch merged to `develop` and closed.
- [ ] Feature tag created on `develop` branch in skeleton project.
- [ ] For major features and customer-facing changes: BAT completed by stream-aligned PO, or formally waived with documented reason in Jira.

> ✅ **BAT — Reinstated as Mandatory** — Business Acceptance Testing (BAT) is mandatory before marking a feature Release-Ready, for major features and any customer-facing changes. BAT is conducted by the PO of the stream-aligned team. The PO may waive BAT with a documented reason in Jira — but a waiver must be explicit, not simply skipped. BAT was previously agreed as mandatory but stopped being practiced because teams were not aware it was required. It is reinstated as of February 2026.

---

## 3. Release Process (RMT)

Once features are Release-Ready, the Release Management Team (RMT), led by Haroon Ahmed, takes over. Both release tracks follow the same process from this point.

### 3.1 Stage Overview

| **Stage** | **Description** | **Owner** | **Gate** |
|---|---|---|---|
| Plan Release | Define scope, timeline, and announce release candidate | PM + Ops + Haroon (Track 1) / Haroon + stream leads (Track 2) | Release plan announced on cx-release-announcements |
| Package Release | Integrate release-ready features into release skeleton, deploy to RMT server, smoke test | Haroon + Junaid (RMT) | Smoke test passed on RMT instance |
| Upgrade Guide | Document upgrade path from all supported previous releases, test upgrade | RMT | All supported upgrade paths tested |
| Deployment Testing | Verify deployment process works, smoke test post-deploy | RMT / QA | Deployment guide verified |
| Regression Testing | Full automated regression cycle on release candidate | RMT + QA | Regression suite passed |
| Performance Testing | Load/performance testing vs. previous release baseline. Minor and major releases only. | RMT / QA | No regressions vs. baseline |
| Security Testing | Trivy scan, OWASP checks, SonarQube, Burp Suite, Nmap | RMT / QA | No critical vulnerabilities |
| Release Merging | Merge develop into master, create production tags | RMT | Master branch updated with release tag |
| Release Notes | Communicate changes, enhancements, and fixes to stakeholders | Haroon / RMT | Release notes published |
| Release Announcement | Email to cx-release-announcements group | Haroon | Announcement sent |

### 3.2 Helm Chart Pipeline

Helm charts are managed and owned by RMT (Haroon and Junaid). Current manual steps in the pipeline:

- Manually edit chart values and tags for each release candidate.
- Coordinate chart sources across GitLab and GitHub — both are currently active sources. RMT confirms which applies per release component.

> ⚠️ **Planned Improvement** — Full automation of the Helm chart pipeline is a planned improvement. Until then, Haroon and Junaid are the joint owners of all manual chart steps. Neither is a single point of failure.

### 3.3 Incremental Build Process

Rather than waiting for all features to complete before packaging, RMT runs an incremental build — continuously integrating release-ready features as they become available.

- **Step 1 — Package:** Integrate each release-ready feature into the release candidate branch of the skeleton repository.
- **Step 2 — Upgrade Guide:** Document changes for each integrated feature in the relevant Jira task.
- **Step 3 — Deploy:** RMT prepares release candidate Helm charts and deploys to the RMT server.
- **Step 4 — Test:** Stream-aligned team runs feature testing and targeted regression. RMT runs automated regression cycle.
- **Step 5 — Final Validation:** Once all planned features are integrated, run a full smoke cycle and automated regression suite.

> ⚠️ **Known Issue** — Teams are not consistently following the Release-Ready criteria before handing off to RMT, which breaks the incremental build flow. Tech leads in each stream team are the first escalation point for this.

### 3.4 IAC Upgrades

- Back up MongoDB and PostgreSQL databases and upload to the IAC backup repository before any release.
- Update IAC pipeline if a new component or ConfigMap is added to or removed from the deployment.
- Always use the IAC pipeline to test the final deployment.

---

## 4. Go / No-Go Decision

Before any release is approved for production, the following criteria must all be satisfied:

| **Criterion** | **Patch Release** | **Minor / Major Release** |
|---|---|---|
| All Release-Ready features smoke tested | ✅ Required | ✅ Required |
| Regression suite fully passed | ✅ Required | ✅ Required |
| Upgrade guide tested for all supported versions | ✅ Required | ✅ Required |
| No critical or blocker bugs open | ✅ Required | ✅ Required |
| Security scan clean (Trivy) | ✅ Required | ✅ Required |
| Performance testing completed | — Not required | ✅ Required |

### 4.1 Go/No-Go Authority

| **Release Track** | **Decision Makers** |
|---|---|
| Track 1 — Customer-driven | PM + Ops + Haroon |
| Track 2 — Product-driven | Haroon + Nabeel |

### 4.2 When a Release is Not Ready

| **Situation** | **Action** |
|---|---|
| Critical / blocker bug open | Hold the release. Do not ship until resolved. |
| Major bug open | Ship with the issue logged as a known issue. Fix targeted in next patch release. |
| Minor bug open | Ship. Fix in next patch release. |
| Customer commitment at risk | Escalate to Ops + JB to decide. Do not unilaterally delay or ship. |

---

## 5. Patch / Hotfix Process

When a bug is identified in a production release, follow this process:

- Create a `hotfix` branch from the `master` branch for the affected production release tag.
- Branch naming convention: `CX4.4.5_b-CIM-420` (version + bug Jira ID).
- Develop and test the fix. Mark as Release-Ready following the standard DoD.
- Merge the `hotfix` branch into `develop`.
- If fixing the latest production tag: merge `hotfix` into `master` and create a new patch tag (e.g. `CX4.4.6`).
- If fixing an older version: add the patch release tag directly on the `hotfix` branch.

> ⚠️ **Dual-Codebase Risk — Process Needed** — When Ops requires a feature on an older release (e.g. a 4.10.x customer needs a 5.x feature), development must be built and packaged on both codebases. This is a reactive, unplanned cost with no formal process today for deciding when it is triggered or how the extra effort is accounted for in sprint planning. This should be formally defined.

---

## 6. Testing Responsibility Matrix

| **Testing Type** | **Who Does It** | **When** | **Trigger** |
|---|---|---|---|
| Feature testing | Stream-aligned QA | During development cycle | Feature reaches QA-Ready |
| Targeted regression | Stream-aligned QA | During feature testing | Feature touches existing functionality |
| Automated regression | RMT | Each incremental build | Feature merged to release candidate |
| Full regression | RMT + QA | Release testing stage | All features integrated |
| Fresh deployment test | RMT / QA | Release packaging | Release candidate ready |
| Upgrade testing | RMT / QA | Release packaging | Release candidate ready |
| Performance testing | RMT / QA | Release testing stage | Minor/major releases only |
| Security testing | RMT / QA | Release testing stage | All releases |

---

## 7. Rollback and Incident Response

> 🔴 **Critical Gap** — No formal rollback mechanism currently exists at Expertflow. The steps below describe the current informal practice. Building a proper rollback mechanism — particularly for data — is a high-priority engineering action item.

### 7.1 Current Rollback Capability

| **Capability** | **Current State** |
|---|---|
| Infrastructure (Helm) | Can redeploy previous Helm chart version. Reliable. |
| IaC State | Can revert IAC to previous state. Reliable. |
| Data (MongoDB / PostgreSQL) | No reliable rollback. DB backups exist but restoring them risks data loss for any transactions that occurred after the backup. High risk. |

### 7.2 Rollback Trigger

A rollback is initiated only for **critical or blocker** severity issues discovered in production. Haroon (RMT) leads the response informally today.

### 7.3 Current Incident Response Steps

In the absence of a formal procedure, the following steps are followed informally:

1. **Detect** — Critical issue identified in production.
2. **Notify** — Haroon notifies Ops immediately via Google Chat.
3. **Log** — Create a Jira incident ticket with full description and reproduction steps.
4. **Announce** — Post on Google Chat to inform relevant teams.
5. **Assess** — Determine if infrastructure rollback (Helm + IaC) is sufficient or if data is affected.
6. **Execute** — If proceeding with rollback, redeploy previous Helm chart version and revert IaC state.
7. **Verify** — Confirm system is stable on previous version.
8. **Communicate** — Update Ops and affected teams on status.

> ⚠️ **Data Rollback Risk** — If a release includes database migrations and a critical issue is found post-deployment, there is currently no safe way to fully restore the previous state without risking data loss. This must be addressed as a priority engineering item by Nabeel.

---

## 8. Team Enablement Status

| **Team** | **Gap** | **Action Owner** |
|---|---|---|
| CXCHAN (Ehtasham) | Not included in release planning. Needs IaC and code review enablement. | Haroon + Nabeel |
| CXBI / WFM (Umar) | Excluded from planning. Servers not in IaC pool. | Nabeel |
| CXVOICE (Ismail) | No IaC available — fully manual environment setup. | Nabeel (IaC build required) |
| QM Team | Excluded from release planning. | Haroon |
| All stream teams | Jira usage inconsistent — statuses not updated correctly. | Tech leads per team |

---

## 9. Open Action Items

| **Action Item** | **Owner** | **Priority** |
|---|---|---|
| Build formal rollback mechanism for data (MongoDB / PostgreSQL) | Nabeel | 🔴 Critical |
| Document formal data rollback procedure once mechanism exists | Nabeel + Haroon | 🔴 Critical |
| IaC setup for Voice team (CXVOICE) | Nabeel | 🟠 High |
| Automate Helm chart pipeline (remove manual steps) | Haroon + Junaid | 🟠 High |
| Resolve dual GitLab/GitHub chart source of truth | Nabeel | 🟠 High |
| Include CXCHAN, CXBI/WFM, QM in release planning | Haroon | 🟠 High |
| Define formal process for dual-codebase requests from Ops | JB + Haroon | 🟠 High |
| Bring CXBI/WFM servers into IaC pool | Nabeel | 🟡 Medium |
| Enable CXCHAN on IaC setup and code review process | Nabeel + Ehtasham | 🟡 Medium |
| Increase regression automation coverage beyond 40% | QA team | 🟡 Medium |
| Communicate testing scope boundary to all stream teams | Haroon + QA leads | 🟡 Medium |
| Communicate BAT reinstatement to all stream-aligned tech leads | Haroon | 🟡 Medium |

---

*Expertflow CX — How We Work v0.2 | February 2026*
