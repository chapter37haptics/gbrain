# Customized Domain (VC to Developer) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Adapt gbrain's skill layer from a VC/executive knowledge domain to a developer knowledge domain, replacing people/companies/deals entity detection with goals/decisions/processes/concepts.

**Architecture:** Code-first, then skills. Three narrow code patches extend the `PageType` union, `inferType()` directory mapper, and `DIR_PATTERN` auto-link regex to recognize new entity types. Six skill files are then rewritten or patched to redirect the agent's detection, filing, retrieval, and quality gates from VC entities to developer entities. No schema, pipeline, or MCP changes.

**Tech Stack:** TypeScript (Bun runtime), Markdown skill files, JSON config

---

## File Map

| Action | File | Responsibility |
|--------|------|----------------|
| Patch | `src/core/types.ts:13,22-27` | Add `goal`, `decision`, `process` to `PageType` union + `ALL_PAGE_TYPES` array |
| Patch | `test/page-type-exhaustive.test.ts:63-89` | Add `goal`, `decision`, `process` cases to exhaustive switch |
| Patch | `src/core/markdown.ts:344-375` | Add `goals/`, `decisions/`, `processes/` to `inferType()` |
| Patch | `src/core/link-extraction.ts:46` | Add `goals`, `decisions`, `processes` to `DIR_PATTERN` regex |
| Rewrite | `skills/conventions/quality.md` | Generalize Iron Law, add developer notability criteria |
| Patch | `skills/brain-ops/SKILL.md` | Replace 8 VC hard-gates with developer entity references |
| Rewrite | `skills/signal-detector/SKILL.md` | Replace VC detection with developer signal detection |
| Rewrite | `skills/_brain-filing-rules.md` | Replace VC taxonomy with developer filing taxonomy |
| Patch | `skills/_brain-filing-rules.json` | Add `goal`, `decision`, `process` kinds + dream paths |
| Rewrite | `skills/RESOLVER.md` | Replace VC triggers with developer triggers |
| Patch | `skills/conventions/brain-first.md` | Replace VC entity conventions with developer entity table |

---

## Task 1: Add developer PageTypes to type system

**Files:**
- Modify: `src/core/types.ts:13` (PageType union)
- Modify: `src/core/types.ts:22-27` (ALL_PAGE_TYPES array)
- Modify: `test/page-type-exhaustive.test.ts:63-89` (exhaustive switch)

- [ ] **Step 1: Add `goal`, `decision`, `process` to the `PageType` union**

In `src/core/types.ts` line 13, append the three new types before the closing semicolon:

```typescript
export type PageType = 'person' | 'company' | 'deal' | 'yc' | 'civic' | 'project' | 'concept' | 'source' | 'media' | 'writing' | 'analysis' | 'guide' | 'hardware' | 'architecture' | 'meeting' | 'note' | 'email' | 'slack' | 'calendar-event' | 'code' | 'image' | 'synthesis' | 'goal' | 'decision' | 'process';
```

- [ ] **Step 2: Add the same three types to `ALL_PAGE_TYPES`**

In `src/core/types.ts` lines 22-27, add the three new types to the array:

```typescript
export const ALL_PAGE_TYPES: readonly PageType[] = [
  'person', 'company', 'deal', 'yc', 'civic', 'project', 'concept',
  'source', 'media', 'writing', 'analysis', 'guide', 'hardware',
  'architecture', 'meeting', 'note', 'email', 'slack', 'calendar-event',
  'code', 'image', 'synthesis', 'goal', 'decision', 'process',
] as const;
```

- [ ] **Step 3: Add cases to the exhaustive switch in the contract test**

In `test/page-type-exhaustive.test.ts`, add three cases to the `classify` function (lines 63-89) before the `default`:

```typescript
        case 'synthesis': return 'doc';
        case 'goal': return 'work';
        case 'decision': return 'doc';
        case 'process': return 'doc';
        default: return assertNever(t);
```

- [ ] **Step 4: Run typecheck to verify the union is consistent**

Run: `bun run typecheck`
Expected: PASS (no type errors). If any switch/assertNever consumer fails, it means there's an exhaustive switch elsewhere that needs new cases — fix those before proceeding.

- [ ] **Step 5: Run unit tests to verify contract test passes**

Run: `bun test test/page-type-exhaustive.test.ts`
Expected: All 4 tests pass, including the round-trip and exhaustive switch tests.

- [ ] **Step 6: Commit**

```bash
git add src/core/types.ts test/page-type-exhaustive.test.ts
git commit -m "feat: add goal, decision, process to PageType union"
```

---

## Task 2: Add developer directory mappings to `inferType()`

**Files:**
- Modify: `src/core/markdown.ts:344-375` (inferType function)
- Test: `test/markdown.test.ts`

- [ ] **Step 1: Write failing tests for the three new directory mappings**

Add tests to `test/markdown.test.ts` inside the existing `inferType` / `parseMarkdown` describe block. Find the section that tests type inference from file paths (look for `people/someone.md` test around line 70-89) and add after it:

