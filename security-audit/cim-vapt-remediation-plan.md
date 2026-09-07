# CIM VAPT Remediation: Action Items in Execution Order

> **Project:** CX / CIM group (Expertflow)
> **Scope:** Remediation sequence for the 22 gaps in `cim-vapt-gap-analysis.md`
> **Date:** 2026-09-03
> **Status:** Awaiting Phase 0 decisions
> **Related:** `cim-vapt-gap-analysis.md`, `vapt-process-draft.md`

Companion to `cim-vapt-gap-analysis.md`. That document says what is broken. This one says what to do about it, in the order the work actually has to happen.

## Why this order and not simply worst-first

A gap analysis naturally sorts by severity. Execution cannot. Five things about this particular situation force a different order, and each one is a place where the obvious plan would backfire.

**1. The gate cannot be switched on before anyone knows what it will stop.** Enabling `only_allow_merge_if_pipeline_succeeds` is a one-line API call, and it is tempting to lead with it because it is the single highest-value change. Doing so on Monday morning would have blocked 59 of the 71 merges inspected. The baseline of what is currently failing, and an agreed list of what is temporarily tolerated, is not preparation for the fix. It is part of the fix.

**2. Signing authority is a blocker, not a footnote.** The waiver list in step 1 requires someone to formally accept risk. The document currently names the Security PO (Section 4), the CTO (Section 5), and VP Engineering (closing summary) as the accepting authority, at Day 8, Day 10 and Day 12. The RACI gives the release sign-off row two R's and no A. Nobody can sign anything until that is resolved. The document fixes that look like cleanup work (E1, E2, E3) are therefore the very first item, not the last.

**3. Evidence retention is an operability prerequisite, not an audit nicety.** Scan artifacts currently expire about an hour after the job. The moment the gate starts blocking merges, the first question every developer asks is "what exactly failed", and the answer will already have been deleted. A gate whose evidence evaporates generates escalations to the Security Architect instead of self-service fixes. The one-line `expire_in` change has to land before enforcement, even though it reads like an audit item.

**4. Fix the templates before pinning them.** Pinning all 47 repos from `ref: 'main'` to a tag is correct, and doing it first would be actively harmful: every subsequent template fix would then require bumping 47 repos instead of one. Fix, tag, then pin. The order is the whole value.

**5. Shift-left before the gate, or the gate lands in the most expensive place.** Branch scans exist but almost never run, because the build they depend on is `when: manual` in 35 of 53 repos. If enforcement arrives before that is fixed, every finding surfaces for the first time at MR time, when the work is done and someone is waiting. Making branch scans actually run is what makes the gate tolerable.

The resulting shape is two tracks that run in parallel rather than one queue. **Visibility work is safe and can start immediately**: nothing it does blocks a developer, so it needs no negotiation. **Enforcement work is sequenced** and gated on decisions. Confusing the two is what usually stalls this kind of programme.

---

## Phase 0: Decisions that block everything else

No engineering. These are meetings and a document revision. Everything downstream waits on them, so they go first even though they look like the least urgent items in the analysis.

**1. Resolve the escalation and sign-off authority.** Pick one accepting authority for risk acceptance and one escalation clock. Fix the RACI so the release sign-off row has exactly one A. Differentiate the P1 and P2 deadlines, or merge the two priorities and admit they are one class.
*Closes:* E1, E2, E3. *Owner:* Security PO with CTO. *Blocks:* items 6, 7, 8.

**2. Name a document owner, version and review cadence for the VAPT process.** The document currently has no owner, no effective date and no approver, which an ISO 27001 auditor will find before they find anything technical.
*Closes:* E9. *Owner:* Security PO.

**3. Name a deputy for the Security Architect role.** The process depends on one person for pentesting, CVSS verification, the release triage snapshot, the monthly production scans, and the P4 base image cycle. That is a single point of failure across five named duties.
*Closes:* E4 in part. *Owner:* Engineering leadership.

**4. Decide the fate of Grype.** It overlaps Trivy almost entirely, fails only on Critical while Trivy fails on High, and installs itself by piping an unpinned script from GitHub `main` into the pipeline. The cheapest correct outcome is probably deletion rather than repair. Removing it also cuts pipeline time on 46 repos. Make this an explicit decision so item 11 knows what to build.
*Closes:* C3. *Owner:* Security Architect with DevOps.

**5. Declare the scanning scope boundary in writing.** Which repos are in scope: shipped services only, or also mock servers, load-test stubs, QA automation and the deployment repos. Seventeen actively developed repos are currently dark, and some may legitimately stay that way. Without a written boundary the coverage number can never be closed out.
*Closes:* prerequisite for B1. *Owner:* Security PO with Component Leads.

---

## Phase 1: Visibility, starting now and in parallel

None of this blocks a developer. It does not need Phase 0 to finish. Start it on day one, because everything in Phase 3 is negotiated against the numbers it produces.

