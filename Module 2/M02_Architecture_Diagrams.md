# ARCHITECTURE DIAGRAMS — MODULE 2
## GitLab Ultimate Overview
### Render at https://mermaid.live

---

## Diagram 1: GitLab Component Architecture

```mermaid
graph TB
    subgraph CLIENT ["Client Layer"]
        BR[Browser / UI]
        GIT[Git CLI]
        API_C[API Clients / GitLab CLI]
    end

    subgraph INGESTION ["Ingestion Layer"]
        WH[GitLab Workhorse\nLarge files / LFS / Artifacts]
        SHELL[GitLab Shell\nSSH Operations]
    end

    subgraph APP ["Application Layer"]
        PUMA[Puma\nRails App / UI / API]
        SIDEKIQ[Sidekiq\nBackground Jobs]
    end

    subgraph DATA ["Data Layer"]
        PG[(PostgreSQL\nAll Platform Data)]
        REDIS[(Redis\nCache + Job Queue)]
    end

    subgraph GIT_STORE ["Git Storage Layer"]
        GITALY[Gitaly\ngRPC Git Service]
        PRAEFECT[Praefect\nHA Coordinator]
        DISK[(Git Repositories\non Disk)]
    end

    subgraph STORAGE ["Object Storage"]
        OBJ[S3 / GCS / Azure Blob / MinIO\nArtifacts, LFS, Images]
    end

    subgraph RUNNER ["Execution Layer (Separate)"]
        R1[GitLab Runner 1\nDocker Executor]
        R2[GitLab Runner 2\nKubernetes Executor]
        R3[GitLab Runner N\nShell Executor]
    end

    BR --> WH
    GIT --> SHELL
    API_C --> WH
    WH --> PUMA
    SHELL --> GITALY
    PUMA --> PG
    PUMA --> REDIS
    PUMA --> SIDEKIQ
    SIDEKIQ --> REDIS
    SIDEKIQ --> R1
    SIDEKIQ --> R2
    SIDEKIQ --> R3
    PUMA --> GITALY
    GITALY --> PRAEFECT
    PRAEFECT --> DISK
    R1 --> OBJ
    R2 --> OBJ
    R3 --> OBJ
    OBJ --> WH

    style CLIENT fill:#3498db,color:#fff
    style INGESTION fill:#9b59b6,color:#fff
    style APP fill:#2ecc71,color:#000
    style DATA fill:#e67e22,color:#fff
    style GIT_STORE fill:#1abc9c,color:#fff
    style STORAGE fill:#95a5a6,color:#fff
    style RUNNER fill:#e74c3c,color:#fff
```

---

## Diagram 2: Request Flow — Developer Push to Pipeline Result

```mermaid
sequenceDiagram
    participant DEV as Developer
    participant SHELL as GitLab Shell\n(SSH)
    participant GITALY as Gitaly\n(Git Storage)
    participant PUMA as Puma\n(Rails App)
    participant SIDEKIQ as Sidekiq\n(Background Jobs)
    participant REDIS as Redis\n(Queue)
    participant RUNNER as GitLab Runner
    participant OBJ as Object Storage
    participant PG as PostgreSQL

    DEV->>SHELL: git push (SSH)
    SHELL->>GITALY: Store commit/ref
    GITALY-->>SHELL: Success
    SHELL->>PUMA: POST notify (HTTP callback)
    PUMA->>PG: Create Pipeline record
    PUMA->>REDIS: Enqueue pipeline trigger job
    REDIS->>SIDEKIQ: Dispatch job
    SIDEKIQ->>PG: Assign job to available Runner
    RUNNER->>PUMA: Poll for pending jobs (API)
    PUMA-->>RUNNER: Job payload (script, variables, image)
    RUNNER->>RUNNER: Execute job (Docker/Shell/K8s)
    RUNNER->>OBJ: Upload artifacts
    RUNNER->>PUMA: Report job result (API)
    PUMA->>PG: Store result + artifact references
    DEV->>PUMA: View pipeline results (browser)
    PUMA->>PG: Query results
    PUMA-->>DEV: Render pipeline UI
```

---

## Diagram 3: Single Application vs Fragmented Toolchain

```mermaid
graph TB
    subgraph FRAGMENTED ["❌ Fragmented Toolchain — 12+ Integration Points"]
        direction TB
        GH2[GitHub\nSCM] -->|webhook| JNK[Jenkins\nCI/CD]
        JNK -->|webhook| SQ[SonarQube\nSAST]
        JNK -->|API| AF[Artifactory\nRegistry]
        SQ -->|manual| JR[Jira\nIssues]
        AF -->|webhook| ARGO[ArgoCD\nDeploy]
        JNK -->|API| SNYK[Snyk\nSCA]
        JNK -->|API| VAULT[HashiCorp Vault\nSecrets]
        ARGO -->|webhook| DD[Datadog\nMonitor]
        
        NOTE1["🔴 Each arrow = separate\nauthentication + maintenance\n+ failure mode"]
    end

    subgraph UNIFIED ["✅ GitLab Ultimate — Single Application"]
        direction LR
        GL["GitLab Ultimate\n(Single Application)"]
        GL --> GSCM[Source Control]
        GL --> GCICD[CI/CD Pipelines]
        GL --> GSAST[SAST + DAST\n+ SCA + Secrets]
        GL --> GREG[Package &\nContainer Registry]
        GL --> GPLAN[Planning]
        GL --> GSEC[Security Dashboard]
        GL --> GCOMP[Compliance\n+ Audit Trail]
        GL --> GDEP[GitOps\nDeployment]
        
        NOTE2["🟢 One authentication\nOne API\nOne audit trail\nOne security policy engine"]
    end

    style FRAGMENTED fill:#fdecea,stroke:#e74c3c
    style UNIFIED fill:#eafaf1,stroke:#2ecc71
    style GL fill:#fc6d26,color:#fff
```