```typescript
  test('inferType: goals/ → goal', () => {
    const result = parseMarkdown('---\ntitle: Test\n---\nBody', 'goals/setup-jwt-auth.md');
    expect(result.type).toBe('goal');
  });

  test('inferType: decisions/ → decision', () => {
    const result = parseMarkdown('---\ntitle: Test\n---\nBody', 'decisions/chose-postgres.md');
    expect(result.type).toBe('decision');
  });

  test('inferType: processes/ → process', () => {
    const result = parseMarkdown('---\ntitle: Test\n---\nBody', 'processes/deploy-to-prod.md');
    expect(result.type).toBe('process');
  });

  test('inferType: decisions/ under projects/ → decision (longest prefix)', () => {
    const result = parseMarkdown('---\ntitle: Test\n---\nBody', 'projects/my-app/decisions/use-redis.md');
    expect(result.type).toBe('decision');
  });
```

- [ ] **Step 2: Run the tests to confirm they fail**

Run: `bun test test/markdown.test.ts`
Expected: The four new tests FAIL (goals/ returns `concept`, decisions/ returns `concept`, processes/ returns `concept`, nested decisions/ returns `project`).

- [ ] **Step 3: Add the directory mappings to `inferType()`**

In `src/core/markdown.ts`, add three lines inside `inferType()`. Place them BEFORE the `/projects/` check (line 364) so `decisions/` under `projects/` matches `decision` first (longest prefix wins):

```typescript
  if (lower.includes('/goals/') || lower.includes('/goal/')) return 'goal';
  if (lower.includes('/decisions/') || lower.includes('/decision/')) return 'decision';
  if (lower.includes('/processes/') || lower.includes('/process/')) return 'process';
  if (lower.includes('/projects/') || lower.includes('/project/')) return 'project';
```

The three new lines go right before the existing `projects/` line. Do NOT remove or change any existing lines — the VC directory mappings stay for backward compatibility.

- [ ] **Step 4: Run the tests to confirm they pass**

Run: `bun test test/markdown.test.ts`
Expected: All tests pass, including the four new ones.

- [ ] **Step 5: Run typecheck**

Run: `bun run typecheck`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add src/core/markdown.ts test/markdown.test.ts
git commit -m "feat: add goals/decisions/processes directory mappings to inferType"
```

---

## Task 3: Add developer directories to `DIR_PATTERN` auto-link regex

**Files:**
- Modify: `src/core/link-extraction.ts:46` (DIR_PATTERN)
- Test: `test/link-extraction.test.ts`

- [ ] **Step 1: Write failing tests for entity ref extraction from developer directories**

Add tests to `test/link-extraction.test.ts` inside the existing `extractEntityRefs` describe block:

```typescript
  test('extractEntityRefs: goals/ directory link', () => {
    const refs = extractEntityRefs('[Setup JWT](goals/setup-jwt-auth)');
    expect(refs).toEqual([{ name: 'Setup JWT', slug: 'goals/setup-jwt-auth' }]);
  });

  test('extractEntityRefs: decisions/ directory link', () => {
    const refs = extractEntityRefs('[Chose Postgres](decisions/chose-postgres)');
    expect(refs).toEqual([{ name: 'Chose Postgres', slug: 'decisions/chose-postgres' }]);
  });

  test('extractEntityRefs: processes/ directory link', () => {
    const refs = extractEntityRefs('[Deploy Flow](processes/deploy-to-prod)');
    expect(refs).toEqual([{ name: 'Deploy Flow', slug: 'processes/deploy-to-prod' }]);
  });

  test('extractEntityRefs: goals/ wikilink', () => {
    const refs = extractEntityRefs('[[goals/setup-jwt-auth|Setup JWT]]');
    expect(refs).toEqual([{ name: 'Setup JWT', slug: 'goals/setup-jwt-auth' }]);
  });
```

- [ ] **Step 2: Run tests to confirm they fail**

Run: `bun test test/link-extraction.test.ts`
Expected: The four new tests FAIL (DIR_PATTERN doesn't match goals/decisions/processes).

- [ ] **Step 3: Add `goals`, `decisions`, `processes` to `DIR_PATTERN`**

In `src/core/link-extraction.ts` line 46, add the three new directories to the regex alternation. Place them at the beginning (longest-first for the regex engine):

```typescript
const DIR_PATTERN = '(?:goals|decisions|processes|people|companies|meetings|concepts|deal|civic|project|projects|source|media|yc|tech|finance|personal|openclaw|entities)';
```

- [ ] **Step 4: Run tests to confirm they pass**

Run: `bun test test/link-extraction.test.ts`
Expected: All tests pass, including the four new ones.

- [ ] **Step 5: Run typecheck**

Run: `bun run typecheck`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add src/core/link-extraction.ts test/link-extraction.test.ts
git commit -m "feat: add goals/decisions/processes to DIR_PATTERN auto-link"
```

---

