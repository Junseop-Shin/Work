# /plan — Architecture Design Document Generator

Invoke the Opus sub-agent to produce a thorough architecture design document for the given task or feature.

## Steps

1. **Gather context** — read relevant existing files:
   - Project structure (`ls`, `Glob`)
   - Existing architecture patterns
   - Related code files
   - Any existing design documents

2. **Delegate to Opus sub-agent** via the Task tool with `model: opus`:
   - Full problem description
   - Constraints and requirements
   - Current codebase context

3. **Produce architecture document** with this structure:

   ```markdown
   # Architecture Plan — <feature/task name>
   ## Date: <date>

   ## Problem Statement
   What we're solving and why.

   ## Requirements
   ### Functional
   - ...
   ### Non-Functional
   - Performance, security, scalability constraints

   ## Proposed Solution
   High-level approach and rationale.

   ## System Design
   ### Components
   | Component | Responsibility | Technology |
   |-----------|---------------|------------|

   ### Data Flow
   1. Step 1
   2. Step 2

   ### API/Interface Design
   Key interfaces or APIs if applicable.

   ## Trade-offs Considered
   | Option | Pros | Cons | Decision |
   |--------|------|------|---------|

   ## Implementation Plan
   ### Phase 1 — <name>
   - [ ] Task 1
   - [ ] Task 2

   ### Phase 2 — <name>
   - [ ] Task 3

   ## Risk Assessment
   | Risk | Likelihood | Impact | Mitigation |
   |------|-----------|--------|-----------|

   ## Open Questions
   - Question 1
   - Question 2
   ```

4. **Present the document** to the user for review and discussion before implementation begins

## Rules

- Use Opus model for the design phase — this is an architecture decision task
- Do not start implementation until the user approves the plan
- Flag open questions explicitly rather than making assumptions
- Consider security implications in the design
