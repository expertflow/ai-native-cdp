# AI-Native Continuous Delivery Pipeline Initiative
## Original Requirements & Context Document

**Date:** January 23, 2026  
**Owner:** Jawad Bokhari, Head of Software Unit, Expertflow

---

## Initial Objective Statement

**Top-Level Objective:** Define an AI-native development process (continuous delivery pipeline)

---

## Requirements Gathering - Questions & Answers

### 1. Scope & Vision: What does "AI-native" mean in this context?

**Answer:** All of the above
- AI-assisted coding (Copilot, Cursor, etc.) as standard practice
- AI-powered testing and quality assurance
- AI-driven code reviews and security scanning
- AI for deployment automation and incident prediction

---

### 2. Primary Driver: What's driving this initiative?

**Answer:** 
- **Capacity:** Doing more with fewer people (especially given the downsizing)
- **Competitive advantage:** Positioning Expertflow as an AI-forward organization

---

### 3. Current State: CD Pipeline Maturity

**Question:** What's your current continuous delivery pipeline maturity?

**Answer:**
- **Automated CI/CD pipelines in place?** Not yet
- **CI (Continuous Integration):** Somewhat automated
- **CD (Continuous Deployment):** Basic solution can be installed based on an IaC initiative
- **What percentage of deployments are automated vs manual?** Mostly manual, especially deployment preparation

**Biggest bottlenecks identified:**

#### Developer commit → merged to main
- Not impressive time
- Developers have cognitive load for environment setup
- Partially manual feature testing

#### Merged code → deployed to staging ⚠️ **WORST PAIN AREA**
- Staging environment is NOT made ready on daily basis (no nightly builds)
- Big-bang release is packed when it's about to be released
- Most issues occur at this stage

#### Staging → production release
- Challenges related to product environment configuration management
- Customers are scared of upgrading the solution because of possible errors

---

### 4. Current AI Tool Usage

**Question:** What AI tools are already being used (if any) by your teams today?

**Answer:**
Developers are using:
- **Cursor**
- **Gemini CLI**
- **AntiGravity**
- **Claude Code**

**Current state:** Ad-hoc usage with no organizational guidance or standardization

---

### 5. Timeline & Checkpoint Goals

**Question:** What's your timeline? By March 2026, what would represent meaningful progress?

**Answer:**
- **Checkpoint:** March 2026
- **Not sure what target should be set**
- **Approach:** 
  1. List down the manual process flow first
  2. Identify steps that can be automated via rule-based bots or autonomous agents
  3. Identify where humans will or should be involved
  4. Use **SAFe Continuous Delivery Pipeline framework** with 4 key areas
  5. Progress on each area independently at the top level

---

### 6. Key Stakeholders

**Question:** Who are your key stakeholders for this initiative?

**Answer:** All of them
- Product Managers (need to understand new workflow)
- Architects and technical leads (whom I assist)
- DevOps team (noted as "poorly performing")
- **Developers as well** (all ~30 of them)

**Reasoning:** "As a head of software unit, I need to be an orchestrator of the complete development pipeline"

---

## Detailed Requirements - Follow-up Questions

### 7. Which SAFe areas are the biggest pain points?

**Answer:** 
- **Continuous Exploration (CE)**
- **Continuous Deployment (CD)**

---

### 8. Current Cycle Time (Baseline - Approximate)

**Developer commit → merged to main:**
- Don't have exact figures
- Time is not impressive
- Developers have cognitive load for environment setup
- Partially manual feature testing

**Merged code → deployed to staging:**
- **This is amongst the worst pain areas**
- Most issues occur here
- Staging environment not made ready on daily basis (no nightly builds)
- Big-bang release packed when about to be released

**Staging → production release:**
- Challenges related to product environment configuration management
- Customers scared of upgrading solution because of possible errors

---

### 9. Team Capacity Context

**How many developers actively contributing code?** Nearly 30

**Time spent on various activities:**
- **Deployment preparation:** THE MOST time-consuming
- **Documentation:** Takes some time, feature documentation not always done nicely
- **Code review:** Not religiously followed everywhere

