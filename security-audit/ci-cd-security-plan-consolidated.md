# CI/CD Security Hardening Plan — Consolidated

> **Project:** ai_native_cdp (Expertflow)  
> **Scope:** CI/CD Pipeline Security for Node.js + Java Microservices  
> **Date:** 2026-06-09  
> **Analyst:** Mary (Business Analyst)  
> **Reviewer:** Murat (Test Architect)  
> **Stakeholder Input:** EF (Engineering Lead)  
> **Status:** Analysis Complete — Ready for Architecture Review  

---

## 1. Executive Summary

### The Business Case

Expertflow's CI/CD pipeline is the **single path to production** for all customer-facing code. Every vulnerability that passes through this pipeline has a direct path to customer environments — including regulated enterprise clients (FNB, Cisco). The current pipeline scores **28/100 on security maturity**, which places it in the "security theater" zone: tools are present but none enforce findings.

**What this means in business terms:**
- A committed AWS key in a feature branch → **production data exposure** within hours of merge
- A Log4j-equivalent in a Java dependency → **no detection until RMT or customer report**
- A tampered container image → **untrusted code deployed to customer Kubernetes clusters**

**The investment required:** ~2 developer-weeks to reach the **75/100 minimum threshold**. The cost of *not* doing this is a single CVE-driven incident that could exceed **6 months of engineering effort** in remediation, customer communication, and reputational damage.

### What This Document Does

This plan consolidates two independent security analyses:
1. **EF's self-assessment** — implementation-focused, sharp on pipeline mechanics
2. **Murat's test-architect audit** — risk-calibrated, comprehensive on controls and scoring

Mary has evaluated both, identified overlaps and gaps, and produced a **single consolidated roadmap** with business-prioritized sequencing, clear ownership, and resource-aware phasing.

---

## 2. Source Evaluation

### 2.1 EF's Self-Assessment — Strengths

| Strength | Evidence |
|----------|----------|
| **Deep pipeline mechanics knowledge** | Identified `docker:dind` privileged mode, Java `when: manual` bypass, `--legacy-peer-deps` root cause |
| **Implementation-ready snippets** | Provided exact `.gitlab-ci.yml` fragments with variable names matching existing pipeline |
| **Tool awareness** | Correctly identified TruffleHog over GitLeaks, npm audit integration |
| **Scoring ambition** | Targeted 77/100 after Phase 2 — aggressive but achievable |

### 2.2 EF's Self-Assessment — Gaps

| Gap | Impact | Why It Matters |
|-----|--------|---------------|
| **Missing Trivy filesystem scan** | Transitive dependency CVEs invisible | Container scanning misses dev deps and build tools |
| **Missing Checkov (IaC beyond Docker)** | K8s/Helm misconfigs undetected | Expertflow deploys to K8s — manifest security is critical |
| **Missing SBOM generation** | No supply chain transparency | EU Cyber Resilience Act 2027 requirement; customer asks growing |
| **Missing DAST (ZAP)** | Runtime security unvalidated | SAST catches code; DAST catches deployed misconfigurations |
| **Missing container signing** | No image provenance | Deployment-time verification impossible |
| **Score inflated (41 vs. 23 actual)** | False confidence in current state | Counted "tools present" as "controls effective" |
| **TruffleHog `--since-commit HEAD~1`** | Misses multi-commit MR secrets | Only scans last commit, not full MR history |

### 2.3 Murat's Audit — Strengths

| Strength | Evidence |
|----------|----------|
| **Risk-calibrated scoring** | 10-dimension rubric with weights; clear threshold definitions |
| **Tool justification depth** | Side-by-side comparisons (TruffleHog vs GitLeaks, Semgrep vs SonarQube) |
| **Two-layer defense model** | Test runner thresholds + SonarQube gates — catches bypass scenarios |
| **Completeness** | Covers 12 security domains vs. EF's 8 |
| **Graceful migration path** | "New Code" Sonar definition to avoid backlog blocking |

### 2.4 Murat's Audit — Gaps

