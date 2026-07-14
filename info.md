# OpenCode System Prompt

## How the system prompt is assembled

Each provider turn, opencode builds the system message from several layers. Order matters — earlier layers are the "harness," later layers append context.

1. **Base harness** — a model-specific text file selected by model ID substring in `packages/opencode/src/session/system.ts`. The files live in `packages/opencode/src/session/prompt/` and are bundled into the binary at build time (e.g. `anthropic.txt` for Claude, `beast.txt`/`gpt.txt` for GPT, `gemini.txt`, `default.txt` fallback).

2. **Environment block** — model ID, working directory, workspace root, git status, platform, date, project references (`<env>`).

3. **Instructions** — auto-discovered `AGENTS.md` / `CLAUDE.md` / `CONTEXT.md` (global + walked up the project tree) plus files/URLs from the `instructions` config array. See `packages/opencode/src/session/instruction.ts`.

4. **MCP instructions** and **skills** blocks.

5. Optional structured-output prompt and a per-message `user.system` string.

Final assembly lives in `packages/opencode/src/session/llm/request.ts` and `packages/opencode/src/session/prompt.ts`.

## Is it configurable?

The bundled base `.txt` harness is not a config field, but the effective prompt can be shaped several ways:

- **Agent `prompt` override (full replacement)** — an agent's markdown body becomes its `prompt` (`packages/core/src/config/plugin/agent.ts`); if present, it **replaces** the built-in model prompt entirely (`packages/opencode/src/session/llm/request.ts`).
- **`instructions` array** in `opencode.json(c)` — appends files, globs, or remote URLs.
- **`AGENTS.md`-family files** — auto-discovered project/global instructions.
- **Plugin hook** `experimental.chat.system.transform` — programmatically mutates the assembled system array.
- **Flags** — `OPENCODE_DISABLE_PROJECT_CONFIG` and `disableClaudeCodePrompt` trim instruction sources.

## Base harness: anthropic.txt vs default.txt

The two prompts reflect different philosophies rather than just model tweaks.

- **anthropic.txt** is **agentic and planning-oriented**: assertive persona ("best coding agent on the planet"), a dedicated section on professional objectivity (disagree when needed, investigate uncertainty), and a full `TodoWrite` task-planning section with examples. It also steers the model toward proactive subagent dispatch.

- **default.txt** is **disciplined and tool-oriented**: modest persona, strict brevity rules (`<4 lines`, no preamble), sections on following project conventions, no code comments, non-surprising behavior, and mandatory lint/typecheck + "never commit unless asked."

Notable asymmetries:

| Section | anthropic.txt | default.txt |
|---|---|---|
| TodoWrite task planning | yes | no |
| Professional objectivity | yes | no |
| Proactiveness / don't surprise | no | yes |
| Code style / conventions | no | yes |
| Lint/typecheck + commit guard | no | yes |
| Specialized subagent dispatch | yes | no |

