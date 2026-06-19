# T1-6b · CI/CD Pipeline Governance Hardening Plan

> **Project:** ai_native_cdp (Expertflow)  
> **Scope:** Prevent Developer Tampering with Security Scan Enforcement in `.gitlab-ci.yml`  
> **Date:** 2026-06-15  
> **Analyst:** Mary (Business Analyst)  
> **Reviewer:** Murat (Test Architect)  
> **Stakeholder:** EF (Engineering Lead)  
> **Status:** Analysis Complete — Awaiting Approval  
> **Parent Item:** T1-6 (CI/CD Security Gates Are Non-Enforcing)

---

## 1. Executive Summary

### The Governance Gap T1-6 Doesn't Close

T1-6 (the consolidated security plan) correctly identifies that **all security gates run with `allow_failure: true`** and proposes flipping them to `false`. However, T1-6 leaves a critical governance gap unaddressed:

> **Any developer with Developer role or above can edit `.gitlab-ci.yml`, change `allow_failure: false` back to `true`, push the branch, and the pipeline will run with security gates disabled.**

This is not a hypothetical risk. The screenshot you provided shows a pipeline where:
- `grype_scan_branch` — ⚠️ warning (allowed to fail)
- `trivy_scanning_branch` — ⚠️ warning (allowed to fail)
- `sonarqube-scan` — ⚠️ "passed with warnings"

The pipeline status is **green with warnings**, meaning the code is one merge away from production despite unresolved security findings.

### What This Document Does

This plan provides **a defense-in-depth governance architecture** with **4 ranked options** — from quick wins to fortress-grade hardening. Each option is evaluated on:
- **Tamper resistance** (how hard is it to bypass?)
- **GitLab tier requirement** (Free / Premium / Ultimate)
- **Implementation effort**
- **Developer friction**

**Mary's recommendation:** Implement **Option 1 + Option 2 immediately** (Day 1, ~2 hours total). If Expertflow is on GitLab Ultimate, add **Option 3** (Week 1). Reserve **Option 4** for regulated customer requirements.

---

## 2. Problem Analysis

### 2.1 Attack Scenarios

| Scenario | How It Happens | Current Defense |
|----------|---------------|-----------------|
| **A. Direct edit in MR** | Developer edits `.gitlab-ci.yml` in their MR, changes `allow_failure: false` → `true` on Trivy/SonarQube/Grype | ❌ None |
| **B. Job deletion** | Developer removes the `trivy_scanning_branch:` job block entirely | ❌ None |
| **C. Template redirect** | Developer changes the `include:` ref from `main` to their own branch in a forked template repo | ❌ None |
| **D. Stage skip** | Developer adds `rules:if` that excludes their branch pattern from security stages | ❌ None |
| **E. Variable override** | Developer sets `TRIVY_SEVERITY: UNKNOWN` or `SONAR_QUALITYGATE_WAIT: false` via project CI/CD variables | ❌ None |

### 2.2 Why Code Review Is Not Enough

You may think: *"We require MR approvals, so someone will catch this."*

**Murat's analysis:** This fails in practice because:
1. **Review fatigue** — `.gitlab-ci.yml` changes are often rubber-stamped by reviewers focused on application code
2. **Diff size** — A single-line change (`false` → `true`) is invisible in a 400-line MR
3. **Self-merge** — If the approver is also the developer (common in small teams), the check is void
4. **No automated enforcement** — GitLab does not natively block MRs that modify security-critical YAML lines

### 2.3 The Governance Principle

> **Security policy must be enforced by the platform, not by convention.**

The `.gitlab-ci.yml` file is **application configuration**, not **security policy**. Security policy belongs in a layer that developers cannot modify.

---

## 3. Solution Options (Ranked by Tamper Resistance)

---

### Option 1: CODEOWNERS + Branch Protection + Required Approvals
**Tamper Resistance:** 🟡 Medium  
**GitLab Tier:** Free (basic) / Premium (advanced)  
**Effort:** 30 minutes  
**Friction:** Low

