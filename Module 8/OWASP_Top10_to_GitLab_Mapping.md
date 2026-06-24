# OWASP TOP 10 → GITLAB ULTIMATE MAPPING TABLE
## Reference: All Deliverable 9 — Security Capability Mapping

---

**OWASP Top 10 (2021 Edition)**  
Source: https://owasp.org/www-project-top-ten/  
GitLab Documentation: https://docs.gitlab.com/ee/user/application_security/

---

## Master Mapping Table

| # | OWASP Category | GitLab Scanner(s) | CI Template | Scan Type | Detection Examples | Limitation |
|---|---|---|---|---|---|---|
| A01 | Broken Access Control | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` `Jobs/DAST.gitlab-ci.yml` | Static + Dynamic | Missing `@login_required`, direct object reference without ownership check, IDOR in API endpoints | Business logic access control requires manual review |
| A02 | Cryptographic Failures | SAST, Secret Detection | `Jobs/SAST.gitlab-ci.yml` `Jobs/Secret-Detection.gitlab-ci.yml` | Static | `hashlib.md5()` for passwords, `ssl.CERT_NONE`, `verify=False`, hardcoded encryption keys, ECB mode | Custom cryptographic implementations need expert review |
| A03 | Injection | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` `Jobs/DAST.gitlab-ci.yml` | Static + Dynamic | String concatenation in SQL queries, `shell=True` with user input, `eval()`, template injection | Complex multi-step injection chains need DAST |
| A04 | Insecure Design | SAST (partial), Code Review | `Jobs/SAST.gitlab-ci.yml` | Static (partial) | Missing rate limiting decorators, no MFA on admin routes, missing CSRF protection | Design flaws require MR review + threat modelling — automation is limited |
| A05 | Security Misconfiguration | SAST, IaC Scanning, Container Scanning | `Jobs/SAST.gitlab-ci.yml` `Jobs/IaC-Scanning.gitlab-ci.yml` `Jobs/Container-Scanning.gitlab-ci.yml` | Static | `DEBUG=True` in production, public S3 buckets, open security groups, containers running as root, missing TLS | Runtime config drift not detected by IaC scan |
| A06 | Vulnerable & Outdated Components | Dependency Scanning, Container Scanning | `Jobs/Dependency-Scanning.gitlab-ci.yml` `Jobs/Container-Scanning.gitlab-ci.yml` | SCA | Known CVEs in `requirements.txt`, `package.json`, `pom.xml`; CVEs in Docker base image OS packages | Zero-day CVEs not in advisory databases |
| A07 | Auth and Session Failures | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` `Jobs/DAST.gitlab-ci.yml` | Static + Dynamic | Missing rate limiting, plaintext password storage, weak session tokens, insecure cookie flags | Session management logic complexity limits automation |
| A08 | Software & Data Integrity Failures | SAST, Dependency Scanning, Container Scanning | `Jobs/SAST.gitlab-ci.yml` `Jobs/Dependency-Scanning.gitlab-ci.yml` | Static + SCA | Unsafe deserialization (`pickle.loads(untrusted)`), unsigned software updates, untrusted CI dependencies | Supply chain attacks require provenance tooling (Module 13) |
| A09 | Logging & Monitoring Failures | SAST | `Jobs/SAST.gitlab-ci.yml` | Static | Missing logging on auth events, sensitive data in log statements, no audit trail for privileged operations | Cannot detect monitoring gaps — requires runtime observability tooling |
| A10 | SSRF | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` `Jobs/DAST.gitlab-ci.yml` | Static + Dynamic | `requests.get(user_supplied_url)` without validation, XXE via external entities, unvalidated webhook URLs | Complex SSRF via redirect chains needs DAST |

---

## Per-Category Detail Cards

---

### A01 — Broken Access Control

