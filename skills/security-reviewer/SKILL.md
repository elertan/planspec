---
name: security-reviewer
description: Security-focused code review for sensitive implementations. Checks for common vulnerabilities, auth issues, input validation, and data handling. Only invoke when impl-spec checkpoint flags security review required, or when code touches auth, user input, database queries, file access, or external credentials.
---

# Security Reviewer

Focused security review for code touching sensitive areas.

## Usage

```
planspec:security-reviewer
```

With specific focus:
```
planspec:security-reviewer --focus auth,input-validation
```

## When to Use

**Invoke when code touches:**
- Authentication / authorization
- User input processing
- Database queries
- File system access
- External API calls with credentials
- Session management
- Cryptographic operations
- Sensitive data handling (PII, payments, etc.)

**Skip when:**
- Pure UI/styling changes
- Internal utility functions with no external input
- Test files only
- Documentation changes

## Process

### Step 1: Identify Security Surface

1. **From impl-spec:** Read which security areas were flagged
2. **From code:** Scan changed files for security-relevant patterns:
   - Auth decorators/middleware
   - Input parsing (body, query params, headers)
   - Database operations
   - File operations
   - HTTP clients / external calls
   - Environment variable access

### Step 2: Focused Review

Review ONLY security-relevant aspects. This is not a general code review.

#### 2.1 Authentication & Authorization

| Check | Look For |
|-------|----------|
| Auth bypass | Routes missing auth middleware |
| Privilege escalation | User can access other users' data |
| Token handling | Tokens logged, exposed in URLs, stored insecurely |
| Session management | Session fixation, no expiry, no invalidation |

**Questions to answer:**
- Can an unauthenticated user reach this?
- Can a user access resources they shouldn't?
- Are tokens/sessions handled securely?

#### 2.2 Input Validation

| Check | Look For |
|-------|----------|
| SQL Injection | String concatenation in queries |
| XSS | Unescaped user input in HTML output |
| Command Injection | User input in shell commands |
| Path Traversal | User input in file paths |
| Type confusion | Missing type validation |

**Questions to answer:**
- Is every external input validated before use?
- Can malformed input cause unexpected behavior?
- Is input sanitized before output?

#### 2.3 Data Handling

| Check | Look For |
|-------|----------|
| Sensitive data exposure | PII in logs, error messages, responses |
| Insecure storage | Plaintext passwords, unencrypted secrets |
| Data leakage | Returning more data than needed |
| Missing encryption | Sensitive data transmitted/stored in clear |

**Questions to answer:**
- Is sensitive data protected at rest and in transit?
- Could error messages leak internal details?
- Does the response include only necessary data?

#### 2.4 External Interactions

| Check | Look For |
|-------|----------|
| SSRF | User-controlled URLs in server requests |
| Credential exposure | API keys in code, logs, or responses |
| Insecure protocols | HTTP instead of HTTPS |
| Missing timeouts | Unbounded external calls |

**Questions to answer:**
- Are external calls properly bounded and authenticated?
- Are credentials stored securely (env vars, secret managers)?
- Could a user make the server request arbitrary URLs?

#### 2.5 Business Logic

| Check | Look For |
|-------|----------|
| Race conditions | TOCTOU issues in sensitive operations |
| Missing rate limiting | Brute-force vulnerable endpoints |
| Insufficient validation | Business rule bypasses |

**Questions to answer:**
- Can the operation be abused at scale?
- Are business rules enforced server-side (not just client)?

### Step 3: Report Findings

```markdown
## Security Review: Phase [N]

### Scope
Files reviewed: [list]
Focus areas: [auth, input, data, external]

### Findings

#### 🔴 Critical (must fix immediately)
- **[Vulnerability type]** in `[file:line]`
  - Risk: [What could happen if exploited]
  - Attack: [How it could be exploited]
  - Fix: [Specific remediation]

#### 🟠 High (must fix before deploy)
- **[Issue]** in `[file:line]`
  - Risk: [Impact]
  - Fix: [Remediation]

#### 🟡 Medium (should fix)
- **[Issue]** in `[file:line]`
  - Risk: [Impact]
  - Fix: [Remediation]

#### ℹ️ Recommendations
- [Hardening suggestion not tied to specific vulnerability]

### Areas Reviewed
- [x] Authentication: [status]
- [x] Input validation: [status]
- [x] Data handling: [status]
- [ ] External calls: Not applicable

### Verdict
✅ PASS - No critical/high issues
❌ FAIL - [N] critical/high issues must be resolved
```

### Step 4: Resolution

1. **Critical/High issues:** Must fix before proceeding
2. **Medium issues:** Should fix, can proceed with tracking
3. **Recommendations:** Optional improvements

**After fixes:**
- Re-review specific fixes
- Verify fix doesn't introduce new issues
- Update verdict

---

## Common Vulnerability Patterns

### SQL Injection
```
# BAD - string concatenation
db.query("SELECT * FROM users WHERE id = " + userId)

# GOOD - parameterized query
db.query("SELECT * FROM users WHERE id = ?", [userId])
```

### XSS
```
# BAD - raw user input in HTML
element.innerHTML = userInput

# GOOD - use textContent or sanitize
element.textContent = userInput
```

### Path Traversal
```
# BAD - direct path concatenation
file = read_file("uploads/" + user_filename)

# GOOD - validate and sanitize path
safe_name = os.path.basename(user_filename)
file = read_file(os.path.join("uploads", safe_name))
```

### Auth Bypass
```
# BAD - missing auth check
@app.route('/admin/users')
def list_users(): ...

# GOOD - explicit auth required
@app.route('/admin/users')
@require_auth
@require_admin
def list_users(): ...
```

---

## Output

**On pass:**
```
✅ Security Review Passed

Scope: auth, input-validation
Files: 4 reviewed
Findings: 0 critical, 0 high, 1 medium (tracked)

Cleared to proceed.
```

**On fail:**
```
🔴 Security Review: Critical Issues

Findings: 1 critical, 2 high

[Detailed findings]

Must resolve critical and high issues before proceeding.
```

---

## Principles

- **Focused:** Only security, not general code quality
- **Risk-based:** Severity based on exploitability and impact
- **Actionable:** Every finding includes how to fix
- **No false sense of security:** This is not exhaustive - flag limitations
- **Blocking on critical/high:** No exceptions for serious vulnerabilities
