---
stepsCompleted: [1, 2]
inputDocuments: ['.ai/memory/project-cicd-dark-launch-drift.md', '.ai/memory/project-cicd-manual-deploy-trigger.md']
session_topic: 'Dark launches — ad hoc, unannounced deployments at customer sites outside the formal release process — why they happen, and how to avoid the need for them / prevent the drift they cause for later official-release upgrades'
session_goals: 'Understand root causes of why the org resorts to dark launches; generate better structural alternatives that avoid the need for them or prevent the state drift they cause'
selected_approach: 'ai-recommended'
techniques_used: ['Five Whys', 'Role Playing', 'Assumption Reversal']
ideas_generated: []
context_file: '.ai/memory/project-cicd-dark-launch-drift.md'
---

# Brainstorming Session Results

**Facilitator:** EF
**Date:** 2026-07-22

## Session Overview

**Topic:** Dark launches — ad hoc, unannounced deployments at customer sites outside the formal release process — why they happen, and how to avoid the need for them / prevent the drift they cause for later official-release upgrades

**Goals:** Understand root causes of why the org resorts to dark launches; generate better structural alternatives that avoid the need for them or prevent the state drift they cause

### Context Guidance

Background from `.ai/memory/project-cicd-dark-launch-drift.md` and `.ai/memory/project-cicd-manual-deploy-trigger.md`: the settled CD design treats `cx-environments-cd` as the complete, self-contained state of every site (chart pins + pre/post-deploy files), written only via the manual `bump:<site>` gate. Dark launches bypass this mechanism, so a site's real state silently diverges from what `cx-environments-cd` records — and the next official-release `bump` fails or misbehaves because it's deploying against a state it doesn't know is wrong.

### Session Setup

Surfaced 2026-07-21 in a discussion with Nabeel. Jawad's term "dark launch" = an unannounced, ad hoc deploy/upgrade of one or more CX components at a specific customer site (typically for an urgent fix), done outside full release management.

## Technique Selection

**Approach:** AI-Recommended Techniques
**Analysis Context:** Dark launches / release-process bypass, with focus on root-cause understanding first, then structural alternatives

**Recommended Techniques:**

- **Five Whys:** Drill from the surface reason ("a customer needed an urgent fix") down to the structural driver — release cadence, fast-track lane, process friction, or organizational pressure.
- **Role Playing:** Look through the eyes of the people actually in the moment — field engineer, release manager, on-call approver, customer — to surface constraints Five Whys alone might miss.
- **Assumption Reversal:** Flip the root-cause assumptions to force concrete mechanism ideas, tied back to the existing `cx-environments-cd` / `bump:<site>` design.

**AI Rationale:** Session goals split cleanly into "why" (root cause, needs a deep/causal technique) and "better ways" (solution generation, needs a technique that can flip assumptions into mechanisms). Role Playing inserted between the two to widen the causal picture beyond a single linear chain before generating alternatives. Stakeholder tone is direct and technical, so structured/deep categories were favored over wild/theatrical ones.

## Technique Execution

### Technique 1: Five Whys