---

### 10. Process Flow Mapping Timeline

**Question:** For "listing down manual process flow first" - when should this be completed?

**Answer:** Complete within a week (by end of January 2026)

**Ownership:** "I'll do this myself"

---

### 11. Quick Wins vs. Foundation Building

**Question:** Would you rather show tangible automation wins in 1-2 areas by March, or use full quarter for thorough mapping?

**Answer:** **Option A** - Show tangible automation wins in 1-2 areas by March (even if small) to build momentum

---

### 12. AI Tool Standardization

**Question:** Do you want to standardize on specific tools by March?

**Answer:** Yes, I want to do that

**Preferences:**
- **Cursor** is recommended for Developers
- **Product Managers and others** might use Gemini and Gemini CLI
- Different tools should serve different purposes (e.g., Cursor for coding, Claude Code for architecture)
- Need to limit to a few tools for standardization (for simplicity)

---

### 13. Biggest Organizational Hurdle

**Question:** What's your biggest organizational hurdle?

**Answer:** 
**"Clarity"** - Teams don't know what they have to do

**Other constraints:**
- Cannot spend lot of money on trainings
- **Collaboration is a challenge** - mostly people working remotely
- Difficult to convince or coach them on importance of collaboration remotely

---

### 14. Success Story for March 2026

**Question:** If you're presenting progress in March 2026, what would make you (and leadership) feel this was a successful quarter?

**Answer:**
1. **Tool finalized** and everyone has started using these tools in their capacity
2. From a **process flow perspective**, some AI agents should be part of the workflow even though basic
3. **Some level of automation** should also be there

---

### 15. Nightly Builds - Current Blocker

**Question:** Is the issue technical capability (IaC not complete) or process discipline (no one owns/runs it)?

**Answer:** 
- **It's a process discipline issue** (not technical)
- **Release Management team** owns CI/CD mostly
- They are currently managing it and will continue doing that

---

### 16. Quick Win Priority

**Question:** Which would give you more visible impact by March - CE automation or CD automation?

**Answer:** **CD automation**

**Focus:** Addressing "big-bang release" and "customer fear of upgrades" problem

---

### 17. Guild Launch & Facilitation

**Question:** Can you commit to bi-weekly facilitation yourself, or need to designate someone?

**Answer:** I would like **Mariyam Mahar to facilitate this**

**Format:** Virtual-only given remote work situation

---

### 18. Measurement Baseline

**Question:** Can you have developers track their time for 1 week on deployment prep, manual testing, environment troubleshooting?

**Answer:** Sure, I can ask

---

## Context from "Currently doing at Expertflow" Document

### Current Responsibilities
- Steer, align, and track progress of teams towards product vision
- Structure teams and individuals as cross-functional teams
- Help teams identify priority and set points of focus
- Review and structure product documentation
- Assist architects and technical leads
- Keep an eye on development process, continuous delivery pipeline, and security standards
- Make recruitment decisions for different roles within development teams
- Mentor junior managers and leads
- Taking care of product documentation

---

## 2026 Targets Context (from Notion Notes)

### Primary Goals for 2026
1. Set a clear target for March 2026 covering:
   - Product readiness targets
   - Process improvements
   - Personal goals

2. Define new team structure and team performance KPIs

3. Define clear roles and responsibilities of product managers with incentivized structure

4. Grow Communities ("Guilds") for collaborative learning and knowledge sharing

5. Push for office-based work (remote working is not working)

6. Organize social virtual room discussions once every two weeks

### Process & Backlog Challenges
- Backlog organization needs work: Create templates and process around backlog orchestration
- Reorganize as initiatives and epics
- Create a system for storing and processing ideas
- Improve release breakdown process

---

## Constraints & Challenges Summary

### Team & Morale
- 30 active developers
- Recent layoffs (Q2-Q4 2025)
- Salary delays affecting morale
- Mostly remote work (identified as not working)
- Difficult to bring team members to office
- Poor collaboration due to remote setup

### Budget
- Cannot spend lot of money on trainings
- Limited budget for AI tool licenses
- Need cost-effective solutions

