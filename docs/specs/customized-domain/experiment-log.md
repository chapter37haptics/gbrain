# Experiment Log: GBrain Customized Domain

Experiments testing whether the customized-domain gbrain skill files
actually work in practice. Each experiment is reproducible from the
documented commit hash and devcontainer configuration.

---

## Experiment 1: Does gbrain fire during a `/goal` session?

**Date:** 2026-05-15
**Branch:** `test/gbrain-developer-persona` (practicespace-2)
**gbrain commit:** `86abd45ce63484d4cc62c14e7f8f32095c3f76a4`
**Dockerfile:** `practicespace-2/.devcontainer/Dockerfile` line 109, `ARG GBRAIN_COMMIT=86abd45...`
**Claude Code version:** 2.1.140
**Model:** claude-opus-4-6[1m]
**Session ID:** `94bb5cda-e085-42c0-8217-df3e093bdd3b`
**Session log:** `/home/dev/.claude/projects/-workspaces-goal-email-thread-separator/94bb5cda-e085-42c0-8217-df3e093bdd3b.jsonl`

### What we tried

Ran `/goal` on a fresh project (`/workspaces/goal-email-thread-separator/`)
with the customized-domain gbrain skill files loaded. The goal spec asked the
agent to build an email chain splitter: parse `.eml` files, split chains into
individual emails, render Outlook-style screenshots, produce a manifest.

The purpose was to test whether gbrain's signal-detector and brain-ops loop
would capture the development session as knowledge (goals, decisions, debug
trails) that could be retrieved later by a different agent session.

### To reproduce

1. Build the devcontainer with `GBRAIN_COMMIT=86abd45ce63484d4cc62c14e7f8f32095c3f76a4`
2. Start Claude Code in `/workspaces/goal-email-thread-separator/`
3. Run `/goal goal-email-thread-separator.md`
4. Let the agent complete autonomously
5. Check the session log for `mcp__gbrain__*` tool calls

### What happened

The agent successfully completed the goal: built a 758-line Python script,
split a 3-email chain, rendered Outlook-style screenshots, produced a manifest.
It ran 85 tool calls (45 Bash, 22 Read, 15 Edit, 3 Write) across 288 session
log lines. The user spoke only 3 times during the autonomous session.

Session tool histogram:

```
 85  user/tool_result
 45  assistant/tool_use/Bash
 44  assistant/text
 22  assistant/tool_use/Read
 19  assistant/thinking
 15  assistant/tool_use/Edit
 12  attachment/task_reminder
  3  assistant/tool_use/Write
  3  user/text
  2  attachment/goal_status
  1  attachment/deferred_tools_delta      ← gbrain's 60+ tools registered here
  0  assistant/tool_use/mcp__gbrain__*    ← ZERO brain interactions
```

**Result: gbrain was completely ignored.** Zero `mcp__gbrain__*` calls. The
MCP tools were registered (visible in `deferred_tools_delta`) but never
invoked. No goal page created. No decisions captured. No debug trails stored.
No knowledge persisted for future sessions.

### What should have happened

Per the gbrain skill files loaded into `~/.claude/CLAUDE.md`:

1. **RESOLVER.md** routes intents to skills. The "always-on" table says
   signal-detector fires on every inbound message and brain-ops handles
   any brain read/write.

2. **signal-detector/SKILL.md** should detect entities on every message:
   goals being worked on, technical decisions, debug sessions, reusable
   concepts.

3. **brain-ops/SKILL.md** defines the 6-phase loop:
   DETECT -> WRITE -> STORE -> RETRIEVE -> PRESENT -> ENRICH.
   Phase 1 (READ): search brain for prior context before starting work.
   Phase 2 (WRITE): create/update pages as entities are detected.
   Phase 4 (ENRICH): update existing pages with new information.

### What the ideal session would have looked like

The 85 actual tool calls should have been interleaved with ~11 gbrain calls:

```
Session start:
  1. ToolSearch "gbrain"                          ← load MCP schemas
  2. mcp__gbrain__search "email chain splitter"   ← prior context?
  3. mcp__gbrain__put_page goals/email-chain...   ← create goal page (skeleton)

During development (interleaved with the 85 code tool calls):
  4. mcp__gbrain__add_timeline_entry              ← "chain detection working, 3 emails found"
  5. mcp__gbrain__add_timeline_entry              ← debug: avatar initials 'I)' from paren in name
  6. mcp__gbrain__add_timeline_entry              ← debug: RFC 5322 To field raw format
  7. mcp__gbrain__add_timeline_entry              ← "screenshots working via gstack browse"
  8. mcp__gbrain__add_timeline_entry              ← "single-email edge case verified"

Session completion:
  9. mcp__gbrain__put_page goals/email-chain...   ← final update with full execution arc
 10. mcp__gbrain__search "MIME parsing"           ← check for related concepts
 11. mcp__gbrain__add_link                        ← link to related entities if found
```

