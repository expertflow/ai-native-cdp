---
name: project-cicd-all-sites-multi-tenant
description: Every CX site (including devops and RMT) is multi-tenant, correcting the CD plan's §2.2 assumption that only mtt-labeled sites need a tenants/ subdirectory
metadata:
  type: project
---

**Fact:** As of July 2026, the stakeholder confirmed every site in the CD folder model is multi-tenant — not just genuinely-MTT-labeled ones. This corrects `cd-initiative-implementation-plan-2026-07.md` §2.2, which had assumed `devops` and `rmt` were single-tenant and could skip the `tenants/<tenant>/` subdirectory entirely (running Campaigns Studio/Conversation Studio/QM inline in the shared `expertflow` namespace instead). CRM-782's Jira description has been updated to reflect this; the plan doc itself has not yet been edited to match (§2.2's folder sketch, including the illustrative `sites/mtt/` example, is now stale).

**Ground-truthing detail:** RMT's real tenants are `mtt01`, `mtt02`, `mtt04` — confirmed via `tmp/transflux/config/tenants.yaml` and `tmp/helm-values/mtt0{1,2,4}-mtt-single-custom-values.yaml`. These tenants were previously assumed (in the plan's illustrative sketch) to belong to a separate hypothetical `sites/mtt/` site; they actually belong under `sites/rmt/tenants/`. `devops` has no real tenant yet — `sites/devops/tenants/devops01/` was created as an explicit placeholder (dummy id) pending a real one.

**Why this matters:** Changes the folder shape for CRM-782 (already implemented on branch `crm-782-sites-tenants-restructure` in `tmp/cx-environments-cd`) and will change scope for the tenant-level `pre-deploy-tenant`/`deploy-tenant` work (Phase 2 as of July 2026 — former Phase 4 was merged into Phase 2) — those jobs now need to run for devops and rmt too, not just a hypothetical MTT-only site.

**How to apply:** When working on any Phase 1–4 CD task that references "single-tenant sites" or assumes devops/rmt skip the tenant layer, treat that as outdated — every site gets a `tenants/` subdirectory. When the plan doc gets a broader revision pass, §2.2 needs updating to match. Related: [[project-cicd-objective-realignment]].
