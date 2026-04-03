# MikeCodeur Skills

Agent Skills for YouTube creators and developers. Compatible with any agent that supports the [Agent Skills](https://agentskills.io) standard.

## Available Skills

| Skill | Description |
|-------|-------------|
| [youthumb-prompts](./youthumb-prompts/) | Generate optimized prompts for YouThumb.ai YouTube thumbnails |
| [youthumb-api](./youthumb-api/) | Interact with the YouThumb.ai API — upload assets, create projects, generate thumbnails |

## Compatibility

These skills follow the [Agent Skills specification](https://agentskills.io/specification) and work with:

- **Claude Code** — `~/.claude/skills/` or `.claude/skills/`
- **Cursor** — `.cursor/skills/`
- **GitHub Copilot** — `.github/skills/`
- **VS Code** — `.vscode/skills/`
- **Gemini CLI** — `.gemini/skills/`
- **OpenCode** — `.opencode/skills/`
- **Junie** — `.junie/skills/`
- **OpenHands, Goose, Amp, Letta, Firebender, Mux** — see their docs

## Installation

### Via skills.sh (recommended)

```bash
npx skillsadd MikeCodeur/skills/youthumb-prompts
```

### Via Claude Code plugin

```
/plugin marketplace add MikeCodeur/skills
```

### Manual

```bash
# Claude Code (personal)
git clone https://github.com/MikeCodeur/skills /tmp/mc-skills
cp -r /tmp/mc-skills/youthumb-prompts ~/.claude/skills/

# Claude Code (project)
cp -r /tmp/mc-skills/youthumb-prompts .claude/skills/

# Cursor
cp -r /tmp/mc-skills/youthumb-prompts .cursor/skills/

# Gemini CLI
cp -r /tmp/mc-skills/youthumb-prompts .gemini/skills/
```

## About

Made by [@MikeCodeur](https://youtube.com/@MikeCodeur_) — YouTuber, dev, and AI tools builder.

- 🎬 [YouTube](https://youtube.com/@MikeCodeur_)
- 🖼️ [YouThumb.ai](https://youthumb.ai) — AI-powered YouTube thumbnails
- 🚀 [Ship-SaaS.now](https://ship-saas.now) — Next.js SaaS boilerplate

## License

MIT