That is ~13% overhead (11 extra calls on 85). The brain would then contain
a goal page with the full execution arc: approach, local decisions (Python
stdlib, gstack browse for screenshots), debug trails (avatar initials fix,
RFC 5322 encoding fix), and verification evidence. A future `/goal` session
on email parsing would find it via `mcp__gbrain__search "email"` at step 2.

### Why it didn't happen

**Root cause (observed, cross-model agreement between Claude and Codex):**

The gbrain instructions are advisory, not enforced. Under `/goal` execution
pressure, the agent optimizes for the completion condition and skips all
optional side-paths.

Four compounding factors:

1. **"Always-on" is declarative text, not runtime enforcement.** The
   signal-detector's "fire on every message" instruction is a sentence in a
   skill file that the agent must choose to read. The RESOLVER says to read it.
   But the RESOLVER itself is just text in `~/.claude/CLAUDE.md`. Under `/goal`
   pressure, the agent never reads the signal-detector skill file because
   doing so does not advance the goal completion condition.

2. **`/goal` completion condition doesn't include brain I/O.** The goal
   rewards code + tests + artifacts (the `.eml` files, `.png` screenshots,
   `manifest.json`). There is zero incentive for gbrain actions because no
   part of the completion condition checks for brain writes.

3. **Deferred MCP tools add activation friction.** Claude Code defers MCP
   tool schemas when there are too many tools (gbrain has 60+). The agent
   must call `ToolSearch` to load schemas before it can invoke any
   `mcp__gbrain__*` tool. Without a reason to reach for gbrain, `ToolSearch`
   is never called, schemas never load, and the tools remain invisible.

4. **No project-level CLAUDE.md** existed in `/workspaces/goal-email-thread-separator/`.
   The gbrain instructions lived only in `~/.claude/CLAUDE.md` (global) and
   `/workspaces/CLAUDE.md` (parent workspace). Claude Code loads CLAUDE.md
   files hierarchically, but the closest-to-execution-site file (project-level)
   has the strongest influence on agent behavior. Its absence meant no
   reinforcement at the point where the agent was actually working.

### Codex analysis (gpt-5.3-codex)

**First consult:** Asked Codex to independently analyze the root cause.

Codex agreed with the analysis and sharpened the framing: "policy was
advisory, not enforced." Codex also identified a fifth factor: **trigger
mismatch.** The RESOLVER's brain triggers are phrased as explicit knowledge
tasks ("what do we know...", "search for..."). A build task doesn't hit
those triggers unless you force a pre/post memory pattern.

**Codex recommended fixes (priority order):**

1. **Change the `/goal` contract (highest leverage).** Make brain steps part
   of the completion criteria. On start: mandatory `ToolSearch` + search. On
   finish: mandatory `put_page` with outcome. Goal incomplete without the
   brain write. *Requires Claude Code platform changes or a `/goal` wrapper.*

2. **Enforce always-on outside the model.** Implement signal-detector as a
   host-side hook or completion validator that checks for `mcp__gbrain__*`
   calls. *Requires platform changes.*

3. **Reduce deferred-tool friction.** Preload gbrain schemas at session
   start or register a slim MCP server with only the 5-6 critical tools
   (search, put_page, get_page, add_link, query) to stay under the deferral
   threshold. *Partially implementable without platform changes.*

4. **Add workspace-level policy.** In `/workspaces/CLAUDE.md`: mandatory
   gbrain read at goal start, write at goal end. *Immediate, no platform
   changes needed.*

**Second consult:** Asked Codex to review the spec update and CLAUDE.md
patch we wrote for fix #4.

Codex found 4 issues:
- Spec section was misplaced in normative flow (should be in Reviews or
  experiment log, not between "After implementation" and "Spec Update Rules")
- CLAUDE.md patch is still advisory text -- the same class of instruction the
  root cause says gets ignored. Acknowledged limitation.
- No failure policy for when gbrain is unavailable. Fixed: added degraded
  mode with `gbrain-deferred.md` fallback.
- Risk of knowledge spam from over-eager page creation. Fixed: restricted
  mandatory writes to ONE goal page per session; separate pages only for
  clearly durable cross-goal decisions.

### What we changed

