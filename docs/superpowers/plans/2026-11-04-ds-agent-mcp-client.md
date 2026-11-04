# ds-agent Generic MCP Client Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let ds-agent workers call tools from any configured MCP server, with Krene as the first consumer, without exposing those tools to the Router.

**Architecture:** Add an M1 `McpToolSource` that starts stdio MCP servers in the background, lists their tools, and namespaces them as `mcp__<server>__<tool>`. `ToolRegistry` merges those specs for the `worker` role only and routes matching calls to the source. Connection failure is isolated; tool results are normalized so images and large payloads never become base64 in the transcript.

**Tech Stack:** TypeScript, Node.js 22, pnpm, `@modelcontextprotocol/sdk`, vitest.

**Spec:** `/Users/sulin/krene-mcp/docs/superpowers/specs/2026-11-04-krene-mcp-design.md` (sections 5, 6.2, 8 Phase 3)

**Target repo:** `/Users/sulin/deepseek-super-agent`

## Global Constraints

- Node.js 22+, TypeScript, pnpm.
- Owner-only repo: only LofiSu contributes; do not add co-author trailers.
- Branch off current `main`; never commit to `main`; never force-push.
- `pnpm typecheck`, `pnpm lint`, and `pnpm test` must pass before a PR.
- Name by function and M0–M11 module (`docs/MODULES.md`), not by phase.
- No new user-facing feature switches; MCP is configured by data, not a boolean gate.
- MCP tools are exposed to `worker` only. Router and watchdog keep their current surface.
- Tool names are `mcp__<server>__<tool>`.
- Child process env allowlist: `PATH`, `HOME`, `LANG`, `TZ`, `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`, plus explicit config.
- Results over 64 KB go to spill; image/binary content is written to disk and referenced by path.

## Review Focus

1. A server that accepts the spawn but never finishes initialize: the first worker turn must still start; `specs()` times out and simply omits that server.
2. A tool returning an image or base64 blob: the tool message must contain a file path, never the encoded bytes.
3. A tool call that exceeds its timeout: the model receives an error result and the worker continues.
4. Invalid or duplicate server names: config parsing rejects them instead of silently shadowing a tool.
5. No `DS_AGENT_MCP_SERVERS`: behavior and existing tests are unchanged.

---

### Task 1: Parse MCP server configuration

**Files:**
- Create: `src/harness/mcp-config.ts`
- Test: `test/mcp-config.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export interface McpServerConfig {
    name: string
    command: string
    args: string[]
    env: Record<string, string>
    timeoutMs: number
  }
  export function parseMcpServers(raw: unknown): McpServerConfig[]
  ```
- Rules: `undefined`/`null` → `[]`; raw must be an array; `name` matches `/^[a-z0-9][a-z0-9_-]{0,31}$/i` and is unique; `command` is a non-empty trimmed string; `args` defaults to `[]` and must be strings; `env` defaults to `{}` and values must be strings; `timeoutMs` defaults to `120000` and must be a positive integer. Every rejection throws `UserError` with a message starting `invalid MCP server config:`.

- [ ] **Step 1: Write the failing test**

```ts
import { describe, expect, it } from 'vitest'
import { parseMcpServers } from '../src/harness/mcp-config.ts'

describe('parseMcpServers', () => {
  it('accepts a minimal server and fills defaults', () => {
    expect(parseMcpServers([{ name: 'krene', command: 'krene-mcp', args: ['serve'] }])).toEqual([
      { name: 'krene', command: 'krene-mcp', args: ['serve'], env: {}, timeoutMs: 120000 },
    ])
  })

  it('returns an empty list for absent configuration', () => {
    expect(parseMcpServers(undefined)).toEqual([])
    expect(parseMcpServers(null)).toEqual([])
  })

  it('rejects duplicate names', () => {
    expect(() => parseMcpServers([
      { name: 'krene', command: 'a' },
      { name: 'krene', command: 'b' },
    ])).toThrow(/duplicate/i)
  })

  it.each([
    [{ name: 'bad name', command: 'a' }, /name/i],
    [{ name: 'krene', command: '   ' }, /command/i],
    [{ name: 'krene', command: 'a', args: [1] }, /args/i],
    [{ name: 'krene', command: 'a', env: { A: 1 } }, /env/i],
    [{ name: 'krene', command: 'a', timeoutMs: 0 }, /timeout/i],
  ])('rejects invalid entry %j', (entry, pattern) => {
    expect(() => parseMcpServers([entry])).toThrow(pattern)
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm vitest run test/mcp-config.test.ts`
Expected: FAIL — module does not exist.

