# VAPT Process \- Draft

### Overview and Connection to the Release Workflow

This document defines when and how vulnerability assessments happen within Expertflow's existing Continuous Delivery Process (CDP). There are two distinct vulnerability identification mechanisms — one that runs continuously throughout the development lifecycle (Track 1) and one that runs at a specific point in the release cycle (Track 2). Neither replaces the other; they catch different classes of vulnerabilities.

The diagram below shows where each track fits within the existing release management workflow:

### **1. Pre-Scan Preparation**

Security activities are categorized into Continuous Automated Scanning and Point-in-Time Manual Assessment.

**Trigger:** When a Release Candidate (RC) is cut for a major/minor version (e.g., CX 5.0, CX 5.1).

**Action:** Release Manager creates a Jira task for VA/Pentesting and assigns it to the Security Architect/Pentester.

**Setup:** Security Architect/Pentester prepares a dedicated Security QA environment using the specific RC build. This must occur before the final release announcement to allow time for remediation.

---

### **2. Vulnerability Identification**

Two parallel tracks run simultaneously to ensure both package-level and logic-level security coverage.

**Track 1 — Automated CI/CD Pipeline Scanning (Trivy & Grype)**

Scanning is fully automated and integrated into the GitLab CI/CD lifecycle. Within the CI stages for merge requests, the `scan` stage runs after the `build-gitlab` stage. This triggers on every MR — feature → develop, develop → main, and any fix MR during the RC phase. It is a permanent gate and runs for the entire lifetime of the product, not just at release time.

Branch scans run on every commit to `master`, `develop`, feature branches (`*_f-*`), and bug branches (`*_b-*`). These are informational only — they do not block the pipeline. Results are saved as artifacts.

MR scans are enforced. Critical and High findings with an available upstream fix block the merge. The developer must resolve the finding, update the affected package to the Fixed Version listed in the Trivy output, and re-run the pipeline before the MR can proceed. This runs alongside the existing `mvn-scan` (SonarQube) gate — both must pass.

**How many times does Track 1 run?** Track 1 MR enforcement runs on every single MR indefinitely — this is by design and has no stopping point. However the release triage (the consolidated snapshot review) runs exactly once per release cycle, during the RC deployment to RMT phase. It is not repeated in a loop. If new CVEs are published after the snapshot is taken, they are handled in the next maintenance cycle. The release gate is evaluated against the snapshot, not against a continuously moving target.

**Track 2 — Manual Penetration Testing (Security Architect/Pentester)**

Track 2 starts when the RC is deployed to RMT and RMT feature testing begins. The Security Architect sets up the Security QA environment using the same RC build deployed on RMT and runs the manual assessment in parallel with the feature tests and automation regression cycle. This ensures Track 2 completes before Feature Freeze, so all pentest findings are in Jira and remediable before the release gate check.

Track 2 finds flaws that automated image scanning cannot detect — logic flaws, authentication bypasses, API misconfigurations, and business-layer issues. These are vulnerabilities in how the application behaves at runtime, not in which packages the image contains.

