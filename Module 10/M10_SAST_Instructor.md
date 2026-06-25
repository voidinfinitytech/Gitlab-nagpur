# INSTRUCTOR HANDBOOK — MODULE 10
## Static Application Security Testing (SAST): How It Works, What It Finds, Lab Guide

---

### Module Metadata

| Field          | Value                                                         |
|----------------|---------------------------------------------------------------|
| Module Number  | 10 of 16                                                      |
| Day            | Day 2                                                         |
| Duration       | 90 minutes (25 min content + 65 min hands-on lab)            |
| Difficulty     | Intermediate — content light; lab is the primary deliverable |
| Prerequisites  | Modules 7–9 complete; working pipeline from Day 1            |
| Lab Produces   | Pipeline with SAST integrated; findings triaged and fixed     |
| Key Files      | `07_Vulnerable_Apps/M09_M10_vulnerable_robovision_app.py`    |

---

## Instructor Preparation Notes

### The Lab is the Module
Module 10 is fundamentally a lab module. The 25-minute content section establishes the mechanics (how SAST works internally, what the GitLab SAST template does, what the JSON report schema looks like). The 65-minute lab is where the learning happens.

**Three things participants must experience first-hand:**
1. Adding one `include:` line to `.gitlab-ci.yml` and watching SAST appear in the pipeline — zero configuration required to start.
2. A finding appearing in the Merge Request security widget *before* code is reviewed — seeing the exact file, line number, CWE ID, and remediation guidance.
3. Fixing the vulnerability, re-running the pipeline, and watching the finding disappear — the feedback loop closing in under 5 minutes.

### Pre-Lab Setup (Do Before the Module)
Ensure participants have:
- The `robovision-controller` project from Day 1 labs
- The vulnerable application file committed to the project (from M09 lab exercise)
- A GitLab Runner registered with Docker executor (or GitLab.com shared runners)
- GitLab Ultimate access (SAST is available in all tiers; advanced rules require Ultimate)

### Common Lab Failure Points
1. **Runner not available** — GitLab.com shared runners should work; self-managed needs a registered Docker executor runner
2. **Pipeline takes too long** — SAST pulls scanner Docker images on first run; allow 5–8 minutes for the first pipeline
3. **No findings appear** — check that the vulnerable file is actually committed; check job log for scanner errors
4. **Findings don't show in MR** — pipeline must be triggered via MR (merge request pipeline), not a branch push; ensure MR exists

---

## Learning Objectives

1. Explain how SAST works internally — from source code to finding, tracing the analysis pipeline.
2. Describe the difference between pattern-matching SAST and dataflow-aware SAST.
3. Add GitLab SAST to a pipeline using the managed template in one `include:` line.
4. Interpret a GitLab SAST finding: severity, CWE ID, affected line, remediation guidance.
5. Triage SAST findings: confirm, dismiss (false positive), or create fix.
6. Fix SQL injection and hardcoded credential findings and verify resolution via pipeline re-run.
7. Explain the false positive problem and how to tune SAST rules in GitLab.

---

## Business Context

### The Cost of Manual Code Review for Security

A typical security code review by a consultant costs £800–£2,000 per day. A 50,000-line application takes 5–10 consultant-days to review thoroughly for security issues. Cost: £4,000–£20,000 per review. Frequency: quarterly at best.

SAST runs on every commit. It never forgets a pattern. It never gets tired at 4pm. It runs the same rules on 50,000 lines in under 3 minutes. At a cadence of 50 commits per day across a team, that is 18,250 scans per year — every one of them consistent, documented, and traceable to a specific commit.

**What SAST Cannot Replace:**
- Business logic review (SAST does not understand your domain model)
- Architecture review (SAST cannot assess if your authentication design is sound)
- Manual testing for logic flaws specific to your application

**What SAST Excels At:**
- Finding known vulnerability patterns consistently, at scale, at speed
- Scanning every line of code, not just the lines a consultant had time to review
- Providing immediate developer feedback with line-level precision

