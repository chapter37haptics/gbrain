# How GBrain Works

## What GBrain is

A persistent knowledge base for AI coding agents. It takes markdown files (and code), indexes them into a searchable database with vector embeddings, and exposes search/query tools so an agent can look things up semantically instead of grepping.

## Components

### 1. Database engine (PGLite or Postgres)

The storage layer. Holds pages, chunks, embeddings, links, tags, timeline entries.

- **PGLite**: Embedded Postgres running via WASM inside the Bun process. Zero external dependencies, no network. File lives at `~/.gbrain/brain.pglite`. Single-process exclusive lock.
- **Postgres** (Supabase): Managed cloud Postgres with pgvector extension. Multi-machine access, connection pooling.

Both engines implement the same `BrainEngine` interface (`src/core/engine.ts`). The engine factory (`src/core/engine-factory.ts`) dynamically imports the right one based on config.

Config lives at `~/.gbrain/config.json` (mode 0600). Respects `GBRAIN_HOME` env var for relocation.

### 2. CLI (`gbrain`)

The primary interface. Entry point: `src/cli.ts`. Installed via `git clone + bun install + bun link`, which creates a global `gbrain` symlink pointing at `src/cli.ts`.

Key commands:
- `gbrain init --pglite` / `gbrain init --non-interactive --url <pg_url>` — create the database
- `gbrain import <dir>` — bulk import markdown files into the database
- `gbrain sync --repo <path>` — incremental git-based sync (diff since last commit)
- `gbrain search <query>` — keyword + vector hybrid search
- `gbrain query <question>` — semantic search with query expansion
- `gbrain get <slug>` / `gbrain put <slug>` — read/write individual pages
- `gbrain export --dir <path>` — dump all pages as markdown files
- `gbrain doctor --json` — health check
- `gbrain embed --stale` — generate missing vector embeddings
- `gbrain extract links --source db` — backfill typed links between pages
- `gbrain serve` — start MCP server (stdio transport)

### 3. MCP server

Exposes gbrain operations as tools callable by Claude Code (or any MCP-compatible agent). Two transports:

- **stdio** (`gbrain serve`): Local pipe, registered via `claude mcp add --scope user gbrain -- /path/to/gbrain serve`. Claude Code spawns it as a subprocess.
- **HTTP** (`gbrain serve --http`): Remote access with OAuth 2.1 + bearer tokens. For cross-machine setups.

The MCP server exposes 51 operations defined in `src/core/operations.ts`. Both CLI and MCP are generated from this single source of truth. The MCP server marks all callers as `remote=true` (untrusted), which tightens filesystem confinement on operations like `file_upload`.

**Token cost:** All 51 tool definitions total ~2,500 tokens (~10K chars of JSON schemas). This is roughly 1-2% of a typical context window — negligible.

### 4. Skills (markdown files)

Fat markdown files in `skills/` that teach agents how to use gbrain. They are NOT code — they're instructions that an LLM reads and follows. The skill resolver (`skills/RESOLVER.md`) routes incoming requests to the right skill.

Key skills:
- **signal-detector** — fires on every inbound message, captures entities in parallel
- **brain-ops** — brain-first lookup on every response
- **query** — semantic search, graph queries
- **enrich** — create/update person and company pages
- **ingest** / **media-ingest** / **meeting-ingestion** — content ingestion
- **setup** — interactive setup wizard
- **maintain** — brain health, link extraction, dream cycle

### 5. Conventions (cross-cutting rules)

Files in `skills/conventions/` that apply to ALL brain-writing operations:
- **quality.md** — citations, back-links, notability gate
- **brain-first.md** — check brain before external APIs
- **brain-routing.md** — which brain (DB) and which source (repo) to target

## Data Flow

```
Markdown files in git repo
        │
        ▼
   gbrain sync (git diff → detect changes)
        │
        ├──▶ Parse markdown (frontmatter + body)          [deterministic]
        ├──▶ Chunk text (recursive splitter / AST-aware)   [deterministic]
        ├──▶ Generate embeddings (OpenAI API call)         [API call, deterministic per input]
        ├──▶ Extract typed links between pages             [deterministic]
        ├──▶ Extract timeline entries from dates            [deterministic]
        │
        ▼
   PGLite / Postgres (pages + chunks + vectors + links)
        │
        ▼
   Search / Query (keyword + vector hybrid)
        │
        ├──▶ Keyword search (pg_trgm trigram matching)     [deterministic]
        ├──▶ Vector search (pgvector cosine similarity)    [deterministic given embeddings]
        ├──▶ Query expansion (LLM rewrites query)          [LLM-dependent]
        ├──▶ Result ranking and dedup                      [deterministic]
        │
        ▼
   Agent reads results, writes new pages back
```

