# INSTRUCTOR HANDBOOK — MODULE 2
## GitLab Ultimate Overview: The Single Application Platform

---

### Module Metadata

| Field          | Value                                              |
|----------------|----------------------------------------------------|
| Module Number  | 2 of 16                                            |
| Day            | Day 1                                              |
| Duration       | 60 minutes (40 min content + 20 min demo/discussion)|
| Difficulty     | Foundational — platform context for all later modules |
| Prerequisites  | Module 1 completed                                 |

---

## Instructor Preparation Notes

### Mental Model to Transfer
Participants must exit this module with one clear idea: **GitLab Ultimate is not a collection of features — it is a unified data model over the entire software delivery lifecycle.** Every event in GitLab — a commit, a pipeline run, a security finding, a deployment — is correlated in a single database. This is what makes cross-cutting capabilities (security policy enforcement, compliance reporting, vulnerability management) possible without custom integrations.

### The Contrast That Makes This Land
The contrast partner for GitLab Ultimate is "the toolchain" — Jenkins + GitHub + SonarQube + Jira + JFrog + Vault + ArgoCD. Don't just list the names. Ask participants to estimate the integration cost: authentication sync, API versioning, webhook maintenance, audit log aggregation, user provisioning across 8+ tools. That cost is the value proposition.

### Calibration for This Audience
AI/ML Engineers often use GitHub because that's where models and datasets live. Robotics/Embedded teams often use legacy SCM (Gerrit, SVN in some cases) or basic GitHub. IoT developers may have separate OTA firmware pipeline tools entirely disconnected from their application CI/CD. Surface these realities. GitLab's value is not "use our tools" — it's "here is a single place where your firmware pipeline, your model training CI, and your cloud API security scanning all produce a unified audit trail."

---

## Learning Objectives

By end of this module participants will be able to:

1. Describe GitLab's architecture as a single-application platform and explain what that means mechanically.
2. Differentiate the core capabilities of GitLab Free, Premium, and Ultimate tiers.
3. Articulate why organisations consolidate from multi-tool toolchains to a unified platform.
4. Identify the components of a GitLab self-managed deployment.
5. Explain GitLab's licensing model and where Ultimate-tier features are relevant to security.

---

## Business Context

### The Integration Tax
In a study by Harness (2022), developers spend an average of 40% of their time on tasks that are not writing code — including managing CI/CD pipelines, coordinating handoffs between tools, and debugging integrations. In organisations with 15+ point tools in their delivery stack, the integration maintenance burden alone can consume the equivalent of 2–3 full-time engineers per year.

More critically for this audience: **fragmented toolchains produce fragmented security**. A SonarQube scan that is not connected to the same audit trail as your deployment history cannot produce a security posture view. When a CVE is discovered in production, the question "which of our 400 services is affected and when did we last scan it?" cannot be answered by querying 15 different tools. It requires a human to aggregate. GitLab Ultimate's single data model makes this query instant.

---

## Module Content — Detailed Teaching Guide

---

### SECTION 1: GitLab Company and Vision (5 minutes)

**Key Facts (Verified against GitLab Documentation):**
- GitLab Inc. was founded in 2011 by Dmitriy Zaporozhets and Sytse "Sid" Sijbrandij.
- The GitLab application (the open-source project) was created in 2011 by Dmitriy Zaporozhets.
- GitLab is open-core: the core SCM/CI is open source (MIT license); premium features are proprietary.
- GitLab has ~30 million registered users and is used by more than 50% of Fortune 100 companies.
- Available as: GitLab.com (SaaS), GitLab Self-Managed, GitLab Dedicated (single-tenant SaaS).

Source: https://about.gitlab.com/company/

**The Mission Statement:**
> *"Everyone can contribute."* — GitLab's mission. The platform is designed to eliminate the barriers between those who create and those who deploy software.

**Teaching Point:**
> "GitLab was built remote-first and open-core from day one. That's architecturally significant: the entire development history of GitLab itself is publicly visible on gitlab.com/gitlab-org/gitlab. When we discuss transparency and auditability in the security modules, remember that GitLab practices what it sells."

---

### SECTION 2: The Single Application Concept (10 minutes)

**Core Architecture Insight:**
Most software delivery toolchains are composed of independent applications that communicate via webhooks and APIs. Each tool has its own:
- User authentication and RBAC
- Audit logging schema
- API versioning
- Data model
- Failure modes

GitLab's architectural claim is different:

