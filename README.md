# skills

[![skills.sh](https://skills.sh/b/devroy10/skills)](https://skills.sh/devroy10/skills)

Agent skills, installable via [skills.sh](https://skills.sh).

## Skills

- **capture-learning** - Capture a notable discovery, lesson, or answered question from the current session into a structured learning document.

## Install

```bash
npx skills add devroy10/skills
```

To install a single skill:

```bash
npx skills add devroy10/skills@capture-learning
```

## Development

Skills live in `skills/<skill-name>/SKILL.md` and follow the [Agent Skills specification](https://agentskills.io).

Test discovery locally before publishing:

```bash
npx skills add . --list
```
