---
name: create-rule
description: >
  Guides authoring of rule files for AI tool plugins. Rules are always-applied
  constraints or behavioral directives loaded into Claude's context automatically.
  Use when user wants to create a rule, add project-wide constraints, or asks
  "create a rule for X" / "/create-rule".
---

# Creating a Rule

## File Layout

```
plugins/<plugin-name>/rules/
└── <rule-name>.md     # one file per concern
```

For project-level rules: `plugins/<name>/rules/`.
For workspace-level rules: `.claude/rules/<name>.md` or CLAUDE.md sections.

## Frontmatter

```yaml
---
description: Short description of what this rule enforces
alwaysApply: true
---
```

**OR** scope to specific file globs:

```yaml
---
description: Dart naming conventions
globs:
  - "**/*.dart"
---
```

**Frontmatter fields:**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `description` | string | Yes | One line, what constraint is enforced |
| `alwaysApply` | bool | Either/or | Load unconditionally |
| `globs` | string[] | Either/or | Load only when matching files are in context |

Use `alwaysApply: true` for workflow rules (branching, commit format, toolchain).
Use `globs` for language/framework rules (only relevant when editing `.dart`, `.ts`, etc.).

## Body Content

Rules are directives — not documentation. Write as imperatives.

**Good:**
```markdown
- Always prefix Flutter/Dart commands with `fvm`
- Never commit directly to `main`; branch from the target branch first
- Use `fvm dart run melos dart-format` for formatting, not `dart format .`
```

**Bad:**
```markdown
This rule explains that the team uses FVM for Flutter version management because...
```

Keep rules short and scannable. If a rule requires explanation, add a one-line reason inline:

```markdown
- Use `fvm` prefix for all Flutter/Dart commands — version pinned per-project via `.fvmrc`
```

## Recommended Structure

```markdown
---
description: <what constraint>
alwaysApply: true
---

# <Rule Name>

[Optional: one-line context for non-obvious constraints]

- Rule 1
- Rule 2
- Rule 3
```

For complex rules with multiple concerns, use sections:

```markdown
## Toolchain

- ...

## Branching

- ...

## Quality Gates

- ...
```

## When to Create a Rule vs. a Skill

| Use Rule | Use Skill |
|----------|-----------|
| Always-on constraint ("never do X", "always use Y") | Multi-step workflow ("how to do X") |
| No user invocation needed | User invokes explicitly |
| Single behavioral directive | Sequence of decisions and actions |
| Short — fits in ~50 lines | Can be 500 lines with references |

## Checklist Before Shipping

```
- [ ] Frontmatter has `description` field
- [ ] Either `alwaysApply: true` OR `globs` array present (not both, not neither)
- [ ] Body is imperative directives, not prose explanation
- [ ] Filename is lowercase, hyphens only
- [ ] File placed in correct `rules/` directory
- [ ] No emoji — use plain words (MUST/NEVER/WARNING) not ✅/❌/⚠️
- [ ] No time-sensitive info ("before v3 use old API")
- [ ] Tested: rule loads correctly and guides expected behavior
```

## Anti-Patterns

| Avoid | Do instead |
|-------|-----------|
| Explaining WHY at length | One-line reason inline, move detail to a skill |
| `alwaysApply: true` + `globs` both set | Pick one scoping method |
| Rules longer than 100 lines | Split into multiple focused rule files |
| Duplicate content from skills/agents | Cross-reference: "See `@workflow` skill for full steps" |
| Vague constraints ("write good code") | Specific, actionable directives ("use `const` constructors where possible") |