> "GitLab is a single application for the entire software development lifecycle." — https://about.gitlab.com/platform/

**What "Single Application" Means Mechanically:**
1. **Single data model** — Issues, commits, pipelines, deployments, security findings, and audit events all live in the same PostgreSQL database. Correlating a security finding with the commit that introduced it and the pipeline that detected it requires no external integration.

2. **Single authentication** — One RBAC system covers source control, CI/CD, package registry, security scanning, and compliance reporting. A permission change propagates everywhere.

3. **Single API** — The GitLab API provides unified access to all platform capabilities. No API translation layer.

4. **Single audit trail** — Every event across the platform — commit, pipeline start/stop, security finding acknowledgment, approval, deployment — is recorded in a single audit log. Compliance reporting queries one source.

**Whiteboard Exercise:**
Draw two diagrams:

*Left: Fragmented toolchain*
```
GitHub [commit event] → Webhook → Jenkins [pipeline event] → Webhook → SonarQube [finding] 
→ Manual export → Jira [ticket] → Manual export → Compliance report
```

*Right: GitLab Single Application*
```
GitLab [commit event → pipeline event → security finding → issue creation → compliance report]
                All correlated in one database. One API. One query.
```

**Ask:** "How long does it take your organisation today to answer: 'Show me all code commits this quarter that introduced a critical security vulnerability and were deployed to production?' How many tools and people does that query touch?"

---

### SECTION 3: GitLab Tiers — Free vs Premium vs Ultimate (10 minutes)

**Instructor Note:** Keep this factual and concise. Do not dwell on pricing. The goal is to establish which capabilities require Ultimate so participants understand why the workshop targets that tier. Verified against: https://about.gitlab.com/pricing/

**Tier Comparison (Security-Focused):**

| Capability                          | Free | Premium | Ultimate |
|-------------------------------------|------|---------|---------|
| Source Control (Git)                | ✅   | ✅      | ✅      |
| CI/CD Pipelines                     | ✅   | ✅      | ✅      |
| Container Registry                  | ✅   | ✅      | ✅      |
| Static Application Security Testing (SAST) | ✅ | ✅ | ✅ (advanced) |
| Secret Detection                    | ✅   | ✅      | ✅ (advanced) |
| Dependency Scanning (SCA)           | ❌   | ❌      | ✅      |
| Dynamic Application Security Testing (DAST) | ❌ | ❌ | ✅ |
| Container Scanning                  | ❌   | ❌      | ✅      |
| IaC Security Scanning               | ❌   | ❌      | ✅      |
| License Compliance                  | ❌   | ❌      | ✅      |
| Security Dashboard                  | ❌   | ❌      | ✅      |
| Vulnerability Management            | ❌   | ❌      | ✅      |
| Security Policies                   | ❌   | ❌      | ✅      |
| SBOM / Dependency List              | ❌   | ❌      | ✅      |
| Compliance Frameworks               | ❌   | ❌      | ✅      |
| Multi-level Merge Request Approvals | ❌   | ✅      | ✅      |
| SAML/SSO                            | ❌   | ✅      | ✅      |
| Audit Events                        | ❌   | ✅      | ✅ (extended) |

Source: https://about.gitlab.com/pricing/feature-comparison/

**Teaching Point:**
> "GitLab Free gives you a highly capable SCM and CI/CD platform. Premium adds governance and enterprise integration features. Ultimate adds the full DevSecOps stack. For this workshop, we are operating in Ultimate because the security capabilities — DAST, Container Scanning, Dependency Scanning, Security Policies — are Ultimate-tier features. These are the capabilities that directly address the failure modes we identified in Module 1."

**Note on SAST in Free Tier:**
Basic SAST (Semgrep-based) is available in GitLab Free. Advanced SAST (formerly Semgrep with proprietary rulesets) requires Ultimate. Be precise about this — trainers who say "SAST is Ultimate only" are factually incorrect and lose credibility.

Source: https://docs.gitlab.com/ee/user/application_security/sast/

---

### SECTION 4: GitLab Architecture Overview (15 minutes)