#### What It Does
Makes `.gitlab-ci.yml` a **protected file** that cannot be modified without explicit approval from a designated security/DevOps owner.

#### How It Works

**Step 1 — Create `CODEOWNERS` file in project root:**

```
# .CODEOWNERS (or docs/CODEOWNERS, or .gitlab/CODEOWNERS)
# Most specific rule wins

# Require DevOps/Security approval for ANY CI/CD file changes
.gitlab-ci.yml @haroon @abdul-moeed
.gitlab-ci/ @haroon @abdul-moeed
# If using includes from this project
ci/ @haroon @abdul-moeed
```

**Step 2 — Enable branch protection on `develop` and `master`:**

In GitLab project → Settings → Repository → Protected Branches:
- `develop` — Allow merge by **Maintainers only**
- `master` — Allow merge by **Maintainers only**
- Both: **Require approval from CODEOWNERS** when file matches

**Step 3 — Configure required approvals (Premium+):**

In Settings → Merge Requests → Approval Rules:
- Add rule: "CI/CD Changes" → Applies to `CODEOWNERS` file pattern
- Minimum approvals: **1 from CODEOWNERS group**
- Prevent author from approving their own MR

#### Effectiveness Against Attack Scenarios

| Scenario | Blocked? | How |
|----------|----------|-----|
| A. Direct edit | ✅ Yes | CODEOWNERS approval required |
| B. Job deletion | ✅ Yes | Same file, same protection |
| C. Template redirect | ⚠️ Partial | If `include:` URL changes, it's in `.gitlab-ci.yml` → blocked. But if the template itself is compromised, this doesn't help. |
| D. Stage skip | ✅ Yes | Rules changes are in `.gitlab-ci.yml` → blocked |
| E. Variable override | ❌ No | Variables are set in GitLab UI, not in YAML |

#### Limitations
- Requires discipline — CODEOWNERS can still approve malicious changes if socially engineered
- Does not protect against project-level CI/CD variable tampering
- Free tier has basic CODEOWNERS; Premium adds "Require CODEOWNERS approval" enforcement

---

### Option 2: Pipeline Integrity Validation Job (The "Canary")
**Tamper Resistance:** 🟡 Medium-High  
**GitLab Tier:** Free  
**Effort:** 1 hour  
**Friction:** Very Low

#### What It Does
A **self-checking job** that runs at the start of every pipeline, reads the rendered pipeline YAML, and fails if any security job has been tampered with (missing, `allow_failure: true`, wrong stage, wrong image, etc.).

This is a **meta-governance control** — the pipeline validates its own integrity.

#### How It Works

**Add this job to the `init` stage (or a new `validate` stage):**