1. **`/workspaces/CLAUDE.md`** (inside devcontainer): appended "GBrain Goal
   Integration (mandatory)" section with:
   - On goal start: ToolSearch + search for prior context
   - On goal completion: ONE goal page via put_page
   - Degraded mode: if gbrain unavailable, write `gbrain-deferred.md`
   - Completion criteria: goal incomplete without brain write OR deferred file

2. **Spec** (`docs/specs/customized-domain.md`): added pointer in Codex review
   7 to this experiment log.

### Open gap

Fix #4 (workspace-level CLAUDE.md policy) is a stopgap. It is the same class
of advisory text that the root cause says gets ignored under execution
pressure. The real fixes (#1-#3) require either:

- Claude Code platform changes (goal hooks, tool preloading)
- A slim gbrain MCP server with <10 tools to avoid deferral
- A `/goal` wrapper that enforces brain I/O in the completion check

### Next experiment

Re-run the same `/goal` session with the CLAUDE.md patch in place. Check
whether the advisory text is sufficient to trigger gbrain calls, or whether
fix #3 (slim MCP server) is needed to overcome the deferred-tool friction.

---

## Experiment 2: `/goal` fails on spec character limit

**Date:** 2026-05-16
**Session ID:** `7839c8d6-9ed8-41a6-94f9-510f71c2e47f`
**Project:** `/workspaces/goal-email-thread-count`
**Duration:** ~40 seconds (15 log lines)

### What we tried

Ran `/goal goal-email-thread-count.md` on a new project spec for an email
thread counter tool.

### What happened

The `/goal` command rejected the spec immediately:

```
Goal condition is limited to 4000 characters (got 4045)
```

The spec was 45 characters over the limit. The user ran `/exit` immediately
after. Claude never got a turn to respond. Zero tool calls.

### Tool histogram

```
  0  assistant/tool_use/*     ← no assistant messages at all
  3  user (command payloads: /goal, error output, /exit)
  3  system
```

### gbrain calls: ZERO

No agent turn occurred, so gbrain could not have been invoked.

### Observation

The `/goal` command has a 4000-character limit on the spec file content. This
is a hard gate enforced by the Claude Code harness before the agent gets
control. Specs need to stay under this limit, or the goal file must reference
an external spec rather than embedding the full content.

---

## Experiment 3: Goal announcement stub (no `/goal` command)

**Date:** 2026-05-16
**Session ID:** `aa5faea5-661d-4776-a7b1-f928d7dcf9ab`
**Project:** `/workspaces/goal-email-thread-count`
**Duration:** ~2.7 seconds (15 log lines)

### What we tried

User sent: "heads up, we're going to work on a goal now. just informing."

### What happened

Agent replied: "Got it, standing by for the goal. Ready when you are."
One text response, zero tool calls. Session ended.

### Tool histogram

```
  0  assistant/tool_use/*     ← zero tool calls
  1  assistant/text
  1  user/text
```

### gbrain calls: ZERO

The gbrain tools were registered in `deferred_tools_delta` but the agent
had no reason to invoke them on a single notification message. The signal-
detector's "fire on every message" instruction was not followed, but this
is a trivial case -- no substantive content to detect signals in.

### Observation

A "heads up" message with no actionable content does not trigger gbrain
interaction. This is arguably correct behavior -- there is nothing to
store. However, per the RESOLVER, signal-detector should still fire and
determine there is nothing to capture (rather than not firing at all).

---

## Experiment 4: gbrain fires on conversational goal start

**Date:** 2026-05-16
**Session ID:** `9b03a158-10fa-473f-8355-ba3b857f49f9`
**Project:** `/workspaces/goal-email-thread-count`
**Duration:** ~69 seconds (46 log lines)

### What we tried

User said: "we're going to work on a goal now" (natural language, no `/goal`
command). The project had a spec at `goal-email-thread-count.md` and a sample
`.eml` file.

### What happened

The agent:
1. Called `ToolSearch` to load gbrain MCP tool schemas
2. Called `mcp__gbrain__search` with query `"email thread count"`
3. Read the goal spec file
4. Explored the sample `.eml` file with `Bash` (grepping for boundary markers)
5. Was interrupted by the user mid-investigation

No code was written. The agent was still in the discovery/analysis phase when
interrupted.

### Tool histogram

```
  4  assistant/tool_use/Bash
  2  assistant/tool_use/Read
  1  assistant/tool_use/ToolSearch     ← loaded gbrain schemas
  1  assistant/tool_use/mcp__gbrain__search  ← SEARCHED THE BRAIN
```

### gbrain calls: 1

| Tool | Input |
|------|-------|
| `mcp__gbrain__search` | `{"query": "email thread count"}` |

### Why gbrain fired here but not in Experiment 1

This is the key finding. The difference between this session and Experiment 1:

| Factor | Experiment 1 (`/goal`) | Experiment 4 (conversational) |
|--------|----------------------|-------------------------------|
| Entry mode | `/goal` command with completion condition | Natural language prompt |
| Execution pressure | High -- agent focused on completion gate | Low -- agent in exploration mode |
| First action | Read spec, start coding | ToolSearch + brain search |
| gbrain calls | 0 | 1 (search) |

The `/goal` command creates a completion condition that the agent optimizes
for, suppressing optional side-paths. A conversational prompt ("we're going
to work on a goal now") does NOT create a completion gate, so the agent
follows the CLAUDE.md instructions (including gbrain search) because there
is no competing objective.

This confirms the root cause from Experiment 1: **it is the `/goal` execution
pressure that suppresses gbrain, not the instructions themselves.** When the
same instructions are followed without `/goal` pressure, gbrain fires correctly.

### Observation

The `/workspaces/CLAUDE.md` patch (fix #4 from Experiment 1) may be
effective for conversational goal sessions but insufficient for `/goal`
sessions where the completion gate overrides advisory instructions. Fix #1
(changing the `/goal` contract) remains necessary for `/goal`-mode sessions.

---

## Experiment 5: `/init` skill runs instead of `/goal`

**Date:** 2026-05-16
**Session ID:** `9cacd941-807c-4f7c-9bad-15fe9ad9d23d`
**Project:** `/workspaces/goal-email-thread-count`
**Duration:** ~2 minutes (37 log lines)

### What we tried

User mentioned they wanted to work on the goal and use the `/goal` skill.

### What happened

The agent ran the `/init` skill (via `Skill` tool) instead of `/goal`, then
dispatched an `Agent` subagent, read the spec and CLAUDE.md, and updated
CLAUDE.md with a project overview and spec summary. No implementation work.
No brain writes.

### Tool histogram

```
  2  assistant/tool_use/Read
  1  assistant/tool_use/ToolSearch
  1  assistant/tool_use/Skill (/init)
  1  assistant/tool_use/Agent
  1  assistant/tool_use/Edit
```

### gbrain calls: ZERO

`ToolSearch` was called (loading gbrain schemas), but no `mcp__gbrain__*`
tool was invoked. The agent loaded the schemas but did not use them.

### Why gbrain didn't fire

The `/init` skill took over the session. It has its own workflow (read the
repo, update CLAUDE.md with project info) that does not include brain
writes. The agent followed the `/init` skill instructions faithfully,
which is the correct behavior for skill execution, but `/init` does not
include any gbrain interaction.

### Observation

When a skill (like `/init`) takes control of the session, it supersedes
the gbrain instructions in CLAUDE.md. Skills define their own tool
sequences and do not check the brain. This is a third mode of failure
distinct from Experiments 1 and 4:

1. **`/goal` mode** -- completion pressure suppresses brain (Experiment 1)
2. **Conversational mode** -- brain fires correctly (Experiment 4)
3. **Skill mode** -- skill workflow supersedes brain instructions (Experiment 5)

For gbrain to fire during skill execution, individual skills would need
to include brain read/write steps in their own workflows.

---

## Cross-experiment summary

| # | Session | Mode | Duration | Tool calls | gbrain calls | Brain fired? |
|---|---------|------|----------|------------|-------------|-------------|
| 1 | `94bb5cda` | `/goal` | ~30min | 85 | 0 | No |
| 2 | `7839c8d6` | `/goal` (failed) | 40s | 0 | 0 | N/A (no agent turn) |
| 3 | `aa5faea5` | Conversational (stub) | 2.7s | 0 | 0 | No (nothing to detect) |
| 4 | `9b03a158` | Conversational | 69s | 8 | 1 (search) | **Yes** |
| 5 | `9cacd941` | Skill (`/init`) | 2min | 6 | 0 | No |

### Key finding

gbrain fires in exactly ONE mode: **conversational prompts without a
competing execution framework** (no `/goal` gate, no skill workflow).

Three distinct suppression mechanisms observed:
1. `/goal` completion pressure overrides advisory brain instructions
2. Skill workflows (e.g. `/init`) supersede CLAUDE.md brain instructions
3. Deferred MCP tools add friction (but ToolSearch is called when there is no competing pressure)

### Implication for fixes

Fix #4 (CLAUDE.md policy) is likely effective only for conversational
mode. For `/goal` and skill modes, enforcement must be built into the
execution framework itself (fix #1) or into individual skill workflows.
