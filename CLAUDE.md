# CLAUDE.md

GBrain is a personal knowledge brain and GStack mod for agent platforms. Pluggable
engines: PGLite (embedded Postgres via WASM, zero-config default) or Postgres + pgvector
+ hybrid search in a managed Supabase instance. `gbrain init` defaults to PGLite;
suggests Supabase for 1000+ files. GStack teaches agents how to code. GBrain teaches
agents everything else: brain ops, signal detection, content ingestion, enrichment,
cron scheduling, reports, identity, and access control.

## Two organizational axes (read this first)

GBrain knowledge is organized along two orthogonal axes. Users AND agents must
understand both, or queries misroute silently.

- **Brain** — WHICH DATABASE. Your personal brain is `host`. You can mount
  additional brains (team-published, each with their own DB and access policy)
  via `gbrain mounts add` (v0.19+). Routing: `--brain`, `GBRAIN_BRAIN_ID`,
  `.gbrain-mount` dotfile.
- **Source** — WHICH REPO INSIDE THE DATABASE. A brain can hold many sources
  (wiki, gstack, openclaw, essays). Slugs scope per source. Routing:
  `--source`, `GBRAIN_SOURCE`, `.gbrain-source` dotfile.

Both axes follow the same 6-tier resolution pattern. Read
`docs/architecture/brains-and-sources.md` for topology diagrams (personal, team
mount, CEO-class with multiple team brains) and
`skills/conventions/brain-routing.md` for the agent-facing decision table.

## Architecture

Contract-first: `src/core/operations.ts` defines ~47 shared operations (v0.29 adds `get_recent_salience`, `find_anomalies`, `get_recent_transcripts`). CLI and MCP
server are both generated from this single source. Engine factory (`src/core/engine-factory.ts`)
dynamically imports the configured engine (`'pglite'` or `'postgres'`). Skills are fat
markdown files (tool-agnostic, work with both CLI and plugin contexts).

**Trust boundary:** `OperationContext.remote` distinguishes trusted local CLI callers
(`remote: false` set by `src/cli.ts`) from untrusted agent-facing callers
(`remote: true` set by `src/mcp/server.ts`). Security-sensitive operations like
`file_upload` tighten filesystem confinement when `remote=true` and default to
strict behavior when unset.

## Key files

> Only files relevant to the customized-domain spec are listed here.
> For the full key-files catalog, see the main CLAUDE.md.

- `src/core/operations.ts` — Contract-first operation definitions (the foundation). `put_page` enforces slug allow-lists when `viaSubagent` and `allowedSlugPrefixes` is set. Auto-link enabled for trusted-workspace writes (skipped only when `remote=true && !trustedWorkspace`).
- `src/core/engine.ts` — Pluggable engine interface (BrainEngine). `clampSearchLimit(limit, default, cap)` takes an explicit cap so per-operation caps can be tighter than `MAX_SEARCH_LIMIT`. Exports `LinkBatchInput` / `TimelineBatchInput` for the bulk-insert API (`addLinksBatch` / `addTimelineEntriesBatch`). `BrainEngine` has a `readonly kind: 'postgres' | 'pglite'` discriminator.
- `src/core/engine-factory.ts` — Engine factory with dynamic imports (`'pglite'` | `'postgres'`)
- `src/core/import-file.ts` — importFromFile + importFromContent (chunk + embed + tags)
- `src/core/sync.ts` — Pure sync functions (manifest parsing, filtering, slug conversion).
- `src/core/markdown.ts` — Frontmatter parsing + body splitter. `splitBody` requires an explicit timeline sentinel (`<!-- timeline -->`, `--- timeline ---`, or `---` immediately before `## Timeline`/`## History`). Plain `---` in body text is a markdown horizontal rule, not a separator. `inferType` auto-types `/wiki/analysis/` → analysis, `/wiki/guides/` → guide, `/wiki/hardware/` → hardware, `/wiki/architecture/` → architecture, `/writing/` → writing (plus the existing people/companies/deals/etc heuristics).
- `src/core/link-extraction.ts` — shared library for the v0.12.0 graph layer. extractEntityRefs (canonical, replaces backlinks.ts duplicate) matches both `[Name](people/slug)` markdown links and Obsidian `[[people/slug|Name]]` wikilinks as of v0.12.3. extractPageLinks, inferLinkType heuristics (attended/works_at/invested_in/founded/advises/source/mentions), parseTimelineEntries, isAutoLinkEnabled config helper. `DIR_PATTERN` covers `people`, `companies`, `deals`, `topics`, `concepts`, `projects`, `entities`, `tech`, `finance`, `personal`, `openclaw`. Used by extract.ts, operations.ts auto-link post-hook, and backlinks.ts.
- `src/core/check-resolvable.ts` — Resolver validation: reachability, MECE overlap, DRY checks, structured fix objects. v0.14.1: `CROSS_CUTTING_PATTERNS.conventions` is an array (notability gate accepts both `conventions/quality.md` and `_brain-filing-rules.md`). New `extractDelegationTargets()` parses `> **Convention:**`, `> **Filing rule:**`, and inline backtick references. DRY suppression is proximity-based via `DRY_PROXIMITY_LINES = 40`.
- `src/core/filing-audit.ts` + `skills/_brain-filing-rules.json` (v0.19) — Check 6 of `check-resolvable`. Parses new `writes_pages:` / `writes_to:` frontmatter on skills and audits their filing claims against the filing-rules JSON. Warning-only in v0.19, upgrades to error in v0.20.
- `src/commands/doctor.ts` — `gbrain doctor [--json] [--fast] [--fix] [--dry-run] [--index-audit]`: health checks. The `graph_coverage` check now short-circuits to `ok: 'No entity pages — graph_coverage not applicable (markdown-only brain)'` when `SELECT COUNT(*) FROM pages WHERE type IN ('entity','person','company','organization')` returns 0; the entity count is woven into the warn message. Relevant to this spec: the `type IN (...)` clause may need to include developer entity types.
- `scripts/check-jsonb-pattern.sh` — CI grep guard. Fails the build if anyone reintroduces (a) the `${JSON.stringify(x)}::jsonb` interpolation pattern (postgres.js v3 double-encodes it), or (b) `max_stalled INTEGER NOT NULL DEFAULT 1` in any schema source file. Wired into `bun test`.
- `src/core/cycle/synthesize.ts` (v0.23) — Synthesize phase: conversation-transcript-to-brain pipeline. Fans out subagents with `allowed_slug_prefixes` (sourced from `skills/_brain-filing-rules.json` `dream_synthesize_paths.globs`). Relevant: the globs array controls which directories the dream cycle can write to.
- `src/core/cycle/patterns.ts` (v0.23) — Patterns phase: cross-session theme detection. Same allow-list path as synthesize.
- `skills/RESOLVER.md` — Skill routing table (based on the agent-fork AGENTS.md pattern)
- `skills/conventions/` — Cross-cutting rules (quality, brain-first, model-routing, test-before-bulk, cross-modal). `skills/_brain-filing-rules.md` and `skills/_output-rules.md` are shared references.
- `skills/signal-detector/SKILL.md` — Always-on idea+entity capture on every message
- `skills/brain-ops/SKILL.md` — Brain-first lookup, read-enrich-write loop, source attribution