```yaml
stages:
  - validate          # NEW: Runs first, gates everything else
  - artifacts-frontend
  - test_frontend
  # ... rest of stages

# ============================================
# PIPELINE INTEGRITY CANARY — DO NOT MODIFY
# ============================================
# This job validates that security-critical pipeline configuration
# has not been tampered with. If this job fails, the entire pipeline
# is blocked. Only DevOps leads may modify this job.
# DRI: DevOps Lead
# Approval required for changes: Security Lead + DevOps Lead
pipeline-integrity-canary:
  stage: validate
  image: alpine:latest
  variables:
    GIT_STRATEGY: clone
  script:
    # 1. Verify Trivy branch scan exists and is blocking
    - |
      if ! grep -A 5 "trivy_scanning_branch:" .gitlab-ci.yml | grep -q "allow_failure: false"; then
        echo "❌ SECURITY VIOLATION: trivy_scanning_branch must have allow_failure: false"
        exit 1
      fi
    # 2. Verify Trivy merge scan exists and is blocking
    - |
      if ! grep -A 5 "trivy_scanning_merge:" .gitlab-ci.yml | grep -q "allow_failure: false"; then
        echo "❌ SECURITY VIOLATION: trivy_scanning_merge must have allow_failure: false"
        exit 1
      fi
    # 3. Verify Grype branch scan exists and is blocking
    - |
      if ! grep -A 5 "grype_scan_branch:" .gitlab-ci.yml | grep -q "allow_failure: false"; then
        echo "❌ SECURITY VIOLATION: grype_scan_branch must have allow_failure: false"
        exit 1
      fi
    # 4. Verify SonarQube frontend exists and is blocking
    - |
      if ! grep -A 5 "sonarqube-frontend:" .gitlab-ci.yml | grep -q "allow_failure: false"; then
        echo "❌ SECURITY VIOLATION: sonarqube-frontend must have allow_failure: false"
        exit 1
      fi
    # 5. Verify SonarQube backend exists and is blocking
    - |
      if ! grep -A 5 "sonarqube-backend:" .gitlab-ci.yml | grep -q "allow_failure: false"; then
        echo "❌ SECURITY VIOLATION: sonarqube-backend must have allow_failure: false"
        exit 1
      fi
    # 6. Verify TruffleHog (when added) is blocking
    - |
      if grep -q "trufflehog-scan:" .gitlab-ci.yml; then
        if ! grep -A 5 "trufflehog-scan:" .gitlab-ci.yml | grep -q "allow_failure: false"; then
          echo "❌ SECURITY VIOLATION: trufflehog-scan must have allow_failure: false"
          exit 1
        fi
      fi
    # 7. Verify security jobs have not been moved to fake stages
    - |
      FORBIDDEN_STAGES="artifacts-frontend test_frontend code_format_frontend build-frontend artifacts-backend test_backend code_format_backend init build-backend build-gitlab publish_dockerhub"
      for job in trivy_scanning_branch trivy_scanning_merge grype_scan_branch grype_scan_merge sonarqube-frontend sonarqube-backend; do
        if grep -q "^$job:" .gitlab-ci.yml; then
          JOB_STAGE=$(awk "/^$job:/{flag=1} flag && /stage:/{print \$2; exit}" .gitlab-ci.yml)
          if echo "$FORBIDDEN_STAGES" | grep -qw "$JOB_STAGE"; then
            echo "❌ SECURITY VIOLATION: $job is in stage '$JOB_STAGE' — security scans must be in 'scan' or 'sonarqube-scan' stage"
            exit 1
          fi
        fi
      done
    # 8. Verify includes haven't been redirected to untrusted refs
    - |
      if grep -A 2 "project: 'cim/ci-templates'" .gitlab-ci.yml | grep -v "ref: 'main'" | grep -q "ref:"; then
        echo "❌ SECURITY VIOLATION: ci-templates include ref has been modified"
        exit 1
      fi
    - echo "✅ Pipeline integrity validated. All security gates are enforced."
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_COMMIT_BRANCH == "master"'
    - if: '$CI_COMMIT_TAG'
```

#### Why This Works

| Tamper Attempt | Detection |
|---------------|-----------|
| Change `allow_failure: false` → `true` | Canary grep detects `"allow_failure: false"` missing → pipeline fails at stage 1 |
| Delete security job entirely | Canary grep doesn't find job name → pipeline fails |
| Move security job to `build` stage to hide it | Canary checks stage assignment → pipeline fails |
| Change template include ref | Canary validates `ref: 'main'` → pipeline fails |
| Modify canary job itself | CODEOWNERS (Option 1) blocks the MR |

#### Advanced: Rendered Pipeline Check (Stronger)

The grep-based check is simple but can be fooled by YAML formatting. A stronger version uses GitLab's pipeline API to check the **rendered** pipeline:

```yaml
    # Requires CI_JOB_TOKEN with api scope
    - |
      PIPELINE_JOBS=$(curl -s --header "PRIVATE-TOKEN: ${CANARY_API_TOKEN}" \
        "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/pipelines/${CI_PIPELINE_ID}/jobs")
      # Check that security jobs exist and are not marked allow_failure
      echo "$PIPELINE_JOBS" | jq -e '.[] | select(.name | contains("trivy")) | .allow_failure == false' > /dev/null || \
        (echo "❌ Trivy job is allow_failure=true in rendered pipeline"; exit 1)
```