Net: `anthropic.txt` optimizes for autonomous, multi-step planning (Claude's strengths); `default.txt` optimizes for concise, convention-respecting tool use across any model.

## Running from source

Requires Bun 1.3+. `bun install` is only needed once (or after dependency changes), not every run.

```bash
bun install     # once, or after dependency changes
bun dev         # TUI (default project: packages/opencode)
bun dev .       # TUI with repo root as the project directory
bun dev /path   # TUI with any directory as the project
bun dev serve   # headless API server on port 4096
```

Other dev entrypoints:
- Web app: `bun run --cwd packages/app dev` (needs server running)
- Desktop: `bun run --cwd packages/desktop dev`

After changing the API/SDK, regenerate clients with `bun run generate` from `packages/client`.

## Compiling a fresh binary

Build a standalone executable (no Bun needed to run it afterward):

```bash
./packages/opencode/script/build.ts --single
```

Then run the compiled TUI directly:

```bash
./packages/opencode/dist/opencode-<platform>/bin/opencode
```

Replace `<platform>` with your platform (e.g. `linux-x64`, `darwin-arm64`). The binary supports the same subcommands (`serve`, `web`, `<directory>`, etc.).

## Nested Sub-Agents Feature Status

Issue #32166 proposes nested sub-agent spawning (up to 5 levels) + multi-agent workflow orchestration. Current status:

- **Not merged** — two open PRs exist but neither is in `dev` or `main`
  - PR #32301: Nested sub-agent spawning (has **merge conflicts**, no reviews)
  - PR #33144: Agent teams + delegation (still **draft**, no reviews)
- **Functional on fork** — author reports "tested, used daily" on their branch
- **Addresses known issues** — would fix #23091 (depth-3 failure) and #13715 (permission hangs)

**Usable right now?** No, unless you use the author's fork directly. The PRs need conflict resolution and code review before merging.

## Sub-Agent Hanging Issues (GitHub Issues)

Multiple issues reported about sub-agents hanging indefinitely. Root causes and key issues:

### Critical Open Issues

| Issue | Title | Status |
|-------|-------|--------|
| **#13715** | Permission asks from nested subagent sessions silently hang | Open (fixed locally by #35823 + #36046) |
| **#35073** | fix: subagent permission asks hang indefinitely (sync subagents treated as interactive) | Open (fixed locally by #35823) |
| **#36762** | Headless `opencode run`: any permission resolving to "ask" hangs forever | Open (fixed locally by #35823) |
| **#33028** | Subagents hang indefinitely after quick bash tool call | Open (fixed locally by #36755 timeout) |
| **#32388** | ACP subagents are invisible and can hang forever on permission prompts | Open (fixed locally by #35823 + #36046) |
| **#11865** | Tasks/Subagents with Codex/OpenAI get stuck with no timeout/retry | Open (fixed locally by #36755 timeout) |
| **#13841** | Explore subagent hangs indefinitely with Claude Opus 4.6 | Open (fixed locally by #36755 timeout) |
| **#23296** | Build stuck for 6h after delegating to explore subagent | Open (fixed locally by #36755 timeout) |
| **#25187** | Main & Sub-agents Randomly Freeze Indefinitely | Open (fixed locally by #36755 timeout) |
| **#35207** | Session hangs after MCP tool-call — no timeout recovery | Open (fixed locally by #36755 timeout) |

### Root Causes

1. **Permission routing broken** - TUI only collects direct children, missing grandchild events (#13715, #7654) — **FIXED locally by #36046**
2. **Subagents treated as interactive** - In headless mode, subagents wait for human input that never comes (#35073) — **FIXED locally by #35823**
3. **No timeout mechanisms** - Many issues mention lack of timeout/retry for hanging subagents — **FIXED locally by #36755**
4. **Auto-approve not inherited** - `--auto` flag doesn't propagate to subagent sessions — **FIXED locally by #35823**
5. **Runner queue discarding notifications** - `Runner.ensureRunning` discards completion notifications when parent is busy (#35066) — **NOT FIXED** (PR #36375)
6. **No interrupt capability** - No way to cancel/steer hanging subagents mid-run (#21458, #23534, #28738) — **NOT FIXED** (PR #32425)

### Related Closed Issues
- #30635 - Permission prompts from nested subagents never shown (closed)
- #23415 - Workflow stalls if sub-agent aborted (closed)
- #25187 - Sub-agents hang on context overflow (closed)

### Proposed Fix

PR #32167 (from proposal #32166) attempts to address nested sub-agent permission routing by carrying `originSessionID`/`originAgent`/`originDepth` metadata to root sessions.

### Active Fix PRs

#### DONE: Fix headless mode permission hangs ✅

PR #35823 (`fix/subagent-permission-hang`) — **fix(cli): answer subagent permission asks in headless run (#35073)**
- **Status:** ✅ MERGED LOCALLY — cherry-picked commit `e39304e8` onto `my` (rebased on latest `origin/dev`); typecheck passes, all 6 tests pass
- **Fix:** Walks the `parentID` chain to answer permission asks from descendant sessions in headless mode
- **Addresses:** Subagents treated as interactive in headless mode; auto-approve not inherited

#### DONE: Fix TUI permission routing for nested subagents ✅

PR #36046 (`fix_subagent_perm`) — **fix(tui): show permission prompts from nested subagent chains**
- **Status:** ✅ MERGED LOCALLY — merged into `my` branch (rebased on latest `origin/dev`); typecheck passes
- **Fix:** Collects full subtree instead of direct children, fixing TUI deadlock on grandchild subagent permissions
- **Addresses:** Permission routing broken (TUI only collects direct children, missing grandchild events)

#### DONE: Add configurable timeout to Task tool ✅

PR #36755 (`task-tool-timeout`) — **fix(opencode): add configurable timeout to Task tool**
- **Status:** ✅ MERGED LOCALLY — merged into `my` branch (rebased on latest `origin/dev`); typecheck passes
- **Fix:** Adds configurable timeout (5min default) via `timeout` parameter so subagents don't hang forever
- **Addresses:** No timeout mechanisms (#11865, #33028, #13841, #23296, #25187, #35207)

#### DONE: Inline subtask tree rendering + permission prompt UX ✅

PR #24638 (`fix/nested-subagent-permissions`) — **fix(tui): propagate permissions from nested subagents and show full subtask tree**
- **Status:** ✅ PORTED LOCALLY — stale PR (targeted old `packages/opencode` layout, conflicted with already-merged #36046). Instead of merging, ported only the two genuinely new pieces onto `my`; typecheck passes
- **Ported:**
  - `taskSubtree` / `sessionStats` / `formatStats` / `childStats` helpers for inline recursive subtask rendering with per-session stats (duration, tools, tokens, cost) and cycle protection
  - Permission prompt UX: `subagentLabel` shows requesting session's description in header; generic tool fallback now displays tool params instead of just tool name
- **Skipped (superseded by #36046):** `descendants` memo — `collectSubtree` already provides descendant traversal in the merged `my` branch

#### TODO: Fix runner queue discarding subagent notifications ✅ MERGEABLE

- PR #36375 (`fix/runner-queue-background-notification`) — **fix(runner): queue work when already running instead of discarding**
  - Status: OPEN (addresses #35066)
  - **Mergeability: CLEAN** — 0 conflicts, merges to `my` without issues
  - Root cause: `Runner.ensureRunning` discards `ops.prompt()` calls when parent is busy, so background subagent completion notifications are lost and parent hangs
  - Fix: Adds `RunningThenRun` state to runner state machine to queue pending work instead of discarding it

#### TODO: Add subagent interrupt capability ⚠️ CONFLICTS (resolvable)

- PR #32425 (`subagent-interrupt`) — **feat(opencode): interrupt a running subagent — steer / cancel / abort**
  - Status: OPEN (addresses #21458, #23534, #28738)
  - **Mergeability: 3 conflicts** in `runtime-flags.ts`, `tool/task.ts`, `tool/task.test.ts`, `session/index.tsx` — all resolvable
  - Adds `task_steer`/`task_cancel`/`task_abort` tools + TUI esc menu to interrupt hanging subagents mid-run
  - Gated behind `OPENCODE_EXPERIMENTAL_SUBAGENT_INTERRUPT`

#### TODO: Add core permission improvements ❌ MAJOR CONFLICTS (needs rebase)

- PR #36403 (`subagent-permissions`) — **fix(core): restore permission-aware subagent guidance**
  - Status: OPEN (245 lines)
  - **Mergeability: 1858 conflicts** — PR is based on an old `dev`, needs full rebase
  - Restores filtered subagent guidance so denied targets aren't offered to the model
