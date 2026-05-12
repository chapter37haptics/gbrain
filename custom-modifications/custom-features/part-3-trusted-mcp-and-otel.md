# Part 3: Trusted MCP Mode + OTel Instrumentation

**Why third:** Parts 1-2 handle devcontainer infra. This part modifies gbrain source code to improve agent-written page quality (D10) and add observability (OQ2, OQ6). Independent of the devcontainer scripts but unlocks full knowledge graph participation and debugging.

**Decisions covered:** D10 (trusted MCP mode), OQ2 (convention compliance observability), OQ6 (MCP tool failure tracking)

**Files to modify (in gbrain repo):**
1. `src/mcp/server.ts` — check GBRAIN_MCP_TRUSTED env var
2. `src/mcp/dispatch.ts` — wrap dispatchToolCall in OTel spans
3. `package.json` — add @opentelemetry/api dependency

---

## 1. src/mcp/server.ts — Trusted MCP mode (D10)

**Current code (line 35-38):**
```typescript
    return dispatchToolCall(engine, name, params, {
      remote: true,
      takesHoldersAllowList: ['world'],
    });
```

**Change to:**
```typescript
    const trusted = process.env.GBRAIN_MCP_TRUSTED === 'true';
    return dispatchToolCall(engine, name, params, {
      remote: !trusted,
      takesHoldersAllowList: trusted ? undefined : ['world'],
    });
```

**What this does:**
- When `GBRAIN_MCP_TRUSTED=true` (set in docker-compose.dev.yml): `remote=false`, no takes filter. Agent-written pages get auto-link extraction, auto-timeline extraction, and full takes visibility — same as CLI callers.
- When unset or `false` (default): `remote=true`, takes filtered to `['world']`. Current security behavior preserved.

**Why:** In a devcontainer, we control the agent. Without this, agent-written pages via `mcp__gbrain__put_page` don't participate in the knowledge graph (no typed links, no timeline entries, no backlink boost). See D10 in spec for the full security analysis.

**Future:** If PR #438 merges (splits auto-link into `full` vs `frontmatter_only` mode), adopt that finer-grained approach instead. But the env var is the right short-term fix.

---

## 2. src/mcp/dispatch.ts — OTel instrumentation (OQ2, OQ6)

**Add import at top of file (after existing imports, line 12):**
```typescript
import { trace, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('gbrain-mcp', '0.28.11');
```

**Wrap the `dispatchToolCall` function (lines 175-207).**

Current:
```typescript
export async function dispatchToolCall(
  engine: BrainEngine,
  name: string,
  params: Record<string, unknown> | undefined,
  opts: DispatchOpts = {},
): Promise<ToolResult> {
  const op = operations.find(o => o.name === name);
  if (!op) {
    return { content: [{ type: 'text', text: `Error: Unknown tool: ${name}` }], isError: true };
  }

  const safeParams = params || {};
  const validationError = validateParams(op, safeParams);
  if (validationError) {
    return {
      content: [{ type: 'text', text: JSON.stringify({ error: 'invalid_params', message: validationError }, null, 2) }],
      isError: true,
    };
  }

  const ctx = buildOperationContext(engine, safeParams, opts);

  try {
    const result = await op.handler(ctx, safeParams);
    return { content: [{ type: 'text', text: JSON.stringify(result, null, 2) }] };
  } catch (e: unknown) {
    if (e instanceof OperationError) {
      return { content: [{ type: 'text', text: JSON.stringify(e.toJSON(), null, 2) }], isError: true };
    }
    const msg = e instanceof Error ? e.message : String(e);
    return { content: [{ type: 'text', text: `Error: ${msg}` }], isError: true };
  }
}
```

