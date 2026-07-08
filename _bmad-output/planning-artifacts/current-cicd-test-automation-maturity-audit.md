# Current CI/CD and Test Automation Maturity Audit

> **Prepared:** July 2026
> **Auditor scope:** `CX-5.4.0/.gitlab-ci.yml` (cim-solution skeleton), `playwright-automation-script/` (`.gitlab-ci.yml`, `CX-Chat-Cases.spec.js`, `playwright.config.js`, `package.json`, `scripts/notify-google-chat.sh`, `README.md`), cross-referenced with `security-audit/cicd-security-audit-and-hardening-guide.md`
> **Classification key:** Severity = Critical/High/Medium/Low · Effort = Small (≤half day) / Medium (≤1 week) / Large (>1 week) · Category = CI / CD / Test / Security

---

## ⚠️ Immediate Action Required (found during audit)

**Live credentials are committed in `CX-5.4.0/.gitlab-ci.yml:337-340`:**

```yaml
REGISTRY_URL:  "gitimages.expertflow.com"
REGISTRY_USER: "efcx"
REGISTRY_PASS: "RecRpsuH34yqp56YRFUb"
POSTGRES_PASS: "Expertflow123"
```

These sit inside the commented-out `deploy_charts` block, but **commented YAML is still committed plaintext** — in this file, in the cim-solution repo it was copied from, and in the full git history of both. The security audit doc flagged "hardcoded internal URLs" (§2.3.4) but missed these actual credentials. Rotate both credentials, replace with masked CI/CD variables, and treat the git history as compromised for these values. This is the single finding in this audit that should be actioned today.

---

## 1. Configuration Review

### 1.1 Pipeline Stages and Job Design

**cim-solution (`CX-5.4.0/.gitlab-ci.yml`)**

The designed pipeline (`detect → build → publish → deploy → test → notify`) is sound, but lines 6–9 show `detect`, `build`, `publish`, `deploy` **all commented out** — the committed state runs only `test → notify`. The RC-version auto-bump logic in the disabled `package_charts` job (lines 100–210: registry pagination, `-rc.N` calculation, Chart.yaml rewrite) is well-engineered, but it is dormant code drifting from reality with every week it stays commented.

| # | Finding | Location | Severity | Effort | Category |
|---|---------|----------|:---:|:---:|:---:|
| 1.1a | Build/publish/deploy stages commented out; pipeline is test-only, so no artifact can be produced from this branch state | `CX-5.4.0/.gitlab-ci.yml:6-9` | High | Medium | CI |
| 1.1b | `regression-test` is `when: manual` (line 468) + `allow_failure: true` (line 469) — regression is informational, never gating | `CX-5.4.0/.gitlab-ci.yml:467-469` | High | Small* | CI/Test |
| 1.1c | `RELEASE_VERSION: "5.5.0"` hardcoded (line 18) overrides branch-derived versioning; forgetting to bump it publishes charts under the wrong version | `CX-5.4.0/.gitlab-ci.yml:18` | Medium | Small | CI |
| 1.1d | `helm lint … \|\| true` swallows lint failures in the (dormant) build job | `CX-5.4.0/.gitlab-ci.yml:228-230` | Medium | Small | CI |
| 1.1e | Version-bump job pushes back to the source branch with `[skip ci]` using `$GITLAB_TOKEN` — bot-push race risk on concurrent MRs; token scope undocumented | `CX-5.4.0/.gitlab-ci.yml:251-263` | Low | Small | CI |

*\*1.1b is Small only mechanically; the documented plan correctly gates it on CRM-763 + flake-rate evidence — see §5.*

**playwright-automation-script (`.gitlab-ci.yml`)**

Two-stage `test → notify` design is clean. `needs: + artifacts: true` (lines 58–60) and the dotenv job-ID handoff (`regression.env`, lines 34–43) are correct patterns.

| # | Finding | Location | Severity | Effort | Category |
|---|---------|----------|:---:|:---:|:---:|
| 1.1f | `rules: - when: always` — pipeline fires on every push of every branch with no MR/branch scoping | `playwright-automation-script/.gitlab-ci.yml:44-45` | Low | Small | CI |
| 1.1g | Redundant `when: always` + `rules: when: always` on notify job (lines 61, 67–68) — the `rules:` key silently wins; dead config invites misreading | `playwright-automation-script/.gitlab-ci.yml:61,67-68` | Low | Small | CI |

