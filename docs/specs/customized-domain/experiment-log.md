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