- [ ] **Step 3: Implement `parseMcpServers` in `src/harness/mcp-config.ts`**

Import `UserError` from `../shared/errors.ts`. Validate in the order listed above; stop at the first invalid entry. Do not mutate the input.

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm vitest run test/mcp-config.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/harness/mcp-config.ts test/mcp-config.test.ts
git commit -m "feat(harness): parse MCP server configuration"
```

---

### Task 2: Widen tool parameter schema and make tool listing async

**Files:**
- Modify: `src/harness/model.ts:8-16` (`ToolSpec`)
- Modify: `src/harness/tools.ts:89-93` (`specsFor`)
- Modify: `src/harness/loop.ts:759` (await)
- Modify: `test/credential-sentinel.test.ts:221-225` (await)
- Test: `test/tool-spec-schema.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export interface ToolSpec {
    name: string
    description: string
    parameters: {
      type: 'object'
      properties?: Record<string, unknown>
      required?: string[]
      additionalProperties?: boolean | Record<string, unknown>
      [key: string]: unknown
    }
  }
  export class ToolRegistry {
    specsFor(role: Role): Promise<ToolSpec[]>
  }
  ```
- Consumes: `ToolSpec` in `src/harness/model.ts`; `Role` from `src/shared/types.ts`.

- [ ] **Step 1: Write the failing test**

```ts
import { expect, it } from 'vitest'
import type { ToolSpec } from '../src/harness/model.ts'

it('accepts full JSON Schema tool parameters', () => {
  const spec: ToolSpec = {
    name: 'example',
    description: 'array and enum',
    parameters: {
      type: 'object',
      properties: {
        refs: { type: 'array', items: { type: 'string' }, minItems: 1 },
        model: { type: 'string', enum: ['a', 'b'] },
      },
      required: ['refs'],
      additionalProperties: false,
    },
  }
  expect((spec.parameters.properties?.refs as { items: { type: string } }).items.type).toBe('string')
})
```

- [ ] **Step 2: Run test and typecheck to verify failure**

Run: `pnpm vitest run test/tool-spec-schema.test.ts && pnpm typecheck`
Expected: FAIL — the current `properties` value type rejects `items`/`enum`.

- [ ] **Step 3: Implement the widening and async listing**

Change `ToolSpec.parameters` as shown. Change `ToolRegistry.specsFor` to `async specsFor(role: Role): Promise<ToolSpec[]>` and return the same array. In `src/harness/loop.ts`, change `const tools = input.tools.specsFor(...)` to `const tools = await input.tools.specsFor(...)`. Update the two `specsFor` calls in `test/credential-sentinel.test.ts` to `await`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm vitest run test/tool-spec-schema.test.ts test/credential-sentinel.test.ts && pnpm typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/harness/model.ts src/harness/tools.ts src/harness/loop.ts test/credential-sentinel.test.ts test/tool-spec-schema.test.ts
git commit -m "refactor(harness): allow full JSON Schema tool parameters"
```

---

### Task 3: Connect to MCP servers and namespace their tools

**Files:**
- Create: `src/harness/mcp.ts`
- Create: `test/fixtures/fake-mcp-server.mjs`
- Test: `test/mcp-source.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export interface McpToolSourceDeps {
    log: (line: string) => void
    metrics: { increment(name: MetricName, by?: number): void }
    redact: (text: string) => string
    spill: { put(body: string): string }
    workspaceDir: string
    baseEnv?: NodeJS.ProcessEnv
  }
  export interface McpToolSourceLike {
    specs(): Promise<ToolSpec[]>
    invoke(name: string, args: Record<string, unknown>): Promise<unknown>
    close(): Promise<void>
  }
  export class McpToolSource implements McpToolSourceLike {
    constructor(configs: McpServerConfig[], deps: McpToolSourceDeps)
    specs(): Promise<ToolSpec[]>
    invoke(name: string, args: Record<string, unknown>): Promise<unknown>
    close(): Promise<void>
  }
  export function mcpToolName(server: string, tool: string): string
  export function childEnv(config: McpServerConfig, baseEnv?: NodeJS.ProcessEnv): Record<string, string>
  ```
