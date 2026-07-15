---
name: project-cicd-transflux-onboarding
description: transflux chart onboarded for RMT (CRM-783 addendum); explains two fields in its values file that look like bugs but aren't, and flags the still-missing pre-deploy config wiring
metadata:
  type: project
---

**Fact:** As of July 2026, `transflux` (chart name `transflux`, CX Data Platform / ETL) is pinned in `sites/rmt/apps.yaml` (`transflux: "5.4.0"`) with a real `sites/rmt/helm-values/transflux-custom-values.yaml`, committed on branch `CRM-782-sites-tenants-restructure` in `tmp/cx-environments-cd` (local commit, not pushed as of this writing). It was missed in the original CRM-783 pass because its source values lived outside `tmp/helm-values/` entirely — the source repo was later moved in as `tmp/helm-values/transflux/` (was `tmp/transflux/`).

**Two fields that look wrong on first read but are correct as-is (confirmed by Haroon):**
- `global.ingressRouter: "mtt03.expertflow.com"` — this is transflux's own dedicated FQDN for this environment. `mtt03` here is **not a tenant** (RMT's real tenants are mtt01/02/04, per [[project-cicd-all-sites-multi-tenant]]) — it's just the hostname chosen for the transflux endpoint itself.
- `ALEMBIC_DB_URL` pointing at `mtt04` — this is a manual, one-tenant-at-a-time scratch value. Alembic schema migrations are run by hand: an operator sshs into the transflux pod, sets this value to whichever tenant is currently being migrated, and runs the migration. It happened to be left at `mtt04` because that tenant was migrated last — it does not mean transflux only serves mtt04, and it isn't meant to represent all tenants simultaneously (unlike most other per-tenant config in this repo, which is expected to be complete/uniform across tenants).

**Known gap, deliberately deferred, not a bug:** transflux's actual runtime config (`config/tenants.yaml`, `dbt_schema/`, per-tenant source/target DB credentials) isn't mounted into the pod by any pipeline job yet. Per the plan doc §2.4/§3.6, that requires a `pre-deploy` stage that doesn't exist yet (`.gitlab-ci.yml` only has `stages: [deploy]` today) — it would move `config/`+`dbt_schema/` into `cim-solution` and regenerate ConfigMaps/Secrets on deploy, tied to CRM-771/772. So the transflux pod will come up and run, but without a working tenant config, until that pre-deploy job lands. Don't treat this as something CRM-783 should have covered — it's explicitly out of scope, later-phase work.

**How to apply:** If you see `mtt03` referenced anywhere near transflux, don't assume it's a stray/incorrect tenant reference — check whether it's this FQDN convention first. If asked to "fix" or "clean up" `ALEMBIC_DB_URL`, don't generalize it into a per-tenant loop without checking whether the manual ssh-and-run-migration workflow is still how this is actually done — that would change an intentional manual process. When picking up the pre-deploy stage work (§3.6, CRM-771/772), this is the ticket/file to extend for transflux specifically. Related: [[project-cicd-all-sites-multi-tenant]], [[project-cicd-objective-realignment]].
