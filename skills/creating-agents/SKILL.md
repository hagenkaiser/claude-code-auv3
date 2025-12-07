---
name: creating-agents
description: Guide for creating effective Claude Code agents and subagents. Use when building new agents, designing multi-agent systems, optimizing agent prompts, or configuring agent tools and permissions.
---

# Creating Optimal Claude Code Agents

## Agent File Structure

Agents are Markdown files with YAML frontmatter:

```markdown
---
name: agent-name
description: When to use this agent and what it does
tools: Read, Grep, Glob, Bash, Edit, Write  # Optional
model: sonnet  # Optional: sonnet, opus, haiku, inherit
skills: skill1, skill2  # Optional: auto-load skills
---

System prompt content here...
```

### File Locations

| Type | Path | Shared? |
|------|------|---------|
| Project | `.claude/agents/agent-name.md` | Yes (via git) |
| Personal | `~/.claude/agents/agent-name.md` | No |

## Frontmatter Reference

| Field | Purpose | Notes |
|-------|---------|-------|
| `name` | Identifier | Lowercase, hyphens, max 64 chars |
| `description` | When/why to invoke | Include trigger keywords |
| `tools` | Allowed tools | Omit to inherit all tools |
| `model` | Model selection | `haiku` (fast), `sonnet` (balanced), `opus` (complex) |
| `skills` | Auto-load skills | Comma-separated skill names |

## System Prompt Design

### Essential Elements

1. **Role Definition**: What the agent IS
2. **Expertise Area**: Domain focus
3. **Approach**: How to tackle problems
4. **Instructions**: Step-by-step procedures
5. **Constraints**: What NOT to do
6. **Tool Guidance**: When/how to use tools

### Prompt Best Practices

```markdown
# [Agent Role]

You are an expert [domain specialist] responsible for [core function].

## Core Responsibilities
- [Primary task 1]
- [Primary task 2]
- [Primary task 3]

## Approach
1. [First step - understand context]
2. [Second step - analyze/plan]
3. [Third step - execute]
4. [Fourth step - verify]

## Tool Usage
- Use Read for [specific purpose]
- Use Edit for [specific purpose]
- Use Bash for [specific purpose]

## Constraints
- Never [dangerous action]
- Always [safety requirement]
- Preserve [important invariant]
```

### Key Principles

- **Be Explicit**: Claude responds best to specific instructions
- **Include Examples**: Show expected behavior in prompts
- **Request Behaviors**: Explicitly ask for desired actions
- **Context Awareness**: Tell agent about compaction if relevant

## Tool Configuration

### Principle of Least Privilege

Only grant tools the agent actually needs:

```markdown
# Read-only analysis agent
tools: Read, Grep, Glob

# Code modification agent
tools: Read, Edit, Write, Grep, Glob

# Full development agent
tools: Read, Edit, Write, Grep, Glob, Bash
```

### Common Tool Sets

| Agent Type | Recommended Tools |
|------------|-------------------|
| Analyzer | Read, Grep, Glob |
| Reviewer | Read, Grep, Glob, Bash (for tests) |
| Implementer | Read, Edit, Write, Grep, Glob, Bash |
| Explorer | Read, Grep, Glob |

## Model Selection

| Model | Use For | Token Cost |
|-------|---------|------------|
| `haiku` | Simple, fast tasks | Lowest |
| `sonnet` | Balanced complexity | Medium |
| `opus` | Complex reasoning | Highest |

**Rule of thumb**: Start with `sonnet`, use `haiku` for simple delegation, reserve `opus` for complex analysis.

## What Makes an Agent Optimal

### Core Principles

1. **Single Responsibility**: One job done well
2. **Clear Scope**: Precise invocation description
3. **Focused Tools**: Minimal necessary access
4. **Explicit Instructions**: No assumptions
5. **Lightweight**: Under 3,000 tokens for coordination

### Characteristics of High-Performance Agents

- Clear input/output contracts
- Specific tool restrictions
- Detailed prompts with examples
- Right model for task complexity
- Well-defined boundaries

## Multi-Agent Architecture

### Recommended Pattern

```
Orchestrator (Coordinator)
├── Tools: Read, limited execution
├── Role: Planning, routing, state
└── Delegates to:
    ├── Specialized Agent 1
    ├── Specialized Agent 2
    └── Specialized Agent 3
```

### Best Practices

- Orchestrator maintains global state
- Subagents are focused and disposable
- Keep coordinator lightweight
- Run independent agents in parallel
- Chain dependent agents sequentially

### Avoid

- Heavy agents (25k+ tokens)
- Multiple competing orchestrators
- Giving all agents all tools

## Writing Good Descriptions

The description determines when Claude invokes your agent.

### Good Description

```yaml
description: Implements audio DSP for AUv3 instruments using AudioKit.
             Use for oscillators, filters, envelopes, effects, and
             voice management.
```

### Bad Description

```yaml
description: Handles audio stuff
```

### Include in Description

1. **What it does**: Core capability
2. **When to use**: Trigger conditions
3. **Keywords**: Terms users would say

## Complete Example

```markdown
---
name: code-reviewer
description: Reviews code for quality, security, and maintainability.
             Use after writing or modifying code, or when explicitly
             requested to review changes.
tools: Read, Grep, Glob, Bash
model: sonnet
---

# Code Reviewer

You are a senior code reviewer ensuring high standards of quality
and security.

## Process

1. Run `git diff` to see recent changes
2. Focus on modified files
3. Begin review immediately

## Review Checklist

- Code is simple and readable
- Functions and variables are well-named
- No duplicated code
- Proper error handling
- No exposed secrets or API keys
- Input validation implemented
- Good test coverage

## Output Format

Provide feedback organized by priority:

### Critical (must fix)
[Issues that block merge]

### Warnings (should fix)
[Issues that should be addressed]

### Suggestions (consider)
[Optional improvements]

Include specific examples of how to fix each issue.
```

## Testing Your Agent

1. Invoke with Task tool using your agent's `subagent_type`
2. Test various prompts matching your description
3. Verify tool usage matches expectations
4. Check output quality and format
5. Test with different models if needed

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Too broad scope | Split into focused agents |
| Vague description | Add specific triggers and keywords |
| Too many tools | Apply least privilege |
| No constraints | Add explicit boundaries |
| Missing examples | Include sample interactions |
| Heavy prompts | Keep under 3k tokens |
