# CI/CD Objective Realignment & Parallel Workplan

> **Prepared:** July 2026
> **Purpose:** Anchor document for the CI/CD initiative — restates the objective and pain points from first principles, defines the target near-term loop, and breaks the work into parallel tracks so nothing waits on a strict sequence unless a real technical dependency requires it.
> **Supersedes-in-spirit (not in detail):** the "Phased Rollout" in `cd-initiative-implementation-plan-2026-07.md` — that document's job-level detail still stands; this document reframes its phases as parallel tracks with explicit convergence gates, not a single sequential timeline.
> **Review cadence:** re-check this document whenever a new workstream is proposed for the CI/CD initiative — if it doesn't trace back to one of the three pillars below, question why it's in scope.

---

## 1. The Objective (Re-anchored)

Three pillars, stated in `ai_native_cdp_master_pipeline.md` and confirmed as the frame for this work:

1. **Increase release velocity and predictability**
2. **Reduce cognitive load on stream-aligned teams** — no tribal knowledge, no waiting on specific humans
3. **Make security non-optional** — a green pipeline must mean the build is actually safe

Underneath all three: the org is operating under real constraints (financial distress, remote work, low morale — per the master doc's own "Honest Context" appendix). That means **visible, felt wins matter as much as architectural correctness.** This initiative has to reduce real manual pain soon enough that people believe it's worth engaging with, not just be technically elegant.

## 2. The Pain Points (What We're Actually Fixing)

| Pain point | Evidence | What it costs |
|---|---|---|
| Everything on RMT is manual | `Road to CD.md`: *"Everything on RMT is manual"* | Every release depends on Haroon/Junaid specifically being available |
| RMT is a single shared instance | `Road to CD.md`: *"Team B can't start until Team A is done on RMT"* | Teams queue behind each other regardless of their own readiness |
| Environment config drift produces false defects | Problem inventory §2.3: *"works on dev, not on RMT; works on one RMT, not another"* | Engineering time burned debugging config mismatches disguised as product bugs |
| No automated notification when RMT changes | `Road to CD.md` #3 | Teams test against stale builds without knowing it |
| No safety net when a deploy breaks something | Master doc §11.6: DB rollback has "no safe path" for migrations; infra rollback didn't exist until this initiative | A bad deploy leaves RMT broken until a human notices and manually intervenes |
| Regression testing is disconnected from deploy | T1-1b exists but is manually triggered, not wired to deploy events | The automation that exists isn't actually saving anyone time yet |
| Release-Ready DoD is unenforced | T1-1, problem inventory §2.2: self-reported, no CI gate | Incomplete work reaches the shared bottleneck and compounds every problem above |
| Security scans are decorative | Security audit: `allow_failure: true` everywhere, score 19–28/100 | A green pipeline doesn't mean a safe build |

## 3. The Target Near-Term Loop

> A stream-aligned team raises an MR toward the release candidate on `cim-solution`. From that point: **build → deploy (with the correct environment-specific config) → test → if broken, automatically revert → notify** — with no manual step in between.

QA's role narrows to what only a human can do: **decide what needs testing, and write that test into `playwright-automation-script` before the MR is raised.** Once that's true, the mechanical act of running the suite and acting on its result should require no QA involvement at all.

This is exactly the shape already implied by `cicd-roadmap-current-state-and-target-architecture.md`'s own Phase 3 target — this document doesn't invent a new destination, it commits to reaching it without waiting on a single linear queue.

---

## 4. Parallel Workplan

Five tracks. **All five start now.** The only sequencing that matters is the four convergence gates in §5 — everything else proceeds independently.

### Track A — Core CD Loop (Deploy → Test → Rollback → Notify)

**Objective:** Automate the mechanical execution described in §3.
**Owner:** Haroon Ahmed (took over from Umar Naveed, July 2026)
**Reference:** `cd-initiative-implementation-plan-2026-07.md` (full job-level design)

> **Scope update (July 2026):** ground-truthing against the Confluence deployment guide revealed the real deployment model is namespace- and tenant-aware (three destinations: `ef-external`, `expertflow`, per-tenant), and that `cim-solution` serves two audiences (Expertflow's own sites and self-serve on-prem customers). This materially expands Track A — see `cd-initiative-implementation-plan-2026-07.md` §2 for the full picture. `transflux` (Expertflow ETL) is confirmed in scope here; its Helm chart already lives in `cim-solution` (no new work), and its config + `dbt_schema` are moving from the standalone `cim/transflux` repo into `cim-solution`'s pre-deployment resources. `cx-data-platform` (the image transflux's chart references) is explicitly **not** this track — tracked separately.

| Task | Ticket |
|---|---|
| Move committed credentials to CI/CD variables (hygiene fix — severity corrected from Critical to Low, see plan §3.2) | [CRM-781](https://expertflow-docs.atlassian.net/browse/CRM-781) |
| Restructure `cx-environments-cd` into `sites/<site>/[tenants/<tenant>/]` folders | [CRM-782](https://expertflow-docs.atlassian.net/browse/CRM-782) — foundational, do first |
| Create `sites/rmt/apps.yaml` + `sites/rmt/values.yaml` | [CRM-783](https://expertflow-docs.atlassian.net/browse/CRM-783) |
| Add inline rollback to `.deploy_env` (helm-upgrade-itself failures) | [CRM-784](https://expertflow-docs.atlassian.net/browse/CRM-784) (implements CRM-769) |
| Set `HELM_WAIT="true"` for RMT | [CRM-785](https://expertflow-docs.atlassian.net/browse/CRM-785) |
| Tighten `bump` token scope + protected-branch config | [CRM-786](https://expertflow-docs.atlassian.net/browse/CRM-786) |
| Wire `include: project:` regression job + `rollback:<site>` + `notify:<site>` (start with `rollback` as `when: manual`, per Gate 1 below) | CRM-763 / CRM-769 — Phase 2, ticket once Phase 1 lands |
| Add site-level `pre-deploy`/`post-deploy` stages | CRM-771 / CRM-772 |
| Add tenant-level `pre-deploy-tenant`/`deploy-tenant` (namespace creation, cross-namespace secret copying, `MTT-single`) | Not yet ticketed — Phase 4 |
| Move `cim/transflux`'s `config/` + `dbt_schema/` into `cim-solution`; archive `cim/transflux` | Not yet ticketed — Phase 4 |
| Re-enable and clean `cim-solution`'s detect/build/publish stages | Part of CRM-763 workstream |
| *(Later, explicitly deferred until the above is live)* GitHub Pages release-promotion automation | Not yet ticketed |
| *(Later, explicitly deferred until the above is live)* Vault migration for hardcoded config values | Not yet ticketed |

### Track B — Test Suite Reliability

**Objective:** Remove the flakiness sources that would make automatic rollback fire on false breaks.
**Owner:** Umar Ikhlaq
**Reference:** `current-cicd-test-automation-maturity-audit.md` findings #4, #6, #7 (serial-mode cascade, `networkidle` waits, hard sleeps, per-run data isolation)

| Task | Ticket |
|---|---|
| Remove file-wide serial mode; suites run independently | New — see §6 |
| Replace `networkidle` waits with element-level expectations | New — see §6 |
| Remove hard sleeps; add per-run data isolation (unique phone/name per run) | New — see §6 |

### Track C — Test Coverage Baseline & Governance

**Objective:** Make "coverage" mean something real, and make "tests exist before MR" an enforced expectation rather than a hope.
**Owner:** Umar Ikhlaq (baseline + cleanup), Haroon (governance gate)

| Task | Ticket |
|---|---|
| Organize full CX test inventory in Jira TM; re-baseline coverage % | CRM-766 |
| Remove/quarantine fake coverage: empty `test.step()` stubs, login-only stub suites | New — see §6 |
| Unblock NodeRED-dependent test automation | CRM-767 |
| Version Playwright suite against CX release tags | CRM-768 |
| Define and implement an MR gate requiring test coverage evidence before `cim-solution` merge | New — see §6 |
| Produce microservice-level integration testing guide | CRM-770 |

### Track D — Security Gates (Pillar 3, Parallel and Non-Negotiable)

**Objective:** Deliver on "security non-optional" — this pillar doesn't wait for Tracks A–C.
**Owner:** Zaryab Baloch + Haroon
**Reference:** `security-audit/ci-cd-security-plan-consolidated.md` (already phased, already owned)

| Task | Ticket |
|---|---|
| Phase 1 "Stop the Bleeding": `allow_failure: false` across all scanners, TruffleHog, build-time SCA | T1-6 (ticket to be created per that plan) |
| Phase 2 "Baseline Security": Semgrep/SpotBugs, Dockle, Checkov, coverage thresholds | T1-6 Phase 2 |

*This track is included here for visibility, not because this document changes its plan — it already has one. The point is to confirm it isn't deprioritized just because this conversation has been focused on Tracks A–C.*

### Track E — Data/Migration Rollback Safety

**Objective:** Cover the gap Track A's `helm rollback` cannot: releases that include a database migration.
**Owner:** Nabeel Ahmad (long-term design), Haroon (interim safeguard)

| Task | Ticket |
|---|---|
| Design session: what "rollback" means per database and migration type (long-term) | T1-3 (existing, sequenced 4th in Nabeel's stack) |
| **Interim safeguard:** any release with a DB migration gets flagged and routed through a manual-approval gate (not automatic rollback) until T1-3 lands | New — see §6 |

---

## 5. Convergence Gates (The Only Real Sequencing)

| Gate | Requires | Unlocks |
|---|---|---|
| **Gate 1** — Rollback flips from `when: manual` to `when: on_failure` | Track B substantially done | Fully automatic revert on regression failure |
| **Gate 2** — "Increase coverage" becomes the operating metric | Track C's Jira TM baseline (CRM-766) + stub cleanup done | Coverage numbers can be trusted quarter-over-quarter |
| **Gate 3** — "Tests required before MR" becomes enforced, not assumed | Track C's governance gate designed + agreed with stream leads | Closes the loop described in §3 for good |
| **Gate 4** — Full CX chart deployment coverage (not just Helm charts) | Track A's pre/post-deploy stages (CRM-771/772) | The deploy step in §3 covers all of what a release actually needs, not just charts |
| **Gate 5** — Migration-carrying releases get real rollback, not just a manual-approval workaround | Track E's long-term design (T1-3) | Removes the last asterisk on "if broken, revert" |

**No gate blocks starting any track.** Track A can wire the loop today with manual rollback as the safety valve; Track B, C, D, E all proceed independently and simply flip specific switches (Gate 1, 2, 3, 5) when their prerequisite work lands.

---

## 6. New Tickets Created (July 2026)

*Track A's Phase 1 is now ticketed too (created as work began, per the "ticket when actually starting" principle — not speculatively for later phases).*

| Track | Ticket | Owner |
|---|---|---|
| A | [CRM-781](https://expertflow-docs.atlassian.net/browse/CRM-781) — Move committed credentials to CI/CD variables | Haroon Ahmed |
| A | [CRM-782](https://expertflow-docs.atlassian.net/browse/CRM-782) — Restructure `cx-environments-cd` into `sites/<site>/[tenants/<tenant>/]` folders | Haroon Ahmed |
| A | [CRM-783](https://expertflow-docs.atlassian.net/browse/CRM-783) — Create `sites/rmt/apps.yaml` + `values.yaml` | Haroon Ahmed |
| A | [CRM-784](https://expertflow-docs.atlassian.net/browse/CRM-784) — Add inline rollback to `.deploy_env` | Haroon Ahmed |
| A | [CRM-785](https://expertflow-docs.atlassian.net/browse/CRM-785) — Set `HELM_WAIT="true"` for RMT | Haroon Ahmed |
| A | [CRM-786](https://expertflow-docs.atlassian.net/browse/CRM-786) — Tighten bump-mechanism token scope and protected-branch config | Haroon Ahmed |
| B | [CRM-777](https://expertflow-docs.atlassian.net/browse/CRM-777) — Harden Playwright suite reliability (serial-mode cascade, `networkidle`, hard sleeps, data isolation) — prerequisite for Gate 1 | Umar Ikhlaq |
| C | [CRM-778](https://expertflow-docs.atlassian.net/browse/CRM-778) — Remove fake/stub coverage in `CX-Chat-Cases.spec.js` (distinct from CRM-766, which is the Jira TM inventory) | Umar Ikhlaq |
| C | [CRM-779](https://expertflow-docs.atlassian.net/browse/CRM-779) — Define and enforce the "tests before MR" governance gate — prerequisite for Gate 3 | Haroon Ahmed |
| E | [CRM-780](https://expertflow-docs.atlassian.net/browse/CRM-780) — Interim manual-approval safeguard for migration-carrying releases until T1-3 lands | Nabeel Ahmad |

---

## 7. What "Done" Looks Like

- A stream team raises an MR → within minutes, they know whether it's deployed, tested, and (if broken) already reverted — with no person having clicked anything.
- QA's day-to-day work is writing and maintaining tests in `playwright-automation-script`, not clicking through regression manually.
- "Coverage" is a number people trust, tied to a real inventory, not inflated by stubs.
- A red security finding blocks a merge — every time, not "when someone reads the report."
- The one open asterisk — DB migrations — has a named interim safeguard, not silence.