| Gap | Impact | Why It Matters |
|-----|--------|---------------|
| **Underweighted docker:dind risk** | Called it "acceptable" | EF correctly identified privileged mode as MEDIUM-HIGH risk |
| **Missed Java `when: manual` bypass** | Branch builds skip scans entirely | EF found this; Murat didn't flag it |
| **No code-format re-enable** | Quality gate drift continues | EF caught this operational gap |
| **Coverage threshold discussion imprecise** | Suggested 60% initially | EF clarified Sonar already enforces 80%/90% — test runner tripwire is what's missing |

### 2.5 Synthesis: Both Inputs Are Valuable, Neither Is Complete Alone

- **EF's assessment** is sharper on **pipeline mechanics** and **implementation feasibility**
- **Murat's audit** is stronger on **risk scoring**, **tool selection rationale**, and **completeness**
- **Mary's role:** Integrate both, add business context (DRI, sequencing, bandwidth), and produce an actionable plan

---

## 3. Gap Analysis Matrix

| # | Security Control | In EF's Plan? | In Murat's Audit? | Consolidated Status |
|---|-----------------|---------------|-------------------|---------------------|
| 1 | `allow_failure: false` on Trivy/Grype | ✅ Yes | ✅ Yes | **Phase 1, Day 1** |
| 2 | TruffleHog secrets scanning | ✅ Yes | ✅ Yes | **Phase 1, Day 1** |
| 3 | `npm ci` + `npm audit` | ✅ Yes | ✅ Yes | **Phase 1, Day 2** |
| 4 | OWASP Dependency-Check (Java) | ✅ Yes | ✅ Yes | **Phase 1, Day 2** |
| 5 | Trivy filesystem scan | ❌ Missing | ✅ Yes | **Phase 1, Day 3** — EF gap |
| 6 | Semgrep SAST (Node) | ✅ Yes | ✅ Yes | **Phase 1, Week 1** |
| 7 | SpotBugs/FindSecBugs (Java) | ✅ Yes | ✅ Yes | **Phase 1, Week 1** |
| 8 | Checkov IaC scan | ❌ Missing | ✅ Yes | **Phase 1, Day 3** — EF gap |
| 9 | Dockle Dockerfile scan | ❌ Missing | ✅ Yes | **Phase 2, Week 2** |
| 10 | Hadolint Dockerfile scan | ✅ Yes | ❌ Not recommended | **Replaced by Dockle** — Murat wins; Checkov covers more ground |
| 11 | Coverage thresholds (jest/Jacoco) | ✅ Yes | ✅ Yes (corrected) | **Phase 2, Week 2** |
| 12 | Re-enable `node-format-frontend` | ✅ Yes | ❌ Missed | **Phase 2, Week 2** — EF wins |
| 13 | docker:dind hardening | ✅ Yes | ⚠️ Underweighted | **Phase 2, Week 2** — EF's finding elevated |
| 14 | Java `when: manual` fix | ✅ Yes | ❌ Missed | **Phase 1, Day 3** — EF wins |
| 15 | SBOM generation (Syft) | ❌ Missing | ✅ Yes | **Phase 2, Week 2** — EF gap |
| 16 | Container signing (Cosign) | ❌ Missing | ✅ Yes | **Phase 3, Month 2** — EF gap |
| 17 | DAST baseline (ZAP) | ❌ Missing | ✅ Yes | **Phase 3, Month 2** — EF gap |
| 18 | SLSA provenance | ❌ Missing | ✅ Yes | **Phase 3, Month 3** — EF gap |
| 19 | License compliance | ❌ Missing | ✅ Yes | **Phase 3, Month 3** — EF gap |
| 20 | Grype removal | ⚠️ Keep | ✅ Remove | **Remove Grype** — Murat wins |
| 21 | SHA digest pinning | ✅ Yes | ✅ Yes | **Phase 2, Week 2** |
| 22 | detect-secrets pre-commit | ✅ Yes | ✅ Yes | **Phase 2, Week 2** |

