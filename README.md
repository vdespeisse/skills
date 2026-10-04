# skills

Personal [Claude Code skills](https://docs.claude.com/en/docs/claude-code/skills), reusable across repos.

| Skill | What it does |
| --- | --- |
| [`ui-design`](skills/ui-design/SKILL.md) | Interactive design demos in a Vue / Nuxt UI app: several presets of one screen, switched from a dev-only floating panel, with the choice copied back as JSON. |

## Install

For every project (user-level), symlink so `git pull` keeps it current:

```sh
git clone git@github.com:vdespeisse/skills.git ~/dev/skills
mkdir -p ~/.claude/skills
ln -s ~/dev/skills/skills/ui-design ~/.claude/skills/ui-design
```

For one repo only, copy the folder into that repo's `.claude/skills/`.
