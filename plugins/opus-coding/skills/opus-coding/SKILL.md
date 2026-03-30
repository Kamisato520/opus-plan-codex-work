---
name: opus-coding
description: "Opus multi-model coding protocol. Decomposes complex coding tasks into subtasks, delegates to parallel GPT-5.4 Codex threads, summarizes via Haiku sub-agents, and performs Opus↔GPT-5.4 joint review with relay mode. Use when user says \"opus编码\", \"多模型编码\", \"parallel coding\", or wants to decompose a complex coding task across multiple models."
argument-hint: [task-description]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, Agent, mcp__codex__codex, mcp__codex__codex-reply
---

# Opus Multi-Model Coding Protocol

Four-phase collaborative coding for: **$ARGUMENTS**

## Constants

- MAX_CODEX_THREADS = 8 — maximum parallel Codex threads in Phase 2
- MAX_REVIEW_ROUNDS = 3 — maximum follow-up rounds in Phase 4
- REASONING_EFFORT = "xhigh" — always use `config: {"model_reasoning_effort": "xhigh"}`
- STATE_FILE = `PIPELINE_STATE.json`
- SUMMARY_FILE = `CODE_SUMMARY.md`
- VERDICT_FILE = `REVIEW_VERDICT.md`

## State Persistence (Compact Recovery)

Long coding sessions may hit context limits and trigger automatic compaction. Persist state to `PIPELINE_STATE.json` at the end of each phase:

```json
{
  "task": "task description from $ARGUMENTS",
  "phase": "code",
  "status": "in_progress",
  "subtasks": [
    {
      "id": 1,
      "description": "Implement data loading module",
      "output_files": ["src/data_loader.py"],
      "dependencies": [],
      "threadId": "019cd392-...",
      "codex_status": "done"
    }
  ],
  "trial_run_command": "python -m pytest tests/ -x",
  "review_threadId": null,
  "review_round": 0,
  "timestamp": "2026-03-13T21:00:00"
}
```

**On every phase transition**, overwrite `PIPELINE_STATE.json` with the current phase and state.

**On startup**, check for `PIPELINE_STATE.json`:
- Does not exist → fresh start
- Exists AND `status` = `"completed"` → fresh start
- Exists AND `status` = `"in_progress"` AND `timestamp` older than 48 hours → fresh start (stale), delete file
- Exists AND `status` = `"in_progress"` AND `timestamp` within 48 hours → **resume** from saved `phase`

## Workflow

### Phase 1 — Plan (Decompose Task)

1. Read `PIPELINE_STATE.json` (resume logic above)
2. Analyze `$ARGUMENTS` thoroughly — understand scope, constraints, existing codebase context
3. Read relevant existing files with `Glob` + `Read` to understand the codebase
4. **Decompose** into 3–8 independent subtasks. For each subtask, specify:
   - `id`: integer
   - `description`: what needs to be implemented (2–4 sentences)
   - `output_files`: list of expected files/functions to create or modify
   - `dependencies`: list of subtask IDs that must complete first (empty = no dependencies)
   - `codex_prompt`: full coding prompt to send to Codex (see Phase 2 template below)
5. Determine `trial_run_command`: the command that validates everything integrates correctly
6. Write `PIPELINE_STATE.json` with `phase: "code"`, all subtasks, and `codex_status: "pending"` for each

**Decomposition principles:**
- Each subtask should be independently implementable by a single Codex thread
- Prefer fewer, larger subtasks over many tiny ones (aim for 3–5)
- Subtasks with dependencies must declare them explicitly
- If the task is small enough for one subtask, use one — don't over-decompose

### Phase 2 — Code (Parallel Codex Threads)

For each group of subtasks with no pending dependencies, call `mcp__codex__codex` **in the same response** (parallel):

```
mcp__codex__codex:
  config: {"model_reasoning_effort": "xhigh"}
  prompt: |
    [SUBTASK N of M — opus-coding protocol]

    Task context: [overall task description from $ARGUMENTS]
    Subtask: [subtask.description]

    Expected outputs:
    [subtask.output_files listed]

    Codebase context:
    [paste relevant existing code, imports, function signatures, type definitions]

    Requirements:
    1. Implement the subtask fully and correctly
    2. Follow the existing code style and conventions
    3. Include brief inline comments only where logic is non-obvious
    4. If you find a bug in adjacent code, note it but do not fix it (out of scope)
    5. If a requirement is ambiguous, implement the most reasonable interpretation and note it

    Output the complete file contents for each output file.
```

- Save each returned `threadId` into `PIPELINE_STATE.json` (`codex_status: "done"`)
- After independent subtasks complete, check if dependent subtasks can now be unblocked
- Launch dependent subtask Codex calls once their prerequisites are `"done"`
- Do NOT exceed MAX_CODEX_THREADS parallel threads at once

**Update `PIPELINE_STATE.json`** after each thread completes, setting `codex_status: "done"` and saving the `threadId`.

### Phase 3 — Read (Haiku Sub-Agent Summarization)

For each completed Codex thread, call `Agent` with `model="claude-haiku-4-5-20251001"` to extract key information:

```
Agent(
  model="claude-haiku-4-5-20251001",
  prompt="""
    Please summarize the following Codex output for subtask N:

    [paste full Codex response]

    Extract and return:
    1. FILES CREATED/MODIFIED: list each file path and a 1-sentence description
    2. FUNCTIONS/CLASSES: list each public function/class signature
    3. ERRORS OR TODOS: any error messages, exceptions, or TODO comments left in code
    4. KEY CODE SNIPPETS: the most important 5–15 lines (critical logic only)
    5. INTEGRATION NOTES: anything the next phase needs to know (imports, API changes, etc.)
  """
)
```