**What is done well:** JUnit + dotenv artifact reports wired correctly (`CX-5.4.0:500-509`); 20-minute job timeout caps runaway runs; the notify script handles the missing-results case with an explicit "INCOMPLETE" card and builds JSON via `jq -n --arg` (injection-safe) rather than string interpolation (`notify-google-chat.sh:30-65, 93-162`).

### 1.2 Caching, Artifacts, and Image Strategy

| # | Finding | Location | Severity | Effort | Category |
|---|---------|----------|:---:|:---:|:---:|
| 1.2a | No `cache:` anywhere in either pipeline — `npm ci`/`npm install` re-downloads dependencies on every run; the cim-solution job also re-clones the Playwright repo each run (`git clone --depth 1`, line 474) | both `.gitlab-ci.yml` files | Medium | Small | CI |
| 1.2b | `image: alpine:latest` — mutable tag, unpinned; build behavior can change under you | `playwright-automation-script/.gitlab-ci.yml:57` | Medium | Small | Security/CI |
| 1.2c | Images pinned by tag (`playwright:v1.61.0-jammy`, `alpine:3.20`) not SHA digest — tag-tampering unmitigated (matches audit-guide §2.3, anti-pattern: mutable tags) | `CX-5.4.0:457,525`; `playwright-automation-script:28` | Medium | Small | Security |
| 1.2d | Inconsistent image sourcing: cim-solution pulls Playwright image from the internal mirror (`gitlab.expertflow.com:9242/...`, line 457, with `pull_policy: if-not-present` — good), while the Playwright repo's own pipeline pulls `mcr.microsoft.com` directly (line 28) | both files | Low | Small | CI |

Artifact strategy is otherwise sound: HTML report + JUnit + `regression.env` retained 1 week with `when: always`, so failed runs keep their evidence.

### 1.3 Parallelization and Speed

| # | Finding | Location | Severity | Effort | Category |
|---|---------|----------|:---:|:---:|:---:|
| 1.3a | `workers: 1` — entire suite serial. Root cause is **shared test accounts** (`umer12`/`umar13` reused across all suites), a test-data problem surfacing as a speed ceiling. 4.7 min today; will scale linearly as suites become real | `playwright.config.js:12-13`; `CX-Chat-Cases.spec.js:19-30` | Medium | Medium | Test |
| 1.3b | No login-state reuse (`storageState` / global-setup): Suites 2, 3, 4 each log in two agents from scratch — repeated ~30–60 s Keycloak flows per suite | `CX-Chat-Cases.spec.js:484-485,547-548,605-606` | Medium | Small | Test |
| 1.3c | File-wide `test.describe.configure({ mode: 'serial' })`: one suite's failure **skips all later suites**, and with `retries: 1` a late flake re-runs the whole chain from the top — worst-case doubles runtime and reports Suites N+1…10 as NOT RUN | `CX-Chat-Cases.spec.js:3`; `playwright.config.js:18` | High | Small | Test |

### 1.4 Secrets and Credential Handling

| # | Finding | Location | Severity | Effort | Category |
|---|---------|----------|:---:|:---:|:---:|
| 1.4a | **Registry + PostgreSQL passwords committed in YAML** (see banner above) | `CX-5.4.0/.gitlab-ci.yml:339-340` | **Critical** | Small | Security |
| 1.4b | Real test usernames/passwords as in-code fallbacks: `umer12`/`12345`, `umar13`/`12345`. Runs "work" without CI variables, so the fallback path gets exercised and the weak creds are documented for anyone with repo access to a shared environment (`mtt02.expertflow.com`, also defaulted at line 9) | `CX-Chat-Cases.spec.js:20-29` | High | Small | Security/Test |
| 1.4c | Commented deploy block falls back to `--insecure-skip-tls-verify=true` when the CA cert doesn't parse — a security downgrade that self-activates on a malformed variable, with only a WARNING line. Will go live when the block is uncommented for CRM-763 | `CX-5.4.0/.gitlab-ci.yml:371-374` | High (latent) | Small | CD/Security |

