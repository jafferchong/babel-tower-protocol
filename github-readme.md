# Babel Tower — Universal Agent Communication Protocol

> **让所有平台的 AI 智能体说同一种话**
>
> A file-based, cross-platform communication protocol for AI agents. Works with WorkBuddy, Claude Code, Codex CLI, Cursor, and any tool that can read/write files.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Python 3.8+](https://img.shields.io/badge/Python-3.8+-green.svg)](https://python.org)

---

## 🤔 Why Babel Tower?

AI agents today suffer from the "Tower of Babel" problem. An agent in WorkBuddy can't talk to an agent in Claude Code. A Codex CLI task can't hand off work to a Cursor sub-agent. Each platform has its own communication mechanism — and they don't speak the same language.

**Babel Tower solves this with the simplest possible common denominator: the file system.**

Every AI tool can read and write files. Babel Tower turns that into a universal message bus using JSONL inbox files.

## ✨ Features

- **Zero dependencies** — Pure Python 3.8+, file system only
- **Task-scoped communication** — Each task gets its own inbox directory
- **Global inbox** — Cross-task notifications for session-spanning messages
- **12 message types** — task_assign, handoff, block, ack, broadcast, etc.
- **Handshake protocol** — "I'm done, you continue" between agents
- **File-locked writes** — Safe concurrent access across platforms
- **Human-readable archives** — Auto-generated conversation summaries
- **Cross-platform Skill** — Load the babel-tower Skill on any platform

## 📦 Quick Start

### 1. Install

```bash
git clone https://github.com/YOUR_USERNAME/babel-tower.git
cd babel-tower
```

No pip install needed — it's just 4 Python scripts.

### 2. Initialize a task's Babel Tower

```bash
python scripts/babel-init.py \
  --task TASK-20260612-001 \
  --members "orchestrator,frontend-dev,backend-dev" \
  --task-name "User auth refactor"
```

### 3. Send messages

```bash
# Task-scoped message
python scripts/babel-send.py \
  --task TASK-20260612-001 \
  --to frontend-dev --type handoff \
  --subject "Backend done" --body "API ready, start frontend work"

# Global message (cross-task notification)
python scripts/babel-send.py \
  --global --to orchestrator \
  --type query --subject "Reminder" --body "Check DB migration status"
```

### 4. Check inbox

```bash
python scripts/babel-check.py --task TASK-20260612-001 --all --unread
python scripts/babel-check.py --global --all --unread
```

### 5. Generate conversation summary

```bash
python scripts/babel-summary.py --task TASK-20260612-001
```

## 🏗️ Architecture

```
workspace/
├── babel-tower/                    # Protocol root
│   ├── scripts/                    # babel-init/send/check/summary
│   ├── protocols/                  # message-schema.json
│   └── templates/                  # manifest.yaml
│
└── tasks/active/TASK-xxx/
    └── babel-tower/                # Task-scoped instance
        ├── manifest.yaml           # Member registry
        ├── inbox/{agent}.jsonl     # Per-agent inbox
        └── .state.json             # Read status tracking
```

See [architecture diagram](docs/architecture-diagram.svg) for full layout.

## 🔌 Cross-Platform Usage

Babel Tower is not tied to any framework. If your agent system supports file I/O, it can join:

### For WorkBuddy
Load `babel-tower` Skill from the skill marketplace or local directory.

### For Claude Code
```bash
# Read your inbox
cat workspace/tasks/TASK-xxx/babel-tower/inbox/claude-agent.jsonl

# Send a handoff
python scripts/babel-send.py --task TASK-xxx --to codex-agent --type handoff ...
```

### For Codex CLI
Same as Claude Code — file system is the universal API.

### For any other tool
If it can `open()` and `write()`, it can use Babel Tower. See [PROTOCOL.md](docs/PROTOCOL.md) for the full message specification.

## 📋 Message Schema

```json
{
  "id": "msg_20260612_173200_a1b2c3",
  "from": "backend-dev",
  "to": "frontend-dev",
  "type": "handoff",
  "priority": "P1",
  "subject": "API module complete",
  "body": "POST /auth/login is ready with tests",
  "task_id": "TASK-20260612-001",
  "artifacts": ["deliverables/auth-api-spec.yaml"],
  "created_at": "2026-06-12T17:32:00+08:00"
}
```

## 🌍 Integrate with Your System

Three steps to integrate Babel Tower into any agent system:

1. **Locate the inbox** — Find `babel-tower/inbox/{your_agent_id}.jsonl` in the active task
2. **Read on startup** — Check your inbox for pending messages at session start
3. **Write to communicate** — Append JSONL lines to other agents' inbox files

If your system uses pre-injected prompts (like GEM's SOUL.md), add the Babel Tower protocol rules to your system prompt. The protocol is designed to be framework-agnostic.

## 📄 License

MIT — Use it, fork it, embed it anywhere.

---

*Built by [@YOUR_HANDLE] in one afternoon. Inspired by the WorkBuddy Agent Teams internal design and the biblical Tower of Babel.*