Collect all Haiku summaries and write `CODE_SUMMARY.md`:

```markdown
# Code Summary — [task from $ARGUMENTS]

Generated: [timestamp]
Trial run command: [command]

## Subtask 1: [description]

### Files
- `path/to/file.py` — [1-sentence description]

### Key Functions
- `function_name(args) -> return_type` — [purpose]

### Errors / TODOs
- [any issues noted]

### Key Snippet
```python
[5–15 lines of critical logic]
```

### Integration Notes
- [anything downstream needs to know]

---

## Subtask 2: ...
```

**Apply the code**: Use `Write` / `Edit` to write the output files from each Codex thread to disk, guided by the Haiku summaries. If a file already exists, use `Edit` to apply only the changed sections.

### Phase 4 — Review (Opus + Codex Relay)

Relay mode: since sub-agents cannot call MCP tools, Opus communicates with the GPT-5.4 reviewer through the main session.

**Step 1**: Compose review request

Read `CODE_SUMMARY.md` and the actual written files. Draft the review prompt:

```
mcp__codex__codex:
  config: {"model_reasoning_effort": "xhigh"}
  prompt: |
    [opus-coding review — Round 1/MAX_REVIEW_ROUNDS]

    Task: [task description from $ARGUMENTS]
    Trial run command: [command]

    Code summary:
    [paste CODE_SUMMARY.md contents]

    Key implementation files:
    [paste relevant file contents]

    Please act as a senior software engineer reviewer.

    1. Score this implementation 1–10 for correctness, completeness, and code quality
    2. List critical issues (bugs, missing edge cases, broken interfaces)
    3. List non-critical issues (style, performance, clarity)
    4. For each critical issue, provide the EXACT fix (code snippet or clear instruction)
    5. State clearly: APPROVED (no critical issues) or NEEDS_FIXES (with list)
```

Save the `threadId` as `review_threadId` in `PIPELINE_STATE.json`.

**Step 2**: Parse verdict

Extract from the Codex response:
- **Score** (1–10)
- **Verdict**: `APPROVED` or `NEEDS_FIXES`
- **Critical issues** (must fix before approval)
- **Non-critical issues** (optional improvements)

**STOP CONDITION**: If verdict is `APPROVED` → skip to Termination.

**Step 3**: Apply fixes (if NEEDS_FIXES)

Apply each critical fix using `Write` / `Edit`. Then call `mcp__codex__codex-reply` for follow-up:

```
mcp__codex__codex-reply:
  threadId: [review_threadId]
  config: {"model_reasoning_effort": "xhigh"}
  prompt: |
    [Round N update — fixes applied]

    Applied fixes:
    1. [Fix 1]: [what was changed]
    2. [Fix 2]: [what was changed]

    Updated files:
    [paste relevant updated file contents]

    Please re-score and re-assess. Are the critical issues resolved?
    Same format: Score, APPROVED/NEEDS_FIXES, remaining issues.
```

Increment `review_round` in `PIPELINE_STATE.json`. Repeat up to MAX_REVIEW_ROUNDS times.

**Step 4**: Run trial command

```bash
[trial_run_command]
```

Capture stdout/stderr. If the command fails, apply fixes and re-run (this counts toward MAX_REVIEW_ROUNDS).

**Step 5**: Write verdict

Write `REVIEW_VERDICT.md`:

```markdown
# Review Verdict — [task from $ARGUMENTS]

Date: [timestamp]
Review rounds: N/MAX_REVIEW_ROUNDS
Trial run: [PASSED / FAILED]

## Final Score: X/10

## Verdict: APPROVED / NEEDS_FIXES

## Issues Addressed
1. [issue] → [fix applied]
2. [issue] → [fix applied]

## Remaining Issues (if any)
- [issues not addressed — flag for manual follow-up]

## Trial Run Output
```
[stdout/stderr from trial_run_command]
```

## Files Changed
- [list of created/modified files]
```

### Termination

1. Update `PIPELINE_STATE.json` with `"status": "completed"`
2. Print final summary to the user:

```
opus-coding complete.

Task: [task description]
Subtasks: N completed
Review: APPROVED / NEEDS_FIXES (X/10)
Trial run: PASSED / FAILED

Files created/modified:
- [file list from CODE_SUMMARY.md]

See CODE_SUMMARY.md and REVIEW_VERDICT.md for full details.
```

## Key Rules

- **Large file handling**: If the Write tool fails due to file size, immediately retry using Bash (`cat << 'EOF' > file`) to write in chunks. Do NOT ask the user for permission — just do it silently.

- **Always use REASONING_EFFORT**: Every `mcp__codex__codex` call MUST include `config: {"model_reasoning_effort": "xhigh"}`. No exceptions.

- **Save all threadIds**: Store every Codex threadId in `PIPELINE_STATE.json` immediately after the call. If context compacts, threadIds are how we continue.

- **Parallel calls in the same response**: Subtasks with no mutual dependencies MUST be called in a single response — do not call them sequentially.

- **Haiku for reading, not coding**: Agent with haiku is for summarizing Codex output only. All actual code writing goes through Codex or direct Write/Edit.

- **Honest error reporting**: Include all errors, TODOs, and failed trial runs in CODE_SUMMARY.md and REVIEW_VERDICT.md. Do not hide failures.

- **Scope discipline**: Do not fix bugs outside the requested subtask scope. Note them in ERRORS/TODOS for the user.

- **Apply code from Codex faithfully**: Do not silently rewrite what Codex returned. Apply it as-is; raise disagreements in the Phase 4 review prompt.
