# Problem Inventory: CI/CD Pipeline Security

> **Status:** Active — Security audit complete, plan consolidated (June 2026)  
> **Sources:** `security-audit/cicd-security-audit-and-hardening-guide.md`, `security-audit/ci-cd-security-plan-consolidated.md`, `.gitlab-ci.yml` (Node + Java pipelines), direct stakeholder input (Zaryab Baloch — Security Lead/PO, Haroon — RMT)  
> **Last updated:** 2026-06-09  
> **Goal:** Consolidate security gaps before integrating into the master priority list  

---

## 1. Pipeline Security Enforcement

### 1.1 All Security Scans Run with `allow_failure: true`
- **Problem:** Every vulnerability scanner in both Node.js and Java pipelines — Trivy (branch + merge), Grype (branch + merge), and SonarQube (frontend + backend) — is configured with `allow_failure: true`. A CRITICAL CVE (CVSS 9.8) produces a red report that no one reads, and the image is still pushed to the registry and deployed.
- **Impact:** Security scans provide a false sense of safety. The pipeline shows green while untested, vulnerable code reaches production. This is the single highest-leverage security intervention in the entire pipeline.
- **Evidence:** Both `.gitlab-ci.yml` files show `allow_failure: true` on `trivy_scanning_branch`, `trivy_scanning_merge`, `grype_scan_branch`, `grype_scan_merge`, `sonarqube-frontend`, and `sonarqube-backend`. SonarQube has `-Dsonar.qualitygate.wait=true` set correctly, but the job still allows failure.
- **Source:** Direct pipeline inspection

### 1.2 SonarQube Quality Gate Has No Teeth
- **Problem:** Two SonarQube instances run with different configs — local lab (Community edition, 80% threshold) and live (Developer edition, 90% threshold). Community edition has minimal security rules. The two-instance setup creates confusion. Neither instance blocks the pipeline because `allow_failure: true` overrides the quality gate.
- **Impact:** Teams don't know which instance applies to them. Security hotspots, injection flaws, and coverage drops flow into releases unchecked. The quality gate is a suggestion, not a requirement.
- **Evidence:** SonarQube frontend job uses `SONAR_HOST_URL: "http://sonarqube.expertflow.com:9000"` (local lab). Backend job uses `SONAR_HOST_URL: "http://sonarqube-live.expertflow.com"` (live). Both have `allow_failure: true`.
- **Source:** `.gitlab-ci.yml` pipeline inspection

---

## 2. Secrets & Credential Management

### 2.1 No Secret Scanning in CI/CD
- **Problem:** Neither pipeline runs any secret detection. CI variables like `SONAR_PASS`, `SONAR_PASS_LIVE`, and `CI_PIPELINE_IID_TOKEN` are referenced in YAML. If any developer accidentally hardcodes these values rather than using variable references, they would be committed and pushed with zero detection.
- **Impact:** Committed secrets are the #1 cause of cloud breaches. A single exposed AWS key or database password in a feature branch can lead to production data exposure within hours of merge.
- **Evidence:** No secret scanning stage exists in either pipeline. No pre-commit hooks are configured. `SONAR_PASS` and `SONAR_PASS_LIVE` appear in plaintext variable references in `.gitlab-ci.yml`.
- **Source:** Direct pipeline inspection; OWASP CI/CD Security Top 10

### 2.2 No Pre-Commit Secret Prevention
- **Problem:** Secrets reach CI because no local check exists before push. Developers can commit API keys, tokens, and passwords without any friction.
- **Impact:** The CI pipeline is the last line of defense, but it's also the most expensive place to catch secrets — by the time CI runs, the secret is already in Git history and may have been exposed to anyone with repo access.
- **Evidence:** No `.pre-commit-config.yaml` exists in the repository. No detect-secrets or similar hook is configured.
- **Source:** Repository inspection; security best practices

---

## 3. Dependency Vulnerability Management

