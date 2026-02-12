# OpenClaw CC Pipeline

This project defines a Claude Code skill for orchestrating multi-turn tasks via a Discord Bot.

> 这个项目定义了一个 Claude Code Skill，通过 Discord Bot 调度多轮 CC 任务。

## Skill Definition

The main skill file is **[SKILL.md](SKILL.md)**. It contains the complete multi-turn orchestration protocol that Claude Code follows when dispatched by an OpenClaw Bot.

## How It Works

```
Discord User → OpenClaw Bot → Task API → Worker → Claude Code CLI → Callback → Discord
```

Each round preserves full CC context via `--session-id` / `--resume`. The user has explicit control between rounds.

## Companion Projects

This skill works in conjunction with:

- **[openclaw-worker](https://github.com/AliceLJY/openclaw-worker)** -- Task API (`server.js`) and Worker (`worker.js`) that relay tasks and deliver callbacks. Required infrastructure.
- **[content-alchemy](https://github.com/AliceLJY/content-alchemy)** -- WeChat article writing skill. The primary real-world use case for this pipeline (3-round workflow).
- **[OpenClaw](https://github.com/openclaw/openclaw)** -- The Discord Bot framework that receives user commands and displays results.

## Project Structure

```
SKILL.md                              # Core skill definition (install this)
CLAUDE.md                             # This file (project context)
README.md                             # Architecture docs and installation guide
examples/
  bot-memory-snippet.md               # Ready-to-paste Bot memory config
  content-alchemy-3-round.md          # 3-round Content Alchemy workflow details
  openclaw-hooks-config.json          # OpenClaw hooks configuration snippet
```

## Notes

- The `examples/` directory contains reference configurations. Do not modify these files during skill execution.
- README.md contains detailed architecture documentation and pitfall guides for humans setting up the infrastructure.
- SKILL.md is written in imperative style for Claude Code to follow. README.md is written in explanatory style for human readers.