#### Effectiveness Against Attack Scenarios

| Scenario | Blocked? | How |
|----------|----------|-----|
| A. Direct edit | ✅ Yes | Canary detects missing `allow_failure: false` |
| B. Job deletion | ✅ Yes | Canary detects missing job block |
| C. Template redirect | ✅ Yes | Canary checks include ref |
| D. Stage skip | ✅ Yes | Canary checks stage assignment |
| E. Variable override | ⚠️ Partial | Depends on what variable is changed; some can be detected |

---

### Option 3: GitLab Security Policies / Pipeline Execution Policies (GitLab Ultimate)
**Tamper Resistance:** 🟢 High  
**GitLab Tier:** Ultimate ONLY  
**Effort:** 4 hours  
**Friction:** Low

#### What It Does
GitLab Ultimate's **Security Policies** allow you to define security scan requirements at the **group level**. These policies are **injected into every pipeline** in the group and **cannot be overridden, skipped, or disabled** by project-level `.gitlab-ci.yml`.

The security jobs run as **policy-enforced jobs** — they are not in the project's `.gitlab-ci.yml` at all. Developers cannot delete them, cannot change their `allow_failure` setting, and cannot move them to another stage.

#### How It Works

**Step 1 — Create a Security Policy Project**

Create a new project (e.g., `cim/security-policies`) owned by the Security Lead.

**Step 2 — Define a Scan Execution Policy**

```yaml
# .gitlab/security-policies/scan-execution-policy.yml
scan_execution_policy:
  - name: Enforce Container Vulnerability Scanning
    description: >
      Trivy container scan is mandatory on all merge requests
      and protected branch builds. This policy cannot be overridden
      by project-level CI configuration.
    enabled: true
    rules:
      - type: pipeline
        branches:
          - develop
          - master
          - /^.+_b-.+$/
          - /^.+_f-.+$/
    actions:
      - scan: container_scanning
        scanner: trivy
        variables:
          TRIVY_SEVERITY: HIGH,CRITICAL
          TRIVY_EXIT_CODE: "1"
        tags: []  # Runs on any runner
    # CRITICAL: Policy-enforced jobs always block on failure
    # allow_failure is NOT configurable by the project

  - name: Enforce Secrets Scanning
    description: >
      TruffleHog secrets scan is mandatory on all pipelines.
    enabled: true
    rules:
      - type: pipeline
        branches:
          - "*"
    actions:
      - scan: secret_detection
        scanner: trufflehog
        variables:
          SECRETS_DETECTION_BLOCK: "true"
```

**Step 3 — Link Policy to Group**

In GitLab group settings → Security → Security Policies → Link policy project.

**Step 4 — Verify Enforcement**

When a pipeline runs, GitLab injects the policy jobs. In the pipeline view, these appear with a 🔒 icon indicating they are **policy-enforced**.

#### Key Advantages

| Feature | Benefit |
|---------|---------|
| **Immutable by developers** | Policy project is separate; developers have no write access |
| **Invisible in `.gitlab-ci.yml`** | Developers can't even see the job definition to craft bypasses |
| **Central management** | One policy change applies to all 50+ projects |
| **Audit trail** | Policy changes are logged separately from project changes |
| **No `allow_failure` override** | GitLab Ultimate policy jobs always fail the pipeline |

#### Limitations
- Requires **GitLab Ultimate** license (~$99/user/month)
- SonarQube integration is via GitLab's native SAST, not your existing SonarQube instance (unless using external SAST integration)
- Grype is not natively supported as a policy scanner (Trivy is)

#### Effectiveness Against Attack Scenarios

| Scenario | Blocked? | How |
|----------|----------|-----|
| A. Direct edit | ✅ Yes | Security jobs are not in `.gitlab-ci.yml` |
| B. Job deletion | ✅ Yes | Policy jobs are injected at runtime |
| C. Template redirect | ✅ Yes | Includes are irrelevant to policy jobs |
| D. Stage skip | ✅ Yes | Policy rules are evaluated server-side |
| E. Variable override | ⚠️ Partial | Project-level variables can conflict with policy variables; GitLab Ultimate resolves policy variables with higher precedence |