## Task 4: Rewrite `quality.md` — root of the delegation chain

**Files:**
- Rewrite: `skills/conventions/quality.md`

This is the most important skill file change. Every other file's Iron Law and notability gate delegates here. The VC scoping ("person or company") must become entity-generic.

- [ ] **Step 1: Rewrite `quality.md`**

Replace the entire contents of `skills/conventions/quality.md` with:

```markdown
# Quality Convention

Cross-cutting quality rules for all brain-writing skills.

## Citations (MANDATORY)

Every fact written to a brain page must carry an inline `[Source: ...]` citation.

- **User's statements:** `[Source: User, {context}, YYYY-MM-DD]`
- **Meeting data:** `[Source: Meeting "{title}", YYYY-MM-DD]`
- **Email/message:** `[Source: email from {name} re: {subject}, YYYY-MM-DD]`
- **Web content:** `[Source: {publication}, {URL}, YYYY-MM-DD]`
- **Social media:** `[Source: X/@handle, YYYY-MM-DD](URL)`
- **Synthesis:** `[Source: compiled from {sources}]`

### Source precedence (highest to lowest)

1. User's direct statements (highest authority)
2. Compiled truth (brain's synthesized understanding)
3. Timeline entries (raw evidence)
4. External sources (API enrichment, web search)

## Back-Linking (MANDATORY)

Every mention of an entity WITH a brain page MUST create a back-link
FROM that entity's page TO the page mentioning it.

Entities: goals, decisions, processes, concepts — any page in a recognized
entity directory.

Format: `- **YYYY-MM-DD** | Referenced in [page title](path) -- context`

An unlinked mention is a broken brain.

## Notability Gate

Before creating a new brain page, check notability:

- **Goals:** Is this a distinct execution arc worth documenting? (Not a sub-step of an existing goal)
- **Decisions:** Does this choice govern future work beyond the current goal?
- **Processes:** Is this repeatable and handoff-worthy? (Not a one-off sequence)
- **Concepts:** Reusable across goals? Stable? Non-procedural? (If it's steps, it's a process)

When in doubt, capture in the current goal page first. Promote to its own page
only when reuse is clear. A missing page can be created later. A junk page
wastes attention and degrades search quality.
```

- [ ] **Step 2: Verify the file reads correctly**

Run: `cat skills/conventions/quality.md`
Expected: The full new content with developer-domain notability criteria.

- [ ] **Step 3: Commit**

```bash
git add skills/conventions/quality.md
git commit -m "feat: generalize quality.md Iron Law and notability gate for developer domain"
```

---

## Task 5: Patch `brain-ops/SKILL.md` — the loop engine (8 sites)

**Files:**
- Modify: `skills/brain-ops/SKILL.md`

Eight hard-gate sites say "person or company" and must be changed to developer entity references. The `writes_to` frontmatter also needs updating.

- [ ] **Step 1: Update `writes_to` frontmatter (lines 22-26)**

Replace:
```yaml
writes_to:
  - people/
  - companies/
  - deals/
  - concepts/
  - meetings/
```

With:
```yaml
writes_to:
  - goals/
  - decisions/
  - processes/
  - concepts/
```

- [ ] **Step 2: Update Iron Law scope (line 49)**

Replace:
```
Every mention of a person or company with a brain page MUST create a back-link
```

With:
```
Every mention of an entity with a brain page MUST create a back-link
```

- [ ] **Step 3: Update Phase 1 description (line 57)**

Replace:
```
Before using ANY external API to research a person, company, or topic:
```

With:
```
Before using ANY external API to research a goal, decision, process, or concept:
```

- [ ] **Step 4: Update Phase 2 trigger (lines 69-71)**

Replace:
```
Every message, meeting, email, or conversation that references a person or company:

1. **Detect entities** — people, companies, deals mentioned
```

With:
```
Every message or conversation that references a goal, decision, process, or concept:

1. **Detect entities** — goals, decisions, processes, concepts mentioned
```

- [ ] **Step 5: Update Phase 2.5 link types (lines 88-89)**

Replace:
```
- Inferred link types: `attended` (meeting -> person), `works_at`, `invested_in`,
  `founded`, `advises`, `source` (frontmatter), `mentions` (default).
```

With:
```
- Inferred link types: `uses` (goal -> concept), `decided_in` (decision -> goal),
  `depends_on` (process -> concept), `source` (frontmatter), `mentions` (default).
```

- [ ] **Step 6: Update Phase 3 description (line 98)**

Replace:
```
Before answering any question about a person, company, or topic:
```

With:
```
Before answering any question about a goal, decision, process, or concept:
```

- [ ] **Step 7: Update Phase 4 ambient enrichment triggers (lines 111-112)**

Replace:
```
- Person mentioned → check brain, create/enrich if needed (spawn background)
- Company mentioned → same
```

With:
```
- Goal mentioned → check brain, create/update if needed (spawn background)
- Decision/process/concept mentioned → same
```

- [ ] **Step 8: Update anti-patterns (line 147)**

