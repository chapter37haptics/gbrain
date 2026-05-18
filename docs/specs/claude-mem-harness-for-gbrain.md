# Spec: Claude-Mem Harness for GBrain

## Goal

Use claude-mem's hook pipeline code AS-IS to give gbrain reliable event
capture, context injection, and session-end consolidation. The hook layer
guarantees the brain-agent loop fires under all execution modes (/goal,
conversational, skill) without modifying gbrain's skill files or MCP server.

## Constraints

1. **claude-mem code can be deleted but not modified.** If a file needs
   changes, create a new file instead.
2. **New files are highly discouraged.** Only create when there is no
   alternative. Prefer reusing existing files by composition.
3. **gbrain's skill files and MCP server are untouched.** The hook layer
   is additive infrastructure, not a replacement for the brain-agent loop.
4. **gbrain's ethos is preserved.** Hooks handle deterministic capture
   (thin harness). Skills handle semantic interpretation (fat skills).

## Why This Works

claude-mem's hook pipeline has two clean layers:

1. **Generic layer** (orchestrator, adapters, types, constants): handles
   stdin parsing, platform normalization, exit codes, and timeout. Zero
   storage coupling. Reusable AS-IS.
2. **Handler layer** (context.ts, observation.ts, summarize.ts): calls
   an HTTP API to store/retrieve observations. The API endpoints are
   the only coupling point.

We reuse layer 1 entirely. For layer 2, we replace the HTTP targets:
instead of calling claude-mem's worker (`/api/sessions/observations`),
the handler appends to a JSONL event log and shells out to gbrain CLI.

## Architecture

```
Claude Code hook event (PostToolUse, SessionStart, Stop)
  |
  v
bun-runner.js (AS-IS from claude-mem)
  |
  v
hook-command.ts (AS-IS from claude-mem, generic orchestrator)
  |
  v
claude-code adapter (AS-IS from claude-mem, normalizes input)
  |
  v
gbrain handler (NEW - replaces claude-mem handlers)
  |
  +--> PostToolUse: append JSONL to ~/.gbrain/events/<session>.jsonl
  +--> SessionStart: gbrain search -> format as additionalContext
  +--> Stop: gbrain consolidate (async, skill-driven)
```

## Files from claude-mem: Reuse AS-IS

These files are copied from claude-mem's source tree without modification.
They form the generic hook pipeline.

| File | Lines | Role |
|------|-------|------|
| `plugin/scripts/bun-runner.js` | ~180 | Node-to-Bun bridge, spawns hook process |
| `src/cli/hook-command.ts` | 116 | Stdin -> adapter -> handler -> stdout |
| `src/cli/types.ts` | 46 | NormalizedHookInput, HookResult, interfaces |
| `src/cli/adapters/index.ts` | 22 | Adapter registry |
| `src/cli/adapters/claude-code.ts` | 42 | Claude Code input normalizer |
| `src/cli/adapters/errors.ts` | 11 | AdapterRejectedInput error class |
| `src/shared/hook-constants.ts` | 26 | Timeouts, exit codes |

**Total reused: 7 files, ~443 lines, zero modifications.**

### Import closure verification (Codex finding #3)

Before copying, verify the import graph of these 7 files resolves
completely within the set. Known risk: `hook-command.ts` imports a
handler index and a stdin reader. The handler index is replaced by the
new file. The stdin reader must be verified: if it exists as a separate
file, add it to the reuse list. If it is inlined in hook-command.ts,
no action needed.

Run: `grep -n "from\|import" src/cli/hook-command.ts` in the claude-mem
repo and trace every local import. Any file outside the 7-file set that
is imported must be either (a) added to the reuse list or (b) inlined
into the new handler file.

## Files from claude-mem: Delete (not needed)

Everything else. The worker service, SQLite storage, ChromaDB sync, SDK
agent, MCP server, viewer UI, mode system, prompt templates, and all
platform adapters except claude-code. gbrain has its own equivalents for
all of these.

## New Files

### File count (corrected per Codex finding #2)