**Summary:**
- **EF found 4 things Murat missed:** docker:dind hardening, Java `when: manual`, format job re-enable, `--legacy-peer-deps` mechanism specificity
- **Murat found 6 things EF missed:** Trivy FS, Checkov, SBOM, signing, DAST, SLSA
- **Both agreed on 12 items**

---

## 4. Consolidated Security Roadmap

### Phase 1 — "Stop the Bleeding" (Week 1) — Score: 28 → 55

These fixes require **zero new infrastructure** and **minimal configuration**. They convert existing decorative controls into enforcing gates.

| # | Control | Owner | Effort | Business Value | Evidence of Need |
|---|---------|-------|--------|---------------|-----------------|
| 1.1 | Set `allow_failure: false` on Trivy branch + merge | DevOps Lead | 30 min | Converts existing scan from decoration to gate | Every security job currently green despite findings |
| 1.2 | Remove Grype (redundant with Trivy) | DevOps Lead | 15 min | Saves 3-5 min per build × 50 MRs/week = ~3 CI hours/week | Grype catches subset of Trivy findings; never justified its runtime cost |
| 1.3 | Set `allow_failure: false` on SonarQube (with "New Code" grace period) | DevOps Lead + Stream Leads | 2 hours | Enforces 80%/90% coverage thresholds that already exist | SonarQube quality gate waits but never blocks |
| 1.4 | Add TruffleHog to `validate` stage | Security Lead / DevOps | 2 hours | Prevents #1 cause of cloud breaches — committed secrets | No secret detection exists; CI variables referenced in YAML |
| 1.5 | Replace `npm install --legacy-peer-deps` with `npm ci` + `npm audit --audit-level=high` | Node Stream Lead | 1 hour | Restores dependency resolution integrity + adds CVE gate | `--legacy-peer-deps` suppresses npm v7+ conflict checks |
| 1.6 | Add OWASP Dependency-Check to Java pipeline (`pom.xml` + CI job) | Java Stream Lead | 3 hours | Closes Java CVE scanning gap completely | Java ecosystem historically high-risk (Log4j, Spring4Shell) |
| 1.7 | Fix Java `gitlab_build_branch` from `when: manual` to `when: on_success` with branch pattern restrictions | DevOps Lead | 1 hour | Eliminates branch-build bypass of security scans | Branch builds never trigger Trivy/Grype unless manually clicked |
| 1.8 | Add Trivy filesystem scan (`trivy fs`) to both pipelines | DevOps Lead | 1 hour | Catches transitive and dev-dependency CVEs invisible to image scan | `node_modules` stripped from final image via `--omit=dev` |
| 1.9 | Add Checkov IaC scan to `validate` stage | DevOps Lead | 1 hour | Catches Dockerfile, K8s, Helm misconfigurations before build | No Dockerfile/K8s security validation exists |

**Phase 1 Exit Criteria:**
- [ ] Every pipeline security job has `allow_failure: false`
- [ ] TruffleHog runs on every MR and develop/master commit
- [ ] `npm audit` fails on HIGH/CRITICAL CVEs
- [ ] OWASP Dependency-Check fails on CVSS ≥ 7
- [ ] Trivy FS scan fails on HIGH/CRITICAL CVEs
- [ ] Checkov fails on Dockerfile/K8s misconfigurations
- [ ] Java branch builds trigger automatically (not manual)
- [ ] Grype removed from both pipelines

**Expected Score:** 55/100 ("Developing" — core controls present and enforced)

---

### Phase 2 — "Baseline Security" (Weeks 2-3) — Score: 55 → 78

These add new tooling layers and close operational gaps. Requires **moderate configuration** but no infrastructure procurement.

