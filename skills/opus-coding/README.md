# opus-coding Skill

> **Opus Multi-Model Coding Protocol** — A four-phase collaborative coding skill for complex tasks, combining Claude Opus planning, parallel GPT-5.4 Codex implementation threads, Haiku-powered summarization, and joint Opus↔Codex review.

---

## Overview

`opus-coding` decomposes complex coding tasks into parallel subtasks, delegates implementation to multiple GPT-5.4 Codex threads, summarizes results via Claude Haiku sub-agents, and performs a joint review with relay mode between Claude Opus and GPT-5.4 Codex.

**Trigger phrases**: `"opus编码"`, `"多模型编码"`, `"parallel coding"`, or any request to decompose a complex coding task across multiple models.

---

## How It Works

```
Phase 1 — Plan       →  Decompose task into 3–8 independent subtasks
Phase 2 — Code       →  Parallel GPT-5.4 Codex threads implement each subtask
Phase 3 — Read       →  Claude Haiku summarizes all Codex outputs → CODE_SUMMARY.md
Phase 4 — Review     →  Opus + Codex relay review → REVIEW_VERDICT.md
```

### Phase 1 — Plan
- Analyzes the task and existing codebase (using `Glob` + `Read`)
- Decomposes into 3–8 subtasks with explicit dependency mapping
- Determines a `trial_run_command` for integration validation
- Writes `PIPELINE_STATE.json` to enable crash recovery

### Phase 2 — Code
- Calls `mcp__codex__codex` in parallel for all dependency-free subtasks
- Maximum **8** concurrent Codex threads
- All calls use `model_reasoning_effort: "xhigh"` for best quality
- Saves every `threadId` to state for recovery

### Phase 3 — Read (Haiku Summarization)
- Spawns `claude-haiku-4-5-20251001` sub-agents to extract key info from each Codex output
- Produces `CODE_SUMMARY.md` covering files, functions, errors/TODOs, key snippets, and integration notes
- Applies all generated code to disk using `Write` / `Edit`

### Phase 4 — Review (Opus + Codex Relay)
- Sends code + summary to GPT-5.4 Codex for senior engineer review
- Codex scores 1–10, outputs `APPROVED` or `NEEDS_FIXES`
- Opus applies critical fixes and iterates up to **3 rounds**
- Runs `trial_run_command` to validate integration
- Writes final `REVIEW_VERDICT.md`

---

## State & Recovery

The skill persists progress in `PIPELINE_STATE.json`. On startup it checks:

| Condition | Action |
|---|---|
| File missing | Fresh start |
| `status: "completed"` | Fresh start |
| `status: "in_progress"` + older than 48h | Fresh start (stale) |
| `status: "in_progress"` + within 48h | **Resume** from saved phase |

---

## Output Files

| File | Description |
|---|---|
| `PIPELINE_STATE.json` | Live state for crash recovery |
| `CODE_SUMMARY.md` | Haiku-distilled summary of all Codex outputs |
| `REVIEW_VERDICT.md` | Final review score, verdict, and trial run results |

---

## Constants

| Constant | Value |
|---|---|
| `MAX_CODEX_THREADS` | 8 |
| `MAX_REVIEW_ROUNDS` | 3 |
| `REASONING_EFFORT` | `"xhigh"` |

---

## Required Tools

- `Bash(*)`, `Read`, `Write`, `Edit`, `Grep`, `Glob`
- `Agent` (for Haiku sub-agents)
- `mcp__codex__codex` — Start a new GPT-5.4 Codex thread
- `mcp__codex__codex-reply` — Continue an existing Codex thread

---

## Key Rules

- 🔴 **Always use `model_reasoning_effort: "xhigh"`** — no exceptions for any Codex call
- 🔴 **Save all threadIds immediately** — required for context compaction recovery
- 🟡 **Parallel subtask calls** — subtasks without mutual dependencies MUST be called in one response
- 🟡 **Haiku for summarizing only** — all actual code writing goes through Codex or direct Write/Edit
- 🟢 **Honest error reporting** — never hide failures in summaries or verdicts
- 🟢 **Scope discipline** — do not fix out-of-scope bugs; note them in ERRORS/TODOS

---

## Example Usage

```
User: opus编码 — 为现有的 FastAPI 服务添加 JWT 认证、权限中间件和单元测试
```

The skill will:
1. Decompose into subtasks: JWT module, middleware, test suite
2. Run parallel Codex threads for each
3. Summarize outputs and write files to disk
4. Run review and trial tests
5. Output final verdict and file list

---

## File Structure

```
opus-coding/
├── SKILL.md      # Full skill protocol (machine-readable instructions)
└── README.md     # This file (human-readable documentation)
```

---

*Part of the [ARIS](https://github.com/Kamisato520/opus-plan-codex-work) skill library.*