Replace:
```
- Answering questions about people/companies without checking the brain first
```

With:
```
- Answering questions about goals/decisions/processes/concepts without checking the brain first
```

- [ ] **Step 9: Verify the file reads correctly**

Run: `cat skills/brain-ops/SKILL.md | head -60`
Expected: Updated frontmatter with developer directories and generalized Iron Law.

- [ ] **Step 10: Commit**

```bash
git add skills/brain-ops/SKILL.md
git commit -m "feat: patch brain-ops 8 hard-gate sites for developer domain"
```

---

## Task 6: Rewrite `signal-detector/SKILL.md`

**Files:**
- Rewrite: `skills/signal-detector/SKILL.md`

Replace the VC-oriented entity detection with developer-domain signal detection. The signal detector fires on every message and is the entry point for knowledge capture.

- [ ] **Step 1: Rewrite the entire file**

Replace the entire contents of `skills/signal-detector/SKILL.md` with:

```markdown
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
```

- [ ] **Step 2: Verify the file reads correctly**

Run: `head -30 skills/signal-detector/SKILL.md`
Expected: Updated frontmatter with `writes_to: goals/, decisions/, processes/, concepts/` and version 2.0.0.

- [ ] **Step 3: Commit**

```bash
git add skills/signal-detector/SKILL.md
git commit -m "feat: rewrite signal-detector for developer domain knowledge capture"
```

---

## Task 7: Rewrite `_brain-filing-rules.md` and patch `_brain-filing-rules.json`

**Files:**
- Rewrite: `skills/_brain-filing-rules.md`
- Modify: `skills/_brain-filing-rules.json`

- [ ] **Step 1: Rewrite `_brain-filing-rules.md`**

Replace the entire contents of `skills/_brain-filing-rules.md` with:

```markdown
# Brain Filing Rules -- MANDATORY for all skills that write to the brain

## The Rule

The PRIMARY SUBJECT of the content determines where it goes. Not the format,
not the source, not the skill that's running.

## Decision Protocol

1. Identify the primary subject (a goal? decision? process? concept?)
2. File in the directory that matches the subject
3. Cross-link from related directories
4. When in doubt: what would you search for to find this page again?

## Operational Rule

Capture everything in `goals/` first. Promote out only when reusable:
- `decision` — if the choice should constrain other goals
- `process` — if it's reproducible and handoff-worthy
- `concept` — if it generalizes beyond the specific case

## Common Misfiling Patterns -- DO NOT DO THESE

| Wrong | Right | Why |
|-------|-------|-----|
| Local decision on goal page → `decisions/` | Keep on `goals/` page | Only durable cross-goal choices go to decisions/ |
| One-off command sequence → `processes/` | Keep on `goals/` page | processes/ is for repeatable, handoff-worthy workflows |
| Project-specific config note → `concepts/` | Keep on `goals/` page | concepts/ is for context-free reusable knowledge |
| Reusable pattern buried in goal page | → `concepts/` | If it applies to more than one goal, promote it |
| Debug trail → separate page | → timeline entry on `goals/` page | Debug trails are structured timeline entries, not pages |
| A series of steps → `concepts/` | → `processes/` | If it has steps, it's a process |

## MECE Boundaries (hard rules)

| Pair | Boundary |
|------|----------|
| goals/ vs decisions/ | goals: what happened in one execution run. decisions: durable choice meant to govern future goals |
| goals/ vs processes/ | goals: narrative + debug trail. processes: canonical reproducible procedure (no session story) |
| goals/ vs concepts/ | goals: applied, context-bound. concepts: context-free reusable understanding |
| decisions/ vs processes/ | decisions: what/why we chose. processes: how to execute |
| decisions/ vs concepts/ | decisions: committed policy for a scope. concepts: explanatory model, no commitment |
| processes/ vs concepts/ | processes: stepwise action. concepts: theory/pattern vocabulary |

## Sanctioned exception: synthesis output is sui generis

The "file by primary subject" rule is for raw ingest. Synthesized output that
is one-of-one to a single source AND a specific reader does not fit any
subject directory cleanly.

Format-prefixed paths under `media/<format>/<slug>` are the sanctioned
exception:

- `media/books/<slug>-personalized.md` (book-mirror output)
- `media/articles/<slug>-personalized.md` (long-form article personalization)

## What `sources/` Is Actually For

`sources/` is ONLY for:
- Bulk data imports (API dumps, CSV exports, snapshots)
- Raw data that feeds multiple brain pages
- Periodic captures (quarterly snapshots, sync exports)

If the content has a clear primary subject (a goal, decision, process, concept),
it does NOT go in sources/. Period.

## Notability Gate

Not everything deserves a brain page. Before creating a new entity page:
- **Goals:** Is this a distinct execution arc? (Not a sub-step of an existing goal)
- **Decisions:** Does this choice govern future work beyond the current goal?
- **Processes:** Is this repeatable and handoff-worthy? (Not a one-off sequence)
- **Concepts:** Reusable across goals? Stable? Non-procedural?
- **When in doubt, DON'T create.** Capture on the goal page first. Promote later.

## Iron Law: Back-Linking (MANDATORY)

Every mention of an entity with a brain page MUST create a back-link
FROM that entity's page TO the page mentioning it. This is bidirectional:
the new page links to the entity, AND the entity's page links back.

Format for back-links (append to Timeline or See Also):
```
- **YYYY-MM-DD** | Referenced in [page title](path/to/page.md) -- brief context
```

An unlinked mention is a broken brain. The graph is the intelligence.

## Citation Requirements (MANDATORY)

Every fact written to a brain page must carry an inline `[Source: ...]` citation.

Three formats:
- **Direct attribution:** `[Source: User, {context}, YYYY-MM-DD]`
- **API/external:** `[Source: {provider}, YYYY-MM-DD]` or `[Source: {publication}, {URL}]`
- **Synthesis:** `[Source: compiled from {list of sources}]`

Source precedence (highest to lowest):
1. User's direct statements (highest authority)
2. Compiled truth (pre-existing brain synthesis)
3. Timeline entries (raw evidence)
4. External sources (API enrichment, web search -- lowest)

When sources conflict, note the contradiction with both citations. Don't
silently pick one.

## Raw Source Preservation

Every ingested item should have its raw source preserved for provenance.

**Size routing (automatic via `gbrain files upload-raw`):**
- **< 100 MB text/PDF**: stays in the brain repo (git-tracked) in a `.raw/`
  sidecar directory alongside the brain page
- **>= 100 MB OR media files** (video, audio, images): uploaded to cloud
  storage with a `.redirect.yaml` pointer left in the brain repo.

## Dream-cycle synthesize / patterns directories (v0.23)

The `synthesize` and `patterns` phases of `gbrain dream` write to a
**fixed allow-list** of paths sourced from `_brain-filing-rules.json`'s
`dream_synthesize_paths.globs` array. Editing that JSON is the ONLY way
to add a new directory the synthesis subagent may write to.

## Brain-to-skill promotion pipeline

When a process proves repeatable (2-3 times with only argument changes),
it graduates from a `processes/` brain page to an actual skill file:

- Brain stores: context, evidence, tradeoffs, project-specific constraints, debug history
- Skill files store: stable, parameterized procedures with deterministic steps
- Promotion rule: if reused successfully 2-3 times with only argument changes, graduate to a skill
- Bidirectional links: process page links to skill file path, skill references source brain pages

## Takes attribution (v0.32+)

When writing a `<!--- gbrain:takes:begin -->` fence, the **holder** column says
WHO BELIEVES the claim, not who it's ABOUT.

1. **Holder ≠ subject.** The test: did this person SAY or CLEARLY IMPLY this?
2. **Atomic claims.** Split compound rows into separate rows. One claim per row.
3. **Amplification ≠ endorsement.** A retweet-only signal caps at `weight 0.55`.
4. **Self-reported ≠ verified.** Self-report → `weight=0.75`, not `holder=world/1.0`.
5. **No false precision.** Use 0.05 increments only.
6. **"So what" test.** Skip metadata-style trivia.
```

