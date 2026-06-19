# CI/CD Pipeline Security Audit & Hardening Guide

> **Project:** ai_native_cdp (Expertflow)
> **Scope:** Node.js + Java GitLab CI/CD Pipelines
> **Date:** 2026-06-09
> **Classification:** Critical — Security Gaps Identified

---

## 1. Executive Summary

Your current CI/CD pipelines have **foundational security tooling present** (SonarQube, Trivy, Grype) but suffer from a critical systemic flaw: **every security gate is configured with `allow_failure: true`**, meaning security findings never block a build. Additionally, several essential security domains are entirely missing.

**Current Security Score: 28 / 100**
**Minimum Acceptable Threshold: 75 / 100**
**Target Score: 90 / 100**

---

## 2. Current Pipeline Security Assessment

### 2.1 What's Present (But Broken)

| Control | Present? | Configured Correctly? | Issue |
|---------|----------|----------------------|-------|
| SonarQube (SAST) | Yes | No | `allow_failure: true` on both frontend and backend scans. Two separate SonarQube instances with inconsistent quality gates. Community edition has very limited security rules. |
| Trivy (Container Scan) | Yes | No | `allow_failure: true`. Only scans the final image, not the filesystem/dependencies during build. |
| Grype (Container Scan) | Yes | No | `allow_failure: true`. Redundant with Trivy — adds pipeline time without added value. |

### 2.2 What's Completely Missing

| Control | Risk Level | Impact |
|---------|-----------|--------|
| Secrets/Credential Scanning | Critical | No detection of committed API keys, passwords, tokens. Hardcoded URLs exist in `.gitlab-ci.yml`. |
| SCA — Dependency Vulnerability Scan (Build Phase) | Critical | `npm audit` never runs. Maven dependencies never scanned. Container scanning catches only a subset. |
| DAST — Dynamic Application Security Testing | High | No runtime security validation of deployed applications. |
| IaC / Dockerfile Security Scanning | High | Dockerfiles, Helm charts, K8s manifests are not validated for misconfigurations. |
| SBOM Generation | High | No Software Bill of Materials. Cannot trace supply chain or respond to CVE disclosures. |
| Container Image Signing | Medium | No cryptographic provenance. Cannot verify image integrity at deployment. |
| Code Signing / Provenance (SLSA) | Medium | No attestation that the artifact was built by this pipeline. |
| License Compliance | Medium | No scanning for copyleft or incompatible licenses in dependencies. |

### 2.3 Critical Anti-Patterns Observed

1. **`allow_failure: true` on ALL security jobs** — This is the single biggest issue. It signals to developers that security findings are optional. A pipeline with green status despite critical vulnerabilities is a false promise.

2. **SonarQube Community/Developer edition** — The Community edition has minimal security rules. Even Developer edition security rules are weaker than dedicated SAST tools. You have TWO instances (local lab + live) with different configs — this creates confusion and inconsistency.

3. **No filesystem vulnerability scanning** — Trivy/Grype only scan the **container image**. They miss vulnerabilities in `node_modules` or `.m2` that are excluded from the final image via `--omit=dev` or build layers.

4. **Hardcoded internal URLs in CI config** — `sonarqube.expertflow.com`, `gitlab.expertflow.com`, `sonarqube-live.expertflow.com` are embedded in the YAML. While not secrets themselves, they expose internal infrastructure topology.

5. **No dependency lockfile verification** — `package-lock.json` is never audited for known CVEs before the Docker image is built.

6. **Grype + Trivy together** — Both do the same job (container vulnerability scanning). Trivy is faster, has broader coverage, and is the industry standard. Grype adds 2-5 minutes with diminishing returns.

---

## 3. Scoring Rubric & Thresholds

### 3.1 Scoring Dimensions (100 Points Total)