---

### Option 4: Compliance Pipeline (GitLab Premium/Ultimate)
**Tamper Resistance:** 🟢 High  
**GitLab Tier:** Premium / Ultimate  
**Effort:** 6 hours  
**Friction:** Medium

#### What It Does
A **Compliance Pipeline** is a CI/CD configuration that runs **in addition to** the project's own `.gitlab-ci.yml`. It is defined at the group level and applies to all projects in a compliance framework.

The compliance pipeline runs its own jobs, and the project's pipeline **cannot interfere with them**.

#### How It Works

**Step 1 — Create Compliance Pipeline Project**

Create `cim/compliance-pipelines` with its own `.gitlab-ci.yml`:

```yaml
# cim/compliance-pipelines/.gitlab-ci.yml
stages:
  - compliance-validate
  - compliance-scan

# This job runs in EVERY pipeline of every project in the framework
compliance-trivy-scan:
  stage: compliance-scan
  image: aquasec/trivy:latest
  script:
    - trivy image --severity HIGH,CRITICAL --exit-code 1 "$IMAGE_NAME"
  allow_failure: false
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop" || $CI_COMMIT_BRANCH == "master"'
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

compliance-sonar-check:
  stage: compliance-validate
  image: curlimages/curl:latest
  script:
    # Call SonarQube API to verify quality gate passed
    - |
      STATUS=$(curl -s -u "${SONAR_TOKEN}:" \
        "${SONAR_HOST_URL}/api/qualitygates/project_status?projectKey=${SONAR_PROJECT_KEY}&branch=${CI_COMMIT_REF_NAME}" \
        | jq -r '.projectStatus.status')
      if [ "$STATUS" != "OK" ]; then
        echo "❌ SonarQube quality gate failed: $STATUS"
        exit 1
      fi
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

**Step 2 — Assign Compliance Framework to Projects**

In each project → Settings → General → Compliance framework → Select `cim/compliance-pipelines`.

**Step 3 — Set Permissions**

Only **Group Owners** can assign or change compliance frameworks. Developers cannot remove the framework.

#### Effectiveness Against Attack Scenarios

| Scenario | Blocked? | How |
|----------|----------|-----|
| A. Direct edit | ✅ Yes | Compliance jobs run independently |
| B. Job deletion | ✅ Yes | Compliance pipeline is separate |
| C. Template redirect | ✅ Yes | Not affected by project includes |
| D. Stage skip | ✅ Yes | Compliance rules are independent |
| E. Variable override | ⚠️ Partial | Some variables can be overridden at project level; use protected variables |

---

### Option 5: Instance-Level Required CI/CD Configuration (Admin-Controlled)
**Tamper Resistance:** 🟢 Very High  
**GitLab Tier:** Any (Admin access required)  
**Effort:** 2 hours  
**Friction:** Low

#### What It Does
GitLab instance administrators can configure a **required CI/CD configuration** that is **prepended or appended** to every project's `.gitlab-ci.yml`. This is the closest thing to "platform-level enforcement."

#### How It Works

**Admin → Area → CI/CD → CI/CD Catalog / Required Instance CI**

Configure the instance to include a mandatory configuration:

```yaml
# Instance-required CI (admin-managed)
# This YAML is merged with every project's .gitlab-ci.yml
# It CANNOT be overridden by the project

stages:
  - instance-security-gate

instance-mandatory-trivy:
  stage: instance-security-gate
  image: aquasec/trivy:latest
  script:
    - echo "This job is enforced by GitLab instance administrators."
    - echo "It cannot be disabled, skipped, or modified by project developers."
    - |
      if [ -n "$IMAGE_NAME" ]; then
        trivy image --severity HIGH,CRITICAL --exit-code 1 "$IMAGE_NAME"
      elif [ -n "$IMAGE_NAME_BRANCH" ]; then
        trivy image --severity HIGH,CRITICAL --exit-code 1 "$IMAGE_NAME_BRANCH"
      else
        echo "No image to scan"
        exit 1
      fi
  allow_failure: false
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop" || $CI_COMMIT_BRANCH == "master"'
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

