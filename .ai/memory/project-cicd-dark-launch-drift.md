---
name: project-cicd-dark-launch-drift
description: New CI/CD pain point (surfaced 2026-07-21, discussion with Nabeel) — ad hoc/unannounced customer-site deploys ("dark launches") bypass the release process and break later official-release upgrades
metadata:
  type: project
---

**Fact:** In a discussion with Nabeel on 2026-07-21, a gap was identified in the CI/CD initiative: engineers sometimes deploy or upgrade individual CX components at a customer site ad hoc — for a fix or similar — without going through full release management, and without publicly announcing it ("dark launch," Jawad's term). When a later *publicly announced* release is deployed/upgraded at that same customer site, the upgrade fails because the dark-launch change left the site in a state the official release didn't expect.

**Why this matters:** the settled CD design ([[project-cicd-manual-deploy-trigger]]) treats `cx-environments-cd` as "the complete, self-contained state of every site" — pins plus pre/post-deploy files, written solely via `bump:<site>`. A dark launch that changes a component's version or config *outside* that mechanism breaks the single-source-of-truth assumption the whole rollback/skip-if-unchanged/state-tracking design depends on. This is a real drift-source the current design doesn't yet account for.

Two needs identified, not yet designed:
1. A proper, sanctioned way to do a dark launch (unannounced, targeted fix at one customer) that still keeps `cx-environments-cd` (or equivalent) as truth for that site — rather than a side-channel change.
2. A way to make the next official-release upgrade succeed cleanly even when a site has diverged via a prior dark launch (detect/reconcile drift, not just fail).

**How to apply:** When next discussing CI/CD track scope ([[project-cicd-objective-realignment]]'s 5 tracks + convergence gates), this likely wants to be folded in as a track or sub-problem under Track A (core CD loop) or E (rollback/migration safety) rather than treated as a wholly separate initiative — the natural fix is "dark launch = a real bump through the same pipeline, just without the public announcement," not a parallel deploy path. Not yet decided; raise with Jawad before designing a mechanism.