| # | Control | Owner | Effort | Business Value | Evidence of Need |
|---|---------|-------|--------|---------------|-----------------|
| 2.1 | Add Semgrep SAST to Node.js pipeline (frontend + backend) | Security Lead | 4 hours | Catches injection, XSS, auth flaws SonarQube misses | SonarQube Community/Developer security rules are minimal |
| 2.2 | Add SpotBugs + FindSecBugs to Java `pom.xml` | Java Stream Lead | 3 hours | Industry-standard Java security static analysis | No Java-specific security SAST exists beyond SonarQube |
| 2.3 | Add Dockle Dockerfile scan to `scan` stage | DevOps Lead | 1 hour | Enforces CIS Docker Benchmark (root user, health checks, secrets) | No Dockerfile best-practice validation exists |
| 2.4 | Add coverage thresholds to jest config (70% tripwire) | Node Stream Lead | 1 hour | Prevents "deleted tests, green pipeline" scenario | `npm run coverage --u` passes at 0% coverage today |
| 2.5 | Add coverage thresholds to Jacoco in `pom.xml` (70% tripwire) | Java Stream Lead | 1 hour | Same protection for Java test suite | `mvn verify` has no coverage gate |
| 2.6 | Re-enable `node-format-frontend` with standard Node image | Node Stream Lead | 30 min | Restores code quality gate; prevents style drift | Job commented out since custom image became unavailable |
| 2.7 | docker:dind hardening (resource limits, seccomp profile) | DevOps Lead | 2 hours | Reduces container escape attack surface | `docker:dind` runs in privileged mode with full host access |
| 2.8 | Pin all Docker base images to SHA digests | DevOps Lead + Stream Leads | 2 hours | Immutable builds — tag overwrite impossible | `docker build --pull` fetches mutable tags |
| 2.9 | Add Syft SBOM generation on develop/master/tag builds | DevOps Lead | 2 hours | Supply chain transparency; rapid CVE response | No SBOM exists; cannot answer "Do we use Log4j?" |
| 2.10 | Add detect-secrets pre-commit hook to repo | Security Lead | 1 hour | Developer-side prevention before CI | Secrets reach CI because no local check exists |

**Phase 2 Exit Criteria:**
- [ ] Semgrep runs on every MR with `allow_failure: false`
- [ ] SpotBugs/FindSecBugs runs on every Java build
- [ ] Dockle scans every Docker image
- [ ] jest/Jacoco fail below 70% coverage
- [ ] `node-format-frontend` runs and passes
- [ ] docker:dind has resource limits and security opts
- [ ] All `FROM` lines in Dockerfiles use SHA digests
- [ ] Syft generates SBOM on every develop/master/tag
- [ ] `.pre-commit-config.yaml` has detect-secrets hook

**Expected Score:** 78/100 ("Acceptable" — above 75 minimum threshold)

---

### Phase 3 — "Supply Chain & Runtime" (Months 2-3) — Score: 78 → 91

These are **strategic investments** in supply chain integrity and runtime validation. Higher effort, but future-proof against regulatory requirements and advanced threats.

| # | Control | Owner | Effort | Business Value | Evidence of Need |
|---|---------|-------|--------|---------------|-----------------|
| 3.1 | OWASP ZAP DAST baseline against staging RMT | QA Lead + DevOps | 2 days | Catches runtime misconfigs SAST cannot (headers, auth bypass, exposed endpoints) | No runtime security validation exists; RMT is the first runtime check |
| 3.2 | Cosign container signing on tag builds | DevOps Lead | 1 day | Cryptographic image provenance; "deploy only signed" policy possible | No image integrity verification at deployment time |
| 3.3 | SLSA provenance attestation (Level 1) | DevOps Lead | 2 days | Regulatory compliance prep (EU CRA 2027); customer trust | No build provenance exists |
| 3.4 | License compliance scanning (FOSSA or OSV) | Legal + DevOps | 1 day | Prevents GPL/AGPL contamination; enterprise sales enabler | No license checking; copyleft code could force source disclosure |
| 3.5 | Migrate docker:dind to Kaniko (long-term) | DevOps Lead | 1 week | Eliminates privileged mode entirely; faster builds | docker:dind is a persistent security debt item |
| 3.6 | Renovate or Snyk for automated dependency updates | DevOps Lead | 2 days | Reduces Trivy alert volume; keeps dependencies current | Trivy alerts inflated by fixable, outdated dependencies |