**Done well:** Playwright repo cloned with `CI_JOB_TOKEN` (`CX-5.4.0:460`) — short-lived, scoped, correct. `GOOGLE_CHAT_WEBHOOK` documented as mask+protect and the script exits gracefully when unset (`notify-google-chat.sh:25-28`). `playwright.config.js:31-40` hard-fails in CI when `BASE_URL` is missing instead of silently testing the wrong environment.

### 1.5 Branch / Release Triggers

| # | Finding | Location | Severity | Effort | Category |
|---|---------|----------|:---:|:---:|:---:|
| 1.5a | `PLAYWRIGHT_BRANCH: "main"` — cim-solution always tests with the tip of the test repo; no version pinning to CX releases. A 5.4.0 deploy can be judged by 5.5.0-era tests. (Already ticketed: CRM-768) | `CX-5.4.0/.gitlab-ci.yml:463` | High | Small | CI/Test |
| 1.5b | Disabled build/publish rules fire only on `merge_request_event`; direct pushes to release branches would produce nothing — acceptable if branch protection enforces MR-only, but that dependency is undocumented | `CX-5.4.0:37-39,74-77` | Low | Small | CI |

---

## 2. Test Automation Review

### 2.1 Test Structure and Maintainability

The suite is one 779-line file mixing config, helpers, a bespoke result tracker, and 10 suites.

| # | Finding | Location | Severity | Effort | Category |
|---|---------|----------|:---:|:---:|:---:|
| 2.1a | **27+ empty `test.step()` stubs claim CIM test-case coverage with zero assertions** — e.g. `CIM-22292 – Verify agent cannot accept chats beyond limit` is an empty `async () => {}`. Reports (JUnit, HTML, and anything mapping steps→Jira) show these case IDs as executed. This materially inflates the "70% coverage" figure and directly undermines CRM-766's coverage re-baseline | `CX-Chat-Cases.spec.js:397-424, 465-476, 534-539, 594-597` | **High** | Medium | Test |
| 2.1b | **Suites 5–10 are stubs that mark features PASSED after login or page-load only**: "Agent MRD", "Agent State", "Notifications" pass on `setupAgent()` alone; "Customer Widget" and "Field Validation" pass on `goto` + `networkidle`. 6 of the "10/10 suites passed" verify nothing domain-specific | `CX-Chat-Cases.spec.js:662-780` | **High** | Large | Test |
| 2.1c | Bespoke `RESULTS` tracker + console summary duplicates the reporter; features get marked as side effects of unrelated flows — `markPassed('Bot Messages')` fires inside `startCustomerChat()` (line 179), `Chat Transcript` inside `closeConversation()` (line 222), so "Bot Messages PASSED" is really "Suite 1's setup reached the bot greeting" | `CX-Chat-Cases.spec.js:51-106,179,222,231` | Medium | Medium | Test |
| 2.1d | ~60 lines of icon-hunting logic duplicated nearly verbatim between `initiateAgentConsult()` (250-305) and `transferToAgent()` (319-384) | `CX-Chat-Cases.spec.js:250-384` | Medium | Small | Test |
| 2.1e | Repo `README.md` is the untouched GitLab template — zero onboarding documentation for the repo the whole regression program depends on | `playwright-automation-script/README.md` | Low | Small | Test |

### 2.2 Page-Object Model Usage

**There is none** — and this contradicts the org's own written standards twice over: the master pipeline doc §11.2 specifies a `features/ + tests/pages/ + tests/steps/` structure in a repo named `cx-automation-playwright`, and the QA Automation plan (team_input) mandates POM with clean step files. The actual repo is `playwright-automation-script`, flat, with free-function helpers inside the spec file. Selector reality also inverts the documented priority order (getByTestId → getByRole → never CSS):