Replace with:
```typescript
export async function dispatchToolCall(
  engine: BrainEngine,
  name: string,
  params: Record<string, unknown> | undefined,
  opts: DispatchOpts = {},
): Promise<ToolResult> {
  const op = operations.find(o => o.name === name);
  if (!op) {
    return { content: [{ type: 'text', text: `Error: Unknown tool: ${name}` }], isError: true };
  }

  const safeParams = params || {};
  const validationError = validateParams(op, safeParams);
  if (validationError) {
    return {
      content: [{ type: 'text', text: JSON.stringify({ error: 'invalid_params', message: validationError }, null, 2) }],
      isError: true,
    };
  }

  const ctx = buildOperationContext(engine, safeParams, opts);

  const span = tracer.startSpan(`gbrain.mcp.${name}`);
  span.setAttribute('gbrain.operation', name);
  span.setAttribute('gbrain.remote', opts.remote ?? true);

  try {
    const result = await op.handler(ctx, safeParams);

    // Convention compliance checks for write operations (OQ2)
    if (name === 'put_page' && typeof result === 'object' && result !== null) {
      const r = result as Record<string, unknown>;
      span.setAttribute('gbrain.convention.auto_link', r.auto_link !== 'skipped');
      span.setAttribute('gbrain.convention.auto_timeline', r.auto_timeline !== 'skipped');
      if (safeParams.body && typeof safeParams.body === 'string') {
        const hasSource = /^source:/m.test(safeParams.body) || /\nsource:/m.test(safeParams.body);
        span.setAttribute('gbrain.convention.has_source_citation', hasSource);
      }
    }

    span.setStatus({ code: SpanStatusCode.OK });
    return { content: [{ type: 'text', text: JSON.stringify(result, null, 2) }] };
  } catch (e: unknown) {
    span.setStatus({ code: SpanStatusCode.ERROR, message: e instanceof Error ? e.message : String(e) });
    span.recordException(e instanceof Error ? e : new Error(String(e)));

    if (e instanceof OperationError) {
      return { content: [{ type: 'text', text: JSON.stringify(e.toJSON(), null, 2) }], isError: true };
    }
    const msg = e instanceof Error ? e.message : String(e);
    return { content: [{ type: 'text', text: `Error: ${msg}` }], isError: true };
  } finally {
    span.end();
  }
}
```

**What this adds:**
- Every MCP tool call produces an OTel span with operation name, remote flag, and latency
- Errors are recorded as span exceptions with status ERROR
- `put_page` calls get convention compliance attributes:
  - `gbrain.convention.auto_link` — did auto-link run? (false if remote=true and GBRAIN_MCP_TRUSTED not set)
  - `gbrain.convention.auto_timeline` — did auto-timeline run?
  - `gbrain.convention.has_source_citation` — does the page body contain `source:` frontmatter?

**All spans flow through:** OTel Collector (already configured in docker-compose) → Jaeger (trace visualization) + Phoenix (LLM observability).

**If OTel is not configured:** `@opentelemetry/api` is a no-op by default — if no SDK is registered, `tracer.startSpan()` returns a no-op span. Zero overhead when OTel is not active.

---

## 3. package.json — Add OTel API dependency

```bash
cd ~/gbrain && bun add @opentelemetry/api
```

This is the API-only package (~50KB). It does NOT include the SDK, exporters, or auto-instrumentation. Those are configured by the environment (the OTel Collector in the devcontainer). The API package just provides the tracer interface.

---

## Verification

### D10 — Trusted mode

```bash
# With GBRAIN_MCP_TRUSTED=true:
# 1. Write a page via MCP
#    (from Claude Code session, use mcp__gbrain__put_page)
# 2. Check that auto-link ran:
gbrain get <slug> --json | jq '.links'
# Should have typed links extracted from the page body
# Without GBRAIN_MCP_TRUSTED=true, links array would be empty

# 3. Check that timeline entries were extracted:
gbrain get <slug> --json | jq '.timeline'
# Should have timeline entries from dates in the page
```

### OQ2/OQ6 — OTel traces

```bash
# 1. Open Jaeger UI (http://localhost:16686)
# 2. Search for service "gbrain-mcp" (or whatever OTEL_SERVICE_NAME resolves to)
# 3. Look for spans named "gbrain.mcp.put_page", "gbrain.mcp.search", etc.
# 4. Each span should show:
#    - gbrain.operation: the operation name
#    - gbrain.remote: true/false
#    - Duration in ms
#    - Status: OK or ERROR
# 5. For put_page spans, check convention attributes:
#    - gbrain.convention.auto_link: true/false
#    - gbrain.convention.has_source_citation: true/false
```

---

## Summary of all three parts

| Part | What it does | Files modified | Blocking dependencies |
|------|-------------|----------------|----------------------|
| **1** | Install gbrain, init PGLite, add env vars | Dockerfile, docker-compose, entrypoint | None |
| **2** | Clone brain repo, session branch, sync, embed, MCP, CLAUDE.md | entrypoint | Part 1 |
| **3** | Trusted MCP mode, OTel instrumentation | server.ts, dispatch.ts, package.json | Parts 1-2 for full testing |
