# INSTRUCTOR HANDBOOK — MODULE 7
## Secure SDLC and DevSecOps: Shift Left, Threat Awareness, Security Gates

---

### Module Metadata

| Field          | Value                                                       |
|----------------|-------------------------------------------------------------|
| Module Number  | 7 of 16                                                     |
| Day            | Day 2 — Opening Module                                      |
| Duration       | 75 minutes (55 min content + 20 min mapping exercise)      |
| Difficulty     | Intermediate — conceptual framing for all Day 2 labs       |
| Prerequisites  | Day 1 complete; participants have a working CI/CD pipeline  |
| Position       | Gateway module — everything in Day 2 builds on this        |

---

## Instructor Preparation Notes

### The Critical Role of This Module
Module 7 is the **conceptual pivot** of the entire workshop. Day 1 built the delivery machinery. Day 2 arms that machinery with security capabilities. This module answers the question: *"How does security actually fit into what we built yesterday?"*

Do not rush it. Participants who leave Module 7 without a clear mental model of where each scan type sits in the pipeline will spend Day 2 confused about why a given scanner runs when it does, what it can and cannot detect, and how findings reach the developer.

### Opening Reframe (First 5 Minutes)
Start Day 2 with this provocation:

> *"Yesterday we built a pipeline that deploys code faster than most organisations have ever shipped. Congratulations — you just built a faster way to deliver vulnerable software to production. Today we fix that."*

Then show this contrast:

```
Without DevSecOps:    Fast pipeline → Fast delivery of vulnerable code
With DevSecOps:       Fast pipeline → Fast feedback on vulnerabilities → Fast fix → Safe delivery
```

The goal is not to slow down the pipeline. The goal is to add **signal** — specific, actionable, early security signal — without adding meaningful latency.

### Audience Calibration

**AI Engineers:** Model files are code artefacts. Model training pipelines are CI/CD pipelines. SAST does not scan model weights, but it scans the Python training code, the inference server code, and the data preprocessing scripts. Secret detection catches API keys in Jupyter notebooks committed to the repo (extremely common). Dependency scanning catches vulnerable ML libraries (PyTorch, TensorFlow, scikit-learn have had CVEs).

**Robotics/Embedded Engineers:** The "production" for them is a deployed robot or device — potentially with humans nearby. The cost model for late vulnerability detection is completely different. A CVE in a robot controller's RPC interface could allow remote command injection on a physical system. Threat modelling for embedded systems must include physical access vectors, communication protocol attacks, and firmware tampering.

**IoT Developers:** OTA update mechanisms are one of the highest-risk attack surfaces in IoT. An unsigned, unverified OTA update channel is a remote code execution vector for every device in the fleet. This module establishes why supply chain security and artifact signing (Module 13) are non-negotiable for IoT.

---

## Learning Objectives

1. Map the security testing landscape (SAST, DAST, SCA, IAST, Container Scanning, IaC Scanning) to specific points in the CI/CD pipeline with rationale for each placement.
2. Explain OWASP SAMM's five business functions and use them to assess an organisation's current security posture.
3. Apply NIST SSDF's four practice groups to a software delivery organisation.
4. Define the vulnerability lifecycle from detection through triage, remediation, and verification.
5. Describe how GitLab Ultimate implements security gates — the difference between a warning and a blocking gate.
6. Conduct a basic threat awareness exercise using the STRIDE model.

---

## Business Context

### The Security Debt Epidemic

According to the Veracode State of Software Security report (2023), **76% of applications have at least one security flaw**. The median time to fix a high-severity finding is 290 days. That gap — between discovery and fix — is the attack window.

The question is not whether organisations have vulnerabilities. They do. The question is:
- How long does it take to *discover* a vulnerability after it is introduced?
- How long does it take to *fix* it once discovered?
- How much *context* does the developer have when they receive the finding?

DevSecOps does not eliminate vulnerabilities. It **collapses the detection and feedback loop** to minutes, while the developer still has full context of the code they just wrote.

