# Personal Skills

A personal collection of custom skills for Claude. Skills teach Claude how to perform specialized, repeatable tasks.

## Structure

```
skills/
├── <skill-name>/
│   └── SKILL.md       # Instructions + YAML frontmatter
└── ...
template/
└── SKILL.md            # Template for new skills
```

## Usage

Skills are loaded by Claude when their description matches the current task. Each skill lives in its own folder under `skills/` and contains a `SKILL.md` file with YAML frontmatter (`name`, `description`) and markdown instructions.

## Creating a New Skill

1. Create a new folder under `skills/<your-skill-name>/`
2. Copy `template/SKILL.md` as a starting point
3. Fill in the `name` (kebab-case) and `description` fields
4. Write the skill instructions in markdown below the frontmatter
5. Optionally add `scripts/`, `references/`, or `assets/` subfolders
