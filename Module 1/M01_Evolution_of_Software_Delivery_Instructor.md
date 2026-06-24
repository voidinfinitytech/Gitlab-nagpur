# INSTRUCTOR HANDBOOK — MODULE 1
## Evolution of Software Delivery: From Waterfall to Platform Engineering

---

### Module Metadata

| Field             | Value                                             |
|-------------------|---------------------------------------------------|
| Module Number     | 1 of 16                                           |
| Day               | Day 1                                             |
| Duration          | 75 minutes (50 min content + 25 min discussion)   |
| Difficulty        | Foundational — sets the frame for all other modules |
| Prerequisites     | None                                              |
| Room Setup        | Whiteboard accessible; projector for diagrams     |

---

## Instructor Preparation Notes

### Mental Model to Transfer
Your single job in this module is to make participants feel the **pain** of each era before you introduce the solution. Do not rush to DevSecOps. Let them sit with the dysfunction of Waterfall, then Agile, then early DevOps. The visceral memory of those pain points is what makes every subsequent GitLab feature feel like a relief rather than a feature list.

### Audience Calibration
This audience (AI Engineers, Robotics/IoT Engineers, Embedded Developers) often works in **mixed-maturity environments** — their firmware team may still be on a 12-month release cycle while their cloud team deploys 20 times a day. This tension is your teaching material. Call it out explicitly.

### Opening Hook (First 3 Minutes)
Ask the room, show of hands:
> *"Who here has been blamed for a production incident caused by a change that was made six weeks ago?"*

Then:
> *"Who here has waited more than two weeks for a security scan to complete before you could ship?"*

These hands are your diagnostic signal. Note who raises hands — those participants will provide the best real-world examples throughout the day.

---

## Learning Objectives

By the end of this module, participants will be able to:

1. **Explain** why software delivery methodologies evolved — with specific failure modes at each transition, not just timelines.
2. **Describe** the fundamental tension that drove the DevOps movement: the Wall of Confusion between development and operations.
3. **Articulate** why DevSecOps emerged as a response to security being a release-blocking bottleneck.
4. **Identify** Platform Engineering as the current response to DevOps toolchain fragmentation.
5. **Map** their own organization's current delivery process to a maturity stage and name the dominant pain point.

---

## Business Context (Instructor Talking Points)

### Why This Matters to the Room

In 2024, Ponemon Institute reported the average cost of a data breach at **$4.88 million** (IBM Cost of a Data Breach Report 2024). The majority of vulnerabilities exploited in production were known vulnerabilities that were never detected during development. This is not a security tools problem — it is a **delivery process problem**. Organizations were not built to deliver security feedback early enough to act on it.

For IoT and Robotics teams specifically: a firmware vulnerability deployed to 50,000 edge devices has a fundamentally different remediation profile than a web application. You cannot `kubectl rollout undo` a robot on a factory floor. The cost of late detection in physical systems is **10–100× higher** than in cloud-native software. This is the business case for Shift Left that your audience will feel immediately.

---

## Module Content — Detailed Teaching Guide

---

### SECTION 1: Waterfall (10 minutes)

#### Instructor Setup
Draw a horizontal timeline on the whiteboard. Label it with these phases:
```
Requirements → Design → Development → Testing → Integration → Deployment → Maintenance
```

Ask: *"What happens when the Requirements phase takes 6 months and the business changes halfway through?"*

#### Core Teaching Points

**The Origin Problem:**
Waterfall emerged in the 1950s–1970s from manufacturing and construction disciplines. Winston Royce's 1970 paper "Managing the Development of Large Software Systems" is often cited as the origin (ironically, Royce himself argued it was flawed for software). The assumption: software could be planned as completely as a bridge.

**The Fundamental Incorrect Assumption:**
> Software requirements are not knowable upfront in the way that physical constraints are.

**The Pain (Make Participants Feel This):**
Narrate this scenario:
> *"A defence contractor spends 18 months gathering requirements. The development team spends 24 months building. Integration testing begins. The testing team discovers the authentication module doesn't match the UI requirements because both teams had different versions of the spec. Six months of rework. The software is delivered 18 months late. The business requirements have changed three times during that period. Half the delivered features are no longer needed."*

**The Metrics that Expose the Problem:**
- CHAOS Report (Standish Group) — in the 1990s, only 16% of software projects were delivered on time, on budget, with full features.
- Average project overrun: 189% of original time estimate.
- Average cost overrun: 222% of original budget.

**Why It Persisted:**
The process *looked* controllable to management. Gantt charts. Phase gates. Sign-offs. The illusion of control was valued over actual delivery.

**Failure Modes to Name Explicitly:**
1. **Big Bang Integration** — nothing tested together until the end; integration failures discovered too late.
2. **Requirement Drift** — by delivery time, the world had changed.
3. **Knowledge Silos** — developers and testers rarely communicated; defects festered for months before discovery.
4. **Feedback Latency** — the gap between introducing a bug and discovering it could be measured in months.

