# ARCHITECTURE DIAGRAMS — MODULE 1
## Evolution of Software Delivery
### All diagrams in Mermaid format — render at https://mermaid.live

---

## Diagram 1: Software Delivery Evolution Timeline

```mermaid
timeline
    title Software Delivery Methodology Evolution
    section 1950s-1990s
        Waterfall : Sequential phases
                  : Requirements → Design → Code → Test → Deploy
                  : Feedback latency — 12-18 months
                  : CHAOS Report — 16% success rate
    section 2001
        Agile : Iterative sprints (2 weeks)
              : Agile Manifesto published
              : Fixed requirements drift
              : Did NOT fix Dev-Ops gap
    section 2009
        DevOps : Broke the Wall of Confusion
               : CI/CD automation
               : Infrastructure as Code
               : Security still an afterthought
    section 2012
        DevSecOps : Security shifted left
                  : Scans at every pipeline stage
                  : SAST, DAST, SCA integrated
                  : OWASP SAMM, NIST SSDF
    section 2020+
        Platform Engineering : Internal Developer Platform
                             : Self-service golden paths
                             : GitLab Ultimate
                             : Single application platform
```

---

## Diagram 2: Waterfall Feedback Latency Problem

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Test as Test Team
    participant Ops as Operations
    participant Bug as Bug (Defect)

    Note over Dev: Month 1-6: Requirements
    Note over Dev: Month 6-12: Development
    Dev->>Bug: Introduces defect in Month 8
    Note over Dev: Month 12-16: Integration
    Note over Test: Month 16-18: Testing
    Test->>Dev: Discovers defect in Month 17
    Note over Bug: 9 months between creation and discovery
    Dev-->>Dev: Developer has context for different code now
    Dev->>Test: Fix requires 3 months of rework
    Note over Ops: Month 21: Finally deployed
    Note over Test: Total delay from defect to fix: 12 months
```

---

## Diagram 3: The Wall of Confusion

```mermaid
graph LR
    subgraph DEV ["🔨 Development Team"]
        D1["Deploy Fast"]
        D2["Ship Features"]
        D3["Change Frequently"]
        D4["Works on My Machine"]
    end

    subgraph WALL ["⛔ WALL OF CONFUSION"]
        W1["Different Goals"]
        W2["Different Incentives"]
        W3["Different Tools"]
        W4["Blame Culture"]
    end

    subgraph OPS ["🖥️ Operations Team"]
        O1["Deploy Stable"]
        O2["Maintain Uptime"]
        O3["Control Changes"]
        O4["Change Advisory Board"]
    end

    D1 -->|conflicts with| W1
    D2 -->|conflicts with| W2
    D3 -->|conflicts with| W3
    D4 -->|conflicts with| W4
    W1 -->|blocks| O1
    W2 -->|blocks| O2
    W3 -->|blocks| O3
    W4 -->|blocks| O4

    style DEV fill:#2ecc71,color:#000
    style WALL fill:#e74c3c,color:#fff
    style OPS fill:#3498db,color:#fff
```

---

## Diagram 4: DevOps — Breaking the Wall

```mermaid
graph TB
    subgraph DEVOPS ["DevOps — Shared Ownership Model"]
        direction LR
        C[Code Commit] --> CI[CI Pipeline]
        CI --> BUILD[Build]
        BUILD --> TEST[Automated Test]
        TEST --> PKG[Package]
        PKG --> DEPLOY[Deploy]
        DEPLOY --> MONITOR[Monitor]
        MONITOR -->|Feedback Loop| C
    end

    subgraph SHARED ["Shared Responsibilities"]
        SR1["Developers write tests"]
        SR2["Developers define pipelines"]
        SR3["Ops provides platform"]
        SR4["Shared on-call rotation"]
    end

    DEVOPS --> SHARED

    style DEVOPS fill:#2ecc71,color:#000
    style SHARED fill:#3498db,color:#fff
```

---

## Diagram 5: Security at the End vs Shift Left

```mermaid
graph LR
    subgraph TRADITIONAL ["❌ Traditional — Security at the End"]
        direction LR
        T1[Commit] --> T2[Build] --> T3[Test] --> T4[Package] --> T5["🔴 Security Gate\n(weeks later)"] --> T6[Deploy]
    end

    subgraph DEVSECOPS ["✅ DevSecOps — Shift Left"]
        direction LR
        S1[Commit] --> SAST["SAST\n(seconds)"]
        SAST --> S2[Build]
        S2 --> SCA["SCA\n(seconds)"]
        SCA --> S3[Test]
        S3 --> CSCAN["Container\nScan"]
        CSCAN --> S4[Package]
        S4 --> DAST["DAST\n(minutes)"]
        DAST --> S5[Deploy]
    end

    style TRADITIONAL fill:#e74c3c,color:#fff
    style DEVSECOPS fill:#2ecc71,color:#000
    style T5 fill:#c0392b,color:#fff
    style SAST fill:#27ae60,color:#fff
    style SCA fill:#27ae60,color:#fff
    style CSCAN fill:#27ae60,color:#fff
    style DAST fill:#27ae60,color:#fff