### 3.1 No Build-Time Dependency Scanning for Node.js
- **Problem:** `npm audit` never runs in the Node.js pipeline. The `node_artifacts_frontend` job uses `npm install --legacy-peer-deps`, which suppresses npm v7+ dependency conflict resolution. A vulnerable transitive dependency can slip in without any warning.
- **Impact:** Container scanning (Trivy/Grype) only scans the final image. `npm install --omit=dev` strips dev dependencies from the image. Transitive vulnerabilities in build tools (webpack, babel) and dev dependencies are invisible to image scanning.
- **Evidence:** `node_artifacts_frontend` script runs `npm install --legacy-peer-deps` with no `npm audit` step. `node-test-frontend` runs `npm run coverage --u` but has no coverage threshold.
- **Source:** `.gitlab-ci.yml` pipeline inspection

### 3.2 No Maven Dependency Vulnerability Scanning
- **Problem:** The Java pipeline runs `mvn verify` for tests but has no OWASP Dependency-Check or equivalent. Java ecosystems are specifically high-risk — Log4j, Spring4Shell, and Struts CVEs have all been catastrophic, and all would pass through the current pipeline undetected at the dependency level.
- **Impact:** A vulnerable Maven dependency in `pom.xml` is invisible until a customer reports it or Trivy image scan happens post-build — by which time the vulnerable artifact is already in the registry.
- **Evidence:** `maven-test` stage runs `mvn verify -s settings.xml` with no dependency vulnerability check. No OWASP plugin exists in `pom.xml`.
- **Source:** `.gitlab-ci.yml` and `pom.xml` inspection

### 3.3 Trivy Only Scans Container Image, Not Filesystem
- **Problem:** Trivy is configured to scan only the final Docker image (`FULL_IMAGE_NAME`). It does not scan the build context (`node_modules`, `.m2/repository`) before the image is built.
- **Impact:** Vulnerabilities in dependencies that are excluded from the final image (dev dependencies, build tools, test frameworks) are completely invisible. A `webpack-dev-server` CVE or a vulnerable test utility would never be detected.
- **Evidence:** `.trivy_scan_template` (included from `ci-templates`) only defines image scanning. No `trivy fs` job exists in either pipeline.
- **Source:** `.gitlab-ci.yml` include inspection; Trivy documentation

---

## 4. Container Security

### 4.1 Grype Runs Redundantly Alongside Trivy
- **Problem:** Both Trivy and Grype perform container vulnerability scanning on every branch build and merge request. Grype catches approximately 95% of what Trivy catches (same vulnerability databases) but adds 3-5 minutes per build.
- **Impact:** Pipeline time wasted = developer time wasted. At ~50 MRs/week, Grype adds 2.5–4 hours of CI time weekly with diminishing returns. The redundancy also creates confusion when the two scanners report different severity for the same CVE.
- **Evidence:** Both `trivy_scanning_branch` and `grype_scan_branch` run after `gitlab_build_branch`. Both have `allow_failure: true`. Grype uses the same NVD data sources as Trivy.
- **Source:** Pipeline inspection; Grype and Trivy documentation comparison

### 4.2 Docker Base Images Not Pinned to SHA Digests
- **Problem:** Dockerfiles use tag-based `FROM` statements (e.g., `FROM node:18.14.1-alpine3.17`). Tags are mutable — a tag can be re-pushed with entirely different content. `docker build --pull` always fetches the latest version of the tag.
- **Impact:** A malicious or accidental tag overwrite introduces unvetted base image content. There is no cryptographic guarantee that the base image used today is the same one used yesterday. This is a supply chain tampering vector.
- **Evidence:** Node pipeline `gitlab_build_branch` uses `docker build --pull`. Java pipeline uses the same. No SHA digest pinning exists in Dockerfiles.
- **Source:** `.gitlab-ci.yml` and Dockerfile inspection

