---
name: create-agent
description: >
  Guides authoring of agent files for AI tool plugins. Agents are specialized
  subagents with a defined persona, expertise, and process steps. Use when user
  wants to create a subagent, add a specialized role, or asks "create an agent
  for X" / "/create-agent".
---

# Creating an Agent

## File Layout

```
plugins/<plugin-name>/agents/
└── <agent-name>.md     # one file per agent role
```

## Frontmatter

```yaml
---
name: agent-name
description: One-line expert summary. Include trigger phrases for when to invoke.
---
```

**Frontmatter fields:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | Yes | Lowercase, hyphens only, ≤64 chars. Must match filename |
| `description` | string | Yes | 3rd person. What expertise + when to invoke. Used for auto-selection |

**Description rules (same as skills):**
- 3rd person: "Reviews Dart code" not "I review Dart code"
- Include trigger phrases: "Use immediately after writing code" or "Invoke with /code-reviewer"
- Specific beats generic

## Body Content

Agent body = system prompt for a specialized Claude instance. Write as "You are..." persona.

**Structure:**

```markdown
You are a [role] for [project context]. You [core expertise].

## Process (follow in order)

When invoked:

1. **Step name** – What to do. Be specific about commands or checks to run.

2. **Step name** – ...

## [Domain Checklist / Standards]

**Category**
- Specific standard
- Another standard

## Output format

How to present results (ordered sections, severity levels, etc.)
```

**Key principles:**
- Agent has a single, focused domain — don't create generalist agents
- Process steps are numbered and ordered — agent must not skip or reorder
- Include concrete commands (e.g. `dart analyze`, `fvm flutter test`) not vague instructions
- Checklist items are specific and actionable, not vague ("names are descriptive" not "good naming")

## When to Create an Agent vs. a Skill

| Use Agent | Use Skill |
|-----------|-----------|
| Needs specialized persona/expertise | General workflow, no persona needed |
| Runs as subagent (delegated task) | Runs in main Claude context |
| Has a defined checklist/review process | Has a multi-step workflow |
| Invoked via `/agent-name` delegation | Invoked by user directly |
| e.g. code-reviewer, test-writer | e.g. git workflow, clean architecture guide |

## Delegation Pattern

Agents are typically invoked from rules or commands like:

```markdown
After finishing code changes: use `/code-reviewer` to review all changed files.
```

Or from development workflow rules using a delegation table:

```markdown
| Task | Invoke with | When to use |
|------|-------------|-------------|
| Code review | `/code-reviewer` | After any code change |
```

## Checklist Before Shipping

```
- [ ] name: lowercase + hyphens, ≤64 chars, matches filename
- [ ] description: 3rd person, what expertise + trigger phrases
- [ ] Body opens with "You are..." persona statement
- [ ] Process steps numbered and ordered
- [ ] Steps include concrete commands, not vague instructions
- [ ] Domain checklist is specific and actionable
- [ ] Output format section tells agent how to structure its response
- [ ] Single focused domain — no generalist agents
- [ ] No emoji — plain words (MUST/CRITICAL/WARNING) not ✅/❌/⚠️
- [ ] No time-sensitive info
- [ ] Tested: agent produces correct focused output when invoked
```

## Anti-Patterns

| Avoid | Do instead |
|-------|-----------|
| "Review everything" generalist | Focused domain (data layer, test suite, security) |
| Vague steps ("review the code") | Specific commands + criteria ("run `dart analyze`, treat every warning as review item") |
| Persona-free bodies | Open with "You are a [expert role]..." |
| Checklist items too broad ("good architecture") | Specific tests ("Presentation depends on domain; domain has no `context` imports") |
| Agent duplicating skill content | Agent references skill: "See `@clean` for architecture rules" |