**Why #1 — Why does an engineer do a dark launch instead of the official release?**
Customer urgency: a bug can't wait for the next release train. The fix is scoped to one specific customer, so it doesn't need a public announcement. The fix is often iterative/hit-and-try (deploy, doesn't work, redeploy, iterate) before it's finalized — but whatever lands at the customer site must still be reconcilable with future production-grade upgrades.

**Why #2 — Why does release management take so long for a fix that only needs to reach one customer?**
The process requires a full regression cycle on the release candidate, product documentation, and a release announcement — applied uniformly regardless of blast radius.

**Why #3 — Why does a single-customer patch go through the same full process as a public release?**
Believed at first: no lighter-weight process had ever been defined for this case (just "a bug fix on an older release, usually a minor improvement").

**Reconfirmation against `docs/How_We_Work.md`:** A Patch/Hotfix process (§5) and a "Track 1: Customer-Driven / Maintenance-patch" release track (§1) *do* exist on paper. But:
- The Go/No-Go table (§4) requires a Patch Release to pass full regression, tested upgrade guide, and security scan — everything a minor/major release needs except performance testing. Barely lighter than a full release.
- §3.1: "Both tracks follow the same process from this point [RMT]" — including the **Release Announcement** stage. There is no documented concept of a deliberately unannounced, single-site patch.
- §5 itself flags "Dual-Codebase Risk — Process Needed": patching an older release for one customer is reactive/unplanned with no formal trigger process today.
- Nothing in the doc addresses the iterative hit-and-try delivery pattern at all.

**Root cause (bedrock):** The only formal release class is "full public release," calibrated for broad blast radius. Even the paper "patch track" inherits that full weight and always ends in a public announcement. There is no lighter-weight, unannounced, single-site, iteration-friendly path — so people bypass the process entirely rather than use a heavy process that doesn't fit the shape of the problem.

**Session decision:** Five Whys concluded here (bedrock reached); proceeding to Role Playing.

### Technique 2: Role Playing

**Persona 1 — Field/Support Engineer** (about to do the dark launch):
Purely time cost. The official patch path isn't distrusted, unknown, or hard to get approved for a single customer — it's simply too slow relative to the urgency, even though it technically exists on paper.

**Persona 2 — Release Manager (Haroon/RMT)** (owns the full process, enforces full regression + announcement on every release including patches):
Full regression is required on every release, including narrow single-customer patches, because RMT has **no way to scope regression down to just the affected component** — the tooling/process can only run the whole regression suite, not a targeted subset tied to blast radius.

**Persona 3 — Customer** (site receiving the urgent dark-launch fix):
Wants the fix delivered as soon as possible — but equally wants assurance that a future official-release upgrade at their site won't corrupt or destroy the change the dark launch made. Speed now, no amnesia later: the fix needs to be durably represented somewhere the next official release respects, not silently lost.

**Persona 4 — On-call/Go-No-Go Approver:**
Turns out this role **doesn't exist today** for dark launches. There is no formal approval gate — the domain expert/engineer making the fix self-certifies that their own change is safe to deploy to the customer site. Nothing forces that knowledge to travel forward to whoever runs the next official upgrade at that site. The real gap isn't a missing checklist item at approval time — it's the **missing handoff artifact**: nothing captures "this was dark-launched, here's what changed" in a form the next release cycle can see and reconcile against.

**Note:** The "stream-aligned dev who builds the iterative fix" persona is the same domain engineer already covered in Persona 1 — no separate perspective needed.

**Role Playing wrap-up:** Four perspectives converge on one shape: speed-vs-process tension (Engineer/Customer) is real but solvable if regression could scope to blast radius (RM) — and the actual structural hole is the absence of any approval/record mechanism (Approver) that would let a dark launch's state travel forward instead of silently drifting from `cx-environments-cd`.

### Technique 3: Assumption Reversal

**Reversal 1 — "Every release must go through the full regression suite."**
Component-level changes are usually deployable in isolation and are not too entangled — but cross-component regression risk can't be fully ruled out. Mitigation idea: scope regression to the changed component, paired with (a) notifying relevant domain owners it's happening, and (b) local-lab testing as a right-sized substitute for full regression rather than skipping validation altogether.

**Reversal 2 — "A release must be publicly announced."**
The announcement's real function isn't public disclosure — it's that RMT uses it to prepare accurate upgrade guides. When a dark launch isn't tracked, RMT builds the next upgrade guide against a baseline that's no longer true at that site, and the guide breaks in the field. So the announcement's job to preserve isn't "tell the world," it's "tell RMT so they build against actual state."

**Reversal 3 — "`cx-environments-cd` only changes via the formal `bump:<site>` gate."**
Key insight: **state capture and candidacy decision are two separate, parallel questions, not one sequential one.**
- *State capture* (mandatory, immediate, regardless of outcome): the instant a dark launch happens, the site's actual deployed state has diverged from `cx-environments-cd` — that's true whether or not the fix is ever formalized. Upgrade-guide branching (customer A has the dark launch, customer B doesn't) already exists the moment the dark launch happens, not only if/when it's accepted as a release candidate.
- *Candidacy decision* (separate, human judgment, can happen later): does this fix get folded into the next official release for everyone, or stay a permanent one-off for that site?
- Humans cannot be relied on to manually record state capture in the heat of an urgent fix — it needs to be a side effect of the deploy mechanism itself, not a separate remembered step.
- **Current constraint surfaced:** there is no single choke point today — dark launches are done ad hoc per engineer (no shared script/Helm command/CI job), so there's no natural place yet to attach automatic state capture.

**SESSION PAUSED HERE — 2026-07-22.** Assumption Reversal's three reversals complete. Next: decide whether to keep generating (e.g., what a shared dark-launch choke point / state-capture mechanism could look like) or move to convergence/synthesis.
