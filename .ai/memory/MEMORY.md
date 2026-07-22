# AI-Native CDP — Project Memory

- [Google Chat Space](reference-google-chat.md) — Primary comms channel for this project; send updates here on PRs and changes (`https://chat.google.com/room/AAAAyffa-Sk`)
- [Progress Sync Workflow](project-progress-sync.md) — How to sync Jira (CRM) → priority list → git; trigger on demand when Jawad asks to update progress
- [Object Model Decoupling Epic](project-object-model-epic.md) — Nabeel's CIM-33653 (In-Progress); Tier-1 microservice contract decoupling to kill the lockstep deployment bottleneck
- [Integration Tests Scope Decision](feedback-integration-tests-scope.md) — Integration tests at microservice CI level only; NOT in cim-solution pipeline (Playwright regression already covers E2E)
- [Zaryab Scope Follow-up](project-zaryab-scope-followup.md) — Discussed 2026-07-01; Zaryab to post progress update on T1-6 security track 2026-07-02
- [CI/CD Objective Realignment](project-cicd-objective-realignment.md) — Re-anchored goal: MR→build→deploy→test→auto-revert→notify with no manual step; work runs as parallel tracks with convergence gates, not a strict sequence
- [CD: All Sites Are Multi-Tenant](project-cicd-all-sites-multi-tenant.md) — devops/RMT also get a tenants/ subdirectory, correcting plan §2.2; RMT's real tenants are mtt01/02/04
- [CD: transflux Onboarding](project-cicd-transflux-onboarding.md) — transflux chart pinned for RMT; explains the mtt03 FQDN and mtt04 ALEMBIC_DB_URL fields (not bugs); pre-deploy config wiring still missing
- [CD: Settled Control-Flow Design](project-cicd-manual-deploy-trigger.md) — bump:<site> is the manual go-live gate + sole writer of full site state (pins AND pre/post-deploy files); commits to cx-environments-cd main auto-deploy what they touched; rollback reverts state too — governs all Phase 2+ implementation
- [Artemis = systemservice, not helm](project-cx-artemis-deployment-mode.md) — official artemis deployment is a system service; don't pin the chart (RMT's pin needs review); vault runs in its own "vault" namespace; .deploy_env's namespace fall-through only fits app-tier charts
- [CRM-763 Bundle](project-cicd-crm-763-bundle.md) — Haroon committed CRM-782 + CRM-783 complete together with parent CRM-763, one delivery unit not staggered tasks
- [Dark Launch Drift](project-cicd-dark-launch-drift.md) — ad hoc customer-site deploys outside cx-environments-cd break later official-release upgrades; needs a sanctioned mechanism