```

---

## Diagram 6: Cost of Defect Detection — Business Case

```mermaid
xychart-beta
    title "Relative Cost to Fix Defect by Detection Phase"
    x-axis ["Design", "Development", "Testing", "Production"]
    y-axis "Relative Cost (×)" 0 --> 110
    bar [1, 6, 15, 100]
```

---

## Diagram 7: Platform Engineering — Tool Sprawl vs Consolidation

```mermaid
graph TB
    subgraph SPRAWL ["❌ Tool Sprawl (Before Platform Engineering)"]
        direction TB
        GH[GitHub] ---|separate auth| JK[Jenkins]
        JK ---|separate auth| SQ[SonarQube]
        SQ ---|separate auth| AF[Artifactory]
        AF ---|separate auth| JR[Jira]
        JR ---|separate auth| HV[HashiCorp Vault]
        HV ---|separate auth| AM[ArgoCD]
        AM ---|separate auth| CR[Checkmarx]
        CR ---|15+ separate tools| SK[Snyk]
    end

    subgraph GITLAB ["✅ GitLab Ultimate — Single Application"]
        direction TB
        GL["GitLab Ultimate"] --> SCM["Source Control"]
        GL --> PIPE["CI/CD Pipelines"]
        GL --> SEC["Security Scanning\n(SAST/DAST/SCA/Secrets)"]
        GL --> PACK["Package Registry"]
        GL --> PLAN["Planning"]
        GL --> AUDIT["Unified Audit Log"]
        GL --> COMP["Compliance & Governance"]
    end

    style SPRAWL fill:#e74c3c,color:#fff
    style GITLAB fill:#2ecc71,color:#000
    style GL fill:#fc6d26,color:#fff
```

---

## Diagram 8: DORA Metrics Visualisation

```mermaid
quadrantChart
    title DORA Performance Quadrants
    x-axis "Low Deployment Frequency" --> "High Deployment Frequency"
    y-axis "High Lead Time" --> "Low Lead Time"
    quadrant-1 Elite Performers
    quadrant-2 High Frequency, High Lead Time
    quadrant-3 Low Performers
    quadrant-4 Low Frequency, Low Lead Time
    Elite: [0.9, 0.9]
    High Performers: [0.7, 0.65]
    Medium Performers: [0.45, 0.4]
    Low Performers: [0.1, 0.15]
```

---

## Diagram 9: DevSecOps Maturity Stages

```mermaid
graph LR
    M0["Stage 0\nNo Security\nin Pipeline"] -->|Add SAST| M1["Stage 1\nBasic SAST\nat Commit"]
    M1 -->|Add SCA| M2["Stage 2\nDependency\nScanning"]
    M2 -->|Add Container +\nSecret Detection| M3["Stage 3\nContainer &\nSecret Scanning"]
    M3 -->|Add DAST +\nPolicies| M4["Stage 4\nFull DevSecOps\n+ Gates"]
    M4 -->|Add SBOM +\nGovernance| M5["Stage 5\nSupply Chain\n+ Compliance"]

    style M0 fill:#e74c3c,color:#fff
    style M1 fill:#e67e22,color:#fff
    style M2 fill:#f1c40f,color:#000
    style M3 fill:#2ecc71,color:#000
    style M4 fill:#27ae60,color:#fff
    style M5 fill:#1a5276,color:#fff
```

---

## Diagram 10: Vulnerability Attack Window Concept

```mermaid
sequenceDiagram
    participant CVE as CVE Published
    participant ORG as Your Organisation
    participant ATK as Attacker

    CVE->>ATK: Vulnerability details public
    Note over ATK: Attacker develops exploit
    CVE->>ORG: TRADITIONAL: Quarterly scan detects it (90 days later)
    ATK->>ORG: Breach occurs (66 days after disclosure — Equifax)
    
    Note over CVE, ORG: Gap = Attack Window
    
    CVE->>ORG: DEVSECOPS: CI/CD scan detects it (next commit — minutes/hours)
    Note over ORG: Patch deployed before exploit weaponised
```

---

## Usage Notes for Instructors

- All diagrams render at https://mermaid.live — paste the code block content
- For whiteboard sessions, use Diagrams 3 (Wall of Confusion) and 5 (Shift Left) as hand-drawn starting points
- Diagram 6 (Cost of Defect) should be shown early and referenced throughout Day 2
- Diagram 9 (Maturity Stages) is used again in Module 16 (DORA Metrics)