### Technical Debt
- DevOps team noted as "poorly performing"
- No nightly builds currently
- Big-bang releases causing integration issues
- Manual deployment preparation very time-consuming
- Configuration management challenges
- Code reviews not consistently followed
- Feature documentation not always done well

### Organizational
- Teams lack clarity on what to do
- Product Managers need to develop appetite for learning (noted as unsuccessful in 2025)
- Community culture ("Guilds") attempted but struggled in 2025
- Development/release management process needs salient improvements

---

## Strategic Context

### Recent History (2025)
**Major Accomplishment:**
- Delivered multi-tenant SaaS platform from demonstration (Q1) to production serving MindBridge tenants (Q4)

**Challenges in 2025:**
- Could not develop appetite for learning in Product Managers
- Could not make salient improvements in development/release management processes
- Could not pump community culture ("Guilds") for learning
- Could not bring team members to office
- Could not improve team morale amid salary delays
- Attempted AI learning initiative - unsuccessful
- Attempted internal collaboration improvements starting July 2025 - found impractical remotely

### Career Context
- 20+ years experience in software development
- 11 years at Expertflow (since May 2008)
- Head of Engineering/Director of Products role
- Targeting senior leadership roles: Head of Engineering, GM Software Development, Director of Products
- Focus on SaaS/Product companies in Pakistan
- Expertise in organizational transformation, product development, contact center platforms, CX systems, VoIP/telecom

---

## Key Success Factors Identified

### For This Initiative to Succeed
1. **Clarity** - Teams need to know what to do (biggest issue)
2. **Quick wins** - Show tangible results by March to build momentum
3. **Standardization** - Limit tools to 2-3 for simplicity
4. **Process discipline** - Not just technology, but workflow changes
5. **Community learning** - Guilds facilitated by Mariyam Mahar
6. **Measurement** - Baseline metrics to show improvement
7. **Focus** - CE and CD only in Q1, not trying to do everything
8. **Leadership visibility** - You orchestrating as Head of Software Unit

---

## Deliverables Expected by March 2026

### Must-Haves
1. ✅ Process flows documented (CE and CD) - completed in Week 1
2. ✅ AI tools standardized and rolled out - everyone using them
3. ✅ Some AI agents integrated into workflow (even if basic)
4. ✅ Nightly builds running (addressing big-bang release problem)
5. ✅ Some level of automation achieved
6. ✅ Measurable reduction in deployment preparation time
7. ✅ Guild sessions running (6 sessions total)

### Success Metrics
- Tool adoption: 70%+ by March
- Deployment prep time: 25% reduction
- Configuration issues: 30% reduction
- Use cases documented: 20+
- Guild attendance: 20+ per session

---

## Implementation Philosophy

### Your Approach
- **Methodical and iterative** - Build foundation before layering achievements
- **Comprehensive documentation** - Detailed representation of experience
- **Focus on transformation** - Not just tools, but organizational change
- **Hands-on leadership** - You'll do process mapping yourself
- **Practical over perfect** - Quick wins to build momentum
- **Team enablement** - Guild-based learning, not top-down mandates

### SAFe Framework Alignment
Using SAFe's 4 aspects of Continuous Delivery Pipeline:
1. **Continuous Exploration (CE)** - Q1 2026 focus
2. **Continuous Integration (CI)** - Somewhat automated already
3. **Continuous Deployment (CD)** - Q1 2026 focus (PRIMARY)
4. **Release on Demand (RoD)** - Future state

---

## Questions Still to Be Determined

1. Specific product readiness targets for March 2026 (you mentioned wanting to define this)
2. Exact budget available for AI tool licenses
3. Specific KPIs for team performance (part of 2026 goals)
4. Personal development goals beyond this initiative
5. How to connect this to backlog orchestration improvements
6. How to measure "competitive advantage" positioning

---

**This document captures the full context of your requirements for the AI-Native Continuous Delivery Pipeline initiative. Use it alongside the execution plan to maintain clarity on why decisions were made and what constraints you're working within.**