---

## Module Content — Teaching Guide

---

### SECTION 1: How SAST Works Internally (10 minutes)

#### The Two Families of SAST Analysis

**1. Pattern Matching / Grepping (Simple)**
The simplest form of SAST: look for dangerous function calls or string patterns.

```
Rule: flag any call to subprocess.run() where shell=True appears
Source: subprocess.run(f"ping {host}", shell=True, ...)
Match: Yes — shell=True with non-literal first argument
```

Fast, low false-negative rate for known bad patterns. High false-positive rate — cannot distinguish:
```python
subprocess.run(["ping", "-c", "1", host], shell=True)  # Still flagged
# But this is safe: list args with shell=True is acceptable
```

**2. Dataflow / Taint Analysis (Sophisticated)**
Tracks how data flows through the program. Identifies when user-controlled data (a *source*) reaches a dangerous function (a *sink*) without sanitisation (a *sanitiser*).

```
Source:    username = request.args.get('username')   ← user-controlled
           (taint propagates through assignment)
Flow:      query = "SELECT * FROM users WHERE name='" + username + "'"
           (taint in username propagates to query)
Sink:      db.execute(query)
           (tainted data reaches SQL execution sink)
No sanitiser found between source and sink.
Result:    CWE-89 SQL Injection — CONFIRMED
```

Dataflow analysis produces far fewer false positives but requires building an AST (Abstract Syntax Tree) or CFG (Control Flow Graph) of the code, which is computationally heavier.

**GitLab's Approach (Semgrep):**
Semgrep uses a hybrid approach — pattern matching with AST awareness. It understands code structure (not just text), enabling context-sensitive rules that reduce false positives. It is significantly faster than full dataflow analysis while being more accurate than pure text grep.

Source: https://semgrep.dev/docs/

#### The SAST Analysis Pipeline (Draw on Whiteboard)

```
Source Code
    ↓
Lexer → Tokens
    ↓
Parser → AST (Abstract Syntax Tree)
    ↓
Rule Engine → Apply security rules to AST nodes
    ↓
Finding Generation → {file, line, rule_id, CWE, severity, code snippet}
    ↓
Report: gl-sast-report.json (GitLab standard schema)
    ↓
Uploaded as artifact → Sidekiq ingests → PostgreSQL → Security Dashboard
```

**Teaching the AST Concept (1 minute):**
```python
# Source code text:
db.execute("SELECT * FROM users WHERE id = " + user_id)

# AST node (simplified):
CallExpression(
  callee: MemberExpression(object: "db", property: "execute"),
  args: [BinaryExpression(
    operator: "+",
    left: Literal("SELECT * FROM users WHERE id = "),
    right: Identifier("user_id")   ← SAST detects: variable in SQL string
  )]
)

# Rule: "flag db.execute() calls where argument contains string concatenation with identifier"
# Match: YES → CWE-89 finding generated
```

---

### SECTION 2: GitLab SAST — The Template Architecture (8 minutes)

#### One Line to Enable SAST

```yaml
include:
  - template: Jobs/SAST.gitlab-ci.yml
```

This single `include:` statement:
1. Pulls the managed SAST template from GitLab's template repository
2. Auto-detects languages in the repository
3. Selects the appropriate analyzer(s) for each language
4. Injects SAST jobs into the pipeline automatically

Source: https://docs.gitlab.com/ee/user/application_security/sast/

#### Language Detection and Analyzer Selection

GitLab SAST auto-detects language and selects the appropriate analyzer:

| Language | Primary Analyzer | Additional |
|---|---|---|
| Python | Semgrep | Bandit (legacy) |
| JavaScript / TypeScript | Semgrep | ESLint security |
| Java | Semgrep | SpotBugs |
| Go | Semgrep | Gosec |
| Ruby | Semgrep | — |
| C / C++ | Semgrep | Flawfinder |
| C# / .NET | Semgrep | — |
| PHP | Semgrep | — |
| Kotlin | Semgrep | — |
| Scala | Semgrep | — |

