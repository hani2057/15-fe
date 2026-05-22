# CLAUDE.md

## Response Format

- Keep individual responses focused and under ~500 tokens of output where possible; split large refactors into multiple turns
- For multi-file refactors, present a plan first, get confirmation, then execute file-by-file

## Convention (Variable Names & Function Parameters)

- Variables storing array-type values should end with a 'List'.
- Variables storing numeric count values should end with a 'Count'.
- When a function has two or more parameters, group them into an object and pass it.

## Behavioral guidelines

1. Think Before Coding

- Don't assume. Don't hide confusion. Surface tradeoffs.
- Before implementing:
  - State your assumptions explicitly. If uncertain, ask.
  - If multiple interpretations exist, present them - don't pick silently.
  - If a simpler approach exists, say so. Push back when warranted.
  - If something is unclear, stop. Name what's confusing. Ask.
  - If there is anything you need to decide before starting the work, ask.

2. Surgical Changes

- Touch only what you must. Clean up only your own mess.
- When editing existing code:
  - Don't "improve" adjacent code, comments, or formatting.
  - Don't refactor things that aren't broken.
  - Match existing style, even if you'd do it differently.
  - If you notice unrelated dead code, mention it - don't delete it.
- When your changes create orphans:
  - Remove imports/variables/functions that YOUR changes made unused.
  - Don't remove pre-existing dead code unless asked.
- The test: Every changed line should trace directly to the user's request.

3. Goal-Driven Execution

- Define success criteria. Loop until verified.
- Transform tasks into verifiable goals
- For multi-step tasks, state a brief plan:

```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```