- Imports: `MetricName` from `../metrics/metrics.ts`.
- Behavior: constructor starts one connection per config without awaiting. `specs()` awaits all connections with a 10-second cap, returns namespaced specs for servers that connected, and omits failed or timed-out servers. `invoke` awaits the matching server's ready promise, throws `UserError` for an unknown name, and resolves names by longest configured `mcp__<server>__` prefix (server names may contain `_`). `childEnv` returns only the allowlist plus `config.env`.

- [ ] **Step 1: Write the fake MCP server fixture**

`test/fixtures/fake-mcp-server.mjs` uses the installed `@modelcontextprotocol/sdk` to serve stdio and registers five tools: `echo` (string `text`, optional string array `tags`), `big` (70 KB text), `fail` (throws `boom`), `hang` (waits 500 ms then returns), and `image` (returns an `image` content block with the base64 of a 1×1 PNG and `mimeType: 'image/png'`). Use the SDK's current registration API for the pinned version. Exit on stdin close.

- [ ] **Step 2: Write the failing test**

```ts
import path from 'node:path'
import { fileURLToPath } from 'node:url'
import { describe, expect, it } from 'vitest'
import { McpToolSource, childEnv, mcpToolName } from '../src/harness/mcp.ts'
import { parseMcpServers } from '../src/harness/mcp-config.ts'

const serverPath = fileURLToPath(new URL('./fixtures/fake-mcp-server.mjs', import.meta.url))

function sourceFor(configs: unknown) {
  return new McpToolSource(parseMcpServers(configs), {
    log: () => {},
    metrics: { increment: () => {} },
    redact: text => text,
    spill: { put: () => 'spill_1' },
    workspaceDir: path.join(fileURLToPath(new URL('.', import.meta.url)), 'tmp-mcp'),
  })
}

describe('McpToolSource', () => {
  it('lists namespaced tools from a connected server', async () => {
    const source = sourceFor([{ name: 'fake', command: process.execPath, args: [serverPath] }])
    const specs = await source.specs()
    expect(specs.map(spec => spec.name)).toEqual(expect.arrayContaining([
      mcpToolName('fake', 'echo'),
      mcpToolName('fake', 'big'),
      mcpToolName('fake', 'fail'),
      mcpToolName('fake', 'hang'),
    ]))
    await source.close()
  })

  it('keeps a good server when another fails to start', async () => {
    const source = sourceFor([
      { name: 'broken', command: '/definitely/not/a/binary' },
      { name: 'fake', command: process.execPath, args: [serverPath] },
    ])
    const specs = await source.specs()
    expect(specs.some(spec => spec.name.startsWith('mcp__fake__'))).toBe(true)
    expect(specs.some(spec => spec.name.startsWith('mcp__broken__'))).toBe(false)
    await source.close()
  })

  it('builds an allowlisted child environment', () => {
    const env = childEnv({ name: 'x', command: 'x', args: [], env: { EXTRA: '1' }, timeoutMs: 1000 }, {
      PATH: '/bin',
      HOME: '/home/u',
      LANG: 'en',
      TZ: 'UTC',
      SECRET: 'nope',
    } as NodeJS.ProcessEnv)
    expect(env).toEqual({ PATH: '/bin', HOME: '/home/u', LANG: 'en', TZ: 'UTC', EXTRA: '1' })
  })
})
```

- [ ] **Step 3: Run test to verify it fails**

Run: `pnpm vitest run test/mcp-source.test.ts`
Expected: FAIL — `src/harness/mcp.ts` does not exist.

- [ ] **Step 4: Implement `McpToolSource` connection and listing**

Use `Client` and `StdioClientTransport` from `@modelcontextprotocol/sdk/client/index.js` and `@modelcontextprotocol/sdk/client/stdio.js`. Start `connect()` in the constructor; store each server's ready promise. `specs()` does `Promise.race([Promise.allSettled(readies), timeout(10_000)])`. Map each MCP tool to `{ name: mcpToolName(server, tool.name), description: tool.description ?? '', parameters: tool.inputSchema }`. Increment `mcp.connect` on success and `mcp.connect_error` on failure.

- [ ] **Step 5: Run test to verify it passes**

