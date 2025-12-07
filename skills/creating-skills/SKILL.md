---
name: creating-skills
description: Guide for creating effective Claude Code skills. Use when building new skills, improving existing skills, or learning skill best practices.
---

# Creating Effective Claude Code Skills

## What Makes a Good Skill

A good skill is **focused**, **discoverable**, and **well-structured**. It solves one problem well rather than many problems poorly.

## Skill Structure

### Minimal Structure
```
my-skill/
└── SKILL.md (required)
```

### Complete Structure (for complex skills)
```
my-skill/
├── SKILL.md          # Core instructions (required)
├── reference.md      # Detailed documentation
├── examples.md       # Usage examples
├── scripts/          # Helper scripts
└── templates/        # Reusable templates
```

## SKILL.md Format

```yaml
---
name: skill-name-here
description: What it does and when to use it. Be specific about triggers.
allowed-tools: Read, Grep, Glob  # Optional: restrict tool access
---

# Skill Title

## Instructions
Step-by-step guidance for Claude.

## Examples
Concrete usage examples.
```

## Frontmatter Rules

| Field | Requirements |
|-------|--------------|
| `name` | Lowercase, hyphens only. Max 64 chars. No "anthropic" or "claude". |
| `description` | What it does + when to use it. Max 1024 chars. Include trigger keywords. |
| `allowed-tools` | Optional. Comma-separated list to restrict tool access. |

## Writing Good Descriptions

The description determines when Claude uses your skill. Include:

1. **What it does** - The capability provided
2. **When to use it** - Trigger conditions
3. **Keyword triggers** - Words users would say

### Good Description
```yaml
description: Extract text and tables from PDF files, fill forms, merge documents. Use when working with PDF files or when the user mentions PDFs, forms, or document extraction.
```

### Bad Description (too vague)
```yaml
description: Helps with documents
```

## Best Practices Checklist

### Focus
- [ ] One skill = one capability
- [ ] Skill name clearly indicates purpose
- [ ] Can explain what it does in one sentence

### Content
- [ ] SKILL.md under 500 lines
- [ ] Longer content split into separate files
- [ ] Only includes info Claude doesn't already know
- [ ] Uses consistent terminology throughout

### Progressive Disclosure
- [ ] Essential info in SKILL.md
- [ ] Advanced topics in separate files
- [ ] Files loaded only when needed

### Discoverability
- [ ] Description includes trigger keywords
- [ ] Description explains what AND when
- [ ] Tested with queries users would actually ask

## Common Mistakes to Avoid

1. **Too broad** - "data-tools" should be split by data type
2. **Vague description** - "Helps with stuff" won't trigger
3. **Too much content** - 2000-line SKILL.md slows Claude down
4. **Missing triggers** - Description lacks keywords users say
5. **Inconsistent terms** - Mixing "endpoint"/"URL"/"route" confuses Claude

## Storage Locations

| Type | Path | Shared? |
|------|------|---------|
| Personal | `~/.claude/skills/` | No |
| Project | `.claude/skills/` | Yes (via git) |

## Testing Your Skill

1. Ask questions that match your description keywords
2. Claude should automatically discover and use the skill
3. Test with different phrasings
4. Test with Haiku, Sonnet, and Opus (they behave differently)

## Example: Well-Structured Skill

```yaml
---
name: generating-commit-messages
description: Generate clear git commit messages from staged changes. Use when writing commits, reviewing diffs, or preparing releases.
---

# Generating Commit Messages

## Process

1. Run `git diff --staged` to see changes
2. Identify the type: feat, fix, refactor, docs, test, chore
3. Write summary under 50 characters (imperative mood)
4. Add body explaining what and why (not how)

## Format

```
<type>: <summary>

<body>
```

## Examples

**Good:** `fix: prevent race condition in user auth`
**Bad:** `fixed stuff` or `Update code`
```

## Quick Reference

```
mkdir -p ~/.claude/skills/my-skill    # Create skill directory
# Write SKILL.md with frontmatter + content
# Test by asking Claude questions that match your description
```
