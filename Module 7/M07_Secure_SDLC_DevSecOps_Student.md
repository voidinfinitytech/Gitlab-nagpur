# STUDENT HANDBOOK — MODULE 7
## Secure SDLC and DevSecOps: Shift Left, Threat Awareness, Security Gates

---

**Workshop:** GitLab Ultimate for Secure SDLC, DevSecOps, AI, Robotics & IoT
**Day:** 2 of 2 | **Module:** 7 of 16 | **Duration:** 75 minutes

---

## Learning Objectives

- Map SAST, DAST, SCA, Container Scanning, IaC Scanning, and Secret Detection to pipeline positions
- Apply OWASP SAMM to assess your organisation's security maturity
- Map NIST SSDF practices to GitLab Ultimate capabilities
- Explain the vulnerability lifecycle from detection through resolution
- Configure security gates: warn-only vs. blocking thresholds
- Conduct a basic STRIDE threat awareness exercise

---

## 1. The Day 2 Reframe

Yesterday you built a pipeline that deploys software faster than most organisations. Today's question:

> **"How do we ensure that what we ship fast, we ship securely?"**

The answer is not to slow the pipeline. It is to embed security signal — specific, actionable, developer-visible — at every stage of the pipeline you already built.

```
WITHOUT DevSecOps:
  Fast pipeline → Fast delivery of vulnerable software

WITH DevSecOps:
  Fast pipeline + Security signal → Fast feedback → Fast fix → Safe delivery
```

The pipeline from Day 1 is your chassis. Today you install the security systems.

---

## 2. The Security Testing Landscape

### Scan Type Reference

Each scan type occupies a specific position in the pipeline based on what it needs to analyse:

| Scan Type | Analyses | Pipeline Position | Finds | Key Limitation |
|---|---|---|---|---|
| **SAST** | Source code (not executed) | Every commit | Injection, hardcoded secrets, unsafe functions | Cannot find runtime-only issues |
| **Secret Detection** | All files + git history | Every commit | API keys, tokens, credentials in code | Only known secret patterns |
| **SCA / Dependency Scanning** | Dependency manifests (`requirements.txt`, `package.json`, etc.) | Every build | Known CVEs in third-party libraries | Only known CVEs (not zero-days) |
| **Container Scanning** | Docker image layers | Every image build | OS package CVEs, outdated base images | Not application-logic flaws |
| **IaC Scanning** | Terraform, K8s manifests | Every IaC change | Misconfigurations, open ports, missing encryption | Not runtime configuration drift |
| **DAST** | Running application (black box) | Post-deploy to test env | Runtime injection, auth flaws, exposed endpoints | Needs running app; slower |

### Pipeline Scan Placement

```
COMMIT TIME                    BUILD TIME                  RUNTIME
     │                              │                          │
  ┌──┴──────────────────────┐   ┌──┴──────────────────┐   ┌──┴──────────┐
  │ SAST                    │   │ Dependency Scanning  │   │ DAST        │
  │ Secret Detection        │   │ Container Scanning   │   │ (QA env)    │
  │ IaC Scanning            │   │ License Compliance   │   │             │
  └─────────────────────────┘   └─────────────────────┘   └─────────────┘
     │                              │                          │
  Seconds                       Seconds–minutes             Minutes
  Developer has                 Still in build               Post-deploy
  full code context             context                      context fading
     │                              │                          │
  Cheapest to fix               Cheap to fix                Moderate to fix
  (1×)                          (6×)                        (15×)
```

### GitLab Scanner Technology Stack

GitLab Ultimate uses these open-source engines under the hood:

| GitLab Feature | Engine | Reference |
|---|---|---|
| SAST | Semgrep + language-specific analyzers | https://docs.gitlab.com/ee/user/application_security/sast/analyzers.html |
| Secret Detection | Gitleaks | https://docs.gitlab.com/ee/user/application_security/secret_detection/ |
| Dependency Scanning | Trivy, Gemnasium | https://docs.gitlab.com/ee/user/application_security/dependency_scanning/ |
| Container Scanning | Trivy | https://docs.gitlab.com/ee/user/application_security/container_scanning/ |
| IaC Scanning | KICS | https://docs.gitlab.com/ee/user/application_security/iac_scanning/ |
| DAST | OWASP ZAP | https://docs.gitlab.com/ee/user/application_security/dast/ |