- [ ] **Step 2: Add `goal`, `decision`, `process` kinds to `_brain-filing-rules.json`**

In `skills/_brain-filing-rules.json`, add three new rule objects to the `rules` array. Insert them after the existing `concept` rule (after line 36):

```json
    {
      "kind": "goal",
      "directory": "goals/",
      "examples": ["development tasks", "/goal executions", "debug sessions"],
      "description": "One /goal execution arc: what was attempted, what happened, decisions made, debug trails, what was learned. The primary authoring unit — capture here first, promote out when reusable."
    },
    {
      "kind": "decision",
      "directory": "decisions/",
      "examples": ["architecture choices", "tool selections", "tradeoff resolutions"],
      "description": "A durable technical choice that governs future work beyond one goal. ADR-style: context, options considered, decision, consequences."
    },
    {
      "kind": "process",
      "directory": "processes/",
      "examples": ["deploy workflows", "setup procedures", "migration runbooks"],
      "description": "A canonical reproducible procedure that is handoff-worthy. Graduates to a skill file after 2-3 successful reuses with only argument changes."
    },
```

- [ ] **Step 3: Add developer directories to `dream_synthesize_paths.globs`**

In `skills/_brain-filing-rules.json`, add three new globs to the `dream_synthesize_paths.globs` array (around line 157-163):

```json
    "globs": [
      "wiki/personal/reflections/*",
      "wiki/originals/*",
      "wiki/personal/patterns/*",
      "wiki/people/*",
      "dream-cycle-summaries/*",
      "goals/*",
      "decisions/*",
      "processes/*"
    ]
```

- [ ] **Step 4: Run the filing-audit test to verify the new kinds are accepted**

Run: `bun test test/filing-audit.test.ts`
Expected: All tests pass. The filing audit reads `_brain-filing-rules.json` for valid directories, so adding the new kinds makes `goals/`, `decisions/`, `processes/` valid `writes_to` targets.