**For Regulated Industries (Automotive, Medical, Defence):**
IEC 62443 (Industrial Control Systems), ISO 21434 (Automotive Cybersecurity), IEC 62443-4-1 (Secure Product Development), and FDA guidance on medical device cybersecurity all require documented security testing integrated into the development process. GitLab's pipeline-based scanning with audit trails directly satisfies the "documented, repeatable security testing" requirement these frameworks mandate.

---

## Module Content — Detailed Teaching Guide

---

### SECTION 1: The Security Testing Landscape — Where Each Scan Lives (15 minutes)

#### The AST Family
Application Security Testing encompasses several distinct approaches. Each has a different position in the pipeline, different detection capabilities, and different false-positive profiles.

**Draw this on the whiteboard — the pipeline with scan placement:**

```
CODE              BUILD              RUNNING APP        DEPLOYED
  │                 │                    │                  │
[SAST]          [SCA/Deps]          [DAST]             [IAST]
[Secrets]       [Container]         [Fuzz]             [RASP]
[IaC]           [License]
  │                 │                    │                  │
Commit-time    Build-time          QA-deploy          Production
  ↑                ↑                    ↑                  ↑
Cheapest        Cheap              Moderate            Expensive
to fix          to fix             to fix              to fix
```

#### Scan Type Comparison

| Scan Type | What It Analyses | When It Runs | What It Finds | Key Limitation |
|---|---|---|---|---|
| **SAST** | Source code (static, not executed) | Every commit | Injection, hardcoded secrets, unsafe functions, logic flaws | Cannot find runtime-only issues; false positives from context-unaware rules |
| **Secret Detection** | All files, git history | Every commit | API keys, tokens, passwords, private keys committed to repo | Only finds secrets that match known patterns; novel secret formats missed |
| **SCA / Dependency Scanning** | `package.json`, `requirements.txt`, `pom.xml`, lock files | Every build | Known CVEs in third-party dependencies | Only finds *known* CVEs; zero-days not detected |
| **Container Scanning** | Docker image layers | Every image build | OS package CVEs, outdated base images | Layer-by-layer scan; application-layer logic flaws not detected |
| **IaC Scanning** | Terraform, K8s manifests, CloudFormation | Every IaC change | Misconfigurations, insecure defaults, open ports, missing encryption | Config drift between IaC and actual deployed state not detected |
| **DAST** | Running application (black box) | Post-deploy to QA | Injection vulnerabilities, auth flaws, exposed endpoints, runtime behaviour | Needs running app; cannot find every code path; slower |
| **IAST** | Running application (with instrumentation) | Runtime in test env | Real runtime behaviour with code path visibility | Requires instrumented agent; complex setup |

**Source:** GitLab Application Security overview — https://docs.gitlab.com/ee/user/application_security/

#### Teaching Point: Complementary, Not Competing
> "Every scan type in this table finds things the others cannot. SAST finds a SQL injection pattern in code that is never called in tests — DAST never would. DAST finds a SQL injection in a dynamically constructed query that SAST's pattern matching missed. They are complementary. The maturity model is not: pick one. The maturity model is: add them progressively, starting with the fastest feedback and lowest configuration overhead — SAST and Secret Detection."

#### GitLab Security Scanner Technology Stack (Validated)

GitLab uses these scanner engines per capability:

| GitLab Capability | Underlying Technology | Source |
|---|---|---|
| SAST | Semgrep (primary), + language-specific analyzers | https://docs.gitlab.com/ee/user/application_security/sast/analyzers.html |
| Secret Detection | Gitleaks | https://docs.gitlab.com/ee/user/application_security/secret_detection/ |
| Dependency Scanning | Trivy, Gemnasium | https://docs.gitlab.com/ee/user/application_security/dependency_scanning/ |
| Container Scanning | Trivy | https://docs.gitlab.com/ee/user/application_security/container_scanning/ |
| IaC Scanning | KICS (Keeping Infrastructure as Code Secure) | https://docs.gitlab.com/ee/user/application_security/iac_scanning/ |
| DAST | OWASP ZAP | https://docs.gitlab.com/ee/user/application_security/dast/ |

