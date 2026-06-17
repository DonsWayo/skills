# skills

Agent skills for AI coding agents (Claude Code, Cursor, Codex, and others), defined as `SKILL.md` files following the open [Agent Skills](https://agentskills.io) format.

## Skills

### [apple-container](skills/apple-container/SKILL.md)

Run Linux containers on macOS with Apple's [`container`](https://github.com/apple/container) CLI — the per-VM, Apple-silicon-native alternative to Docker Desktop. Covers the full command set, networking and DNS, migrating off Docker Desktop, multi-container orchestration with [Container-Compose](https://github.com/Mcrich23/Container-Compose), and troubleshooting.

The skill uses progressive disclosure: a lean `SKILL.md` for the everyday workflow, with depth in `references/`:

| File | Covers |
|------|--------|
| [`SKILL.md`](skills/apple-container/SKILL.md) | Quick start, command cheat sheet, top migration gotchas |
| [`references/commands.md`](skills/apple-container/references/commands.md) | Full command reference, every flag |
| [`references/networking.md`](skills/apple-container/references/networking.md) | Networking model, DNS, host↔container traffic, macOS 15 limits |
| [`references/docker-migration.md`](skills/apple-container/references/docker-migration.md) | Docker Desktop migration gotchas |
| [`references/container-compose.md`](skills/apple-container/references/container-compose.md) | Container-Compose field support matrix |
| [`references/troubleshooting.md`](skills/apple-container/references/troubleshooting.md) | Errors, hard limitations, TOML config |

## Install

### As an agent skill (skills.sh CLI)

Works with Claude Code, Cursor, Codex, and 70+ other agents:

```bash
npx skills add https://github.com/DonsWayo/skills --skill apple-container
```

Browse it at [skills.sh/donswayo/skills/apple-container](https://skills.sh/donswayo/skills/apple-container).

### As a Claude Code plugin

```
/plugin marketplace add https://github.com/DonsWayo/skills.git
/plugin install apple-container@donswayo-skills
```

> Use the full `.git` URL — the `owner/repo` shorthand can fail on the capitalized owner name.

## License

MIT