> **Key insight:** GitLab does not build proprietary scanners from scratch. It integrates, orchestrates, and presents results from industry-standard open-source tools — in a unified pipeline with a single dashboard and developer feedback loop.

---

## 3. OWASP SAMM — Measuring Security Maturity

### What SAMM Is

OWASP Software Assurance Maturity Model (SAMM) v2.0 is a vendor-neutral framework for evaluating and improving software security programmes.

**Reference:** https://owaspsamm.org/model/

### The Five Business Functions

```
┌──────────────┬────────────┬─────────────┬────────────┬─────────────┐
│  GOVERNANCE  │   DESIGN   │  IMPLEMENT  │   VERIFY   │  OPERATIONS │
├──────────────┼────────────┼─────────────┼────────────┼─────────────┤
│ Strategy &   │ Threat     │ Secure      │ Architect  │ Incident    │
│ Metrics      │ Assessment │ Build       │ Review     │ Management  │
│              │            │             │            │             │
│ Policy &     │ Security   │ Secure      │ Req-Driven │ Environment │
│ Compliance   │ Requiremts │ Deployment  │ Testing    │ Management  │
│              │            │             │            │             │
│ Education &  │ Security   │ Defect      │ Security   │ External    │
│ Guidance     │ Architect  │ Management  │ Testing    │ Comms       │
└──────────────┴────────────┴─────────────┴────────────┴─────────────┘
```

### Maturity Levels

| Level | Description |
|---|---|
| 0 | Activity not performed |
| 1 | Ad-hoc, largely undocumented, inconsistent |
| 2 | Defined, documented, consistently performed |
| 3 | Measured, optimised, continuously improving |

### GitLab Ultimate Mapped to SAMM

| SAMM Practice | SAMM Level Enabled | GitLab Capability |
|---|---|---|
| Secure Build | Level 2 | SAST, SCA, Secret Detection, Container Scan in pipeline |
| Security Testing | Level 2 | DAST, Security Dashboard |
| Defect Management | Level 2 | Vulnerability management, issue creation from findings |
| Policy & Compliance | Level 2–3 | Security Policies, Compliance Frameworks, Audit Events |
| Architecture Review | Level 1 | MR review process with CODEOWNERS for security files |
| Threat Assessment | Level 1 | Requires separate threat modelling tool/process |
| Incident Management | Level 1 | Audit events, but external SIEM integration needed for Level 2+ |

---

## 4. NIST SSDF — The Secure Software Development Framework

### What It Is

NIST SP 800-218 (February 2022) defines practices for secure software development. Referenced by US Executive Order 14028 (2021), increasingly cited in defence, healthcare, and regulated industry procurement requirements.

**Reference:** https://csrc.nist.gov/Projects/ssdf

### The Four Practice Groups

| Group | Code | Focus |
|---|---|---|
| Prepare the Organisation | PO | Roles, tools, environment security, criteria |
| Protect the Software | PS | Source code protection, component integrity, release archiving |
| Produce Well-Secured Software | PW | Secure design, coding, build, code review, vulnerability testing |
| Respond to Vulnerabilities | RV | Identify, assess, remediate, root-cause analysis |

### SSDF → GitLab Mapping

| SSDF Practice | What It Requires | GitLab Implementation |
|---|---|---|
| PO.3 — Secure tooling | Build environment must be secure | Runner isolation (Docker executor), separate security Runners |
| PO.5 — Secure dev environments | Dev environments protected | Project RBAC, protected branches, MR requirements |
| PS.1 — Protect source code | Unauthorised access prevented | Branch protection, CODEOWNERS, access controls |
| PS.2 — Protect components | Software components from tampering | Signed artifacts, Package Registry with checksums |
| PW.5 — Secure coding | Adhere to secure coding practices | SAST in pipeline provides automated coding standard enforcement |
| PW.7 — Code review | Human code review process | Merge Request + Approval Rules + CODEOWNERS |
| PW.8 — Test for vulnerabilities | Executable code tested | SAST, DAST, SCA, Container Scanning in pipeline |
| RV.1 — Identify vulnerabilities | Identify and confirm in releases | Security Dashboard, Dependency List, vulnerability report |
| RV.2 — Assess and remediate | Prioritise and fix | Vulnerability Management with severity, triage, issue creation |