---

### SECTION 2: OWASP SAMM — Measuring Security Maturity (10 minutes)

**What OWASP SAMM Is:**
The Software Assurance Maturity Model (SAMM) is a framework for evaluating and improving software security posture. It is vendor-neutral and technology-agnostic.

Source: https://owaspsamm.org/model/

**The Five Business Functions:**

```
┌─────────────────────────────────────────────────────────────┐
│                    OWASP SAMM v2.0                         │
├────────────┬────────────┬────────────┬────────────┬─────────┤
│ GOVERNANCE │   DESIGN   │  IMPLEMENT │   VERIFY   │OPERATIONS│
├────────────┼────────────┼────────────┼────────────┼─────────┤
│ Strategy & │ Threat     │ Secure     │ Architect  │ Incident │
│ Metrics    │ Assessment │ Build      │ Review     │ Mgmt    │
│            │            │            │            │         │
│ Policy &   │ Security   │ Secure     │ Req-driven │ Environ  │
│ Compliance │ Requiremts │ Deployment │ Testing    │ Mgmt    │
│            │            │            │            │         │
│ Education &│ Security   │ Defect     │ Security   │ External │
│ Guidance   │ Architect  │ Management │ Testing    │ Comms   │
└────────────┴────────────┴────────────┴────────────┴─────────┘
```

**Maturity Levels per Practice:**
- **Level 0:** Activity not performed
- **Level 1:** Ad-hoc, largely undocumented
- **Level 2:** Defined, documented, consistently performed
- **Level 3:** Optimised, measured, continuously improving

**GitLab Ultimate Coverage Against SAMM:**

| SAMM Practice | GitLab Capability |
|---|---|
| Secure Build (Implement) | SAST, Secret Detection, SCA, Container Scan, IaC Scan in CI pipeline |
| Security Testing (Verify) | DAST, Security Dashboard, MR security findings widget |
| Defect Management (Implement) | Vulnerability Management, Issue creation from findings |
| Architecture Review (Verify) | Security Policies, compliance frameworks |
| Incident Management (Operations) | Audit Events, Vulnerability tracking |
| Policy & Compliance (Governance) | Compliance Frameworks, Protected Environments, Approval Rules |

**How to Use SAMM in This Workshop:**
After the exercise at end of this module, participants map their organisation to SAMM levels. This becomes their **baseline measurement** that they return to in Module 16 (DORA + Maturity) to define a roadmap.

---

### SECTION 3: NIST SSDF — The Government Framework (8 minutes)

**What the SSDF Is:**
NIST Special Publication 800-218, the Secure Software Development Framework (SSDF), published February 2022. It is the US government's framework for secure software development, increasingly referenced in executive orders and federal procurement requirements.

Source: https://csrc.nist.gov/Projects/ssdf

**Why It Matters to This Audience:**
- US Federal government contractors must comply (Executive Order 14028, 2021)
- European NIS2 Directive aligns with similar principles
- Defence, healthcare, critical infrastructure — all reference NIST frameworks
- Increasingly cited in insurance requirements for cyber liability coverage

**The Four Practice Groups:**

```
PO — Prepare the Organisation
  PO.1   Define security requirements for software development
  PO.2   Implement roles and responsibilities
  PO.3   Implement tooling and build environment security
  PO.4   Define and use criteria for software security checks
  PO.5   Implement and maintain secure environments for development

PS — Protect the Software
  PS.1   Protect source code from unauthorised access
  PS.2   Protect software components from tampering
  PS.3   Archive and protect each software release

PW — Produce Well-Secured Software
  PW.1   Design software to meet security requirements
  PW.4   Reuse existing, well-secured software components
  PW.5   Create source code by adhering to secure coding practices
  PW.6   Configure the compilation and build process
  PW.7   Review and/or analyse human-readable code
  PW.8   Test executable code to identify vulnerabilities
  PW.9   Configure software to have secure settings by default

RV — Respond to Vulnerabilities
  RV.1   Identify and confirm vulnerabilities in software releases
  RV.2   Assess, prioritise, and remediate vulnerabilities
  RV.3   Analyse vulnerabilities to identify root causes
```

