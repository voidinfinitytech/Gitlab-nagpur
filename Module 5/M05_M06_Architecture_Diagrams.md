# ARCHITECTURE DIAGRAMS — MODULES 5 & 6
## CI/CD Fundamentals and Enterprise Delivery Pipelines
### Render at https://mermaid.live

---

## Diagram 1: GitLab CI/CD Pipeline Anatomy

```mermaid
graph LR
    subgraph PIPELINE ["Pipeline: triggered by git push"]
        direction LR
        
        subgraph S1 ["Stage: validate\n(parallel jobs)"]
            J1[lint-python]
        end
        
        subgraph S2 ["Stage: test\n(parallel jobs)"]
            J2[unit-tests]
            J3[integration-tests]
        end
        
        subgraph S3 ["Stage: build\n(parallel jobs)"]
            J4[build-image]
        end
        
        subgraph S4 ["Stage: package\n(parallel jobs)"]
            J5[generate-manifest]
        end
    end

    S1 -->|all pass| S2
    S2 -->|all pass| S3
    S3 -->|all pass| S4

    J2 -->|artifacts: test-results.xml| J5
    J4 -->|artifacts: IMAGE_TAG| J5

    style S1 fill:#3498db,color:#fff
    style S2 fill:#2ecc71,color:#000
    style S3 fill:#e67e22,color:#fff
    style S4 fill:#9b59b6,color:#fff
```

---

## Diagram 2: Job Execution Flow (Runner Perspective)

```mermaid
sequenceDiagram
    participant GL as GitLab (Puma)
    participant REDIS as Redis Queue
    participant SIDEKIQ as Sidekiq
    participant RUNNER as GitLab Runner
    participant DOCKER as Docker Daemon
    participant OBJ as Object Storage

    GL->>REDIS: Enqueue job payload
    REDIS->>SIDEKIQ: Dispatch job
    SIDEKIQ->>GL: Assign job to available Runner

    RUNNER->>GL: Poll: GET /api/v4/jobs/request
    GL-->>RUNNER: Job payload (image, script, variables, token)

    RUNNER->>DOCKER: Pull job image (python:3.11-slim)
    DOCKER-->>RUNNER: Image ready
    
    RUNNER->>GL: Clone repository (HTTPS + CI_JOB_TOKEN)
    GL-->>RUNNER: Repository code

    RUNNER->>DOCKER: Start container, run before_script
    RUNNER->>DOCKER: Run script commands
    RUNNER->>GL: Stream log output (real-time)
    
    DOCKER-->>RUNNER: Job complete (exit code)
    RUNNER->>OBJ: Upload artifacts (test-results.xml, coverage.xml)
    OBJ-->>RUNNER: Upload confirmed
    
    RUNNER->>GL: Report result: PASSED
    GL->>GL: Update pipeline job record in PostgreSQL
    Note over GL: CI_JOB_TOKEN expires — connection closed
```

---

## Diagram 3: Artifact Flow Between Stages

```mermaid
graph TB
    subgraph BUILD_STAGE ["build stage"]
        BJ["build-image job\n→ Produces: Docker image digest"]
        BJ -->|"artifact: image-digest.txt"| OBJ1[(Object Storage)]
    end

    subgraph TEST_STAGE ["test stage"]
        TJ["unit-tests job\n→ Produces: test-results.xml, coverage.xml"]
        TJ -->|"reports: junit, coverage"| OBJ2[(Object Storage)]
        TJ -->|"GitLab parses coverage %"| PIPE_UI[Pipeline UI\ncoverage badge]
    end

    subgraph PACKAGE_STAGE ["package stage"]
        PJ["generate-manifest job\n→ Reads artifact from build stage\n→ Produces: deployment-manifest.yaml"]
        OBJ1 -->|"downloaded automatically\nto workspace"| PJ
        PJ -->|"artifact: deployment-manifest.yaml"| OBJ3[(Object Storage)]
    end

    OBJ2 -->|"junit report parsed"| MR_WIDGET[MR Test Summary\nWidget]

    style BUILD_STAGE fill:#e67e22,color:#fff
    style TEST_STAGE fill:#2ecc71,color:#000
    style PACKAGE_STAGE fill:#9b59b6,color:#fff
```

---

## Diagram 4: CI/CD Variables — Security Hierarchy

```mermaid
graph TB
    subgraph INSECURE ["❌ INSECURE — Never Do This"]
        I1[".gitlab-ci.yml hardcoded:\nAPI_KEY: sk-abc123\nDB_PASS: mypassword"]
        I2["Visible to anyone\nwith repo read access.\nStored in git history.\nCannot be rotated easily."]
    end

    subgraph SECURE ["✅ SECURE — Variable Store"]
        direction TB
        V1["Settings → CI/CD → Variables"]
        V1 --> V2["MASKED ✅\nValue hidden in job logs\nAll secrets must be masked"]
        V1 --> V3["PROTECTED ✅\nOnly available on\nprotected branches/tags\nUse for production credentials"]
        V1 --> V4["ENVIRONMENT SCOPE ✅\nDifferent values\nper environment\nDev ≠ QA ≠ Prod"]
    end

    subgraph BEST ["✅ BEST — CI_JOB_TOKEN for Registry Auth"]
        T1["docker login -u $CI_REGISTRY_USER\n         -p $CI_JOB_TOKEN\n            $CI_REGISTRY"]
        T2["Token is:\n• Scoped to this job only\n• Expires on job completion\n• No long-lived credential"]
    end

    style INSECURE fill:#e74c3c,color:#fff
    style SECURE fill:#2ecc71,color:#000
    style BEST fill:#27ae60,color:#fff
```

---