| # | Finding | Location | Severity | Effort | Category |
|---|---------|----------|:---:|:---:|:---:|
| 2.2a | No POM layer; helpers and specs interleaved in one file; doc/practice divergence including repo name | whole spec vs. master doc §11.2 | Medium | Large | Test |
| 2.2b | Brittle selectors of exactly the class the standard forbids: `.composer-main` (197), `ef-contact-item, mat-option, [class*="contact-item"]` (261, 335), `button[mattooltip="Queue Transfer"]` (628), `img[alt=…]`/`[title=…]` fallback arrays (279-284, 353-358), `textarea#messageTextarea` (180) — Angular dynamic classes make these rot on every frontend build | `CX-Chat-Cases.spec.js` as cited | High | Large* | Test |
| 2.2c | Zero `data-testid` usage — the "first choice" selector strategy requires a dev-team agreement that has not happened (flagged as pending in the QA plan since June) | entire spec | Medium | Medium (cross-team) | Test |

*\*Large because the durable fix is the `data-testid` program with frontend teams, not just rewriting locators.*

### 2.3 Data / Test-Environment Handling

| # | Finding | Location | Severity | Effort | Category |
|---|---------|----------|:---:|:---:|:---:|
| 2.3a | Hardcoded customer phone numbers per suite (`'0900'`, `'1500'`, `'13000'`, `'14009'`) — no per-run uniqueness; residue from a crashed run (an open conversation for `0900`) can poison the next run on the same environment | `CX-Chat-Cases.spec.js:431,487,550,608` | High | Small | Test |
| 2.3b | No setup/teardown of environment state: tests assume agents start logged-out and chat-free; there is no cleanup of orphaned conversations on failure (the `finally` blocks close browser contexts, not server-side state) | throughout | Medium | Medium | Test |
| 2.3c | Environment coupling: queue name defaults to `'Umer'` (a person's name), widget URL hardcodes `serviceIdentifier=1100` — config that will silently mismatch on any environment other than mtt02 | `CX-Chat-Cases.spec.js:16,32` | Medium | Small | Test |

### 2.4 Flakiness Risks

| # | Finding | Location | Severity | Effort | Category |
|---|---------|----------|:---:|:---:|:---:|
| 2.4a | `waitForLoadState('networkidle')` in the shared `waitForPage()` helper and Suites 8/10 — discouraged by Playwright, and this is a **chat application with persistent websocket/polling traffic**; networkidle is nondeterministic here by nature | `CX-Chat-Cases.spec.js:114,729,771` | High | Small | Test |
| 2.4b | Hard sleeps `waitForTimeout(600)` after hover to wait for icon reveal | `CX-Chat-Cases.spec.js:277,351` | Medium | Small | Test |
| 2.4c | Serial-chain blast radius + whole-chain retry (see 1.3c) — the flake-rate metric the roadmap depends on (<5% to un-pause T1-1) cannot be measured meaningfully while one flake cascades into 9 skips | `CX-Chat-Cases.spec.js:3` | High | Small | Test |
| 2.4d | Race-tolerant patterns exist but hide failures: `Promise.race` of two locator waits with `.catch(() => {})` (269-272) means both can time out silently and the subsequent `isVisible()` check picks a non-existent row, producing a confusing downstream error | `CX-Chat-Cases.spec.js:269-273,343-347` | Medium | Small | Test |
| 2.4e | No flake tracking: `retries: 1` masks single flakes with no record that a retry occurred (the JSON reporter captures it, but nothing consumes `stats.flaky`) | `playwright.config.js:18`; `notify-google-chat.sh:44-54` | Medium | Small | Test |

### 2.5 BDD / Gherkin Coverage

Not applicable in practice — no `features/`, no Gherkin, no `@smoke`/`@regression` tags despite both planning docs specifying them. The smoke suite is approximated by a grep hack: `--grep 'Suite 1[^0-9]'` (`package.json:7`). Either implement the documented BDD layer or formally drop it from the standard; the current state is a standing doc/reality contradiction. **(Medium / Medium / Test)**

---

## 3. Security Review

### 3.1 SAST / DAST / Dependency Scanning Presence

Confirms the security-audit guide's findings for these two repos specifically:

| # | Finding | Location | Severity | Effort | Category |
|---|---------|----------|:---:|:---:|:---:|
| 3.1a | Zero security jobs in either pipeline: no `npm audit`, no secret scanning, no SAST, no Checkov/IaC scanning of the Helm charts this skeleton exists to package | both `.gitlab-ci.yml` files | High | Medium | Security |
| 3.1b | `npm install` (not `npm ci`) in the Playwright repo's own pipeline — lockfile not strictly honored, unreproducible dep tree (cim-solution's clone-and-run uses `npm ci` correctly at `CX-5.4.0:476`; the repos disagree) | `playwright-automation-script/.gitlab-ci.yml:31` | Medium | Small | Security/CI |