| Attribute | Detail |
|---|---|
| **OWASP Rank** | #1 (moved from #5 in 2017) |
| **CWE IDs** | CWE-200, CWE-201, CWE-352, CWE-862, CWE-863 |
| **Attack Pattern** | IDOR (Insecure Direct Object Reference), privilege escalation, path traversal, forced browsing |
| **Real Breach** | Optus (2022) — 9.8M records via unauthenticated sequential API |
| **GitLab Scanner** | SAST (Semgrep), DAST (ZAP) |
| **SAST Rule Examples** | Missing `@require_permission`, unguarded admin route, direct DB query by user-supplied ID |
| **DAST Coverage** | Role-based access testing, parameter manipulation |
| **GitLab Template** | `include: template: Jobs/SAST.gitlab-ci.yml` |
| **Prevention** | Deny by default; validate ownership server-side; use ABAC/RBAC libraries |
| **NIST SSDF** | PW.5, PW.8 |

---

### A02 — Cryptographic Failures

| Attribute | Detail |
|---|---|
| **OWASP Rank** | #2 (was "Sensitive Data Exposure") |
| **CWE IDs** | CWE-259, CWE-327, CWE-331 |
| **Attack Pattern** | Offline brute force of weak hashes, MITM on unencrypted connections, key theft |
| **Real Breach** | LinkedIn 2012 — 117M accounts, unsalted SHA1 passwords |
| **GitLab Scanner** | SAST (Semgrep), Secret Detection (Gitleaks) |
| **SAST Rule Examples** | `hashlib.md5(password)`, `hashlib.sha1(password)`, `ssl.CERT_NONE`, `verify=False` |
| **Secret Detection** | RSA private keys, EC private keys, PEM certificates in code |
| **GitLab Template** | `Jobs/SAST.gitlab-ci.yml`, `Jobs/Secret-Detection.gitlab-ci.yml` |
| **Prevention** | Use bcrypt/Argon2/scrypt for passwords; TLS 1.2+ everywhere; rotate secrets; use key vaults |
| **NIST SSDF** | PW.5, PS.2 |

---

### A03 — Injection

| Attribute | Detail |
|---|---|
| **OWASP Rank** | #3 (was #1 from 2010–2017) |
| **CWE IDs** | CWE-77, CWE-78, CWE-89, CWE-90 |
| **Attack Pattern** | SQL injection, command injection, LDAP injection, template injection, XSS (see CWE Top 25) |
| **Real Breach** | British Airways 2018 — SQL injection, £20M GDPR fine |
| **GitLab Scanner** | SAST (Semgrep + language-specific analyzers), DAST (ZAP) |
| **SAST Rule Examples** | `"SELECT * FROM users WHERE id = " + user_id`, `subprocess.run(cmd, shell=True)`, `eval(user_input)` |
| **DAST Coverage** | Automated injection payloads against all input parameters |
| **GitLab Template** | `Jobs/SAST.gitlab-ci.yml`, `Jobs/DAST.gitlab-ci.yml` |
| **Prevention** | Parameterised queries; ORMs; input allowlisting; avoid `shell=True`; use `subprocess.run([list])` |
| **NIST SSDF** | PW.5, PW.8 |

---

### A04 — Insecure Design

| Attribute | Detail |
|---|---|
| **OWASP Rank** | #4 (new in 2021) |
| **CWE IDs** | CWE-73, CWE-183, CWE-209, CWE-256 |
| **Attack Pattern** | No rate limiting → brute force; no CSRF → state-changing attacks; insecure password reset |
| **Real Breach** | Multiple medical device firmware vulnerabilities — absent design-time security |
| **GitLab Scanner** | SAST (limited), Code Review (primary) |
| **SAST Rule Examples** | Missing rate-limiting decorator on login endpoint, missing CSRF token check |
| **GitLab Integration** | MR templates with security design checklist; CODEOWNERS for auth/design files |
| **GitLab Template** | `Jobs/SAST.gitlab-ci.yml` (partial) |
| **Prevention** | Threat modelling (STRIDE) before coding; security requirements in issue definition; design review gate |
| **NIST SSDF** | PW.1, PO.4 |

---

### A05 — Security Misconfiguration

