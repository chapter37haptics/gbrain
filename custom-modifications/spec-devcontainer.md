# GBrain Deterministic Devcontainer Setup

## Goal

Make GBrain work automatically when a devcontainer starts, with no interactive prompts. Control knowledge accumulation via environment variables.

## Context

GBrain is a persistent knowledge base backed by Postgres + pgvector. It stores markdown pages, chunks them, embeds them with OpenAI, and exposes semantic search to AI coding agents via CLI and MCP.

The data flow is: **markdown files in git repo → `gbrain sync` → database (PGLite) → search/query**

- The **git repo** is the source of truth (markdown files, version-controlled)
- The **database** is a rebuildable search index (chunks + vector embeddings)
- PGLite is embedded Postgres via WASM — zero external dependencies, no network needed for init

## Decisions

### D1 — GBRAIN_MODE env var ✅

| Value | Behavior |
|-------|----------|
| `off` (default) | gbrain binary is on PATH but no brain initialized, no MCP registered |
| `local` | PGLite brain initialized at container start, MCP registered, knowledge accumulates in container-local database |

**Review status:** Solid. Thin-client mode work (PR #732, #772) validates our `local` PGLite-only approach. No changes needed.

### D2 — Brain repo (source of truth) ⚠️

A dedicated GitHub repo holds the brain's markdown files. Configured via:

- `GBRAIN_REPO` — GitHub repo URL (e.g., `https://github.com/user/brain.git`). Empty = no repo, empty brain.
- `GBRAIN_BRANCH` — base branch to clone from (default: `main`)

The repo is cloned to `/home/dev/brain` at container start.

**Constraint — use default source only:** Issue #891 reveals that `gbrain sync --source-id <name>` writes to `default` instead of the named source in v0.32. Do not use named sources. One brain repo, one default source.

**Constraint — sync-back gap:** Issue #344 confirms that `put_page`, `timeline-add`, and `link` are DB-only — they don't write back to markdown. PR #438 proposes `exportToBrainRepo()` with `GBRAIN_BRAIN_ROOT` env var to write pages back to `${BRAIN_ROOT}/<slug>.md` after `put_page`, but it hasn't merged. See D4 for the workaround.

### D3 — Install at build time, init at runtime ⚠️

- **Dockerfile (build time):** Clone gbrain, `bun install`, `bun link` — binary on PATH. No init, no MCP.
- **entrypoint.sh (runtime):** If `GBRAIN_MODE=local`, run `gbrain init --pglite --json`, clone brain repo (if configured), sync, register MCP.

Rationale: gbrain install is slow (~30s for bun install). Init needs runtime secrets (OPENAI_API_KEY for embeddings) and git credentials (GH_TOKEN for brain repo clone).

**Known install issues (from GitHub):**

| Issue | Problem | Mitigation |
|-------|---------|------------|
| #111 | `bun link` creates symlink to `cli.ts` which lacks executable bit — MCP invocation fails with `Permission denied` on Linux | Add `chmod +x src/cli.ts` after `bun install` |
| #526 | `bun install` fails because postinstall script is unparseable by Bun's shell | Use `bun install --ignore-scripts` then manual setup |
| #388 | Permission denied on gbrain binary after install | Same fix: `chmod +x src/cli.ts` |

**Dockerfile install block should be:**
```dockerfile
ARG GBRAIN_VERSION=v0.28.11
RUN git clone --depth 1 --branch ${GBRAIN_VERSION} https://github.com/garrytan/gbrain.git ~/gbrain \
    && cd ~/gbrain && bun install --ignore-scripts && chmod +x src/cli.ts && bun link
```

**Entrypoint runtime init should include WAL recovery:**

Issue #251 (stale PGLite lock after SIGKILL/OOM) + PR #564 (`gbrain recover-wal`): If a container crashes or is force-killed, PGLite's WAL can be corrupted and the lock file left behind. Run recovery before init:
```bash
gbrain recover-wal 2>/dev/null || true
gbrain init --pglite --json
```

### D4 — Session branches ⚠️

On container start, the entrypoint:

1. Clones `GBRAIN_REPO` at `GBRAIN_BRANCH` (e.g., `main`)
2. Creates a new branch: `session/YYYY-MM-DD_HH:MM` (timestamped)
3. Runs `gbrain sync --repo /home/dev/brain --no-embed` to import existing content into PGLite
4. All new knowledge the agent writes during this session gets committed to the session branch

This means:
- Every container starts with the full knowledge from `main`
- New knowledge accumulates on an isolated branch
- Good experiments: merge session branch into `main`
- Bad experiments: delete the branch, knowledge is gone
- Multiple containers can run simultaneously without conflicts (unique branch names)

**Critical constraint — PGLite single-owner lock (Issue #677):**

PGLite has an exclusive file lock. When `gbrain serve` (MCP server) is running, no other `gbrain` process can access the database. This means:
- `gbrain export` cannot run in a background loop alongside the MCP server
- `gbrain embed --stale` must run **before** the MCP server starts
- All maintenance must go through MCP tools or happen when MCP is not running

**Revised approach for step 4 (sync-back to git):**

Since background `gbrain export` is blocked by the PGLite lock, we have two options:

**Option A — Post-session export (simpler):** When the MCP server exits (container shutdown or session end), run `gbrain export --dir /home/dev/brain && cd /home/dev/brain && git add -A && git commit -m "session export" && git push origin HEAD`. This loses crash recovery but is simple.

**Option B — MCP-driven export (better):** The agent calls `mcp__gbrain__export` (if available) or we add a Claude Code hook (`user:stop`) that:
1. Kills the MCP server (releases PGLite lock)
2. Runs `gbrain export --dir /home/dev/brain`
3. Commits and pushes the session branch

**Option C — Adopt PR #438 (best, when available):** PR #438 adds `exportToBrainRepo()` that writes back to markdown after every `put_page`. This is real-time sync-back with no lock contention (runs inside the same process that holds the lock). Monitor this PR — if it merges, pin a version that includes it.

**Recommended:** Start with Option A (post-session), upgrade to Option C when PR #438 merges.

### D5 — Artifacts sync: off ✅

Artifacts are files that gstack skills produce (CEO plans, designs, reviews, retros, learnings) stored in `~/.gstack/`. The artifacts sync feature commits them to a private git repo for cross-machine persistence.

This is a gstack concern, not a gbrain concern. The brain repo (D2/D4) handles knowledge persistence independently. Set `artifacts_sync_mode=off` and skip the prompt.

**Review status:** Solid. No issues contradict this.

### D6 — Transcript ingest: off ✅

Transcript ingest imports past Claude Code/Codex conversation transcripts from `~/.claude/projects/` into gbrain as searchable pages. The devcontainer starts fresh with no prior transcripts. New session transcripts are ephemeral and die with the container. Knowledge persistence is handled by the brain repo + session branches (D4). Set `transcript_ingest_mode=off` and skip the prompt.

**Review status:** Solid. Fresh devcontainer = no transcripts to ingest.

### D7 — Auto-update check: skip ✅

GBrain version is pinned in the Dockerfile via `GBRAIN_VERSION` build arg. Container rebuilds handle updates. No need for runtime update checks. Skip the prompt entirely.

**Review status:** Solid. Issue #486 (`check-update --json returns no_releases`) reinforces this — the update check is broken anyway.

### D8 — CLAUDE.md config block: generate deterministically ⚠️

After successful init, write the `## GBrain Configuration` block to CLAUDE.md in the entrypoint with the known values (mode=local-stdio, engine=pglite, MCP registered, etc.). No prompt needed.

**Constraint — no CLI examples (Issue #835, OQ4):**

Issue #835 shows that even `gbrain doctor` recommends non-existent CLI commands. The setup skill and sync skill also inject CLI examples into CLAUDE.md. Since we use MCP as the sole interface (D9), CLI examples create ambiguity and waste agent attention.

**The entrypoint should write a minimal MCP-only block:**

```markdown
## GBrain Configuration
<!-- gbrain-managed: do not modify -->
- Engine: PGLite (local, container-ephemeral)
- Interface: MCP stdio (tools prefixed `mcp__gbrain__`)
- Use `mcp__gbrain__search` for keyword search, `mcp__gbrain__query` for semantic search
- Use `mcp__gbrain__get_page` / `mcp__gbrain__put_page` for read/write
- DO NOT use `gbrain` CLI commands — all operations go through MCP tools
- Brain repo: /home/dev/brain (session branch)
```

The `<!-- gbrain-managed: do not modify -->` sentinel prevents the setup/sync skills from appending CLI examples if they run later.

### D9 — Agent integration: MCP via stdio ⚠️

Use MCP stdio transport (`gbrain serve` spawned by Claude Code) as the sole integration method. Register in entrypoint before agent session starts via `claude mcp add --scope user gbrain -- /path/to/gbrain serve`. CLI is available as fallback but not the primary interface.

Rationale: MCP gives typed tools with schemas, structured responses, and automatic discovery. CLI requires the agent to know commands, construct bash strings, and parse text output.

**Known MCP issues (from GitHub):**

| Issue | Problem | Impact | Mitigation |
|-------|---------|--------|------------|
| #677 | PGLite single-owner lock — MCP server blocks all other `gbrain` processes | No background maintenance possible | Run `embed --stale` before MCP starts; all maintenance via MCP tools |
| #739 | PGLite cold start can timeout before stdio handshake completes | MCP server fails to connect | Pre-warm: run `gbrain init` before MCP registration so PGLite is already initialized |
| #833/#831 | `extract_facts` tool has invalid JSON schema (missing `items` on array) | OpenAI strict validators reject entire tool list | Doesn't affect Claude Code (lenient validator); fix upstream if needed |
| #713 | `whoami` returns `unknown_transport` over stdio | Diagnostic confusion | Don't rely on whoami for verification |
| #692/#591/#676 | Multiple fixes for stdio MCP cleanup on client disconnect | Stale PGLite locks after Claude Code exits | Add cleanup in entrypoint: kill orphaned `gbrain serve` processes on start |

**Entrypoint sequence (order matters due to PGLite lock):**
```bash
# 1. Clean up stale state
gbrain recover-wal 2>/dev/null || true
pkill -f "gbrain serve" 2>/dev/null || true

# 2. Init (creates PGLite database)
gbrain init --pglite --json

# 3. Sync brain repo (uses PGLite, releases lock when done)
gbrain sync --repo /home/dev/brain --no-embed

# 4. Generate embeddings (uses PGLite, releases lock when done)
gbrain embed --stale  # requires OPENAI_API_KEY

# 5. Register MCP (gbrain serve will acquire lock when Claude Code starts)
claude mcp add --scope user gbrain -- "$(command -v gbrain)" serve
```

Steps 2-4 run sequentially and release the PGLite lock between each. Step 5 only registers the config — `gbrain serve` doesn't start until Claude Code launches a session.

### D10 — Trusted MCP mode for devcontainers ⚠️

**Problem:** When the agent writes a page via `mcp__gbrain__put_page`, gbrain skips auto-link extraction and auto-timeline extraction because MCP callers are hardcoded as `remote: true` (untrusted). This is a security measure against search ranking manipulation — an untrusted caller influenced by prompt injection could plant slug references in page content (e.g., `see meetings/board-q1`), which auto-link would turn into graph edges, inflating the target page's ranking via the backlink boost in `hybridSearch`. The attack chain: prompt injection → agent writes planted references → auto-link creates links → backlink boost promotes attacker-chosen pages in future search results.

**Why this matters for us:** In a devcontainer, we control the agent and its inputs. There is no untrusted caller — it's our agent writing to our brain. Without auto-link and auto-timeline, agent-written pages don't participate in the knowledge graph. They're searchable but isolated — no typed relationships, no timeline entries, no backlink boost. This undermines the value of accumulating knowledge.

**Solution — simple env var (our approach):**
```typescript
// line 36 in src/mcp/server.ts
remote: process.env.GBRAIN_MCP_TRUSTED === 'true' ? false : true,
```

Security stays on by default. Devcontainers opt in by setting `GBRAIN_MCP_TRUSTED=true` in docker-compose.dev.yml. This gives agent-written pages the same auto-link and auto-timeline behavior as CLI-written pages.

**Alternative — PR #438's nuanced approach (better if it merges):**

PR #438 proposes splitting auto-link into `full` (local callers) vs `frontmatter_only` (remote callers), and always allowing timeline extraction for remote writes. This is architecturally better because:
- Timeline extraction is safe (entries attach to the current page only, no ranking manipulation)
- Frontmatter-only link extraction is safe (structured data, not regex on free text)
- Only body-regex link extraction is the attack vector

If PR #438 merges, adopt its approach instead of the binary trusted/untrusted flag.

**Additional context — Issue #160:** Confirms the security concern is real. `extractAndEnrich` writes pages from untrusted text without any gate (regex extracts capitalized names, creates `people/` and `companies/` pages automatically). This validates the `remote: true` default.

## Answered Open Questions

### OQ1 — Skills mix deterministic triggers with LLM-dependent work

**Answer: Don't build `gbrain skill-split`. Use Claude Code hooks for triggers, compress skills for judgment.**

Skills are evolving rapidly (224 routing targets as of PR #804). Building a tool to parse and split skills would be fragile and outdated immediately.

**Approach:**
1. **Claude Code hooks for deterministic triggers:**
   - "Run signal-detector on every message" → `user:afterToolCall` hook that calls `mcp__gbrain__extract_facts`
   - "Brain-first lookup" → `user:beforeToolCall` hook that auto-queries gbrain before web searches
2. **Compress skills for LLM judgment parts:**
   - PR #859 demonstrates 50% reduction (25KB → 13KB) through structured routing tables
   - PR #884 proposes dense structural context files (~5KB) for agent orientation
   - Apply same compression to remaining skill content
3. **Keep skills focused on what actually needs LLM judgment** — entity extraction, page writing decisions, enrichment quality

### OQ2 — Convention compliance observability

**Answer: Instrument `src/mcp/dispatch.ts` with OTel spans. Emit convention-check events after `put_page`.**

PR #586 (v0.26.3) already adds observability infrastructure. Build on it.

**Implementation:**
```typescript
// In src/mcp/dispatch.ts
import { trace, SpanStatusCode } from '@opentelemetry/api'
const tracer = trace.getTracer('gbrain-mcp')

// Wrap each dispatch in a span
const span = tracer.startSpan(`gbrain.mcp.${operationName}`)
span.setAttribute('gbrain.operation', operationName)
span.setAttribute('gbrain.caller', ctx.remote ? 'mcp' : 'cli')
```

**Convention-specific checks after `put_page`:**
- `gbrain.convention.quality` — does the page have `source:` in frontmatter?
- `gbrain.convention.backlinks` — did auto-link fire and create links?
- `gbrain.convention.brain_first` — was `search`/`query` called before `put_page` in this session?

The brain-first check requires correlating across tool calls via trace context. Simplest approximation: track per-session whether search was called before put_page, emit a warning span if not.

Spans flow through existing OTel Collector → Jaeger + Phoenix pipeline.

**Prerequisite:** `@opentelemetry/api` as a gbrain dependency.

### OQ3 — Context engineering for convention/skill adherence

**Answer: Measure first (OQ2), then compress, then relocate.**

1. **Measure first** (depends on OQ2): Use OTel traces to identify which conventions get followed vs dropped. Don't optimize blind.

2. **Compress to structured checklists:**
   ```markdown
   ## Quality Convention (checklist)
   Before writing any brain page:
   - [ ] Page has `source:` frontmatter with URL or reference
   - [ ] Back-links created for any referenced slugs
   - [ ] Content passes notability gate (not ephemeral/trivial)
   ```
   Checklists get followed more reliably than prose. PR #859 demonstrates 50% size reduction through structured format.

3. **Relocate critical conventions to tool descriptions:** The MCP tool schema for `put_page` could include convention requirements in its `description` field. Rules at the point of action > rules in a separate document.

4. **Feedback loop:** Low-adherence conventions (measured via OQ2) get rewritten and re-measured. High-adherence conventions are left alone.

5. **Escape hatch:** Conventions that consistently fail LLM compliance should become deterministic validators (hooks or post-write checks), not instructions. Relates to OQ1.

### OQ4 — Prevent CLI command injection into CLAUDE.md

**Answer: Generate the CLAUDE.md block ourselves in the entrypoint. Don't run the setup skill.**

Since D8 says "generate deterministically," we control what goes into CLAUDE.md. The entrypoint writes a minimal MCP-only block (see D8 above). Key elements:

1. **No CLI examples at all** — only reference `mcp__gbrain__*` tool names
2. **Sentinel comment** (`<!-- gbrain-managed: do not modify -->`) prevents the setup/sync skills from appending CLI examples later
3. **MCP tool schemas are self-documenting** — they have descriptions in the JSON schema, so the agent doesn't need separate guidance

If the `/setup-gbrain` or `/sync-gbrain` gstack skill runs during a session, it will see the sentinel and skip the CLAUDE.md modification. If it doesn't respect the sentinel, the worst case is duplicated/conflicting guidance — the "DO NOT use CLI" instruction in our block takes precedence.

### OQ5 — Track MCP registration success/failure on session start

**Answer: Verify via `claude mcp list` + emit OTel log event. Don't fall back, don't retry.**

**Entrypoint verification:**
```bash
# Register
claude mcp add --scope user gbrain -- "$(command -v gbrain)" serve

# Verify registration persisted (not just exit code 0)
if claude mcp list 2>/dev/null | grep -q 'gbrain'; then
    echo "gbrain MCP registered"
    # Emit OTel log event via HTTP (otel-cli not needed)
    curl -s -X POST "http://otel-collector:4318/v1/logs" \
        -H "Content-Type: application/json" \
        -d '{"resourceLogs":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"gbrain-entrypoint"}}]},"scopeLogs":[{"logRecords":[{"body":{"stringValue":"MCP registration succeeded"},"severityText":"INFO"}]}]}]}' \
        2>/dev/null || true
else
    echo "ERROR: gbrain MCP registration failed" >&2
    curl -s -X POST "http://otel-collector:4318/v1/logs" \
        -H "Content-Type: application/json" \
        -d '{"resourceLogs":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"gbrain-entrypoint"}}]},"scopeLogs":[{"logRecords":[{"body":{"stringValue":"MCP registration FAILED"},"severityText":"ERROR"}]}]}]}' \
        2>/dev/null || true
    echo "FAILED" > /tmp/gbrain-mcp-status
fi
```

**On failure:** Don't fall back to CLI mode (defeats D9). Don't retry (likely a config issue, not transient). Log the error and let the container start. The agent will see missing `mcp__gbrain__*` tools and can diagnose.

**Note:** Issue #713 (whoami returns unknown_transport over stdio) means we can't use whoami to verify the live connection. `claude mcp list` confirms the config is persisted, which is sufficient — the actual connection happens when Claude Code starts a session.

### OQ6 — Track MCP tool failures during a session

**Answer: Wrap `dispatchOperation()` in `src/mcp/dispatch.ts` with OTel spans.**

Same instrumentation point as OQ2, focused on operational health:

```typescript
// In src/mcp/dispatch.ts
import { trace, SpanStatusCode } from '@opentelemetry/api'
const tracer = trace.getTracer('gbrain-mcp')

export async function dispatchOperation(name: string, args: any, opts: OperationOpts) {
    const span = tracer.startSpan(`gbrain.mcp.${name}`)
    span.setAttribute('gbrain.operation', name)
    span.setAttribute('gbrain.remote', opts.remote ?? true)
    try {
        const result = await originalDispatch(name, args, opts)
        span.setStatus({ code: SpanStatusCode.OK })
        return result
    } catch (err) {
        span.setStatus({ code: SpanStatusCode.ERROR, message: err.message })
        span.recordException(err)
        throw err
    } finally {
        span.end()
    }
}
```

**What this gives us:**
- Every `mcp__gbrain__*` call as a trace in Jaeger
- Success/failure status per call
- Latency measurements
- Error details (PGLite lock contention, embedding API down, corrupt database, etc.)
- Correlation with Claude Code's own OTel traces via propagated trace context

**Prerequisite:** Add `@opentelemetry/api` to gbrain's `package.json`. The OTel collector endpoint (`OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317`) is already configured in docker-compose.dev.yml.

## New Issues Discovered During Review

### NI1 — #890 regression: sync hard-errors without OPENAI_API_KEY

In v0.32, `gbrain sync` hard-errors and imports zero pages when `OPENAI_API_KEY` is not set. In v0.18 it continued with per-chunk warnings. Our entrypoint uses `gbrain sync --repo /home/dev/brain --no-embed`, which should skip embedding, but the `--no-embed` flag may not be sufficient in v0.32.

**Impact:** If we upgrade past v0.28.11, sync may fail without OPENAI_API_KEY even with `--no-embed`.

**Mitigation:** Pin to v0.28.11 for now. Before upgrading, verify that `--no-embed` bypasses the API key requirement. If not, set `OPENAI_API_KEY` before sync and let embed happen inline, or wait for the upstream fix.

### NI2 — PGLite lock prevents background maintenance (#677)

The MCP server holds an exclusive PGLite lock. No other `gbrain` process can run while it's active. This constrains:
- No background `gbrain embed --stale` alongside MCP
- No background `gbrain export` for session branch commits (affects D4)
- No `gbrain doctor` diagnostics during a session

**Mitigation:** Run all maintenance (sync, embed, export) before MCP starts or after it exits. During a session, all operations must go through MCP tools.

### NI3 — Stale PGLite locks after container crash (#251)

If a container is force-killed (SIGKILL, OOM), PGLite's lock directory and WAL can be corrupted. The next container start will fail to acquire the lock.

**Mitigation:** Run `gbrain recover-wal` and kill orphaned `gbrain serve` processes at the start of the entrypoint, before `gbrain init`.

### NI4 — MCP stdio cleanup on disconnect (#692, #591, #676)

Multiple PRs fix the MCP stdio server not cleaning up when the client (Claude Code) disconnects. Without these fixes, the `gbrain serve` process lingers with the PGLite lock held, preventing subsequent commands.

**Mitigation:** The entrypoint already kills orphaned `gbrain serve` on startup (NI3). For in-session cleanup, PR #692 (merged in v0.28.2+) and PR #676 should handle it. Verify these are included in our pinned version.

### NI5 — Ranking results inverted (#895)

`gbrain query` returns results in inverted order — least relevant first. This is a v0.31.3 bug. Our v0.28.11 may not be affected, but worth testing.

**Mitigation:** Test `gbrain query` on our version. If affected, this is a search quality issue that undermines the value of the brain.