### 3.2 Image and Supply-Chain Controls

Covered in 1.2b–1.2d: mutable `alpine:latest`, tag-not-digest pinning, split image sourcing. No SBOM, no signing (consistent with audit guide §2.2 — planned in T1-6 Phases 2–3, not present here).

### 3.3 Secret Management

Covered in 1.4a–1.4c. Summary: one **Critical** committed-credential finding, one High in-code default-credential finding, one High latent TLS-bypass, against genuinely good patterns for the webhook and job-token handling.

---

## 4. Consolidated Findings Register

| ID | Finding (short) | Severity | Effort | Category |
|----|-----------------|:---:|:---:|:---:|
| 1.4a | Registry + Postgres passwords committed in YAML | **Critical** | Small | Security |
| 2.1a | Empty CIM test.steps fake coverage | High | Medium | Test |
| 2.1b | Suites 5–10 stubs report PASSED | High | Large | Test |
| 1.4b | Default weak credentials in spec source | High | Small | Security/Test |
| 1.4c | Latent `--insecure-skip-tls-verify` fallback in deploy block | High | Small | CD/Security |
| 1.1a | Build/publish/deploy stages commented out | High | Medium | CI |
| 1.1b | Regression manual + allow_failure (non-gating) | High | Small* | CI/Test |
| 1.5a | Test suite unpinned to CX version (`main`) | High | Small | CI/Test |
| 2.2b | Brittle CSS/mattooltip/class selectors | High | Large | Test |
| 2.3a | Hardcoded phone numbers, no run isolation | High | Small | Test |
| 2.4a | `networkidle` waits on a websocket app | High | Small | Test |
| 2.4c | File-wide serial mode: cascade skips + whole-chain retry | High | Small | Test |
| 3.1a | No security jobs in either pipeline | High | Medium | Security |
| 1.2a | No caching (npm, repo clone) | Medium | Small | CI |
| 1.2b | `alpine:latest` unpinned | Medium | Small | Security/CI |
| 1.2c | Tag pinning, not digest pinning | Medium | Small | Security |
| 1.3a | `workers: 1` forced by shared accounts | Medium | Medium | Test |
| 1.3b | No storageState login reuse | Medium | Small | Test |
| 1.1c | Hardcoded `RELEASE_VERSION` | Medium | Small | CI |
| 1.1d | `helm lint \|\| true` | Medium | Small | CI |
| 2.1c | Bespoke result tracker, side-effect passes | Medium | Medium | Test |
| 2.1d | Duplicated consult/transfer logic | Medium | Small | Test |
| 2.2a | No POM; contradicts own standard §11.2 | Medium | Large | Test |
| 2.2c | No data-testid program | Medium | Medium | Test |
| 2.3b | No state cleanup on failure | Medium | Medium | Test |
| 2.3c | Person-named queue, hardcoded serviceIdentifier | Medium | Small | Test |
| 2.4b | Hard sleeps after hover | Medium | Small | Test |
| 2.4d | Silent `Promise.race(...).catch()` swallowing | Medium | Small | Test |
| 2.4e | No flake-rate tracking | Medium | Small | Test |
| 2.5 | BDD/tags specified but absent | Medium | Medium | Test |
| 3.1b | `npm install` vs `npm ci` inconsistency | Medium | Small | Security/CI |
| 1.1e | Bot-push race, undocumented token scope | Low | Small | CI |
| 1.1f | `when: always` — no branch scoping | Low | Small | CI |
| 1.1g | Redundant when/rules keys | Low | Small | CI |
| 1.2d | Split image sourcing across repos | Low | Small | CI |
| 1.5b | MR-only build depends on undocumented branch protection | Low | Small | CI |
| 2.1e | Template README | Low | Small | Test |