- [ ] **Step 5: Run the skills-conformance test**

Run: `bun test test/skills-conformance.test.ts`
Expected: All tests pass. The signal-detector and brain-ops skills now declare `writes_to` directories that exist in the filing rules JSON.

- [ ] **Step 6: Commit**

```bash
git add skills/_brain-filing-rules.md skills/_brain-filing-rules.json
git commit -m "feat: rewrite filing rules for developer domain taxonomy"
```

---

## Task 8: Rewrite `RESOLVER.md` — routing table

**Files:**
- Rewrite: `skills/RESOLVER.md`

Replace VC-oriented triggers with developer-oriented triggers. Keep the table structure and all non-VC skills (thinking skills, operational, setup, identity).

- [ ] **Step 1: Rewrite `RESOLVER.md`**

Replace the entire contents of `skills/RESOLVER.md`. **IMPORTANT:** All quoted trigger phrases in table rows must remain unchanged — the resolver test (D5/C) fuzzy-matches quoted phrases against each skill's frontmatter triggers. Since we are NOT modifying the underlying skills (query, enrich, data-research, etc.), their trigger phrases must stay the same. Only change unquoted descriptive text and the disambiguation rules.

```markdown
# GBrain Skill Resolver

This is the dispatcher. Skills are the implementation. **Read the skill file before acting.** If two skills could match, read both. They are designed to chain (e.g., ingest then enrich for each entity).

## Always-on (every message)

| Trigger | Skill |
|---------|-------|
| Every inbound message (spawn parallel, don't block) | `skills/signal-detector/SKILL.md` |
| Any brain read/write/lookup/citation | `skills/brain-ops/SKILL.md` |

## Brain operations

| Trigger | Skill |
|---------|-------|
| "What do we know about", "tell me about", "search for", "who is", "background on", "notes on" | `skills/query/SKILL.md` |
| "Who knows who", "relationship between", "connections", "graph query" | `skills/query/SKILL.md` (use graph-query) |
| Creating/enriching a goal, decision, process, or concept page | `skills/enrich/SKILL.md` |
| Where does a new file go? Filing rules | `skills/repo-architecture/SKILL.md` |
| Fix broken citations in brain pages | `skills/citation-fixer/SKILL.md` |
| "citation audit", "check citations", "fix citations" | `skills/citation-fixer/SKILL.md` (focused fix). For broader brain health, chain into `skills/maintain/SKILL.md` |
| "Research", "track", "extract from email", "investor updates", "donations" | `skills/data-research/SKILL.md` |
| Share a brain page as a link | `skills/publish/SKILL.md` |
| "validate frontmatter", "check frontmatter", "fix frontmatter", "frontmatter audit", "brain lint" | `skills/frontmatter-guard/SKILL.md` |

## Content & media ingestion

| Trigger | Skill |
|---------|-------|
| User shares a link, article, or idea | `skills/idea-ingest/SKILL.md` |
| "watch this video", "process this YouTube link", "ingest this PDF", "save this podcast", "process this book", "summarize this book", "PDF book", "ingest it into my brain", "what's in this screenshot", "check out this repo" | `skills/media-ingest/SKILL.md` |
| Meeting transcript received | `skills/meeting-ingestion/SKILL.md` |
| Generic "ingest this" (auto-routes to above) | `skills/ingest/SKILL.md` |

## Thinking skills (from GStack)

| Trigger | Skill |
|---------|-------|
| "Brainstorm", "I have an idea", "office hours" | GStack: office-hours |
| "Review this plan", "CEO review", "poke holes" | GStack: ceo-review |
| "Debug", "fix", "broken", "investigate" | GStack: investigate |
| "Retro", "what shipped", "retrospective" | GStack: retro |

> These skills come from GStack. If GStack is installed, the agent reads them directly.
> If not, brain-only mode still works (brain skills function without thinking skills).

## Operational

| Trigger | Skill |
|---------|-------|
| Task add/remove/complete/defer/review | `skills/daily-task-manager/SKILL.md` |
| Morning prep, meeting context, day planning | `skills/daily-task-prep/SKILL.md` |
| Daily briefing, "what's happening today" | `skills/briefing/SKILL.md` |
| Cron scheduling, quiet hours, job staggering | `skills/cron-scheduler/SKILL.md` |
| Save or load reports | `skills/reports/SKILL.md` |
| "Create a skill", "improve this skill" | `skills/skill-creator/SKILL.md` |
| "Skillify this", "is this a skill?", "make this proper" | `skills/skillify/SKILL.md` |
| "Compress my resolver", "AGENTS.md too large", "RESOLVER.md too big", "functional area dispatcher", "shrink routing table" | `skills/functional-area-resolver/SKILL.md` |
| "Is gbrain healthy?", morning health check, skillpack-check | `skills/skillpack-check/SKILL.md` |
| Post-restart health + auto-fix, smoke test | `skills/smoke-test/SKILL.md` |
| Cross-modal review, second opinion | `skills/cross-modal-review/SKILL.md` |
| "Validate skills", skill health check | `skills/testing/SKILL.md` |
| Webhook setup, external event processing | `skills/webhook-transforms/SKILL.md` |
| "Spawn agent", "background task", "parallel tasks", "steer agent", "pause/resume agent", "gbrain jobs submit", "submit a gbrain job", "submit a shell job", "shell job" | `skills/minion-orchestrator/SKILL.md` |
| "present options", "ask before proceeding", "choice gate", "user decision" | `skills/ask-user/SKILL.md` |

## Setup & migration

| Trigger | Skill |
|---------|-------|
| "Set up GBrain", first boot | `skills/setup/SKILL.md` |
| "Now what?", "fill my brain", "cold start", "bootstrap", "import my data", "what should I import first" | `skills/cold-start/SKILL.md` |
| "Migrate from Obsidian/Notion/Logseq" | `skills/migrate/SKILL.md` |
| Brain health check, maintenance run | `skills/maintain/SKILL.md` |
| "Extract links", "build link graph", "populate timeline" | `skills/maintain/SKILL.md` (extraction sections) |
| "Run dream", "process today's session", "synthesize my conversations", "consolidate yesterday's conversations", "what patterns did you see", "did the dream cycle run" | `skills/maintain/SKILL.md` (dream cycle section) |
| "Brain health", "what features am I missing", "brain score" | Run `gbrain features --json` |
| "Set up autopilot", "run brain maintenance", "keep brain updated" | Run `gbrain autopilot --install --repo ~/brain` |
| Agent identity, "who am I", customize agent | `skills/soul-audit/SKILL.md` |
| "Populate links", "extract links", "backfill graph" | `skills/maintain/SKILL.md` (graph population phase) |
| "Populate timeline", "extract timeline entries" | `skills/maintain/SKILL.md` (graph population phase) |

## Identity & access (always-on)

| Trigger | Skill |
|---------|-------|
| Non-owner sends a message | Check `ACCESS_POLICY.md` before responding |
| Agent needs to know its identity/vibe | Read `SOUL.md` |
| Agent needs user context | Read `USER.md` |
| Operational cadence (what to check and when) | Read `HEARTBEAT.md` |

## Disambiguation rules

When multiple skills could match:
1. Prefer the most specific skill (meeting-ingestion over ingest)
2. If the user mentions a URL, route by content type (link → idea-ingest, video → media-ingest)
3. If the user mentions a goal/decision/process/concept, check if enrich or query fits better
4. Chaining is explicit in each skill's Phases section
5. When in doubt, ask the user (see `skills/ask-user/SKILL.md` for the choice-gate pattern)

## Conventions (cross-cutting)

These apply to ALL brain-writing skills:
- `skills/conventions/quality.md` — citations, back-links, notability gate
- `skills/conventions/brain-first.md` — check brain before external APIs
- `skills/conventions/brain-routing.md` — which brain (DB) and which source (repo) to target; cross-brain federation is latent-space only
- `skills/conventions/subagent-routing.md` — when to use Minions vs inline work
- `skills/ask-user/SKILL.md` — choice-gate pattern for human input at decision points
- `skills/_brain-filing-rules.md` — where files go
- `skills/_output-rules.md` — output quality standards

## Uncategorized

| Trigger | Skill |
|---------|-------|
| "personalized version of this book", "mirror this book", "two-column book analysis", "apply this book to my life", "how does this book apply to me" | `skills/book-mirror/SKILL.md` |
| "enrich this article", "enrich brain pages", "batch enrich", "make brain pages useful" | `skills/article-enrichment/SKILL.md` |
| "strategic reading", "read this through the lens of", "apply this to my problem", "what can I learn from this about", "extract a playbook from" | `skills/strategic-reading/SKILL.md` |
| "concept synthesis", "synthesize my concepts", "find patterns across my notes", "build my intellectual map", "trace idea evolution" | `skills/concept-synthesis/SKILL.md` |
| "perplexity research", "what's new about", "current state of", "web research", "what changed about" | `skills/perplexity-research/SKILL.md` |
| "crawl my archive", "find gold in my archive", "archive crawler", "scan my dropbox for", "mine my old files for" | `skills/archive-crawler/SKILL.md` |
| "verify this academic claim", "check this study", "academic verify", "validate citation", "is this study real" | `skills/academic-verify/SKILL.md` |
| "make pdf from brain", "brain pdf", "convert brain page to pdf", "publish this page as pdf", "export brain page" | `skills/brain-pdf/SKILL.md` |
| "voice note", "ingest this voice memo", "transcribe and file", "voice note ingest", "save this audio note" | `skills/voice-note-ingest/SKILL.md` |
```