---

## 5. Shift Left — The Economics

### Why "Shift Left" Is Not Optional

Cost to fix a defect by discovery phase (IBM Systems Sciences Institute):

| Phase Detected | Relative Cost | Typical Feedback Time |
|---|---|---|
| Design | 1× | Design review meeting |
| Development (SAST at commit) | 6× | Seconds — MR security widget |
| Testing (DAST at QA) | 15× | Minutes — pipeline result |
| Production (penetration test) | 100× | Weeks–months after introduction |

**Example calculation:**
An organisation introduces 2 critical vulnerabilities per month. If detected at commit-time vs. production:

```
Commit-time:     2 × 1 hour fix  =  2 hours/month
Production:      2 × 100 hours   =  200 hours/month  (+ breach risk + compliance impact)

Annual difference:  ~2,376 hours of engineering time
                    + elimination of breach risk from each undetected vulnerability
```

### What Shift Left Requires From Developers

Shift Left only works if:
1. **Findings reach developers immediately** — not in a weekly email or dashboard nobody checks
2. **Findings are specific** — file, line number, CWE ID, remediation guidance
3. **False positives are manageable** — tuned rulesets, not 500 findings per commit
4. **Developers have context to fix** — they just wrote the code

GitLab delivers all four via the **Merge Request security widget** — findings appear in the same view the developer uses for code review.

---

## 6. Security Gates: Warn vs. Block

### The Two Modes

| Mode | Pipeline Behaviour | When to Use |
|---|---|---|
| **Warn** (`allow_failure: true`) | Findings reported; pipeline passes | Starting out; establishing baseline; historical debt present |
| **Block** (`allow_failure: false` or Security Policy) | Pipeline fails on threshold breach | Mature team; historical findings addressed; zero tolerance for specific severity |

### GitLab Security Policy (Production Approach)

Security Policies (GitLab Ultimate) decouple gate configuration from pipeline YAML:

```yaml
# Security Policy (defined in Security → Policies, not .gitlab-ci.yml)
type: scan_result_policy
name: "Block new critical vulnerabilities"
rules:
  - type: scan_finding
    branches: [main]
    scanners: [sast, dependency_scanning, container_scanning]
    severity_levels: [critical]
    vulnerability_states: [newly_detected]   # ← Only NEW findings trigger the gate
    vulnerabilities_allowed: 0
actions:
  - type: require_approval
    approvals_required: 1
    group_approvers_ids: [security-team]
```

Source: https://docs.gitlab.com/ee/user/application_security/policies/scan-result-policies.html

**The `newly_detected` Key:**
This setting blocks MRs that *introduce* critical vulnerabilities, but does NOT block MRs touching code that already has historical vulnerabilities. This lets teams adopt security gates without inheriting the entire technical debt backlog as a blocker.

### Recommended Adoption Sequence

```
Month 1:  Add SAST + Secret Detection → warn only → build baseline
Month 2:  Fix critical/high SAST findings → enable blocking for new critical SAST
Month 3:  Add Secret Detection blocking → zero tolerance for secrets
Month 4:  Add SCA → warn only → fix critical CVEs
Month 5:  Enable SCA blocking (CVSS ≥ 9.0)
Month 6:  Add Container Scanning → warn → fix critical base image CVEs
Month 12: Full Security Policy coverage, security team approval for exceptions
```

---

## 7. The Vulnerability Lifecycle

### States in GitLab

```
                ┌──────────────────────────────────────┐
                │  Scanner detects vulnerability       │
                └──────────────────┬───────────────────┘
                                   ↓
                            ┌──────────────┐
                            │  DETECTED    │  (Automatic)
                            └──────┬───────┘
                                   ↓ Security team reviews
               ┌───────────────────┴──────────────────────┐
               ↓                                          ↓
      ┌────────────────┐                     ┌────────────────────┐
      │   CONFIRMED    │                     │    DISMISSED       │
      │ (real issue)   │                     │ (false positive /  │
      └───────┬────────┘                     │  accepted risk)    │
              ↓                              └────────────────────┘
      ┌───────────────────┐                  Requires justification
      │   IN PROGRESS     │                  — audit trail created
      │ (fix in progress) │
      └───────┬───────────┘
              ↓ Next scan no longer detects it
      ┌───────────────────┐
      │    RESOLVED       │  (Automatic on next clean scan)
      └───────────────────┘
```

