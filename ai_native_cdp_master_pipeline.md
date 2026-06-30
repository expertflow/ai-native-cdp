# AI-Native Continuous Delivery Pipeline — Complete Lifecycle

**Document Purpose:** Define the end-to-end development process from inception to release, covering every phase of the CI/CD pipeline with continuous documentation embedded at each stage.

**Owner:** Jawad Bokhari, Head of Software Unit  
**Last Updated:** May 22, 2026  
**Status:** Living Document — Updated Quarterly

---

## 1. Pipeline Overview: Inception → Release

The CDP is not a set of disconnected tools. It is a single coherent system where **quality gates are shifted left** and **documentation is generated continuously** rather than written retroactively.

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              COMPLETE DEVELOPMENT LIFECYCLE                                          │
├─────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                      │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐      │
│  │  CONTINUOUS  │   │  CONTINUOUS  │   │  CONTINUOUS  │   │  CONTINUOUS  │   │   RELEASE    │      │
│  │  EXPLORATION │ → │  INTEGRATION │ → │  DEPLOYMENT  │ → │ DOCUMENTATION│ → │  ON DEMAND   │      │
│  │     (CE)     │   │     (CI)     │   │     (CD)     │   │     (CDoc)   │   │    (RoD)     │      │
│  └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘      │
│         │                 │                 │                 │                 │                 │
│    ┌────┴────┐      ┌────┴────┐      ┌────┴────┐      ┌────┴────┐      ┌────┴────┐              │
│    ▼         ▼      ▼         ▼      ▼         ▼      ▼         ▼      ▼         ▼              │
│  IDEA    STRUCTURED BUILD    TEST   STAGE   PROD    AUTO    LIVING  FEATURE  DARK    │
│         REQUIRE  & VERIFY & QUALITY DEPLOY  DEPLOY  DOCS    DOCS   TOGGLE   LAUNCH   │
│                                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Phase-by-Phase Breakdown

### Phase 1: Continuous Exploration (CE) — Inception to Ready-for-Dev

**Goal:** Transform raw customer needs into validated, well-structured requirements ready for development.

| Stage | Activity | AI Agent / Tool | Continuous Documentation Output |
|-------|----------|----------------|--------------------------------|
| **1.1 Customer Ingestion** | Capture customer need, support ticket, or market insight | **Gatekeeper Agent** — validates completeness, interrogates missing context (Persona, Trigger, Outcome) | Structured requirement card with completeness score |
| **1.2 Backlog Organization** | Prioritize and organize backlog items | **PM Agent** (Gemini CLI) — suggests prioritization, identifies dependencies | Updated backlog with rationale, dependency map |
| **1.3 Requirements Elaboration** | Convert need into structured PRD / user stories | **Spec Agent (MVP)** — meeting notes → structured PRD + user stories | PRD v1.0, User Stories with acceptance criteria |
| **1.4 Rapid Prototyping** | Generate visual understanding before development | **Rapid Prototyper** — Mermaid.js diagrams, wireframe descriptions from stories | Sequence diagrams, UI flow documentation |
| **1.5 Technical Feasibility** | Assess technical impact and tech debt risk | **Tech Debt Sentinel** — scans Code Architecture Index + Tech Debt Backlog | Feasibility report, risk scoring, ADR if needed |
| **1.6 Design & Architecture** | Create technical design and architecture | **Architect Agent** (Claude Code) — generates technical specs, patterns | Technical Design Document, Architecture Decision Record |
| **1.7 Implementation Ready** | Final review before dev handoff | **Readiness Checker** — validates story completeness, acceptance criteria | "Ready for Dev" gate sign-off with checklist |

**Continuous Documentation in CE:**
- Every requirement automatically gets a completeness score
- PRDs are versioned and linked to originating customer need
- Architecture decisions are recorded as ADRs at the point of decision
- Feasibility reports become living documents updated as context changes

---

### Phase 2: Continuous Integration (CI) — Code to Validated Artifact

**Goal:** Ensure every code change is built, tested, and validated automatically before merging.

| Stage | Activity | AI Agent / Tool | Continuous Documentation Output |
|-------|----------|----------------|--------------------------------|
| **2.1 Developer Onboarding** | Environment setup, context loading | **Dev Agent** (Cursor) — code completion, context awareness | Developer environment manifest, context notes |
| **2.2 AI-Assisted Development** | Write code with AI pair programming | **Cursor / Claude Code** — coding, refactoring, debugging, unit test generation | Code with inline documentation, AI-generated tests |
| **2.3 Code Review** | Review pull requests for quality, security, standards | **Code Review Agent** — automated first-pass review, flags issues | Review report with severity, suggested fixes |
| **2.4 Build Automation** | Compile, package, create artifacts | CI Pipeline (Jenkins/GitHub Actions) | Build log, artifact manifest, SBOM |
| **2.5 Automated Testing** | Unit tests, integration tests, static analysis | **Test Agent** — self-healing tests, coverage expansion | Test report, coverage metrics, quality gate status |
| **2.6 Merge to Main** | Integrate validated code | CI Pipeline + branch protection | Merge commit log, validated artifact |

**Continuous Documentation in CI:**
- Every PR includes AI-generated summary of changes
- Test reports are published automatically per build
- Code coverage trends are tracked and documented
- Security scan results become part of artifact metadata

---

### Phase 3: Continuous Deployment (CD) — Artifact to Production

**Goal:** Deploy validated artifacts through staging to production safely and repeatably.

