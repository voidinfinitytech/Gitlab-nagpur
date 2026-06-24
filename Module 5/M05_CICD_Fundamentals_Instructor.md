# INSTRUCTOR HANDBOOK — MODULE 5
## GitLab CI/CD Fundamentals: Pipelines, Stages, Jobs, Runners, Artifacts

---

### Module Metadata

| Field          | Value                                              |
|----------------|----------------------------------------------------|
| Module Number  | 5 of 16                                            |
| Day            | Day 1                                              |
| Duration       | 75 minutes (30 min content + 45 min lab)          |
| Difficulty     | Intermediate                                       |
| Prerequisites  | Modules 1–4; Docker basics                        |
| Lab Requires   | GitLab with a registered Runner (Docker executor) |

---

## Instructor Preparation Notes

### Mental Model to Transfer
A GitLab CI/CD pipeline is a **directed graph of jobs** executed by Runners. The `.gitlab-ci.yml` file is a **declarative specification** — you describe what you want, GitLab orchestrates when and where it runs. Every job runs in an **isolated environment** (container), receives its context from GitLab (variables, cloned repo), produces **artifacts**, and reports a result.

The core insight participants must internalise:
> "The pipeline is not a script that runs on your machine. It is a declarative specification that GitLab resolves into jobs, assigns to Runners, and executes in isolated environments — on infrastructure you control."

This distinction matters for security: you can run sensitive security scans on dedicated Runners with restricted network access, entirely separate from build Runners.

### Opening Failure Story
> "A startup had a CI pipeline that worked perfectly for 6 months. Then a developer updated the Docker base image used for builds. The new image included a `curl` to an external endpoint in its entrypoint. Every pipeline job was exfiltrating environment variables (including AWS credentials stored as CI variables) to an attacker-controlled server. They lost access to their entire AWS account. The lesson: CI variables are secrets — Runner isolation and image provenance matter enormously."

This story establishes: (1) why environment isolation matters; (2) why you should use trusted, scanned base images; (3) why CI variables need careful management. All covered in this module.

---

## Learning Objectives

1. Explain the relationship between pipeline, stages, and jobs in GitLab CI.
2. Write a `.gitlab-ci.yml` file defining multi-stage pipelines with real dependencies.
3. Distinguish between GitLab Runner executor types and their security implications.
4. Use artifacts to pass build outputs between pipeline stages.
5. Manage CI/CD variables and understand the difference between protected, masked, and raw variables.
6. Diagnose common CI pipeline failures using job logs and pipeline visualisation.

---

## Business Context

### Why CI/CD Mastery Enables DevSecOps

Every DevSecOps capability in Day 2 (SAST, Dependency Scanning, Container Scanning) is implemented as **a CI job in the pipeline**. If participants don't understand how jobs, stages, artifacts, and variables work, the security module labs will fail in unexpected ways.

More importantly: the pipeline is the **enforcement point** for security policy. If the pipeline is misconfigured, security scanning is silently bypassed. Understanding pipeline mechanics is a prerequisite for understanding why a security gate can be circumvented.

**For Robotics/Embedded Teams:** The CI pipeline is the mechanism for:
- Cross-compilation (build firmware on x86, deploy to ARM)
- Hardware-in-the-loop testing triggers
- OTA package generation
- Signed firmware artifact creation

The fundamentals here directly support Module 15 (IoT/Robotics pipelines).

---

## Module Content — Detailed Teaching Guide

---

### SECTION 1: The `.gitlab-ci.yml` File (5 minutes)

**Location and Discovery:**
GitLab looks for `.gitlab-ci.yml` in the **root of the repository** by default. Custom paths can be configured in `Settings → CI/CD → General → CI/CD configuration file`.

Source: https://docs.gitlab.com/ee/ci/yaml/

**The Declarative Model:**
Unlike Jenkins (imperative Groovy scripts), GitLab CI/CD is **declarative** — you describe the desired state, GitLab resolves execution order and parallelism.

**Validation:**
Before pushing, validate YAML with:
- GitLab Web IDE (real-time validation)
- GitLab CI Lint tool: `Project → CI/CD → Pipelines → CI Lint`
- GitLab API: `POST /api/v4/projects/:id/ci/lint`

Source: https://docs.gitlab.com/ee/ci/yaml/lint.html

---

### SECTION 2: Stages and Jobs (10 minutes)

**Pipeline Structure:**
```yaml
stages:
  - build       # Stage 1: all build jobs run first
  - test        # Stage 2: all test jobs run after all build jobs pass
  - package     # Stage 3: runs after all test jobs pass
  - deploy      # Stage 4: runs after all package jobs pass
```

**Key Rules:**
1. Jobs in the **same stage** run **in parallel** (if Runners available)
2. Jobs in the **next stage** run only when **all jobs in the current stage pass**
3. A stage with a failing job **stops the pipeline** (unless `allow_failure: true`)