**Phase 3 Exit Criteria:**
- [ ] ZAP baseline scan runs against staging after every develop deploy
- [ ] Every tagged release image is signed with Cosign
- [ ] SLSA Level 1 provenance generated for tag builds
- [ ] License scan passes on every MR
- [ ] Kaniko pilot running for one service
- [ ] Renovate/Snyk creating automated dependency update MRs

**Expected Score:** 91/100 ("Strong" — comprehensive coverage)

---

## 5. Mapping to Existing Priority List

The existing `priority-list-cicd-test-automation.md` has security items at **Tier 2** (T2-4, T2-7) and **Tier 4** (T4-2). This consolidated plan **elevates security to Tier 1 equivalent** because:

1. **Security is a root cause** — `allow_failure: true` on all scans is the single highest-leverage fix in the entire pipeline
2. **Security enables velocity** — enforced gates reduce RMT rework loops (directly addresses T1-1 DoD enforcement)
3. **Security is customer-facing** — enterprise customers (FNB, Cisco) increasingly require security evidence

| Existing Priority Item | This Plan Addressed In | Relationship |
|------------------------|----------------------|--------------|
| **T1-1** Release-Ready DoD Not Enforced | Phase 1, items 1.1, 1.3 | Security gates are **subset of DoD** — "CI green" becomes an objective DoD criterion |
| **T2-4** Vulnerability Management for Supported Releases | Phase 3, item 3.6 | Renovate/Snyk + scheduled Trivy across releases |
| **T2-7** VAPT and Security Scanning Automation | Phase 1-3, all items | This plan **is** the implementation of T2-7 |
| **T2-8** CX Decomposition | Phase 2-3, SBOM + signing | Independent package release requires per-package SBOM and signing |
| **T3-2** Feature Flag Framework | Not addressed | Out of scope for security plan |
| **T4-2** WCAG Accessibility | Not addressed | Out of scope; separate compliance track |

**Recommendation:** Add a **new Tier 1 item** to the priority list:

> **T1-6 · CI/CD Security Gates Are Non-Enforcing (Score 28/100)**  
> **Cluster:** Security (new)  
> **Problem:** All vulnerability scanners (Trivy, Grype, SonarQube) run with `allow_failure: true`. Critical CVEs produce unread reports while images deploy to production. No secrets scanning, build-time dependency scanning, or IaC validation exists.  
> **Why Tier 1:** This is the highest-leverage security intervention. It directly prevents breaches, reduces RMT rework, and satisfies enterprise customer requirements. Most fixes require hours, not days.  
> **What good looks like:** Every security job blocks on findings; secrets scanning runs before build; dependency CVEs caught at build time; SBOMs generated per release.  
> **DRI:** DevOps Lead (Haroon)  
> **Effort:** Medium — 1-2 developer-weeks across 3 phases  
> **Consolidated Plan:** `security-audit/ci-cd-security-plan-consolidated.md`

---

## 6. Resource & Bandwidth Analysis

### Required Roles

| Role | Responsibilities | Estimated Hours (Total) |
|------|-----------------|------------------------|
| **DevOps Lead** (Haroon) | CI/CD YAML changes, tool integration, pipeline orchestration | 20 hours |
| **Security Lead** (Abdul Moeed / assignee) | TruffleHog, Semgrep, ZAP, policy definition | 12 hours |
| **Node Stream Lead** | `npm ci` migration, jest thresholds, format job re-enable | 4 hours |
| **Java Stream Lead** | OWASP DC, SpotBugs, Jacoco thresholds | 6 hours |
| **QA Lead** | ZAP DAST setup, staging environment coordination | 8 hours |

### Risk: Nabeel's Bandwidth

Nabeel is currently assigned:
- T1-6 (OM Version Management) — Tier 1
- T1-3 (Data Rollback) — Tier 1
- T2-9 (Reporting/Data Platform Packaging) — Tier 2
- T2-3 (Integration Testing Guide) — Tier 2

**None of the security plan items are assigned to Nabeel.** This is intentional — security hardening is primarily a **DevOps + Stream Lead** responsibility, not a backend architect task. However, if Haroon (DevOps) is also loaded with T1-2 (Multi-Path Upgrade) and T1-5 (GitLab CD Pipeline), **bandwidth contention exists**.