| Attribute | Detail |
|---|---|
| **OWASP Rank** | #5 |
| **CWE IDs** | CWE-16, CWE-611 |
| **Attack Pattern** | Default credentials, verbose error pages, publicly accessible cloud storage, debug mode in production |
| **Real Breach** | Capital One 2019 — misconfigured WAF, public S3 bucket, 100M records |
| **GitLab Scanner** | IaC Scanning (KICS), Container Scanning (Trivy), SAST |
| **IaC Rule Examples** | `acl = "public-read"` on S3, open `0.0.0.0/0` security groups, missing KMS encryption |
| **Container Scan** | `USER root` in Dockerfile, missing `HEALTHCHECK`, outdated base OS |
| **GitLab Template** | `Jobs/IaC-Scanning.gitlab-ci.yml`, `Jobs/Container-Scanning.gitlab-ci.yml` |
| **Prevention** | IaC for all infrastructure; no manual console changes; scan manifests in CI |
| **NIST SSDF** | PO.3, PW.9 |

---

### A06 — Vulnerable and Outdated Components

| Attribute | Detail |
|---|---|
| **OWASP Rank** | #6 |
| **CWE IDs** | CWE-937, CWE-1035, CWE-1104 |
| **Attack Pattern** | Exploit known CVE in unpatched dependency; OS package RCE via unpatched container base image |
| **Real Breach** | Equifax 2017 — Apache Struts CVE-2017-5638, 147M records; Log4Shell 2021 — CVSS 10.0 |
| **GitLab Scanner** | Dependency Scanning (Trivy/Gemnasium), Container Scanning (Trivy) |
| **Dependency Scan** | `requirements.txt`, `package.json`, `pom.xml`, `go.mod`, `Cargo.toml`, `Gemfile` |
| **Container Scan** | OS packages in Docker image layers |
| **GitLab Template** | `Jobs/Dependency-Scanning.gitlab-ci.yml`, `Jobs/Container-Scanning.gitlab-ci.yml` |
| **Prevention** | Automated SCA in CI; pin dependency versions; update policy (e.g., Dependabot/Renovate); SBOM |
| **NIST SSDF** | PW.4, RV.1, RV.2 |

---

### A07 — Identification and Authentication Failures

| Attribute | Detail |
|---|---|
| **OWASP Rank** | #7 |
| **CWE IDs** | CWE-287, CWE-295, CWE-306, CWE-307, CWE-798 |
| **Attack Pattern** | Credential stuffing, brute force, session fixation, token theft, hardcoded credentials |
| **Real Breach** | Dropbox 2012 — reused password from LinkedIn breach compromised internal admin |
| **GitLab Scanner** | SAST, Secret Detection, DAST |
| **SAST Rule Examples** | `password == "hardcoded_pass"`, missing `max_login_attempts`, `session.permanent = True` |
| **Secret Detection** | Hardcoded passwords, API tokens, default credentials in config files |
| **GitLab Template** | `Jobs/SAST.gitlab-ci.yml`, `Jobs/Secret-Detection.gitlab-ci.yml` |
| **Prevention** | MFA; account lockout; bcrypt passwords; rotate credentials; use Vault/SSM for secrets |
| **NIST SSDF** | PW.5, PS.1 |

---

### A08 — Software and Data Integrity Failures

| Attribute | Detail |
|---|---|
| **OWASP Rank** | #8 (new in 2021, includes A8:2017 Insecure Deserialization) |
| **CWE IDs** | CWE-345, CWE-353, CWE-426, CWE-494, CWE-502 |
| **Attack Pattern** | Unsafe deserialization → RCE; unsigned software updates; compromised dependency; CI/CD compromise |
| **Real Breach** | SolarWinds 2020 — SUNBURST malware in signed software update; 18,000 customers |
| **GitLab Scanner** | SAST (deserialization), Dependency Scanning, Container Scanning |
| **SAST Rule Examples** | `pickle.loads(user_data)`, `yaml.load(data)` (without Loader), Java `ObjectInputStream` |
| **Supply Chain** | Dependency Scanning + SBOM + signed artifacts (Module 13) |
| **GitLab Template** | `Jobs/SAST.gitlab-ci.yml`, `Jobs/Dependency-Scanning.gitlab-ci.yml` |
| **Prevention** | Sign all artifacts; verify signatures before use; pin dependency hashes; protect CI/CD pipeline |
| **NIST SSDF** | PS.2, PS.3, PW.5 |

---

### A09 — Security Logging and Monitoring Failures

