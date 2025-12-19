# SKILL.md Format Specification

## Required Structure

Every skill requires a `SKILL.md` file with:
1. **YAML frontmatter** (required fields: `name`, `description`)
2. **Markdown body** (instructions for using the skill)

## YAML Frontmatter

```yaml
---
name: skill-name
description: One-sentence description of what the skill does and when to use it
---
```

### Required Fields

| Field | Description |
|-------|-------------|
| `name` | Unique identifier for the skill (lowercase, hyphens) |
| `description` | Clear description that helps agent decide when to use this skill |

### Description Guidelines

The `description` is the primary trigger mechanism. Include:
- What the skill does
- When to use it (specific scenarios, file types, or tasks)

**Good example:**
```yaml
description: Structured approach for web research tasks requiring multiple sources and synthesis
```

**Bad example:**
```yaml
description: A skill for research
```

## Markdown Body

The body contains detailed instructions loaded when agent reads the file.

### Recommended Sections

```markdown
# Skill Name

## When to Use
- Scenario 1
- Scenario 2

## Process / Workflow
### Step 1: ...
### Step 2: ...
### Step 3: ...

## Best Practices
- Practice 1
- Practice 2

## Resources (if applicable)
- `scripts/helper.py` - Description
- `references/guide.md` - Description
```

## Complete Example

```yaml
---
name: web-research
description: Structured approach for conducting comprehensive web research using subagents for parallel information gathering
---

# Web Research Skill

This skill provides a structured approach to conducting comprehensive web research using the `task` tool to spawn research subagents.

## When to Use

Use this skill when you need to:
- Research complex topics requiring multiple information sources
- Gather and synthesize current information from the web
- Conduct comparative analysis across multiple subjects

## Research Process

### Step 1: Create and Save Research Plan

1. **Create a research folder**
   ```bash
   mkdir research_[topic_name]
   ```

2. **Write a research plan file** using `write_file`:
   - The main research question
   - 2-5 specific subtopics to investigate
   - How results will be synthesized

### Step 2: Delegate to Research Subagents

Use the `task` tool to spawn subagents:
- Clear, specific research question
- Instructions to write findings to: `research_[topic_name]/findings_[subtopic].md`
- Budget: 3-5 web searches maximum
- Run up to 3 subagents in parallel

### Step 3: Synthesize Findings

1. Run `list_files research_[topic_name]` to see created files
2. Use `read_file` for each findings file
3. Create comprehensive response with citations

## Best Practices

- Plan before delegating
- Clear subtopics with non-overlapping scope
- File-based communication between agents
- 3-5 searches per subtopic is usually sufficient
```

## Directory Structure

```
skill-name/
├── SKILL.md              # Required: YAML frontmatter + instructions
├── scripts/              # Optional: executable code
│   └── helper.py
├── references/           # Optional: documentation to load into context
│   └── guide.md
└── assets/               # Optional: files used in output
    └── template.txt
```

## Parsing Logic

```python
import re

def parse_skill_metadata(skill_md_path: Path) -> dict | None:
    content = skill_md_path.read_text(encoding="utf-8")

    # Match YAML frontmatter between --- delimiters
    frontmatter_pattern = r"^---\s*\n(.*?)\n---\s*\n"
    match = re.match(frontmatter_pattern, content, re.DOTALL)

    if not match:
        return None

    frontmatter = match.group(1)

    # Parse key-value pairs
    metadata = {}
    for line in frontmatter.split("\n"):
        kv_match = re.match(r"^(\w+):\s*(.+)$", line.strip())
        if kv_match:
            key, value = kv_match.groups()
            metadata[key] = value.strip()

    # Validate required fields
    if "name" not in metadata or "description" not in metadata:
        return None

    return metadata
```

## Override Hierarchy

```
User Skills:    ~/.myagent/skills/         (base)
Project Skills: .myagent/skills/           (overrides user skills with same name)
```

When both directories have a skill with the same name, the project skill takes precedence.