---

## 5. Top 10 Prioritized Remediation Actions

1. **Rotate and remove the committed registry/Postgres credentials** (`CX-5.4.0/.gitlab-ci.yml:339-340`). Rotate `efcx` registry password and `Expertflow123` today; move to masked+protected CI variables; assume git history is compromised. *(Critical · Small · Security — do first, independent of everything else)*
2. **Delete credential fallbacks from `CX-Chat-Cases.spec.js:20-29`** — require `AGENT1_USER` etc. and fail fast with a clear error (the config already does this for `BASE_URL`; extend the same pattern). *(High · Small · Security)*
3. **Restore coverage integrity: remove or explicitly quarantine the empty CIM `test.step()` stubs and the login-only Suites 5–10.** Tag stubs `@stub`, exclude them from pass counts and the notify card, and re-state the real coverage number as part of CRM-766. Reports must stop claiming verification that doesn't exist — this also protects the credibility the roadmap needs QA to place in automation. *(High · Medium · Test)*
4. **Remove file-wide serial mode** (`CX-Chat-Cases.spec.js:3`) so suites are independent; a Suite 1 failure must not skip Suites 2–10 or trigger whole-chain retries. Prerequisite for any meaningful flake-rate measurement (<5% gate for T1-1 un-pause). *(High · Small · Test)*
5. **Pin the test suite to CX releases** — implement CRM-768 (`PLAYWRIGHT_BRANCH` → `PLAYWRIGHT_REF` = `v5.4.0` tag), and change `npm install` → `npm ci` in `playwright-automation-script/.gitlab-ci.yml:31`. *(High · Small · CI/Test)*
6. **Kill the top flake sources:** replace `networkidle` waits with element-level expectations (a websocket chat app will never reliably be network-idle), remove the two `waitForTimeout(600)` sleeps in favor of `locator.hover()` + visible-wait on the icon, and stop swallowing `Promise.race` timeouts. *(High · Small · Test)*
7. **Add per-run data isolation:** generate unique customer phone/name per run (e.g. timestamp-derived), and parameterize `serviceIdentifier` + queue name as env vars. Then provision a second agent-account pair to unlock `workers: 2` and add `storageState` login reuse. *(High→Medium · Medium · Test)*
8. **Add the minimum security jobs to both pipelines** per the already-approved T1-6 Phase 1: secret scanning (would have caught finding #1), `npm audit` in the Playwright repo, Checkov over `kubernetes/helm/` in cim-solution — all `allow_failure: false`. *(High · Medium · Security)*
9. **Fix the deploy block before it goes live with CRM-763:** remove the `--insecure-skip-tls-verify=true` fallback (fail hard on a bad CA instead, lines 371-374), remove `helm lint || true`, and source registry/Postgres values from CI variables. Cheapest moment to fix it is while it's still commented out. *(High-latent · Small · CD/Security)*
10. **Start the POM/selector refactor as an incremental track:** extract shared helpers into page objects (dedupe the consult/transfer twins first, `CX-Chat-Cases.spec.js:250-384`), and open the `data-testid` agreement with the frontend teams — the durable fix for every brittle-selector finding. Align repo structure (or the standard in master doc §11.2) so documentation and reality match. *(Medium · Large · Test — run alongside, don't block, items 1–9)*

---

### Maturity Verdict

The pipeline **wiring** (artifact handoffs, job tokens, notify robustness, reporter config) is better than the maturity scores in the benchmark report suggest — whoever built the T1-1b loop knew GitLab CI well. The gaps are in **what the pipeline is allowed to enforce** (nothing yet), **what the tests actually verify** (4 real suites, 6 stubs, 27 no-op steps), and **hygiene debt** (committed credentials, unpinned refs, shared test data). Items 1–6 above are all Small/Medium effort and would materially change the trustworthiness of the green card the team sees in Google Chat every run — which is exactly the currency the Phase 2/3 roadmap plans to spend.