---

## Diagram 4: GitLab Tier Capability Map

```mermaid
graph LR
    subgraph FREE ["GitLab Free"]
        F1["✅ Git SCM"]
        F2["✅ Basic CI/CD"]
        F3["✅ Container Registry"]
        F4["✅ Basic SAST\n(Semgrep)"]
        F5["✅ Basic Secret Detection"]
        F6["✅ 5 users max / shared Runners"]
    end

    subgraph PREMIUM ["GitLab Premium\n(adds to Free)"]
        P1["✅ Multi-level MR Approvals"]
        P2["✅ SAML / SSO"]
        P3["✅ Audit Events"]
        P4["✅ Code Owners"]
        P5["✅ Protected Environments"]
        P6["✅ Merge Request Analytics"]
    end

    subgraph ULTIMATE ["GitLab Ultimate\n(adds to Premium)"]
        U1["✅ Dependency Scanning (SCA)"]
        U2["✅ DAST"]
        U3["✅ Container Scanning"]
        U4["✅ IaC Security Scanning"]
        U5["✅ License Compliance"]
        U6["✅ Security Dashboard"]
        U7["✅ Vulnerability Management"]
        U8["✅ Security Policies"]
        U9["✅ SBOM / Dependency List"]
        U10["✅ Compliance Frameworks"]
        U11["✅ Advanced SAST"]
    end

    FREE --> PREMIUM --> ULTIMATE

    style FREE fill:#95a5a6,color:#fff
    style PREMIUM fill:#3498db,color:#fff
    style ULTIMATE fill:#fc6d26,color:#fff
```

---

## Diagram 5: GitLab Runner Executor Types

```mermaid
graph TB
    subgraph RUNNER ["GitLab Runner"]
        POLL["Poll GitLab API\nfor pending jobs"]
        
        POLL --> EXEC["Select Executor"]
        
        EXEC --> SHELL_E["Shell Executor\n• Runs directly on Runner host\n• No isolation\n• Fast\n• Legacy use cases"]
        EXEC --> DOCKER_E["Docker Executor\n• Runs in Docker container\n• Good isolation\n• Most common\n• Requires Docker daemon"]
        EXEC --> K8S_E["Kubernetes Executor\n• Runs in K8s Pod\n• Best isolation at scale\n• Auto-scales\n• Cloud-native teams"]
        EXEC --> VM_E["VirtualBox/Parallels\n• Runs in VM\n• Full OS isolation\n• Slower\n• Windows builds"]
        EXEC --> CUSTOM_E["Custom Executor\n• User-defined\n• Any environment\n• IoT/embedded use cases\n• Air-gapped environments"]
    end

    style RUNNER fill:#e74c3c,color:#fff
    style SHELL_E fill:#95a5a6,color:#fff
    style DOCKER_E fill:#2ecc71,color:#000
    style K8S_E fill:#3498db,color:#fff
    style VM_E fill:#9b59b6,color:#fff
    style CUSTOM_E fill:#e67e22,color:#fff
```

---

## Diagram 6: GitLab Deployment Models

```mermaid
graph LR
    subgraph SAAS ["GitLab.com (SaaS)"]
        S1["Multi-tenant"]
        S2["Hosted by GitLab Inc."]
        S3["Auto-updated"]
        S4["Shared Runners included"]
        S5["✅ Fastest adoption"]
        S6["❌ Data leaves your network"]
    end

    subgraph SM ["GitLab Self-Managed"]
        SM1["Your infrastructure"]
        SM2["You manage upgrades"]
        SM3["Your runners"]
        SM4["✅ Data residency"]
        SM5["✅ Air-gapped support"]
        SM6["❌ Operational overhead"]
    end

    subgraph DED ["GitLab Dedicated"]
        D1["Single-tenant SaaS"]
        D2["Managed by GitLab"]
        D3["Isolated infrastructure"]
        D4["✅ Compliance friendly"]
        D5["✅ No infra overhead"]
        D6["❌ Higher cost"]
    end

    style SAAS fill:#3498db,color:#fff
    style SM fill:#e67e22,color:#fff
    style DED fill:#9b59b6,color:#fff
```

---

## Usage Notes

- Diagram 1 and 2 are primary teaching diagrams for Module 2
- Diagram 3 (Fragmented vs Unified) is the most effective whiteboard diagram for generating discussion
- Diagram 4 (Tier Map) used when explaining why Ultimate is required for security features
- Diagram 5 (Runner Executors) referenced again in Module 5 (CI/CD Fundamentals) and Module 15 (IoT/Robotics)
- Diagram 6 (Deployment Models) used when organisations ask about air-gapped/regulated deployments
