# STUDENT HANDBOOK — Module 1
## Evolution of Software Delivery

---

## Learning Objectives

By the end of this module you will be able to:

1. Explain why each SDLC model emerged and what problem it solved.
2. Differentiate DevOps (speed) from DevSecOps (secure speed).
3. Explain "shift left" as an economic and engineering principle.
4. Identify where your organization sits on the SDLC maturity spectrum.
5. Describe Platform Engineering and its relevance to secure delivery.

---

## 1. The Waterfall Model

### Origin
Waterfall originated in manufacturing and defense contracting. The US DoD standard MIL-STD-2167A (1988) formalized sequential, phase-gated software development. Software was treated like physical construction: plan everything, then build.

### Phases
```
Requirements → Design → Implementation → Testing → Deployment → Maintenance
```

### Why It Failed for Modern Software

| Problem | Impact |
|---------|--------|
| Requirements gathered upfront | Business needs changed by delivery time |
| Testing at the end | Defects discovered too late, cost 100x more to fix |
| Security as final audit | Vulnerabilities baked in from design |
| 12–24 month release cycles | Software obsolete on delivery |
| Big Bang delivery | Nothing integrated until the very end |

### The Cost of Late Defect Discovery

Research cited by NIST SP 800-64 shows the relative cost to fix a defect:

| Phase Found | Relative Fix Cost |
|-------------|-------------------|
| Requirements | 1x |
| Design | 5x |
| Coding | 10x |
| Testing | 20x |
| Production | 100x+ |

> **This is the economic foundation of "Shift Left" security.**

---

## 2. Agile Development

### Origin
The **Agile Manifesto** (February 2001, Snowbird, Utah) — 17 practitioners proposed a different approach:

**Four Values:**
1. Individuals and interactions **over** processes and tools
2. Working software **over** comprehensive documentation
3. Customer collaboration **over** contract negotiation
4. Responding to change **over** following a plan

### What Agile Solved
- Reduced feedback cycles from months to weeks (sprints)
- Working software delivered incrementally
- Teams closer to customer needs

### What Agile Did NOT Solve

| Remaining Problem | Consequence |
|-------------------|-------------|
| Development bottleneck moved to Operations | Code ready, servers not |
| No automated deployment | Manual releases remained slow and error-prone |
| Security still an afterthought | Retrospectives don't catch SQL injections |
| Tooling and environment sprawl | Each team did things differently |

> **Agile made development faster. It did not make deployments safer or faster.**

---

## 3. DevOps

### Origin
**DevOpsDays Ghent, 2009** — Patrick Debois coined "DevOps." The Gene Kim book *The Phoenix Project* (2013) brought it mainstream.

### Core Insight
Development and Operations are part of the same value stream. Organizational silos create artificial delays and quality failures.

### The Three Ways of DevOps (Gene Kim)

| Way | Description | Practice |
|-----|-------------|---------|
| 1st Way — Flow | Fast left-to-right delivery | CI/CD, automated testing, small batches |
| 2nd Way — Feedback | Fast right-to-left feedback | Monitoring, alerting, rapid incident response |
| 3rd Way — Learning | Continual experimentation | Blameless postmortems, hypothesis-driven delivery |

### DORA Metrics (Key DevOps Performance Indicators)

The **DevOps Research and Assessment (DORA)** team at Google identified four key metrics that predict high-performing technology organizations:

| Metric | Definition | Elite Performers |
|--------|-----------|-----------------|
| Deployment Frequency | How often you deploy to production | Multiple times per day |
| Lead Time for Changes | Time from commit to production | Less than 1 hour |
| Change Failure Rate | % of deployments causing incidents | 0–5% |
| Mean Time to Recovery | Time to recover from failure | Less than 1 hour |

**Source:** DORA State of DevOps Report — https://dora.dev/research/

### What DevOps Left Unsolved

> *"DevOps optimized speed. But speed without security means you can deploy vulnerabilities faster."*

The **2020 SolarWinds attack** demonstrated that an adversary could compromise a build pipeline itself — making fast, trusted delivery a weapon.

---

## 4. DevSecOps

### Origin
- Gartner coined "DevSecOps" approximately 2012.
- US Executive Order on Cybersecurity (May 2021) mandated SSDF compliance for federal software suppliers.
- NIST formalized the **Secure Software Development Framework (SSDF) SP 800-218** in 2022.

### Core Principle — Shift Left Security

Move security checks as early as possible in the SDLC:

```
[Where Security Lives]

  Waterfall:   .......................[PENTEST]
  Agile:       .................[QA SECURITY]
  DevSecOps:   [SCAN][SCAN][SCAN][SCAN][GATE][MONITOR]
               ^Design ^Code ^Build ^Test ^Deploy ^Runtime
```

### What DevSecOps Adds

| Capability | What It Catches |
|------------|-----------------|
| SAST (Static Analysis) | Code-level vulnerabilities (injection, hardcoded secrets) |
| DAST (Dynamic Analysis) | Runtime vulnerabilities in running applications |
| SCA (Software Composition Analysis) | Vulnerable open source dependencies |
| Secret Detection | API keys, tokens, passwords committed to code |
| Container Scanning | Vulnerable base images, OS packages |
| IaC Scanning | Misconfigured Terraform, Kubernetes manifests |
| SBOM Generation | Complete inventory of software components |
| Artifact Signing | Proof of artifact integrity and provenance |