Source: https://docs.gitlab.com/ee/user/application_security/sast/analyzers.html

**Why This Matters:**
A polyglot repository (Python API + JavaScript frontend + Go service + C firmware) gets ALL appropriate analyzers applied automatically. No per-language configuration needed.

#### The gl-sast-report.json Schema

Every SAST analyzer outputs to the same JSON schema — this is GitLab's normalisation contribution:

```json
{
  "version": "15.0.7",
  "vulnerabilities": [
    {
      "id": "abc123def456",
      "category": "sast",
      "name": "SQL Injection",
      "description": "Potential SQL injection...",
      "severity": "Critical",
      "confidence": "High",
      "scanner": {
        "id": "semgrep",
        "name": "Semgrep"
      },
      "location": {
        "file": "app/main.py",
        "start_line": 47,
        "end_line": 47,
        "class": "login",
        "method": "login"
      },
      "identifiers": [
        {
          "type": "cwe",
          "name": "CWE-89",
          "value": "89",
          "url": "https://cwe.mitre.org/data/definitions/89.html"
        },
        {
          "type": "semgrep_id",
          "name": "python.lang.security.audit.formatted-sql-query.formatted-sql-query",
          "value": "python.lang.security.audit.formatted-sql-query"
        }
      ],
      "links": [
        {"url": "https://owasp.org/www-community/attacks/SQL_Injection"}
      ]
    }
  ],
  "scan": {
    "scanner": {"id": "semgrep", "name": "Semgrep"},
    "type": "sast",
    "start_time": "2024-11-15T14:30:00",
    "end_time": "2024-11-15T14:32:15",
    "status": "success"
  }
}
```

**Key Fields to Teach:**
- `severity`: Critical / High / Medium / Low / Info — directly maps to Security Policy thresholds
- `confidence`: High / Medium / Low — affects false positive likelihood
- `location.file` + `location.start_line` — exact location, clickable in GitLab UI
- `identifiers[].type = "cwe"` — the CWE ID, linkable to CWE database
- `scanner.id` — which analyzer produced this finding

---

### SECTION 3: False Positives and Tuning (7 minutes)

#### The False Positive Problem

False positives are findings that the scanner reports as vulnerabilities but are actually safe. They are one of the primary reasons developers disable or ignore security scanners.

**Example of a Common False Positive:**
```python
import subprocess

# This is safe: list arguments, no shell interpretation
# SOME SAST rules flag all subprocess.run() calls with shell=True regardless of arg type
result = subprocess.run(["ping", "-c", "3", "8.8.8.8"], shell=True)
```

Semgrep's rule `python.lang.security.audit.subprocess-shell-true` flags ANY `shell=True` — even when the first argument is a list (safe). This is a false positive.

**The False Positive Spectrum:**

| False Positive Rate | Effect on Developers |
|---|---|
| 0–5% | Trustworthy; developers act on findings |
| 5–20% | Tolerable; developers occasionally need to verify |
| 20–50% | Alert fatigue; developers start dismissing without reading |
| >50% | Scanner disabled or ignored entirely |

#### Managing False Positives in GitLab

**Method 1: Dismiss in Security Dashboard**
- Navigate to Security → Vulnerability Report
- Select finding → Dismiss → Choose reason: "False positive"
- Requires justification (audit-logged)
- Finding no longer surfaces in dashboard but is preserved in history

**Method 2: `.semgrepignore` / Inline Rule Suppression**
```python
# Suppress a specific Semgrep rule on a specific line
result = subprocess.run(["ping", host], shell=True)  # nosemgrep: subprocess-shell-true
```

**Method 3: SAST Variable Configuration in Pipeline**
```yaml
variables:
  SAST_EXCLUDED_PATHS: "spec, test, tests, tmp"    # Exclude test directories
  SAST_EXCLUDED_ANALYZERS: "flawfinder"             # Exclude a specific analyzer
  SEMGREP_RULES: >-                                 # Custom rules or exclude built-ins
    p/default
    --exclude-rule=python.lang.security.audit.subprocess-shell-true
```

