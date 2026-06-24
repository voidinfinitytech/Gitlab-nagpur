# INSTRUCTOR HANDBOOK — MODULE 4
## Source Control and Governance: Repository Strategy, Merge Requests, and Branch Protection

---

### Module Metadata

| Field          | Value                                                  |
|----------------|--------------------------------------------------------|
| Module Number  | 4 of 16                                                |
| Day            | Day 1                                                  |
| Duration       | 75 minutes (35 min content + 40 min hands-on lab)     |
| Difficulty     | Intermediate — participants know Git; this covers governance patterns |
| Prerequisites  | Modules 1–3 completed; GitLab account accessible      |
| Lab Requires   | GitLab Ultimate instance (self-managed or GitLab.com trial) |

---

## Instructor Preparation Notes

### Mental Model to Transfer
Source control governance is not about Git commands. It is about answering the question:
> *"How do we ensure that no change reaches production without the right people having reviewed it, the right tests having run, and a complete audit trail existing for compliance?"*

Every mechanism in this module — protected branches, CODEOWNERS, approval rules, merge request workflows — is a specific answer to a specific failure mode. Teach the failure first. The participants know Git. They likely don't know why large engineering organisations have burned themselves by NOT having these controls.

### Opening Failure Story (Use This)
> "In 2020, a developer at a Fortune 500 company accidentally force-pushed to the main branch of their monorepo at 11pm Friday. The commit wiped three weeks of merged work. The repository had no branch protection, no merge request requirement, and no backup policy outside the repository itself. Recovery took 18 hours of engineering time over a weekend. The incident post-mortem listed: 'Implement branch protection rules' as a remediation action — something that would have taken 5 minutes to configure before the incident."

This story has two purposes: (1) establishes visceral motivation for branch protection; (2) illustrates that governance controls are not bureaucracy — they are recovery mechanisms.

### Calibration for Audience
- **AI Engineers**: Likely use GitHub for models and data versioning. Monorepo vs polyrepo tensions are real (model weights vs code). CODEOWNERS for model validation gates is a concrete use case.
- **Robotics/Embedded Engineers**: May use Gerrit or SVN. The shift to GitLab MR workflows is substantive. Hardware constraint files (CAD, URDF, config) as code requiring approval review is a relevant extension.
- **IoT Developers**: OTA firmware versioning strategy is directly relevant to branching strategy. Tagging and release branches for firmware version management.

---

## Learning Objectives

1. Compare monorepo and polyrepo strategies and articulate the trade-offs with supporting rationale.
2. Explain Trunk-Based Development and contrast it with Git Flow in terms of batch size and integration risk.
3. Configure GitLab Merge Request approval rules, including multi-level and CODEOWNERS-based approvals.
4. Implement protected branch rules that enforce code review, CI passing, and prevent force-push.
5. Use CODEOWNERS to map file ownership to approval requirements.
6. Articulate how these controls together create a compliance-ready audit trail.

---

## Business Context

### Why Source Control Governance Matters Beyond "Not Losing Code"

**SOC 2 Type II** requires evidence that production code changes are reviewed and approved. The audit control maps directly to:
- Merge Request approvals (who approved)
- Protected branch rules (only approved MRs can merge)
- Audit events (complete trail in GitLab)

**PCI DSS 6.3.2** requires maintaining an inventory of bespoke and custom software. GitLab's SBOM and dependency tracking (in later modules) depends on clean, governed repository structure.

**NIST SSDF PS.1.1** requires protecting all forms of code from unauthorized access and tampering — directly addressed by protected branches and approval rules.

**For Robotics/Automotive (ISO 26262, IEC 61508):**
Safety-critical software change management requires documented approvals and traceability from change request to implementation to verification. GitLab's MR + approval chain produces this trail automatically.

---

## Module Content — Detailed Teaching Guide

---

### SECTION 1: Repository Strategy — Monorepo vs Polyrepo (10 minutes)

#### The Origin Problem
In the early days of enterprise software, each team owned their repository independently. Code sharing happened through binary dependencies (JARs, DLLs, shared libraries). This created fragmentation: a security fix in a shared library required updates in 40 downstream repositories — each with their own release cycle.

**The Monorepo Motivation:**
Google, Facebook, Twitter, and Microsoft all operate on monorepos. Google's internal monorepo (Piper) reportedly contains ~2 billion lines of code across a single repository. The motivation: atomic cross-cutting changes. A security fix to a shared authentication library can be deployed as a single commit that also updates all 400 services that call it — verified by a single CI pipeline.

