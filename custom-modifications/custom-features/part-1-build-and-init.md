# Part 1: Build-Time Install + Runtime Init + Env Vars

**Why first:** Everything else depends on gbrain being installed and the database initialized. This is the foundation.

**Decisions covered:** D1 (GBRAIN_MODE), D3 (build vs runtime), D5 (artifacts off), D6 (transcripts off), D7 (auto-update skip)

**Files to modify:**
1. `practicespace-2/.devcontainer/Dockerfile` — add gbrain install layer
2. `practicespace-2/.devcontainer/docker-compose.dev.yml` — add env vars
3. `practicespace-2/.devcontainer/entrypoint.sh` — add GBRAIN_MODE gating + PGLite init

---

## 1. Dockerfile — Add gbrain install layer

Insert after the gstack clone block (line 79), before `ENTRYPOINT`.

```dockerfile
# GBrain — persistent knowledge base (build-time install, runtime init)
ARG GBRAIN_VERSION=v0.28.11
RUN git clone --depth 1 --branch ${GBRAIN_VERSION} https://github.com/garrytan/gbrain.git ~/gbrain \
    && cd ~/gbrain \
    && bun install --ignore-scripts \
    && chmod +x src/cli.ts \
    && bun link
```

**Why `--ignore-scripts`:** Issue #526 — Bun's built-in shell can't parse the postinstall script.
**Why `chmod +x`:** Issue #111 — `bun link` creates a symlink to `cli.ts` which lacks the executable bit on Linux. MCP invocation fails with `Permission denied`.

**Expected result:** `gbrain` binary on PATH at `~/.bun/bin/gbrain`.

**Insert point:** After the gstack block (line 79: `&& ./setup --no-prefix --host claude -q`), before `ENTRYPOINT ["/entrypoint.sh"]` (line 81).

---

## 2. docker-compose.dev.yml — Add env vars

Add these environment variables to the `dev` service:

```yaml
    environment:
      # --- existing vars stay ---
      - CLAUDE_CODE_ENABLE_TELEMETRY=1
      - CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
      - OTEL_EXPORTER_OTLP_PROTOCOL=grpc
      - OTEL_SERVICE_NAME=claude-code
      - OTEL_METRICS_EXPORTER=otlp
      - OTEL_LOGS_EXPORTER=otlp
      - OTEL_TRACES_EXPORTER=otlp
      # --- gbrain (new) ---
      - GBRAIN_MODE=off
      - GBRAIN_REPO=
      - GBRAIN_BRANCH=main
      - GBRAIN_MCP_TRUSTED=false
```

**`GBRAIN_MODE`** defaults to `off`. Set to `local` to enable.
**`GBRAIN_REPO`** is empty by default (no brain repo). Set to a GitHub URL to enable.
**`GBRAIN_BRANCH`** defaults to `main`.
**`GBRAIN_MCP_TRUSTED`** defaults to `false`. Set to `true` in devcontainers where you trust the agent.

---

## 3. entrypoint.sh — Add GBRAIN_MODE gating + PGLite init

Insert a new section **before** the final `exec "$@"` (line 40). The section is gated on `GBRAIN_MODE=local` — if `GBRAIN_MODE` is `off` or unset, it does nothing.

```bash
# ── GBrain deterministic init ────────────────────────────────────────
if [ "${GBRAIN_MODE:-off}" = "local" ]; then
    echo "[gbrain] mode=local — initializing PGLite brain..."

    # Clean up stale state from previous container crash (Issue #251)
    gbrain recover-wal 2>/dev/null || true
    pkill -f "gbrain serve" 2>/dev/null || true

    # Initialize PGLite database (zero network, zero prompts)
    if gbrain init --pglite --json > /tmp/gbrain-init.json 2>&1; then
        echo "[gbrain] PGLite initialized"
    else
        echo "[gbrain] ERROR: init failed" >&2
        cat /tmp/gbrain-init.json >&2
    fi
fi
```

**Insert point:** After the git identity block (line 38: `[ -n "$GIT_EMAIL" ] && git config --global user.email "$GIT_EMAIL"`), before `exec "$@"` (line 40).

**What this does:**
- Gates everything on `GBRAIN_MODE=local` (D1)
- Recovers from WAL corruption / stale locks (NI3, Issue #251)
- Kills orphaned `gbrain serve` processes (NI4, Issues #692/#591)
- Runs `gbrain init --pglite --json` — fully non-interactive, zero network (D3)
- Captures init output to `/tmp/gbrain-init.json` for debugging

**What this skips (by not calling them):**
- Artifacts sync prompt (D5) — never invoked
- Transcript ingest prompt (D6) — never invoked
- Auto-update check (D7) — never invoked

---

## Verification

After building the container:

```bash
# 1. With GBRAIN_MODE=off (default):
gbrain --version          # should work (binary on PATH)
ls ~/.gbrain/config.json  # should NOT exist (no init)

# 2. With GBRAIN_MODE=local:
gbrain --version          # should work
gbrain doctor --json      # should show healthy PGLite brain
ls ~/.gbrain/config.json  # should exist
ls ~/.gbrain/brain.pglite # should exist (PGLite data directory)
```

---

## What Part 2 builds on

Part 1 gives us: gbrain installed, PGLite initialized (if mode=local), env vars in place.

Part 2 adds: brain repo clone, session branch, sync, embed, MCP registration, CLAUDE.md generation.
