# Personalskills

A public collection of agent skills, each tested on a small concrete task before it is added.

## Layout

- `skills/<name>/SKILL.md` — Chinese skill body (frontmatter `name` is the kebab-case id)
- `skills/<name>/SKILL.en.md` — English counterpart (same structure; rename to `SKILL.md` when an English-only agent needs it)
- `skills/<name>/sources.md` — source video id and links
- `index.json` — catalog of every skill, including purpose, sources, and the test that was run
- `CHANGELOG.md` — what was added or changed, by date

Skills are recorded only after a real, executable test passes (or is explicitly marked as not run, with a reason).