**Monorepo Advantages:**

| Advantage | Mechanism |
|---|---|
| Atomic cross-service changes | Single MR can modify library + all consumers |
| Unified dependency version | One lockfile; no diamond dependency problem |
| Single pipeline definition | One `.gitlab-ci.yml` driving all services |
| Simplified code search | `git grep` across entire codebase |
| Shared tooling and standards | One linter config, one security policy |

**Monorepo Disadvantages:**

| Disadvantage | At Scale |
|---|---|
| CI/CD performance | Every commit triggers full pipeline unless path filtering used |
| Repository clone time | Large repos are slow to clone on Runners |
| Access control granularity | Repository-level RBAC only (though CODEOWNERS mitigates) |
| Merge conflicts | High-frequency teams block each other |
| Tooling requirements | Requires path-based pipeline triggers (GitLab `changes:` rule) |

**Polyrepo Advantages/Disadvantages:**

| Polyrepo Advantage | Polyrepo Disadvantage |
|---|---|
| Team autonomy | Cross-service changes require N separate MRs |
| Independent CI/CD velocity | Dependency version drift (transitive conflicts) |
| Fine-grained access control | Security policy enforcement per-repo overhead |
| Smaller clone size | No atomic cross-cutting change |

**GitLab's Support for Both:**
GitLab supports monorepo patterns through:
- `rules: changes:` — trigger jobs only when specific paths change
- `include:` — reference shared `.gitlab-ci.yml` templates from a central repo
- DAG pipelines (`needs:`) — parallelise across services within one pipeline
- CODEOWNERS — file-level ownership within one repository

Source: https://docs.gitlab.com/ee/ci/yaml/#ruleschanges

**The Pragmatic Answer for Most Teams:**
> "Neither is universally right. The decision depends on: team size, service coupling, release cadence, and tooling maturity. IoT teams with tightly-coupled firmware + cloud services often benefit from monorepo — one pipeline, one version number, atomic release. AI teams with large model artifacts often prefer polyrepo — the model repository uses Git LFS or a model registry; the inference service is a separate repo. We will return to this in Module 15."

---

### SECTION 2: Branching Strategies (8 minutes)

#### Trunk-Based Development vs Git Flow

**Git Flow (High-Level Overview Only — this audience knows it):**
- Long-lived `main`, `develop`, `release/*`, `hotfix/*`, `feature/*` branches
- Release preparation on dedicated branches
- Merge back to `develop`, then to `main` via release

**The Problem with Git Flow at High Velocity:**
Long-lived feature branches accumulate **integration debt**. A branch that diverges from `main` for 3 weeks will have a painful merge. The larger the batch, the higher the integration risk. Git Flow was designed for infrequent releases (weekly/monthly). It is structurally incompatible with continuous deployment.

**Trunk-Based Development:**
All developers commit to a single trunk (`main`/`master`) at least once per day. Features are protected by **feature flags**, not branches. Releases are cut from trunk, not from release branches.

**Key Properties:**

| Property | Git Flow | Trunk-Based Development |
|---|---|---|
| Branch lifetime | Days to weeks | Hours to 1 day |
| Merge frequency | Low (at release) | High (daily) |
| Integration risk | High (big-bang merge) | Low (small increments) |
| CI feedback loop | Slow (branch may be stale) | Fast (always on current code) |
| Feature isolation | By branch | By feature flag |
| Release process | From release branch | From trunk + tag |
| DORA Lead Time | Long | Short |

**Source:** Google's DORA Research confirms Trunk-Based Development as a key practice in elite-performing organisations.  
Reference: https://dora.dev/capabilities/trunk-based-development/

**Practical Recommendation for This Workshop:**
> "For teams that cannot commit to TBD immediately, use a **simplified Git Flow**: `main` (production), `feature/*` (short-lived, < 3 days), `hotfix/*`. No long-lived `develop` branch. Every feature branch requires a Merge Request before merging to `main`. This gives you governance without the complexity of full Git Flow."

---

### SECTION 3: Merge Requests — The Governance Engine (8 minutes)

#### What a Merge Request Is (Mechanically)
A Merge Request (MR) in GitLab is not just a "diff review." Mechanically it is:
- A database record linking a source branch to a target branch
- A container for: code diff, pipeline results, security findings, approval records, comments, linked issues
- An enforcement point for: CI pass requirements, approval requirements, CODEOWNERS sign-off
- An audit record: every approval, comment, commit addition, and pipeline run is logged with timestamp and actor

