# opus-coding Plugin

> 🇨🇳 [简体中文文档](./README_ZH.md)

`opus-coding` implements the **Opus Multi-Model Coding Protocol** — a four-phase collaborative workflow that assigns each stage of the coding process to the model best suited for it.

---

## Overview

Complex coding tasks often fail in single-model sessions because context fills up long before the work is done. `opus-coding` solves this by:

1. Letting **Claude Opus** (strong at reasoning and planning) own the architecture
2. Delegating implementation to **up to 8 parallel GPT-5.4 Codex threads**
3. Using **Claude Haiku** (fast, cheap) to summarize outputs and reduce context load
4. Closing the loop with a **relay review** between Opus and Codex

The result: industrial-scale refactoring of 8,000+ lines across a single session, with automatic crash recovery.

---

## Four-Phase Protocol

### Phase 1 — Plan (Claude Opus)

- Reads the codebase with `Glob` + `Read`
- Decomposes the task into **3–8 subtasks** with an explicit dependency graph
- Selects a `trial_run_command` (e.g. `pytest`, `npm test`) for final validation
- Writes `PIPELINE_STATE.json` for crash recovery

**Decomposition principles:** prefer fewer, larger subtasks over many small ones. Subtasks with no mutual dependencies are launched in parallel.

### Phase 2 — Code (GPT-5.4 Codex, parallel)

- Calls `mcp__codex__codex` simultaneously for all dependency-free subtasks
- Every call uses `config: { "model_reasoning_effort": "xhigh" }` — no exceptions
- Maximum **8** concurrent threads
- Stores every `threadId` in `PIPELINE_STATE.json` immediately after the call

Once independent subtasks complete, dependent subtasks are unblocked and launched.

### Phase 3 — Read (Claude Haiku)

For each completed Codex thread, a `claude-haiku-4-5-20251001` sub-agent extracts:
- Files created / modified
- Public function and class signatures
- Errors, exceptions, TODO comments
- Critical code snippets (5–15 lines)
- Integration notes for the next phase

All extracted info is compiled into `CODE_SUMMARY.md`. Codex-generated code is then written to disk using `Write` / `Edit`.

### Phase 4 — Review (Opus + Codex Relay)

Since sub-agents cannot call MCP tools, Opus acts as a relay — it forwards code and summaries to GPT-5.4 Codex for review, applies fixes, and iterates.

The review loop:
1. Codex scores the implementation **1–10** and returns `APPROVED` or `NEEDS_FIXES`
2. Opus applies all critical fixes using `Write` / `Edit`
3. Opus calls `mcp__codex__codex-reply` with the updated code for re-assessment
4. Repeat up to **MAX_REVIEW_ROUNDS = 3**
5. Run `trial_run_command` and capture output
6. Write final `REVIEW_VERDICT.md`

---

## Crash Recovery

| Condition | Behavior |
|---|---|
| `PIPELINE_STATE.json` missing | Fresh start |
| `status: "completed"` | Fresh start |
| `status: "in_progress"` + age > 48h | Fresh start (stale) |
| `status: "in_progress"` + age ≤ 48h | **Resume** from saved phase |

---

## Output Files

| File | Description |
|---|---|
| `PIPELINE_STATE.json` | Live state — enables crash recovery |
| `CODE_SUMMARY.md` | Haiku-distilled summary of all Codex outputs |
| `REVIEW_VERDICT.md` | Final score, verdict, and trial run results |

---

## Constants

| Constant | Value | Notes |
|---|---|---|
| `MAX_CODEX_THREADS` | 8 | Max parallel implementation threads |
| `MAX_REVIEW_ROUNDS` | 3 | Max review/fix iterations |
| `REASONING_EFFORT` | `"xhigh"` | Applied to every Codex call |

---

## Required Tools

| Tool | Purpose |
|---|---|
| `Bash(*)`, `Read`, `Write`, `Edit`, `Grep`, `Glob` | Standard file and shell operations |
| `Agent` | Spawn Claude Haiku sub-agents for summarization |
| `mcp__codex__codex` | Start a new GPT-5.4 Codex thread |
| `mcp__codex__codex-reply` | Continue an existing Codex thread (relay review) |

---

## Key Rules

- 🔴 **Always use `model_reasoning_effort: "xhigh"`** — mandatory for every Codex call, no exceptions
- 🔴 **Save all threadIds immediately** — losing a threadId breaks relay review and crash recovery
- 🟡 **Parallel calls in one response** — subtasks with no mutual dependencies MUST be launched simultaneously
- 🟡 **Haiku for reading only** — Haiku sub-agents summarize; all code writing goes through Codex or direct Write/Edit
- 🟢 **Honest error reporting** — include all failures, errors, and TODOs in CODE_SUMMARY.md and REVIEW_VERDICT.md
- 🟢 **Scope discipline** — never silently fix out-of-scope bugs; log them in ERRORS/TODOS for the user

---

## Example Session

```
User: opus编码 — add JWT auth, permission middleware, and unit tests to the existing FastAPI service
```

The protocol will:
1. Read the existing service structure, identify auth-related files
2. Decompose: `[jwt_module]`, `[middleware]`, `[test_suite]` — middleware depends on jwt_module
3. Launch Codex threads for `jwt_module` and `test_suite` in parallel; once done, launch `middleware`
4. Haiku summarizes each output; code is written to disk
5. Codex reviews the combined implementation; Opus applies fixes (up to 3 rounds)
6. Run `pytest tests/` — capture output → `REVIEW_VERDICT.md`

---

## Trigger Phrases

Load the plugin, then use any of:

```
/opus-coding:opus-coding <task>
```

Or naturally in your message: `opus编码` · `多模型编码` · `parallel coding`

---

## File Structure

```
opus-coding/
├── SKILL.md          ← Full execution protocol (machine-readable)
├── README.md         ← This file (English)
└── README_ZH.md      ← 中文文档
```

---

*Part of the [opus-plan-codex-work](https://github.com/Kamisato520/opus-plan-codex-work) plugin marketplace.*
