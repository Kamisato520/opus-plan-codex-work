# opus-plan-codex-work — Claude Code Plugin Marketplace

> 🇨🇳 [简体中文文档](./README_ZH.md)

A Claude Code **plugin marketplace** hosting the **opus-coding** plugin: a four-phase multi-model coding protocol using Claude Opus for planning, parallel GPT-5.4 Codex threads for implementation, Claude Haiku for summarization, and a joint Opus↔Codex relay review.

---

## Quick Install

**Step 1 — Register this marketplace in Claude Code:**

```
/plugin marketplace add https://github.com/Kamisato520/opus-plan-codex-work
```

**Step 2 — Install the plugin:**

```
/plugin install opus-coding
```

---

## Usage

After installation, start the protocol in any Claude Code session:

```
/opus-coding:opus-coding <your task description>
```

You can also trigger it naturally by saying: **`opus编码`** · **`多模型编码`** · **`parallel coding`**

---

## Why opus-coding?

Traditional single-model interactions tend to hallucinate or lose context on changes exceeding ~2,000 lines. `opus-coding` breaks through this limit by decomposing tasks into parallel subtasks, enabling industrial-scale refactoring of **8,000+ lines** across a single session.

---

## Four-Phase Protocol

| Phase | Model | What Happens |
|---|---|---|
| **1 — Plan** | Claude Opus | Reads codebase, decomposes task into 3–8 subtasks with dependency graph |
| **2 — Code** | GPT-5.4 Codex (×8 max) | Parallel implementation threads, `reasoning_effort: xhigh` |
| **3 — Read** | Claude Haiku | Summarizes all outputs → writes files to disk → `CODE_SUMMARY.md` |
| **4 — Review** | Opus + Codex relay | Senior engineer review loop, up to 3 rounds → `REVIEW_VERDICT.md` |

See [plugins/opus-coding/README.md](./plugins/opus-coding/README.md) for the full technical reference.

---

## Crash Recovery

All state is persisted to `PIPELINE_STATE.json` after every phase. If the session is interrupted or context is compacted, the protocol resumes automatically within 48 hours — no manual restart needed.

---

## Repository Structure

```
opus-plan-codex-work/
├── .claude-plugin/
│   └── marketplace.json              ← Marketplace manifest
├── plugins/
│   └── opus-coding/                  ← Plugin package
│       ├── .claude-plugin/
│       │   └── plugin.json           ← Plugin manifest
│       ├── skills/
│       │   └── opus-coding/
│       │       ├── SKILL.md          ← Full execution protocol (machine-readable)
│       │       └── README.md         ← Skill reference
│       ├── README.md                 ← Plugin full documentation (English)
│       └── README_ZH.md              ← Plugin documentation (中文)
├── README.md                         ← This file (English)
└── README_ZH.md                      ← 中文说明
```

---

## Requirements

- Claude Code ≥ 1.0.33
- `mcp__codex__codex` MCP tool (GPT-5.4 Codex access)
- `mcp__codex__codex-reply` MCP tool (for review relay)

## License

MIT © [Kamisato520](https://github.com/Kamisato520)