| File | Type | Why it cannot be avoided |
|------|------|--------------------------|
| `src/cli/handlers/gbrain-handler.ts` | Handler | Replaces claude-mem's handler layer with gbrain targets |
| `~/.gbrain/hooks/gbrain-hook.sh` | Shell wrapper | Entry point called by Claude Code hook registration. Pipes stdin to bun-runner.js, writes stdout. Cannot be inlined into an existing file because Claude Code hooks require a command path. |

**Total new files: 2** (1 TypeScript handler + 1 shell wrapper for all 3 events).

The 3 hook events (PostToolUse, SessionStart, Stop) are routed by the
single shell wrapper using an argument. No need for 3 separate scripts.

The `gbrain consolidate` CLI command is added to gbrain's existing
`src/commands/` directory. This is a new command in an existing command
registry file, not a new standalone file (same pattern as `gbrain doctor`,
`gbrain sync`, etc).

### `gbrain-handler.ts` (the handler)

Implements the `EventHandler` interface from claude-mem's `types.ts`.
Handles three events:

#### PostToolUse

Deterministic. No LLM call. Appends one JSONL line to the session event log.

```
Input: NormalizedHookInput (toolName, toolInput, toolResponse, sessionId, cwd)
Output: HookResult { continue: true }
Side effect: append to ~/.gbrain/events/<sessionId>.jsonl
```

Fields: timestamp, tool_name, tool_input (truncated to 500 chars),
tool_exit_code, cwd, session_id. No judgment. No filtering.