Run: `pnpm vitest run test/mcp-source.test.ts`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/harness/mcp.ts test/fixtures/fake-mcp-server.mjs test/mcp-source.test.ts
git commit -m "feat(harness): connect MCP servers and list namespaced tools"
```

---

### Task 4: Invoke tools, normalize results, enforce timeouts

**Files:**
- Modify: `src/harness/mcp.ts`
- Test: `test/mcp-invoke.test.ts`

**Interfaces:**
- `invoke(name: string, args: Record<string, unknown>): Promise<unknown>` returns:
  - a string when the result is text only;
  - `{ paths: string[], mimeTypes: string[] }` metadata appended when image/resource blocks are written to disk;
  - `{ spillId: string, bytes: number, preview: string }` when the JSON form exceeds 64 KB.
- Throws `UserError` with `mcp tool <name> timed out after <ms>ms` on timeout.

- [ ] **Step 1: Write the failing test**

```ts
import fs from 'node:fs'
import path from 'node:path'
import { fileURLToPath } from 'node:url'
import { describe, expect, it } from 'vitest'
import { McpToolSource } from '../src/harness/mcp.ts'
import { parseMcpServers } from '../src/harness/mcp-config.ts'

const serverPath = fileURLToPath(new URL('./fixtures/fake-mcp-server.mjs', import.meta.url))
const workspaceDir = fileURLToPath(new URL('./tmp-mcp-invoke', import.meta.url))

async function source(timeoutMs = 5_000) {
  return new McpToolSource(parseMcpServers([{ name: 'fake', command: process.execPath, args: [serverPath], timeoutMs }]), {
    log: () => {},
    metrics: { increment: () => {} },
    redact: text => text,
    spill: { put: () => 'spill_1' },
    workspaceDir,
  })
}

