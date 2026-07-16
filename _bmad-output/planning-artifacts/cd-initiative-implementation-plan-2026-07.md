# CD Initiative Implementation Plan — cim-solution + cx-environments-cd

> **Prepared:** July 2026
> **Scope:** T1-5 (CRM-763) — turning Umer Naveed's CD POC into the real, environment-aware deployment pipeline
> **Inputs:** `tmp/cx-charts-cd/.gitlab-ci.yml`, `tmp/cx-environments-cd/.gitlab-ci.yml` + `devops/apps.yaml` + `devops/values.yaml` (POC repos), `CX-5.4.0/.gitlab-ci.yml` (real cim-solution skeleton), `priority-list-cicd-test-automation.md`, `cicd-roadmap-current-state-and-target-architecture.md`, `current-cicd-test-automation-maturity-audit.md`, Confluence "(5.4.0) Deployment Guide" and "(5.4.0) Data Platform Deployment Guide", `tmp/helm-values/*`, `tmp/transflux/*`
> **See also:** `cicd-objective-realignment-and-parallel-workplan-2026-07.md` — reframes this plan's "Phased Rollout" (§6) as parallel tracks with explicit convergence gates. `cicd-pipeline-diagram-2026-07.html` — visual rendering of the core loop in §4.
> **Note on Jira:** per stakeholder instruction, no new Jira tickets are created from this revision — work items identified below are documented in §7 and will be ticketed when work on them actually begins.

