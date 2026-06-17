# Object Model Decoupling Epic — Nabeel

**Type:** project
**Tags:** jira, object-model, nabeel, rmt, tier-1, deployment

Nabeel Ahmad has started the object-model work (previously flagged as something he "was supposed to do") under epic **[CIM-33653 — CX Object Model Decoupling (The Distributed Monolith Deployment Bottleneck)](https://expertflow-docs.atlassian.net/browse/CIM-33653)**.

- Status: **In-Progress** (as of 2026-06-17). Created 2026-06-09. Reporter & assignee: Nabeel Ahmad. Priority: Major.
- Scope: migrate ~40 microservices off shared compiled Java classes → language-neutral contract system (central schema registry, automated pipelines, explicit versioning, loose data binding). Goal: eliminate lockstep deployment bottleneck + close Node.js validation gap.
- Background/deep-dive Confluence: https://expertflow-docs.atlassian.net/wiki/x/UICOPg
- Maps to the Tier 1 "object modeling data" / multipath-upgrade liability raised in the Road-to-CDP meetings (microservices delivered as monoliths, manual project model versioning).
- No subtasks/comments on the epic yet — story breakdown not in Jira as of 2026-06-17.
