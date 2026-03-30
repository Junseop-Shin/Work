# /explain — Deep Code Explanation

Provide a thorough explanation of the specified file, function, or code section.

## Steps

1. **Identify target**
   - Use the argument as file path, function name, or concept
   - If no argument: explain the most recently modified file

2. **Read the target file(s)** — also read imports and dependencies as needed

3. **Produce explanation at three levels:**

   ### 1. What It Does (One Paragraph)
   Plain English summary of the purpose and behavior. No jargon.

   ### 2. How It Works (Step by Step)
   Walk through the logic flow:
   - Entry point and inputs
   - Each major step with the "why" not just the "what"
   - Output and side effects
   - Error cases and how they're handled

   ### 3. Key Patterns & Decisions
   - Design patterns used (and why)
   - Non-obvious choices with rationale
   - Dependencies and what they contribute
   - Potential gotchas or edge cases to be aware of

4. **Data flow diagram** (text-based if helpful):
   ```
   Input → [Validation] → [Transform] → [Persist] → Output
                                ↓
                          [Error Handler]
   ```

5. **Related files** to read next if deeper understanding is needed

## Rules

- Tailor depth to complexity — simple functions need 3 lines, complex modules need full breakdown
- Use concrete examples with actual values from the code
- Highlight any security-relevant behavior
- Note if anything looks wrong or unexpected (but don't fix without being asked)