### 4.3 docker:dind Runs in Privileged Mode Without Hardening
- **Problem:** Both pipelines use Docker-in-Docker (`docker:dind`) as a service without seccomp or AppArmor profiles. By default, this runs in privileged mode, giving the container full host kernel access.
- **Impact:** If a malicious dependency or build step executes in this context, it has a path to escape the container and compromise the runner host — potentially the GitLab instance itself.
- **Evidence:** `gitlab_build_branch` and `gitlab_build_merge` both define `services: - docker:dind` with no `security_opt`, no resource limits, and no custom daemon configuration.
- **Source:** `.gitlab-ci.yml` inspection; CIS Docker Benchmark

### 4.4 No Dockerfile Best-Practice Scanning
- **Problem:** No tool validates Dockerfiles for security misconfigurations before build. Common issues like running as root, missing `.dockerignore`, using `ADD` instead of `COPY`, or missing `HEALTHCHECK` go undetected.
- **Impact:** Misconfigured Dockerfiles are direct attack vectors. A `USER root` directive or an overly permissive `chmod` in a Dockerfile introduces vulnerabilities that persist in every image built from it.
- **Evidence:** No Dockerfile linting or security scanning stage exists. No Hadolint, Dockle, or Checkov job runs against Dockerfiles.
- **Source:** Pipeline inspection; Dockerfile review

---

## 5. Infrastructure-as-Code Security

### 5.1 No K8s/Helm Manifest Security Validation
- **Problem:** Expertflow deploys to Kubernetes via Helm charts, but no security scanning exists for K8s manifests, Helm charts, or Terraform. Misconfigurations like privileged containers, missing resource limits, secrets in ConfigMaps, or excessive RBAC permissions go undetected.
- **Impact:** A misconfigured Helm chart can deploy a container with `privileged: true`, mount the host filesystem, or expose a service without network policies. These are deployment-time vulnerabilities invisible to code scanning.
- **Evidence:** No IaC scanning tool (Checkov, kube-score, kubescape) is configured in CI. Helm charts are manually edited with no automated validation.
- **Source:** Pipeline inspection; infrastructure documentation

---

## 6. Runtime Security

### 6.1 No Dynamic Application Security Testing (DAST)
- **Problem:** No runtime security validation of deployed applications exists. SAST tools (SonarQube, Semgrep) catch code-level issues but cannot detect runtime misconfigurations like missing security headers, exposed admin endpoints, CORS misconfigurations, or authentication bypass.
- **Impact:** Runtime vulnerabilities are discovered only in RMT or — worse — by customers. The Angular frontend + Node/Java backends are exposed to the internet; missing CSP headers or exposed `/actuator` endpoints are exploitable.
- **Evidence:** No ZAP, Burp Suite, or equivalent DAST tool runs in CI or against staging. The only runtime check is manual RMT testing.
- **Source:** Pipeline inspection; OWASP Top 10 2021

---

## 7. Supply Chain & Provenance

### 7.1 No Software Bill of Materials (SBOM)
- **Problem:** No SBOM is generated for any release. When the next Log4j-equivalent CVE is announced, there is no way to quickly answer: "Do we use this component? In which version? In which services?"
- **Impact:** CVE response time is measured in days instead of minutes. Enterprise customers (FNB, Cisco) increasingly require SBOMs for vendor security assessments. The EU Cyber Resilience Act mandates SBOMs for software products by 2027.
- **Evidence:** No SBOM generation stage exists. No Syft, CycloneDX, or SPDX tooling is configured.
- **Source:** Pipeline inspection; regulatory trends

### 7.2 No Container Image Signing
- **Problem:** Images pushed to the GitLab registry have no cryptographic signature. At deployment time, Kubernetes pulls the image without verifying its provenance.
- **Impact:** A compromised registry, a man-in-the-middle attacker, or a malicious insider could substitute a tampered image. The deployment pipeline has no way to verify that the image was built by the official CI pipeline.
- **Evidence:** No Cosign, Notary, or equivalent signing step exists. No image verification policy exists in Kubernetes.
- **Source:** Pipeline inspection; supply chain security standards