| Stage | Activity | AI Agent / Tool | Continuous Documentation Output |
|-------|----------|----------------|--------------------------------|
| **3.1 Nightly Staging Builds** | Automatically deploy main branch to staging | CI/CD Pipeline (automated) | Deployment log, environment state snapshot |
| **3.2 Environment Validation** | Verify staging environment configuration | **Config Validator Agent** — checks configs against standards | Configuration compliance report |
| **3.3 Smoke Testing** | Validate deployment health post-deploy | **Smoke Test Agent** — automated health checks | Smoke test report, pass/fail with diagnostics |
| **3.4 Integration Testing** | End-to-end validation in staging | **Synthetic Customer Agent** — E2E test automation | E2E test report, regression detection |
| **3.5 Release Packaging** | Prepare production-ready release | **Release Storyteller Agent** — auto-generates release notes | Release notes (customer-facing, technical-internal, Confluence) |
| **3.6 Production Deployment** | Deploy to production with safety checks | CI/CD Pipeline + approval gates | Production deployment log, rollback snapshot |
| **3.7 Post-Deploy Verification** | Validate production health | **Golden Path Verification** — automated checks | Production health report, incident detection |

**Continuous Documentation in CD:**
- Every deployment is automatically documented with what changed, why, and by whom
- Release notes are generated from Jira versions + git history (multi-audience)
- Configuration drift is detected and documented
- Post-deployment health checks create a living operational record

---

### Phase 4: Continuous Documentation (CDoc) — Living Documentation System

**Goal:** Ensure documentation is never stale, always generated from source, and serves multiple audiences.

| Stage | Activity | AI Agent / Tool | Output |
|-------|----------|----------------|--------|
| **4.1 Code Documentation** | Generate API docs, READMEs from source | **Doc Agent** — parses code, generates API docs | Auto-generated API documentation, updated READMEs |
| **4.2 Architecture Documentation** | Maintain architecture diagrams and decision records | **Arch Doc Agent** — reads code, updates diagrams | Current architecture diagrams, updated ADRs |
| **4.3 Runbooks & SOPs** | Document operational procedures | **Ops Doc Agent** — infers from deployment logs, configs | Operational runbooks, troubleshooting guides |
| **4.4 Customer Documentation** | Update user guides, release notes | **Tech Writer Agent** — transforms technical docs to user-friendly | Customer-facing docs, FAQ updates |
| **4.5 Knowledge Base** | Capture team learnings, tips, patterns | **Guild Knowledge Agent** — aggregates from Guild sessions | Team knowledge repo, best practices library |
| **4.6 Documentation Validation** | Ensure docs are current and accurate | **Doc Validator Agent** — cross-references code vs docs | Documentation freshness report, gap analysis |

**Documentation Principles:**
- **Never write twice:** Documentation is generated from source (code, configs, tickets, commits)
- **Audience-aware:** Same change generates different docs for developers, ops, customers, leadership
- **Living, not static:** Docs update automatically when source changes
- **Embedded in workflow:** Documentation is a pipeline stage, not a post-release task

---

### Phase 5: Release on Demand (RoD) — Controlled Customer Exposure

**Goal:** Release features to customers when business-ready, with full control and observability.

| Stage | Activity | AI Agent / Tool | Continuous Documentation Output |
|-------|----------|----------------|--------------------------------|
| **5.1 Feature Toggles** | Enable/disable features per customer | Feature Flag System | Feature flag state documentation, rollout plan |
| **5.2 Canary Releases** | Gradual rollout to subset of users | **Matrix Validator** — checks compatibility before rollout | Canary release report, compatibility validation |
| **5.3 Dark Launches** | Deploy invisible, test in production | Deployment pipeline + monitoring | Dark launch log, verification results |
| **5.4 Customer Upgrade Assistant** | Guide customers through upgrades | **Upgrade Assistant Agent** — personalized upgrade guidance | Customer-specific upgrade instructions |
| **5.5 Rollback Decision Support** | Automated rollback recommendations | **Rollback Agent** — monitors metrics, suggests rollback | Rollback decision log, incident timeline |
| **5.6 Post-Release Monitoring** | Track feature adoption, issues | Observability stack + AI anomaly detection | Post-release dashboard, customer impact report |

**Continuous Documentation in RoD:**
- Every feature release has an automatically generated rollout plan
- Customer upgrade instructions are personalized based on their environment
- Rollback decisions are documented with data-driven rationale
- Post-release reports feed back into CE for iterative improvement

---

## 3. The Shift-Left Pipeline: Six AI Agents

These six agents form the intelligent backbone of the CDP. They do not replace humans — they **shift quality gates earlier** so humans validate rather than originate.

