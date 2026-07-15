---
name: project-cx-artemis-deployment-mode
description: Artemis (ActiveMQ) is officially deployed as a system service, NOT via its helm chart — don't pin it in any site's apps.yaml without revisiting; vault runs in its own "vault" namespace
metadata:
  type: project
---

**Fact (from Haroon, July 2026, during devops pipeline validation):** CX has two deployment options for artemis/ActiveMQ — a helm chart (exists in `cim-solution`'s `kubernetes/helm/activemq-artemis/`, chart name `artemis`) and a host-level **system service**. The system service is the **official** deployment mode. The helm chart pin was therefore removed from `sites/devops/apps.yaml`; **`sites/rmt/apps.yaml` still pins `artemis: "5.4.0-rc.1"` and needs the same review** before deploy:rmt runs for real (likely should be removed too, alongside the already-excluded qm/cisco-scheduler/eleveo).

Also dropped from the devops validation set (not needed there, still pinned on RMT): `redis-thirdparty`, `clamav`.

**Related namespace fact, same session:** `vault` deploys into its **own `vault` namespace**, not `ef-external` or `expertflow` — `.deploy_env` now maps it explicitly. Evidence: CD plan §3.6's cross-namespace secret-copy pairs use `vault:` as a source namespace, and the devops VM's leftover `vault-csi-provider-clusterrole` carries `release-namespace: vault` ownership annotations (the collision that exposed the bug: unmapped charts fall through to `expertflow` in `.deploy_env`'s case statement).

**How to apply:** If asked to add artemis to any site's `apps.yaml`, or if a chart-detection/bump flow surfaces artemis as publishable, flag the systemservice decision first — deploying the chart alongside the system service would double-run ActiveMQ. When RMT's apps.yaml gets its pre-go-live review, remove or confirm the artemis pin. When onboarding any new chart, check `.deploy_env`'s namespace case statement covers it — the default `expertflow` fall-through is only correct for application-tier charts. Related: [[project-cicd-manual-deploy-trigger]], [[project-cicd-transflux-onboarding]].