**GitLab Ultimate SSDF Mapping:**

| SSDF Practice | GitLab Capability |
|---|---|
| PO.3 — Secure build environment | Runner isolation, Docker executor, protected environments |
| PO.5 — Secure dev environments | GitLab project RBAC, protected branches |
| PS.1 — Protect source code | Branch protection, CODEOWNERS, audit events |
| PS.2 — Protect components from tampering | Signed artifacts, Package Registry, Container Registry |
| PW.5 — Secure coding practices | SAST, Secret Detection in MR pipeline |
| PW.7 — Code review | Merge Request + Approval Rules |
| PW.8 — Test for vulnerabilities | SAST, DAST, SCA, Container Scanning |
| RV.1 — Identify vulnerabilities | Security Dashboard, Dependency List |
| RV.2 — Assess and remediate | Vulnerability Management, Issue creation |

---

### SECTION 4: Shift Left — Mechanical Explanation (8 minutes)

**The Feedback Loop is Everything:**
Three dimensions of a security feedback loop matter:

1. **Speed** — How quickly does the developer receive the finding after introducing the vulnerability?
2. **Context** — How much code context does the developer have when receiving the finding?
3. **Specificity** — How actionable is the finding? (File + line + CWE + remediation vs. "you have vulnerabilities")

**Traditional (Right) vs. DevSecOps (Left):**

| Dimension | Traditional (Pen Test at Release) | DevSecOps (SAST at Commit) |
|---|---|---|
| Speed | Weeks to months | Seconds to minutes |
| Context | Developer is on next project | Developer just wrote the code |
| Specificity | PDF report of findings | File/line/CWE in MR widget |
| Cost to fix | 100× (production finding) | 1× (commit-time finding) |
| Developer experience | Demoralising (big report) | Actionable (specific finding) |

**The Asymmetry of Shift Left:**
> "Adding SAST to a CI pipeline takes approximately 30 minutes and costs seconds of pipeline time per commit. The alternative — a quarterly penetration test — costs $30,000–$150,000 per engagement and produces a report that arrives when developers have moved on to three other features. The economics of Shift Left are not debatable."

**What Shift Left Does NOT Mean:**
- It does not mean eliminating penetration tests (DAST and pentests are complementary)
- It does not mean developers become security experts (it means developers get expert-level tooling)
- It does not mean every finding blocks the pipeline on day one (maturity-based rollout)

---

### SECTION 5: Security Gates — Warn vs. Block (8 minutes)

**The Two Modes:**
GitLab security scanners can be configured to:
1. **Warn** — Report findings, pipeline passes, developer sees results in MR widget
2. **Block** — Pipeline fails on findings meeting threshold criteria (severity, type)

**The Maturity Progression:**

```
Week 1:  Add SAST to pipeline → allow_failure: true (warn only)
         Reason: Don't break builds on day one; establish baseline

Month 1: Review findings → fix critical/high → enable blocking for new critical/high
         Reason: Historical vulnerabilities handled; new critical vulns blocked

Month 3: Enable Secret Detection blocking → no secrets ever merge
         Reason: Zero tolerance for secrets in code

Month 6: Enable SCA blocking for critical CVEs (CVSS ≥ 9.0)
         Reason: Mature team, dependency hygiene established

Month 12: Full Security Policy enforcement
          Reason: All scan types, defined thresholds, security team approval for exceptions
```

**GitLab Implementation of Security Gates:**

*Method 1: `allow_failure` in pipeline YAML (basic)*
```yaml
sast:
  allow_failure: false  # Pipeline fails if SAST job fails
```

*Method 2: Security Policies (Ultimate — correct production approach)*
Navigate to: **Security → Policies → New policy → Scan result policy**