```
┌────────────────────────────────────────────────────────────────────┐
│                    THE SIX AI AGENTS                               │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. THE GATEKEEPER          │  2. THE RAPID PROTOTYPER            │
│     Ingest & Sanitize            Visualize Before Build           │
│     Input: Raw need              Input: User story                │
│     Output: Structured req       Output: Mermaid diagrams         │
│     Gate: Completeness           Gate: Shared understanding       │
│                                                                    │
│  3. THE TECH DEBT SENTINEL  │  4. THE RELEASE STORYTELLER         │
│     Assess Feasibility           Auto-Document Releases           │
│     Input: Ticket + code index   Input: Jira + git history        │
│     Output: Risk score           Output: Multi-audience docs      │
│     Gate: Go/No-go before dev    Gate: Release clarity            │
│                                                                    │
│  5. THE MATRIX VALIDATOR    │  6. THE SYNTHETIC CUSTOMER          │
│     Check Compatibility          Continuous QA Verification       │
│     Input: pom.xml, package.json Input: Acceptance criteria       │
│     Output: Compatibility report Output: Test results             │
│     Gate: Pre-deployment check   Gate: Post-deployment health     │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 4. Continuous Documentation: The Fifth Pillar

Traditionally, documentation is treated as a post-development activity. In this CDP, **documentation is a continuous pipeline stage** that runs in parallel with every other phase.

### Documentation Taxonomy

| Type | Audience | Generated By | Trigger | Storage |
|------|----------|-------------|---------|---------|
| **Requirements Docs** | PMs, Architects | Spec Agent | New PRD created | Confluence / Jira |
| **Architecture Docs** | Architects, Devs | Arch Doc Agent | Design decisions | Architecture wiki |
| **API Documentation** | Developers, Integrators | Doc Agent | Code commit | Auto-generated site |
| **Release Notes** | Customers, Support | Release Storyteller | Version cut | Confluence + Customer portal |
| **Runbooks** | DevOps, Support | Ops Doc Agent | Deployment | Internal wiki |
| **Test Reports** | QA, Devs | Test Agents | Every build | Build artifact + dashboard |
| **Knowledge Base** | All team members | Guild Knowledge Agent | Guild sessions | Shared repo |
| **Customer Docs** | End users | Tech Writer Agent | Feature release | Customer help center |

### Documentation Workflow

```
Every Code Change:
    │
    ├──→ Build & Test ──→ Test Report (auto-generated)
    │
    ├──→ Code Analysis ──→ API Docs Update (auto-generated)
    │
    ├──→ Architecture Scan ──→ Arch Diagram Update (auto-generated)
    │
    ├──→ Deployment ──→ Runbook Update (auto-generated)
    │
    └──→ Release ──→ Release Notes (auto-generated, multi-audience)
