---
name: project-cicd-confluence-docs-published
description: 7-page CD pipeline doc tree published in Confluence (EF space, under existing CI/CD Guidelines) — branch/CI-CD setup, brownfield/fresh-site onboarding, coverage boundary, architecture diagram, FAQ
metadata:
  type: project
---

**Fact:** July 2026 — published a 7-page documentation tree in Confluence covering the automated CD pipeline (`cim-solution` + `cx-environments-cd`), as a child tree under the existing **"CI/CD Guidelines"** page (space `EF`, page id `2000252`):

- Parent: **"CX Automated CD Pipeline"** (id `2224783361`) — overview, the 3 real manual clicks today (`detect_charts`, `bump:<site>`, `deploy:<site>` — NOT the "two decisions" target design, which isn't built yet), Phase 1 scope statement.
- Children: **Branch & CI/CD Variable Setup** (`2224685062`), **Onboarding an Existing (Brownfield) Site** (`2223308840`), **Onboarding a Fresh Site** (`2223538196`), **Pipeline Coverage Boundary** (`2224357383`), **Architecture Diagram** (`2223964182`), **FAQ** (`2224291849`).

**Deliberate decision:** did NOT edit the existing DTDO-space **"Expertflow CX - Continuous Deployment Guide"** (page id `2187460669`) even though it documents the exact same `agent-desk` release-name bug found this session — that page is Umer Naveed's own POC documentation and was explicitly left untouched; the new EF-space tree is the canonical, updated source going forward. Reference the DTDO guide for ServiceAccount/token setup mechanics (already thorough there, §4.1–4.6) rather than duplicating it.

**Real tooling constraint hit:** no Confluence attachment-upload tool was available in this session — could not attach the prepared `.drawio` architecture diagram file directly. Worked around it by handing the user the raw mxGraph XML to paste into Confluence's draw.io editor via Extras → Edit Diagram — confirmed successfully embedded afterward. Also: raw `<ac:link>`/`<ri:page>` storage-format tags are NOT valid input to the page-creation tool (it expects plain `<a href>` HTML and converts internally) — using them directly causes a validation error.

**How to apply:** For any future edits to this pipeline's behavior, update the EF-space tree (not DTDO). If DTDO's routing table is ever revisited, its `agent-desk` entry has the same bug as ours did — flag it if asked to touch that page. Related: [[project-cicd-deployment-guide-crosscheck]], [[project-cicd-chart-onboarding-automation]].