**6. Produce the true baseline.** Scan the current released image of every in-scope component and produce one list deduplicated by CVE ID, with severity, CVSS vector, fixed version and affected component. The 71-MR sample in the gap analysis is two MRs per repo and is not sufficient to negotiate a waiver list.
*Feeds:* items 7, 8. *Owner:* Security Architect. *Depends on:* item 5.

**7. Triage the baseline into fix-now, fix-soon, and waive.** Apply the severity policy from Section 4 to the real list. This is where the P1 and P2 deadline decision from item 1 gets its first real exercise, and where you find out whether the policy is workable at this volume.
*Owner:* Security Architect with Component Leads. *Depends on:* items 1, 6.

**8. Get the waiver list formally accepted and time-boxed.** Written, in Jira, with compensating control and re-evaluation date, per Section 4. Every waiver needs an expiry, or the list becomes permanent.
*Owner:* whoever item 1 designated. *Depends on:* items 1, 7.

**9. Add scanning to the two CD repos, in report-only mode.** `cx-environments-cd` and `cx-charts-cd` have no scanning of any kind, and account for 37 of the 38 direct pushes to protected branches. They determine what reaches production. They are the sharpest single gap in the group and the fix is non-blocking, so it needs no waiver negotiation.
*Closes:* part of B2. *Owner:* DevOps.

**10. Onboard the 17 dark repos in report-only mode.** Determine first what each actually produces, since some may be libraries with no container image. Add scanning that reports and does not block. Visibility is cheap and uncontroversial; enforcement on these repos comes later with everything else.
*Closes:* B1. *Owner:* DevOps with Component Leads. *Depends on:* item 5.

---

## Phase 2: Make the gate operable

These are the changes that have to be in place before enforcement, or enforcement creates more problems than it solves. All of them are edits to `cim/ci-templates`.

**11. Fix the scan templates in one pass.** Remove the `|| echo "...Continuing"` that swallows Critical findings. Align the Trivy versions between `trivy.yaml` and `trivy-merge.yaml`. Either pin Grype to a released version and set `--fail-on high`, or remove it, per item 4. Decide once whether branch scans block or inform, and make the template say so.
*Closes:* A2, A4, C3, C4. *Owner:* DevOps with Security Architect. *Depends on:* item 4.

**12. Set `expire_in` on scan artifacts to the audit retention period.** Twelve months is the usual SOC 2 answer. This is one line and it is the difference between a gate developers can self-service and a gate that generates escalations.
*Closes:* D1. *Owner:* DevOps. *Must precede:* item 15.

**13. Make branch scans actually run.** Remove the dependency on a manual build, or add a scheduled branch scan. Fix the `only:` refs to include `master` and `develop`. Without this, enforcement pushes every discovery to the latest and most expensive possible moment.
*Closes:* B3, B4. *Owner:* DevOps. *Must precede:* item 15.

**14. Tag `ci-templates` and repoint the 47 repos at the tag.** Only after items 11, 12 and 13 have landed, otherwise each of them becomes a 47-repo change. Protect the templates repo at the same time.
*Closes:* C5. *Owner:* DevOps. *Depends on:* items 11, 12, 13.

---

## Phase 3: Close the gate

The payload. Everything above exists to make this survivable.

**15. Enable enforcement, all three settings together.** `only_allow_merge_if_pipeline_succeeds = true`, and `allow_merge_on_skipped_pipeline = false` (currently unset on 52 of 53 repos, and 4 of the 71 sampled MRs had a manual or skipped pipeline that would otherwise slip through), and set protected-branch `push_access_levels` to "No one" so the MR cannot be bypassed by a direct push. Any one of the three alone leaves a way around.
*Closes:* A1, A5. *Owner:* DevOps. *Depends on:* items 8, 11, 12, 13.

**16. Roll it out in waves, not all 53 at once.** Start with three or four repos that already pass, confirm the developer experience is workable, then expand. A group-wide flip with no pilot converts a security improvement into an outage.
*Owner:* DevOps with Release Manager.

**17. Turn on the SonarQube gate deliberately or drop the claim.** 22 repos have it non-blocking and 8 have no Sonar job at all. Either make it a real co-gate or stop describing it as one in the document. Leaving it ambiguous is worse than either choice.
*Closes:* A3. *Owner:* Component Leads with Security PO.

---

## Phase 4: Scan what actually ships

Enforcement on MRs still leaves the released artifact unscanned. This is the gap most likely to embarrass the process in an audit, because it is the one the customer receives.

**18. Add a scan job to the tag and publish pipeline.** The tagged release image is currently never scanned by anything. Add a post-merge scan on `master` and `develop` at the same time.
*Closes:* B5. *Owner:* DevOps.

