# INSTRUCTOR HANDBOOK — MODULE 6
## Enterprise Delivery Pipelines: Environments, Promotion, and Approval Gates

---

### Module Metadata

| Field          | Value                                                       |
|----------------|-------------------------------------------------------------|
| Module Number  | 6 of 16                                                     |
| Day            | Day 1                                                       |
| Duration       | 75 minutes (25 min content + 50 min lab)                   |
| Difficulty     | Intermediate–Advanced                                       |
| Prerequisites  | Module 5 complete; working `.gitlab-ci.yml` from Lab 5     |
| Lab Produces   | Complete multi-env pipeline used in Day 2 security labs    |

---

## Instructor Preparation Notes

### Mental Model to Transfer
An enterprise delivery pipeline is a **risk reduction mechanism**. The movement from Development → QA → Staging → Production is not bureaucracy. It is a controlled exposure ladder — each environment serves as a test of increasing fidelity. Changes that fail at lower environments never reach production. Manual approval gates are the points where **human judgement** supplements automated testing.

### Key Insight to Drive Home
> "The manual approval gate before production is not a sign of low DevOps maturity. It is a **deliberate policy decision** that says: 'Automated tests are necessary but not sufficient for production confidence.' Elite organisations have manual gates — they just make them fast (minutes, not days) and evidence-based (approving on dashboard data, not intuition)."

### Connection Forward to Security
The pipeline built in this module is the foundation for Day 2. In Module 10, participants will add SAST to this pipeline. In Module 11, Secret Detection and Dependency Scanning. In Module 12, Container Scanning. The final capstone lab extends this pipeline to the full DevSecOps pipeline. Make this connection explicit: "We are building the chassis now. In Day 2, we install the security systems."

---

## Learning Objectives

1. Explain GitLab Environments and their role in the delivery pipeline.
2. Build a multi-stage pipeline with environment promotion (Dev → QA → Production).
3. Configure a manual approval gate that blocks automatic promotion to production.
4. Implement deployment strategies: direct, blue/green, canary (conceptual).
5. Use GitLab Environments to track deployment history and enable rollback.
6. Understand Release governance and how to create GitLab Releases from the pipeline.

---

## Business Context

### Why Environment Promotion Matters

**The Production Incident Pattern:**
Most production incidents follow this pattern: a change that was tested in a low-fidelity environment (developer laptop, basic CI) behaved differently in production due to environment-specific configuration (database connections, network topology, secrets, load). Environment promotion — testing in progressively higher-fidelity environments before production — reduces this.

**The Change Failure Rate (DORA):**
DORA's Change Failure Rate measures the percentage of changes that require a hotfix or rollback after deployment. Elite performers: 0–5%. Low performers: 46–60%. Environment promotion and manual gates are key contributors to low Change Failure Rate.

**For IoT/Firmware Teams:**
The "environments" concept maps directly to firmware deployment tiers:
- **Development** — Developer bench device
- **QA** — Validated hardware test rig
- **Field Pilot** — 10 devices in controlled field test
- **Production** — Full OTA rollout to device fleet

GitLab Environments can model each tier with approval gates before OTA rollout.

---

## Module Content — Detailed Teaching Guide

---

### SECTION 1: GitLab Environments (8 minutes)

**What a GitLab Environment Is:**
An Environment in GitLab is a named target for deployment. It is NOT an infrastructure resource — GitLab does not create the environment. It is a **tracking record** that associates a pipeline deployment with a named environment.

Source: https://docs.gitlab.com/ee/ci/environments/

**What Environments Track:**
- Current deployed version (which commit/tag is deployed)
- Deployment history (timeline of all deployments)
- Environment URL (link to the running service)
- Deployment status (running, stopped, in progress)
- Who deployed what, when

**Environment Types:**
```yaml
# Static environment — always exists
deploy-production:
  environment:
    name: production
    url: https://api.robovision.io

# Dynamic environments — created per branch/MR (review apps)
deploy-review:
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_COMMIT_REF_SLUG.review.robovision.io
    on_stop: stop-review
```