**Core Components (Verified: https://docs.gitlab.com/ee/development/architecture.html):**

| Component | Role |
|---|---|
| **Puma (Rails)** | The GitLab web application — handles HTTP requests, renders UI, exposes API |
| **Sidekiq** | Background job processing (pipeline triggers, emails, webhooks, security report processing) |
| **PostgreSQL** | Primary relational database — stores all GitLab data (users, projects, pipelines, security findings) |
| **Redis** | Cache layer, session storage, Sidekiq job queue |
| **Gitaly** | Git repository storage service — provides an RPC interface to Git operations, runs on same or separate nodes |
| **GitLab Shell** | Handles SSH operations for Git (clone, push over SSH) |
| **GitLab Workhorse** | Reverse proxy that handles large file uploads/downloads (LFS, artifacts) before they reach Puma |
| **GitLab Runner** | Separate process that executes CI/CD pipeline jobs — can be registered on any infrastructure |
| **Minio / Object Storage** | Artifact storage (S3-compatible; GitLab supports AWS S3, GCS, Azure Blob, MinIO) |
| **Elasticsearch / OpenSearch** | Optional — enables advanced code search, security dashboard search |
| **Prometheus** | Metrics collection (GitLab ships with Prometheus integration) |
| **Grafana** | Metrics dashboards (optional, integrated with Prometheus) |

**The Request Flow:**
Trace this on the whiteboard or slide:

```
Developer Push (SSH/HTTPS)
    ↓
GitLab Shell (SSH) / GitLab Workhorse (HTTPS)
    ↓
Gitaly (Git operations on disk)
    ↓
Puma (Rails application — creates pipeline record in PostgreSQL)
    ↓
Sidekiq (background job: triggers pipeline, notifies Runners)
    ↓
GitLab Runner (picks up job, executes on Executor)
    ↓
Job artifacts uploaded → Object Storage (via Workhorse)
    ↓
Results stored in PostgreSQL (test results, security findings)
    ↓
Developer sees results in GitLab UI (Puma serves from PostgreSQL/Redis)
```

**Why Gitaly Exists (Internal Mechanics):**
> Gitaly was introduced because GitLab scaled to millions of repositories. Direct NFS access to git repositories at scale produces severe performance problems (NFS locking, latency). Gitaly wraps all git operations in a gRPC service, enabling:
> - Horizontal scaling of git storage
> - Per-operation access control
> - Distributed storage (Gitaly Cluster / Praefect for HA)

**Why GitLab Runner is Separate:**
> GitLab Runner was intentionally designed as a separate component because:
> - Security isolation — pipeline code executes on separate infrastructure
> - Scalability — Runner fleet scales independently of the GitLab application
> - Flexibility — Runners can be registered on any infrastructure (cloud, on-premises, air-gapped)
> - Technology diversity — Shell, Docker, Kubernetes, VirtualBox, custom executors

Source: https://docs.gitlab.com/runner/

---

### SECTION 5: Deployment Models (5 minutes)

| Model | Description | When to Use |
|---|---|---|
| **GitLab.com (SaaS)** | Hosted by GitLab Inc. — multi-tenant | Fastest to adopt; no infrastructure management |
| **GitLab Self-Managed** | Hosted on your infrastructure | Data residency, air-gapped, maximum control |
| **GitLab Dedicated** | Single-tenant SaaS hosted by GitLab | Compliance requirements + managed operations |

**For IoT/Robotics Audience:**
> "For teams building firmware for air-gapped devices or operating in regulated industries (defence, medical, automotive), Self-Managed is typically the deployment model. You control the infrastructure, the backup, the version. The trade-off is operational overhead. We will explore how Runners can be deployed to edge infrastructure in Module 15."

---

### SECTION 6: Why Organisations Consolidate (5 minutes)

**The Five Consolidation Drivers:**

1. **Security Posture** — Unified security scanning with a single security dashboard. No vulnerability data falling through integration gaps.

2. **Compliance Auditability** — Single audit trail. "Who approved this deployment?" is one query, not a cross-tool investigation.

3. **Developer Experience** — One login, one UI, one CLI. New engineer onboarding goes from "learn 8 tools" to "learn GitLab".

4. **Operational Cost** — Fewer integration maintenance contracts. Fewer API version upgrades. Fewer tool-specific outages.

5. **Governance Velocity** — Security policies defined once, enforced everywhere. Compliance frameworks attached to groups, not individual projects.

**The Honest Trade-Off:**
> "GitLab does not always have the deepest feature in every category. Jira has more PM workflow customisation. GitHub has a larger open-source community. Checkmarx has deeper language-specific SAST rules. The trade-off is: breadth and integration depth vs. point-tool depth. For most enterprise DevSecOps programmes, the integration benefit outweighs the feature depth delta."

---

## Demo Guide (20 minutes)

### Live GitLab.com Demo — Orientation Walk-Through

> **Prerequisites:** Instructor has a GitLab.com account and an Ultimate trial enabled, OR GitLab Self-Managed with Ultimate licence.

**Demo Sequence:**

1. **Project Structure** (3 min)
   - Navigate to a sample project
   - Show: repository, issues, merge requests, pipelines, security tab — all in one navigation
   - Point: "Everything you need — from code to security findings — in one left nav bar"

2. **Security Dashboard** (4 min)
   - Navigate to Security → Security Dashboard
   - Show: vulnerability counts by severity, by scanner type
   - Navigate to one finding: severity, CWE, affected file, remediation guidance
   - Point: "This is what the integration gives you — a single view across SAST, DAST, SCA, Container Scan"

3. **Pipeline Visualization** (4 min)
   - Navigate to CI/CD → Pipelines → Click a pipeline
   - Show: pipeline graph with stages and jobs
   - Show: Security jobs running in the pipeline (sast, dependency_scanning, etc.)
   - Point: "Security is not a separate process — it is a stage in the same pipeline"

4. **Merge Request Integration** (4 min)
   - Navigate to a Merge Request with security findings
   - Show: security findings widget in MR view
   - Point: "The developer sees security findings in the same view as code review — no context switch"

5. **Audit Events** (2 min)
   - Navigate to Admin → Audit Events (or Group → Compliance → Audit Events)
   - Show: event log of pipeline runs, approvals, deployments
   - Point: "Single audit trail across all platform events"

6. **GitLab Runner Registration** (3 min)
   - Navigate to Settings → CI/CD → Runners
   - Show: registered runners, executor type
   - Point: "Runners are the execution layer — completely separate from the GitLab application. Can be on any infrastructure."

---

## Discussion Questions (Built Into Demo)

1. "Looking at this security dashboard — in your current organisation, how would you generate this view today? How many tools would it require?"

2. "The Merge Request security widget shows findings before code is merged. What does your current process for getting security findings to developers look like?"

3. "Every event in that audit log was generated automatically — no manual log entry. What would it take to produce an equivalent audit trail with your current toolchain?"

---

## Failure Modes and Anti-Patterns

| Anti-Pattern | Description | Consequence |
|---|---|---|
| Ultimate Features Unused | Organisation purchases Ultimate but only uses CI/CD; security scanning never enabled | Cost without benefit; security posture unchanged |
| Runners on Wrong Infrastructure | Runners registered on developer laptops or shared infrastructure with production access | Security isolation broken; pipeline code can access production secrets |
| Default Branch Protection Off | Main branch unprotected; direct commits bypassing MR review | CODEOWNERS/approval rules ineffective |
| Security Findings Suppressed | Teams configure `allow_failure: true` on security jobs to avoid pipeline failure | Green pipeline hides critical vulnerabilities |
| Single Admin Account | Entire GitLab instance managed by a single admin account | RBAC security control failure; blast radius unlimited |

---

## Knowledge Check Questions

**L1 (Motivation):**
> "What problem does GitLab's 'single application' architecture solve that a multi-tool toolchain cannot solve easily?"

**L2 (Mechanism):**
> "Describe what happens inside GitLab after a developer pushes a commit — trace through at least four internal components."

**L3 (Mental Model):**
> "If an organisation wants to generate a compliance report showing every deployment this quarter that was approved by two reviewers and passed all security scans — which GitLab components make that query possible?"

**L4 (Mastery):**
> "An organisation is running GitLab.com Free. Their CISO requires dependency scanning on all projects. What is the minimum action required? What are the alternatives if migrating to Ultimate is not immediately possible?"

---

## References and Citations

1. **GitLab Architecture Overview** — https://docs.gitlab.com/ee/development/architecture.html
2. **GitLab Pricing and Feature Comparison** — https://about.gitlab.com/pricing/feature-comparison/
3. **GitLab Runner Documentation** — https://docs.gitlab.com/runner/
4. **Gitaly Documentation** — https://docs.gitlab.com/ee/administration/gitaly/
5. **GitLab SAST Documentation** — https://docs.gitlab.com/ee/user/application_security/sast/
6. **GitLab Single Application Platform** — https://about.gitlab.com/platform/
7. **GitLab Deployment Options** — https://about.gitlab.com/install/
8. **GitLab Security Dashboard** — https://docs.gitlab.com/ee/user/application_security/security_dashboard/
9. **GitLab Audit Events** — https://docs.gitlab.com/ee/administration/audit_events.html
