# /review — Code Review Report Generator

Analyze changed files and produce a structured review report covering bugs, security issues, and code quality.

## Steps

1. **Identify changed files**
   ```bash
   git diff --name-only HEAD
   git diff --stat HEAD
   ```

2. **Read each changed file** using the Read tool

3. **Analyze across four dimensions:**

   ### Bugs & Logic Errors
   - Off-by-one errors, null/undefined access
   - Incorrect conditionals or loop boundaries
   - Missing error handling at system boundaries
   - Race conditions or async misuse

   ### Security Issues (OWASP Top 10)
   - SQL injection, XSS, command injection
   - Hardcoded secrets or credentials
   - Insecure deserialization
   - Missing input validation at boundaries
   - Overly permissive access controls

   ### Code Quality
   - Premature abstractions or over-engineering
   - Dead code or unused imports
   - Overly complex functions (consider cyclomatic complexity)
   - Missing tests for critical paths

   ### Performance
   - N+1 query patterns
   - Unnecessary re-renders or recomputations
   - Missing indexes or caching opportunities

4. **Generate report** in this format:

   ```markdown
   ## Code Review Report — <date>

   ### Summary
   Files reviewed: X | Issues found: Y (Critical: Z, Warning: W, Info: I)

   ### Critical Issues
   - **[BUG/SEC]** `file.ts:42` — Description of issue
     ```suggestion
     // suggested fix
     ```

   ### Warnings
   - **[QUALITY]** `file.ts:15` — Description

   ### Info
   - **[PERF]** `file.ts:88` — Observation

   ### Approved
   - Areas that look good and why
   ```

5. **Output the report** to the user (do not auto-fix unless asked)

## Rules

- Report issues, do not auto-apply fixes without explicit request
- Prioritize Critical > Warning > Info
- Include file path and line number for every issue
- If no issues found, say so clearly
