# /cleanup — Dead Code and Import Cleanup

Remove unused code, dead imports, and redundant logic from specified files.

## Steps

1. **Identify target**
   - If argument provided: clean that file
   - Otherwise: `git diff --name-only HEAD` for recently changed files

2. **Read each file** fully before making any changes

3. **Identify for removal:**

   ### Unused Imports
   - Imports not referenced anywhere in the file
   - Type imports that can be inlined or removed

   ### Dead Code
   - Functions/variables declared but never called/referenced
   - Commented-out code blocks (remove unless they explain current code)
   - `console.log`, `console.debug` debugging statements
   - TODO comments older than the current feature (flag, don't remove)

   ### Redundant Logic
   - Duplicate conditions or branches
   - Variables assigned but immediately overwritten
   - Early returns that make later code unreachable

   ### Type Redundancy (TypeScript)
   - Explicit type annotations that TypeScript infers automatically
   - `as any` casts that can be replaced with proper types

4. **Apply changes** using Edit tool — minimal diffs only

5. **Report what was removed:**
   ```
   ## Cleanup Report — <file>
   - Removed imports: X (list them)
   - Removed dead functions: Y (list them)
   - Removed debug logs: Z
   - Simplified conditions: (describe)
   ```

## Rules

- Only remove code that is **provably unused** — no guessing
- Do NOT remove TODOs that are still relevant — flag them instead
- Do NOT refactor logic — only remove dead code
- Run `tsc --noEmit` or `eslint` after changes if available
- If unsure whether something is used, leave it and note it