## Commands

Run `gbrain --help` or `gbrain --tools-json` for full command reference.

Key commands:
- `gbrain init` — defaults to PGLite (no Supabase needed), scans repo size, suggests Supabase for 1000+ files
- `gbrain sync` — rebuild DB from markdown repo
- `gbrain extract links|timeline|all` — batch link/timeline extraction
- `gbrain doctor` — health checks
- `gbrain search` / `gbrain query` — keyword / hybrid search

## Testing

### Test command tiers (v0.26.4 — parallel fast loop)

Five tiers of test commands, each with a clear scope:

| Command | What it runs | Wallclock | When to use |
|---|---|---|---|
| `bun run test` | Parallel unit-test fast loop. 8-shard fan-out via `scripts/run-unit-parallel.sh`, then a serial pass over `*.serial.test.ts`. Excludes `*.slow.test.ts` and `test/e2e/*`. No pre-checks, no typecheck. | ~85s on a Mac dev box (3650+ tests) | Inner edit loop. Default. |
| `bun run verify` | CI's authoritative pre-test gate set: `check:privacy && check:jsonb && check:progress && check:wasm && bun run typecheck`. The 4 checks `.github/workflows/test.yml` runs on shard 1 + typecheck. Single source of truth — CI literally calls `bun run verify`. | ~12s (wasm-compile dominates) | Before pushing; before `/ship`. |
| `bun run test:full` | `verify && bun run test && bun run test:slow && [smart e2e]`. The local equivalent of "everything CI runs." Smart e2e: runs e2e only when `DATABASE_URL` is set; else loud skip notice to stderr. | ~3-5min depending on slow + e2e | Pre-merge sanity, before opening a PR. |
| `bun run test:slow` | Just the `*.slow.test.ts` set (intentional cold-path correctness checks). | seconds-to-minutes | When touching slow-path code. |
| `bun run test:serial` | Just the `*.serial.test.ts` set (cross-file-contention quarantine; runs at `--max-concurrency=1`). | ~1s per quarantined file | Debugging a specific quarantined file. |
| `bun run test:e2e` | Real Postgres E2E. Requires Docker + `DATABASE_URL`. Sequential (template-DB parallelization is a v0.27+ TODO). | ~5-10min | Pre-ship; nightly. |
| `bun run check:all` | All 7 historical pre-checks (privacy + jsonb + progress + no-legacy-getconnection + trailing-newline + wasm + exports-count). Superset of `verify`. | ~10s | Local-only sweep. The 4 not in `verify` are nice-to-haves. |

### File taxonomy