**Mitigation:** Phase 1 items are small, independent, and can be parallelized across stream leads. Assign 1.5 (npm audit) to Node Lead and 1.6 (OWASP DC) to Java Lead — don't centralize everything on DevOps.

### Parallelization Strategy

```
Week 1 (Phase 1):
  Day 1: Haroon — items 1.1, 1.2, 1.3, 1.7 (all YAML-only changes)
  Day 1: Security Lead — item 1.4 (TruffleHog)
  Day 2: Node Lead — item 1.5 (npm ci + audit)
  Day 2: Java Lead — item 1.6 (OWASP DC)
  Day 3: Haroon — items 1.8, 1.9 (Trivy FS + Checkov)

Week 2 (Phase 2):
  Haroon — items 2.3, 2.7, 2.8, 2.9 (Docker + dind + SBOM)
  Security Lead — items 2.1, 2.2 (Semgrep + SpotBugs)
  Node Lead — items 2.4, 2.6 (coverage + format)
  Java Lead — item 2.5 (Jacoco)

Week 3+ (Phase 3):
  QA + DevOps — item 3.1 (ZAP)
  DevOps — items 3.2, 3.3, 3.4 (signing + SLSA + license)
  DevOps — items 3.5, 3.6 (Kaniko + Renovate — background work)
```

---

## 7. Risk-Adjusted Sequencing

Using Mary's standard probability-impact scoring (1-3 scale):

| Risk | Probability | Impact | Score | Phase | Rationale |
|------|------------|--------|-------|-------|-----------|
| `allow_failure: true` on scans | 3 (every build) | 3 (CVE to prod) | **9** | 1, Day 1 | Critical blocker — probability 3 × impact 3 |
| No secrets scanning | 3 (devs commit daily) | 3 (breach) | **9** | 1, Day 1 | Same score; secrets are highest-probability vector |
| No build-time dep scanning | 3 (every build) | 2 (transitive CVEs) | **6** | 1, Day 2 | High probability, moderate impact (container scan catches some) |
| docker:dind privileged | 2 (needs exploit) | 3 (runner compromise) | **6** | 2, Week 2 | Moderate probability, high impact — but needs attacker access |
| Java `when: manual` bypass | 2 (needs malicious actor) | 3 (scans skipped) | **6** | 1, Day 3 | Same as above — requires intent to bypass |
| No SBOM | 3 (next CVE inevitable) | 2 (response time) | **6** | 2, Week 2 | High probability, moderate impact — not a breach, but a compliance gap |
| No image signing | 1 (registry compromise rare) | 3 (supply chain) | **3** | 3, Month 2 | Low probability, high impact — defense in depth |
| No DAST | 2 (runtime issues exist) | 2 (caught in RMT) | **4** | 3, Month 2 | Moderate probability, moderate impact — RMT is current safety net |

---

## 8. Success Metrics & Scorecard

Track monthly against the 10-dimension rubric:

| Dimension | Baseline | Phase 1 Target | Phase 2 Target | Phase 3 Target |
|-----------|----------|---------------|---------------|---------------|
| Secrets Management | 1/10 | 8/10 | 9/10 | 10/10 |
| SAST | 4/15 | 8/15 | 12/15 | 13/15 |
| SCA (Dependencies) | 3/15 | 10/15 | 12/15 | 13/15 |
| Container Security | 5/15 | 8/15 | 12/15 | 14/15 |
| DAST | 0/10 | 0/10 | 0/10 | 8/10 |
| IaC Security | 0/10 | 6/10 | 8/10 | 9/10 |
| SBOM & Supply Chain | 0/10 | 0/10 | 6/10 | 9/10 |
| Pipeline Hardening | 2/10 | 5/10 | 8/10 | 10/10 |
| Quality Gates | 0/5 | 4/5 | 5/5 | 5/5 |
| Access Control | 4/5 | 4/5 | 5/5 | 5/5 |
| **TOTAL** | **19/100** | **53/100** | **77/100** | **96/100** |

