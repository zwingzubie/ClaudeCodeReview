# Security Review

You are performing a security audit of the current code changes. Your job is to find real
vulnerabilities, not to produce a checklist for its own sake.

## What to Audit

**Scope:** Review the files changed in the current branch versus main. Run:
```
git diff main...HEAD --name-only
```
Then read each changed file in full before making findings.

---

## Checks to Perform

### 1. OWASP Top 10 Patterns
- **Injection (A03):** Look for SQL, NoSQL, command, or LDAP injection. Flag any place where
  user-controlled input is concatenated into a query, shell command, or eval expression.
- **Broken Auth (A07):** Check that new routes apply authentication middleware. Verify JWT
  validation is not bypassed. Look for missing `requireAuth` or `requireRole` calls.
- **Sensitive Data Exposure (A02):** Check that passwords, tokens, and PII are never logged,
  returned in API responses unnecessarily, or stored unhashed.
- **Security Misconfiguration (A05):** Look for CORS set to `*`, debug endpoints left open,
  verbose error messages that expose stack traces to clients.
- **Vulnerable Components (A06):** Note any newly imported packages — flag if they are
  unfamiliar, abandoned, or if their name looks like a typo of a popular package (typosquatting).
- **SSRF (A10):** Flag any code that makes HTTP requests using URLs derived from user input
  without validation against an allowlist.

### 2. Hardcoded Secrets
Search for patterns: API keys, tokens, passwords, private keys, connection strings.
Look for: variable names containing `key`, `secret`, `password`, `token`, `credential`,
and values that look like base64 strings, UUIDs, or long hex strings assigned inline.

### 3. XSS and Output Encoding
In frontend code: check that user-supplied content is never rendered via `dangerouslySetInnerHTML`,
`innerHTML`, `document.write`, or `eval`. Verify that React components escape content by default
and flag any explicit bypasses.

### 4. Authorization Logic
For every new API endpoint or data access function: verify that the caller's identity and
role is checked *before* data is returned or mutated. Flag optimistic authorization patterns
where auth is checked after the expensive operation.

### 5. Dependency Review
List any packages added or upgraded in `package.json`. For each:
- Note the version and whether it pins to an exact version or a range.
- Flag any package you are not familiar with as INFO.
- Flag any package with a known CVE in its current version as BLOCKING.

---

## Output Format

Group findings by severity. Use exactly these labels:

### BLOCKING
*Must be resolved before merge. These are confirmed or highly likely vulnerabilities.*

**[FILE:LINE]** — [Finding title]
[Explanation of what is wrong and what the impact is.]
[Concrete fix or what to change.]

### ADVISORY
*Should be addressed but are not necessarily merge-blocking. Discuss with the team.*

**[FILE:LINE]** — [Finding title]
[Explanation and recommendation.]

### INFO
*Observations worth tracking. No immediate action required.*

**[FILE]** — [Observation]

---

## After Findings

If there are BLOCKING findings: state clearly "This PR has BLOCKING security findings and
should not be merged until they are resolved."

If there are no BLOCKING findings: state "No blocking security issues found. Advisory and
Info items above are recommended improvements."

Always end with the command the engineer should run to do their own scan:
```bash
semgrep scan --config=auto --severity=ERROR .
npm audit --audit-level=high
```