**Transition Question for Class:**
> *"What was the core problem Waterfall tried to solve with phase gates?"*
> Answer: Scope control and risk management. This is important — Waterfall was not irrational. It was the wrong tool for an unpredictable domain.

---

### SECTION 2: Agile (10 minutes)

#### Core Teaching Points

**The Origin:**
The Agile Manifesto (2001) was written by 17 practitioners in Snowbird, Utah. They had all experienced the dysfunction of heavyweight processes first-hand. The manifesto is 68 words. That brevity was intentional.

**What Agile Actually Changed:**
- Replaced linear phases with **iterative cycles** (sprints)
- Replaced comprehensive documentation with **working software** as the primary progress measure
- Replaced contract negotiation with **customer collaboration**
- Replaced plan-following with **responding to change**

**What Agile Did NOT Change (Critical Point):**
> Agile fixed the requirements-gathering and iteration cycle. It did not fix the gap between Development and Operations. Developers could now ship code faster — but the path from "code done" to "running in production" remained broken.

**The New Pain Agile Created:**
- Faster development cycles exposed the bottleneck downstream: operations teams operating on ITIL change management cycles with 2-week change advisory boards.
- Developers shipping every 2 weeks, operations deploying every 6 weeks. The backlog grew.
- Quality shifted: instead of a big waterfall defect at the end, teams now had *continuous* defect discovery with *insufficient* deployment infrastructure to respond.

**For This Audience:**
Robotics and IoT teams frequently still operate in a **Waterfall-within-Agile hybrid** — Agile for software, Waterfall for hardware. Point this out as a real pattern they likely recognize.

**Failure Modes:**
1. **ScrumBut** — "We do Scrum, but we skip retrospectives, but we don't do demos, but..." The ceremony without the discipline.
2. **Velocity as a metric** — Teams optimizing points rather than outcomes.
3. **Agile Theater** — Standups, boards, and sprints with no actual change in delivery behaviour.

---

### SECTION 3: DevOps (15 minutes)

#### Core Teaching Points

**The Origin:**
Patrick Debois coined "DevOps" in 2009, inspired by Andrew Shafer's "Agile Infrastructure" talk at Agile 2008. The Velocity Conference talk by John Allspaw and Paul Hammond — "10+ Deploys Per Day: Dev and Ops Cooperation at Flickr" — demonstrated it was achievable at scale.

**The Core Problem Being Solved:**
The Wall of Confusion:

```
DEVELOPMENT SIDE        |    OPERATIONS SIDE
------------------------|------------------------
"Works on my machine"   |  "Not my problem to fix"
Deploy fast             |  Deploy stable
Change often            |  Change control process
Feature branches        |  Locked production
Devs control code       |  Ops controls servers
```