Source: https://docs.gitlab.com/ee/user/application_security/vulnerability_report/

### The Dismiss Audit Trail

Every dismissal in GitLab requires a reason:
- False positive
- Acceptable risk
- No longer applicable
- Used in tests only

The dismissal, reason, dismissing user, and timestamp are all recorded in the audit log. This satisfies compliance requirements for documented risk acceptance.

---

## 8. Threat Awareness — STRIDE

### Why Threat Modelling Matters Before Scanning

Security scanners find *what you have*. Threat modelling identifies *what you need to look for*. The two are complementary: threat modelling informs which SAST rules to prioritise, which DAST tests to configure, and which architecture decisions to revisit.

### STRIDE Quick Reference

| Threat | Description | RoboVision Example |
|---|---|---|
| **S**poofing | Claiming a false identity | Fake robot controller injecting sensor data |
| **T**ampering | Modifying data or code | Unsigned OTA firmware replaced by attacker |
| **R**epudiation | Denying an action occurred | No audit log for robot command execution |
| **I**nformation Disclosure | Leaking sensitive data | Telemetry stream exposed on open port |
| **D**enial of Service | Making service unavailable | Command API flooded; emergency stop delayed |
| **E**levation of Privilege | Gaining unauthorised access level | Unauthenticated API access to admin commands |

---

## 9. Discussion Exercise — Security Placement Mapping

In groups of 3, using your pipeline from the Day 1 exercise:

1. **Map current scans** — mark where (if anywhere) each scan type currently exists in your pipeline
2. **Identify gaps** — list scan types completely absent
3. **Prioritise** — which single scan type would have the highest impact if added today?
4. **SAMM baseline** — rate your organisation on:
   - Secure Build: Level 0 / 1 / 2 / 3
   - Security Testing: Level 0 / 1 / 2 / 3
   - Defect Management: Level 0 / 1 / 2 / 3

---

## Key Terminology

| Term | Definition |
|---|---|
| SAST | Static Application Security Testing — scan source code without executing |
| DAST | Dynamic Application Security Testing — test running application |
| SCA | Software Composition Analysis — scan dependencies for known CVEs |
| IaC Scanning | Infrastructure as Code scanning — scan Terraform/K8s manifests |
| Secret Detection | Scan repository for committed credentials/tokens |
| OWASP SAMM | Software Assurance Maturity Model — framework for measuring security programme maturity |
| NIST SSDF | Secure Software Development Framework — NIST SP 800-218 |
| Security Gate | Pipeline control point that can warn or block based on security findings |
| Vulnerability State | Detection → Confirmed → In Progress → Resolved (or Dismissed) |
| STRIDE | Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege |
| Newly Detected | GitLab vulnerability state: introduced by this specific MR (not pre-existing) |
| CVE | Common Vulnerabilities and Exposures — unique identifier for known vulnerabilities |
| CVSS | Common Vulnerability Scoring System — 0–10 severity scale |

---

## Personal Notes

_Current organisation SAMM levels:_
- Secure Build: ______ / 3
- Security Testing: ______ / 3
- Defect Management: ______ / 3

_Single highest-impact scan to add:_ _______________

_Biggest barrier to adoption:_ _______________

---

## References

1. GitLab Application Security — https://docs.gitlab.com/ee/user/application_security/
2. GitLab Security Dashboard — https://docs.gitlab.com/ee/user/application_security/security_dashboard/
3. GitLab Vulnerability Report — https://docs.gitlab.com/ee/user/application_security/vulnerability_report/
4. GitLab Security Policies — https://docs.gitlab.com/ee/user/application_security/policies/
5. OWASP SAMM v2.0 — https://owaspsamm.org/model/
6. NIST SSDF SP 800-218 — https://csrc.nist.gov/Projects/ssdf
7. GitLab SAST Analyzers — https://docs.gitlab.com/ee/user/application_security/sast/analyzers.html
