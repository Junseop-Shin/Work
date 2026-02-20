# /commit — Conventional Commit Generator

Analyze staged changes and create a properly formatted commit, then push to a feature branch and open a PR.

## Steps

1. **Check staged changes**
   ```bash
   git diff --cached --stat
   git diff --cached
   ```

2. **Analyze changes** — determine the appropriate commit type:
   - `feat`: new feature or capability
   - `fix`: bug fix
   - `docs`: documentation only
   - `refactor`: code restructure without behavior change
   - `test`: adding or updating tests
   - `chore`: build, tooling, dependency updates
   - `ci`: CI/CD pipeline changes

3. **Determine scope** — the module, file, or area affected (optional but helpful)

4. **Draft commit message** following Conventional Commits:
   ```
   type(scope): short description (max 72 chars)

   - Detail bullet 1
   - Detail bullet 2

   Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
   ```

5. **Show the drafted message** to the user and ask for approval before committing.

6. **After approval**, commit and push:
   ```bash
   git commit -m "<approved message>"
   git push -u origin <current-branch>
   ```

7. **Create PR** if on a feature branch (not `main`):
   ```bash
   gh pr create --title "<commit title>" --body "<summary>"
   ```

## Rules

- Never commit directly to `main` — if on `main`, prompt user to create a branch first
- Never use `--no-verify` or skip hooks
- Always show the commit message for approval before executing
- If nothing is staged, run `git status` and report back