**The Three Ways (Gene Kim's Framework — "The Phoenix Project"):**
1. **Flow** — Accelerate the flow of work from Dev to Ops to Customer. Eliminate constraints. Reduce batch sizes.
2. **Feedback** — Create fast, amplified feedback loops from right to left. Fix problems where they are found.
3. **Continual Learning** — Create a culture of continual experimentation and learning from failure.

**DORA Research (Reference Point):**
The DevOps Research and Assessment (DORA) program — now part of Google Cloud — has tracked organizational performance metrics since 2014. Their research identifies four key metrics:

| Metric                    | Elite Performers          | Low Performers            |
|---------------------------|---------------------------|---------------------------|
| Deployment Frequency      | On-demand (multiple/day)  | Less than once per 6 months |
| Lead Time for Changes     | Less than 1 hour          | 6+ months                 |
| Change Failure Rate       | 0–5%                      | 46–60%                    |
| Mean Time to Recovery     | Less than 1 hour          | 1+ week                   |

Source: DORA State of DevOps Report 2023 (https://dora.dev/research/)

**What DevOps Introduced:**
- CI/CD pipelines as the primary delivery mechanism
- Infrastructure as Code (IaC) — configuration managed like software
- Automated testing integrated into the delivery process
- Monitoring and observability as first-class engineering concerns
- Shared responsibility between development and operations

**The Pain DevOps Did NOT Solve:**
> Security was still an afterthought. CI/CD pipelines automated the deployment of vulnerable code faster than ever before. Organizations were now shipping insecure software at elite performance levels.

**The Anecdote to Tell:**
> *"Equifax 2017. Apache Struts vulnerability (CVE-2017-5638) was publicly disclosed on March 7, 2017. Equifax's systems were breached on May 13, 2017 — 66 days later. The vulnerability was in a known, scannable dependency. An automated dependency scanner in the CI pipeline would have surfaced it immediately. 147 million customers' personal data was exposed. The breach cost Equifax approximately $1.4 billion in settlements and remediation."*

**Instructor Note:**
This example lands hard with security engineers. For IoT/robotics audiences, substitute the Triton/TRISIS ICS malware attack (2017) which targeted Schneider Electric safety instrumented systems — emphasizing that vulnerabilities in embedded and OT systems have physical-world consequences.

---

### SECTION 4: DevSecOps (15 minutes)

#### Core Teaching Points

**The Origin:**
Gartner first used the term "DevSecOps" around 2012. The concept crystallized as organizations realized that adding security scanning as a post-deployment step defeated the purpose of CI/CD velocity. If you deploy fast but only scan for vulnerabilities monthly, you're running faster toward the cliff.

**The Fundamental Shift:**
> Security transforms from a **gate at the end** to a **capability embedded throughout**.

Draw this on the whiteboard:

```
TRADITIONAL (Security at the End):
Dev → Dev → Dev → Dev → Dev → SEC GATE → Deploy
                                  ↑
                         "You failed. Rewrite."
                         (3 months of work thrown away)

DEVSECOPS (Security Shifted Left):
Dev → [SAST] → Dev → [SCA] → Dev → [Container Scan] → [DAST] → Deploy
  ↑              ↑              ↑                 ↑
Immediate     Immediate      Immediate        Immediate
feedback      feedback       feedback         feedback
```

**What "Shift Left" Actually Means:**
The "left" refers to the position on a pipeline timeline diagram — left is earlier, right is later. Security scans moved from post-deployment (far right) to code commit (far left). The earlier a vulnerability is detected, the cheaper it is to fix.

**Cost of Defect Fix (IBM Systems Sciences Institute):**
| Phase of Detection    | Relative Cost to Fix |
|-----------------------|----------------------|
| Design                | 1×                   |
| Development           | 6×                   |
| Testing               | 15×                  |
| Production            | 100×                 |

This is the business case for Shift Left in a single table. Return to it throughout the workshop.

**Key DevSecOps Capabilities (GitLab Ultimate Context):**
1. **SAST** — Static Application Security Testing (analyse code for vulnerabilities at commit)
2. **DAST** — Dynamic Application Security Testing (test running application)
3. **SCA** — Software Composition Analysis / Dependency Scanning
4. **Secret Detection** — Find API keys, passwords, tokens committed to repositories
5. **Container Scanning** — Analyse Docker images for known CVEs
6. **IaC Security** — Terraform, Kubernetes manifest security scanning
7. **SBOM** — Software Bill of Materials generation

**Reference: OWASP SAMM (Software Assurance Maturity Model)**
OWASP SAMM defines five business functions for secure software development:
- Governance, Design, Implementation, Verification, Operations
Source: https://owaspsamm.org/

**Reference: NIST SSDF (Secure Software Development Framework)**
NIST SP 800-218 SSDF organises practices into four groups:
- Prepare the Organization (PO), Protect the Software (PS), Produce Well-Secured Software (PW), Respond to Vulnerabilities (RV)
Source: https://csrc.nist.gov/Projects/ssdf

---

### SECTION 5: Platform Engineering (10 minutes)

#### Core Teaching Points

**The Origin:**
Platform Engineering emerged around 2019–2022 as a response to a new problem: DevOps toolchain fragmentation. Organizations had adopted CI/CD — but each team chose their own stack. The result was a heterogeneous patchwork of Jenkins, GitHub Actions, CircleCI, ArgoCD, SonarQube, Artifactory, Vault, and dozens of other tools. The cognitive load on development teams became unsustainable.

**The Core Problem:**
> Platform Engineering addresses the Internal Developer Experience (IDP — Internal Developer Platform). The goal is to give development teams a curated, opinionated, production-ready platform so they can focus on business logic, not infrastructure.

**The Gartner Definition:**
Gartner predicts that by 2026, 80% of large software engineering organisations will establish platform engineering teams. (Gartner, 2023)

**The Core Pattern:**
- Self-service infrastructure provisioning
- Golden paths — opinionated, pre-approved deployment patterns
- Developer portals (Backstage, Port)
- Standardised CI/CD templates
- Centralised security and compliance policies

**Why This Matters for GitLab Ultimate:**
GitLab Ultimate is, architecturally, a Platform Engineering response. Its single-application approach eliminates the integration overhead of a fragmented toolchain. One authentication system. One audit trail. One API. One security policy engine.

**The Cynical Question (and the Real Answer):**
> *"Isn't this just vendor lock-in in disguise?"*
> Answer: Every platform creates dependency. The trade-off is: dependency on a single, coherent platform with defined interfaces vs. dependency on 15 tools with custom integrations that each break independently. The question is which dependency is easier to manage, replace, and reason about.

---

## Discussion Exercise (25 minutes)

### Exercise: Map Your Current Delivery Process

**Instructions (Give to participants on slide):**

In groups of 3–4, spend 15 minutes mapping your current software delivery process. Answer these questions:

1. **Where does a code change start?** (IDE, branch, issue?)
2. **How many manual handoffs occur between developer commit and production?**
3. **Where does security currently sit in your process?** (Before or after deployment?)
4. **What is your actual deployment frequency today?**
5. **What is your longest feedback loop?** (How long between making a mistake and knowing about it?)
6. **What tool or process is your biggest bottleneck right now?**

Draw your process on paper or the whiteboard.

**Debrief (10 minutes):**
Ask 2–3 groups to present. Listen for:
- Security appearing only at the end (common)
- Manual approval gates with no defined SLA
- "We don't really know our deployment frequency"
- Separate teams owning separate stages with no shared tooling

These become recurring reference points throughout the workshop. When you introduce GitLab capabilities in later modules, refer back to the specific pain points surfaced here.

---

## Failure Modes and Anti-Patterns (Instructor Reference)

| Anti-Pattern | What It Looks Like | Why It Persists | Intervention |
|---|---|---|---|
| DevOps Theater | Teams have CI/CD pipelines but security is still a separate manual gate. Agile standups. No actual change in lead time. | Org structure unchanged. Tool adoption without process change. | DORA metrics expose the gap. Lead time measurement makes the bottleneck visible. |
| Security as Police | Security team owns the tooling, dev teams have no visibility. Developers get a report 3 days before release. | Historical separation of roles. | Give developers immediate feedback in MR. Make security a developer tool, not an audit tool. |
| Pseudo-Shift-Left | SAST added to pipeline but findings are suppressed or ignored. "Green pipeline" despite 40 critical findings. | Teams pressured to ship. No policy enforcement. | Security policies with failure thresholds. Auto-blocking of critical findings. |
| Tool Proliferation | Each team owns their own security scanner. No aggregated view. Findings lost. | Autonomy without governance. | Platform-level consolidation. Centralised security dashboard. |

---

## Knowledge Check Questions

Use these at the end of the module or at the start of the next to verify retention:

**L1 (Motivation):**
> "What specific failure did Agile solve that Waterfall couldn't? What failure did it *not* solve?"

**L2 (Mechanism):**
> "Explain the Wall of Confusion. What was each side optimising for and why did that cause conflict?"

**L3 (Mental Model):**
> "If your team doubles its deployment frequency tomorrow without changing your security process, what happens to your risk profile?"

**L4 (Mastery):**
> "Your organisation has just adopted CI/CD. Deployment frequency has increased from monthly to daily. Three months later, the CISO reports that the number of production vulnerabilities has *increased*. Diagnose the failure."

---

## Timing Guide

| Section                          | Time     | Notes                                        |
|----------------------------------|----------|----------------------------------------------|
| Opening hook                     | 3 min    | Hands up; note respondents                   |
| Section 1: Waterfall             | 10 min   | Draw timeline on whiteboard                  |
| Section 2: Agile                 | 10 min   | Contrast pair: what it fixed vs. what it didn't |
| Section 3: DevOps                | 15 min   | Wall of Confusion diagram; DORA metrics      |
| Section 4: DevSecOps             | 15 min   | Cost of defect table; shift-left diagram     |
| Section 5: Platform Engineering  | 10 min   | GitLab as platform engineering response      |
| Discussion Exercise              | 25 min   | 15 min group work + 10 min debrief           |
| **Total**                        | **88 min** | Compress Agile to 7 min if running long    |

---

## References and Citations

1. **Agile Manifesto (2001)** — Beck, K. et al. https://agilemanifesto.org/
2. **DORA State of DevOps Report 2023** — https://dora.dev/research/2023/dora-report/
3. **NIST SP 800-218 — Secure Software Development Framework (SSDF)** — https://csrc.nist.gov/Projects/ssdf
4. **OWASP SAMM v2.0** — https://owaspsamm.org/model/
5. **IBM Cost of a Data Breach Report 2024** — https://www.ibm.com/reports/data-breach
6. **GitLab DevSecOps Platform Overview** — https://about.gitlab.com/solutions/dev-sec-ops/
7. **The Phoenix Project** — Kim, G., Behr, K., Spafford, G. (2013) — The Wall of Confusion narrative
8. **Standish Group CHAOS Report** — Referenced via Standish Group International
9. **"10+ Deploys Per Day" (2009)** — Allspaw, J., Hammond, P. — Velocity Conference 2009
10. **Gartner Platform Engineering Prediction 2023** — https://www.gartner.com/en/articles/what-is-platform-engineering
11. **Royce, W.W. (1970)** — "Managing the Development of Large Software Systems" — IEEE WESCON