```

---

## 5. Metrics & Quality Gates

### Per-Phase Metrics

| Phase | Key Metric | Target | Measurement |
|-------|-----------|--------|-------------|
| CE | PRD creation time | 8h → 2h (75% reduction) | Time from need to ready-for-dev |
| CE | Requirement completeness | 90%+ | Gatekeeper completeness score |
| CI | Build success rate | 95%+ | Successful builds / total builds |
| CI | Code review cycle | 24h → 8h | PR open to merge time |
| CI | Test coverage trend | +5% per quarter | Line/branch coverage |
| CD | Deployment frequency | Daily (nightly) | Deployments per week |
| CD | Deployment prep time | -25% reduction | Hours spent on release prep |
| CD | Config-related failures | -30% reduction | Production incidents from config |
| CDoc | Documentation lag | 14 days → 1 day | Time from change to doc update |
| CDoc | Doc freshness score | 90%+ | % of docs current vs stale |
| RoD | Customer upgrade fear | Measurable reduction | Support tickets post-upgrade |
| RoD | Rollback rate | <5% | Rollbacks / total deployments |

### Quality Gates (Hard Stops)

| Gate | Location | Criteria | Owner |
|------|----------|----------|-------|
| **Requirement Complete** | CE → CI | Gatekeeper score ≥ 80%, Tech Debt Sentinel approved | Product Manager |
| **Code Quality** | CI → CD | All tests pass, Code Review approved, Coverage maintained | Tech Lead |
| **Config Valid** | CD → Staging | Config Validator passes, no security issues | DevOps |
| **Smoke Test Pass** | Staging → Prod | All smoke tests green, no critical issues | QA Lead |
| **Documentation Current** | Prod → Customer | Release notes generated, customer docs updated | Tech Writer |

---

## 6. Tool Stack by Phase

| Phase | Primary Tools | Purpose | Cost (Monthly) |
|-------|--------------|---------|---------------|
| CE | Gemini CLI, Claude Code | Requirements, PRDs, Architecture | $0-140 |
| CI | Cursor, Claude Code | Coding, Testing, Code Review | $600 |
| CD | Cursor, Gemini CLI | Deployment scripts, Validation | $0-200 |
| CDoc | Gemini CLI, Claude Code | Doc generation, Validation | $0-140 |
| RoD | All tools | Monitoring, Rollback, Support | $0 |
| **Total** | | | **~$700-1,100/month** |

---

## 7. Implementation Roadmap

### Q1 2026 (Completed)
- [x] Foundation: Nightly builds, tool standardization, baselines
- [x] Process mapping for CE and CD
- [x] AI tool rollout (Cursor, Gemini, Claude Code)
- [x] Guild launched (Mariyam Mahar facilitating)
- [x] Baseline metrics established

### Q2 2026 (Current)
- [ ] **Spec Agent MVP** — PRD time from 8h → 2h
- [ ] **AI Development Mandate** — No PR merged without AI-generated unit tests
- [ ] **Config Validator Agent** — Reduce config-related failures
- [ ] **Release Storyteller** — Auto-generate release notes
- [ ] **Continuous Documentation Pipeline** — Auto-generate API docs from code

### Q3 2026
- [ ] **Tech Debt Sentinel** — Full feasibility automation
- [ ] **Matrix Validator** — Pre-deployment compatibility checks
- [ ] **Synthetic Customer** — E2E test automation expansion
- [ ] **Doc Validator Agent** — Ensure docs stay current

### Q4 2026
- [ ] **Full RoD capability** — Feature toggles, canary, dark launches
- [ ] **Customer Upgrade Assistant** — Personalized upgrade guidance
- [ ] **Rollback Decision Agent** — Automated rollback recommendations
- [ ] **3x productivity metrics** — Defensible with data

---

## 8. Organizational Integration

### Roles & Responsibilities

| Role | CE Responsibility | CI Responsibility | CD Responsibility | CDoc Responsibility |
|------|------------------|------------------|------------------|-------------------|
| **Product Manager** | Own requirements, validate Gatekeeper output | Review AI-generated tests for coverage | Approve release notes for accuracy | Review customer-facing docs |
| **Architect** | Validate Tech Debt Sentinel, approve ADRs | Review AI code review for critical systems | Validate deployment architecture | Maintain architecture wiki |
| **Developer** | Clarify requirements, provide input | Own code quality, use AI tools daily | Monitor staging health | Update inline code docs |
| **QA Engineer** | Validate acceptance criteria completeness | Validate test coverage, edge cases | Validate smoke tests, E2E | Review test documentation |
| **DevOps** | Provide environment context | Maintain CI pipeline | Own CD pipeline, config management | Maintain runbooks |
| **Tech Writer** | Review PRD clarity | — | Review release notes | Own doc generation pipeline |
| **Release Manager** | — | — | Orchestrate deployments, monitor metrics | Document release procedures |

### Guild Sessions (Continuous Learning)

| Session # | Focus Area | Target Audience | Deliverable |
|-----------|-----------|----------------|-------------|
| 1-6 (Q1) | Tool onboarding, nightly builds | All | Baseline adoption |
| 7-12 (Q2) | Spec Agent, AI development mandate | PMs, Devs | PRD time reduction |
| 13-18 (Q3) | Advanced agents, validation | Architects, QA | Quality gate maturity |
| 19-24 (Q4) | RoD, customer experience | All | Full pipeline maturity |

---

## 9. Risk & Mitigation

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Low tool adoption | High | Guild sessions, peer learning, 1:1 coaching |
| Documentation becomes overhead | Medium | Auto-generation first, human review second |
| Agents produce incorrect docs | Medium | Human validation gates, freshness scoring |
| Team sees this as "more process" | High | Frame as "less manual work", show time savings |
| Remote work hinders collaboration | Medium | Async docs, recorded sessions, virtual office hours |

---

## 10. Success Definition

The CDP is successful when:

1. **Every requirement** that enters development has been validated by the Gatekeeper and has a completeness score ≥ 80%
2. **Every code change** triggers automated build, test, and documentation generation
3. **Every deployment** to staging happens automatically (nightly) and to production with one-click approval
4. **Every release** has auto-generated, audience-appropriate documentation
5. **Documentation lag** is measured in hours, not days or weeks
6. **Developers spend <25% of their time** on deployment preparation (vs current baseline)
7. **Customers upgrade without fear** because releases are small, frequent, and well-documented

---

## Reference Documents

| Document | Location | Purpose |
|----------|----------|---------|
| **This Document (Master)** | `ai_native_cdp_master_pipeline.md` | Complete lifecycle reference — the single source of truth |
| Backlog Management System | `BacklogManagement/` | Team Topologies, WIP management, agent skills |
| Historical Requirements (archived) | `docs/archive/ai_native_cdp_requirements.md` | Original Jan 2026 Q&A, baseline context |
| Historical Q1 Plan (archived) | `docs/archive/ai_native_cdp_plan.md` | Original 10-week Q1 execution plan |
| Historical Strategy (archived) | `docs/archive/AINative_Transformation_Strategy.md` | Original March 2026 strategy conversation |

---

## Appendices

### Appendix A: Key Strategic Insights

*From the March 2026 strategic assessment — these principles guide all CDP decisions.*

1. **Process change is the goal — tools are secondary**
   Tools create alignment, but "which tool" matters less than "how does work flow." The six agents enforce behavioral change that currently depends on individual willpower.

2. **The system is only as strong as its gates**
   Every phase fails if it's optional. The Gatekeeper fails if PMs can route around it. The Tech Debt Sentinel fails if architects don't act on the report. Gates must be in the critical path, not advisory.

3. **Build for staff turnover**
   Processes must be embedded in systems and tools — not dependent on specific individuals. A Jira workflow survives a PO leaving. A personal commitment doesn't.

4. **The two-track imperative**
   Internal transformation generates the content. External documentation of the journey builds the proof. Every win, every learning, every failure documented publicly builds the portfolio regardless of outcome.

5. **Finish over start**
   WIP discipline (Team Topologies principle) is the single highest-leverage process change. `WIP limit = team size × 1.5`. Pull system only. Aging items flagged at 5 days.

---

### Appendix B: The Honest Context

*Understanding the real organizational environment is essential for making the CDP realistic and effective.*

**Organizational Constraints:**
- Financial distress: delayed salaries affecting morale
- Remote work: makes cultural cohesion and behavior change significantly harder
- Leadership bandwidth: key decisions require CEO/COO approval, creating bottlenecks

**Team Reality:**
- Low morale; best people actively job hunting
- "Marriage of convenience" — team staying because market is difficult, not conviction
- Engagement gap: initiative requires personal investment in a high-stress environment

**Process Reality:**
- Discovery failures: unorganized backlog, features scoped for next lead not long-term capability
- Delivery failures: manual deployment prep, customer fear of upgrades (nightly builds now addressed)
- PM accountability gap: POs expected to act as PMs without clarity or KPIs
- Technical debt: accumulates silently until it becomes the primary bottleneck

**What This Means for the CDP:**
- Processes must be embedded in tools, not dependent on individual willpower
- Quick wins are essential to build momentum in a low-trust environment
- Documentation must be auto-generated, not a burden on already-overloaded people
- The transformation itself must demonstrate tangible time savings for developers

---

### Appendix C: Communication Templates

*Reusable templates for rolling out CDP changes and maintaining team communication.*

#### Template 1: Initiative Announcement

**Subject:** Launching AI-Native Development Initiative — Your Input Needed

Hi Team,

I'm excited to announce that we're embarking on an initiative to establish an **AI-Native Continuous Delivery Pipeline** at Expertflow. This will help us:
- ✅ Deliver faster with higher quality
- ✅ Reduce manual deployment prep and testing burden
- ✅ Position Expertflow as an AI-forward organization

**What's happening:**
- Phase 1: Mapping current development process and identifying automation opportunities
- Phase 2: Standardizing on AI tools (Cursor for devs, Gemini for PMs, etc.)
- Phase 3: Automating parts of our workflow with AI agents

**What's in it for you:**
- Less time on tedious manual work
- Better tools to amplify your productivity
- Clearer process and expectations

We're also running an "AI-Native Dev Guild" facilitated by Mariyam Mahar — stay tuned!

Questions? Let's discuss in our next standup or ping me directly.

Thanks,  
Jawad

---

#### Template 2: AI Tool Usage Guidelines Announcement

**Subject:** AI Tool Standards — Making Us All More Productive

Hi Team,

As part of our AI-Native Development initiative, we're standardizing on the following AI tools:

**For Developers:**
- **Primary:** Cursor (coding, debugging, refactoring)
- **Secondary:** Claude Code (complex architecture decisions)

**For Product Managers:**
- **Primary:** Gemini + Gemini CLI (requirements, user stories, documentation)

**For Architects/Tech Leads:**
- **Primary:** Claude Code (architecture design, technical specs)
- **Secondary:** Cursor (code reviews)

**For Release Management/DevOps:**
- **Primary:** Gemini CLI + Cursor (CI/CD scripting, automation)

**Why standardize?**
- Share best practices and learnings
- Build organizational capability, not just individual productivity
- Create reusable prompts and workflows

Let's build this capability together!

Jawad

---

#### Template 3: Guild Session Invite

**Subject:** AI-Native Dev Guild — Session [#]: [Topic] — [Date]

Hi Team,

Join us for **Session [#]** of our AI-Native Dev Guild!

**When:** [Day], [Date], 2:00–3:30 PM PKT  
**Where:** [Virtual meeting link]  
**Facilitated by:** Mariyam Mahar

**Agenda:**
- [Topic 1] (XX min)
- [Topic 2] (XX min)
- Live Demos (XX min)
- Q&A & Discussion (XX min)

**Recording will be available** for those who can't attend live.

See you there!  
Mariyam & Jawad

---

### Appendix D: AI Agent Implementation Examples

#### Example 1: Environment Configuration Validator

**Purpose:** Validate environment configurations before deployment to prevent configuration-related failures.

**Approach:**
- AI reads deployment configuration files
- Compares against known good configurations
- Identifies inconsistencies, missing values, security issues
- Suggests corrections

**Tools:** Claude Code or Gemini CLI

**Sample Prompt Template:**
```
Review this deployment configuration for [environment]:
[paste config]

