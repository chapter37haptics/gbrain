# Part 2: Brain Repo + Session Branches + MCP + CLAUDE.md

**Why second:** Requires Part 1 (gbrain installed, PGLite initialized). This is the main runtime workflow that makes gbrain usable by the agent.

**Decisions covered:** D2 (brain repo), D4 (session branches), D8 (CLAUDE.md), D9 (MCP via stdio), OQ4 (no CLI in CLAUDE.md), OQ5 (MCP registration verification)

**Files to modify:**
1. `practicespace-2/.devcontainer/entrypoint.sh` — extend the gbrain init block from Part 1

---

## 1. entrypoint.sh — Extend the GBRAIN_MODE=local block

Replace the Part 1 gbrain block with this expanded version. The Part 1 init logic stays; we add brain repo clone, sync, embed, MCP registration, and CLAUDE.md generation after it.

```bash
# ── GBrain deterministic init ────────────────────────────────────────
if [ "${GBRAIN_MODE:-off}" = "local" ]; then
    echo "[gbrain] mode=local — initializing PGLite brain..."

    # ── Step 1: Clean up stale state (from Part 1) ──
    gbrain recover-wal 2>/dev/null || true
    pkill -f "gbrain serve" 2>/dev/null || true

    # ── Step 2: Initialize PGLite database ──
    if gbrain init --pglite --json > /tmp/gbrain-init.json 2>&1; then
        echo "[gbrain] PGLite initialized"
    else
        echo "[gbrain] ERROR: init failed" >&2
        cat /tmp/gbrain-init.json >&2
    fi

    # ── Step 3: Clone brain repo + create session branch (D2, D4) ──
    BRAIN_DIR="/home/dev/brain"
    if [ -n "${GBRAIN_REPO:-}" ]; then
        if [ ! -d "$BRAIN_DIR/.git" ]; then
            echo "[gbrain] cloning brain repo: $GBRAIN_REPO"
            git clone --single-branch --branch "${GBRAIN_BRANCH:-main}" \
                "$GBRAIN_REPO" "$BRAIN_DIR" 2>&1 || {
                echo "[gbrain] ERROR: brain repo clone failed" >&2
            }
        else
            echo "[gbrain] brain repo already cloned, pulling latest..."
            git -C "$BRAIN_DIR" fetch origin 2>/dev/null || true
            git -C "$BRAIN_DIR" reset --hard "origin/${GBRAIN_BRANCH:-main}" 2>/dev/null || true
        fi

        # Create timestamped session branch
        if [ -d "$BRAIN_DIR/.git" ]; then
            SESSION_BRANCH="session/$(date +%Y-%m-%d_%H:%M)"
            git -C "$BRAIN_DIR" checkout -b "$SESSION_BRANCH" 2>/dev/null && \
                echo "[gbrain] session branch: $SESSION_BRANCH" || \
                echo "[gbrain] WARN: could not create session branch" >&2
        fi
    fi

    # ── Step 4: Sync brain repo into PGLite (D2) ──
    if [ -d "$BRAIN_DIR/.git" ]; then
        echo "[gbrain] syncing brain repo into PGLite..."
        if gbrain sync --repo "$BRAIN_DIR" --no-embed 2>&1; then
            echo "[gbrain] sync complete (keyword search ready)"
        else
            echo "[gbrain] WARN: sync failed (brain may be empty)" >&2
        fi
    fi

    # ── Step 5: Generate embeddings (requires OPENAI_API_KEY) ──
    # Must run BEFORE MCP registration — PGLite single-owner lock (Issue #677)
    # means gbrain serve will block all other gbrain CLI commands.
    if [ -n "${OPENAI_API_KEY:-}" ]; then
        echo "[gbrain] generating embeddings (this may take a minute)..."
        gbrain embed --stale 2>&1 || echo "[gbrain] WARN: embed failed" >&2
        echo "[gbrain] embeddings complete (semantic search ready)"
    else
        echo "[gbrain] OPENAI_API_KEY not set — skipping embeddings (keyword search only)"
    fi

    # ── Step 6: Register MCP server (D9) ──
    # gbrain serve will be spawned by Claude Code when it starts a session.
    # We only register the config here — the process starts later.
    GBRAIN_BIN="$(command -v gbrain)"
    if [ -n "$GBRAIN_BIN" ]; then
        claude mcp add --scope user gbrain -- "$GBRAIN_BIN" serve 2>/dev/null

        # Verify registration (OQ5)
        if claude mcp list 2>/dev/null | grep -q 'gbrain'; then
            echo "[gbrain] MCP registered: $GBRAIN_BIN serve"
            # Emit OTel log event if collector is reachable
            curl -s -X POST "${OTEL_EXPORTER_OTLP_ENDPOINT:-http://otel-collector:4317}/v1/logs" \
                -H "Content-Type: application/json" \
                -d '{"resourceLogs":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"gbrain-entrypoint"}}]},"scopeLogs":[{"logRecords":[{"body":{"stringValue":"MCP registration succeeded"},"severityText":"INFO"}]}]}]}' \
                2>/dev/null || true
        else
            echo "[gbrain] ERROR: MCP registration failed" >&2
            echo "FAILED" > /tmp/gbrain-mcp-status
            curl -s -X POST "${OTEL_EXPORTER_OTLP_ENDPOINT:-http://otel-collector:4317}/v1/logs" \
                -H "Content-Type: application/json" \
                -d '{"resourceLogs":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"gbrain-entrypoint"}}]},"scopeLogs":[{"logRecords":[{"body":{"stringValue":"MCP registration FAILED"},"severityText":"ERROR"}]}]}]}' \
                2>/dev/null || true
        fi
    else
        echo "[gbrain] ERROR: gbrain binary not found on PATH" >&2
    fi

    # ── Step 7: Generate CLAUDE.md config block (D8, OQ4) ──
    # MCP-only — no CLI examples. Sentinel prevents skills from appending CLI guidance.
    CLAUDEMD="/home/dev/.claude/CLAUDE.md"
    mkdir -p "$(dirname "$CLAUDEMD")"
    if [ ! -f "$CLAUDEMD" ] || ! grep -q 'gbrain-managed' "$CLAUDEMD" 2>/dev/null; then
        cat >> "$CLAUDEMD" << 'GBRAIN_BLOCK'

## GBrain Configuration
<!-- gbrain-managed: do not modify -->
- Engine: PGLite (local, container-ephemeral)
- Interface: MCP stdio (tools prefixed `mcp__gbrain__`)
- Use `mcp__gbrain__search` for keyword search, `mcp__gbrain__query` for semantic search
- Use `mcp__gbrain__get_page` / `mcp__gbrain__put_page` for read/write
- DO NOT use `gbrain` CLI commands directly — all operations go through MCP tools
GBRAIN_BLOCK

        # Add brain repo info if configured
        if [ -n "${GBRAIN_REPO:-}" ] && [ -n "${SESSION_BRANCH:-}" ]; then
            cat >> "$CLAUDEMD" << EOF
- Brain repo: /home/dev/brain (branch: ${SESSION_BRANCH})
- New knowledge written via \`mcp__gbrain__put_page\` is stored in PGLite
EOF
        fi

        echo "[gbrain] CLAUDE.md config block written"
    else
        echo "[gbrain] CLAUDE.md already has gbrain config block"
    fi

    echo "[gbrain] init complete"
fi
```

