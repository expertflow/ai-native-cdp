# CIM VAPT Gap Analysis

> **Project:** CX / CIM group (Expertflow)
> **Scope:** All 109 non-archived projects under GitLab group `cim` (group id 50)
> **Date:** 2026-09-03
> **Source document:** `vapt-process-draft.md` (this folder)
> **Classification:** Critical — the enforcement gate described in the process is absent across the group
> **Status:** Analysis complete — awaiting the Phase 0 decisions in the remediation plan
> **Related:** `cim-vapt-remediation-plan.md`, `cicd-security-audit-and-hardening-guide.md`

## Method

Enumerated all 109 non-archived projects under group 50 via the GitLab REST API. Fetched `.gitlab-ci.yml` from each default branch. Read the shared job templates in `cim/ci-templates`. Read project merge-gate settings, approval rules, and protected-branch push and merge permissions. Sampled 267 merge requests merged since 2026-06-01 and inspected 71 of them at job level, including raw job traces. Checked 565 single-parent commits on protected default branches for merge-request association.

**Confidence note:** the 71-MR inspection is the two most recent MRs per active repo, not an exhaustive pass. The direction is unambiguous but the exact grandfathering scope needs the full baseline described in Action 6.

## Headline

The process document describes controls that are largely not enforced. The scan jobs exist, they are correctly written, and they genuinely fail on real vulnerabilities. Nothing downstream acts on that failure, and roughly half the group is never scanned at all.

| Measure | Value |
| --- | --- |
| Repos with a merge gate configured | 0 of 53 with CI |
| Projects with no CI on the default branch | 56 of 109 |
| Release images scanned by the pipeline | 0 |
| Scan evidence retention | about 1 hour |

Head pipeline status at the moment of merge, across 71 merged MRs inspected since 2026-06-01:

| Status | Count |
| --- | --- |
| failed, merged anyway | 59 |
| manual or skipped, never completed | 4 |
| passed | 8 |

---

## A. Enforcement: the gate does not exist

### A1. No merge gate is configured in any repo (Critical)

`only_allow_merge_if_pipeline_succeeds = false` in all 53 CIM repos that have CI. `approvals_before_merge = 0` and zero approval rules in all 53. Protected branches allow Maintainers to merge with no pipeline condition and no code-owner approval.

In nearly all 59 failures above, the failing job is `trivy_scanning_merge` with `allow_failure=false`. Job traces confirm real vulnerability exits from `trivy --exit-code 1 --severity HIGH --ignore-unfixed`, not infrastructure errors.

| Merge request | Merged | Pipeline | Failing scan job |
| --- | --- | --- | --- |
| `cim/cim-backend!93` | 2026-09-01 | failed | trivy + grype |
| `cim/media-routing-engine!277` | 2026-09-02 | failed | trivy + grype |
| `cim/campaigns!51` | 2026-08-21 | failed | trivy + sonarqube |
| `cim/unified-agent!454` | 2026-09-01 | failed | trivy + grype |
| `cim/realtime-reports-manager!224` | 2026-09-02 | failed | trivy + grype |

**The draft says:** Track 1 "is a permanent gate and runs for the entire lifetime of the product." Section 6: "MR scan failures block the merge automatically."

### A2. The MR Trivy template swallows every Critical finding (Critical)

`ci-templates/trivy-merge.yaml` ends the CRITICAL check with a trailing `|| echo "The previous command has some errors..Continuing"`, so a Critical finding cannot fail the job. Only the HIGH check propagates its exit code. Even with A1 fixed, Criticals would still pass through.

**The draft says:** "Critical and High findings with an available upstream fix block the merge."

### A3. SonarQube is not the co-gate the draft claims (High)

22 repos set `allow_failure: true` on the Sonar job. 8 repos have no Sonar job at all: `cim-rasa-bot`, `cim-solution`, `cim-solution-helm`, `cx-charts-cd`, `cx-data-platform`, `cx-environments-cd`, `efcx-compatibility`, `helm-connector`.

**The draft says:** "This runs alongside the existing mvn-scan (SonarQube) gate, both must pass."

### A4. Branch scans behave the opposite of the policy, inconsistently (Medium)

`.trivy_scan_template` sets `allow_failure: false` with `--exit-code 1`, so branch scans block. Then 13 repos override it back to `allow_failure: true` on the branch jobs, including `apple-store-connector`, `cx-data-platform`, `cx-load-hub`, `cx-tenant`, `google-playstore-connector`, `migration-utility`, `unified-admin`, `web-channel-manager`. Neither the template nor the repos match the policy, and the repos do not match each other.

**The draft says:** "These are informational only, they do not block the pipeline."