**Environment Scoped Variables:**
Variables can be scoped to specific environments:
```
Variable: DATABASE_URL
  - Scope: production → value: postgresql://prod.db:5432/robovision
  - Scope: staging    → value: postgresql://staging.db:5432/robovision
  - Scope: *          → value: postgresql://localhost:5432/robovision_dev
```
Source: https://docs.gitlab.com/ee/ci/environments/#environment-variables-and-environments

**Protected Environments:**
When an environment is set as "protected," only users with the Deployment role can deploy to it. This prevents low-privilege users from deploying to production.
Source: https://docs.gitlab.com/ee/ci/environments/protected_environments.html

---

### SECTION 2: Manual Approval Gates (7 minutes)

**The `when: manual` Job:**
```yaml
deploy-production:
  stage: deploy-prod
  when: manual          # Pipeline pauses here; requires human to click Play
  script:
    - ./deploy.sh production
  environment:
    name: production
```

**What Happens at a Manual Gate:**
1. Pipeline runs automatically through all preceding stages
2. Pipeline pauses at the manual job — shows a ▶️ Play button in the UI
3. Only users with Developer+ role (or configured restriction) can click Play
4. Clicking Play triggers the job; a record is created of who approved, when
5. The approval is logged in GitLab Audit Events

**Who Can Click Play:**
By default, any Developer+ role user can trigger a `when: manual` job. For production gates, restrict this via Protected Environments:
```
Settings → CI/CD → Protected Environments → production
  → Allowed to deploy: [specific users, groups, or roles]
```

**Multi-Approval Pattern:**
GitLab does not have native multi-person approval for manual jobs (as of GitLab 17.x). Workarounds:
1. Use Merge Request approval rules as the gate — promote only from approved MRs
2. Use GitLab's Deployment Approvals (Ultimate) — requires explicit approval from multiple users before pipeline proceeds
   Source: https://docs.gitlab.com/ee/ci/environments/deployment_approvals.html

**Deployment Approvals (Ultimate):**
In GitLab Ultimate, Protected Environments support **Deployment Approvals** — require N approvals from a group before deployment proceeds:
```
Protected Environment: production
  Required approval rules:
    - Rule: 2 approvals from group 'senior-engineers'
    - Rule: 1 approval from group 'security-team'
```
This is the production-grade multi-approval gate.

---

### SECTION 3: Deployment Strategies (5 minutes)

**Direct (Recreate) Deployment:**
Stop old version, start new version. Simple but causes downtime.
```
Before: [v1.0] running
Deploy: Stop v1.0, start v1.1
After:  [v1.1] running
Downtime: Yes (between stop and start)
```

**Rolling Deployment:**
Replace instances one at a time. No downtime if N>1 instances.
```
Before: [v1.0][v1.0][v1.0]
Step 1: [v1.1][v1.0][v1.0]
Step 2: [v1.1][v1.1][v1.0]
Step 3: [v1.1][v1.1][v1.1]
```
GitLab: Kubernetes deployment rolling update handles this natively.

**Blue/Green Deployment:**
Maintain two identical environments (blue=current, green=new). Switch traffic atomically.
```
Blue:  [v1.0] ← 100% traffic
Green: [v1.1] ← 0% traffic (warming up)

Switch: Blue ← 0% / Green ← 100%
Rollback: Switch back (takes seconds)
```

**Canary Deployment:**
Route small percentage of traffic to new version. Gradually increase.
```
v1.0 ← 95% traffic
v1.1 ← 5% traffic (canary)
  → Monitor: error rates, latency
  → Good: increase to 10%, 25%, 50%, 100%
  → Bad: route 0% to canary, rollback
```
GitLab supports canary deployments via Kubernetes canary annotations.
Source: https://docs.gitlab.com/ee/user/project/canary_deployments.html

