# security-skills

A collection of Claude skills for security-focused engineering work — design
reviews, threat modeling, control mapping, and related documentation tasks.
Each skill lives in its own folder under `skills/` and can be installed
independently.

## Skills

| Skill | Description |
|---|---|
| [`security-design-review-dataflow-diagram`](skills/security-design-review-dataflow-diagram) | Generates a dataflow diagram and a per-endpoint-pair security-controls table from an RFC, design doc, or system description. |

More skills will be added here over time.

## Install

Each skill folder is self-contained. To install one:

- Copy the skill's folder into your skills directory (e.g. `/mnt/skills/user/`), or
- Install its packaged `.skill` file through your Claude client's skill picker.

## Adding a new skill

1. Create a new folder under `skills/<skill-name>/`.
2. Add a `SKILL.md` with YAML frontmatter (`name`, `description`) followed by
   the skill's instructions in Markdown.
3. Add a row to the table above.
4. Open a PR.

## License

MIT (or replace with your preferred license before publishing).