**Job Anatomy:**
```yaml
job-name:
  stage: test               # Which stage this job belongs to
  image: python:3.11        # Docker image to use (Docker executor)
  before_script:            # Commands run before script
    - pip install -r requirements.txt
  script:                   # Main job commands (required)
    - pytest tests/ -v
  after_script:             # Commands run after script (even on failure)
    - echo "Tests complete"
  artifacts:                # Files to preserve after job
    paths:
      - test-results/
    expire_in: 1 week
  rules:                    # Conditions for running this job
    - if: '$CI_COMMIT_BRANCH == "main"'
  variables:                # Job-specific environment variables
    PYTEST_TIMEOUT: "30"
  tags:                     # Runner tags — select specific Runner
    - docker
    - linux
```

**`before_script` vs `script` vs `after_script`:**
- `before_script`: Setup steps. If this fails, job fails and `script` doesn't run
- `script`: Main job logic. Failure here fails the job
- `after_script`: Cleanup/notification. Runs even on failure. Exit code ignored

---

### SECTION 3: Runners and Executors (8 minutes)

**Runner Types:**
Source: https://docs.gitlab.com/runner/

1. **Shared Runners** (GitLab.com) — available to all projects; managed by GitLab
2. **Group Runners** — available to all projects in a group
3. **Project Runners** — available only to specific projects

**When to Use Project-Specific Runners:**
- Security-sensitive jobs (SAST, secret scanning) — isolate to dedicated Runner
- Hardware access (embedded build, physical test bench)
- Air-gapped environments
- Jobs requiring specific hardware (GPU for AI training, special build tools)

**Runner Tags:**
Tags route jobs to specific Runners:
```yaml
# Job runs only on a Runner tagged "security" — your dedicated security scanner
sast-scan:
  tags:
    - security
    - docker-privileged
```
Source: https://docs.gitlab.com/ee/ci/runners/configure_runners.html

**Security Implication of Shell Executor (Teach This):**
```
Shell Executor Risk:
  Pipeline job → runs directly on Runner host
  → can read Runner host files
  → can modify Runner host files
  → subsequent jobs inherit modifications

Docker Executor Protection:
  Pipeline job → runs in fresh Docker container
  → container destroyed after job
  → no state leaks between jobs
  → cannot access Runner host filesystem (unless Docker socket mounted)
```

**Privileged Mode Warning:**
Docker-in-Docker (DinD) requires `--privileged` mode. This gives the container ROOT on the host's Docker daemon. Only use on dedicated, hardened Runners.

---

### SECTION 4: Artifacts (7 minutes)

**Purpose:** Artifacts are files produced by a job that:
1. Are preserved after the job completes (uploaded to Object Storage)
2. Can be downloaded from the GitLab UI
3. Can be passed to downstream jobs in the pipeline

**Artifact Declaration:**
```yaml
build-job:
  script:
    - make build
    - mkdir -p dist
    - cp bin/app dist/
  artifacts:
    name: "build-${CI_COMMIT_SHORT_SHA}"
    paths:
      - dist/
      - build.log
    expire_in: 30 days
    when: always     # Upload artifacts even on job failure (useful for test reports)
```

**Passing Artifacts Between Stages:**
```yaml
stages:
  - build
  - test

build:
  stage: build
  script:
    - gcc main.c -o app
  artifacts:
    paths:
      - app         # This binary is available to downstream jobs

test:
  stage: test
  script:
    - ./app --test  # Uses artifact from build stage
  # Artifacts from previous stages are automatically available
```

**Security Relevance:**
Security scan results are artifacts:
```
SAST job produces:    gl-sast-report.json      (artifact)
SCA job produces:     gl-dependency-scanning-report.json (artifact)
Container scan:       gl-container-scanning-report.json  (artifact)
```
These are uploaded to Object Storage and ingested by Sidekiq into the security dashboard. Understanding artifacts explains *how* security findings reach the dashboard.

Source: https://docs.gitlab.com/ee/ci/yaml/#artifacts

**Reports Artifacts (Special Type):**
```yaml
sast:
  artifacts:
    reports:
      sast: gl-sast-report.json    # Parsed by GitLab security dashboard
```
The `reports:` key is special — GitLab parses these files and renders them in the MR security widget, rather than just storing them as downloadable files.

---

### SECTION 5: Variables and Secrets (10 minutes)

**Variable Types:**
Source: https://docs.gitlab.com/ee/ci/variables/

| Type | Source | Use Case |
|---|---|---|
| Predefined CI/CD Variables | GitLab automatically sets | `$CI_COMMIT_SHA`, `$CI_PROJECT_NAME`, `$CI_JOB_TOKEN` |
| Project/Group CI Variables | Settings → CI/CD → Variables | API keys, credentials, configuration |
| `.gitlab-ci.yml` Variables | Declared in YAML | Non-secret configuration values |
| Job variables | `variables:` key in job | Job-specific overrides |

**Critical Predefined Variables (Memorise These):**