**19. Add Trivy config and IaC scanning to `cim-solution` and the helm and `*-cd` repos.** `cim/cim-solution` is the most active repo in the group and defines what is deployed. Neither track covers Kubernetes manifests or chart values today.
*Closes:* rest of B2. *Owner:* DevOps with Security Architect.

**20. Add secret scanning across the group.** Leaked credentials in Git history are covered by neither track. This is a different class of finding from everything above and needs its own triage path, so it comes after the container pipeline is stable rather than bundled into it.
*Closes:* part of C1. *Owner:* Security Architect.

---

## Phase 5: Make the audit trail produce itself

The document promises SOC 2 and ISO 27001 continuity. Today that depends on a person assembling reports by hand from artifacts that have already expired.

**21. Automate the consolidated release report.** A job that collects each component's `gl-container-scanning-report.json`, deduplicates by CVE ID, and emits the report Section 7 describes. Assigning this to a person across roughly 50 components guarantees it is skipped under release pressure.
*Closes:* D2, D3. *Owner:* DevOps with Security PO. *Depends on:* item 12.

**22. Produce the no-fix-available list that P4 depends on.** Every blocking Trivy call uses `--ignore-unfixed`, so findings with no upstream fix are invisible. Section 4's P4 class and the no-fix exception path currently have no data source. Add a non-blocking scan without that flag whose only job is to produce this list.
*Closes:* C2. *Owner:* DevOps.

**23. Stand up the risk acceptance register.** One consolidated view of what risk is currently being carried, with expiry dates, rather than exceptions scattered across Jira epics.
*Closes:* E8. *Owner:* Security PO. *Depends on:* item 8, which populates it.

**24. Define the base image inventory and refresh cadence.** P4 remediation depends on a "scheduled base image update cycle" that has no owner, inventory or schedule. Repos like `bs4`, `busybox`, `kafka` and `load-balancer` look like the beginnings of that inventory and are themselves unscanned.
*Closes:* E7. *Owner:* DevOps with Security Architect.

---

## Phase 6: Close the document and enter steady state

**25. Give the monthly production scans a tracker and a miss-handling rule.** Named Jira project, retained evidence, and a defined escalation if a month is skipped. Extend coverage from production images to the running cluster configuration.
*Closes:* E5. *Owner:* Security Architect.

**26. Add KEV and EPSS to the prioritisation inputs.** Today a CVE on the CISA KEV list gets no special handling, while any Low finding whose title happens to contain "RCE" triggers a manual CVSS review. That is backwards.
*Closes:* E6. *Owner:* Security Architect.

**27. Define the Track 2 schedule, scope floor and re-test SLA.** Duration, minimum test scope per release, re-test turnaround, and what happens if the pentest cannot finish before Feature Freeze.
*Closes:* rest of E4. *Owner:* Security PO with Security Architect.

**28. Reissue the VAPT process document.** Fold in every decision from Phases 0 through 5, and add the missing architecture diagram the Overview refers to. The document should describe what the pipeline now does, verified against it, rather than what it was hoped it would do.
*Owner:* Security PO. *Depends on:* everything above.

---

## Verification checkpoints

| After | Check | Expected |
| --- | --- | --- |
| Item 15 | `glab api --hostname gitlab.expertflow.com projects/<id> \| jq .only_allow_merge_if_pipeline_succeeds` | `true` for all in-scope repos |
| Item 15 | Open a throwaway MR against a repo with a known High finding | Merge button actually disabled, not merely a red icon |
| Item 15 | Protected branch `push_access_levels` | "No one" on every protected branch |
| Item 10 | Re-run the group sweep: list projects, fetch `.gitlab-ci.yml` per default branch, grep for `trivy` | Unscanned list empty, or every entry explicitly waived under item 5 |
| Item 12 | `artifacts_expire_at` on a completed scan job | At least 12 months out, and last month's `gl-container-scanning-report.json` still downloadable |
| Item 18 | Inspect the most recent tag pipeline | Contains a scan job that ran against the published image |
| Item 21 | Cut a release | The consolidated report is generated without anyone assembling it by hand |

## Watch-outs

- **The waiver list is where this programme lives or dies.** If it is large, undated and unowned, the gate will be turned back off within a month. Every entry needs an expiry and a name against it.
- **Item 15 will be unpopular on the day.** Item 16's wave rollout and item 13's shift-left work are what buy the goodwill to keep it on. Cutting either to save time is a false economy.
- **The baseline in item 6 is a snapshot of a moving target.** New CVEs are published continuously. The draft already handles this correctly by gating against the snapshot rather than a live feed. Preserve that, and resist the temptation to re-baseline mid-release.
- **`cim/cim-solution` deserves separate attention throughout.** It is the most active repo in the group, it has no scanning, it has protected release branches that Maintainers can push to directly, and it determines what is deployed. It appears in items 9, 15 and 19 and should not be allowed to slip in any of them.
