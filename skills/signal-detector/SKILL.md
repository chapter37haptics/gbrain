---
name: signal-detector
version: 2.0.0
description: |
  Always-on ambient signal capture for developer knowledge. Fires on every
  inbound message to detect goals, decisions, processes, concepts, and
  original thinking. Spawn as a cheap sub-agent in parallel, never block
  the main response.
triggers:
  - every inbound message (always-on)
tools:
  - search
  - query
  - get_page
  - put_page
  - add_link
  - add_timeline_entry
mutating: true
writes_pages: true
writes_to:
  - goals/
  - decisions/
  - processes/
  - concepts/
---

# Signal Detector — Developer Knowledge Capture

Lightweight sub-agent that fires on every inbound message to capture TWO things
with EQUAL priority:

1. **Original thinking** — the user's ideas, observations, frameworks
2. **Developer knowledge signals** — goals, decisions, processes, concepts

Original thinking is AT LEAST as valuable as entity extraction. Ideas are the
intellectual capital. Entities are bookkeeping. Both compound over time.

## Contract

This skill guarantees:
- Fires on every message (no exceptions unless purely operational)
- Runs in parallel (spawned, never blocks main response)
- Captures ideas with the user's EXACT phrasing (no paraphrasing)
- Detects developer knowledge signals and creates/enriches brain pages
- Logs a one-line summary of what was captured
- Back-links all entity mentions (Iron Law)
- Citations on every fact written

> **Convention:** See `skills/conventions/quality.md` for Iron Law back-linking.

Every time this skill creates or updates a brain page that mentions another entity:
1. Check if that entity has a brain page
2. If yes → add a back-link FROM their page TO the page you just created/updated
3. Format: `- **YYYY-MM-DD** | Referenced in [page title](path) — brief context`
4. An unlinked mention is a broken brain.

## Phases

### Phase 1: Idea/Observation Detection (PRIMARY)

When the user expresses a novel thought, observation, thesis, or framework:
- If it's the user's **original thinking** (they generated it) → create/update `concepts/{slug}`
- If it's a **reusable pattern or mental model** → create/update `concepts/{slug}`

**Capture exact phrasing.** The user's language IS the insight. Don't paraphrase.

**Cross-linking (MANDATORY):** Every concept MUST link to related goals, decisions,
and processes. A concept without cross-links is a dead concept.

### Phase 2: Developer Knowledge Detection (SECONDARY)

Scan every message for these signals:

1. **Goal signals** — "set up JWT auth", "migrate to Postgres", "fix the deploy",
   any /goal invocation or development task being worked on
   - Check brain: `gbrain search "goal name"`
   - If no page → create `goals/{slug}` with approach, environment, initial state
   - If page exists → update with new progress, debug trails, decisions made

2. **Decision signals** — "we chose X because Y", "decided to", "tradeoff",
   "going with", "ruling out"
   - If the decision governs future work beyond this goal → create `decisions/{slug}`
   - If the decision is local to the current goal → log on the goal page
   - Always record: what was decided, why, what alternatives were considered

3. **Process signals** — "to deploy, you need to", "the workflow is", "steps to",
   "how to set up", repeatable sequences
   - Create `processes/{slug}` with preconditions, steps, verification
   - Only if the process is reproducible and handoff-worthy

4. **Concept signals** — "event sourcing works by", "the repository pattern",
   "Docker needs this flag because", tool knowledge, pattern explanations
   - Create/update `concepts/{slug}` with context-free reusable understanding
   - Must be: reusable, cross-goal, stable, non-procedural

5. **Debug signals** — "the bug was caused by", "root cause was", "fixed by"
   - Add structured timeline entry to the active goal page (NOT a separate page)
   - Format: `- **YYYY-MM-DD** | Debug — **Symptom:** X. **Root cause:** Y. **Fix:** Z.`

For each entity:
- `gbrain search "name"` — does a page exist?
- If NO page → check notability (see quality.md). If notable, create with enrichment.
- If page exists but THIN → enrich with new information
- If page exists and RICH → add timeline entry if there's new dated information

**Auto-link (v0.10.1):** When you write/update a page that references another
entity, the auto-link post-hook on `put_page` automatically creates the graph
edge. You don't need to call `gbrain link` manually. Timeline entries still
need explicit calls.

### Phase 3: Signal Logging

Always log a one-line summary:
- `Signals: 0 ideas, 0 entities, 0 facts (skipped: operational)`
- `Signals: 1 concept (captured → concepts/x), 1 goal (updated → goals/y), 1 decision (created → decisions/z)`

This makes the ambient capture loop debuggable.

## Output Format

No visible output to the user. This skill runs silently in the background.
The output is brain pages created/updated and the signal log line.

## Anti-Patterns

- Blocking the main response to wait for signal detection to complete
- Paraphrasing the user's original thinking instead of capturing exact phrasing
- Creating pages for non-notable entities (one-off mentions, sub-steps)
- Skipping back-links after creating/updating pages
- Running on purely operational messages ("ok", "thanks", "do it")
- Creating a separate page for debug trails (they go on the goal page)
- Filing a concept that's really a process (if it has steps, it's a process)

## Tools Used

- `search` — check if entity page exists
- `query` — semantic search for related context
- `get_page` — load existing entity pages
- `put_page` — create/update brain pages
- `add_link` — cross-reference entities
- `add_timeline_entry` — record events on entity timelines