```yaml
# Example Security Policy (GitLab policy YAML)
type: scan_result_policy
name: Block critical vulnerabilities
description: Fail MR if critical severity vulnerability is introduced
enabled: true
rules:
  - type: scan_finding
    branches:
      - main
    scanners:
      - sast
      - dependency_scanning
      - container_scanning
    vulnerabilities_allowed: 0
    severity_levels:
      - critical
    vulnerability_states:
      - newly_detected
actions:
  - type: require_approval
    approvals_required: 1
    user_approvers_ids: []
    group_approvers_ids: [security-team]
```

Source: https://docs.gitlab.com/ee/user/application_security/policies/scan-result-policies.html

**The "Newly Detected" Distinction:**
> "The `vulnerability_states: newly_detected` setting is critical. It means: only block the MR if *this MR introduces a new vulnerability*. Existing historical vulnerabilities in the codebase do not block new MRs. This lets teams adopt blocking gates without inheriting a backlog of pre-existing findings as blockers."

---

### SECTION 6: The Vulnerability Lifecycle (6 minutes)

**GitLab Vulnerability States (Validated):**
Source: https://docs.gitlab.com/ee/user/application_security/vulnerability_report/

```
DETECTED → CONFIRMED → IN PROGRESS → RESOLVED
    ↓           ↓
DISMISSED   DISMISSED
(false positive, acceptable risk, not applicable)
```

**Each State's Meaning:**

| State | Meaning | Who Sets It | Audit Record |
|---|---|---|---|
| Detected | Scanner found it; not yet reviewed | Scanner (automatic) | Yes |
| Confirmed | Human reviewed; confirmed real issue | Security/Dev team | Yes |
| Dismissed | False positive, or accepted risk | Security/Dev team | Yes — requires justification |
| In Progress | Fix being developed | Developer | Yes |
| Resolved | Vulnerability no longer detected | Pipeline re-scan | Yes — auto-set when next scan passes |

**The Dismiss Workflow (Important):**
When a developer dismisses a finding as a false positive, GitLab requires a **dismissal reason**. This creates an auditable record: who dismissed, when, and why. This is critical for compliance — "we knowingly accepted this risk" must be documented.

**For IoT/Embedded Teams:**
The vulnerability lifecycle applies to firmware CVEs identically. A CVE in a third-party library used by an embedded application has the same triage states. The difference: remediation in embedded requires a new firmware release + OTA deployment cycle, which may take weeks. The "In Progress" duration for embedded CVEs is typically much longer than web applications. Risk acceptance tracking is therefore more important.

---

### SECTION 7: Threat Awareness — STRIDE Basics (5 minutes — discussion only, no lab)

**Why Include Threat Modelling:**
OWASP SAMM Threat Assessment practice and NIST SSDF PW.1 both require threat modelling. It is the design-time security activity that informs what you are scanning for.

**STRIDE (for contextual awareness):**

| Letter | Threat | Example for RoboVision |
|---|---|---|
| S | Spoofing | Attacker impersonates a robot controller to inject false sensor data |
| T | Tampering | Attacker modifies firmware during OTA update (unsigned update) |
| R | Repudiation | Robot executes dangerous command; operator denies issuing it (no audit log) |
| I | Information Disclosure | Telemetry stream exposes robot position to unauthorized parties |
| D | Denial of Service | Flooding the command API causes robot to stop responding to safety-stop commands |
| E | Elevation of Privilege | Command API allows unauthenticated users to send movement commands |

**Teaching Point:**
> "STRIDE is not a tool you run — it is a conversation you have during design. The output is a threat list. That threat list directly maps to: what SAST rules matter for your codebase, what DAST tests to configure, what network policies to enforce. GitLab's security scanners are most valuable when you know which threats you are scanning for."

---

## Discussion Exercise (20 minutes)

### Exercise: Security Placement Mapping

**Instructions:** In groups of 3, take your pipeline diagram from the Day 1 exercise (Module 1). Answer:

1. Mark where on your pipeline each scan type currently exists (SAST, SCA, Secret Detection, Container Scan, DAST). If it doesn't exist anywhere, mark it "missing."

