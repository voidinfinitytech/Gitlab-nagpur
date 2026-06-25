# SLIDE-BY-SLIDE TEACHING NOTES — MODULE 7
## Secure SDLC and DevSecOps

---

> **Position note:** This is the first module of Day 2. The room energy may be lower than Day 1 morning. Use the opening provocation (Slide 2) to re-engage immediately.

---

### SLIDE 1 — Day 2 Opening / Module Title

**Show:** "Day 2: DevSecOps with GitLab Ultimate"

**Say:**
> "Yesterday we built the machinery — a pipeline that takes code from commit to production with governed review, automated testing, and multi-environment promotion. Today we answer the question that security engineers have been sitting on since Module 1: how do you make sure everything moving through that pipeline is secure? Let's start with a challenge."

**Timing:** 1 minute

---

### SLIDE 2 — The Day 2 Reframe (Opening Provocation)

**Show:**
```
Without DevSecOps:
    Fast pipeline → Fast delivery of VULNERABLE software

With DevSecOps:
    Fast pipeline + Security signal → Fast feedback → Fast fix → Safe delivery
```

**Say:**
> "Everything we built yesterday — the four-stage pipeline, the manual production gate, the environment promotion — that entire machine is currently a fast way to deliver vulnerable software. We have automated deployment. We have NOT automated security. Today we fix that. And critically: we fix it without slowing down the pipeline. Security signal should add seconds to pipeline time — not hours."

**Ask the room:**
> "Before I tell you what the answer looks like — what does your current organisation do between 'developer writes code' and 'code has been checked for security issues'? What is that process today?"

Listen for: "We have a quarterly pentest." "Our CISO runs a scan before major releases." "We don't really have a process." All of these are the motivation for Day 2.

**Timing:** 5 minutes

---

### SLIDE 3 — The Security Testing Landscape

**Show:** Scan type table (SAST, Secret Detection, SCA, Container Scan, IaC, DAST) with pipeline position column

**Say:**
> "There are six distinct types of security scanning relevant to a modern software delivery pipeline. Each one runs at a different point, analyses a different artefact, and finds a different class of vulnerability. None of them is a substitute for any other. They are complementary."

**Walk through each row briefly (2 min total):**
- SAST: "Source code, every commit, finds injection and logic flaws"
- Secret Detection: "Every commit, finds committed credentials — an extremely common mistake"
- SCA: "Dependency manifests, finds known CVEs in your libraries"
- Container Scanning: "Docker image layers, finds OS-level vulnerabilities"
- IaC: "Terraform and Kubernetes configs, finds misconfigurations"
- DAST: "The running application, finds runtime vulnerabilities SAST cannot see"

**Critical Point:**
> "Write this down: SAST finds issues DAST cannot find. DAST finds issues SAST cannot find. The mature answer is: both. The maturity progression is: add them one at a time, starting with the fastest and cheapest."

**Timing:** 8 minutes

---

### SLIDE 4 — GitLab Scanner Technology Stack

**Show:** Table — GitLab feature + underlying engine + documentation link

**Say:**
> "GitLab does not build all its scanners from scratch. It integrates industry-standard open-source tools — Semgrep for SAST, Trivy for dependency and container scanning, OWASP ZAP for DAST, Gitleaks for secret detection. What GitLab adds is: pipeline integration, unified reporting schema, the security dashboard, MR widget, and policy enforcement. The scanners are proven. The integration and presentation is GitLab's contribution."

**Why this matters:**
> "If someone asks 'is GitLab's SAST as good as [expensive vendor]?' — the honest answer is: the underlying Semgrep engine is the same engine that many enterprise tools are built on. What you're comparing is ruleset depth and reporting UI, not fundamentally different technology."

**Timing:** 4 minutes

---

### SLIDE 5 — Pipeline Scan Placement Diagram

**Show:** The pipeline diagram with scans mapped to positions (Diagram 1 from architecture file)

