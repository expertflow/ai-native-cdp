# AGENTS.md — AI-Native CDP Working Agreement

Shared, tool-neutral instructions for **any** AI assistant (Claude Code, Cursor, Codex, Gemini, ChatGPT, etc.) used by **any** team member working in this repository. The goal is consistent behavior regardless of which AI subscription a teammate uses. Read this before acting.

## What this repo is

The synthesized plan, priorities, and progress for the AI-Native CDP / CI-CD, test-automation and delivery-process initiative. It is the single place where the overall picture is maintained — not individual tickets (those live in Jira) and not real-time chatter (that lives in Google Chat).

## Authoritative documents

| Document | Role |
|----------|------|
| [`_bmad-output/planning-artifacts/priority-list-cicd-test-automation-security-consolidated.md`](_bmad-output/planning-artifacts/priority-list-cicd-test-automation-security-consolidated.md) | **Active master** priority list (T1–T4 items, owners, status, Jira links). Edit THIS copy. |
| [`_bmad-output/planning-artifacts/priority-list-cicd-test-automation.md`](_bmad-output/planning-artifacts/priority-list-cicd-test-automation.md) | Original artifact — preserved unchanged for provenance. Do not edit. |
| [`ai_native_cdp_master_pipeline.md`](ai_native_cdp_master_pipeline.md) | Target pipeline architecture, branch/release model, and roadmap checkboxes. |

## Core rule — keep the work plan in sync

When you learn a **new fact about work assignments, Jira epics/tickets, ownership, or delivery progress** (from a meeting note, a Jira lookup, or the user telling you directly), record it in the **authoritative priority list** above — not only in personal/agent memory. Personal memory is per-user and invisible to teammates; the priority list is the shared truth.

Concretely, when something changes:
1. Update the relevant T-item in the consolidated priority list: add/update its **Jira link, assignee, and status**, and add a dated `**Update (YYYY-MM-DD):**` note if the situation changed.
2. Update the **owner index** table at the bottom so the person's row reflects the item.
3. Update roadmap checkboxes in `ai_native_cdp_master_pipeline.md` where applicable.
4. Commit and push with a clear message; do not commit unless the user asks or the change is a plan update of this kind.

Place an item under the T-number whose described problem it addresses. If a new Jira epic maps to an existing item, link it there rather than creating a parallel list (e.g. CIM-33653 → T2-8, the Object Model decoupling blocker).

## External systems — always via MCP, never ad-hoc

- **Jira & Confluence** — via the Atlassian MCP server. This initiative spans the **CRM** project (e.g. CRM-706 multi-version upgrade, CRM-763 CD pipeline) and the **CIM** project (e.g. CIM-33653 Object Model decoupling). Jira ticket status is authoritative for individual items.
- **GitLab** (`https://gitlab.expertflow.com`) — always use the `gitlab` MCP server. Never use the `glab` CLI or raw `curl`/`fetch` for GitLab API calls.

## Communication

Primary channel: **Google Chat space "AI-Driven QA, CI/CD"** — `spaces/AAAAyffa-Sk` (https://chat.google.com/room/AAAAyffa-Sk).
When a PR is opened, a significant change lands, or a noteworthy update is ready, post a concise message there (what changed, why, and a link).

## Progress-sync workflow (on demand)

When a maintainer asks to "sync progress" or "update the plan":
1. Query Jira for the status of every ticket linked in the priority list (known: CRM-706, CRM-763, CIM-33653; search for others tagged to this initiative).
2. Update the consolidated priority list — one status line per affected T-item, per the Core rule above.
3. Update the master pipeline roadmap checkboxes where applicable.
4. Commit and push with a summary of what changed.

Cadence is on-demand (manually triggered), not scheduled.