**The MR as an Audit Artefact:**
When a SOC 2 auditor asks "Show me evidence that this production change was reviewed and approved," the MR provides: who created it, who reviewed it, who approved it, what CI jobs passed, what security findings were present, and when it was merged. All in one URL.

**MR Lifecycle:**
```
Developer creates feature branch
    → Commits code
    → Pushes branch
    → Creates MR (source: feature/x, target: main)
    → Pipeline auto-triggers (CI jobs + security scans)
    → Reviewer(s) assigned (manual or via CODEOWNERS)
    → Reviewer(s) review code, leave comments
    → Developer addresses comments, pushes additional commits
    → Required approvals given
    → CI pipeline passes (or override with justification)
    → MR merged to target branch
    → Source branch deleted (optional)
    → Audit trail complete
```

**Source:** https://docs.gitlab.com/ee/user/project/merge_requests/

#### Merge Request Settings (Security-Relevant)

| Setting | Location | Security Impact |
|---|---|---|
| Require pipelines to succeed | Project → Settings → Merge Requests | Prevents merging code with failing CI (including security scans) |
| Require all threads resolved | Project → Settings → Merge Requests | Ensures review comments are addressed |
| Allow merge only if all status checks pass | With external status checks | Integrates external gates |
| Delete source branch on merge | MR settings | Reduces branch sprawl |
| Squash commits on merge | MR settings | Clean history; but loses granular commit context |
| Merge commit message template | Project settings | Include issue/ticket references automatically |

---

### SECTION 4: Approval Rules (8 minutes)

**What Approval Rules Enforce:**
Approval rules define *who* must approve an MR before it can be merged and *how many* approvers are required. They are the primary governance control on code review quality.

**Types of Approval Rules in GitLab:**
Source: https://docs.gitlab.com/ee/user/project/merge_requests/approvals/

1. **Project-level approval rules** — apply to all MRs in the project
   - Example: "Require 2 approvals from group 'senior-engineers'"

2. **Code Owner approval** — automatically adds code owners as required approvers when their files are touched

3. **Security approvals** — auto-require security team approval when new security vulnerabilities are introduced
   - Configured via Security Policies (Ultimate feature)
   - Source: https://docs.gitlab.com/ee/user/application_security/policies/

4. **Merge Request approval settings:**
   - `Prevent approval by author` — author cannot approve their own MR
   - `Prevent approvals by users who add commits` — contributor cannot approve after adding commits
   - `Require user password to approve` — adds friction to inadvertent approvals
   - `Remove all approvals when commits are added` — forces re-review when code changes

**Security Engineering Note (Teach This):**
The combination of:
- `Prevent approval by author: ON`
- `Remove all approvals when commits are added: ON`
- `Require pipelines to succeed: ON`
- CODEOWNERS for security-sensitive files

creates a **four-eyes principle** that is auditable and tamper-resistant. This directly satisfies PCI DSS 6.3.2 and SOC 2 change management controls.

---

### SECTION 5: CODEOWNERS (7 minutes)

**What CODEOWNERS Is:**
A file (`.gitlab/CODEOWNERS` or `CODEOWNERS` in root) that maps file patterns to GitLab users or groups. When a file matching a pattern is modified in an MR, the corresponding owners are automatically added as required approvers.

**Source:** https://docs.gitlab.com/ee/user/project/codeowners/

**Syntax:**
```
# Format: [path pattern]  @user_or_group

# Security team must approve all changes to authentication code
/src/auth/             @security-team @lead-engineer

# DevOps team must approve CI/CD pipeline changes
/.gitlab-ci.yml        @devops-team

# Anyone on the firmware team approves embedded code
/firmware/             @firmware-team

# Legal must approve license file changes
/LICENSE               @legal-team @cto

# IoT device configuration requires both firmware and cloud team
/device-configs/       @firmware-team @cloud-team

# Default ownership for everything else
*                      @tech-leads
```

**Sections (GitLab Ultimate Feature):**
CODEOWNERS supports **Sections** — named groups of ownership rules that appear as separate approval gates in the MR:

```
[Security Review]
/src/auth/             @security-team
/src/crypto/           @security-team

[Infrastructure Review]
/.gitlab-ci.yml        @devops-team
/terraform/            @devops-team

[API Review]
/src/api/              @backend-leads
```