**For IoT/Firmware Teams:**
Canary = staged OTA rollout. Deploy new firmware to 1% of devices. Monitor telemetry. Expand or rollback based on device health data.

---

### SECTION 4: GitLab Releases (5 minutes)

**What a GitLab Release Is:**
A Release in GitLab is a snapshot of the project at a point in time. It includes:
- A Git tag
- Release notes (auto-generated from merge commit messages)
- Associated artifacts (binaries, packages, firmware images)
- Links to deployment environments

Source: https://docs.gitlab.com/ee/user/project/releases/

**Creating Releases from the Pipeline:**
```yaml
create-release:
  stage: release
  image: registry.gitlab.com/gitlab-org/release-cli:latest
  script:
    - echo "Creating release $CI_COMMIT_TAG"
  release:
    name: "RoboVision $CI_COMMIT_TAG"
    description: "Release $CI_COMMIT_TAG — see CHANGELOG.md"
    tag_name: $CI_COMMIT_TAG
    assets:
      links:
        - name: "Docker Image"
          url: "$CI_REGISTRY_IMAGE:$CI_COMMIT_TAG"
  rules:
    - if: '$CI_COMMIT_TAG'    # Only run on tag pipelines
```

**For Firmware Teams:**
Attach firmware binaries as release assets:
```yaml
  release:
    assets:
      links:
        - name: "Firmware Image v$CI_COMMIT_TAG"
          url: "$CI_PROJECT_URL/-/jobs/$CI_JOB_ID/artifacts/raw/firmware.bin"
          link_type: package
```

---

## Lab Guide — Enterprise Pipeline (50 minutes)

> Full lab in M06_Lab_Workbook.md

**Lab Builds:**
Extending the Module 5 pipeline to a full multi-environment delivery:
```
validate → test → build → deploy-dev → deploy-qa → [MANUAL GATE] → deploy-prod
```

With:
- Environment scoped variables (DB_URL per environment)
- Protected production environment (Maintainers only)
- Deployment history tracking
- Manual approval gate
- Rollback demonstration

---

## Failure Modes and Anti-Patterns

| Anti-Pattern | Description | Consequence |
|---|---|---|
| No environment tracking | `deploy` job runs scripts but no `environment:` keyword | No deployment history; no rollback reference; no audit trail |
| Unprotected production environment | Any Developer can deploy to production | Accidental or malicious production deployments |
| Manual gate with no timeout | `when: manual` jobs block pipeline indefinitely | Pipelines queue up; stale deployments in QA |
| Environment variable sprawl | Production credentials in `*` scope instead of production scope | All environments get production credentials; isolation broken |
| Skip QA gate under pressure | `allow_failure: true` on QA deploy | Unverified code reaches production |
| No rollback plan | Deployment pipeline with no rollback job or procedure | Incidents last longer because recovery is slow |

---

## Knowledge Check Questions

**L1:** "What is the difference between a GitLab Environment and an actual deployment environment?"

**L2:** "A developer changes an environment-scoped variable from `staging` scope to `*` scope. What is the security consequence?"

**L3:** "Your production deployment job has `when: manual`. The QA deployment job failed. What happens to the production manual job?"

**L4:** "Design a deployment pipeline for a firmware update to 50,000 IoT devices. What environments would you define, what manual gates would you require, and how would you implement a canary rollout?"

---

## References

1. **GitLab Environments** — https://docs.gitlab.com/ee/ci/environments/
2. **Protected Environments** — https://docs.gitlab.com/ee/ci/environments/protected_environments.html
3. **Deployment Approvals** — https://docs.gitlab.com/ee/ci/environments/deployment_approvals.html
4. **GitLab Releases** — https://docs.gitlab.com/ee/user/project/releases/
5. **Canary Deployments** — https://docs.gitlab.com/ee/user/project/canary_deployments.html
6. **DORA Change Failure Rate** — https://dora.dev/research/
7. **GitLab Environment Variables Scoping** — https://docs.gitlab.com/ee/ci/environments/#environment-variables-and-environments
