# opus-plan-codex-work — Claude Code Plugin Marketplace

> A Claude Code **plugin marketplace** hosting the **opus-coding** skill: four-phase parallel coding with Claude Opus, GPT-5.4 Codex threads, Haiku summarization, and joint review.

---

## Install

**Step 1 — Add this marketplace:**

```
/plugin marketplace add https://github.com/Kamisato520/opus-plan-codex-work
```

**Step 2 — Install the plugin:**

```
/plugin install opus-coding
```

---

## Usage

After installation, trigger in any Claude Code session:

```
/opus-coding:opus-coding <task description>
```

Or say: **`opus编码`** / **`多模型编码`** / **`parallel coding`**

---

## Available Plugins

| Plugin | Description |
|---|---|
| `opus-coding` | Four-phase multi-model coding: Plan → Parallel Codex → Haiku summary → Review |

---

## How opus-coding Works

```
Phase 1 — Plan    Analyze codebase → decompose into 3–8 subtasks
Phase 2 — Code    Parallel GPT-5.4 Codex threads (max 8)
Phase 3 — Read    Claude Haiku summarizes → writes files to disk
Phase 4 — Review  Opus + Codex relay, up to 3 rounds → REVIEW_VERDICT.md
```

See [plugins/opus-coding/README.md](./plugins/opus-coding/README.md) for full documentation.

---

## Repository Structure

```
opus-plan-codex-work/
├── .claude-plugin/
│   └── marketplace.json         ← Marketplace manifest
├── plugins/
│   └── opus-coding/             ← Plugin package
│       ├── .claude-plugin/
│       │   └── plugin.json      ← Plugin manifest
│       ├── skills/
│       │   └── opus-coding/
│       │       ├── SKILL.md     ← Full protocol
│       │       └── README.md
│       └── README.md
└── README.md                    ← This file
```

---

## Requirements

- Claude Code ≥ 1.0.33
- `mcp__codex__codex` MCP tool
- `mcp__codex__codex-reply` MCP tool

## License

MIT © [Kamisato520](https://github.com/Kamisato520)