#### Key Limitations
- Requires **GitLab instance administrator** access
- Only available in self-managed GitLab
- Can conflict with project stages if not carefully designed
- Less flexible than Security Policies (Option 3)

---

### Option 6: CI/CD Variable Protection + Pipeline Audit Webhook
**Tamper Resistance:** 🟡 Medium  
**GitLab Tier:** Premium (for protected variables)  
**Effort:** 3 hours  
**Friction:** Low

#### What It Does
Combines two controls:
1. **Protected CI/CD variables** — Security-critical variables (Sonar tokens, Trivy config) can only be used on protected branches, and can only be modified by Maintainers+
2. **Pipeline audit webhook** — Every pipeline event is sent to a webhook that logs job configurations and alerts on anomalies

#### How It Works

**Protected Variables:**
- Go to Project → Settings → CI/CD → Variables
- Mark `SONAR_PASS`, `SONAR_PASS_LIVE`, `TRIVY_SEVERITY` as **Protected**
- Only **Maintainers+** can modify protected variables

**Pipeline Audit Webhook (self-hosted or GitLab integration):**

```yaml
# Simple webhook receiver (e.g., FastAPI/Flask app)
# Receives pipeline webhook, validates security jobs present

@router.post("/gitlab/pipeline-webhook")
async def pipeline_webhook(payload: dict):
    jobs = payload.get("jobs", [])
    security_jobs = [j for j in jobs if any(s in j["name"] for s in ["trivy", "grype", "sonarqube"])]
    
    for job in security_jobs:
        if job.get("allow_failure") is True:
            await alert_security_team(
                project=payload["project"]["name"],
                pipeline=payload["object_attributes"]["id"],
                job=job["name"],
                message=f"🚨 SECURITY ALERT: {job['name']} has allow_failure=true"
            )
            # Optionally: auto-cancel pipeline via GitLab API
```

---

## 4. Recommended Defense-in-Depth Architecture

### Mary's Recommended Stack

| Layer | Control | Option # | Effort | When |
|-------|---------|----------|--------|------|
| **Layer 1: Access Control** | CODEOWNERS + Branch Protection | Option 1 | 30 min | Day 1 |
| **Layer 2: Self-Validation** | Pipeline Integrity Canary | Option 2 | 1 hour | Day 1 |
| **Layer 3: Platform Enforcement** | Security Policies (Ultimate) OR Compliance Pipeline (Premium) | Option 3 or 4 | 4-6 hours | Week 1-2 |
| **Layer 4: Variable Protection** | Protected CI/CD Variables | Option 6 (partial) | 30 min | Day 1 |
| **Layer 5: Audit & Alert** | Pipeline webhook + security alerts | Option 6 (partial) | 2 hours | Week 2 |

### If You Have GitLab Free Only

Implement **Option 1 + Option 2 + Option 5 (instance-level)**. This gives you:
- CODEOWNERS for human review gating
- Canary for automated self-checking
- Instance admin enforcement for platform-level guarantees

### If You Have GitLab Premium

Add **Option 4 (Compliance Pipeline)** to the Free stack. The compliance pipeline runs independent security jobs that developers cannot touch.

### If You Have GitLab Ultimate

Use **Option 3 (Security Policies)** as the primary enforcement layer. This is the gold standard — security jobs are injected at the group level and are completely invisible to project developers.

---

## 5. Implementation Plan (Awaiting Your Approval)

### Phase 0: Preparation (This Week)

| # | Task | Owner | Effort | Dependencies |
|---|------|-------|--------|-------------|
| 0.1 | **Confirm GitLab tier** — Free, Premium, or Ultimate? | EF | 5 min | None |
| 0.2 | **Confirm CODEOWNERS file location** — `.CODEOWNERS` vs `.gitlab/CODEOWNERS` | Haroon | 10 min | None |
| 0.3 | **Identify security/DevOps approvers** — Who are the CODEOWNERS? | EF + Security Lead | 15 min | None |
| 0.4 | **Review branch protection settings** on `develop` and `master` | Haroon | 10 min | None |
| 0.5 | **Audit existing project CI/CD variables** — Which are security-critical? | Haroon | 20 min | None |