Check for:
1. Missing required environment variables
2. Inconsistencies with production standards
3. Security vulnerabilities (exposed secrets, weak settings)
4. Performance optimization opportunities

Provide specific recommendations with severity levels.
```

#### Example 2: Automated Release Notes Generator

**Purpose:** Generate release notes automatically from git commit history.

**Approach:**
- AI reads commit messages between releases
- Categorizes changes (features, fixes, improvements)
- Generates customer-friendly release notes
- Includes upgrade instructions if needed

**Tools:** Gemini CLI or Claude Code

**Sample Prompt Template:**
```
Generate release notes from these commits:
[paste git log]

Format as:
## What's New
- [user-facing features]

## Improvements
- [enhancements]

## Bug Fixes
- [fixes]

## Upgrade Notes
- [breaking changes, migration steps]

Use clear, customer-friendly language.
```

---

### Appendix E: Detailed Risk Register

| Risk | Likelihood | Impact | Mitigation Strategy | Owner |
|------|-----------|--------|---------------------|-------|
| **Low tool adoption due to remote work** | High | High | Guild sessions for peer learning; share success stories weekly; make adoption part of 1:1s; recognize early adopters | Jawad |
| **DevOps team unable to implement automation** | Medium | High | Jawad directly oversees Release Mgmt; escalate blockers immediately; simplify initial implementation | Jawad |
| **Budget constraints for tool licenses** | Medium | Medium | Start with free tiers (Gemini); pilot with 10 users first; show ROI before full rollout | Jawad |
| **Guild facilitator unavailable** | Low | Medium | Jawad co-facilitates as backup; record all sessions for async; identify backup facilitator early | Mariyam |
| **Team morale too low for adoption** | High | High | Focus on reducing their pain; show quick wins early; celebrate small successes; connect to career growth | Jawad |
| **Remote work hinders collaboration** | High | Medium | Record all Guild sessions; use async communication effectively; virtual office hours for Q&A; share wins publicly | Mariyam |
| **Scope creep — trying to automate too much** | Medium | Medium | Stick to prioritized roadmap; defer non-P0/P1 items to next quarter | Jawad |
| **Documentation seen as overhead** | Medium | Medium | Auto-generate first, human review second; frame as "less manual work"; show time savings | Jawad |

---

## 11. Engineering Standards & Operational Practices

This section defines the concrete engineering practices that make the CDP operate reliably. Where Phase 2 and 3 describe *what* happens, this section defines *how* it must happen — the specifics of branching, gates, artifact management, environment promotion, team coordination, and incident response.

---

### 11.1 Branch Strategy & PR Gates

#### Branching Model

Expertflow uses a **trunk-based development with short-lived feature branches** approach, centred on two long-lived branches:

| Branch | Purpose | Who merges to it | Protection |
|--------|---------|-----------------|-----------|
| `develop` | Integration target for all completed features | Developers via MR (merge request) | CI required; Tech Lead approval required |
| `master` | Production-ready code only | RMT (Haroon) at release time | RMT-only write access; CI required |

**Feature Branch Rules:**
- Spawn from `develop` — never from `master`
- Naming convention: `feature/CIM-{Jira-ID}-short-description` (e.g. `feature/CIM-420-routing-overflow`)
- Branches must be short-lived: closed on merge, not reused
- Skeleton project feature branches must mirror the component branch and be tagged on merge

**Hotfix Branch Rules:**
- Spawn from `master` at the specific production release tag being patched
- Naming convention: `hotfix/CX{version}_b-CIM-{Jira-ID}` (e.g. `hotfix/CX5.0.3_b-CIM-515`)
- Merge into both `master` (to create new patch tag) and back into `develop` (to preserve fix)
- If patching an older release not on current `master`, apply tag directly on the hotfix branch

#### PR / Merge Request Gates

Every merge request to `develop` must pass **all** of the following before it can be merged. These are hard gates — no exceptions without a documented waiver from the Tech Lead.

| Gate | Tool | Criteria | Blocks Merge? |
|------|------|----------|--------------|
| **CI Build** | GitLab CI | Clean build with no compilation errors | Yes |
| **Unit Tests** | Jest / JUnit (per stack) | All tests pass; zero failures | Yes |
| **AI-Generated Tests** | Cursor / Claude Code | At least one AI-generated unit test included per story (Q2 mandate) | Yes (from Q2 2026) |
| **Code Coverage** | Coverage tool in CI | Coverage must not decrease from baseline; target +5% per quarter | Yes |
| **Static Analysis** | SonarQube | No new critical or blocker issues introduced | Yes |
| **Security Scan** | Trivy | No new critical or high CVEs introduced | Yes |
| **Peer Code Review** | GitLab MR | At least one reviewer other than the author must approve | Yes |
| **Tech Lead Approval** | GitLab MR | Tech Lead of the stream team must approve | Yes |
| **Jira Linked** | GitLab + Jira | MR description must reference the Jira ticket (`CIM-XXX`) | Yes |
| **No Open Sub-tasks** | Jira | All sub-tasks on the story must be closed | Yes |

> **PR Description Standard:** Every MR must include: (1) a 2–3 line summary of what changed and why, (2) a link to the Jira story, (3) testing instructions for the reviewer, and (4) any configuration or database changes called out explicitly. AI-generated PR summaries (via Cursor or Claude Code) satisfy the summary requirement.

---

### 11.2 Automated Test Suite & Quality Gates Before Merge

#### Test Layer Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    TEST PYRAMID                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│          ┌─────────────────────────┐                       │
│          │     E2E / BDD Tests     │  ← RMT + QA           │
│          │   (Playwright + Gherkin) │    Runs on staging    │
│          └─────────────────────────┘                       │
│       ┌─────────────────────────────────┐                  │
│       │      Integration Tests          │  ← Dev + QA      │
│       │   (API-level, cross-service)    │    Runs in CI     │
│       └─────────────────────────────────┘                  │
│    ┌─────────────────────────────────────────┐             │
│    │           Unit Tests                    │  ← Dev       │
│    │  (component-level, AI-generated)        │    Runs in CI│
│    └─────────────────────────────────────────┘             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### Test Repository

All E2E automation lives in a dedicated GitLab repo: `cx-automation-playwright`

```
qa-automation-playwright/
├── .gitlab-ci.yml
├── playwright.config.ts
├── package.json
├── features/           ← Gherkin scenarios (plain English)
│   ├── routing.feature
│   └── agent-desk.feature
└── tests/
    ├── pages/          ← Page Object Model (selectors + actions)
    │   ├── AgentDeskPage.ts
    │   └── WebChatWidget.ts
    └── steps/          ← Gherkin step implementations
        └── routingSteps.ts
