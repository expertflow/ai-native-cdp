---
name: project-cicd-chart-onboarding-automation
description: Use Case 2 (new-chart onboarding) built and validated live — bump classifies unpinned charts by scanning real state (no metadata file), seeds pin+values, routes through onboard/<site> branch+MR
metadata:
  type: project
---

**Fact:** As of July 2026, "Use Case 2" — automating onboarding of a chart never before pinned on a site — is built and validated end-to-end on real infrastructure (devops, `channels` chart). Previously a confirmed gap: `.bump_env` only ever updated existing pins, silently skipping (later: loudly warning but still skipping) anything unpinned.

**How it works, in `cim-solution`'s `.bump_env` (`CX-5.4.0`, branch `CRM-763-factory-loop-test`):**
- For each published chart, checks every apps.yaml file relevant to the target site (site-level + every tenant). If unpinned in ALL of them, it's genuinely new to that site.
- Classifies a new chart as tenant-scoped or site-scoped by scanning **every other site's real, already-committed apps.yaml data** (any tenant-folder pin anywhere → tenant-scoped; any site-level pin anywhere → site-scoped) — deliberately **no separate metadata file**, since the deployed state itself is already the source of truth and can't drift out of sync the way a parallel list could. A chart found in neither, anywhere, ever, defaults to site-level with an explicit reviewer-confirm flag in the MR description.
- Seeds the pin + a values file (from the chart's own packaged `values.yaml` via `helm show values`) into the right location(s).
- Routes the **whole changeset** (not just the new chart) through a branch named `onboard/<site>` and an MR, rather than direct push — atomicity over speed, matches CD plan §3.6 item 3. Detects an already-open `onboard/<site>` MR via the GitLab API and pushes additional commits to it instead of duplicating.
- Hardened error handling: the three new GitLab API calls (branch lookup, MR list, MR create) explicitly check HTTP status and fail loudly on anything unexpected (401/403 etc.) instead of silently behaving as if nothing were wrong — a first draft had this bug (treating any non-200 as "doesn't exist").

**Real prerequisite this surfaced:** `GITLAB_TOKEN` (already used for the Maintainer-gate members-API check on `cim-solution`) now ALSO needs `api`-scope access to the **`cx-environments-cd`** project specifically (branch lookup + MR create/list) — it never needed to reach that project before. Must confirm the token's identity is a member of both projects.

**Validated live:** `channels` onboarded onto `devops` for real — MR opened, reviewed (with the "removed the externals" correction along the way, see [[project-cicd-deployment-guide-crosscheck]]), merged, deployed as release `cx-channels` in namespace `expertflow`, `REVISION: 1`.

**How to apply:** When onboarding any new chart type going forward, this mechanism should handle it automatically — a human only needs to review the seeded MR (values + placement confirmation if unclassified). If a second tenant-scoped chart type is ever introduced, revisit the "release = tenant id" assumption (see [[project-cicd-tenant-deploy-implementation]]) since it only holds while each tenant runs exactly one tenant-scoped chart. Related: [[project-cicd-manual-deploy-trigger]], [[project-cicd-tenant-deploy-implementation]].