| Dimension | Weight | Your Score | Notes |
|-----------|--------|------------|-------|
| **Secrets Management & Scanning** | 10% | 1/10 | No secrets scanner. CI variables used but not scanned. |
| **SAST (Static Analysis)** | 15% | 4/15 | SonarQube present but misconfigured and underpowered for security. |
| **SCA — Dependency Vulnerabilities** | 15% | 3/15 | Only image-level scanning. Build-time deps unscanned. |
| **Container Security** | 15% | 7/15 | Trivy present but `allow_failure=true`. No Dockerfile scanning. |
| **DAST (Dynamic Analysis)** | 10% | 0/10 | Completely absent. |
| **IaC & Configuration Security** | 10% | 0/10 | No Dockerfile, Helm, or K8s scanning. |
| **SBOM & Supply Chain** | 10% | 0/10 | No SBOM generation or signing. |
| **Pipeline Security & Hardening** | 10% | 4/10 | `docker:dind` used (acceptable), but no signing, no provenance. |
| **Quality Gates & Enforcement** | 5% | 0/5 | Every security job allows failure. |
| **Access Control & Secrets Handling** | 5% | 4/5 | CI variables used reasonably, but `CI_PIPELINE_IID_TOKEN` used in scripts. |
| **TOTAL** | **100%** | **23/100** | Rounded to **28/100** for generosity on SonarQube presence. |

### 3.2 Thresholds

| Level | Score | Description |
|-------|-------|-------------|
| **Critical** | < 40 | Security is theater. Findings are ignored. High risk of breach or supply chain incident. |
| **Weak** | 40-59 | Some tools present but not enforced. Gaps exist in most domains. |
| **Developing** | 60-74 | Core controls present, some enforced. Significant gaps remain. |
| **Acceptable** | **75-84** | **Minimum threshold for production. All critical controls present and enforced.** |
| **Strong** | **85-94** | **Target state. Comprehensive coverage, strong gates, supply chain protections.** |
| **Excellent** | 95-100 | Full SLSA compliance, automated remediation, zero-trust pipeline. |

**Your Goal: Move from 28 -> 90.**

---

## 4. Industry Standard Security Steps (Ranked by Urgency)

### Tier 1 — CRITICAL (Implement Immediately)

These prevent the highest-impact incidents and require minimal effort.

| Rank | Step | Why Critical | Effort |
|------|------|-------------|--------|
| **1** | **Enable Security Gates (`allow_failure: false`)** | A security scan that never fails is worse than no scan — it creates false confidence. | 1 hour |
| **2** | **Secrets Scanning (TruffleHog)** | Committed secrets are the #1 cause of cloud breaches. Must catch them before merge. | 2 hours |
| **3** | **Build-Time Dependency Scanning (Trivy FS + npm audit / OWASP DC)** | Container scanning misses dev dependencies and transitive vulnerabilities. Must scan at build time. | 3 hours |
| **4** | **SAST with Security-Focused Rules (Semgrep + SpotBugs/FindSecBugs)** | SonarQube Community/Developer misses OWASP Top 10 categories. Dedicated SAST tools catch injection, XSS, auth flaws. | 4 hours |
| **5** | **IaC / Dockerfile Scanning (Checkov + Dockle)** | Misconfigured Dockerfiles and K8s manifests are direct attack vectors. | 2 hours |

### Tier 2 — HIGH (Next Sprint)

| Rank | Step | Why High | Effort |
|------|------|----------|--------|
| **6** | **SBOM Generation (Syft)** | Required for supply chain transparency and rapid CVE response. Regulatory trend (EU CRA, US EO 14028). | 2 hours |
| **7** | **Container Image Signing (Cosign)** | Prevents image tampering and enables "deploy only signed images" policies. | 3 hours |
| **8** | **DAST Baseline (OWASP ZAP)** | Catches runtime issues SAST cannot: auth bypass, misconfigured headers, exposed endpoints. | 1 day |
| **9** | **Remove Redundant Grype** | Trivy already covers this. Pipeline time saved = developer time saved. | 30 min |

### Tier 3 — MEDIUM (Next Quarter)

| Rank | Step | Why Medium | Effort |
|------|------|------------|--------|
| **10** | **SLSA Provenance Attestation** | Cryptographic proof of how the artifact was built. Future-proofing against supply chain attacks. | 1 day |
| **11** | **License Compliance Scanning** | Prevents accidental inclusion of GPL/AGPL code that could force source code disclosure. | 4 hours |
| **12** | **Fuzzing / Property-Based Testing** | Catches edge-case crashes and parser vulnerabilities. | 2-3 days |

---

## 5. Recommended Open Source Tools (Justified)

### 5.1 Secrets Scanning: TruffleHog

**Why TruffleHog over GitLeaks:**