```

#### Selector Strategy (BDD / Playwright)

Angular apps at Expertflow generate dynamic CSS classes. Use selectors in this priority order:

1. `page.getByTestId('...')` — requires `data-testid` attributes; target state once agreed with dev teams
2. `page.getByRole('button', { name: '...' })` — stable ARIA roles; current default
3. `page.getByLabel('...')` / `page.getByText('...')` — fallback for labelled fields and text
4. CSS selectors — **avoid**; brittle after every Angular build

#### Quality Gate Thresholds

| Test Type | Gate | Target | Blocking |
|-----------|------|--------|---------|
| Unit tests | All pass | 100% pass rate | Yes |
| Unit test coverage | Line coverage | ≥70% per component; ≥80% overall (trend target) | Yes (no decrease) |
| Integration tests | All pass | 100% pass rate | Yes |
| SonarQube | Blocker/critical issues | 0 new issues | Yes |
| Trivy | Critical/high CVEs | 0 new CVEs | Yes |
| E2E smoke suite | Smoke suite | 100% on staging post-deploy | Yes (blocks promotion) |
| Full regression | Automated regression | ≥95% pass rate | Yes (blocks release) |

> Current state: automated regression coverage is approximately 40%. The Q3 target is to expand the Playwright suite to achieve ≥60% E2E coverage of critical user journeys across CXAGENT, CXVOICE, and CXCHAN.

---

### 11.3 Build Artifact Packaging & Versioning

#### Version Numbering

Expertflow uses **semantic versioning**: `CX{MAJOR}.{MINOR}.{PATCH}`

| Increment | Trigger | Example |
|-----------|---------|---------|
| MAJOR | Breaking API changes or architectural shift | CX4 → CX5 |
| MINOR | New feature sets, planned releases (product-driven) | CX5.0 → CX5.1 |
| PATCH | Bug fixes, hotfixes, customer-driven maintenance | CX5.0.3 → CX5.0.4 |

Version is set in the **skeleton project** — the integration repository that stitches together component versions from all stream teams.

#### Build Artifacts

Each CI build produces the following artifacts, tagged with the build version and commit SHA:

| Artifact | Format | Registry | Retention |
|----------|--------|----------|-----------|
| Service container images | Docker | GitLab Container Registry | Tagged releases: permanent; feature builds: 30 days |
| Helm charts | `.tgz` chart package | GitLab / GitHub (resolving dual-source — see §11.5) | Tagged releases: permanent |
| Test reports | JUnit XML + HTML | GitLab CI artifacts | 90 days |
| Security scan reports | Trivy JSON + SARIF | GitLab CI artifacts | 90 days |
| SBOM (Software Bill of Materials) | CycloneDX JSON | Attached to release tag | Permanent |

#### Artifact Tagging Convention

```
# Feature build (develop branch)
image: expertflow/cx-routing:develop-{feature-tag}

