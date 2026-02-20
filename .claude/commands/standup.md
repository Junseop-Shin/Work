# /standup — Daily Standup Note Generator

Summarize today's commits and work into a structured standup report.

## Steps

1. **Get today's commits**
   ```bash
   git log --since="midnight" --until="now" --pretty=format:"%h %s" --author="$(git config user.name)"
   git log --since="midnight" --until="now" --stat
   ```

2. **If no commits today**, check recent activity:
   ```bash
   git log --since="yesterday" --pretty=format:"%h %s" -10
   ```

3. **Read commit diffs** for context on what changed:
   ```bash
   git show --stat <commit-hash>
   ```

4. **Generate standup report**:

   ```markdown
   ## Daily Standup — <date>

   ### Yesterday / Since Last Update
   - <summary of what was completed>
   - Commits:
     - `abc1234` feat(auth): add JWT refresh token logic
     - `def5678` fix(api): resolve null pointer in user endpoint

   ### Today
   - <planned work based on open tasks or next logical steps>

   ### Blockers
   - <any blockers, or "None">

   ### Notes
   - <any relevant context for the team>
   ```

5. **Output the report** to the user

## Rules

- Base "Yesterday" section on actual git commits, not assumptions
- Keep each item concise — one line per task
- List blockers explicitly; if none, say "None"
- If working on a feature branch, note the branch name
- Format dates as YYYY-MM-DD