Concurrency (Codex finding #9): JSONL appends are atomic at the OS level
for writes under PIPE_BUF (4KB on Linux). Each event line is well under
this. For extra safety, use `O_APPEND` mode which guarantees atomic
appends regardless of size.

Cost: ~1ms per tool call (file append). Exits 0 always.

#### SessionStart

Deterministic. No LLM call. Queries gbrain for prior context.

```
Input: NormalizedHookInput (sessionId, cwd)
Output: HookResult { additionalContext: "<formatted brain context>" }
Side effect: none
```

Output key is `additionalContext` (Codex finding #4: Claude Code hook
protocol uses `additionalContext` for context injection, not
`systemMessage`).

Implementation: shell out to `gbrain search` with the project name
derived from cwd. Format results as markdown. Return in the HookResult.

If gbrain is unavailable (CLI missing, DB not initialized), return
empty `additionalContext`. Exits 0 always.

#### Stop

**Async fire-and-forget.** The handler spawns `gbrain consolidate` as
a detached background process and immediately returns `HookResult
{ continue: true }`. This way the hook exits in <100ms (Codex finding
#8: LLM consolidation under a 120s timeout is unreliable).

The consolidation runs outside the hook timeout. If it fails, the event
log persists at `~/.gbrain/events/<sessionId>.jsonl` for manual or
next-session processing.

```
Input: NormalizedHookInput (sessionId)
Output: HookResult { continue: true }
Side effect: spawns background consolidation process
```

### `gbrain-hook.sh` (the shell wrapper)

```bash
#!/bin/sh
exec bun ~/.gbrain/hooks/bun-runner.js hook claude-code "$1"
```

One wrapper, one argument (`post-tool-use`, `session-start`, or `stop`).
Claude Code hook registration points all 3 events at this script with
different args.

## `gbrain consolidate` CLI command

Added to gbrain's existing command registry (not a new file). This is the
bridge between the hook layer (Plane A) and the skill layer (Plane B).

```
gbrain consolidate --session <id> [--events-dir ~/.gbrain/events]
```

Implementation:
1. Read `~/.gbrain/events/<sessionId>.jsonl`
2. Format events as a structured tool-call timeline
3. Feed the timeline to the signal-detector skill (via the existing
   `gbrain serve` MCP interface or direct function call) which decides
   what entities to extract
4. Call `put_page` to create/update a goal page with the skill's output
5. Optionally call `add_link` for cross-references
6. On success: delete the processed event log
7. On failure: leave the event log intact, exit non-zero

Cleanup responsibility is solely in `consolidate` (Codex finding #7:
the Stop handler does NOT delete the event log; it only spawns
consolidate which owns the lifecycle).

### How skills are invoked (Codex finding #5)

The consolidate command is deterministic orchestration. The skill
invocation happens through gbrain's existing MCP tool interface: the
command calls `put_page` with the event timeline as content. The
signal-detector skill file (loaded into CLAUDE.md) determines page
structure when the agent reads the page later. For richer consolidation,
the command can spawn a short Claude Code session (`claude -p` with
the event log as input and signal-detector instructions as system
prompt) to produce the structured goal page. This is the same pattern
as the test-gbrain-agent.sh round-trip test in the devcontainer.

### MCP usage clarification (Codex finding #6)

Hooks do NOT use MCP. The PostToolUse handler writes JSONL (file I/O).
The SessionStart handler shells out to `gbrain search` (CLI). The Stop
handler spawns `gbrain consolidate` (CLI).

The consolidate command MAY use gbrain's MCP tools internally (via
`gbrain serve` stdio interface or direct PGLite calls). This is gbrain
CLI-to-CLI, not hook-to-MCP. The hooks never touch MCP.

## Hook Registration

In the devcontainer entrypoint or Claude Code settings:

```json
{
  "hooks": {
    "PostToolUse": [{
      "command": "~/.gbrain/hooks/gbrain-hook.sh post-tool-use",
      "timeout": 5000
    }],
    "SessionStart": [{
      "command": "~/.gbrain/hooks/gbrain-hook.sh session-start",
      "timeout": 10000
    }],
    "Stop": [{
      "command": "~/.gbrain/hooks/gbrain-hook.sh stop",
      "timeout": 120000
    }]
  }
}
```

## Existing gbrain CLI commands used (Codex finding #11)

| Command | Exists? | Used by |
|---------|---------|---------|
| `gbrain search` | Yes (MCP tool + CLI) | SessionStart handler |
| `gbrain serve` | Yes (MCP server) | consolidate (internal) |
| `put_page` | Yes (MCP tool) | consolidate |
| `add_link` | Yes (MCP tool) | consolidate |
| `gbrain consolidate` | **New** (added to existing command registry) | Stop handler |

## What This Preserves

| gbrain principle | How preserved |
|-----------------|---------------|
| Thin harness | Hooks are 7 reused files + 2 new files |
| Fat skills | Signal-detector and brain-ops still do all interpretation |
| Harness-agnostic | Hook layer is an optional Claude Code adapter; skills work without it |
| Markdown is code | Skill files unchanged; they drive consolidation |
| Self-rewriting skills | Skills can evolve based on consolidation patterns |
| FS-canonical | Pages go through put_page -> export -> markdown repo |

## What This Fixes

| Experiment finding | How fixed |
|-------------------|-----------|
| 0/85 gbrain calls under /goal | PostToolUse captures every tool call automatically |
| No context injection | SessionStart injects brain context before prompt |
| No session-end summary | Stop triggers async skill-driven consolidation |
| Deferred MCP tool friction | Hooks use CLI + JSONL, not MCP tools |
| Skill/goal pressure suppression | Hooks run outside agent control |

## Scope

- **Files copied from claude-mem:** 7 (unmodified) + import closure additions TBD
- **New files created:** 2 (`gbrain-handler.ts` + `gbrain-hook.sh`)
- **New gbrain CLI commands:** 1 (`consolidate`, added to existing command registry)
- **gbrain skill files changed:** 0
- **gbrain MCP server changed:** 0

## Open Questions

1. **Import closure:** Does the 7-file set resolve cleanly? Must verify
   before implementation.
2. **Consolidation quality:** Does `claude -p` with the event log produce
   good goal pages, or does the signal-detector skill need adaptation?
3. **Event log size:** A 30-minute /goal session with 85 tool calls
   produces ~85 JSONL lines (~50KB). Is this within consolidation's
   context budget?