# Release candidate
image: expertflow/cx-routing:CX5.1.0-rc1

# Production release
image: expertflow/cx-routing:CX5.1.0
```

The skeleton project is the authoritative source for which component version tags compose each release. The skeleton project tag (`CX5.1.0`) is created by RMT at release time and pins all component versions.

#### SBOM Policy

Every production release must include a CycloneDX SBOM attached to the GitLab release tag. The SBOM is generated automatically by the CI pipeline and lists all direct and transitive dependencies. This supports compliance, vulnerability tracking, and customer security reviews.

---

### 11.4 Environment Promotion: Dev → Staging → Production

#### Environment Map

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        ENVIRONMENT CHAIN                                 │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Feature Dev Server        RMT Staging Server       Production           │
│  (per stream team)         (shared, RMT-managed)    (customer)           │
│                                                                          │
│  ┌──────────────┐          ┌──────────────────┐     ┌──────────────┐    │
│  │ feature/     │  QA      │ develop / RC     │     │ master tag   │    │
│  │ branch build │ ──Pass──▶│ incremental      │────▶│ production   │    │
│  │              │          │ builds (nightly) │     │ deploy       │    │
│  └──────────────┘          └──────────────────┘     └──────────────┘    │
│                                                                          │
│  IaC: team-owned           IaC: RMT-managed          IaC: customer      │
│  Manual for Voice          Automated nightly          Helm + IaC         │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

#### Promotion Criteria

| Promotion | Criteria | Authorized By |
|-----------|----------|--------------|
| Feature branch → Develop | All PR gates pass (§11.1); QA-Passed in Jira; Tech Lead MR approval | CI pipeline + Tech Lead |
| Develop → RMT Staging (nightly) | Clean CI build on `develop`; no failing unit tests | Automated (CI/CD pipeline) |
| Staging RC → Production | Go/No-Go checklist complete (§4 of How_We_Work.md); smoke tests green; regression suite ≥95% pass | Haroon + Nabeel (Track 2) / PM + Ops + Haroon (Track 1) |
| Production → Rollback | Critical/blocker issue confirmed post-deploy (§11.6) | Haroon (RMT) with JB notification |

#### IaC Promotion Rules

- Each promotion step uses the IaC pipeline, not manual Helm edits
- Database backups (MongoDB + PostgreSQL) are created and uploaded to the IaC backup repository **before every staging and production deployment**
- If a new component or ConfigMap is added or removed, the IaC pipeline must be updated before the release deploys — this is a blocking gate

#### Known Environment Gaps (Action Items)

| Team | Gap | Target State | Owner |
|------|-----|-------------|-------|
| CXVOICE | No IaC; fully manual setup | IaC available by Q3 2026 | Nabeel |
| CXCHAN | Needs IaC enablement | Enabled by Q3 2026 | Nabeel + Ehtasham |
| CXBI/WFM | Servers not in IaC pool | In IaC pool by Q3 2026 | Nabeel |

---

### 11.5 Release Coordination Across Teams

Expertflow releases integrate components from multiple stream teams: CXAGENT (routing, agent desk), CXCHAN (channels: web chat, email, social), CXVOICE (CTI, telephony, RTC), CXBI (WFM, analytics), CXPLAT (platform, IAM), and Quality Management. Coordinating these streams is one of the highest-risk points in the release process.

#### Release Coordination Model

```
┌────────────────────────────────────────────────────────────────────────┐
│                     RELEASE COORDINATION FLOW                          │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Stream Teams (CXAGENT, CXCHAN, CXVOICE, CXBI, CXPLAT, QM)           │
│       │                                                                │
│       │  Signal via Jira: feature status → Release-Ready              │
│       │                                                                │
│       ▼                                                                │
│  Skeleton Project (RMT)                                                │
│  Haroon + Junaid integrate Release-Ready features incrementally        │
│       │                                                                │
│       │  Gate: Release-Ready DoD (§2.6 of How_We_Work.md)             │
│       │                                                                │
│       ▼                                                                │
│  RMT Staging — Incremental builds, regression, smoke                   │
│       │                                                                │
│       │  Gate: Go/No-Go decision (§4 of How_We_Work.md)               │
│       │                                                                │
│       ▼                                                                │
│  Production Release                                                    │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