| Factor | GitLeaks | TruffleHog v3 |
|--------|----------|---------------|
| Detection Method | Regex-based + entropy | Regex + entropy **+ verified secrets** |
| False Positives | High — many regex hits | Lower — verifies by calling provider APIs |
| Built-in Detectors | ~100 | **800+** |
| Custom Rules Required? | Often — you complained about this | Rare for common secrets |
| Credential Verification | No | **Yes** — tests if the secret actually works |
| Performance | Fast | Fast (Go-based) |

**Your specific complaint about GitLeaks is exactly why TruffleHog exists.** GitLeaks relies heavily on regex patterns you must maintain. TruffleHog's "verified secrets" feature actually attempts to authenticate with the provider (AWS, Slack, GitHub, etc.) and only reports confirmed leaks. This eliminates the "extensive rules for each scenario" problem you described.

**Pipeline Integration:**
```yaml
trufflehog-scan:
  stage: validate
  image: trufflesecurity/trufflehog:latest
  script:
    - trufflehog git file://. --only-verified --fail
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_COMMIT_BRANCH == "master"'
```

> **Trigger:** Every merge request + develop/master commits
> **Stage:** `validate` (first stage — catch leaks before any build)

---

### 5.2 SAST: Semgrep + SpotBugs/FindSecBugs

**Why Semgrep over SonarQube for Security:**

| Factor | SonarQube Community | SonarQube Developer | Semgrep |
|--------|--------------------|---------------------|---------|
| Security Rule Depth | Minimal | Moderate | **Deep — OWASP Top 10, CWE, SANS 25** |
| Languages | Many | Many | **30+ including Java, TypeScript, JS** |
| Custom Rules | Complex plugin API | Complex | **Simple YAML-like syntax** |
| CI Integration | Heavy (server required) | Heavy | **Lightweight CLI, no server** |
| Speed | Slow (full analysis) | Slow | **Fast (diff-aware scanning)** |
| Cost | Free | $$$/year | **Free (Community) + OSS engine** |

SonarQube is excellent for **code quality** (cyclomatic complexity, duplication) but mediocre for **security**. Semgrep was built by security engineers at r2c and has deep coverage of injection flaws, auth bypasses, XSS, and insecure deserialization.

**For Java specifically:** Add **SpotBugs** + **FindSecBugs** plugin. This is the industry standard Java security static analysis combination. It catches:
- SQL Injection via dynamic queries
- XPath Injection
- Weak cryptography
- Insecure randomness
- XSS in servlets/JSP

**Pipeline Integration (Node.js):**
```yaml
semgrep-sast:
  stage: test_frontend
  image: returntocorp/semgrep:latest
  script:
    - semgrep ci --config=auto --json --output=semgrep-report.json
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_COMMIT_BRANCH == "master"'
  artifacts:
    reports:
      sast: semgrep-report.json
```

**Pipeline Integration (Java):**
```xml
<!-- In pom.xml -->
<plugin>
  <groupId>com.github.spotbugs</groupId>
  <artifactId>spotbugs-maven-plugin</artifactId>
  <version>4.8.0</version>
  <configuration>
    <plugins>
      <plugin>
        <groupId>com.h3xstream.findsecbugs</groupId>
        <artifactId>findsecbugs-plugin</artifactId>
        <version>1.13.0</version>
      </plugin>
    </plugins>
  </configuration>
</plugin>
```

> **Trigger:** Every MR + develop/master
> **Stage:** `test_frontend` / `test` (after compilation, before containerization)

---

### 5.3 SCA — Dependency Vulnerability Scanning: Trivy FS + OWASP Dependency-Check

**Why this combination:**

**For Node.js:** `npm audit` is built-in and zero-config. It checks against the NPM advisory database. However, it only covers direct dependencies well. **Trivy filesystem scan** (`trivy fs`) covers both direct and transitive dependencies using multiple databases (GitHub Advisory, NVD, GLAD, etc.).

**For Java:** **OWASP Dependency-Check** is the gold standard. It correlates Maven dependencies against the NVD (National Vulnerability Database) and maintains its own database of Java-specific CVEs. It integrates directly with Maven via plugin.

**Why not just Trivy image scan?**
Your current Trivy setup only scans the **container image**. But:
- Node.js `npm install --omit=dev` strips dev dependencies from the image
- Maven `package` may exclude test dependencies
- Transitive vulnerabilities in build tools (webpack, babel) are invisible in the final image