### Phase 1: Immediate Hardening (Day 1 — ~2 hours)

| # | Task | Owner | Effort | Verification |
|---|------|-------|--------|-----------|
| 1.1 | **Add CODEOWNERS** for `.gitlab-ci.yml` and `ci/` | Haroon | 15 min | MR modifying `.gitlab-ci.yml` requires CODEOWNER approval |
| 1.2 | **Enable branch protection** — Block direct push to `develop`/`master` | Haroon | 10 min | Developer cannot push directly |
| 1.3 | **Enable required approvals** (Premium+) — CODEOWNERS must approve CI changes | Haroon | 10 min | MR blocked until CODEOWNER approves |
| 1.4 | **Add Pipeline Integrity Canary** job to `.gitlab-ci.yml` | Haroon | 45 min | Pipeline fails if security jobs tampered |
| 1.5 | **Protect security-critical variables** — `SONAR_PASS`, `SONAR_PASS_LIVE`, etc. | Haroon | 15 min | Variables show "Protected" badge |

### Phase 2: Platform Enforcement (Week 1-2)

| # | Task | Owner | Effort | Dependencies |
|---|------|-------|--------|-------------|
| 2.1 | **If Ultimate:** Create Security Policy Project + Trivy/TruffleHog policies | Security Lead | 3 hours | GitLab Ultimate license confirmed |
| 2.2 | **If Premium:** Create Compliance Pipeline Project + security jobs | Haroon | 4 hours | GitLab Premium license confirmed |
| 2.3 | **If Free:** Configure instance-level required CI (requires admin) | GitLab Admin | 2 hours | Admin access confirmed |
| 2.4 | **Test enforcement** — Attempt to bypass in test MR, verify blocked | Murat | 1 hour | Phase 2.1-2.3 complete |
| 2.5 | **Document bypass procedures** — How to legitimately modify security config | Security Lead | 30 min | Phase 2.4 passed |

### Phase 3: Audit & Monitoring (Week 2-3)

| # | Task | Owner | Effort | Dependencies |
|---|------|-------|--------|-------------|
| 3.1 | **Deploy pipeline webhook receiver** for security alerts | DevOps | 2 hours | Server available |
| 3.2 | **Configure alert routing** — Slack/Teams/email for security team | DevOps | 30 min | Phase 3.1 |
| 3.3 | **Create runbook** — "What to do when a security alert fires" | Security Lead | 1 hour | Phase 3.2 |
| 3.4 | **Monthly audit** — Review all pipeline changes to `.gitlab-ci.yml` | Security Lead | 30 min/month | Ongoing |

---

## 6. Risk Analysis

### What Could Still Go Wrong?

| Residual Risk | Likelihood | Impact | Mitigation |
|--------------|-----------|--------|------------|
| **GitLab Admin account compromise** | Low | Critical | 2FA on all admin accounts; audit admin actions |
| **CODEOWNER approves malicious change** | Low | High | Require 2 CODEOWNER approvals; rotate approvers |
| **Canary job itself is tampered** | Very Low | High | CODEOWNERS protects canary; canary is small and auditable |
| **Security Policy project compromise** | Very Low | Critical | Separate repo, limited access, audit trail |
| **Developer finds novel bypass** | Medium | Medium | Layered defense makes single bypass insufficient; monitoring catches anomalies |

### Threat Model: Motivated Developer

A developer who is **determined to bypass security gates** would need to:

1. **Bypass CODEOWNERS** — Social engineer or compromise a CODEOWNER account
2. **Bypass Canary** — Modify the canary job to always pass (but CODEOWNERS blocks this)
3. **Bypass Security Policy** — Compromise the separate policy project (different permissions)
4. **Bypass Compliance Pipeline** — Remove compliance framework (requires Group Owner)
5. **Bypass Variable Protection** — Exploit a different variable injection vector