> **Decisions confirmed with stakeholder (July 2026):**
> 1. `cx-charts-cd` was a throwaway POC only. The real chart source of truth stays **cim-solution** (`CX-5.4.0`). `cx-environments-cd` is real and becomes the deploy-side repo.
> 2. Regression integration: `cx-environments-cd` imports QA's test job definition via `include: project:` and runs it locally (§4, §5).
> 3. Environments (devops, RMT, future team envs) are **independent** — no promotion ordering or gating between them.
> 4. **GitLab Agent for Kubernetes is deferred** (§3.1) — static `KUBE_URL`/`KUBE_TOKEN` stays for now; revisit once a GitLab admin confirms KAS availability.
> 5. `playwright-automation-script`'s own pipeline is untouched — standalone QA harness, not the aggregate-notify owner.
> 6. `CX-5.4.0`'s dormant `regression-test`/`notify-google-chat` jobs are superseded by §4's design and should be removed once the new pipeline is live.
> 7. **The real deployment model is namespace- and tenant-aware, and `cim-solution` serves two audiences** (Expertflow's own sites and self-serve on-prem customers) — see new §2, which materially reshapes the folder model in §3 and §4 below.
> 8. **`transflux` (Expertflow ETL / Data Platform) is in scope.** Its Helm chart already lives in `cim-solution` (zero new pipeline work); its config + `dbt_schema` are moving from the standalone `cim/transflux` repo into `cim-solution`'s pre-deployment resources; its application image source (`cim/cx-data-platform`) gets CI hygiene tracked on a **separate track**, not this one.
> 9. **The committed `REGISTRY_PASS`/`efcx` credential is a low-privilege, intentional, customer-facing read-only distribution credential** — not a leaked secret. Severity corrected from Critical to a pipeline-hygiene item (§3.2). Deployment guides keep it as-is; only the CI/CD pipeline files move it to a CI/CD variable, and only as best practice, not urgent rotation.
> 10. **Two follow-on phases confirmed, explicitly sequenced after the core loop (§4) is live, not before:** automating the GitLab-registry-to-GitHub-Pages release promotion (§6, Phase 6) and migrating hardcoded secrets to Vault (§6, Phase 7).
> 11. **Every site is multi-tenant — corrects §2.2's original single-tenant-devops/rmt assumption (confirmed July 2026, during CRM-782/783 implementation).** `devops` and `rmt` each get a `tenants/` subdirectory too, same shape as every other site — not just sites judged "genuinely multi-tenant." Ground-truthing also corrected the illustrative `sites/mtt/` example from the original §2.2 sketch: `mtt01`/`mtt02`/`mtt04` are **RMT's own tenants**, not a separate hypothetical site. `devops` currently has one placeholder tenant (`devops01`) pending real onboarding data — see revised §2.2.

---

## 1. What the POC Got Right (Keep These)

| Pattern | Where | Why it's correct |
|---------|-------|-------------------|
| **Build/deploy separation** | `cx-charts-cd` builds+publishes; `cx-environments-cd` only deploys | This is the standard GitLab multi-project split — the build repo doesn't need cluster credentials at all, and the deploy repo doesn't need to know how to package a chart. Converges naturally onto **cim-solution (build/publish) → cx-environments-cd (deploy)**. |
| **`<site>/apps.yaml` = pinned desired state** | `cx-environments-cd/devops/apps.yaml` | A tiny, readable, diffable file mapping chart name → exact version for that site. This *is* the GitOps "desired state in Git" primitive. Keep it — now extended with tenant nesting, see §2. |
| **`<site>/values.yaml` = environment overrides only** | `cx-environments-cd/devops/values.yaml` | Correctly minimal — only what differs (`ingressRouter`, `imageRegistry`), no secrets. Directly answers the original ask: "deploy the new chart but use the environment-specific values.yaml, not the default one." |
| **GitLab `environment:` keyword scoping** | `deploy:devops` / `deploy:rmt` jobs, `environment: name: "..."` | This makes GitLab inject only that environment's scoped CI/CD variables — a real least-privilege mechanism, already used correctly. |
| **Per-site job = one small config diff to onboard** | Comment block, `cx-environments-cd/.gitlab-ci.yml:124-134` | Onboarding a new site is "add a folder + copy one job block" — low friction, matches the platform-engineering goal of self-service. |

---

## 2. The Real Deployment Model — Namespaces, Tenancy, and Two Audiences

This section is the ground-truth correction that reshapes the folder model in §3–§4. It comes from reading the Confluence "(5.4.0) Deployment Guide" and "(5.4.0) Data Platform Deployment Guide" directly, and cross-referencing against the real `helm-values/` files and `transflux/` — not from the POC, which only demonstrated a single-namespace, single-tenant shape.

### 2.1 Three namespace destinations, not two

| Namespace | Contains | Deployed | Charts (examples) |
|---|---|---|---|
| `ef-external` | Third-party/infra | Once per site | postgresql, mongodb, redis, minio, keycloak, apisix, vault, clamav, activemq-artemis |
| `expertflow` | Shared, multi-tenant-aware CX components | Once per site, serves all tenants via FQDN-based routing | Core, AgentDesk, Channels, Campaigns, Reporting, transflux, metabase, CiscoConnector/Scheduler/SyncService, Eleveo, Middleware(-cronjob), expertflow-ai |
| **`<tenant-id>`** (created per tenant) | Charts that do **not** yet support multi-tenancy (technical debt) | Once **per tenant**, on every site that has one onboarded (every site gets the `tenants/` folder shape — see §2.2) | `MTT-single` (hosts Campaigns Studio, Conversation Studio, QM-backend, QM-connector for that one tenant) |

The standalone `QM` chart is stale and being retired in favor of `MTT-single` — no separate handling needed for it going forward.

### 2.2 Every site is multi-tenant — folder shape is uniform

**Correction (July 2026, during CRM-782/783 implementation):** the original version of this section assumed only "genuinely multi-tenant" sites needed a `tenants/` subdirectory, with `devops`/`rmt` treated as single-tenant exceptions running Campaigns Studio/Conversation Studio/QM inline as part of the shared `expertflow` namespace deployment. The stakeholder corrected this directly: **every site is multi-tenant**, including Expertflow's own `devops` and `rmt` dev-test sites — there is no single-tenant exception. Ground-truthing against the real deployment data (`tmp/helm-values/`, `tmp/transflux/config/tenants.yaml`) corrected a second assumption in the same pass: the original folder sketch below imagined a separate hypothetical `sites/mtt/` site hosting tenants `mtt01`/`mtt02`/`mtt04`. That site doesn't exist — **`mtt01`/`mtt02`/`mtt04` are RMT's own tenants.** This also resolves the open question that was tracked in §8.

So the folder model is uniform: every `sites/<site>/` gets a `tenants/<tenant>/` subdirectory, same shape, no exceptions. Whether a tenant folder holds real deployment data or a placeholder depends on whether that site has actually onboarded a real tenant yet — `devops` hasn't, so it gets one dummy placeholder tenant until real data exists.

```
cx-environments-cd/
├── sites/
│   ├── devops/
│   │   ├── apps.yaml                # dummy POC data — real devops chart/version pins not yet available
│   │   ├── helm-values/
│   │   │   └── agent-desk-custom-values.yaml  # dummy POC data
│   │   └── tenants/
│   │       └── devops01/            # placeholder — no real devops tenant onboarded yet
│   │           ├── apps.yaml
│   │           └── helm-values/
│   │               └── MTT-single-custom-values.yaml
│   └── rmt/
│       ├── apps.yaml                # real chart:version pins, ef-external + expertflow namespace charts
│       ├── helm-values/             # real per-chart Helm overrides, one file per chart — MANDATORY,
│       │                            #   no shared/fallback file (see the note below).
│       │                            #   Naming matches tmp/helm-values/: <chart>-custom-values.yaml
│       │   ├── cx-custom-values.yaml         # Core chart (Helm name "cx", release name "ef-cx")
│       │   ├── agent-desk-custom-values.yaml
│       │   ├── keycloak-custom-values.yaml
│       │   └── ...                           # one per chart pinned in apps.yaml
│       └── tenants/
│           ├── mtt01/
│           │   ├── apps.yaml        # MTT-single chart pin
│           │   └── helm-values/
│           │       └── MTT-single-custom-values.yaml  # tenantId, FQDN, transflux tenant config (§2.4)
│           ├── mtt02/
│           │   ├── apps.yaml
│           │   └── helm-values/
│           │       └── MTT-single-custom-values.yaml
│           └── mtt04/
│               ├── apps.yaml
│               └── helm-values/
│                   └── MTT-single-custom-values.yaml
```

Onboarding a new tenant on an existing site is: add a folder under `sites/<site>/tenants/<new-id>/`. Onboarding an entirely new site is: add a `sites/<new-site>/` folder, including its own `tenants/` subdirectory (a single placeholder tenant if nothing real is onboarded yet — see `devops01` above).

**Second correction, surfaced while populating real RMT data (CRM-783):** the original design (§1, §3) assumed one shared `sites/<site>/values.yaml` was enough, passed via `-f` to every chart's `helm upgrade` in the deploy loop — true only while the POC's `devops/values.yaml` had 2-3 generic keys. Real RMT data is ~20 substantially different charts (`cx-campaigns-custom-values.yaml`, `ef-keycloak-custom-values.yaml`, etc.), each scoped to its own chart — merging all of them into one shared file risks key collisions and forces irrelevant config onto every unrelated chart's install. The first fix attempt kept the shared file as a fallback for any chart without its own values file; the stakeholder correctly rejected this — a silent generic-config deploy is exactly the "looks green, is actually broken" failure mode the initiative exists to remove. **There is no shared/fallback values file.** Every chart pinned in `apps.yaml` must have its own `sites/<site>/helm-values/<chart>-custom-values.yaml` (named after the real Helm chart name — `Chart.yaml`'s `name:` field, not always the same as the release name or the source values filename — with the `-custom-values.yaml` suffix and `helm-values/` folder matching the naming convention already used in `tmp/helm-values/`); `.deploy_env` hard-fails with a clear error if one is missing, rather than deploying with incomplete config. (RMT's `minio`/`postgresql` values were the last gap — resolved July 2026, see §8; all 22 RMT charts now have real values files.)

### 2.3 Two audiences — `cim-solution` is also the on-prem customer's repo

The deployment guide's own documented process is `git clone -b CX-5.4.0 https://efcx:...@gitlab.expertflow.com/cim/cim-solution.git` — **customers self-deploying on-prem clone the same repo** that holds Expertflow's own chart source. This has one direct consequence for this plan: anything that needs to be customer-visible (templates, generic bootstrap scripts, non-secret pre-deployment resources) belongs in `cim-solution`; anything that's Expertflow's own internal deployment data (real tenant secrets, cluster tokens, `cx-environments-cd` itself) must not leak into a repo customers clone. This distinction is what drives the `transflux` decision below.

### 2.4 `transflux` (Expertflow ETL / Data Platform) — three separate things, three separate treatments

| Piece | Today | Decision |
|---|---|---|
| **The `transflux` Helm chart** | Already at `CX-5.4.0/kubernetes/helm/transflux` | Already covered by `cim-solution`'s existing chart lifecycle (§3, §4) — zero new pipeline work, it's picked up automatically once `detect_charts`/`package_charts`/`publish_charts` are re-enabled. |
| **`cim/transflux`'s `config/` + `dbt_schema/`** | Standalone repo, no CI/CD at all, manually cloned and hand-edited (`vi transflux/config/tenants.yaml`) as part of the guide's manual on-prem process | **Move the whole `config/` directory (including populated `tenants.yaml`) and `dbt_schema/` into `cim-solution`**, alongside the existing `pre-deployment/`/`post-deployment/` resources. The default credentials inside `tenants.yaml` (`Expertflow123` etc.) are the same intentionally-shared defaults documented throughout the rest of the guide — not a leak, consistent with how every other component's default password is handled. Once moved, `cim/transflux` has no remaining purpose and should be archived. |
| **`cim/cx-data-platform`** (the application image `transflux`'s chart references) | Own repo, own build, standard microservice shape | **Out of scope for this plan.** Tracked as its own CI-hygiene item (build pipeline, security scanning per T1-6) on a separate track — confirmed by stakeholder. |

**Why this split, not "keep everything in `cim/transflux`" or "move everything into `cx-environments-cd`":** `dbt_schema/` and the config templates are release-versioned content that on-prem customers also need — they belong wherever customers already look, which is `cim-solution` (§2.3). They are not sensitive (§2.4 default-credential note above), so there's no reason to wall them off in an internal-only repo. Moving them into `cx-environments-cd` (as an earlier draft of this plan proposed) would have broken the on-prem self-serve path entirely, since customers have no access to that repo.

**Forward-looking note, not a decision to act on now:** if/when CX decomposes into independently-releasable packages (T2-8 / CIM-33653), `transflux` is a natural first candidate to carve back out into its own fully independent release (chart + config + schema together, with its own CI/CD) — at which point this consolidation would be reversed. Worth remembering when that initiative reaches this component; not a reason to hold off now.

**How `pre-deploy` handles it, concretely, once the move is done:**

> **Superseded in part (July 2026 design review, §3.6):** the `git archive --remote=cim-solution` fetch below is no longer the mechanism. Per §3.6 item 2, `bump:<site>` copies the actual pre/post-deployment files into `sites/<site>/` in `cx-environments-cd` itself — the pre-deploy job reads them locally from the checkout, no cross-repo fetch at deploy time. The cert-export/ConfigMap-regeneration steps below remain accurate.

```yaml
# cx-environments-cd — pre-deploy job for a transflux-hosting namespace
pre-deploy:mtt-shared:
  stage: pre-deploy
  script:
    - |
      set -e
      # config/ + dbt_schema/ now live in cim-solution — same pinned ref already used for chart versions,
      # no second repo to track
      git archive --remote=https://gitlab.expertflow.com/cim/cim-solution.git v5.4.0 \
        kubernetes/pre-deployment/transflux | tar -x -C ./transflux-resources

      # Export Mongo/MySQL certs — scripted, not a human running kubectl by hand
      ./scripts/export-transflux-certs.sh
      kubectl -n expertflow create secret generic ef-transflux-mysql-certs-secret \
        --from-file=certificates/mysql_certs --dry-run=client -o yaml | kubectl apply -f -

      # Regenerate both ConfigMaps idempotently
      kubectl -n expertflow create configmap ef-transflux-config-cm \
        --from-file=transflux-resources/config --dry-run=client -o yaml | kubectl apply -f -
      kubectl -n expertflow create configmap ef-transflux-dbt-schema-cm \
        --from-file=transflux-resources/dbt_schema --dry-run=client -o yaml | kubectl apply -f -
```

---

## 3. What Must Change Before This Is Production CD

### 3.1 Cluster authentication: static bearer tokens (kept for now) — GitLab Agent deferred

**Status: Deferred by stakeholder decision.** `cx-environments-cd/.gitlab-ci.yml:66-84` builds a kubeconfig from `KUBE_URL` + `KUBE_TOKEN` CI/CD variables per environment and calls the API directly. This differs from the direction recorded in `priority-list-cicd-test-automation.md:205` ("Decided approach: Native GitLab agent, not push-based deployment"), but registering the GitLab Agent requires GitLab-admin involvement (confirming KAS is enabled on `gitlab.expertflow.com`) that isn't available on this timeline. **Keep the static-token model through Phase 5 below; nothing in the pipeline design depends on the cluster-auth method.**

**What this means practically, so it isn't silently forgotten:**
- The credential-hygiene fix in §3.2 (masked, environment-scoped variables, no cleartext) is now the *only* safeguard on cluster access — treat it as non-negotiable, not optional polish.
- Each additional site cluster onboarded means one more long-lived `KUBE_TOKEN` to create, mask, and eventually rotate. Fine at the current 2–3 sites; worth revisiting once onboarding reaches 4–5.
- **Revisit trigger:** once a GitLab admin confirms KAS is enabled, migrating is a small, contained change — not a redesign of anything in §4.

### 3.2 Move committed credentials to CI/CD variables — hygiene, not incident (severity corrected)

**Finding:** `cx-environments-cd/.gitlab-ci.yml:38-41` has these values hardcoded and actively used:
```yaml
REGISTRY_URL: "gitimages.expertflow.com"
REGISTRY_USER: "efcx"
REGISTRY_PASS: "RecRpsuH34yqp56YRFUb"
POSTGRES_PASS: "Expertflow123"
```

**Corrected understanding (per stakeholder, July 2026):** `efcx`/`RecRpsuH34yqp56YRFUb` is an **intentional, low-privilege, read-only distribution credential** — the same one documented openly in the Confluence deployment guide (`git clone -b CX-5.4.0 https://efcx:RecRpsuH34yqp56YRFUb@gitlab.expertflow.com/cim/cim-solution.git`) so on-prem customers can self-serve their clone. It is **not** a leaked secret, and the deployment guides should keep it exactly as documented — no redaction needed there. `POSTGRES_PASS: "Expertflow123"` is the same category of intentionally-shared default described throughout the guide.

**What actually changes, and why it's still worth doing:** move both out of the CI/CD pipeline files and into masked GitLab CI/CD variables — not because the values are dangerous, but because hardcoding *any* credential in pipeline YAML is inconsistent practice, makes future rotation harder if the access model ever changes, and will generate false-positive noise once secret-scanning (TruffleHog, T1-6) lands and starts flagging every hardcoded string that looks like a credential. Small, contained fix; low urgency.

**Severity:** Low (downgraded from Critical). **Effort:** Small. **Owner:** Haroon Ahmed (took over from Umar Naveed, July 2026).

### 3.3 Chart-to-namespace/overrides mapping — now informed by §2, still deferred

**Finding:** The `case "$name" in ...` block (`cx-environments-cd/.gitlab-ci.yml:101-114`) only special-cases a handful of charts by name, with a working catch-all for everything else.

**Status: still deferred**, but now with a clearer target shape thanks to §2 — when this is picked up, the three-way namespace classification (§2.1) should drive it directly: chart → namespace-category (`ef-external` / `expertflow` / per-tenant) as declarative metadata, not a growing bash case statement. Revisit only when a third chart genuinely needs a special override that doesn't fit the catch-all.

**Severity:** Low. **Effort:** Medium. **Owner:** Haroon Ahmed + stream leads — only when actually needed.

### 3.4 Missing `rmt/` folder

**Finding:** `deploy:rmt` job exists in the pipeline but no `rmt/apps.yaml` or `rmt/values.yaml` exists yet — the job will fail immediately if triggered.

**Fix — done (CRM-783, July 2026):** Created `sites/rmt/apps.yaml` (21 real chart:version pins, cross-referenced against `CX-5.4.0/kubernetes/helm/*/Chart.yaml`) and, per §2.2's second correction, one **mandatory** real values file per chart under `sites/rmt/helm-values/<chart>-custom-values.yaml` — no shared/fallback file; `.deploy_env` hard-fails on a chart with no dedicated values file rather than deploying it with generic config. 19 of 21 charts have real values; `minio` and `postgresql` are pinned but currently have none, so `deploy:rmt` will fail on them until real values are handed over (tracked in §8). RMT is **not** single-tenant (§2.2 correction) — its real tenants `mtt01`/`mtt02`/`mtt04` are onboarded under `sites/rmt/tenants/`, separately from this site-level work.

**Severity:** High (blocks any real use of `deploy:rmt`). **Effort:** Small→Medium (grew once real per-chart data replaced the single-file assumption). **Owner:** Haroon Ahmed (also owns RMT values).

### 3.5 No rollback (ties to CRM-769)

**Finding:** `.deploy_env`'s script is a straight `helm upgrade --install`. On failure it dumps `kubectl get pods,events` and exits — no automatic `helm rollback`, and `HELM_WAIT="false"` by default means the pipeline can report success before pods are actually healthy.

**Fix:** Add a rollback step to `.deploy_env`, gated on the `helm upgrade` exit code:
```bash
helm upgrade --install "$REL" "cxcharts/$name" --version "$ver" \
  --namespace "$NS" --create-namespace -f "$VALUES" $EXTRA $WAIT_ARGS $DRY \
  || {
    echo "FAILED deploying $name — rolling back to previous revision";
    helm rollback "$REL" -n "$NS" || echo "WARN: rollback also failed, manual intervention needed";
    kubectl get pods,events -n "$NS" | tail -n 30;
    exit 1;
  }
```
This directly implements CRM-769's rollback ticket, at the *correct* location now that deploy authority has moved to `cx-environments-cd`.

**Update (July 2026):** implemented — plus a `DRY_RUN` guard the snippet above lacks (a dry-run failure is a validation error; rolling back a real release over it would be wrong). **Addendum from the July 2026 design review (§3.6 item 6):** rollback must also revert the *declared state* — the `apps.yaml` pins (and copied pre/post-deploy files) in `cx-environments-cd`, committed with `[skip ci]` — not just the cluster. Cluster-only rollback leaves the repo pointing at the failed version, and the next deploy re-arms the failure. Applies to both this inline rollback and the Phase-2 `rollback:<site>` stage.

Also: set `HELM_WAIT="true"` (or a per-site override) at least for RMT, so a bad rollout is caught by the deploy job itself. **(Done, July 2026 — job-level override on `deploy:rmt`.)**

**Severity:** High. **Effort:** Small. **Owner:** Haroon Ahmed (took over from Umar Naveed, July 2026).

### 3.6 Pre-deploy and post-deploy resources — namespace-aware, tenant-aware (ties to CRM-771/772)

**Finding:** This pipeline only ever runs `helm upgrade`. Real deployment also requires, per §2: namespace creation for new tenants, cross-namespace cert/secret/configmap copying (`ef-external`/`expertflow`/`vault` → `<tenant-id>`), transflux config generation, tenant creation, Grafana/Metabase provisioning, MinIO data.

**Settled control-flow design (July 2026 design review — supersedes the earlier "manual trigger on pre-deploy" correction, which was itself based on a misreading of the bump commits as automatic; the POC's `bump:<site>` is a manual per-team button, and that turns out to be the right design):**

1. **Two human decision points, each a one-button choice — everything else automatic.** `detect_charts` stays manual: with several MRs open, it picks *which MR* enters the pipeline (prioritization). `bump:<site>` stays manual: it is THE go-live decision — *which site* takes the change, and when. Nothing downstream of bump needs a click. This re-scopes CRM-763's "no manual step" acceptance criterion to its real intent: **no manual work, only one-button manual decisions** — never a procedure someone must know by heart.
2. **`cx-environments-cd` is the complete, self-contained state of every site.** Charts are represented by version pin (the package lives in the Helm registry — the pin fully identifies it). Pre/post-deployment resources have no registry, so the **actual files** live in the repo, per site. `bump:<site>` therefore writes both: chart pins into `apps.yaml` *and* verbatim copies of the MR's changed/added pre/post-deployment files into `sites/<site>/` (deletions honored via `--name-status`). One changeset advances the whole site's state atomically — site level and **all tenant folders together** (never one MR per tenant).
3. **New-chart onboarding (re-litigation of the no-fallback-values decision, resolved):** when the changeset includes a chart not yet pinned on the target site, bump routes the **entire changeset** through a single MR on `cx-environments-cd` instead of pushing to main: the new pin plus `helm-values/<chart>-custom-values.yaml` seeded verbatim from the chart's own `values.yaml`; the reviewer edits the site-specific fields; **merging that MR is the single deploy moment for everything in the changeset** (atomicity over speed — the existing-chart updates in the same changeset deliberately wait on the review). Known-chart changesets push directly to main. Both paths end identically: one commit on main = one deploy moment. A repeat bump while the onboarding MR is open updates that MR, never opens a duplicate.
4. **Commit-as-trigger:** any commit landing on `cx-environments-cd`'s main deploys what it touched. Per-site scoping via `rules: changes:` (`sites/rmt/**` → RMT's chain only). `[skip ci]` opts out — used by rollback's state-revert commits and by bulk/structural changes. Direct config edits by engineers are a first-class path (edit → push → that site redeploys exactly what changed — the common RMT enable/disable-config case). Known sharp edge, accepted: this is an opt-*out* model — a bulk commit without `[skip ci]` deploys everything it touched; main stays protected/reviewed to mitigate.
5. **Unified skip-if-unchanged:** an in-cluster marker per site stores the last-applied `cx-environments-cd` commit SHA. Every chain run diffs `<marker>..HEAD -- sites/<site>/` — one mechanism for everything: only charts whose pin/values changed get `helm upgrade`; only pre/post-deploy files that changed get applied; everything else is logged "unchanged, skipped". First-ever run (no marker) applies everything. The marker updates only on successful chain completion; rollback resets it.
6. **Rollback reverts state, not just cluster:** `apps.yaml` + the copied files are the record of what's deployed, so the Phase-2 `rollback:<site>` stage must both `helm rollback` the affected releases *and* revert the corresponding pins/files in `cx-environments-cd` (committed with `[skip ci]`). Without the state-revert, the next deploy re-arms the failure. The inline `.deploy_env` rollback (CRM-784) has the same declared-state drift and gets the same treatment when Phase 2 lands.
7. **Process assumption, not pipeline-enforced:** two MRs touching the same pre/post-deploy file never deploy concurrently — team rule: deploy one, test, merge to the RC branch; the second MR pulls the RC branch (resolving the conflict git now forces) before it deploys. Tripwire in the pipeline: bump warns loudly when it is about to overwrite a `sites/<site>/` file modified since the last bump.
8. **Stage roles under this model** — bump writes the order (git only, no cluster access: the factory repo deliberately holds no `KUBE_*` credentials); `pre-deploy` prepares the ground (applies changed ConfigMaps/Secrets/certs/RBAC/tenant namespaces from the copied files); `deploy` installs the changed charts (helm, inline rollback, `HELM_WAIT`); `post-deploy` finishes on the running stack (application-level tenant creation, Grafana, MinIO seeding — idempotent); `test`/`rollback`/`notify` judge, undo if needed, report.

**Fix:** Two distinct pre-deploy shapes, not one generic stage:

```yaml
stages:
  - pre-deploy
  - deploy
  - post-deploy

# Site-level (ef-external + expertflow namespace charts) — existing shape, unchanged
pre-deploy:rmt:
  extends: .pre_deploy_site
  ...

# Tenant-level (MTT-single into a per-tenant namespace) — new, only for multi-tenant sites
pre-deploy-tenant:mtt01:
  extends: .pre_deploy_tenant
  script:
    - kubectl get namespace mtt01 || kubectl create namespace mtt01
    - |
      for pair in "ef-external:mongo-mongodb-ca" "ef-external:redis-crt" "ef-external:ef-postgresql-crt" \
                  "expertflow:ef-logback-cm" "expertflow:ef-cx-efconnections-cm" \
                  "expertflow:ef-gitlab-secret" "expertflow:conversation-studio-approle-secret" \
                  "vault:tls-ca" "vault:tls-server-client" "vault:tls-server-vault"; do
        SRC_NS="${pair%%:*}"; NAME="${pair##*:}"
        kubectl get secret "$NAME" -n "$SRC_NS" -o yaml \
          | sed "s/namespace: $SRC_NS/namespace: mtt01/" \
          | kubectl apply -f - 2>/dev/null || \
        kubectl get configmap "$NAME" -n "$SRC_NS" -o yaml \
          | sed "s/namespace: $SRC_NS/namespace: mtt01/" \
          | kubectl apply -f -
      done
```

This is exactly what CRM-771/772 scope, and §2.4 gives the transflux-specific piece its own concrete landing place.

**Severity:** High. **Effort:** Large (this is most of the remaining CD work). **Owner:** Haroon Ahmed (took over from Umar Naveed, July 2026).

### 3.7 Governance on the `bump` mechanism

**Finding:** `cx-charts-cd`'s `.bump_env` clones `cx-environments-cd` with a token that has `write_repository` and pushes directly to `main` with no merge request — a reasonable POC shortcut, a governance gap for production.

**Recommendation:**
1. Scope the bump token to a **Project Access Token** with the minimum role (`write_repository` only, on `cx-environments-cd` alone), rotated on a schedule.
2. Use **GitLab Protected Branches "Allowed to push"** for `main`, explicitly listing the bump-bot rather than leaving the branch fully unprotected.

**Addendum (July 2026 design review) — who may click `detect_charts` and `bump:<site>`: Maintainers only, enforced in-script.** The two manual decision buttons (§3.6 item 1) get gated so only users with Maintainer access can trigger them — necessary at latest when the factory moves into `cim-solution`, where every stream developer is a member (today `cx-charts-cd`'s small membership is the accidental gate).

**Mechanism (verified 2026-07-10):** the first-choice mechanism, GitLab **Protected Environments**, is a Premium feature and `gitlab.expertflow.com` is **Free tier** — confirmed by the absence of the "Protected environments" section in cim-solution's Settings → CI/CD (full Maintainer view, so not a permissions artifact). The decided mechanism is therefore the **in-script Maintainer check**: the gated jobs' first step resolves the clicking user (`$GITLAB_USER_ID` is set to whoever runs a manual job) against the members API and fails unless `access_level >= 40`:

```bash
LEVEL=$(curl -s --header "PRIVATE-TOKEN: $API_TOKEN" \
  "$CI_API_V4_URL/projects/$CI_PROJECT_ID/members/all/$GITLAB_USER_ID" | jq -r '.access_level // 0')
[ "$LEVEL" -ge 40 ] || { echo "Only Maintainers may trigger this job (you: level $LEVEL)"; exit 1; }
```

Known limits, accepted: enforcement-after-click (the button is visible to all members; unauthorized runs fail immediately with a clear message), and the check lives in `.gitlab-ci.yml` so it's removable by anyone with write access to the CI file — but that set is Maintainers, so the threat model holds. Applies to `detect_charts` and every `bump:<site>` job, in the factory repo (`cx-charts-cd` today; re-apply when the factory moves to `cim-solution`). **Upgrade path:** if the instance is ever licensed to Premium, switch to Protected Environments (`environment: name: <site>` on bump, synthetic `release-gate` environment with `action: prepare` on detect; "Allowed to deploy" = Maintainers) — UI-enforced and not script-removable.

**Severity:** Low. **Effort:** Small. **Owner:** Haroon Ahmed (took over from Umar Naveed, July 2026).

### 3.8 Chart build hygiene carried over from cim-solution's disabled block

- `helm lint ... || true` swallows lint failures — fix when re-enabling
- `RELEASE_VERSION` hardcoded — **intentional, not debt (clarified July 2026):** each release-candidate branch carries its own `RELEASE_VERSION`, so MRs targeting different release trains (e.g. 5.4.0 and 5.5.0 in parallel) version their charts into the right train. The remaining checklist item is only: setting it correctly when a new RC branch is cut.
- No `cache:` for Helm dependency downloads across pipeline runs

---

## 4. Complete Pipeline Design (`include:`-based, namespace- and tenant-aware)

> **Visual version:** `cicd-pipeline-diagram-2026-07.html` (same folder) — renders the core loop below.

> **Read §3.6's settled control-flow design first (July 2026):** it governs how everything below triggers — `bump:<site>` is the manual go-live decision and writes complete site state (pins **and** pre/post-deploy files); the deploy chain fires on commits to `cx-environments-cd` main scoped by `rules: changes:`; skip-if-unchanged is driven by a per-site last-applied-SHA marker. The job sketches below remain valid for *what each job does*, but where a sketch implies a different trigger mechanism, §3.6 wins.

This section gives the concrete, job-by-job design across all three repos: chart lifecycle in `cim-solution` (now including `transflux`'s config/schema per §2.4), deploy lifecycle in `cx-environments-cd` (now site- **and** tenant-aware per §2.2), test *ownership* in `playwright-automation-script`.

### 4.1 Full pipeline shape

```
cim-solution (chart lifecycle)                     cx-environments-cd (deploy lifecycle — ONE pipeline)
─────────────────────────────                       ────────────────────────────────────────────────────
stages: detect → build → publish → bump             include:
                                                       - project: 'qa/playwright-automation-script'
detect_charts (MR-triggered, existing)                  ref: 'v5.4.0'                    # CRM-768 pin
      │                                                  file: '.gitlab-ci-template.yml'
      ▼
package_charts (RC version bump, existing;          stages: pre-deploy → deploy → post-deploy → test → rollback → notify
  now also versions transflux's config/dbt_schema
  since they moved here, §2.4)                      SITE-LEVEL (ef-external + expertflow namespaces):
      │                                              pre-deploy:<site> → deploy:<site> → post-deploy:<site>
      ▼
publish_charts (push to Helm registry, existing)     TENANT-LEVEL (multi-tenant sites only, per-tenant namespace):
      │                                              pre-deploy-tenant:<site>:<tenant> → deploy-tenant:<site>:<tenant>
      ▼
bump:<site>  (manual button, one per site ──────▶
  updates <site>/apps.yaml pin in                    test:<site/tenant>  — extends: .playwright_regression
  cx-environments-cd)                                       │              rules: $RUN_REGRESSION == "true"
                                                              │              runs IN this pipeline; own local artifacts
                                                       ┌──────┴──────┐
                                                       ▼             (no separate job on success —
                                            rollback:<site/tenant>     notify reads absence of
                                            when: on_failure           rollback artifact as "passed")
                                                       │             │
                                                       └──────┬──────┘
                                                              ▼
                                                       notify:<site/tenant>  — when: always
                                                       needs: [test, rollback (optional: true)]
                                                       posts aggregate deploy+test+rollback card
```

`playwright-automation-script`'s own `.gitlab-ci.yml` is **completely untouched** — standalone QA authoring/debugging only.

### 4.2 Where each stage physically lives, and why

| Stage | Repo | Rationale |
|-------|------|-----------|
| detect / build / publish | `cim-solution` | Chart lifecycle stays where the charts live — including transflux's config/schema now that they've moved here (§2.4) |
| bump | `cim-solution` → pushes into `cx-environments-cd` | Promotion trigger stays adjacent to "a new chart version exists" |
| pre-deploy / deploy / post-deploy (site-level) | `cx-environments-cd` | `ef-external` + `expertflow` namespace charts, once per site |
| pre-deploy-tenant / deploy-tenant (multi-tenant sites only) | `cx-environments-cd` | Per-tenant namespace creation, cross-namespace secret copying, `MTT-single` deployment (§2.2, §3.6) |
| test | `cx-environments-cd` (job definition **included** from `playwright-automation-script`, executed locally) | Runs in the same pipeline as everything reacting to its result — §5 explains why `include:` fits better than a separate triggered pipeline |
| rollback | `cx-environments-cd` | Needs `helm` + cluster access, which only exists in this repo's job context |
| notify | `cx-environments-cd` | Only this repo has visibility into all outcomes needed for one aggregate card |

### 4.3 Critical correctness requirement: `allow_failure` must be `false` on the test job

Today's dormant `CX-5.4.0/.gitlab-ci.yml:469` has `allow_failure: true` on `regression-test` — which is *why* `notify-google-chat` currently fires regardless of actual test outcome. For `rollback`'s `rules: when: on_failure` to ever actually fire, **the test job must NOT have `allow_failure: true`.** A real failure has to propagate as a real failure:

```yaml
# cx-environments-cd/.gitlab-ci.yml
include:
  - project: 'qa/playwright-automation-script'
    ref: 'v5.4.0'
    file: '.gitlab-ci-template.yml'

test:rmt:
  extends: .playwright_regression
  stage: test
  variables:
    BASE_URL: "https://rmt.expertflow.com"
  # allow_failure defaults to false — do not set it to true here.
  needs:
    - deploy:rmt
  rules:
    - if: '$RUN_REGRESSION == "true"'

rollback:rmt:
  stage: rollback
  needs:
    - test:rmt
  rules:
    - when: on_failure
  script:
    - helm rollback "$CORE_RELEASE" -n "$APP_NS"
  artifacts:
    reports:
      dotenv: rollback.env   # writes ROLLBACK_OCCURRED=true — notify reads this

notify:rmt:
  stage: notify
  needs:
    - job: test:rmt
    - job: rollback:rmt
      optional: true
  rules:
    - when: always
  script:
    - |
      if [ -f rollback.env ]; then
        echo "Deploy succeeded, tests FAILED, rollback executed"
      else
        echo "Deploy + tests PASSED, no rollback needed"
      fi
```

### 4.4 Prerequisite: fix test flakiness before wiring auto-rollback

Per `current-cicd-test-automation-maturity-audit.md`: file-wide serial mode (one suite's failure cascades into 9 "not run" results), `networkidle` waits against a websocket app, hardcoded per-suite phone numbers with no run isolation. If `rollback` is wired to fire automatically while these are unresolved, **a flaky test will trigger a real production rollback**. Land the maturity audit's remediation items (now tracked as CRM-777/778) **before** enabling auto-rollback for RMT. Until then, keep rollback's trigger `when: manual`.

---

## 5. Why `include: project:` Instead of `trigger:` or Inline Clone-and-Run

Today, the same "clone Playwright repo, `npm ci`, run tests" logic exists in two places: `playwright-automation-script`'s own pipeline and `CX-5.4.0`'s dormant `regression-test` job. Adding a third copy — or a `trigger:`-based bridge job pointed at a separate downstream pipeline — would still mean the test-execution logic is either duplicated, or run somewhere that requires extra machinery to get its result and artifacts back.

**Why not `trigger:`?** It's suited to two sides that are each independently meaningful pipelines. "Did the regression pass after this deploy" isn't a separate story worth its own pipeline — it's one step in the story of *this deploy*. Using `trigger:` here meant bridge-job `strategy: depend` semantics, a cross-project artifact-visibility caveat, and a `$CI_PIPELINE_SOURCE == "pipeline"` skip-rule inside `playwright-automation-script` purely to stop it double-notifying — machinery solving a problem `trigger:` itself introduced.

**Why `include: project:` fits better:**
- **One place owns "how do we run regression"** — QA's own repo.
- **CRM-768 (suite versioning) applies once** — pinning `ref:` to a release tag is one line in one place.
- **One pipeline, one place to look** — Deploy, Test, Rollback, Notify all in `cx-environments-cd`'s own pipeline graph.
- **Artifacts land locally** — no cross-project API call needed for the notify card.
- **`playwright-automation-script` needs zero changes.**
- **Respects "environments are independent"** — `RUN_REGRESSION` is a per-site opt-in flag.

**One genuine tradeoff, named plainly:** with `trigger:`, a run would show up as its own pipeline record inside `playwright-automation-script`'s history. With `include:`, it doesn't. Not treated as a loss — the meaningful unit of work is "did this deploy succeed" — but a real behavioral difference worth knowing.

---

## 6. Phased Rollout

### Phase 1 — Foundation fixes (this sprint)
- Move committed credentials to CI/CD variables (§3.2) — hygiene fix, do alongside other §3 edits
- ~~Create `sites/rmt/apps.yaml` + `sites/rmt/values.yaml` under the new folder shape (§2.2, §3.4)~~ **Done (July 2026, CRM-782/783):** full `sites/<site>/tenants/<tenant>/` restructure; all 22 RMT charts (incl. transflux, minio, postgresql) pinned with real per-chart `helm-values/` files
- ~~Add rollback to `.deploy_env` for helm-upgrade-itself failures (§3.5)~~ **Done (July 2026, CRM-784)**
- ~~Set `HELM_WAIT="true"` for RMT~~ **Done (July 2026, CRM-785)**
- Tighten `bump` token scope + protected-branch config (§3.7)

### Phase 2 — Full CD loop: control flow, deployment coverage, regression + rollback wiring (§3.6 settled design, §4, §5; **absorbs former Phase 4, merged July 2026**)

Ordered by dependency — the state-writing side first, then the stages that consume it, then the judge/undo/report tail:

- Extend `bump:<site>` per §3.6: copy changed/added pre/post-deployment files into `sites/<site>/` (deletions included, all tenant folders in the same changeset); overwrite tripwire warning; new-chart onboarding via single auto-generated MR (pin + seeded values, whole changeset held together)
- Flip `cx-environments-cd` deploy chain to commit-as-trigger: per-site `rules: changes: sites/<site>/**`, `[skip ci]` opt-out, chain runs unattended after any qualifying commit (bump push, onboarding-MR merge, or direct config edit)
- Implement the unified last-applied-SHA marker + diff-scoped apply/skip for charts and pre/post-deploy files (§3.6 item 5)
- Add site-level `pre-deploy`/`post-deploy` stages (§3.6) — the consumers of bump's copied files; this is where transflux's runtime config (tenants.yaml ConfigMap, dbt_schema — §2.4) finally gets mounted, closing the known "chart deploys but comes up unconfigured" gap from CRM-783
- Add tenant-level `pre-deploy-tenant`/`deploy-tenant` for every site (§2.2, §3.6) — per-tenant namespace creation, cross-namespace secret/cert copies, MTT-single deployment
- Re-enable and clean up `cim-solution`'s `detect/build/publish` stages (the factory move), confirming `transflux`'s moved config/`dbt_schema` (§2.4) is included in what gets versioned/published — and re-applying the Maintainer gate (§3.7) in `cim-solution` when the buttons move there
- Add `include: project:` + `test:<site>` jobs, starting with RMT
- Confirm `allow_failure` is **not** `true` on the test job (§4.3)
- Add `rollback:<site>` — start with `when: manual` until Phase 3 lands; must revert declared state (pins + files, `[skip ci]`) as well as the cluster (§3.6 item 6), and reset the marker
- Add `notify:<site>` aggregating deploy+test+rollback status
- Remove `regression-test`/`notify-google-chat` from `CX-5.4.0/.gitlab-ci.yml`
- (§3.3 chart-override metadata generalization stays deferred)

### Phase 3 — Test-suite hardening (prerequisite to auto-rollback)
- Remove file-wide serial mode cascade, kill top flake sources, restore coverage integrity (CRM-777/778)
- **Once flake rate is measurably low:** flip rollback from `when: manual` to `when: on_failure` for RMT

### Phase 4 — ~~Full deployment coverage, namespace- and tenant-aware~~ **Merged into Phase 2 (July 2026)**
All former Phase 4 items now live in Phase 2 above — the split had bump copying pre/post-deploy files in Phase 2 while the stages that *apply* them waited for Phase 4, leaving the files inert in the repo and transflux unconfigured in the meantime. Merging closes that gap in one phase. Number kept as a tombstone so existing references ("Phase 4" in tickets/memories) still resolve here.

### Phase 5 — Cluster access convergence (deferred, revisit once GitLab admin confirms KAS availability)
- Register GitLab Agent, migrate off static `KUBE_URL`/`KUBE_TOKEN`

### Phase 6 — GitHub Pages release promotion (explicitly sequenced after the core loop — Phases 1–3 — not before)
**Confirmed in scope, lower priority.** Today, Haroon manually promotes charts from the GitLab RC registry to the public `expertflow.github.io/charts` repo once testing passes and a release is ready. Document this process formally first; automating it (a `publish:release` job triggered on a release tag, pushing to GitHub Pages) is the follow-on once the core loop (Phases 1–4) is stable. Not started.

### Phase 7 — Secrets → Vault migration (explicitly sequenced after the core loop — Phases 1–3 — not before)
**Confirmed in scope, lower priority.** Given Vault is already deployed as part of the stack, the durable fix for the broader hardcoded-credential pattern found across `helm-values/*.yaml` and `transflux/config/*.yaml` (mostly intentional shared defaults per §3.2, but worth consolidating regardless) is migrating them to Vault-backed references rather than plaintext YAML — following the same secret-copy-between-namespaces pattern the deployment guide already uses for TLS certs. Not started; scope this properly once Phases 1–4 are live.

---

## 7. Mapping to Existing Jira Tickets, and New Work Items (Not Yet Ticketed)

### Existing tickets

| Ticket | How this plan changes/completes it |
|--------|-------------------------------------|
| CRM-763 (T1-5, GitLab CD pipeline) | This plan **is** the concrete implementation path |
| CRM-769 (rollback job) | Re-scoped to `cx-environments-cd`: inline (§3.5) and downstream (§4.3) mechanisms |
| CRM-771 (pre-deploy resources) | Lands as `cx-environments-cd`'s `pre-deploy`/`pre-deploy-tenant` stages (§3.6), extended with the namespace/tenant model from §2 |
| CRM-772 (post-deploy resources) | Lands as `cx-environments-cd`'s `post-deploy` stage (§3.6) |
| CRM-768 (Playwright versioning) | Consumed by the `include: project:`'s `ref:` pin (§5) |
| CRM-777/778 (Playwright hardening/stub cleanup) | Prerequisite for Phase 3, gates auto-rollback |

### New work items identified this revision — **not yet ticketed, per stakeholder instruction; ticket when work begins**

| Work item | Where it lands | Ticket |
|---|---|---|
| Move `REGISTRY_PASS`/`POSTGRES_PASS` to CI/CD variables | §3.2 | [CRM-781](https://expertflow-docs.atlassian.net/browse/CRM-781) |
| Restructure `cx-environments-cd` folders into `sites/<site>/[tenants/<tenant>/]` shape | §2.2, §3.4 | [CRM-782](https://expertflow-docs.atlassian.net/browse/CRM-782) — foundational, do first |
| Create `sites/rmt/apps.yaml` + `values.yaml` | §3.4 | [CRM-783](https://expertflow-docs.atlassian.net/browse/CRM-783) |
| Add inline rollback to `.deploy_env` | §3.5 | [CRM-784](https://expertflow-docs.atlassian.net/browse/CRM-784) |
| Set `HELM_WAIT="true"` for RMT | §3.5 | [CRM-785](https://expertflow-docs.atlassian.net/browse/CRM-785) |
| Tighten `bump` token scope + protected-branch config | §3.7 | [CRM-786](https://expertflow-docs.atlassian.net/browse/CRM-786) |
| Tenant-level `pre-deploy-tenant`/`deploy-tenant` jobs (namespace creation, cross-namespace secret/configmap copying) | §3.6 | Not yet ticketed — Phase 2 (was Phase 4, merged July 2026), large, ticket once Phase 1 lands |
| Move `cim/transflux`'s `config/` + `dbt_schema/` into `cim-solution`; archive `cim/transflux` | §2.4 | Not yet ticketed — Phase 2 (was Phase 4, merged July 2026) |
| `cx-data-platform` CI hygiene (build pipeline, security scanning) | Separate track (confirmed) | Not this initiative's scope |
| GitHub Pages release-promotion automation | §6, Phase 6 | Not yet ticketed — explicitly deferred until Phases 1–4 are live |
| Vault migration for hardcoded config values | §6, Phase 7 | Not yet ticketed — explicitly deferred until Phases 1–4 are live |

---

## 8. Open Items

- ~~**Rollback target granularity:**~~ **Direction settled (July 2026 design review):** rollback reverts exactly what the failing changeset advanced — `helm rollback` on each release the run upgraded, plus a `[skip ci]` revert of the same pins/files in `cx-environments-cd` (§3.6 item 6). Charts untouched by the changeset are never rolled back. Implementation detail (Phase 2): the changeset diff that scoped the deploy also scopes the rollback.
- **Chart-metadata format (§3.3):** worth a short design spike before committing to a specific file format — not blocking Phase 1/2 work. **Partially bitten already (July 2026):** `.deploy_env`'s namespace case statement only covered POC-era charts; unmapped charts silently fall through to `expertflow`. Found when the devops dry-run collided vault (own `vault` namespace, now mapped explicitly). Any chart onboarding must check the case statement covers it.
- **RMT `artemis` pin needs review before deploy:rmt goes real:** artemis is officially deployed as a **system service**, not via its helm chart (confirmed July 2026, during devops validation — the chart pin was dropped from devops for this reason). `sites/rmt/apps.yaml` still pins `artemis: "5.4.0-rc.1"` — likely to remove, decide alongside the RMT go-live review.
- ~~**`sites/mtt/` naming:** confirm the actual site-level grouping name...~~ **Resolved (July 2026):** there is no separate `mtt` site — `mtt01`/`mtt02`/`mtt04` are RMT's own tenants. §2.2 updated accordingly.
- ~~**`minio`/`postgresql` real values for RMT**~~ **Resolved (July 2026):** real values handed over and committed as `sites/rmt/helm-values/{minio,postgresql}-custom-values.yaml` (carried verbatim from `tmp/helm-values/ef-{minio,postgresql}-custom-values.yaml`). All 22 RMT charts (incl. `transflux`, added same week) now have values files — `deploy:rmt` no longer has a known hard-fail. Residual note: minio's `rootUser`/`rootPassword` are still the stock chart defaults (`minioadmin`/`minioadmin`) — real handed-over file, not a placeholder, but flag if it should be tightened.
- ~~**devops real chart/version pins and values**~~ **Resolved (July 2026) — and the deferral reversed:** devops (fresh VM, `gitconnect.expertflow.com` / 192.168.2.205, RKE2 pre-existing) became the pipeline-validation site *ahead of* RMT — a fresh cluster with no users is the safer shakeout target. `sites/devops/` now has a real 12-chart set (full ef-external tier + `cx` Core + `agent-desk`), values carried from `sites/rmt/helm-values/` with devops-only deltas (`ingressRouter` → `gitconnect.expertflow.com`, hostAliases → the VM IP). Known leftovers flagged in the commit: `cx`'s `ACTIVEMQ_*_URL` still point at the RMT-era external ActiveMQ; minio auth is stock defaults.

- **Go-live cutover checklist (July 2026 — collected during factory-move testing; run these together when the pipeline is declared ready, none before):**
  1. Merge the `CRM-782-sites-tenants-restructure` MR into `cx-environments-cd` `main` — until then, `main` still has the flat POC layout.
  2. Flip `ENV_REPO_BRANCH` in cim-solution's `.bump_env` from `CRM-782-sites-tenants-restructure` back to `main`.
  3. **Migrate the charts registry from project 1343 (CX-Charts-CD) to cim-solution's own** — stakeholder-agreed (July 2026): keep 1343 during testing for continuity (existing versions + RC numbering), move at go-live so cx-charts-cd can be archived outright. Steps: flip `CHARTS_PROJECT_ID`/`HELM_REPO_URL` in cim-solution, flip `CHARTS_REPO_URL` in cx-environments-cd, run one factory pass to populate the fresh registry, re-pin via bump.
  4. CRM-786 hardening: protect `main` (bump-bot allow-listed), mark `ENV_REPO_TOKEN`/`GITLAB_TOKEN` Protected once pipelines run from protected refs, masking + job-token allowlist hygiene.
  5. Archive `cim/cx-charts-cd` (repo already superseded; registry too, after step 3).
  6. Merge `5.4.0_f-CRM-765` (factory + regression job) into its RC branch — the moment the factory goes live for every stream team's MRs; Maintainer gate must already be verified working (it will have been, during the loop test).