Source: https://docs.gitlab.com/ee/user/application_security/sast/#available-cicd-variables

**Method 4: Security Policies (Ultimate — Production)**
Configure at group level — findings from specific rules that meet false-positive criteria are auto-dismissed. Centralised control, not per-project configuration.

**Teaching Point:**
> "The goal is not zero false positives. The goal is a false positive rate low enough that developers trust and act on findings. Start with the default ruleset. After 30 days, review dismissed findings to identify patterns. Create targeted suppressions only for confirmed false positives with documented justification."

---

## Lab Overview (65 minutes total)

The lab has four phases:

| Phase | Duration | What Happens |
|---|---|---|
| **Phase 1: Enable SAST** | 15 min | Add vulnerable app + SAST template to pipeline; trigger pipeline |
| **Phase 2: Read the Findings** | 15 min | Open MR; examine security widget; read findings; map to CWEs |
| **Phase 3: Fix the Vulnerabilities** | 25 min | Fix SQL injection + hardcoded credentials; re-run |
| **Phase 4: Triage a False Positive** | 10 min | Dismiss one finding; document justification; observe audit trail |

> Full step-by-step: `04_Lab_Workbook/M10_SAST_Lab.md`

---

## Failure Modes and Anti-Patterns

| Anti-Pattern | Description | Consequence |
|---|---|---|
| `allow_failure: true` on SAST job | SAST never blocks; findings are informational | Developers stop checking; findings accumulate |
| Suppressing all findings on a file | `# nosemgrep` on every line | SAST scanning disabled by developer; real vulnerabilities hidden |
| Dismissing without justification | GitLab requires reason but teams write "not applicable" for everything | Audit trail meaningless; compliance evidence invalid |
| SAST only on main branch | Feature branch code never scanned | Vulnerabilities reach main before detection |
| Ignoring `confidence: Low` findings | Low confidence ≠ not real | Real vulnerabilities get lower confidence scores too |
| No SAST for embedded/C code | "SAST is only for web apps" | Memory safety issues in firmware go undetected |

---

## Knowledge Check Questions

**L1:** "What is the difference between pattern-matching SAST and taint/dataflow SAST? Which is Semgrep?"

**L2:** "Walk me through exactly what happens after a SAST job completes in GitLab — trace from `gl-sast-report.json` to the MR security widget."

**L3:** "A SAST finding shows CWE-89, severity Critical, confidence High, in `app/login.py` line 47. The developer says 'we validate input before this function is called.' How do you verify that claim before dismissing?"

**L4 (Mastery):** "Your SAST pipeline runs 200 rules on a Python application and generates 150 findings on first run. Your team has two weeks to address this. Design a prioritisation and triage strategy — including which findings to fix, which to suppress, and how to prevent new ones while the backlog is cleared."

---

## References

1. **GitLab SAST Documentation** — https://docs.gitlab.com/ee/user/application_security/sast/
2. **GitLab SAST Analyzers** — https://docs.gitlab.com/ee/user/application_security/sast/analyzers.html
3. **GitLab SAST CI/CD Variables** — https://docs.gitlab.com/ee/user/application_security/sast/#available-cicd-variables
4. **GitLab SAST Template (source)** — https://gitlab.com/gitlab-org/gitlab/-/blob/master/lib/gitlab/ci/templates/Jobs/SAST.gitlab-ci.yml
5. **GitLab Security Report Schema** — https://gitlab.com/gitlab-org/security-products/security-report-schemas
6. **Semgrep Documentation** — https://semgrep.dev/docs/
7. **Semgrep Rule Registry** — https://semgrep.dev/explore
8. **GitLab Vulnerability Dismissal** — https://docs.gitlab.com/ee/user/application_security/vulnerability_report/#dismiss-a-vulnerability
