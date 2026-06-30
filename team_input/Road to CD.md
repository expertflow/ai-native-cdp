# Road to CD — Meeting Notes

Source: https://expertflow-docs.atlassian.net/wiki/spaces/EF/pages/2024210454/Road+to+CD

## Core Problems

1. **Waiting in queue** — Team B can't start until Team A is done on RMT. Work is piling up instead of flowing.
2. **Everything on RMT is manual** — Building, deploying, testing all done manually. Every step needs someone to do it manually, which is slow and depends on specific people being available. (RMT/QA)
3. **RMT changes don't reach teams automatically** — When something changes, nobody gets notified or updated automatically; teams can easily miss it or fall behind. A proper mechanism for announcing RMT changes systematically is needed.

## Open Action Items

- [ ] **Nabeel** — Put together an integration testing guide for Java and Node; use AI tooling to speed this up. The guide should be practical enough for teams to follow without hand-holding. Status: No Progress.
- [ ] **Haroon** — Investigate how to automate feature deployment on RMT. The goal is to remove the manual request process and let deployments trigger without waiting on a specific person.
- [ ] **GitLab CD POC** — In progress; demo expected week of 2026-06-02.
- [x] **Umar** — Complete the automation testing cycle. Target: 70% coverage across chat use cases. ✓ Done.
- [ ] **Umar** — Script improvements from 70% coverage work; due 2026-06-04.
- [x] **Zaryab** — T1-6 security gate enforcement: approach finalized with team (2026-06-15). Pipeline implementation pending. See `priority-list-cicd-test-automation.md` T1-6.
- [ ] **Zaryab** — Evaluate Renovate vs Dependabot for automated dependency security updates (2026-06-19, in progress). See `docs/cicd_objectives_gaps.md` Gap 3.

## June 1, 2026 Meeting Points

- **Automated regression** following [QA Automation](https://docs.google.com/document/d/1ToghfQ9FzLjtO_GxvlXD3y13dLzo2saqChJ1QKmPvMg/edit?tab=t.0#heading=h.rrrtsavjciyf) — need to define objectives and milestones.
- **Voice deployment via CD** — voice is not part of the `cim-solution` skeleton:
  - Structural definition of components and how they are deployed
  - Release management mechanism for each component
  - Automated testing
- **Reporting and data-platform deployment** (Metabase + Airflow):
  - Need to define how they should be packaged and released
- Stream-aligned teams need to ensure proper onboarding for the release process.
- **Hot-fix** must be release-ready for the latest release as well.
- **Object Model (OM)** — version management and release strategy:
  - Need to either remove the OM dependency or automate the process (manual update takes too much time and effort)
- **VAPT process** on released and to-be-released:
  - Announce the policy
  - Automate the scanning and fixing — security hotfix
  - Related: [VAPT page 1](https://expertflow-docs.atlassian.net/wiki/spaces/EF/pages/1238893777), [VAPT page 2](https://expertflow-docs.atlassian.net/wiki/spaces/EF/pages/1941504045), [VAPT page 3](https://expertflow-docs.atlassian.net/wiki/x/AwD8cw), [VAPT page 4](https://expertflow-docs.atlassian.net/wiki/spaces/~55705835b89813bf8740fc84ac066dff1360ed/pages/1340506148)
- **Feature flag framework** finalization

## June 2026 — Security & Supply Chain (Zaryab / team alignment)

- **Security quality gates in Definition of Done** — Release-Ready DoD should include blocking Trivy, Grype, and SonarQube passes. DoD location reviewed (`priority-list` T1-1, `docs/How_We_Work.md`); enforcement still via self-report today.
- **T1-6 — Security scan enforcement** — Team agreed on phased rollout (`allow_failure: false` after CVE triage). Governance options documented in `security-audit/T1-6b-ci-cd-pipeline-governance-hardening-plan.md`.
- **Multi-layer packaging & centralized hardened dependency images** — **R&D complete; not continued.** Security Lead evaluated a platform where developers borrow Security-maintained hardened dependencies across all GitLab repos (Node, Java, Python). Team decision: too operationally complex (parallel feature branches, develop sync risk, regression attribution). **Alternative path:** blocking CI scans (T1-6, T2-10) + Renovate/Dependabot MRs to **develop** (Gap 3) + multi-stage Dockerfiles (build heavy / runtime light via T2-12). Details: `docs/cicd_objectives_gaps.md` Gap 9.

## Related Documents

- [Release retro](https://expertflow-docs.atlassian.net/wiki/spaces/EF/pages/505511940)
- [Release retro 2](https://expertflow-docs.atlassian.net/wiki/spaces/EF/pages/1945501729)
- [EF Internal page](https://expertflow-docs.atlassian.net/wiki/spaces/EF/pages/2000252)
- [Sessions with Awais on improving CI/CD](https://expertflow-docs.atlassian.net/wiki/spaces/EF/pages/1636827164)
