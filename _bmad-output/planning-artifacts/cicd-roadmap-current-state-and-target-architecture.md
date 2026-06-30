# CI/CD Roadmap: Current State, Target Architecture & Phased Migration

> **Status:** Living document — updated as initiatives progress  
> **Last updated:** June 30, 2026  
> **Authors:** Haroon (RMT), Nabeel, Umar Ikhlaq (Playwright), Umar Naveed (CD Pipeline)  
> **Related documents:**
> - `priority-list-cicd-test-automation.md` — Working priority list (T1–T4)
> - `deep-dive-t1-1-release-ready-dod-enforcement-revised.md` — DoD deep dive + T1-1b spec
> - `team_input/Road to CD.md` — Source strategic document
> - `docs/cicd_objectives_gaps.md` — Gap analysis

---

## 1. Executive Summary

This document captures where Expertflow's CI/CD pipeline stands today, where we want it to be, and the concrete phased steps to get there. It is written for both technical implementers and stakeholders who need to understand the dependencies between initiatives.

**The headline:** We are a Release Train organization (Model A) moving toward automated Environment Per MR (Model B). We will NOT jump to Trunk-Based Development (Model C) until our automated test trust and feature-flag infrastructure are mature. The path is deliberate, phased, and low-risk.

---

## 2. Current State (Where We Are)

### 2.1 Release Model: Release Candidate Branch (Release Train)

```
Stream team feature branch ──▶ MR to cim-solution RC branch
                                      │
                                      ▼
                            ┌─────────────────────┐
                            │  CI: Build charts   │
                            │  Publish to registry│
                            └─────────────────────┘
                                      │
                                      ▼
                            ┌─────────────────────┐
                            │  MANUAL: Deploy RMT │  ← Haroon / Junaid
                            │  MANUAL: Trigger QA │  ← QA team
                            │  MANUAL: Regression │  ← QA team (hours)
                            └─────────────────────┘
                                      │
                                      ▼
                            ┌─────────────────────┐
                            │  QA manual test     │  ← Hours to days
                            │  Human approval     │  ← Tech lead reviews
                            └─────────────────────┘
                                      │
                                      ▼
                         Merge microservice MR + cim-solution MR
                                      │
                                      ▼
                         Mark Jira → Release-Ready
```

### 2.2 What Works Today

| Component | Status | Owner | Notes |
|-----------|--------|-------|-------|
| **Helm chart build & publish** | ✅ Automated | `cim-solution` CI | Auto-bumps RC versions, publishes to GitLab Package Registry |
| **Trivy security scan** | ✅ Automated | GitLab CI | Runs on microservice MRs |
| **Code review gate** | ✅ Enforced | GitLab | Tech lead approval required |
| **Playwright regression suite** | ✅ CI-ready | Umar Ikhlaq | 10/10 suites pass in CI (4.7 min). JSON + HTML + JUnit reporters configured. |
| **Google Chat notification** | ✅ Working | This initiative | Card posted on every regression run with branch, author, results, links |

### 2.3 What Is Manual Today (The Friction)

| Step | Who | Time Cost | Pain |
|------|-----|-----------|------|
| Deploy to RMT | Haroon / Junaid | 30 min – hours | Context-switch, tribal knowledge, no rollback |
| Trigger regression | Human (QA or RMT) | 15 min – hours | Must remember, find pipeline, click button |
| Run regression tests | QA team | Hours | Manual click-through of core flows |
| Feature validation on RMT | QA team | Hours – days | Queued behind other features |
| Merge approval after RMT | Tech lead | Minutes – hours | Reviews results, approves merge |
| Notify stream teams of RC change | RMT | Ad-hoc | Teams don't know RC advanced |

**Result:** The critical path from "feature ready" to "Release-Ready" is ~80% manual. Only ~20% is automated (build + publish).

### 2.4 Active Initiatives In Flight