describe('McpToolSource.invoke', () => {
  it('passes array arguments through', async () => {
    const mcp = await source()
    await expect(mcp.invoke('mcp__fake__echo', { text: 'hi', tags: ['a', 'b'] }))
      .resolves.toContain('"tags":["a","b"]')
    await mcp.close()
  })

  it('spills oversized text results', async () => {
    const mcp = await source()
    await expect(mcp.invoke('mcp__fake__big', {})).resolves.toEqual({
      spillId: 'spill_1',
      bytes: expect.any(Number),
      preview: expect.any(String),
    })
    await mcp.close()
  })

  it('writes image blocks to disk instead of returning base64', async () => {
    const mcp = await source()
    const result = await mcp.invoke('mcp__fake__image', {}) as { paths: string[] }
    expect(result.paths).toHaveLength(1)
    expect(fs.existsSync(result.paths[0])).toBe(true)
    expect(JSON.stringify(result)).not.toContain('iVBORw0KGgo')
    await mcp.close()
  })

  it('surfaces a tool error without closing the source', async () => {
    const mcp = await source()
    await expect(mcp.invoke('mcp__fake__fail', {})).rejects.toThrow(/boom/)
    await expect(mcp.invoke('mcp__fake__echo', { text: 'still alive' })).resolves.toContain('still alive')
    await mcp.close()
  })

  it('times out a slow call', async () => {
    const mcp = await source(50)
    await expect(mcp.invoke('mcp__fake__hang', {})).rejects.toThrow(/timed out after 50ms/)
    await mcp.close()
  })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm vitest run test/mcp-invoke.test.ts`
Expected: FAIL — `invoke` is not implemented.

- [ ] **Step 3: Implement invocation and normalization**

Resolve the server by longest configured `mcp__<server>__` prefix, await its ready promise, then call `client.callTool({ name: tool, arguments: args })` under `Promise.race` with the server's `timeoutMs`. Normalize `result.content`: concatenate `text` blocks; for `image` blocks decode base64 and write `workspaceDir/<server>/<randomUUID>.<ext>` (create the directory recursively); for `resource` blocks write text or binary similarly. Merge text and path metadata. If `JSON.stringify(normalized).length > 64 * 1024`, call `deps.spill.put` and return the spill shape. Increment `mcp.call` on success, `mcp.call_error` on failure, and `mcp.call_timeout` on timeout.

- [ ] **Step 4: Run test to verify it passes**

Run: `pnpm vitest run test/mcp-invoke.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/harness/mcp.ts test/mcp-invoke.test.ts
git commit -m "feat(harness): invoke MCP tools with normalization and timeouts"
```

---

### Task 5: Wire MCP into ToolRegistry, Runtime, and metrics

**Files:**
- Modify: `src/harness/tools.ts` (`ToolDeps`, constructor, `specsFor`, `invoke`, new `close`)
- Modify: `src/runtime.ts` (parse config, construct source, keep registry, close in `finish`)
- Modify: `src/metrics/metrics.ts` (`METRIC_NAMES`)
- Test: `test/mcp-registry.test.ts`

**Interfaces:**
- `ToolDeps` gains `mcp?: McpToolSourceLike`.
- `ToolRegistry.specsFor('worker')` includes `mcp.specs()`; `'router'` and `'watchdog'` never do.
- `ToolRegistry.invoke` routes `mcp__` calls to `mcp.invoke`.
- `ToolRegistry.close(): Promise<void>` awaits `mcp.close()`.
- `RuntimeOptions` gains `mcpServers?: unknown`.
- New metric names: `mcp.connect`, `mcp.connect_error`, `mcp.call`, `mcp.call_error`, `mcp.call_timeout`. Add them to both `METRIC_NAMES` and `blankCounters()` in `src/metrics/metrics.ts`.

- [ ] **Step 1: Write the failing test**

Build a `ToolRegistry` with the existing helper shape used in `test/highs.test.ts`, passing a stub `mcp`:

```ts
const stub = {
  specs: async () => [{ name: 'mcp__fake__echo', description: 'echo', parameters: { type: 'object' as const } }],
  invoke: async (name: string, args: Record<string, unknown>) => ({ name, args }),
  close: async () => { closed = true },
}
```

Assert: `(await tools.specsFor('worker')).some(t => t.name === 'mcp__fake__echo')` is true; `(await tools.specsFor('router')).some(...)` is false; `await tools.invoke({ id: '1', name: 'mcp__fake__echo', args: { text: 'hi' } }, { sessionId: 's', role: 'worker' })` returns the stub result; `await tools.close()` sets `closed`.

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm vitest run test/mcp-registry.test.ts`
Expected: FAIL — `ToolRegistry` does not accept `mcp` and `specsFor` does not merge it.

- [ ] **Step 3: Implement the wiring**

In `ToolRegistry`, merge `await this.deps.mcp.specs()` only when `role === 'worker'`, and call `this.deps.mcp.invoke` for names starting `mcp__`. In `Runtime.create`, parse `opts.mcpServers ?? baseEnv.DS_AGENT_MCP_SERVERS`, catch invalid JSON and log `[mcp] DS_AGENT_MCP_SERVERS ignored: ...`, construct one `McpToolSource` (only when configs are non-empty), pass it to `ToolRegistry`, assign the registry to a new private field, and `await this.tools.close()` in `finish()` inside the existing drain block. Add the five metric names to `METRIC_NAMES` and their zero entries to `blankCounters()`.

- [ ] **Step 4: Run test and full gates**

Run: `pnpm vitest run test/mcp-registry.test.ts && pnpm typecheck && pnpm lint`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/harness/tools.ts src/runtime.ts src/metrics/metrics.ts test/mcp-registry.test.ts
git commit -m "feat(runtime): expose configured MCP tools to workers"
```

---

### Task 6: Document the MCP module and configuration

**Files:**
- Modify: `docs/MODULES.md` (M1 section)
- Modify: `README.md` (configuration section)
- Test: none (documentation); verified by the full gates

**Interfaces:**
- Document exactly: `DS_AGENT_MCP_SERVERS` JSON shape, `mcp__<server>__<tool>` naming, worker-only exposure, env allowlist, 64 KB spill threshold, and the fact that connection failure logs `mcp.connect_error` but does not block the runtime.

- [ ] **Step 1: Update `docs/MODULES.md`**

Add `mcp-config.ts` and `mcp.ts` to M1 with one sentence each: config parsing/validation, and stdio connection plus namespaced tool surface.

- [ ] **Step 2: Update `README.md`**

Add a short "MCP servers" section with the JSON example from the spec, the env allowlist, and how to point `command` at an absolute path.

- [ ] **Step 3: Run the full gates**

Run: `pnpm typecheck && pnpm lint && pnpm test`
Expected: PASS. Paste the summary lines in the PR.

- [ ] **Step 4: Commit**

```bash
git add docs/MODULES.md README.md
git commit -m "docs: describe the MCP tool surface and configuration"
```

---

## Self-Review Notes

- Spec §6.2 items map to tasks: config (Task 1), schema fidelity (Task 2), connect/namespace (Task 3), normalize/spill/timeout (Task 4), worker-only + Runtime lifecycle + metrics (Task 5), docs (Task 6).
- Type consistency: `McpServerConfig` is defined once in Task 1 and imported by Task 3. `McpToolSource` methods keep the same signatures across Tasks 3–5.
- Review Focus tests live in: hung server (Task 3 bad-server test), image/base64 (Task 4), timeout (Task 4), duplicate names (Task 1), no config (Task 5 behavior is unchanged when `mcp` is absent).