### OWASP Software Assurance Maturity Model (SAMM)

OWASP SAMM defines a maturity model across **5 business functions**:

| Function | Focus |
|----------|-------|
| Governance | Policies, compliance, education |
| Design | Threat modeling, secure architecture |
| Implementation | Secure coding, dependency management |
| Verification | Security testing, requirements validation |
| Operations | Incident response, vulnerability management |

Each function has **3 maturity levels (0–3)**. Level 0 = no activity; Level 3 = mature, measured practice.

**Source:** https://owaspsamm.org

### NIST Secure Software Development Framework (SSDF)

NIST SP 800-218 defines **4 practice groups**:

| Practice Group | Abbreviation | Focus |
|---------------|-------------|-------|
| Prepare the Organization | PO | Tools, training, environment |
| Protect the Software | PS | Protect code, environments, tools |
| Produce Well-Secured Software | PW | Secure design, coding, testing |
| Respond to Vulnerabilities | RV | Identify, report, address vulnerabilities |

**Source:** https://csrc.nist.gov/Projects/ssdf

---

## 5. Platform Engineering

### Origin
As DevSecOps toolchains expanded (GitHub + Jenkins + SonarQube + Nexus + Snyk + Jira + PagerDuty + Vault...), developer cognitive load became unsustainable. Platform Engineering emerged as a discipline around 2020–2022.

### CNCF Definition
> *"Platform engineering is the discipline of designing and building toolchains and workflows that enable self-service capabilities for software engineering organizations."*

**Source:** CNCF Platform Engineering Whitepaper — https://tag-app-delivery.cncf.io/whitepapers/platform/

### The Internal Developer Platform (IDP)

A Platform Team builds "golden paths" — pre-approved, pre-secured delivery pipelines that developers self-serve from.

```
Developer Request → Golden Path → Pre-built Pipeline → Security Built In → Deploy
```

Instead of:
```
Developer → Find tools → Configure Jenkins → Add SonarQube → Configure Nexus → Add Snyk → ...
```

### Why This Matters for AI / IoT / Robotics Teams

For teams delivering:
- AI model artifacts with provenance requirements
- Firmware images with signing and OTA delivery
- Robotics software with deterministic build requirements
- Edge device software with air-gap and offline deployment needs

...a unified platform with pre-built secure pipelines is not a convenience. It is a requirement.

---

## 6. Evolution Summary

```
SDLC Evolution Timeline

1970s-1990s:  WATERFALL     — Sequential, plan-driven, big-bang
2001:         AGILE         — Iterative, customer-focused, sprint-based
2009:         DEVOPS        — Dev+Ops unified, CI/CD, automation
2012+:        DEVSECOPS     — Security embedded throughout
2020+:        PLATFORM ENG  — Self-service, golden paths, IDP
```

| Model | Primary Win | Primary Gap |
|-------|-------------|-------------|
| Waterfall | Structured, documented | Too slow, defects found too late |
| Agile | Faster iteration | Operations and security still siloed |
| DevOps | Automated delivery | Security not embedded |
| DevSecOps | Security in pipeline | Toolchain sprawl, cognitive overload |
| Platform Engineering | Self-service, standardized | Requires mature platform team investment |

---

## 7. Discussion Exercise — Map Your Delivery Process

### Individual Work (5 minutes)

Answer these questions about your current software delivery process:

1. **Phases:** List the phases from idea/requirement to production.
2. **Handoffs:** Where does work transfer between teams/people?
3. **Lead Time:** Roughly how long from commit to production?
4. **Security Touchpoint:** Where does security currently appear?
5. **Security Tool:** Do you have automated security scanning in your pipeline today?

### Pair Discussion (5 minutes)

Share with your partner:
- Where is your biggest bottleneck?
- Where is your security blind spot?
- What would have to change to halve your lead time?

### Group Debrief

Be ready to share one insight about where your process sits on the SDLC maturity spectrum.

---

## 8. Key Concepts Summary

| Term | Definition |
|------|-----------|
| SDLC | Software Development Life Cycle — the end-to-end process of building software |
| Shift Left | Moving security testing earlier in the pipeline to find defects cheaper |
| CI/CD | Continuous Integration / Continuous Delivery — automated build, test, deploy |
| DevSecOps | Cultural and technical practice of embedding security into DevOps workflows |
| DORA Metrics | Four research-validated metrics for measuring software delivery performance |
| OWASP SAMM | Maturity model for measuring an organization's software security practices |
| NIST SSDF | Framework for secure software development from the US government |
| SBOM | Software Bill of Materials — complete inventory of software components |
| IDP | Internal Developer Platform — self-service developer toolchain |
| Platform Engineering | Discipline of building developer self-service platforms |

---

## References

| Source | URL |
|--------|-----|
| Agile Manifesto | https://agilemanifesto.org |
| DORA Research | https://dora.dev/research/ |
| NIST SSDF SP 800-218 | https://csrc.nist.gov/Projects/ssdf |
| OWASP SAMM | https://owaspsamm.org |
| CNCF Platform Whitepaper | https://tag-app-delivery.cncf.io/whitepapers/platform/ |
| US EO on Cybersecurity | https://www.whitehouse.gov/briefing-room/presidential-actions/2021/05/12/executive-order-on-improving-the-nations-cybersecurity/ |
| OWASP Top 10 | https://owasp.org/www-project-top-ten/ |
