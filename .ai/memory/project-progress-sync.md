**Type:** project
**Tags:** jira, progress, sync, priority-list, workflow

## Progress Sync — How This Project Tracks Status

**Three-layer model:**
- **Jira (CRM project)** — individual work items and ticket status (authoritative)
- **Google Chat space** (`spaces/AAAAyffa-Sk` — "AI-Driven QA, CI/CD") — real-time status updates and discussion
- **This git repo** — synthesized overall picture: plan, priorities, and progress against them

**How to apply:** When Jawad asks to sync progress or update the plan, do the following:
1. Query Jira CRM project for status of all tickets linked in the priority list (known: CRM-763, CRM-706; search for others tagged to this initiative)
2. Update `_bmad-output/planning-artifacts/priority-list-cicd-test-automation.md` — add/update a status line per T1/T2/T3 item based on Jira ticket state
3. Update `ai_native_cdp_master_pipeline.md` roadmap section checkboxes where applicable
4. Commit and push with a clear commit message summarizing what changed

**Cadence:** On-demand (Jawad triggers manually, not scheduled).