- Scan the entire CX solution using pentesting tools (e.g., Zap, Burp, Nmap, Wireshark) following the Penetration Testing Guide: [Penetration Testing Guide](https://expertflow-docs.atlassian.net/wiki/spaces/EF/pages/884277326/Penetration+Testing+Guide)
- Produce a Pentest report in PDF format including a summary, severity, and details of all found vulnerabilities.
- Create a Jira epic for that specific release scan (e.g., pentest vulnerabilities of CX 5.0) and attach the PDF report to it.
- Create a Jira bug for each vulnerability found and attach it to the epic.
- Assign the Jira epic to the Security Product Owner for remediation.

**New P1/P2 Findings Discovered After the Release Triage Snapshot**

The release triage snapshot taken at RC deployment is the definitive finding list for the release. However two scenarios can surface new findings after the snapshot:

**Scenario 1** — A fix MR merged during RC stabilization or the Feature Stabilization window introduces a new vulnerable dependency. The Track 1 MR scan on that fix MR will catch it. If it is P1/P2, it is treated as a release gate blocker. The Discovery/Verification Date is set to the date the fix MR scan surfaced it.

**Scenario 2** — The Final Regression Suite surfaces a new P1/P2 finding that was not in the snapshot. This is rare but possible if a new CVE is published between the snapshot date and the regression date. If this happens, the Security Architect flags it immediately, it is classified per Section 4, and if it is P1/P2 it becomes a release gate blocker. The release does not promote until it is resolved or formally excepted.

In both scenarios the same severity policy and SLA timers apply. The source of discovery does not change the policy — only the Discovery/Verification Date and the severity determine the response.

Findings discovered after the release is promoted to production are handled by the monthly production scans in Section 9.

---

### **3. Roles & Responsibilities (RACI Matrix)**

|  |  |  |  |  |
| --- | --- | --- | --- | --- |
| **Phase** | **Developer** | **RM/Component  Lead/PO** | **Security PO** | **Pen Tester** |
| Branch Commits | R/A | I | I | - |
| Merge Requests | R | A | C | - |
| Release Pentesting | I | I | C/A | R/A |
| Triage & Fixes | R | C | A | - |
| Final Release Sign-off | I | R | R | - |

> **R = Responsible | A = Accountable | C = Consulted | I = Informed**

- The Security PO/Architect is accountable for the final release sign-off decision.

---

### **4. Severity Classification & Fix Policy**

For CI/CD identified vulnerabilites. Severity is verified by the Security Architect/Component Lead based on Common Vulnerability Scoring System version 3 (CVSS v3) vectors. Trivy's severity label alone is not sufficient for prioritization — the CVSS score and attack vector must be verified for every Critical and High finding, and for any Low or Medium finding whose title mentions Remote Code Execution (RCE), arbitrary code execution, privilege escalation, or authentication bypass. See the \[Vulnerability Scanning Guide — [Vulnerability Scanning in CX CI/CD Pipeline: Configuration, Trivy Output, and CVSS Score Interpretation](https://expertflow-docs.atlassian.net/wiki/spaces/EF/pages/1945894915/Vulnerability+Scanning+in+CX+CI+CD+Pipeline+Configuration+Trivy+Output+and+CVSS+Score+Interpretation) \] for how to look these up.

**P1 — Immediate Fix**

**Criteria**: Any Critical or High finding where the attack vector is Network (`AV:N` — exploitable remotely over the internet). Additionally, any finding at any severity level where the exploit type is Remote Code Execution (RCE — attacker can run arbitrary commands on the target system), arbitrary code execution, privilege escalation (attacker gains higher system permissions than intended), or authentication bypass (attacker circumvents login or identity verification) — regardless of Trivy severity label or attack vector. Exploit type takes precedence over attack vector when the two criteria conflict.

**Enforcement**: Pipeline blocks MR for CI/CD findings. Immediate hotfix branch required for pentest findings, following the naming convention in CI/CD Guidelines: `{BASE_RELEASE_TAG}_b-{JIRA_ID}`.

**Fix deadline**: Within 7 days of Discovery/Verification Date.

**P2 — Fix Before Release**

**Criteria**: Critical or High finding where the attack vector is Local (`AV:L` — attacker needs an existing local session on the machine) or Adjacent (`AV:A` — attacker must be on the same local network or Virtual Private Network (VPN) segment) AND the exploit type is not RCE, arbitrary code execution, privilege escalation, or authentication bypass. If the exploit type qualifies for P1, it is P1 regardless of attack vector.

**Enforcement**: Hard release gate. Release cannot promote to production until resolved or a formal Risk Acceptance is signed by the Security PO.

**Fix deadline**: Within 7 days of Discovery/Verification Date.

**P3 — Fix in Next Minor Release**

**Criteria**: Medium findings with an available upstream fix.

**Fix deadline**: Must be resolved before the next minor release cycle closes.

**P4 — Scheduled Cleanup**

**Criteria**: Low findings, or any severity with no upstream fix available. P4 findings are batched and resolved during the next scheduled base image update cycle, where the team upgrades the underlying container base image (e.g., Alpine, Ubuntu) and rebuilds all affected components. The Security Architect is responsible for scheduling this cycle and confirming via Trivy re-scan that all batched P4 findings are cleared.

Any Low finding whose title mentions RCE, arbitrary code execution, or privilege escalation must have its CVSS vector verified by the Security Architect before P4 treatment is accepted.

**False Positives and No-Fix-Available Exceptions**

For any P1 or P2 finding that cannot be remediated due to a confirmed false positive or no available upstream fix, the following must be documented in the Jira epic and the Vulnerability Report before the Security PO approves an exception:

- CVE ID or pentest finding ID and affected component
- CVSS base score and full attack vector string
- Reason for exception (false positive evidence or upstream fix status with reference link)
- Compensating control in place describing how the risk is mitigated without a code fix
- Re-evaluation date set to the next release cycle

No exception may be approved verbally. Written documentation in Jira is mandatory for audit purposes.

---

### **5. **SLA Clocks & Release Gates

**CRITICAL: **

**All SLA timers start from the Discovery/Verification Date** — the date the vulnerability is confirmed and the Jira bug is created, not the assignment date. This ensures compliance with ISO 27001 audit standards.

**Fix Deadlines by Priority:**

| Priority | Fix Deadline from Discovery Date |
| --- | --- |
| P1 | 7 days |
| P2 | 7 days |
| P3 | Before next minor release cycle closes |
| P4 | P4 findings are batched and resolved during the next scheduled base image update cycle, where the team upgrades the underlying container base image (e.g., Alpine, Ubuntu) and rebuilds all affected components. The Security Architect is responsible for scheduling this cycle and confirming via Trivy re-scan that all batched P4 findings are cleared |

**Release Gate:** Zero open P1 and P2 findings at release. The release does not ship if any P1 or P2 finding from either Track 1 or Track 2 remains open and unexcepted.

**Escalation Path:**

- **Day 8 with open P1/P2:** Security PO escalates to Component Lead/PO and Release Manager.
- **Day 12 with open P1/P2:** Security PO escalates to Higher Management. Release date is held until the finding is resolved or a formal written Risk Acceptance is approved by CTO.

---

### 6. Triage and Remediation Workflow

**Track 1 — CI/CD Pipeline Findings**

CI/CD pipeline findings are generated continuously on every commit and Merge Request (MR). The developer is the first to see branch scan results. MR scan failures block the merge automatically.

**P1 and P2 (Critical/High):**

- **At MR scan failure:** Pipeline blocks the merge. The developer reviews the Trivy/Grype output, identifies the affected package and fixed version, updates the base image or dependency, and re-runs the pipeline to confirm the fix before the MR can be merged.
- **At release triage (before Release Candidate):** Security Architect reviews all open CI/CD findings across components, verifies CVSS vectors via the Aqua Vulnerability Database (AVD) link or Trivy JavaScript Object Notation (JSON) output, deduplicates by CVE ID, and confirms P1/P2 classification. Discovery/Verification Date is set at this point for any finding not already resolved.
- **Within 24 hours of Discovery/Verification Date:** Security PO assigns any unresolved Jira bugs to relevant teams.
- **Within 7 days of Discovery/Verification Date:** Developer creates a hotfix branch (e.g., `hotfix/CX5.0-critical-vul-1`), updates the affected package to the fixed version listed in Trivy output, and confirms the fix via pipeline re-scan before merging.

**P3 and P4 (Medium/Low):**

- **Within 8 days of Discovery/Verification Date:** Security Architect and Security PO review and triage all Medium and Low findings. Any Low with an RCE-type (Remote Code Execution) exploit description is verified against its CVSS vector before P4 assignment.
- **Within 9 days:** Security PO assigns Jira bugs to relevant developers.
- **Within 15 days:** Developer creates a hotfix branch (e.g., `hotfix/CX5.0-major-vul-1`), updates the affected package, and confirms via pipeline re-scan before merging. P4 findings are batched into the next planned base image refresh with no individual branch required.

**Track 2 — Pentest Findings**

Pentest findings are generated during the Release Candidate (RC) phase by the Security Architect. They enter the system through a Jira epic and individual Jira bugs, not through the pipeline.

**P1 and P2 (Critical/High):**

- **Within 24 hours of Discovery/Verification Date:** Security Architect reviews all Critical and High pentest findings, verifies CVSS vectors, and confirms P1 or P2 classification per Section 4.
- **Within 24 hours:** Security PO assigns Jira bugs to relevant teams.
- **Within 7 days:** Developer creates a hotfix branch (e.g., `hotfix/CX5.0-pentest-critical-1`), implements the fix, and the Security Architect performs a targeted re-test to confirm the vulnerability is resolved before the branch is merged.

**P3 and P4 (Medium/Low):**

- **Within 8 days of Discovery/Verification Date:** Security Architect and Security PO review and triage all Medium and Low pentest findings. Any Low with an RCE-type (Remote Code Execution) exploit description is verified against its CVSS vector before P4 assignment.
- **Within 9 days:** Security PO assigns Jira bugs to relevant developers.
- **Within 15 days:** Developer implements the fix and the Security Architect re-tests to confirm resolution. P4 findings are batched and addressed in the next release cycle.

### 7. Vulnerability Reporting (Audit Artifact)

At each release the Security PO generates the Consolidated Vulnerability Report and attaches it to the release notes page. The report must include:

- Per-component CI/CD scan results deduplicated by CVE ID (not raw Trivy row counts).
- Summary of pentest findings by severity with Jira epic reference.
- For any open P1 or P2 at release time: CVE ID or pentest finding ID, CVSS base score, full attack vector string, affected component, exception justification, compensating control, and re-evaluation date.
- A release-level summary statement confirming whether the zero P1/P2 gate is met or listing all approved exceptions.

The report is an internal document and is not published externally. Customers under NDA may request full CVE-level detail. Historical reports are retained per release for SOC 2 and ISO 27001 audit continuity.

---

### 8. Maintenance Release Delivery

The RMT plans and delivers the maintenance release with the Product Owner following the Release Process: [Release Process Workflow](https://expertflow-docs.atlassian.net/wiki/spaces/EF/pages/458620991/Release+Process+Workflow)

P1 and P2 vulnerabilities from both tracks are addressed in the priority patch. A subsequent patch is planned for P3 findings if not resolved in the main release cycle.

The RM team updates the maintenance release notes with a summary of fixed vulnerabilities referencing CVE IDs and pentest finding IDs where applicable. Full raw scan output and pentest reports remain internal to Expertflow.

---

### 9. Post-Release & Maintenance

- Notify users via email about maintenance releases.
- Update the Vulnerability Report to reflect remediated items and close associated Jira bugs.
- Re-evaluate all open deferred P3 and P4 items and carry them forward to the next release cycle Jira epic if unresolved.
- **Monthly Production Scans:** The Security Architect runs Trivy scans against all production images on the first working day of each month to catch newly published CVEs against already-deployed versions. Findings are reviewed by the Security PO, ticketed in the active release cycle Jira epic, and triaged per the severity policy in Section 4. The Security Architect is accountable for ensuring scans are executed and results are reviewed each month.

---

This process ensures:

✅ Shift-Left: Developers catch vulnerabilities in branch scans before the team sees them.   
✅ Gatekeeping: Team Leads enforce the MR security gate before merging. Pipeline blocks Critical and High findings with available fixes.   
✅ Dual-Track Coverage: Automated scanning covers package-level risks; manual pentesting covers logic and business-layer risks.   
✅ CVSS-Informed Prioritization: Severity labels are verified against CVSS vectors before fix priority is assigned.   
✅ Compliance: SLA clocks start at discovery; exceptions require written compensating controls approved by the Security PO.   
✅ Risk Accountability: VP Engineering provides final accountability for any unpatched P1/P2 risks held open beyond Day 10.   
✅ Audit-Ready: Reports archived per release for SOC 2 and ISO 27001 continuity.