| Initiative | Status | Owner | Jira | Unblocks |
|-----------|--------|-------|------|----------|
| **T1-1b** Playwright regression in CI | ✅ Integrated & verified in cim-solution | Umar Ikhlaq + Haroon | CRM-765 (Resolved) | T1-1 (DoD enforcement), T1-4 (notification reuse) |
| **T1-5** GitLab CD auto-deploy to RMT | 🔄 In progress | Umar Naveed | CRM-763 | Regression auto-trigger, T1-4 closure |
| **T1-4** RC build notification | 🔄 In progress — Phase 1 mechanism reuses T1-1b's Google Chat webhook; blocked on CRM-763 for auto-trigger, canonical version display still unscoped | Haroon / RMT | — | Team sync |
| **T1-6** Object Model version management | 🔄 In progress | Nabeel + Ahsan | — | Independent releases (T2-8) |
| **T1-1** DoD enforcement | ⏸️ Paused | Haroon + tech leads | — | Revisit after T1-1b stable |

---

## 3. Target Architecture (Where We Want to Be)

### 3.1 The Vision: Automated Release Train with Quality Gates

```
Feature MR opened ──▶ Build ──▶ Pre-RMT gates ──▶ Deploy RMT ──▶ Regression ──▶ Notify
       │                                                                    │
       │                                                                    ▼
       │                                                           ┌──────────────┐
       │                                                           │  PASS → QA   │
       │                                                           │  validates   │
       │                                                           │  (manual)    │
       │                                                           └──────────────┘
       │                                                                    │
       │                                                                    ▼
       │                                                           ┌──────────────┐
       │                                                           │  QA PASS →   │
       │                                                           │  Auto-merge  │
       │                                                           │  to RC       │
       │                                                           └──────────────┘
       │
       └──────────────────────────────────────────────────────────────────────▶
                                                          FAIL → Fixed → Re-deploy
```

### 3.2 Target Pipeline (cim-solution)

```yaml
stages:
  - detect      # What changed?
  - build       # Package Helm charts
  - publish     # Push to registry
  - deploy      # Auto-deploy to RMT (T1-5)
  - test        # Playwright regression (T1-1b)
  - notify      # Google Chat card + QA assignment
```

### 3.3 What "Good" Looks Like

| Capability | Current | Target | Initiative |
|-----------|---------|--------|------------|
| Deploy to RMT | Manual | Auto on MR | T1-5 (CRM-763) |
| Regression trigger | Manual | Auto after deploy | T1-1b |
| Regression execution | Manual QA | Automated Playwright (4.7 min) | T1-1b ✅ |
| Result visibility | Ad-hoc | Google Chat card + artifact link | T1-1b ✅ |
| RC change notification | Manual | Auto post-deploy | T1-4 |
| Merge gating | Human judgment | Automated (future) | T1-1 (paused) |
| Rollback on failure | None | One-click / auto (future) | Post-T1-5 |
| Stream env sync | Manual pull | Auto update (future) | Post-T1-5 |

---

## 4. What's Keeping Us Away (Blockers & Gaps)

### 4.1 Immediate Blockers (This Quarter)

| Blocker | Why It Blocks | Resolution Path |
|---------|--------------|-----------------|
| **RMT deploy is manual** | Regression can't auto-trigger if deploy isn't automated | T1-5 (Umer) — in progress |
| **No rollback mechanism** | If regression fails, RMT stays broken; teams can't safely auto-deploy | Post-T1-5 — design rollback job |
| **Test flakiness unproven** | We don't yet know if Playwright is stable enough to gate merges | Run on real features for 2–3 sprints |
| **QA doesn't yet trust automation** | Manual QA is still the source of truth for "RMT tested" | Build trust through consistent green runs |

### 4.2 Structural Gaps (Next 1–2 Quarters)

| Gap | Impact | When We Address It |
|-----|--------|-------------------|
| **No feature flag framework** | Can't safely merge incomplete work; prevents trunk-based dev | T3-2 — after CD is stable |
| **OM versioning incomplete** | Stream teams can't release independently | T1-6 — Nabeel's current focus |
| **No integration tests** | Defects escape to RMT that unit tests miss | T2-3 — after OM done |
| **Data rollback (DB) doesn't exist** | Schema migrations carry unrecoverable risk | T1-3 — Nabeel's last priority |
| **IaC gaps (CXVOICE, CXBI)** | Parts of the platform can't be deployed automatically | T2-1 — parallel track |

