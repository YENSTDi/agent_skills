---
name: langchain-skills
description: Implement Skills system on LangChain with progressive disclosure pattern. Use when building agent skills, implementing middleware for skill injection, or designing YAML-based skill discovery systems.
---

# LangChain Skills Implementation

This skill provides a complete guide to implementing the Skills pattern on LangChain, featuring progressive disclosure for efficient context management.

## Core Concept

```
Traditional: Load all skill instructions at once (context explosion)
Progressive Disclosure: Load metadata only, read full instructions on-demand
```

**Two-layer loading:**
- **Layer 1** (System Prompt): `name + description + path` (~80 tokens/skill)
- **Layer 2** (On-demand): Full SKILL.md content via `read_file`

## Implementation Workflow

### Step 1: Define Skill Format

Create `SKILL.md` with YAML frontmatter:

```yaml
---
name: web-research
description: Structured approach for web research tasks
---

# Web Research Skill

## When to Use
- Research complex topics
- Gather information from multiple sources

## Process
1. Create research plan
2. Delegate to subagents
3. Synthesize findings
```

See [references/skill-format.md](references/skill-format.md) for complete format specification.

### Step 2: Create SkillsMiddleware

Implement `AgentMiddleware` to inject skills into system prompt.

See [references/middleware-pattern.md](references/middleware-pattern.md) for complete implementation.

**Key methods:**
- `before_agent()`: Scan skills directory, extract YAML metadata
- `wrap_model_call()`: Inject skills list into system prompt

### Step 3: Integrate with Agent

```python
from langchain.agents.middleware.types import AgentMiddleware

agent_middleware = [
    SkillsMiddleware(
        skills_dir="~/.myagent/skills",
        assistant_id="myagent",
        project_skills_dir=".myagent/skills",  # optional
    ),
    # ... other middleware
]
```

## Progressive Disclosure Design

See [references/progressive-disclosure.md](references/progressive-disclosure.md) for design principles.

**System Prompt injection format:**

```markdown
## Skills System

**Available Skills:**

- **web-research**: Structured approach for web research tasks
  → Read `~/.myagent/skills/web-research/SKILL.md` for full instructions

- **code-review**: Code review with checklist
  → Read `~/.myagent/skills/code-review/SKILL.md` for full instructions
```

**Agent behavior:**
1. See skill list in system prompt
2. Match user request to skill description
3. Call `read_file(skill_path)` to load full instructions
4. Follow the skill's workflow

## Directory Structure

```
~/.myagent/skills/           # User-level (global)
├── web-research/
│   ├── SKILL.md             # Required: YAML frontmatter + instructions
│   └── helper.py            # Optional: supporting files
└── code-review/
    ├── SKILL.md
    └── checklist.md

.myagent/skills/             # Project-level (overrides user skills)
└── project-specific/
    └── SKILL.md
```