Each section must be independently approved — providing multi-gate governance in a single MR.

**For Robotics/IoT Teams:**
```
[Safety Review]
/firmware/safety/         @safety-engineers
/robot-controller/limits/ @safety-engineers

[Security Review]
/firmware/crypto/         @security-team
/ota-update/              @security-team

[Hardware Interface]
/drivers/                 @embedded-team
/bsp/                     @embedded-team
```

---

### SECTION 6: Protected Branches (4 minutes)

**What Protected Branches Enforce:**
Source: https://docs.gitlab.com/ee/user/project/protected_branches.html

| Protection | Setting | What It Prevents |
|---|---|---|
| Allowed to merge | Specific role (e.g., Maintainers only) | Low-privilege users merging directly |
| Allowed to push | No one / Maintainers | Direct commits to protected branch |
| Allow force push | OFF | History rewriting |
| Code owner approval required | ON | Bypass of CODEOWNERS rules |
| Require approval from code owners | ON | Code owner auto-add as approver |

**Recommended Production Configuration:**
```
Branch: main
  - Allowed to merge: Maintainers
  - Allowed to push: No one (only via MR)
  - Force push: Not allowed
  - Code owner approval: Required
  - Require MR: Yes (enforced by push restriction)
```

**Why "No one can push" Matters:**
If developers can push directly to `main`, all MR governance is optional. A developer under time pressure will push directly. The protection makes MR review mandatory, not aspirational.

---

## Hands-On Lab Guide (40 minutes)

> Full detailed lab in M04_Lab_Workbook.md

**Lab Summary:** Participants will:
1. Create a GitLab project
2. Configure protected branch on `main`
3. Create a feature branch
4. Add a CODEOWNERS file
5. Configure approval rules
6. Create a Merge Request
7. Perform code review (pair exercise)
8. Approve and merge
9. Verify audit trail

---

## Failure Modes and Anti-Patterns

| Anti-Pattern | What It Looks Like | Business Risk |
|---|---|---|
| Protected branch with no push restriction | Main is "protected" but developers can still push directly | MR governance is optional; compliance evidence is absent |
| Approving your own MR | `Prevent approval by author` not enabled | Four-eyes principle broken; self-approved changes reach production |
| Approval farming | Senior engineer approves 20 MRs without reading them | No actual review; rubber stamp audit trail |
| CODEOWNERS commented out | Security-critical file changed without required owner review | Ownership bypassed; critical path unreviewed |
| Stale approvals allowed | `Remove approvals when commits added` = OFF | Code changes after approval; approved code is not deployed code |
| Force push permitted | History can be rewritten on any branch | Audit trail tampered; compliance evidence invalid |
| No MR template | Developers skip security review checklist | Security consideration bypassed in review process |

---

## Knowledge Check Questions

**L1 (Motivation):**
> "What specific governance failure does CODEOWNERS solve that a general approval rule does not?"

**L2 (Mechanism):**
> "Trace what happens in GitLab when a developer pushes a commit that modifies `/src/auth/login.py` on a branch where a CODEOWNERS rule maps that path to `@security-team`."

**L3 (Mental Model):**
> "A developer bypasses the MR process and pushes directly to `main` — and the pipeline still runs. How could this happen, and how would you prevent it?"

**L4 (Mastery):**
> "Design the CODEOWNERS and approval rules for a monorepo containing: a firmware component, a cloud API, a web frontend, and CI/CD pipeline definitions. What sections would you create, and who would own each?"

---

## References and Citations

1. **GitLab Merge Requests** — https://docs.gitlab.com/ee/user/project/merge_requests/
2. **GitLab Approval Rules** — https://docs.gitlab.com/ee/user/project/merge_requests/approvals/
3. **GitLab CODEOWNERS** — https://docs.gitlab.com/ee/user/project/codeowners/
4. **GitLab Protected Branches** — https://docs.gitlab.com/ee/user/project/protected_branches.html
5. **GitLab CI rules:changes** — https://docs.gitlab.com/ee/ci/yaml/#ruleschanges
6. **DORA Trunk-Based Development** — https://dora.dev/capabilities/trunk-based-development/
7. **NIST SSDF PS.1.1** — https://csrc.nist.gov/Projects/ssdf
8. **GitLab Security Approval Policies** — https://docs.gitlab.com/ee/user/application_security/policies/
9. **GitLab Audit Events** — https://docs.gitlab.com/ee/administration/audit_events.html
