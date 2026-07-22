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