- [ ] **Step 2: Run resolver test**

Run: `bun test test/resolver.test.ts`
Expected: All tests pass. The resolver test checks that every trigger in RESOLVER.md matches a skill's frontmatter `triggers:` entry.

- [ ] **Step 3: Commit**

```bash
git add skills/RESOLVER.md
git commit -m "feat: rewrite RESOLVER.md routing table for developer domain"
```

---

## Task 9: Patch `brain-first.md` — retrieval conventions

**Files:**
- Modify: `skills/conventions/brain-first.md`

- [ ] **Step 1: Update the header (line 3)**

Replace:
```
**Read this before doing ANY entity/person/company/fact lookup.**
```

With:
```
**Read this before doing ANY entity/goal/decision/process/concept lookup.**
```

- [ ] **Step 2: Replace the entity page conventions table (lines 53-67)**

Replace the entire "Entity Page Conventions" section:

```markdown
## Entity Page Conventions

Standard directory structure:

| Directory | Type | Example |
|-----------|------|---------|
| `goals/` | goal | `goals/setup-jwt-auth.md` |
| `decisions/` | decision | `decisions/chose-postgres-over-sqlite.md` |
| `processes/` | process | `processes/deploy-to-production.md` |
| `concepts/` | concept | `concepts/event-sourcing.md` |

When creating new pages, include proper frontmatter with `type`, `title`,
and `tags` fields. See `skills/_brain-filing-rules.md` for page templates.
```

