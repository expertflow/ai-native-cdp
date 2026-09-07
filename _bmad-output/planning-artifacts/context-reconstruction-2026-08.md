# AI-Native CDP — Context Reconstruction

> **Prepared:** 2026-08-29  
> **Purpose:** Reconstruct the programme's current context from the repository and the AI-Driven QA, CI/CD Google Chat space. This is a baseline for discussion, not a progress report or a replacement for Jira verification.

## Executive read

AI-Native CDP is **not an abandoned programme**. Its repository became stale after late July while delivery discussions and some work continued in Google Chat and Jira. The result is a coordination failure: the written plan is detailed but does not reliably describe the programme's current state.

The programme's original aim remains sound: make releases faster and more predictable, reduce manual coordination, and make security and quality gates meaningful. The practical near-term loop is:

`release-candidate change → deploy → regression evidence → rollback or notification`

The broader AI-native vision (specification, testing, documentation, and release evidence) is still largely a target model rather than a consistently practiced delivery method.

## What is established

### Design and prior implementation

- The project has a substantial target architecture and a detailed CD implementation design covering environment-aware deployment, test execution, notification, rollback, and release-state tracking.
- Automated Playwright regression and Google Chat notification were integrated into the release workflow in June. The written plan records that they were still manually triggered pending the CD pipeline.
- The GitLab CD work was subsequently expanded to cover real site, namespace, tenant, and configuration differences. This was necessary ground-truthing, but it made the core delivery track materially larger than the original proof of concept.
- Test reliability, coverage integrity, security enforcement, migration safety, and release governance were intentionally decomposed into parallel tracks with explicit convergence gates.

### Evidence from later team discussion

- Through July and early August, team members continued to report work on CD pipeline integration, deployment scenarios, security, test management, and test-case recovery.
- The CD pipeline was being integrated first with an internal cloud environment, with a later plan to release it to stream-aligned teams. The latest chat evidence reviewed does not confirm the final production-ready state of that rollout.
- Team members were still reporting manual update pain across multiple development instances in early August. This supports the original premise that deployment automation remains a real operational need.
- Test management became an active follow-on topic. A tool evaluation was shared in late August, with feedback still pending; this is newer than the repository's active plan.

## Current state by workstream

| Workstream | Best supported state | Confidence | What remains unknown |
|---|---|---:|---|
| Core CD pipeline | Active work continued after the repository plan; deployment scenarios and integration were being tested. | Medium | Whether the full environment-aware loop is live, who owns its remaining steps, and what is demonstrably automated today. |
| Automated regression | A CI-integrated regression and notification capability exists, but the plan warned that the suite was not yet trusted enough for release governance. | Medium | Current reliability, actual coverage, whether it is triggered automatically after deployment, and what releases it protects. |
| Test management | Tool selection/evaluation was active in August. | Medium | Selected tool, Jira integration status, recovered test cases, and actual adoption. |
| Security | A new task and an intention to improve the pipeline security posture were reported in early August. | Low | Implemented controls, verification evidence, and whether the planned blocking gates are live. |
| Release process / dark launches | A later design problem was identified: urgent customer-site deployments can bypass the declared-state mechanism and cause upgrade drift. | High | The agreed operational mechanism and whether it has been implemented. |
| Shared programme management | The repository's stated active-master document is absent and the remaining priority list is dated. | High | A current, Jira-verified source of truth. |

## Why momentum was lost

This is not primarily an effort or team-quality problem. The programme accumulated five mutually dependent but separately owned tracks:

1. environment-aware deployment;
2. trustworthy automated tests;
3. release and rollback safety;
4. security enforcement; and
5. test-case governance.

Each is legitimate, but the shared plan was not maintained as work moved into chat and Jira. That made the programme hard to steer as a single initiative and obscured which outcomes had actually landed.

## Boundaries for the next discussion

- Do not treat the repository's last status as current fact without Jira or implementation verification.
- Do not restart the entire programme from its target architecture.
- Do not call the programme AI-native merely because it contains AI-agent concepts; the reusable proof will be a repeatable, observed delivery practice.
- First recover an accurate current-state view, then choose one bounded outcome with an accountable owner and a short evidence cycle.

## Sources consulted

- `AGENTS.md` and `.ai/memory/MEMORY.md`
- `ai_native_cdp_master_pipeline.md`
- `docs/How_We_Work.md`
- priority list, maturity audit, objective-realignment plan, CD implementation plan, project brief, and checkpoint notes under `_bmad-output/planning-artifacts/`
- relevant project-memory notes under `.ai/memory/`
- AI-Driven QA, CI/CD Google Chat history, with emphasis on the May–August 2026 programme messages

## Next step

Before proposing a new initiative, reconcile the project's active work against Jira and the implementation repositories. The output should be a small, current workboard: completed, in progress, blocked, owner, and next evidence needed. Only then select the limited-scope reference initiative.
