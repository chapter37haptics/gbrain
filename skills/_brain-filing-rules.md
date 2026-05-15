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