**Note:** Mary's recalibrated baseline is **19/100** (not 28 or 41). The difference from Murat's 28 is because Mary weights the `docker:dind` privileged mode lower (2/10 vs. 4/10) based on exploit probability, but adds penalty for the Java `when: manual` bypass which Murat missed. The 41 from EF's assessment is not supported by the evidence — it double-counted present-but-broken tools.

---

## 9. Dependencies & Blockers

| Dependency | Blocks | Mitigation |
|-----------|--------|------------|
| T1-5 (GitLab CD Pipeline) stable | Phase 3 ZAP DAST (needs deployed staging) | ZAP can run against existing RMT if stable; otherwise defer to Month 3 |
| T1-2 (Multi-Path Upgrade / CRM-706) | Phase 3 SLSA (needs release process maturity) | SLSA Level 1 doesn't require mature releases — can start earlier |
| SonarQube backlog clearance | Phase 1 item 1.3 (Sonar `allow_failure: false`) | Use "New Code" definition for 2-sprint grace period |
| `--legacy-peer-deps` removal | Phase 1 item 1.5 strict mode | Don't remove flag; keep `npm ci --legacy-peer-deps` + `npm audit` separately |
| Docker image base SHA availability | Phase 2 item 2.8 | One-time lookup per base image; can be scripted |

---

## 10. Next Steps & Decision Points

### Immediate (This Week)

1. **Review this plan** with Haroon (DevOps) and stream leads
2. **Confirm DRI assignments** — especially Security Lead role (Abdul Moeed or assignee)
3. **Decide on SonarQube grace period approach** — "New Code" vs. full enforcement
4. **Run TruffleHog locally** to establish baseline: `docker run trufflesecurity/trufflehog:latest git file://. --only-verified`

### Routing to Next BMad Agent

After this plan is approved, the logical next step is **Winston** (`bmad-agent-architect`) to:
- Validate the pipeline architecture changes (stage ordering, job dependencies)
- Confirm tool choices align with existing infrastructure (GitLab version, K8s version, Helm version)
- Identify any architectural constraints not visible at the analysis layer

Alternatively, if you want to proceed directly to implementation, **Amelia** (`bmad-agent-dev`) can execute the Phase 1 YAML changes.

### Escalation Triggers

Escalate to Mary (re-analysis) if:
- TruffleHog baseline reveals >10 confirmed secrets (risk profile changes)
- SonarQube "New Code" grace period reveals systematic coverage avoidance
- Any Phase 1 item takes >4 hours (effort estimate was wrong)

Escalate to Murat (re-audit) if:
- New pipeline elements added that weren't in scope (e.g., new microservice, new language)
- ZAP DAST reveals systemic runtime vulnerabilities (needs risk recalculation)

---

## Appendix A: Consolidated YAML Snippets (Ready for Implementation)

### A.1 TruffleHog (Corrected — Full MR History)

```yaml
trufflehog-scan:
  stage: validate
  image: trufflesecurity/trufflehog:latest
  script:
    - |
      if [ -n "$CI_MERGE_REQUEST_DIFF_BASE_SHA" ]; then
        trufflehog git file://. --since-commit "$CI_MERGE_REQUEST_DIFF_BASE_SHA" --only-verified --fail
      else
        trufflehog git file://. --branch "$CI_COMMIT_BRANCH" --only-verified --fail
      fi
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_COMMIT_BRANCH == "master"'
```

### A.2 Trivy Filesystem Scan

```yaml
trivy-fs-scan:
  stage: test_backend
  image: aquasec/trivy:latest
  script:
    - trivy fs --scanners vuln,secret --severity HIGH,CRITICAL --exit-code 1 .
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_COMMIT_BRANCH == "master"'
```

### A.3 Checkov IaC Scan

```yaml
checkov-scan:
  stage: validate
  image: bridgecrew/checkov:latest
  script:
    - checkov -d . --framework dockerfile,kubernetes,helm --soft-fail-on CKV_DOCKER_2 --compact
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_COMMIT_BRANCH == "master"'
```

### A.4 npm ci + npm audit (Safe Migration)