| Variable | Value | Use Case |
|---|---|---|
| `CI_COMMIT_SHA` | Full commit hash | Tag artifacts, identify build |
| `CI_COMMIT_SHORT_SHA` | Short hash (8 chars) | Human-readable artifact naming |
| `CI_COMMIT_BRANCH` | Branch name | Conditional job execution |
| `CI_PROJECT_NAME` | Project name | Artifact naming |
| `CI_PROJECT_URL` | Project URL | Links in notifications |
| `CI_JOB_TOKEN` | Short-lived job token | Pull from package registry, clone repos |
| `CI_REGISTRY` | Container registry hostname | Push/pull images |
| `CI_REGISTRY_IMAGE` | Full image path | `$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA` |
| `CI_ENVIRONMENT_NAME` | Environment name | Deployment configuration |
| `CI_PIPELINE_SOURCE` | Trigger source | Differentiate push/schedule/API triggers |

**Variable Protection Levels:**
```
Protected:    Only available in pipelines for protected branches/tags
              Use for production credentials, deploy keys
              
Masked:       Value hidden in job logs
              Use for all secrets (API keys, passwords)
              Cannot be shorter than 8 chars; must match masking pattern
              
Raw:          Plain value, visible in logs
              Use only for non-sensitive configuration
```

**Security Rule (Absolute):**
> Never store secrets as plain variables in `.gitlab-ci.yml`. Secrets belong in GitLab CI/CD Variables (masked + protected). Anyone with repository read access can see `.gitlab-ci.yml`. Anyone who can clone the repo can see hardcoded secrets.

**CI_JOB_TOKEN — Least Privilege Pattern:**
```yaml
# DON'T: Use long-lived personal access token
deploy:
  script:
    - docker login -u $DOCKER_USERNAME -p $DOCKER_PAT  # ❌ PAT persists after job

# DO: Use CI_JOB_TOKEN (scoped, short-lived)
deploy:
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_JOB_TOKEN $CI_REGISTRY  # ✅ Expires with job
```

Source: https://docs.gitlab.com/ee/ci/jobs/ci_job_token.html

---

## Hands-On Lab (45 minutes)

> Full step-by-step in M05_Lab_Workbook.md

**Lab Summary:**
1. Create a Python Flask application in the `robovision-controller` project from Module 4
2. Write a `.gitlab-ci.yml` with: build, test, lint, package stages
3. Commit and trigger pipeline
4. Analyse logs for each stage
5. Review generated artifacts
6. Add a CI variable and use it in the pipeline
7. Introduce a test failure — observe pipeline behaviour
8. Fix the failure — verify pipeline recovery

---

## Failure Modes and Anti-Patterns

| Anti-Pattern | Description | Consequence |
|---|---|---|
| `allow_failure: true` on all jobs | Jobs fail silently; pipeline always "green" | Security scans bypass; defects ship to production |
| Secrets in `.gitlab-ci.yml` | Hardcoded API keys, passwords in YAML | Exposed to anyone with repo read access; visible in git history forever |
| Shell executor for security jobs | SAST/DAST runs on shared Shell executor | Scanner output accessible to other jobs; potential cross-contamination |
| Artifact `expire_in` not set | Artifacts accumulate indefinitely | Object Storage fills; performance degrades |
| `after_script` not used for cleanup | Sensitive files left in workspace | Leak to subsequent jobs or other pipeline runs |
| No pipeline for MRs | Pipeline only runs on push to main | Defects reach main without pre-merge testing |
| Variables not masked | Secrets visible in job logs | Log access = credential exposure |

---

## Knowledge Check Questions

**L1:**
> "Why does GitLab run jobs in the same stage in parallel? What is the implication for job dependencies?"

**L2:**
> "Trace exactly what happens between when a developer pushes a commit and when a Runner starts executing a job."

**L3:**
> "A pipeline has three stages: build, test, deploy. The test stage has 4 jobs. The third test job fails. What happens to the fourth test job? What happens to the deploy stage?"

**L4:**
> "A security scan job uses a Docker image pulled from Docker Hub. The scan passes. Three weeks later, the same image has been updated and now contains malware. How does GitLab allow you to protect against this?"

---

## References

1. **GitLab CI/CD YAML Reference** — https://docs.gitlab.com/ee/ci/yaml/
2. **GitLab Runner** — https://docs.gitlab.com/runner/
3. **GitLab CI/CD Variables** — https://docs.gitlab.com/ee/ci/variables/
4. **CI_JOB_TOKEN** — https://docs.gitlab.com/ee/ci/jobs/ci_job_token.html
5. **GitLab Artifacts** — https://docs.gitlab.com/ee/ci/yaml/#artifacts
6. **GitLab CI Lint** — https://docs.gitlab.com/ee/ci/yaml/lint.html
7. **Predefined Variables** — https://docs.gitlab.com/ee/ci/variables/predefined_variables.html
8. **GitLab Runner Executors** — https://docs.gitlab.com/runner/executors/