You need **build-time scanning** AND **image scanning**.

**Pipeline Integration (Node.js):**
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

**Pipeline Integration (Java):**
```xml
<!-- In pom.xml or CI script -->
<plugin>
  <groupId>org.owasp</groupId>
  <artifactId>dependency-check-maven</artifactId>
  <version>9.0.0</version>
  <configuration>
    <failBuildOnCVSS>7</failBuildOnCVSS>
  </configuration>
</plugin>
```

> **Trigger:** Every MR + develop/master
> **Stage:** `test_backend` / `test` (immediately after dependency resolution)

---

### 5.4 Container Security: Trivy Image + Dockle

**Keep Trivy, drop Grype, add Dockle.**

| Factor | Trivy | Grype | Dockle |
|--------|-------|-------|--------|
| Vulnerability DB | NVD, GitHub Advisory, Alpine SecDB, etc. | NVD + custom | N/A |
| OS Package Scanning | Excellent | Good | N/A |
| Application Package Scanning | Excellent | Good | N/A |
| Dockerfile Best Practices | No | No | **CIS Docker Benchmark** |
| Secrets in Image | Yes | No | Yes |
| Speed | Fast | Medium | Fast |

**Dockle** scans Dockerfiles and images for:
- Use of `root` user
- Hardcoded secrets in layers
- Missing `.dockerignore`
- Use of `latest` tag
- Sensitive directory permissions
- Health check absence

This is complementary to Trivy — Trivy finds CVEs, Dockle finds misconfigurations.

**Pipeline Integration:**
```yaml
dockle-scan:
  stage: scan
  image: goodwithtech/dockle:latest
  script:
    - dockle --exit-code 1 --format json $FULL_IMAGE_NAME > dockle-report.json
  allow_failure: false
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_COMMIT_BRANCH == "master"'
```

> **Trigger:** After Docker image build
> **Stage:** `scan` (existing)

---

### 5.5 DAST: OWASP ZAP

**Why ZAP:**
OWASP ZAP is the world's most widely used web app scanner. It's maintained by the OWASP foundation, completely open source, and has a robust automation framework (`zap-baseline.py`, `zap-full-scan.py`).

**What it catches that SAST cannot:**
- Authentication bypass in runtime
- Missing security headers (HSTS, CSP, X-Frame-Options)
- Exposed admin endpoints
- CORS misconfigurations
- Insecure cookie flags
- Rate limiting absence

**Pipeline Integration:**
```yaml
zap-baseline:
  stage: deploy  # or a new 'dast' stage
  image: owasp/zap2docker-stable:latest
  script:
    - zap-baseline.py -t $STAGING_URL -J zap-report.json
  allow_failure: false  # Set to true initially while tuning
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
  artifacts:
    reports:
      dast: zap-report.json
```

> **Trigger:** After deployment to staging (develop branch)
> **Stage:** Post-deploy (after `build-gitlab`, requires running app)

---

### 5.6 IaC Security: Checkov

**Why Checkov:**
- Supports Dockerfiles, Kubernetes manifests, Helm charts, Terraform, CloudFormation
- 750+ built-in policies based on CIS benchmarks
- No custom rules needed for standard compliance
- Can run as pre-commit hook or CI job

**What it catches:**
- `USER` not set in Dockerfile (runs as root)
- Privileged containers in K8s
- Missing resource limits
- Secrets in ConfigMaps
- Excessive RBAC permissions

**Pipeline Integration:**
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

> **Trigger:** Every MR + develop/master
> **Stage:** `validate` (catches misconfigurations before build)

---

### 5.7 SBOM Generation: Syft

**Why Syft:**
- Generates SBOMs in SPDX and CycloneDX formats (industry standards)
- Scans images, directories, or archives
- Pairs naturally with Grype (same company, Anchore) — but works standalone
- Required for:
  - Supply chain transparency
  - Rapid response to CVE disclosures ("Do we use Log4j?")
  - Regulatory compliance (EU Cyber Resilience Act)

**Pipeline Integration:**
```yaml
syft-sbom:
  stage: scan
  image: anchore/syft:latest
  script:
    - syft $FULL_IMAGE_NAME -o spdx-json=sbom.spdx.json
    - syft $FULL_IMAGE_NAME -o cyclonedx-json=sbom.cyclonedx.json
  artifacts:
    paths:
      - sbom.spdx.json
      - sbom.cyclonedx.json
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_COMMIT_BRANCH == "master"'
    - if: '$CI_COMMIT_TAG'
```