| Attribute | Detail |
|---|---|
| **OWASP Rank** | #9 |
| **CWE IDs** | CWE-117, CWE-223, CWE-532, CWE-778 |
| **Attack Pattern** | Slow exfiltration undetected; brute force undetected; privilege escalation undetected |
| **Impact** | IBM: average breach detection takes 204 days without effective logging |
| **GitLab Scanner** | SAST (detect missing logging in critical functions) |
| **SAST Rule Examples** | Authentication functions with no logging; exception handlers that swallow errors silently |
| **GitLab Platform** | GitLab Audit Events provides platform-level logging; application logging is code concern |
| **GitLab Template** | `Jobs/SAST.gitlab-ci.yml` (limited coverage) |
| **Prevention** | Log auth events (success + failure); log access denials; centralise logs (SIEM); alert on anomalies |
| **NIST SSDF** | RV.3 |

---

### A10 — Server-Side Request Forgery (SSRF)

| Attribute | Detail |
|---|---|
| **OWASP Rank** | #10 (new in 2021) |
| **CWE IDs** | CWE-918 |
| **Attack Pattern** | Attacker causes server to fetch internal resource: `http://169.254.169.254/` (AWS metadata), internal APIs, file:// |
| **Real Breach** | Capital One 2019 — SSRF via misconfigured WAF → EC2 metadata → IAM creds → S3 access, 100M records |
| **GitLab Scanner** | SAST, DAST |
| **SAST Rule Examples** | `requests.get(url)` where `url` is derived from user input without hostname validation |
| **DAST Coverage** | Automated SSRF probe via ZAP against URL-accepting parameters |
| **GitLab Template** | `Jobs/SAST.gitlab-ci.yml`, `Jobs/DAST.gitlab-ci.yml` |
| **Prevention** | Allowlist permitted URL schemes and hostnames; block private IP ranges; disable redirects; use metadata blocking |
| **NIST SSDF** | PW.5, PW.8 |

---

## GitLab Template Quick Reference

Add all security scans to your pipeline with:

```yaml
# .gitlab-ci.yml — include all security scan templates
include:
  # SAST — static analysis (runs on every commit)
  - template: Jobs/SAST.gitlab-ci.yml

  # Secret Detection — credential scanning (runs on every commit)
  - template: Jobs/Secret-Detection.gitlab-ci.yml

  # Dependency Scanning — SCA (runs on build)
  - template: Jobs/Dependency-Scanning.gitlab-ci.yml

  # Container Scanning — Docker image CVEs (runs post-build)
  - template: Jobs/Container-Scanning.gitlab-ci.yml

  # IaC Scanning — Terraform/K8s misconfigs (runs on IaC change)
  - template: Jobs/IaC-Scanning.gitlab-ci.yml

  # DAST — dynamic testing (runs post-deploy to test env)
  - template: Jobs/DAST.gitlab-ci.yml
```

*Source: https://docs.gitlab.com/ee/user/application_security/#security-scanning-tools*

---

## OWASP Category Coverage Summary

| Category | SAST | Secret Detection | Dependency Scanning | Container Scanning | IaC Scanning | DAST | Code Review |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| A01 Broken Access Control | ⚠️ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| A02 Cryptographic Failures | ✅ | ✅ | ❌ | ❌ | ❌ | ⚠️ | ✅ |
| A03 Injection | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| A04 Insecure Design | ⚠️ | ❌ | ❌ | ❌ | ❌ | ⚠️ | ✅ |
| A05 Security Misconfiguration | ✅ | ❌ | ❌ | ✅ | ✅ | ⚠️ | ✅ |
| A06 Vulnerable Components | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ | ⚠️ |
| A07 Auth Failures | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ |
| A08 Integrity Failures | ✅ | ❌ | ✅ | ⚠️ | ❌ | ❌ | ✅ |
| A09 Logging Failures | ⚠️ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| A10 SSRF | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |

**Legend:** ✅ Strong coverage | ⚠️ Partial coverage | ❌ Not covered by this scanner

> **Key takeaway:** No single scanner covers all categories. SAST + DAST + SCA + Secret Detection together provide coverage for all 10 categories at varying depths. Code Review remains essential for A01 and A04.
