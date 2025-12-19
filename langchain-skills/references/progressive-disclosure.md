# Progressive Disclosure Design

## The Problem

Traditional approach loads all skill instructions into system prompt:

```
System Prompt
├── Skill 1 full instructions (3K tokens)
├── Skill 2 full instructions (5K tokens)
├── Skill 3 full instructions (4K tokens)
└── ...

Problem: Context explosion, accuracy drops, high cost
```

## The Solution

Progressive disclosure loads metadata only, full content on-demand:

```
System Prompt (lean)
├── Skill 1: name + description + path (80 tokens)
├── Skill 2: name + description + path (80 tokens)
└── ...

Agent needs skill → read_file(skill_path) → Full instructions
```

## Two-Layer Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: Always Visible (System Prompt)                    │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ name: "web-research"                                  │ │
│  │ description: "Use this skill for web research..."    │ │
│  │ path: "~/.myagent/skills/web-research/SKILL.md"      │ │
│  └───────────────────────────────────────────────────────┘ │
│                          │                                  │
│                          ▼ Agent calls read_file            │
│                                                             │
│  Layer 2: On-Demand                                         │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ # Web Research Skill                                  │ │
│  │ ## When to Use                                        │ │
│  │ ## Research Process                                   │ │
│  │ ### Step 1: Create Research Plan                      │ │
│  │ ### Step 2: Delegate to Subagents                     │ │
│  │ ### Step 3: Synthesize Findings                       │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Layer Comparison

| Layer | Content | When Visible | Token Cost |
|-------|---------|--------------|------------|
| **Layer 1** | name + description + path | Always in System Prompt | ~80 tokens/skill |
| **Layer 2** | Full SKILL.md content | When Agent calls read_file | On-demand |

## System Prompt Template

```python
SKILLS_SYSTEM_PROMPT = """

## Skills System

You have access to a skills library that provides specialized capabilities.

{skills_locations}

**Available Skills:**

{skills_list}

**How to Use Skills (Progressive Disclosure):**

Skills follow a **progressive disclosure** pattern - you know they exist
(name + description above), but you only read the full instructions when needed:

1. **Recognize when a skill applies**: Check if the user's task matches any skill's description
2. **Read the skill's full instructions**: Use read_file with the path shown above
3. **Follow the skill's instructions**: SKILL.md contains step-by-step workflows
4. **Access supporting files**: Skills may include scripts, configs, or reference docs

**When to Use Skills:**
- When the user's request matches a skill's domain
- When you need specialized knowledge or structured workflows
- When a skill provides proven patterns for complex tasks

Remember: Skills are tools to make you more capable. When in doubt, check if a skill exists!
"""
```

## Skills List Format

```python
def _format_skills_list(skills: list[SkillMetadata]) -> str:
    """Format skills for system prompt display."""
    if not skills:
        return "(No skills available yet.)"

    # Group by source
    user_skills = [s for s in skills if s["source"] == "user"]
    project_skills = [s for s in skills if s["source"] == "project"]

    lines = []

    if user_skills:
        lines.append("**User Skills:**")
        for skill in user_skills:
            lines.append(f"- **{skill['name']}**: {skill['description']}")
            lines.append(f"  → Read `{skill['path']}` for full instructions")
        lines.append("")

    if project_skills:
        lines.append("**Project Skills:**")
        for skill in project_skills:
            lines.append(f"- **{skill['name']}**: {skill['description']}")
            lines.append(f"  → Read `{skill['path']}` for full instructions")

    return "\n".join(lines)
```

## Example Output

```markdown
## Skills System

**User Skills**: `~/.myagent/skills`
**Project Skills**: `.myagent/skills` (overrides user skills)

**Available Skills:**

**User Skills:**
- **web-research**: Structured approach for web research tasks
  → Read `~/.myagent/skills/web-research/SKILL.md` for full instructions
- **code-review**: Code review with checklist
  → Read `~/.myagent/skills/code-review/SKILL.md` for full instructions

**Project Skills:**
- **project-deploy**: Project-specific deployment workflow
  → Read `.myagent/skills/project-deploy/SKILL.md` for full instructions
```

## Agent Execution Flow

```
User: "Help me research quantum computing trends"
      │
      ▼
Agent: Sees skill list in system prompt
      │
      ▼
Agent: Matches "research" → "web-research" skill
      │
      ▼
Agent: read_file("~/.myagent/skills/web-research/SKILL.md")
      │
      ▼
Agent: Gets full instructions, follows 3-step workflow:
      1. Create research_quantum_computing/ and plan
      2. Spawn 3 parallel research subagents
      3. Synthesize findings into final report
```

## Benefits

| Benefit | Description |
|---------|-------------|
| **Token Efficient** | Only ~80 tokens/skill in system prompt |
| **Scalable** | Can have many skills without context explosion |
| **Self-Documenting** | Agent reads instructions when needed |
| **LLM Agnostic** | Works with any model that has read_file capability |
| **Flexible** | Skills can contain scripts, configs, references |
