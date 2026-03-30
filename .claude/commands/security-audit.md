# /security-audit — OWASP Security Audit

Run a focused security audit on changed or specified files.

## Steps

1. **Identify target files**
   - If argument provided: audit that file/directory
   - Otherwise: `git diff --name-only HEAD` for changed files

2. **Read each target file** using the Read tool

3. **Run dependency audit**
   ```bash
   npm audit --audit-level=high 2>/dev/null || true
   ```

4. **Check for hardcoded secrets**
   Search for patterns: API keys, tokens, passwords, private keys in code

5. **Audit against OWASP Top 10:**

   - **Injection**: String-interpolated SQL/shell commands, unsanitized inputs
   - **Broken Auth**: Weak session handling, missing token expiry, no rate limiting
   - **Sensitive Data**: Secrets in code, weak encryption, over-logging of PII
   - **Broken Access Control**: Missing auth checks, IDOR vulnerabilities
   - **Security Misconfiguration**: Debug flags, permissive CORS, default credentials
   - **XSS**: Unescaped user input rendered in HTML/JSX (dangerouslySetInnerHTML)
   - **Vulnerable Dependencies**: Known CVEs in package.json/requirements.txt
   - **Missing Security Headers**: CSP, HSTS, X-Frame-Options

6. **Generate report:**

   ```
   ## Security Audit — <date>

   ### Summary
   Files: X | Findings: Y (Critical: C, High: H, Medium: M, Low: L)

   ### Critical
   - **[SEC]** `file:line` — issue description
     Fix: remediation steps

   ### Dependencies
   <audit output summary>

   ### Approved
   - What looks secure
   ```

## Rules

- Report only — do not auto-fix
- Every finding must have: file, line number, severity, remediation
- If nothing found, state explicitly: "No security issues found"