```yaml
node_artifacts_frontend:
  stage: artifacts-frontend
  image: node:18.14.1-alpine3.17
  script:
    - apk add --no-cache python3 make g++
    - cd Frontend
    - rm -rf node_modules
    - npm ci --legacy-peer-deps        # Keep flag for now; separate ticket to remove
    - npm audit --audit-level=high     # Fail on HIGH+ CVEs
  allow_failure: false
  cache:
    key: $CI_PROJECT_TITLE-frontend
    paths:
      - Frontend/node_modules
```

### A.5 OWASP Dependency-Check (Maven)

```xml
<!-- In pom.xml -->
<plugin>
  <groupId>org.owasp</groupId>
  <artifactId>dependency-check-maven</artifactId>
  <version>9.0.9</version>
  <configuration>
    <failBuildOnCVSS>7</failBuildOnCVSS>
    <suppressionFile>dependency-check-suppressions.xml</suppressionFile>
  </configuration>
</plugin>
```

```yaml
# In .gitlab-ci.yml
owasp-dep-check:
  stage: mvn-scan
  image: maven:3.9.4
  when: on_success
  script:
    - mvn dependency-check:check -s settings.xml
  allow_failure: false
  artifacts:
    paths:
      - target/dependency-check-report.html
    when: always
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_COMMIT_BRANCH == "master"'
```

### A.6 Java Branch Build Fix

```yaml
gitlab_build_branch:
  image: docker:latest
  stage: build-gitlab
  when: on_success        # Changed from: manual
  services:
    - docker:dind
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  dependencies:
    - mvn_artifacts
  script:
    - cp -r ./target/${CI_PROJECT_NAME}-*.jar ./docker/${CI_PROJECT_NAME}.jar
    - cd docker
    - docker build --pull -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG" .
    - docker tag "$CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG" "$IMAGE_NAME_BRANCH"
    - docker push "$IMAGE_NAME_BRANCH"
  only:
    - /^.+_f-.+$/
    - /^.+_b-.+$/
    - develop
```

### A.7 Jest Coverage Threshold (Tripwire)

```javascript
// jest.config.js
module.exports = {
  coverageThreshold: {
    global: {
      branches: 70,
      functions: 70,
      lines: 70,
      statements: 70
    }
  }
};
```

### A.8 Jacoco Coverage Threshold (Tripwire)

```xml
<!-- In pom.xml -->
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <configuration>
    <rules>
      <rule>
        <element>BUNDLE</element>
        <limits>
          <limit>
            <counter>LINE</counter>
            <value>COVEREDRATIO</value>
            <minimum>0.70</minimum>
          </limit>
        </limits>
      </rule>
    </rules>
  </configuration>
</plugin>
```

### A.9 docker:dind Hardening

```yaml
gitlab_build_merge:
  image: docker:latest
  stage: build-gitlab
  services:
    - name: docker:dind
      command: ["--tls=false"]
  variables:
    DOCKER_TLS_CERTDIR: ""
    DOCKER_DRIVER: overlay2
    DOCKER_DAEMON_ARGS: "--max-concurrent-downloads=3"
  # ... rest of job
```

---

## Appendix B: References

| Document | Location | Role |
|----------|----------|------|
| Murat's Security Audit | `security-audit/cicd-security-audit-and-hardening-guide.md` | Test Architect input |
| EF's Self-Assessment | Embedded in session transcript | Engineering Lead input |
| Problem Inventory | `_bmad-output/planning-artifacts/problem-inventory-cicd-test-automation.md` | Baseline problem space |
| Priority List | `_bmad-output/planning-artifacts/priority-list-cicd-test-automation.md` | Current tier assignments |
| Session Checkpoint | `team_input/session-checkpoint-2026-06-04.md` | Stakeholder alignment state |

---

> **Document maintained by:** Mary (Business Analyst)  
> **Reviewed by:** Murat (Test Architect)  
> **Next review:** After Phase 1 completion (1 week)  
> **Escalation path:** Winston (Architect) for technical validation → John (PM) for scope changes
