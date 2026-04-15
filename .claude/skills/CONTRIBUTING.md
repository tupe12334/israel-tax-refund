# Skills Contributing Guidelines

## Creating or Updating a Skill

### Rules

1. **Never set `disable-model-invocation: true`.**
   All skills must be invocable via the Skill tool (agent). Do not add `disable-model-invocation: true` to any skill's frontmatter.

2. **Save user-provided information immediately.**
   Any data, answer, or detail the user provides during a skill's flow must be persisted to disk the moment it is received — before asking the next question, before any processing, and before the skill ends. Never batch saves to the end of a session or step.
