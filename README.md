# opus-coding — Claude Code Plugin

> A Claude Code plugin implementing the **Opus Multi-Model Coding Protocol**: four-phase parallel coding with Claude Opus planning, parallel GPT-5.4 Codex implementation threads, Claude Haiku summarization, and joint Opus↔Codex review.

---

## Install

**Option A — Use with `--plugin-dir` flag:**

```bash
git clone https://github.com/Kamisato520/opus-plan-codex-work.git
claude --plugin-dir ./opus-plan-codex-work
```

**Option B — Add to your project's `.claude.json`:**

```json
{
  "plugins": [
    { "path": "/absolute/path/to/opus-plan-codex-work" }
  ]
}
```

---

## Usage

In any Claude Code session after loading the plugin:

```
/opus-coding:opus-coding <task description>
```

Or naturally trigger it by saying: **`opus编码`**, **`多模型编码`**, or **`parallel coding`**

---

## How It Works

```
Phase 1 — Plan    Analyze codebase → decompose into 3–8 subtasks
Phase 2 — Code    Parallel GPT-5.4 Codex threads (max 8) implement each subtask
Phase 3 — Read    Claude Haiku summarizes outputs → writes files to disk
Phase 4 — Review  Opus + Codex relay review, up to 3 rounds → REVIEW_VERDICT.md
```

See [skills/opus-coding/README.md](./skills/opus-coding/README.md) for full documentation.

---

## Plugin Structure

```
opus-plan-codex-work/            ← git clone this
├── .claude-plugin/
│   └── plugin.json              ← Plugin manifest (required)
├── skills/
│   └── opus-coding/
│       ├── SKILL.md             ← Full protocol (machine-readable)
│       └── README.md            ← Skill documentation
└── README.md                    ← This file
```

---

## Requirements

- Claude Code ≥ 1.0.33
- `mcp__codex__codex` MCP tool (GPT-5.4 Codex access)
- `mcp__codex__codex-reply` MCP tool (for review relay)

---

## License

MIT © [Kamisato520](https://github.com/Kamisato520)