### 4.3 Risk: The "One Branch, Multiple Environments" Question

When Umer's CD job is complete, stream-aligned teams may want to deploy to their own environments (not just RMT). This creates a decision:

| Option | Behavior | Risk |
|--------|----------|------|
| **A. RMT only gets regression** | QA and stream envs deploy without automated test | Stream envs may diverge from RMT config |
| **B. Every env gets regression** | 5 envs × 5 min = 25 min + compute cost | Notification fatigue, pipeline bloat |
| **C. Env-specific test subsets** | RMT = full regression; QA = smoke; dev = none | More config to maintain |

**Current position:** Start with **Option A**. RMT is the integration environment — it is the only place where the full stack is deployed and the only place where full regression provides signal. Stream team environments are for their own component testing, not full-stack validation. We can revisit Option C later.

---

## 5. Phased Roadmap (How We Get There)

### Phase 1: Now — Prove the Loop (June–July 2026)

**Goal:** Show that automated regression + notification works end-to-end on real RMT deploys.

```
build ──▶ publish ──▶ [MANUAL] deploy ──▶ [MANUAL] regression ──▶ notify
```

**Deliverables:**
- ✅ Playwright repo CI-ready with JSON reporter + Google Chat card
- ✅ `cim-solution` `regression-test` job clones Playwright repo and runs tests
- ✅ `notify-google-chat` job posts card with results + artifact links
- 🔄 Pilot with 3–5 real RMT feature deployments
- 🔄 Measure flaky rate (target: <5%)

**Key decision:** Keep `regression-test` as `when: manual` until T1-5 (auto-deploy) is ready.

### Phase 2: Auto-Deploy + Auto-Test (August–September 2026)

**Goal:** Remove the manual deploy and manual regression trigger.

```
build ──▶ publish ──▶ deploy ──▶ regression ──▶ notify
```

**Dependencies:** T1-5 (CRM-763) must be complete.

**Deliverables:**
- `deploy-rmt` job runs automatically on RC branch commits
- `regression-test` changes from `when: manual` to `when: on_success` + `needs: [deploy-rmt]`
- `notify` job runs `when: always` — posts green card on pass, red card on fail
- QA only starts validation when regression passes (no more testing broken builds)

**New capability introduced:** The pipeline now produces a **trusted artifact**: `RMT-Regression-Passed`. This is the signal that unblocks T1-1.

### Phase 3: Gate the Merge (October–November 2026)

**Goal:** Prevent merging to RC until RMT regression + QA validation pass.

```
MR opened ──▶ build ──▶ deploy ──▶ regression ──▶ QA manual ──▶ [auto-merge to RC]
```

**Dependencies:**
- 2–3 sprints of stable regression runs (<5% flaky)
- QA confirms they trust automated result over manual regression
- T1-1 un-paused

**Deliverables:**
- MR-B (cim-solution → RC) is blocked from merging until regression passes
- QA manual testing becomes a validation step, not a regression step
- Release-Ready DoD enforcement revisited (T1-1)

### Phase 4: Stream Environment Auto-Sync (2027)

**Goal:** Stream-aligned team environments automatically update with the latest RC.

```
RC branch updated ──▶ RMT deploy ──▶ regression pass ──▶ notify ──▶ auto-update team envs
```

**Dependencies:** T1-4 notification + stable CD pipeline.

**Deliverables:**
- Stream team envs auto-deploy from RC branch after RMT regression passes
- Teams always develop against latest validated code
- "Sync with RMT" becomes a non-event

### Phase 5: Decomposition + Independent Releases (2027+)

**Goal:** Stream teams release their own components independently.

**Dependencies:** T1-6 (OM versioning), T2-8 (package decomposition), T3-2 (feature flags).

**This is the strategic end state** — but it requires all preceding phases to be stable.

---

## 6. How This Plan Helps Us Move Forward

### 6.1 It Sequences Risk Correctly

