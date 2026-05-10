# opc-skills

A personal collection of [Claude Code](https://claude.com/claude-code) skills — practical, hands-on workflows distilled from real use.

## Skills

| Skill | What it does |
|---|---|
| [`video-subtitles`](./skills/video-subtitles/) | End-to-end pipeline for adding burned-in subtitles to screen recordings on macOS using ffmpeg + whisper.cpp + Subtitle Edit. |

## Install

This repo is structured as a Claude Code plugin. To use it locally:

```bash
# Clone anywhere
git clone <this-repo> ~/code/opc-skills

# Symlink into your Claude Code plugins directory
ln -s ~/code/opc-skills ~/.claude/plugins/opc-skills
```

Restart Claude Code. The skills will appear in your available-skills list and can be invoked via the `Skill` tool or by name.

## Layout

```
opc-skills/
├── .claude-plugin/
│   └── plugin.json         # plugin manifest
├── skills/
│   └── <skill-name>/
│       └── SKILL.md        # skill definition (frontmatter + body)
└── README.md
```

Each skill is a directory under `skills/` containing at minimum a `SKILL.md` with YAML frontmatter (`name`, `description`) followed by the skill body. See [`skills/video-subtitles/SKILL.md`](./skills/video-subtitles/SKILL.md) for an example.

## Adding a new skill

1. Create `skills/<skill-name>/SKILL.md`
2. Add YAML frontmatter — `name` (kebab-case) and a `description` that lists the trigger phrases Claude should match on
3. Write the body as a clear, step-by-step workflow
4. Add the skill to the table above

## License

MIT