### A5. Maintainers can push directly to protected branches, bypassing the MR entirely (Medium, latent)

74 of the 75 protected branches in the group set `push_access_levels = Maintainers`. Only one is set to "No one". A Maintainer can therefore push straight to `master`, `develop` or `main` without opening an MR, so no scan runs at all. `cim/cim-solution` extends this to its release branches `CX-5.10.0`, `CX-5.11.0` and `CX-5.12.0`.

Measured exploitation is currently low. Of 565 single-parent commits on protected default branches since 2026-06-01, 527 arrived through a merge request and 38 did not:

| Repo | Branch | Direct pushes |
| --- | --- | --- |
| `cim/cx-environments-cd` | main | 25 |
| `cim/cx-charts-cd` | main | 12 |
| `cim/tylnex_sms_connector` | main | 1 |

This matters mainly because 37 of the 38 are in the two CD repos, which have no scanning of any kind (B2) and determine what is deployed. Fixing A1 without also fixing A5 leaves the bypass open for the day someone reaches for it.

Additionally, `allow_merge_on_skipped_pipeline` is unset on 52 of 53 repos. When the gate is enabled it must be explicitly set to false, otherwise the 4 manual-or-skipped cases seen in the sample would count as passing.

---

## B. Coverage: what is never scanned

### B1. 56 of 109 projects have no `.gitlab-ci.yml` on their default branch (Critical)

Empty repos, QA automation and the `load-test/*` mocks are fairly excluded. These 17 are actively developed and are not test doubles.

| Project | Last activity | State |
| --- | --- | --- |
| `cim/case-management` | 2026-09-02 | CI exists only on feature branch `1.0-f_CIM-33658` |
| `cim/transflux` | 2026-09-01 | 10 branches, no CI on any of them |
| `cim/amq-auth-plugin` | 2026-08-13 | 42 branches, no CI |
| `cim/cisco-teams-synchronizer` | 2026-06-19 | CI only on a feature branch |
| `cim/tenant-management-portal` | 2026-05-11 | CI only on a feature branch |
| `cim/tenant-config-service` | 2026-04-09 | No CI |
| `cim/bs4` | 2026-03-10 | No CI |
| `cim/workflow-builder/node-red-auth-keycloak` | 2026-02-10 | No CI |
| `cim/workflow-builder/node-red-context-redis` | 2025-12-23 | No CI |
| `cim/workflow-builder/node-red-resilient-hooks` | 2025-12-23 | No CI |
| `cim/workflow-builder/message-preprocessor` | 2025-12-23 | CI only on a feature branch |
| `cim/frontend-logger` | 2025-12-04 | CI only on a feature branch |
| `cim/web-form` | 2025-08-25 | No CI |
| `cim/customer-attributes-microfrontend` | 2025-08-20 | No CI |
| `cim/cx-historical-dashboards` | 2025-07-15 | No CI |
| `cim/cim-reports` | 2025-07-14 | No CI |
| `cim/list-management` | 2025-06-10 | CI only on a feature branch |

The "CI only on a feature branch" cases are worse than they look. The default branch has no pipeline at all, so nothing scans the branch the release is cut from. Separately, `twitter_mock_server` and `apple-store-mock-server` ship Dockerfiles with no CI and should be explicitly declared in or out of scope.

### B2. Six repos have CI but no Trivy or Grype (High)

`cim-solution`, `cim-solution-helm`, `cx-charts-cd`, `cx-environments-cd`, `efcx-compatibility`, `helm-connector`. `cim/cim-solution` is the single most active repo in the group and is the one that defines what actually gets deployed. Nothing in either track covers Helm charts, Kubernetes manifests, or IaC.

### B3. Branch scans exclude `master` and `develop` in 43 of 53 repos (Medium)

The scan jobs use `only: [/^.+_f-.+$/, /^.+_b-.+$/]`, so only feature and bug branches match.

**The draft says:** "Branch scans run on every commit to master, develop, feature branches and bug branches."

### B4. Branch scans usually never run at all (High)

In 35 of 53 repos `gitlab_build_branch` is `when: manual`, and the scan jobs declare `needs: [gitlab_build_branch]`. No manual click means no image, which means no scan. The shift-left claim in the draft's closing summary does not hold in practice.

### B5. The released artifact is never scanned (Critical)

`gitlab_publish` runs `only: tags`, and there is no scan job on the tag pipeline. Track 1 scans a throwaway MR build image and never the tagged release image that ships to customers. There is also no post-merge pipeline on `main` or `master`. This leaves the release triage snapshot, which the draft calls "the definitive finding list for the release", with no pipeline-produced input for the actual RC artifact.

---

## C. Scanner scope and supply chain