## Deterministic vs LLM-Dependent

### Fully deterministic (no LLM needed)

| Component | What it does |
|-----------|-------------|
| `gbrain init` | Creates database, runs schema migrations |
| `gbrain sync` | Git diff → detect changed files → import |
| Markdown parsing | Frontmatter extraction, body parsing (uses `marked` library) |
| Text chunking | Recursive text splitter for markdown; tree-sitter AST-aware chunker for code |
| Link extraction | Regex + frontmatter-based detection of `[[wikilinks]]`, YAML refs, slug references |
| Timeline extraction | Date pattern detection from page content |
| Keyword search | PostgreSQL `pg_trgm` trigram matching — pure SQL |
| Vector search | pgvector cosine similarity — pure SQL (given pre-computed embeddings) |
| `gbrain export` | Dumps pages back to markdown files |
| `gbrain doctor` | Health checks (connection, schema version, embed coverage) |
| `gbrain migrate` | Copies all data between engines (PGLite ↔ Postgres) |
| MCP server | Starts stdio/HTTP server, dispatches tool calls |

### Requires API calls (deterministic per input, but needs network + API key)

| Component | What it does | API |
|-----------|-------------|-----|
| Embedding generation | Converts text chunks to 1536-dim vectors | OpenAI `text-embedding-3-large` |
| `gbrain embed --stale` | Backfills missing embeddings | OpenAI |

### Requires LLM judgment (non-deterministic)

| Component | What it does | When it fires |
|-----------|-------------|---------------|
| Query expansion | Rewrites a search query into 2-3 variant queries for better recall | `gbrain query` (not `gbrain search`) |
| Signal detector skill | Identifies people, companies, concepts in a message | Every inbound message (if skill is active) |
| Enrich skill | Creates/updates brain pages with researched information | When agent encounters a new entity |
| Dream cycle | Overnight synthesis: cross-session patterns, entity consolidation | Cron job (nightly) |
| All skills in `skills/` | Skills are LLM instructions — they require an agent to interpret and execute them | When the agent routes to a skill |

### Key insight for devcontainer setup

The core pipeline — **init → sync → chunk → embed → search** — is almost entirely deterministic. The only API call is embedding generation (OpenAI). Everything else is local computation.

LLM judgment is only needed for:
1. **Query expansion** (optional — `gbrain search` skips it, only `gbrain query` uses it)
2. **Skills** (the agent reading skill files and acting on them — this is the agent itself, not gbrain)

This means gbrain can be fully initialized and populated with zero LLM calls if you skip embeddings (`--no-embed`). Keyword search works immediately. Semantic search works after `gbrain embed --stale` runs (which calls OpenAI).

## Installation Summary

```
# 1. Clone and link (build time — Dockerfile)
git clone --depth 1 https://github.com/garrytan/gbrain.git ~/gbrain
cd ~/gbrain && bun install && bun link
# Result: `gbrain` binary on PATH

# 2. Initialize database (runtime — entrypoint)
gbrain init --pglite --json
# Result: ~/.gbrain/config.json + ~/.gbrain/brain.pglite

# 3. Import content (runtime — entrypoint)
gbrain sync --repo /path/to/brain-repo --no-embed
# Result: pages + chunks in PGLite (keyword-searchable)

# 3b. Create session branch (runtime — entrypoint)
git -C /path/to/brain-repo checkout -b session/$(date +%Y-%m-%d_%H:%M)
# Result: isolated branch for this container's knowledge writes

# 4. Generate embeddings (runtime — background, optional)
gbrain embed --stale
# Result: vector embeddings in PGLite (semantic-searchable)
# Requires: OPENAI_API_KEY

# 5. Register MCP (runtime — entrypoint)
claude mcp add --scope user gbrain -- $(command -v gbrain) serve
# Result: Claude Code can call mcp__gbrain__search, mcp__gbrain__query, etc.
```

## Integration with Claude Code

Claude Code interacts with gbrain through two channels:

1. **MCP tools** (`mcp__gbrain__*`): Direct programmatic access to search, get, put, query, etc. These are the 41 operations from `src/core/operations.ts`, exposed via the MCP server. Loaded at Claude Code session start.

2. **Skills** (via gstack): Markdown instruction files that tell the agent HOW to use gbrain effectively. The skill resolver routes tasks to the right skill. Skills call gbrain CLI commands or MCP tools as part of their workflow. This is where LLM judgment happens — the agent reads the skill and decides what to do.

The brain-first lookup protocol (injected into CLAUDE.md or AGENTS.md) establishes the default behavior:
1. `gbrain search "name"` — keyword match, fast
2. `gbrain query "question"` — hybrid search with expansion
3. `gbrain get <slug>` — direct page read
4. `grep` fallback — only if gbrain returns zero results
