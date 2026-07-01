---
name: feedback-integration-tests-scope
description: Integration tests belong at microservice CI level only — not in cim-solution pipeline
metadata:
  type: feedback
---

Integration tests should only be added at the **microservice CI pipeline** level, not in the cim-solution skeleton pipeline.

**Why:** The Playwright regression cycle in cim-solution already covers end-to-end validation after deployment. Adding a separate integration test stage in cim-solution would be redundant with what regression already catches. The value of integration tests is the early feedback loop per service — catching Keycloak mismatches, async issues, and cross-component contract failures before the build ever reaches RMT.

**How to apply:** When suggesting where to add integration tests, only recommend the microservice CI pipeline (after unit tests, before image build). Never propose a separate integration-test stage in cim-solution or the skeleton pipeline. Confirmed July 1, 2026 in session with Nabeel, Umar Ikhlaq, and Haroon.