#### Cross-Team Dependency Rules

- All stream teams must be present in Track 2 release planning — the current exclusion of CXCHAN, CXBI/WFM, and QM is a documented risk that Haroon owns to resolve
- Cross-component dependencies must be declared in Jira at story creation time (linked as "Depends on")
- If Team A's feature depends on Team B's API or component version, Team B's component must be Release-Ready **before** Team A's feature can be packaged
- Minimum version requirements (e.g. "requires CXVOICE ≥ 5.1.0") are documented in Jira comments on the feature ticket, not in verbal agreements

#### Release Cut-Off Rules

| Release Type | Cut-Off Rule | Scope Decision Authority |
|-------------|-------------|------------------------|
| Track 1 (Customer-driven) | PM + Ops decide scope; stream teams may not be consulted; shortened to meet customer deadline | PM + Ops + Haroon |
| Track 2 (Product-driven) | Collaborative cut-off agreed by Haroon + Nabeel + stream leads; late additions require explicit approval | Haroon + Nabeel |

#### Dual-Codebase Requests (Ops to Older Releases)

When Ops requires a feature from a new release to be backported to an older customer deployment:

1. Ops raises the request to JB and Haroon with the customer name and target version
2. JB and Haroon assess effort — the feature must be built and packaged on both codebases
3. Sprint planning for the stream team must account for the extra effort explicitly — it is not absorbed silently
4. The hotfix/backport branch follows the hotfix naming convention (§11.1)

> This process is currently informal. Formalizing it is a Q2/Q3 action item for JB + Haroon.

#### Jira Hygiene Requirements

Release coordination depends entirely on Jira statuses being accurate. The following are mandatory, not optional:

- Feature status updated to **QA-Ready** when ready for QA (by developer, same day)
- Feature status updated to **QA-Passed** when QA completes (by QA, same day)
- Feature status updated to **Release-Ready** only when all DoD items are checked (by developer + QA together)
- All configuration changes, DDL/DML scripts, and component version dependencies documented in Jira comments before Release-Ready status is set

---

### 11.6 Rollback & Hotfix Process

#### Rollback Decision Criteria

A production rollback is initiated when a **critical or blocker severity** issue is confirmed in production. Severity is assessed by Haroon (RMT) with input from Ops. Lower-severity issues are addressed via fast-follow patch releases, not rollbacks.

| Severity | Response | Timeline |
|----------|---------|---------|
| Critical / Blocker | Immediate rollback assessment; rollback if infrastructure-only issue | Within 2 hours of detection |
| Major | No rollback; fast-follow patch targeted for next release | Within 1 sprint |
| Minor | No rollback; patch in next scheduled release | Normal release cycle |

#### Current Rollback Capabilities

| Component | Rollback Method | Reliability | Notes |
|-----------|----------------|-------------|-------|
| Infrastructure (Helm) | Redeploy previous Helm chart version | Reliable | Use `helm rollback` via IaC pipeline |
| IaC State | Revert IaC to previous state | Reliable | Restore from IaC repo previous commit |
| Data (MongoDB) | Restore from pre-deployment backup | High risk of data loss | No transactions after backup are recoverable |
| Data (PostgreSQL) | Restore from pre-deployment backup | High risk of data loss | No transactions after backup are recoverable |

> **Critical Gap:** Database rollback has no safe path if the release includes data migrations and transactions have occurred post-deployment. This is a P0 engineering action item owned by Nabeel. Until resolved, every release with database migrations must include a data migration rollback script (reverse migration) reviewed by Nabeel before the release ships.

#### Incident Response Steps (Formal Procedure)

1. **Detect** — Issue identified in production by monitoring, customer report, or automated health check
2. **Triage** — Haroon assigns severity; notifies Ops (Google Chat) and JB within 15 minutes
3. **Log** — Jira incident ticket created within 30 minutes with: description, reproduction steps, affected customers, severity
4. **Assess** — Determine if rollback is feasible: Is it infrastructure-only (safe)? Does it involve data migrations (risky)?
5. **Decide** — Haroon + Nabeel make the rollback/hold decision. JB approves if data risk is involved.
6. **Execute Rollback (if approved):**
   - Redeploy previous Helm chart version via IaC pipeline
   - Revert IaC state
   - Restore database backup only if data corruption is confirmed and data loss risk is accepted by JB + Ops
7. **Verify** — Confirm system is stable on previous version; run smoke tests
8. **Communicate** — Haroon posts status update to `cx-release-announcements` and Google Chat; Ops notifies affected customers
9. **Post-Incident Review** — Within 5 business days: root cause analysis documented in Jira, preventive action items created, runbook updated

#### Hotfix Process

```
master (CX5.0.3 tag)
    │
    └── hotfix/CX5.0.3_b-CIM-515
            │
            ├── Develop + test fix
            ├── QA-Passed (stream QA)
            ├── Code review approved (Tech Lead)
            ├── CI pipeline green
            │
            ├──▶ Merge to master → tag CX5.0.4
            │
            └──▶ Merge to develop (prevent regression on next release)
```

**Hotfix SLA targets:**

| Severity | Fix Delivery Target | Who Decides to Hotfix |
|----------|--------------------|-----------------------|
| Critical (P1) | Within 24 hours of root cause identified | JB + Ops + Haroon |
| High (P2) | Within 1 sprint | Haroon + stream Tech Lead |
| Medium and below | Next scheduled release | Normal planning |

---

*This document is the master reference for the AI-Native CDP. Update it at each quarterly checkpoint to reflect current state and upcoming priorities.*