> **Trigger:** develop, master, tags
> **Stage:** `scan`

---

### 5.8 Container Signing: Cosign

**Why Cosign (Sigstore):**
- Industry standard for container signing (adopted by Kubernetes, npm, PyPI, GitHub)
- Keyless signing via OIDC (no key management overhead)
- Integrates with GitLab CI via `SIGSTORE_ID_TOKEN`
- Enables "deploy only signed images" policy in Kubernetes

**Pipeline Integration:**
```yaml
cosign-sign:
  stage: publish_gitlab
  image: bitnami/cosign:latest
  id_tokens:
    SIGSTORE_ID_TOKEN:
      aud: sigstore
  script:
    - cosign sign --yes $FULL_IMAGE_NAME
  rules:
    - if: '$CI_COMMIT_TAG'
```

> **Trigger:** Tags (releases)
> **Stage:** `publish_gitlab`

---

## 6. Complete Hardened Pipeline Architecture

### 6.1 Node.js Pipeline — Recommended Stages

```
validate          -> checkov (IaC) + trufflehog (secrets)
   |
test_frontend     -> node-test-frontend + semgrep-sast (frontend)
   |
test_backend      -> node-test-backend + trivy-fs (deps) + semgrep-sast (backend)
   |
build-frontend    -> angular_build_frontend
   |
build-backend     -> node_build_backend
   |
build-gitlab      -> gitlab_build_branch / gitlab_build_merge
   |
scan              -> trivy-image + dockle + syft-sbom
   |
publish           -> cosign-sign (tags only)
```

### 6.2 Java Pipeline — Recommended Stages

```
validate          -> checkov (IaC) + trufflehog (secrets) + maven-validate
   |
compile           -> maven-compile
   |
test              -> maven-test + spotbugs/findsecbugs + trivy-fs
   |
mvn-scan          -> sonarqube-check (quality only, not security gate)
   |
build             -> mvn_build / mvn_artifacts
   |
build-gitlab      -> gitlab_build_branch / gitlab_build_merge
   |
scan              -> trivy-image + dockle + syft-sbom
   |
publish           -> gitlab_publish + cosign-sign (tags only)
```

### 6.3 Git Action Triggers

| Step | Merge Request | Feature Branch | Develop | Master/Tag | Staging Deploy |
|------|--------------|----------------|---------|------------|----------------|
| TruffleHog | Yes | Yes | Yes | Yes | — |
| Checkov | Yes | Yes | Yes | Yes | — |
| Semgrep / SpotBugs | Yes | Yes | Yes | Yes | — |
| Trivy FS / npm audit / OWASP DC | Yes | Yes | Yes | Yes | — |
| Unit Tests | Yes | Yes | Yes | Yes | — |
| SonarQube | Yes | Yes | Yes | — | — |
| Docker Build | Yes | Yes | Yes | Yes | — |
| Trivy Image | Yes | Yes | Yes | Yes | — |
| Dockle | Yes | Yes | Yes | Yes | — |
| Syft SBOM | — | — | Yes | Yes | Yes |
| ZAP DAST | — | — | — | — | Yes |
| Cosign Sign | — | — | — | Yes (tag) | — |

---

## 7. Immediate Action Items (This Week)

### Day 1 — Gates (1 hour)
1. Change ALL `allow_failure: true` on security jobs to `allow_failure: false`
2. If you need a grace period, set `allow_failure: false` only on `develop` and `master`, keep `true` on feature branches temporarily.

### Day 1-2 — Secrets (2 hours)
1. Add TruffleHog job to `validate` stage
2. Run it locally first: `docker run trufflesecurity/trufflehog:latest git file://. --only-verified`
3. Fix any findings before enabling in CI

### Day 2-3 — Build-Time Dependencies (3 hours)
1. Node.js: Add `npm audit --audit-level=high` to `node-artifacts-backend` or as a separate job
2. Node.js: Add `trivy fs --scanners vuln --severity HIGH,CRITICAL --exit-code 1 .` job
3. Java: Add OWASP Dependency-Check Maven plugin to `pom.xml` with `<failBuildOnCVSS>7`

