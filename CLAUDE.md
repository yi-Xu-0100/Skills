# Skills Repository

This repository is a personal collection of Claude skills. Each skill is a folder under `skills/<skill-name>/` containing a `SKILL.md` file.

## Adding a New Skill

1. Create `skills/<kebab-case-name>/SKILL.md`
2. Frontmatter must include `name` and `description`
3. The description is the primary trigger mechanism — describe both what the skill does and when Claude should activate it

## Conventions

- Skill names are kebab-case (e.g., `lf-to-crlf`)
- Descriptions are in Chinese for skills targeting Chinese-language workflows, English otherwise
- Instructions are in markdown with progressive disclosure (summary first, details below)
- If a skill needs executable scripts, place them in `scripts/` inside the skill folder
- If a skill needs reference documents, place them in `references/`

## Current Skills

- `lf-to-crlf` — Converts LF line endings to CRLF in modified git files
