---
name: project-cicd-crm-763-bundle
description: Haroon committed to completing CRM-782 and CRM-783 together with CRM-763 (parent CD pipeline ticket), not as independently-schedulable sub-tasks
metadata:
  type: project
---

**Fact:** Haroon Ahmed ensured CRM-782 (sites/tenants folder restructure) and CRM-783 (RMT apps.yaml + values.yaml) will be completed together with CRM-763 (Setup GitLab CD pipeline, the parent ticket) — confirmed by Jawad 2026-07-21. All three are being treated as one delivery unit, not staggered independently.

**Why:** CRM-763's acceptance criteria depend on the site/tenant folder shape CRM-782 establishes and the real RMT chart/values work CRM-783 delivers — per CRM-763's own ticket description, CRM-782/783/784/785 are already done on branch `CRM-782-sites-tenants-restructure` in `cx-environments-cd` (not yet merged), so this bundling reflects actual delivery sequencing, not just a stated intent.

**How to apply:** When tracking progress or status-reporting on CRM-763, treat CRM-782/783 completion as part of the same milestone rather than separate line items. Related: [[project-cicd-all-sites-multi-tenant]].