### Day 3-5 — SAST (4 hours)
1. Add Semgrep to Node.js pipeline (frontend + backend)
2. Add SpotBugs + FindSecBugs to Java `pom.xml`
3. Review and triage initial findings

### Week 2 — Container & IaC (2 hours)
1. Add Dockle job to `scan` stage
2. Add Checkov job to `validate` stage
3. Remove redundant Grype jobs (or keep one, drop the other)

### Week 3-4 — Advanced (3-5 days)
1. Add Syft SBOM generation
2. Add Cosign signing for tagged releases
3. Set up ZAP baseline scan against staging environment
4. Document everything in Confluence

---

## 8. Confluence Documentation & References

### Standards & Frameworks Referenced

| Standard | Relevance |
|----------|-----------|
| **OWASP CI/CD Security Top 10** | Comprehensive guide to securing CI/CD pipelines. Covers poisoned pipeline execution, insecure artifacts, insufficient logging. |
| **SLSA (Supply-chain Levels for Software Artifacts)** | Framework for provenance and supply chain integrity. Level 1 = provenance; Level 3 = hermetic builds + signed provenance. |
| **NIST SSDF (Secure Software Development Framework)** | US government standard. Core practices: Prepare the Organization, Protect the Software, Produce Well-Secured Software, Respond to Vulnerabilities. |
| **CIS Docker Benchmark v1.6.0** | Specific hardening guidance for Docker containers and hosts. |
| **OWASP ASVS 4.0** | Application Security Verification Standard — defines security requirements by level. |

### Recommended Confluence Pages to Create

1. **"CI/CD Security Policy"** — Define which scans run at which stages, severity thresholds, and exception process.
2. **"Security Tooling Runbook"** — How to interpret findings from each tool, who to contact, SLAs for remediation.
3. **"Dependency Vulnerability Response Playbook"** — What to do when a Critical CVE is found (e.g., Log4Shell scenario).
4. **"SBOM & Supply Chain Transparency"** — Where SBOMs are stored, how to query them, retention policy.
5. **"Pipeline Security Scorecard"** — Monthly tracking of pipeline security score against the rubric in this document.

---

## 9. Tool Comparison Summary

| Security Domain | Current Tool | Recommended Tool | Why Change |
|----------------|-------------|-----------------|------------|
| Secrets Scanning | None | **TruffleHog** | Verified secrets, 800+ detectors, no custom rules needed |
| SAST (Node) | SonarQube (weak) | **Semgrep** | Security-focused, 30+ languages, fast, deep OWASP coverage |
| SAST (Java) | SonarQube (weak) | **SpotBugs + FindSecBugs** | Industry standard Java security analysis |
| SCA (Node) | None | **npm audit + Trivy FS** | Built-in + comprehensive multi-DB coverage |
| SCA (Java) | None | **OWASP Dependency-Check** | Gold standard for Java CVE correlation |
| Container CVE | Trivy + Grype | **Trivy only** | Grype is redundant; Trivy is faster and more comprehensive |
| Container Config | None | **Dockle** | CIS benchmark for Dockerfiles |
| IaC Security | None | **Checkov** | 750+ policies, K8s/Helm/Dockerfile support |
| DAST | None | **OWASP ZAP** | World's most used web scanner, completely OSS |
| SBOM | None | **Syft** | SPDX/CycloneDX standard output, integrates with Sigstore |
| Signing | None | **Cosign** | Industry standard, keyless signing, Kubernetes-native |

---

## 10. Final Recommendation

**Do NOT try to implement everything at once.** The highest-ROI sequence is:

1. **Fix `allow_failure: true`** — This single change makes your existing tools actually useful.
2. **Add TruffleHog** — Prevents the #1 cause of cloud breaches.
3. **Add build-time dependency scanning** — Catches what container scanning misses.
4. **Add Semgrep / SpotBugs** — Catches code-level vulnerabilities SonarQube misses.
5. **Everything else** — Can follow in subsequent sprints.

This 4-step sequence will take your score from **28 -> 75+** within one week.

---

> **Document maintained by:** BMad Security Analysis
> **Next review:** After implementation of Tier 1 controls
> **Related artifacts:** `problem-inventory-cicd-test-automation.md`, `priority-list-cicd-test-automation.md`
