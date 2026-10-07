# security-skills

A collection of Claude skills for security-focused engineering work — design
reviews, threat modeling, control mapping, and related documentation tasks.
Each skill lives in its own folder under `skills/` and can be installed
independently.

## Skills

| Skill | Description |
|---|---|
| [`security-design-review-dataflow-diagram`](skills/security-design-review-dataflow-diagram) | Generates a dataflow diagram (actors, APIs, services, databases, and the direction data moves between them) from an RFC, design doc, or system description. Edges are colored by stated encryption status and marked where they carry sensitive data. See [how it compares to generating a diagram without the skill](skills/security-design-review-dataflow-diagram/what_difference_from_claude_native_skill.md). |
| [`security-design-review-controls-table`](skills/security-design-review-controls-table) | Generates a per-endpoint-pair security-controls table (authentication, authorization, sanitization, data protection, logging) from an RFC, design doc, or system description. See [its README](skills/security-design-review-controls-table/README.md) for why this is useful in a security review, and [`examples/profile-service-example.md`](skills/security-design-review-controls-table/examples/profile-service-example.md) for sample output. |

These two are siblings and designed to be used together or independently —
each extracts its own copy of the RFC's endpoint-pair data, so neither
requires the other to be installed. More skills will be added here over time.

## Examples

[`examples/profile-service-rfc.md`](examples/profile-service-rfc.md) is the sample RFC both skills' example output was generated from — a "Profile Service" covering notification preferences, notification sharing, and account-profile (PII) management. Use it to try either skill yourself and compare your output against the examples linked above.

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
