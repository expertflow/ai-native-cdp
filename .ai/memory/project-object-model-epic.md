# Object Model Decoupling Epic — Nabeel

**Type:** project
**Tags:** jira, object-model, nabeel, rmt, tier-1, deployment

Nabeel Ahmad has started the object-model work (previously flagged as something he "was supposed to do") under epic **[CIM-33653 — CX Object Model Decoupling (The Distributed Monolith Deployment Bottleneck)](https://expertflow-docs.atlassian.net/browse/CIM-33653)**.

- Status: **In-Progress** (last synced 2026-06-22). Created 2026-06-09. Reporter & assignee: Nabeel Ahmad. Priority: Major.
- Scope: migrate ~40 microservices off shared compiled Java classes → language-neutral contract system (central schema registry, automated pipelines, explicit versioning, loose data binding). Goal: eliminate lockstep deployment bottleneck + close Node.js validation gap.
- Background/deep-dive Confluence: https://expertflow-docs.atlassian.net/wiki/x/UICOPg
- Maps to the Tier 1 "object modeling data" / multipath-upgrade liability raised in the Road-to-CDP meetings (microservices delivered as monoliths, manual project model versioning).
- Schema repo: `github.com/expertflow/object-model` (`json-schema` branch) — linked by Nabeel Jun 17.

## Child Issue Status (as of 2026-06-22)

| Issue | Summary | Status | Notes |
|-------|---------|--------|-------|
| CIM-33688 | Create Central `cx-schema` Repository | In-Review | Schema repo on `json-schema` branch (Jun 17) |
| CIM-33689 | Setup Initial JSON Schemas with Metadata & Versioning | In-QA | Schemas published to GitHub (Jun 17) |
| CIM-33690 | Build Automated Multi-Language Publishing Pipelines | In-Progress | "POC for publishing OM is completed" — Nabeel (Jun 22) |
| CIM-33692 | Implement Loose Data Binding & Runtime Verification (Java) | In-Progress | "Done in facebook connector" — used as end-to-end POC service (Jun 22) |
| CIM-33691 | Build Automated CI Compatibility Gate | Open | Not started |
| CIM-33693 | Implement Dynamic Validation in Node.js Services | Open | Not started |

**Why:** Nabeel is ahead of where the memory last recorded ("no subtasks/comments"). Three stories in active progress as of today; first real-service proof of loose data binding (Facebook connector) completed.