## Diagram 5: Enterprise Pipeline — Dev → QA → Production

```mermaid
graph LR
    COMMIT["git push\nmain"] --> LINT["validate\nlint ✅"]
    LINT --> TEST["test\nunit-tests ✅"]
    TEST --> BUILD["build\ndocker-image ✅"]
    
    BUILD --> DEV["deploy-dev\n🔄 automatic\nEnv: development\nDB: dev-db"]
    DEV --> QA["deploy-qa\n🔄 automatic\nSmoke tests ✅\nEnv: qa\nDB: qa-db"]
    QA --> SUMMARY["pre-prod\nsummary ✅"]
    
    SUMMARY --> MANUAL{"deploy-production\n▶️ MANUAL GATE\n\nOnly Maintainers\ncan click Play"}
    MANUAL -->|"Approved"| PROD["production\n🏭 deployed\nEnv: production\nDB: prod-db\nAudit: ✅"]
    MANUAL -->|"Not approved\n(pipeline paused)"| HOLD[Pipeline waits\nno timeout limit]
    
    SUMMARY --> ROLLBACK["rollback-production\n▶️ MANUAL (emergency)"]
    ROLLBACK --> PREV["Re-deploys\nprevious stable\nimage tag"]

    style COMMIT fill:#3498db,color:#fff
    style MANUAL fill:#e74c3c,color:#fff
    style PROD fill:#2ecc71,color:#000
    style ROLLBACK fill:#e67e22,color:#fff
```

---

## Diagram 6: Environment Scoped Variables — How They Inject

```mermaid
sequenceDiagram
    participant DEV_JOB as deploy-dev job\n(environment: dev)
    participant QA_JOB as deploy-qa job\n(environment: qa)
    participant PROD_JOB as deploy-production job\n(environment: production)
    participant GL as GitLab Variable Store

    Note over GL: Variable: DATABASE_URL\n  dev scope    → postgresql://dev-db\n  qa scope     → postgresql://qa-db\n  production scope → postgresql://prod-db

    DEV_JOB->>GL: Request variables for environment: dev
    GL-->>DEV_JOB: DATABASE_URL = postgresql://dev-db
    DEV_JOB->>DEV_JOB: Uses dev database ✅

    QA_JOB->>GL: Request variables for environment: qa
    GL-->>QA_JOB: DATABASE_URL = postgresql://qa-db
    QA_JOB->>QA_JOB: Uses QA database ✅

    PROD_JOB->>GL: Request variables for environment: production
    GL-->>PROD_JOB: DATABASE_URL = postgresql://prod-db (MASKED)
    PROD_JOB->>PROD_JOB: Uses production database ✅

    Note over DEV_JOB, PROD_JOB: Same YAML, different behaviour\nbased on environment context.\nNo hardcoded environment logic in pipeline.
```

---

## Diagram 7: Deployment History and Rollback

```mermaid
graph TB
    subgraph HISTORY ["Environments → production → Deployment History"]
        D3["#3 — abc123 — v1.2.0\nDeployed: Today 14:32\nBy: senior-dev\n[currently deployed]"]
        D2["#2 — def456 — v1.1.0\nDeployed: Yesterday 11:05\nBy: tech-lead\n[▶️ Re-deploy] ← ROLLBACK TARGET"]
        D1["#1 — ghi789 — v1.0.0\nDeployed: Last week\nBy: tech-lead\n[▶️ Re-deploy]"]
    end

    D3 -->|"INCIDENT: bad behaviour detected"| ROLLBACK_DECISION{Rollback decision}
    ROLLBACK_DECISION -->|"Click Re-deploy on #2"| D2
    D2 -->|"Re-runs deploy-production\nwith def456 image tag"| STABLE["Production restored\nto v1.1.0\n✅ Stable"]

    style D3 fill:#e74c3c,color:#fff
    style D2 fill:#f39c12,color:#fff
    style STABLE fill:#2ecc71,color:#000
```

---

## Diagram 8: Deployment Strategy Comparison

```mermaid
graph LR
    subgraph RECREATE ["Recreate (Simple)"]
        R1["v1.0 → STOP"] --> R2["v1.1 → START"]
        R_NOTE["Downtime: YES\nRisk: Low\nComplexity: Low"]
    end

    subgraph ROLLING ["Rolling Update"]
        RO1["v1.0, v1.0, v1.0"] --> RO2["v1.1, v1.0, v1.0"] --> RO3["v1.1, v1.1, v1.0"] --> RO4["v1.1, v1.1, v1.1"]
        RO_NOTE["Downtime: NO\nRisk: Medium\nComplexity: Medium"]
    end

    subgraph BLUEGREEN ["Blue/Green"]
        BG1["Blue: v1.0 ← 100% traffic"] --> BG2["Green: v1.1 warming up\nBlue: v1.0 ← 100%"] --> BG3["Green: v1.1 ← 100%\nBlue: standby"]
        BG_NOTE["Downtime: NO\nRisk: Low (instant rollback)\nComplexity: High (2x infra)"]
    end

    subgraph CANARY ["Canary"]
        C1["v1.0 ← 100%"] --> C2["v1.0 ← 95%, v1.1 ← 5%"] --> C3["v1.0 ← 50%, v1.1 ← 50%"] --> C4["v1.1 ← 100%"]
        C_NOTE["Downtime: NO\nRisk: Very Low\nComplexity: High\nIdeal for IoT OTA rollout"]
    end

    style RECREATE fill:#e74c3c,color:#fff
    style ROLLING fill:#f39c12,color:#fff
    style BLUEGREEN fill:#2ecc71,color:#000
    style CANARY fill:#3498db,color:#fff
```