### C1. Container image scanning is the only source of findings (High)

No SCA against source manifests (`pom.xml`, `package.json`), no secret scanning anywhere in the group, no SAST beyond SonarQube, no IaC or misconfiguration scanning, no SBOM, no DAST. Leaked credentials in Git history are covered by neither track.

### C2. `--ignore-unfixed` makes the whole P4 class unobservable (High)

Every blocking Trivy invocation uses `--ignore-unfixed`. Section 4 defines P4 as "Low findings, or any severity with no upstream fix available", and the exception process requires documenting upstream fix status. Nothing in CI produces that list, so both the P4 category and the no-fix exception path have no data source at all.

### C3. Grype is piped unpinned from GitHub at job runtime (High)

`curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh`, from `main`, executed inside the pipeline. Unpinned, network dependent, and a supply-chain exposure inside the security control itself. It also fails in the wrong direction: `--fail-on critical` only, so Grype ignores High entirely.

### C4. Trivy version drift between the two templates (Medium)

`trivy.yaml` pins `0.64.0`, `trivy-merge.yaml` pins `0.63.0`, and `--offline-scan` is set on branch scans only. Branch and MR results are not comparable.

### C5. All 47 repos include `ci-templates` at a mutable ref (Medium)

`ref: 'main'`. Any edit to `trivy-merge.yaml` instantly changes the security posture of every pipeline in the group with no review, no version pin, and no record of which template version produced a given result. For an ISO 27001 control this is a change-management gap as much as a technical one.

---

## D. Evidence and audit trail

### D1. Scan artifacts expire about an hour after the job (Critical)

On `cim-backend` pipeline 66463, `trivy_scanning_merge` shows `artifacts_expire_at` roughly one hour after job start, and the only surviving file is `job.log`. The reports the job produced (`*-image-vulnerabilities-critical-trivy.txt`, `-high-trivy.txt`, `gl-container-scanning-report.json`) are already gone and return 404 on download.

### D2. The container_scanning report upload goes nowhere (High)

The instance is GitLab 17.4.2-ee, but `groups/50/vulnerability_findings` and `projects/:id/vulnerabilities` both return nothing, so the Security Dashboard is not available or not licensed. Section 7 requires per-component results "deduplicated by CVE ID, not raw Trivy row counts" across roughly 50 components. That can only be produced by hand today, from artifacts that have already expired.

### D3. The release triage snapshot has no owner, location, format or tooling (Medium)

The draft makes it the basis of the release gate and the definitive finding list for a release. Nothing produces it.

---

## E. Gaps inside the document itself

| Ref | Gap | Detail |
| --- | --- | --- |
| E1 | P1 and P2 share a 7-day deadline | The split changes enforcement wording only, not urgency, which makes the CVSS classification work in Section 4 largely ceremonial. Section 6 then introduces a third timing pattern the Section 5 table never mentions. |
| E2 | Three escalation authorities, two day counts | Section 5 says Day 8 to Component Lead and RM, Day 12 to Higher Management with CTO risk acceptance. The closing summary says VP Engineering is accountable beyond Day 10. Section 4 says the Security PO signs. |
| E3 | The RACI is incomplete | No row for release triage, monthly production scans, or risk acceptance. No column for VP Engineering or CTO, both of whom the document makes decision-makers. "Final Release Sign-off" lists two R's and no A, contradicting the note directly beneath it. |
| E4 | Track 2 has no schedule or capacity plan | No duration, no minimum test scope per release, no re-test SLA, no defined behaviour if the pentest cannot finish before Feature Freeze, and a single pentester as a single point of failure. |
| E5 | Monthly production scans have no tracker or miss-handling | No named Jira project, no evidence retention, no escalation if a month is skipped, and coverage of production images only, not the running cluster configuration. |
| E6 | No KEV or EPSS input | Prioritisation keys on attack vector plus exploit-type keywords parsed from the finding title. A CVE on the CISA KEV list gets no special handling, while any Low whose title says "RCE" triggers manual review. |
| E7 | No approved base image policy | P4 remediation depends on "the next scheduled base image update cycle", but no inventory, owner or cadence is defined. Repos like `bs4`, `busybox`, `kafka` and `load-balancer`, all unscanned, look like exactly that inventory. |
| E8 | No risk acceptance register | Exceptions live per Jira epic with no consolidated view, so nobody can answer "what risk are we currently carrying" without walking every epic. |
| E9 | The document has no owner, version, or review cadence | No effective date and no approver, which is itself an ISO 27001 documented-information gap. The Overview also references a diagram that is not present in the file. |

---

See `cim-vapt-remediation-plan.md` for the sequenced remediation plan.

Internal document. Not for external distribution.