### 7.3 No Build Provenance (SLSA)
- **Problem:** There is no attestation that an artifact was built by this specific pipeline, from this specific commit, with these specific dependencies.
- **Impact:** Without provenance, a compromised build environment could inject malicious code without detection. Forensic investigation of supply chain attacks is impossible without a signed build record.
- **Evidence:** No SLSA Level 1 (provenance), Level 2 (signed provenance), or Level 3 (hermetic builds) practices exist.
- **Source:** Pipeline inspection; SLSA framework

---

## 8. Quality Gate & Process Enforcement

### 8.1 Java Branch Build Is Manual, Bypassing Security Scans
- **Problem:** The Java pipeline's `gitlab_build_branch` job has `when: manual`. This means branch builds never trigger Trivy/Grype scans unless someone manually clicks the build button. A developer can push to a feature branch and merge via MR (where merge-scans run), but the *branch* image itself is never scanned.
- **Impact:** If someone deploys the branch image directly — for hot-fixes, ad-hoc testing, or air-gapped environments — it bypasses all security scanning. This is an intentional CI cost-saving measure that creates a security gap.
- **Evidence:** `gitlab_build_branch` in Java `.gitlab-ci.yml` has `when: manual` with comment: `#+==================Runs on every commit (manual)========================+`.
- **Source:** `.gitlab-ci.yml` inspection

### 8.2 Code Format Job Disabled in Node Pipeline
- **Problem:** The `node-format-frontend` job is entirely commented out. It was disabled because the custom image (`gitlab.expertflow.com:9242/general/node:CSN-3623`) became unavailable. Without format enforcement, code style drift accumulates and mixed formatting makes security review harder.
- **Impact:** Inconsistent formatting increases cognitive load during code review. Security-relevant code (auth checks, input validation) is harder to spot and verify when buried in formatting noise. The absence of this gate also signals that quality enforcement is optional.
- **Evidence:** Lines 75–93 of Node `.gitlab-ci.yml` show the entire `node-format-frontend` job commented out with `#`.
- **Source:** `.gitlab-ci.yml` inspection

### 8.3 No Coverage Threshold at Test Runner Level
- **Problem:** `npm run coverage --u` and `mvn verify` run tests with coverage reporting but no minimum threshold. A developer can delete all tests and the pipeline still passes.
- **Impact:** Untested code paths are unvalidated attack surfaces — injection points, authentication bypasses, and business logic errors often live in paths with no test coverage. The SonarQube threshold (80%/90%) exists but is unenforced due to `allow_failure: true`.
- **Evidence:** `node-test-frontend` runs `npm run coverage --u` with no `coverageThreshold` in `jest.config.js`. `maven-test` runs `mvn verify` with no Jacoco threshold in `pom.xml`.
- **Source:** `.gitlab-ci.yml`, `jest.config.js`, and `pom.xml` inspection

---

## 9. Missing Information

The following security assessments have been identified but **not yet formally incorporated** into this inventory:

| Source | Status | Action Needed |
|--------|--------|---------------|
| Confluence: CI/CD Security Policy | 🔴 Not yet created | Create after Phase 1 implementation |
| Confluence: Security Tooling Runbook | 🔴 Not yet created | Create after tool selection finalized |
| Confluence: Dependency Vulnerability Response Playbook | 🔴 Not yet created | Create after OWASP DC + Trivy FS deployed |
| `.gitlab-ci.yml` template files in `ci-templates` repo | ⚠️ Partially reviewed | Review `.trivy_scan_template`, `.grype_template` for hardening |

---

## Next Steps

1. **Validate inventory:** Review with Zaryab Baloch (Security Lead/PO), Haroon (DevOps), and stream leads for completeness.
2. **Prioritize:** Classify problems by urgency/impact and integrate into the master priority list (`priority-list-cicd-test-automation.md`).
3. **Assign DRIs:** Confirm ownership for each security control — Security Lead for policy/tooling, DevOps for CI integration, Stream Leads for language-specific items.
4. **Proceed to architecture review:** Route to Winston (System Architect) for pipeline stage ordering and infrastructure compatibility validation.
5. **Begin Phase 1 implementation:** Enable enforcing gates (`allow_failure: false`) and add TruffleHog as immediate wins.