- [ ] **Step 3: Verify the file reads correctly**

Run: `cat skills/conventions/brain-first.md`
Expected: Developer entity table with goals/decisions/processes/concepts rows.

- [ ] **Step 4: Commit**

```bash
git add skills/conventions/brain-first.md
git commit -m "feat: update brain-first.md entity conventions for developer domain"
```

---

## Task 10: Full verification pass

**Files:**
- None modified — verification only

- [ ] **Step 1: Run typecheck**

Run: `bun run typecheck`
Expected: PASS

- [ ] **Step 2: Run full unit test suite**

Run: `bun run test > /tmp/customized_domain_tests.txt 2>&1; echo "EXIT=$?"; tail -50 /tmp/customized_domain_tests.txt`
Expected: All tests pass. Zero failures.

- [ ] **Step 3: Run the PageType consumer audit**

Run: `grep -rn 'PageType\|ALL_PAGE_TYPES' src/ --include='*.ts' | grep -v node_modules | grep -v 'import.*PageType'`

Review the output for any switch statements, whitelist arrays, or filter expressions that enumerate page types. The new types (`goal`, `decision`, `process`) must not be silently excluded by any existing filter. Key files to check:
- `src/core/facts/eligibility.ts` — `ELIGIBLE_TYPES` array. This is intentionally narrow (note/meeting/slack/email/calendar-event/source/writing). Developer types are NOT eligible for facts backstop, which is correct (goals/decisions/processes are structured pages, not conversation-shaped).
- `src/commands/doctor.ts` — `graph_coverage` check uses `type IN ('entity', 'person', 'company', 'organization')`. This is a Tier 2 change (not loop-breaking). Note it but don't block on it.

- [ ] **Step 4: Run skills conformance test**

Run: `bun test test/skills-conformance.test.ts`
Expected: All tests pass.

- [ ] **Step 5: Run filing-audit test**

Run: `bun test test/filing-audit.test.ts`
Expected: All tests pass.

- [ ] **Step 6: Run check-resolvable test**

Run: `bun test test/check-resolvable.test.ts`
Expected: All tests pass.

- [ ] **Step 7: Run resolver test**

Run: `bun test test/resolver.test.ts`
Expected: All tests pass.

- [ ] **Step 8: Spot-check the inferLinkType limitation**

Run: `grep -n 'inferLinkType' src/core/link-extraction.ts | head -5`

Note: `inferLinkType()` classifies developer entity relationships as `mentions` (the default fallback). This is a known v1 limitation per the spec. The function uses regex heuristics tuned for VC relationships (founded, invested_in, works_at, attended). Adding developer-specific heuristics (uses, decided_in, depends_on) is a Tier 2 follow-up.

---

## Task 11 (Tier 2, optional): Update doctor.ts graph_coverage check

**Files:**
- Modify: `src/commands/doctor.ts:1378`

This is a Tier 2 change — nice to have but not loop-breaking.

- [ ] **Step 1: Update the type filter in graph_coverage check**

In `src/commands/doctor.ts` line 1378, expand the SQL `type IN (...)` clause:

Replace:
```sql
SELECT COUNT(*)::int AS count FROM pages WHERE type IN ('entity', 'person', 'company', 'organization')
```

With:
```sql
SELECT COUNT(*)::int AS count FROM pages WHERE type IN ('entity', 'person', 'company', 'organization', 'goal', 'decision', 'process')
```

- [ ] **Step 2: Run doctor test if one exists**

Run: `bun test test/doctor.test.ts 2>/dev/null || echo "No doctor test file"`
Expected: Either passes or no test file exists.

- [ ] **Step 3: Commit**

```bash
git add src/commands/doctor.ts
git commit -m "feat: include developer types in doctor graph_coverage check"
```