2. For each missing scan type, estimate:
   - What class of vulnerability would it catch?
   - How frequently does your code introduce that class of vulnerability?

3. Using the cost-of-defect table (1× design → 100× production): estimate the annual cost saving if you detected one critical vulnerability per month at commit-time vs. post-production.

4. Identify your organisation's current OWASP SAMM maturity level for:
   - Secure Build (PW.5 equivalent)
   - Security Testing (Verify)
   - Defect Management

**Debrief Points to Surface:**
- Most teams have SAST nowhere, or in a quarterly external scan
- Dependency scanning is almost universally absent until a Log4Shell-type event
- SAMM Level 0–1 is the honest baseline for most organisations
- The gap between "where we are" and "where GitLab Ultimate can take us in 3 months" is the value conversation

---

## Failure Modes and Anti-Patterns

| Anti-Pattern | Description | Consequence |
|---|---|---|
| `allow_failure: true` on all security jobs | Every security scan result is informational only — never blocks | Green pipeline regardless of findings; false security confidence |
| Scan-and-ignore | Scans run, findings appear in dashboard, no triage process | Vulnerability count grows indefinitely; no improvement |
| Big-bang security adoption | All scans enabled with blocking on day one | Pipeline broken by thousands of historical findings; teams revolt |
| Security team owns all findings | Developers have no visibility into their own scan results | Security bottleneck; slow remediation; adversarial relationship |
| Dismissal without justification | Developers dismiss findings to make pipeline green | No audit trail; real vulnerabilities hidden as "false positives" |
| Scan without policy | Scans produce findings but no policy defines when to block | Security state undefined; each MR decides ad-hoc |
| DAST as the only security layer | Organisation relies solely on DAST; skips SAST and SCA | DAST misses code-path coverage; finds far fewer issues than SAST+SCA combined |

---

## Knowledge Check Questions

**L1 (Motivation):**
> "What specific problem does running SAST at commit-time solve that a quarterly penetration test does not?"

**L2 (Mechanism):**
> "A SAST job runs, finds a SQL injection, and the pipeline is configured with `allow_failure: false`. Walk through exactly what happens to: (a) the pipeline, (b) the MR, (c) the developer."

**L3 (Mental Model):**
> "Your security policy uses `vulnerability_states: newly_detected`. A developer opens an MR that modifies a file that already contained a known vulnerability. A new critical vulnerability is also introduced in the same MR. Which vulnerability triggers the blocking gate?"

**L4 (Mastery):**
> "Design a 6-month DevSecOps adoption plan for a team that currently has no security scanning. Define: which scans to add each month, when to switch from warn to block, and what OWASP SAMM level the team will reach by month 6."

---

## References and Citations

1. **GitLab Application Security Overview** — https://docs.gitlab.com/ee/user/application_security/
2. **GitLab SAST Analyzers** — https://docs.gitlab.com/ee/user/application_security/sast/analyzers.html
3. **GitLab Security Dashboard** — https://docs.gitlab.com/ee/user/application_security/security_dashboard/
4. **GitLab Vulnerability Report** — https://docs.gitlab.com/ee/user/application_security/vulnerability_report/
5. **GitLab Security Policies** — https://docs.gitlab.com/ee/user/application_security/policies/
6. **GitLab Scan Result Policies** — https://docs.gitlab.com/ee/user/application_security/policies/scan-result-policies.html
7. **OWASP SAMM v2.0** — https://owaspsamm.org/model/
8. **NIST SP 800-218 SSDF** — https://csrc.nist.gov/Projects/ssdf
9. **Veracode State of Software Security 2023** — https://www.veracode.com/state-of-software-security-report
10. **OWASP ZAP (DAST)** — https://www.zaproxy.org/
11. **Semgrep (GitLab SAST engine)** — https://semgrep.dev/
12. **Trivy (Container + Dependency Scanning)** — https://trivy.dev/
13. **KICS (IaC Scanning)** — https://kics.io/
14. **Gitleaks (Secret Detection)** — https://gitleaks.io/
15. **DORA Research** — https://dora.dev/research/