We do NOT try to automate everything at once. Each phase validates the next:
1. **Phase 1** proves the test suite is stable → justifies auto-triggering it
2. **Phase 2** proves auto-deploy + auto-test works → justifies making it a gate
3. **Phase 3** proves the gate is trustworthy → justifies enforcing DoD
4. **Phase 4** proves teams can consume RC automatically → justifies independent releases

### 6.2 It Protects RMT Stability

The manual gates (deploy trigger, regression trigger) remain until the automated alternative is proven. We never remove a safety net before the replacement is trustworthy.

### 6.3 It Builds Team Trust Incrementally

QA and stream leads see the automated regression working on real features before they're asked to trust it. The Google Chat card makes results visible and transparent. By the time we propose making it a merge gate, the team has already seen 2–3 sprints of reliable results.

### 6.4 It Aligns with Existing Initiatives

| Initiative | Fits Into Phase | How |
|-----------|----------------|-----|
| T1-5 (CD pipeline) | Phase 2 | Provides the `deploy-rmt` job that triggers regression |
| T1-1b (Playwright) | Phase 1–2 | The test stage that runs after deploy |
| T1-4 (Notification) | Phase 1 | The `notify` stage that posts to Google Chat |
| T1-1 (DoD enforcement) | Phase 3 | Revisited after regression artifact is trusted |
| T1-6 (OM versioning) | Phase 5 | Enables independent component releases |

---

## 7. Decision Log

| Date | Decision | Rationale | Reversible? |
|------|----------|-----------|-------------|
| June 2026 | Use Google Chat (not Slack) | Expertflow uses Google Workspace; no Slack workspace | ❌ No — committed |
| June 2026 | Single combined card (deploy + regression) | Simpler, one message per pipeline run | ✅ Yes — can split later |
| June 2026 | Notify orchestrator job (not inline) | Future-proof for rollback job insertion | ✅ Yes — can refactor |
| June 2026 | Plain text author mention (not user mapping) | Zero maintenance; keyword alerts work now | ✅ Yes — can add mapping later |
| June 2026 | Only RMT gets full regression (not QA/stream envs) | RMT is the integration environment; others are component-scoped | ✅ Yes — can expand later |
| June 2026 | Keep T1-1 paused | ScriptRunner rejected; enforce after pipeline produces trusted artifact | ✅ Yes — will un-pause in Phase 3 |
| June 2026 | Stay inline (not `include` or trigger) | Simpler for current phase; migrate to `include` when stable | ✅ Yes — can refactor later |

---

## 8. Open Questions

| Question | Who Should Answer | Impact |
|----------|-----------------|--------|
| How will Umer structure the CD job? Separate per-env jobs or parameterized? | Umar Naveed | Determines how `regression-test` job uses `needs:` |
| Should QA ever run regression automatically, or is manual QA validation separate? | Hassan (QA Lead) | Determines if QA env gets a test stage |
| Do stream teams deploy from `cim-solution` branches or their own repos? | Jawad + stream leads | Determines if `cim-solution` pipeline runs for team envs |
| Should regression failures notify a different Google Chat space per team? | Haroon + stream leads | Determines notification routing complexity |
| When do we make regression results blocking for MR merge? | Jawad + Haroon + POs | Determines T1-1 un-pause timing |

---

## 9. Success Metrics

| Phase | Metric | Target | Measurement |
|-------|--------|--------|-------------|
| Phase 1 | Regression runtime | < 5 min | GitLab job duration |
| Phase 1 | Flaky test rate | < 5% | Failed runs due to non-code issues / total runs |
| Phase 1 | Card delivery | 100% | Every regression run posts to Google Chat |
| Phase 2 | RMT deploy frequency | 2× per day | Number of auto-deploys |
| Phase 2 | Time from merge to regression result | < 10 min | Deploy + regression total duration |
| Phase 3 | Manual QA regression time | 0 hrs | QA no longer runs manual regression |
| Phase 3 | MR merge confidence | > 90% | Surveys: "Do you trust the regression result?" |
| Phase 4 | Stream env freshness | < 4 hrs lag behind RC | Time between RC merge and team env update |

---

*Document created: June 22, 2026*
*Next review: After T1-5 (CRM-763) demo — target July 2026*
