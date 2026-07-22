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