**Whiteboard version:** Draw the pipeline → then add scan types at each stage

**Say:**
> "Look at where SAST sits versus where DAST sits. SAST runs at commit time — when the developer is still in context, still thinking about the code they just wrote, and a finding takes 30 seconds to understand and fix. DAST runs against a deployed application — which means you need a running environment, which means you're in a QA stage, which means the developer has moved on to something else, and a finding requires them to context-switch back. The position in the pipeline directly determines the cost to act on a finding."

**Return to the cost table** from Module 1 (1×/6×/15×/100×):
> "Commit-time = 6×. Post-deploy QA = 15×. These are not arbitrary numbers. They represent the cost of context switch, the cost of finding the right commit, the cost of regression testing, the cost of re-reviewing the MR."

**Timing:** 8 minutes

---

### SLIDE 6 — OWASP SAMM Overview

**Show:** The five business function grid

**Say:**
> "OWASP SAMM gives us a vocabulary for measuring security programme maturity. The five business functions are: Governance, Design, Implementation, Verification, Operations. Each has practices underneath it. Each practice has three maturity levels. This is not a checklist — it is a measurement framework."

**Interactive moment:**
> "Quick show of hands: who here has a documented threat assessment process for their software? [Pause] Who has a defined security build process — meaning: specific scans run automatically on every commit? [Pause] Who has a vulnerability management process with defined SLAs for critical issues? [Pause]. What you're looking at is roughly where your organisation sits on SAMM."

**Timing:** 6 minutes

---

### SLIDE 7 — GitLab vs SAMM Coverage

**Show:** The SAMM coverage map (Diagram 3)

**Say:**
> "GitLab Ultimate gets you to SAMM Level 2 on Implementation and Verification — essentially automatically. You add GitLab, enable the security scanners, and you have Secure Build and Security Testing at Level 2 without building a custom security programme. Where GitLab has gaps: threat modelling, education programmes, incident management beyond audit events. Those require process and tooling outside GitLab."

**Timing:** 4 minutes

---

### SLIDE 8 — NIST SSDF

**Show:** The four practice groups — PO, PS, PW, RV

**Say:**
> "NIST SSDF is increasingly referenced in US government contracts and defence procurement. It maps almost exactly to what we are doing in this workshop: Prepare the Organisation [PO], Protect the Software [PS], Produce Well-Secured Software [PW], Respond to Vulnerabilities [RV]. The entire Day 2 content directly addresses PW and RV. Day 1 content directly addresses PO and PS."

**For the regulated audience:**
> "If you're in defence, healthcare, automotive, or any federally regulated industry — SSDF compliance is either already required or will be required within 18 months. GitLab's pipeline + audit trail provides evidence for almost every PW and RV practice. Compliance is a by-product of doing DevSecOps correctly."

**Timing:** 5 minutes

---

### SLIDE 9 — Shift Left: The Economics

**Show:** Cost of fix table + calculation example

**Say:**
> "I want to do one specific calculation. Assume your team introduces two critical vulnerabilities per month — which is conservative. If you catch them at commit time with SAST: two hours of fixing per month. If you catch them at a quarterly penetration test: the developer has been on four other projects since then, context is completely gone, the fix requires archaeology, regression testing, re-deployment. Minimum: 100 hours per month. That's an annualised difference of roughly 2,400 engineering hours. That is more than one full-time engineer per year. SAST added to a pipeline takes 30 minutes to configure."

**Timing:** 5 minutes

---

### SLIDE 10 — Security Gates: Warn vs. Block

**Show:** The gate decision flowchart (Diagram 6)

**Say:**
> "The question I always get asked is: 'Should we set the security scanner to fail the build?' My answer: yes, eventually. Not on day one. Here is the maturity progression."

Walk through the six-month adoption sequence:
> "Month 1: warn only. You need to see what's already in your codebase before you start blocking. Month 2: fix critical findings, then enable blocking for NEW critical findings. The key word is 'newly detected' — which is a specific GitLab policy setting that means: 'only block if THIS MR introduced the finding, not if the finding was already there when this MR touched the file.' That distinction lets you adopt blocking gates without inheriting 200 historical findings as blockers."