- `*.test.ts` → fast loop (parallel 8-shard fan-out).
- `*.slow.test.ts` → run via `bun run test:slow` only (intentional cold-path tests; would dominate the fast loop's wallclock).
- `*.serial.test.ts` → run via `bun run test:serial` after the parallel pass completes; uses `--max-concurrency=1`. Quarantine for tests that share file-wide state and race when run alongside other files in the same `bun test` process.
- `test/e2e/*.test.ts` → real-Postgres E2E. Skipped when `DATABASE_URL` is unset.

### Key test files for this spec

These existing tests validate the code paths being changed:

- `test/markdown.test.ts` — frontmatter parsing, `inferType` heuristics
- `test/link-extraction.test.ts` — `extractEntityRefs`, `DIR_PATTERN`, `inferLinkType`
- `test/skills-conformance.test.ts` — skill frontmatter + required sections validation
- `test/resolver.test.ts` — RESOLVER.md coverage, routing validation (every trigger must match a frontmatter `triggers:` entry)
- `test/filing-audit.test.ts` — `writes_pages` / `writes_to` frontmatter, filing-rules JSON validation
- `test/check-resolvable.test.ts` — resolver reachability, MECE overlap, gap detection, DRY checks

### Pre-ship verification

Before shipping: `bun test && bun run test:e2e` (or `bun run ci:local` for the full gate).
After /ship: run /document-release; check README.md, CLAUDE.md, CHANGELOG.md, TODOS.md, docs/.

### E2E test DB lifecycle (condensed)

1. Check for `.env.testing` — copy from sibling worktree if missing
2. Start test DB: `docker run -d --name gbrain-test-pg -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=gbrain_test -p PORT:5432 pgvector/pgvector:pg16`
3. Bootstrap schema: `DATABASE_URL=postgresql://postgres:postgres@localhost:PORT/gbrain_test bun run src/cli.ts doctor --json > /dev/null 2>&1`
4. Run tests: `DATABASE_URL=postgresql://postgres:postgres@localhost:PORT/gbrain_test bun run test:e2e`
5. Tear down: `docker stop gbrain-test-pg && docker rm gbrain-test-pg`

### Spec-specific test rule

Do not modify or quarantine existing tests for this spec. If a test fails, fix the
product code — the spec is additive-only and existing tests should pass unchanged.

### Typecheck is mandatory

**Always run typecheck before pushing.** `bun test` (the bun runner)
skips TypeScript type checking — it only enforces runtime behavior.
Three ways to actually gate on types:

1. `bun run test` (npm script in `package.json`) — includes `bun run typecheck`
   plus the four shell pre-checks (`check-jsonb-pattern.sh`,
   `check-progress-to-stdout.sh`, `check-trailing-newline.sh`,
   `check-wasm-embedded.sh`) before the runner. Use this mid-branch.
2. `bun run typecheck` — `tsc --noEmit` standalone. Fast (~5s on this repo).
3. `bun run ci:local` — the full local CI gate from Path A.

The trap is: writing a new test, running `bun test test/foo.test.ts`,
seeing it pass, pushing — and CI's separate typecheck stage rejects an
invalid type literal that the runner accepted. Caught one of these
shipping the v0.23.2 round-trip E2E (`type: 'reflection'` is not a
member of `PageType`). Run `bun run typecheck` once before push, even
when only test files changed.

## Build

`bun build --compile --outfile bin/gbrain src/cli.ts`

## Skills

Read the skill files in `skills/` before doing brain operations. GBrain ships 29 skills
organized by `skills/RESOLVER.md` (`AGENTS.md` is also accepted as of v0.19):

**Original 8 (conformance-migrated):** ingest (thin router), query, maintain, enrich,
briefing, migrate, setup, publish.

**Brain skills (ported from an upstream agent fork):** signal-detector, brain-ops, idea-ingest, media-ingest,
meeting-ingestion, citation-fixer, repo-architecture, skill-creator, daily-task-manager.

**Operational + identity:** daily-task-prep, cross-modal-review, cron-scheduler, reports,
testing, soul-audit, webhook-transforms, data-research, minion-orchestrator.

**Conventions:** `skills/conventions/` has cross-cutting rules (quality, brain-first,
model-routing, test-before-bulk, cross-modal). `skills/_brain-filing-rules.md` and
`skills/_output-rules.md` are shared references.

## Capturing test output (NEVER pipe through `tail` / `head`)

**Iron rule:** when running `bun test`, `bun run test:e2e`, `bun run typecheck`,
or any other test/check command, redirect to a file FIRST, then `tail` the file
separately:

```bash
# RIGHT — full output preserved, real exit code visible
bun test > /tmp/ship_units.txt 2>&1
echo "EXIT=$?"
tail -50 /tmp/ship_units.txt
grep -E '(fail\)|✗|error:)' /tmp/ship_units.txt | head -30
```

```bash
# WRONG — exit code is `tail`'s (always 0), failures truncated, ship gates fail open
bun test 2>&1 | tail -10
```

## Skill routing

When the user's request matches an available skill, ALWAYS invoke it using the Skill
tool as your FIRST action. Do NOT answer directly, do NOT use other tools first.
The skill has specialized workflows that produce better results than ad-hoc answers.