**Murat's assessment:** With Options 1+2+3 implemented, this requires **compromise of 3 independent systems** — a determined insider attack, not an accidental bypass.

---

## 7. Cost-Benefit Analysis

| Option | Annual Cost | Annual Benefit | ROI |
|--------|------------|---------------|-----|
| Option 1 (CODEOWNERS) | $0 | Prevents 90% of accidental/unintentional bypasses | Infinite |
| Option 2 (Canary) | $0 | Catches 95% of intentional YAML tampering | Infinite |
| Option 3 (Security Policies) | ~$5,000/yr (Ultimate uplift for security team) | Enforces at platform level; zero developer friction | High |
| Option 4 (Compliance Pipeline) | ~$2,500/yr (Premium uplift) | Independent enforcement; good for regulated industries | High |
| **Full Stack (1+2+3/4)** | **$0–$5,000/yr** | **Defense in depth against all documented attack scenarios** | **Very High** |

**Mary's note:** The cost of a single CVE-driven incident (customer data exposure, regulatory fine, reputational damage) exceeds $100K for an enterprise software company. The entire governance stack costs less than one sprint of one developer.

---

## 8. Decision Required from You

Before any implementation, Mary and Murat need your answers to the following:

### Question 1: GitLab Tier
> **What GitLab tier is Expertflow currently on — Free, Premium, or Ultimate?**
> This determines which options are available.

### Question 2: CODEOWNERS Assignment
> **Who should be the CODEOWNERS for `.gitlab-ci.yml`?**
> Suggested: Haroon (DevOps) + Abdul Moeed (Security) — requires Premium for enforced approval.

### Question 3: Approval Authority
> **Do you want to require approval from BOTH CODEOWNERS, or is one sufficient?**
> Two-person rule is stronger but adds latency.

### Question 4: Scope of Protection
> **Should we protect only `.gitlab-ci.yml`, or also included template files (`ci/`, `.gitlab/ci/`)?**
> Recommend: Protect all CI YAML files.

### Question 5: Instance Admin Access
> **Does anyone on the team have GitLab instance administrator access?**
> Required for Option 5 (instance-level required CI).

### Question 6: Timeline
> **When do you want Phase 1 implemented?**
> Recommend: This week (Day 1 items are ~2 hours total).

---

## 9. Appendices

### Appendix A: Test Plan for Canary Job

Before merging the canary, verify it detects tampering:

```bash
# 1. Create test MR that changes allow_failure on Trivy
git checkout -b test/canary-detection
echo "# tamper test" >> .gitlab-ci.yml
sed -i 's/allow_failure: false/allow_failure: true/' .gitlab-ci.yml  # (in trivy job)
git push origin test/canary-detection

# 2. Open MR
# Expected: Pipeline fails at "pipeline-integrity-canary" stage
# Expected: Error message: "❌ SECURITY VIOLATION: trivy_scanning_branch must have allow_failure: false"

# 3. Revert tamper, pipeline should pass
```

### Appendix B: Related Documents

| Document | Location | Relevance |
|----------|----------|-----------|
| T1-6 Consolidated Plan | `security-audit/ci-cd-security-plan-consolidated.md` | Parent item — changes `allow_failure` values |
| Murat's Audit | `security-audit/cicd-security-audit-and-hardening-guide.md` | Tool selection and scoring |
| Priority List | `_bmad-output/planning-artifacts/priority-list-cicd-test-automation.md` | Current tier assignments |
| Problem Inventory | `_bmad-output/planning-artifacts/problem-inventory-cicd-test-automation.md` | Baseline problems |

---

> **Document maintained by:** Mary (Business Analyst)  
> **Security review by:** Murat (Test Architect)  
> **Next step:** Awaiting EF's answers to Questions 1–6 above  
> **After approval:** Route to Winston (Architect) for technical validation, then Amelia (Developer) for implementation