---

## Execution order rationale

The sequence is carefully ordered around the **PGLite single-owner lock** (Issue #677):

| Step | Uses PGLite? | Lock held? | Notes |
|------|-------------|------------|-------|
| 1. Cleanup | No | No | Kill stale processes, recover WAL |
| 2. Init | Yes | Briefly | Creates DB, releases when done |
| 3. Clone repo | No | No | Pure git operation |
| 4. Sync | Yes | Briefly | Imports markdown → PGLite, releases when done |
| 5. Embed | Yes | Briefly | Generates vectors, releases when done |
| 6. Register MCP | No | No | Only writes Claude Code config |
| 7. CLAUDE.md | No | No | Only writes a file |

Steps 2, 4, and 5 each acquire and release the PGLite lock. Step 6 only registers the MCP config — `gbrain serve` doesn't start until Claude Code opens a session, at which point it acquires the lock for the duration of the session.

---

## Verification

```bash
# With GBRAIN_MODE=local and GBRAIN_REPO set:

# 1. Brain repo cloned?
ls /home/dev/brain/.git                    # should exist

# 2. Session branch created?
git -C /home/dev/brain branch --show-current  # should be session/YYYY-MM-DD_HH:MM

# 3. Pages synced?
gbrain search "test" 2>/dev/null           # should return results (if brain has content)

# 4. MCP registered?
claude mcp list 2>/dev/null | grep gbrain  # should show gbrain

# 5. CLAUDE.md has config block?
grep "gbrain-managed" ~/.claude/CLAUDE.md  # should match

# 6. No CLI examples in CLAUDE.md?
grep "gbrain search" ~/.claude/CLAUDE.md   # should NOT match (only mcp__ references)
```

---

## What Part 3 builds on

Part 2 gives us: brain repo cloned, session branch created, content synced + embedded, MCP registered, CLAUDE.md configured.

Part 3 adds: trusted MCP mode (auto-link/timeline for agent-written pages) and OTel instrumentation of MCP dispatch for observability.