**Timing:** 8 minutes

---

### SLIDE 11 — Vulnerability Lifecycle

**Show:** State diagram (Diagram 5)

**Say:**
> "Every finding goes through a lifecycle: Detected — Confirmed — In Progress — Resolved. Or: Detected — Dismissed with justification. The dismiss workflow is important from a compliance perspective. When a developer dismisses a finding as a false positive, GitLab requires a reason. That reason, the dismissing user, and the timestamp are all permanently logged in the audit trail. You can never deny you knew about a vulnerability — the record exists."

**Ask:**
> "What's the risk if developers can dismiss findings without a required justification? [Answer: no audit trail, real vulnerabilities hidden, compliance evidence fabricated]"

**Timing:** 5 minutes

---

### SLIDE 12 — STRIDE Threat Awareness (Brief)

**Show:** STRIDE table with RoboVision examples

**Say:**
> "I want to spend 5 minutes on threat modelling — not because we're going to do it in depth, but because it contextualises everything we do with scanners today. STRIDE is a mnemonic: Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege. For each component in your system, you ask: which of these six threats applies?"

**For the IoT/Robotics audience:**
> "Think about your OTA update channel. Tampering: what stops an attacker from replacing your firmware image with malicious code? If the answer is 'nothing' — then Module 13's content on signed artifacts and supply chain security is critical for you. An unsigned OTA update channel is a remote code execution vector for your entire device fleet."

**Timing:** 5 minutes

---

### SLIDE 13 — Discussion Exercise Setup

**Show:** The four exercise questions

**Instructions:**
> "15 minutes in groups of three. Use your pipeline diagram from yesterday's Day 1 exercise. Map where each scan type currently runs. Identify the single highest-impact scan type to add. Rate yourselves on three SAMM practices. When you come back, I want to hear two things: your biggest gap, and your highest-impact next step."

**During group work:** Walk the room, listen for:
- Teams with zero automated scanning — focus them on SAST as first step
- Teams with quarterly pentests and nothing else — validate the cost calculation
- Teams with "compliance requires X scan" — ask if it runs automatically or manually

**Debrief:** Ask 2 groups to share. Capture on whiteboard: "current state → target state → first action."

**Timing:** 20 minutes

---

### SLIDE 14 — Bridge to Module 8

**Say:**
> "We now have the framework: where scans sit in the pipeline, how maturity works, what the vulnerability lifecycle looks like. In Module 8 we go deep on the specific vulnerabilities that these scanners are looking for — the OWASP Top 10. Understanding what you're scanning for is as important as understanding how to scan. The developer who understands SQL injection fixes the SAST finding in 5 minutes. The developer who treats it as a mysterious warning from a tool they don't understand will dismiss it or work around it."

**Timing:** 2 minutes

---

## Module 7 Timing Summary

| Slide | Topic | Time |
|---|---|---|
| 1 | Opening | 1 min |
| 2 | Day 2 Reframe | 5 min |
| 3 | Scan Landscape | 8 min |
| 4 | GitLab Stack | 4 min |
| 5 | Pipeline Placement | 8 min |
| 6 | OWASP SAMM | 6 min |
| 7 | GitLab vs SAMM | 4 min |
| 8 | NIST SSDF | 5 min |
| 9 | Shift Left Economics | 5 min |
| 10 | Security Gates | 8 min |
| 11 | Vulnerability Lifecycle | 5 min |
| 12 | STRIDE | 5 min |
| 13 | Discussion Exercise | 20 min |
| 14 | Bridge to M08 | 2 min |
| **Total** | | **86 min** |

> Compress slides 6–8 (SAMM/SSDF) to 10 minutes combined if running long — they are reference material. Do not compress slides 10 (gates) or 13 (exercise).
