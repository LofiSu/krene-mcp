# krene-mcp Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a stdio MCP server that uses the operator's own Krene account to run design conversations and generate/edit images, with a one-time browser login and a native HTTP + WebSocket runtime.

**Architecture:** A small TypeScript CLI and MCP server. Phase 0 records real HTTP/WS traffic after a manual login, sanitizes it into fixtures, and answers the protocol questions in the spec. Later tasks implement the session, protocol codec, socket client, task pipeline, and MCP tools against those fixtures, so the test suite never needs the network.

**Tech Stack:** TypeScript, Node.js 22, pnpm, `@modelcontextprotocol/sdk`, `ws`, `zod`, vitest, Playwright (optional, login/capture only).

**Spec:** `/Users/sulin/krene-mcp/docs/superpowers/specs/2026-11-04-krene-mcp-design.md`

**Target repo:** `/Users/sulin/krene-mcp`

## Global Constraints

- Node.js 22+, TypeScript, pnpm.
- MCP transport is stdio only; no HTTP transport in v1.
- `krene-login` is CLI-only. MCP tools never open a browser or wait for a human.
- Runtime uses native HTTP + WebSocket; no DOM automation after login.
- Session file is `~/.krene-mcp/session.json` with mode `0600`; never commit it.
- Tokens, cookies, user identifiers, phone numbers, and signed URL parameters never appear in fixtures, logs, or errors.
- Input paths are confined to the allowed roots; export URLs are confined to Krene hosts.
- One generation per session at a time (`BUSY` on contention).
- `pnpm typecheck` and `pnpm test` must pass at every commit.
- Run implementation on a feature branch or worktree created from `main`.

## Review Focus

1. Token expired or the server answers `code: 4001`: the tool returns `AUTH_EXPIRED`; no reconnect loop and no browser.
2. An unknown `trigger` or changed field: parse fails as `PROTOCOL_DRIFT` with the build version, instead of crashing the process.
3. A truncated or malformed binary image frame: clear error, bounded memory, no partial file left behind.
4. `imagePath` / `destDir` escaping the allowed root, or a non-Krene export URL: rejected before any I/O.
5. Two clients starting a generation at once: the second gets `BUSY`, no duplicate board or double charge.
6. Session file created with the wrong permissions: `0600` is enforced and tested.

---

### Task 1: Scaffold the CLI, config, and logging

**Files:**
- Create: `package.json`, `tsconfig.json`, `vitest.config.ts`
- Create: `src/config.ts`, `src/log.ts`, `src/cli.ts`
- Create: `test/helpers.ts` (`tempHome()` → `fs.mkdtempSync(path.join(os.tmpdir(), 'krene-mcp-'))`)
- Test: `test/config.test.ts`, `test/log.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export interface Config {
    home: string
    sessionPath: string
    outputDir: string
    lockPath: string
    socketBase: string
    requestTimeoutMs: number
    toolTimeoutMs: number
    lang: string
  }
  export function loadConfig(env?: NodeJS.ProcessEnv): Config
  export function redact(text: string): string
  export function logLine(event: string, fields?: Record<string, unknown>): void
  ```
- Defaults: `KRENE_HOME` or `~/.krene-mcp`; session `session.json`; output `output/`; lock `lock`; socket base `wss://ng.krene.com/qa_psa/moodWs`; request timeout 30 000 ms; tool timeout 120 000 ms; lang `zh-CN`. `KRENE_SOCKET_BASE`, `KRENE_REQUEST_TIMEOUT_MS`, `KRENE_TOOL_TIMEOUT_MS`, `KRENE_LANG` override.
- `package.json`: `"type": "module"`, bin `krene-mcp` → `dist/cli.js`; scripts `build`, `typecheck`, `test`, `dev`, `capture`, `login`, `serve`. Dependencies `@modelcontextprotocol/sdk`, `ws`, `zod`; dev `typescript`, `tsx`, `vitest`, `@types/node`, `@types/ws`; optional `playwright`.

- [ ] **Step 1: Write the failing tests**

```ts
import { expect, it } from 'vitest'
import { loadConfig } from '../src/config.ts'

it('uses defaults under KRENE_HOME', () => {
  const config = loadConfig({ KRENE_HOME: '/tmp/krene-home' } as NodeJS.ProcessEnv)
  expect(config.sessionPath).toBe('/tmp/krene-home/session.json')
  expect(config.outputDir).toBe('/tmp/krene-home/output')
  expect(config.toolTimeoutMs).toBe(120_000)
  expect(config.lang).toBe('zh-CN')
})

it('honors timeout and language overrides', () => {
  const config = loadConfig({
    KRENE_HOME: '/tmp/k',
    KRENE_REQUEST_TIMEOUT_MS: '5000',
    KRENE_TOOL_TIMEOUT_MS: '9000',
    KRENE_LANG: 'en',
  } as NodeJS.ProcessEnv)
  expect(config.requestTimeoutMs).toBe(5000)
  expect(config.toolTimeoutMs).toBe(9000)
  expect(config.lang).toBe('en')
})

it('rejects non-positive timeouts', () => {
  expect(() => loadConfig({ KRENE_TOOL_TIMEOUT_MS: '0' } as NodeJS.ProcessEnv)).toThrow(/timeout/i)
})
```

```ts
import { expect, it } from 'vitest'
import { redact } from '../src/log.ts'

it('redacts auth headers and query tokens', () => {
  const text = 'K-Authorization: Bearer abc123 token=xyz&x=1 phone=13800138000'
  const out = redact(text)
  expect(out).not.toContain('abc123')
  expect(out).not.toContain('xyz')
  expect(out).not.toContain('13800138000')
})
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run test/config.test.ts test/log.test.ts`
Expected: FAIL — modules and scripts do not exist.

- [ ] **Step 3: Implement the scaffold**

Create the package manifest, TypeScript config (`module: NodeNext`, `strict`, `outDir: dist`), vitest config, and the three source files. `cli.ts` dispatches `login`, `capture`, `sanitize`, `status`, `serve`; unknown commands print usage and exit 2. `login`, `sanitize`, `status`, and `serve` may call stubs in this task.

- [ ] **Step 4: Run tests and typecheck**

Run: `pnpm vitest run test/config.test.ts test/log.test.ts && pnpm typecheck`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add package.json pnpm-lock.yaml tsconfig.json vitest.config.ts src test
git commit -m "feat: scaffold the krene-mcp CLI and config"
```

---

### Task 2: Record a live session into captures

**Files:**
- Create: `src/capture/recorder.ts`
- Modify: `src/cli.ts` (`capture` command)
- Test: `test/capture-format.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export interface CaptureOptions { outDir: string; url: string; headed: boolean }
  export interface CaptureResult { dir: string; frames: number; requests: number }
  export function summarizeFrame(frame: { direction: 'sent' | 'received'; payload: string; at: number }): string
  export function redactHeaders(headers: Record<string, string>): Record<string, string>
  export async function recordCapture(opts: CaptureOptions): Promise<CaptureResult>
  ```
- `recordCapture` dynamically imports `playwright`, launches Chromium, opens `https://www.krene.com/design/`, attaches a CDP session, and writes `frames.jsonl`, `http.jsonl`, and `manifest.json` until SIGINT. It records `Network.webSocketCreated`, `webSocketFrameSent`, `webSocketFrameReceived`, and `webSocketClosed`; HTTP records method, URL, status, and redacted headers.

- [ ] **Step 1: Write the failing test**

```ts
import { expect, it } from 'vitest'
import { redactHeaders, summarizeFrame } from '../src/capture/recorder.ts'

it('keeps frame direction and byte count, not the payload', () => {
  expect(summarizeFrame({ direction: 'sent', payload: '{"prompt":"secret"}', at: 1 }))
    .toBe('sent bytes=19 at=1')
})

it('redacts auth and cookie headers', () => {
  expect(redactHeaders({
    'K-Authorization': 'Bearer abc',
    cookie: 'session=def',
    'X-Language': 'zh-CN',
  })).toEqual({ 'K-Authorization': '[redacted]', cookie: '[redacted]', 'X-Language': 'zh-CN' })
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm vitest run test/capture-format.test.ts`
Expected: FAIL — module does not exist.

- [ ] **Step 3: Implement the recorder and CLI command**

`capture --out <dir>` defaults to `captures/<ISO timestamp>`. Print the output path and "log in, then do one design and one image generation" before waiting. Exit cleanly on SIGINT after flushing files.

- [ ] **Step 4: Run tests**

Run: `pnpm vitest run test/capture-format.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/capture/recorder.ts src/cli.ts test/capture-format.test.ts
git commit -m "feat(capture): record HTTP and WebSocket traffic"
```

---

### Task 3: Sanitize captures and answer the protocol questions

**Files:**
- Create: `src/capture/sanitize.ts`
- Create: `docs/protocol-notes.md`
- Create: `fixtures/design-session.jsonl`, `fixtures/generate-image.jsonl`, `fixtures/edit-image.jsonl` (one JSON frame per line)
- Test: `test/capture-sanitize.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export function sanitizeCapture(rawDir: string, outDir: string): { files: string[]; redactions: number }
  ```
- Redacts at minimum: `K-Authorization`, `Authorization`, `cookie`, `krene_user`, `token=` query values, phone numbers, email addresses, and signed URL params (`Signature`, `Expires`, `X-Amz-*`, `token`).
- `docs/protocol-notes.md` must answer the eight questions in spec §10: connect URL/headers, handshake, `analysis`/`executeTask` payloads, `sendPrompt` relationship, result shape (JSON vs binary), `krene_user` value, board/conversion relation, and where credits are deducted.

- [ ] **Step 1: Write the failing test**

Create `test/fixtures/raw-capture/frames.jsonl` with one line containing `token=SECRET123`, `krene_user: 13800138000`, and a signed URL, plus one clean line. Assert after `sanitizeCapture`:

```ts
const out = fs.readFileSync(path.join(outDir, 'frames.jsonl'), 'utf8')
expect(out).not.toContain('SECRET123')
expect(out).not.toContain('13800138000')
expect(out).toContain('[redacted]')
expect(result.redactions).toBeGreaterThan(0)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm vitest run test/capture-sanitize.test.ts`
Expected: FAIL — module does not exist.

- [ ] **Step 3: Implement the sanitizer**

Stream each line, parse JSON, redact matching fields recursively, and write the sanitized line. Copy only `manifest.json` and the two JSONL files. Refuse to write into `rawDir`.

- [ ] **Step 4: Run the capture manually (Phase 0 gate)**

Run: `pnpm capture --out captures/phase0`
Complete login, one design conversation, and one image generation in the opened browser, then press Ctrl-C. Run `pnpm tsx src/cli.ts sanitize captures/phase0 fixtures`, then split the sanitized frames into the three named fixture files. Fill `docs/protocol-notes.md` by inspecting `fixtures/`.

Expected: sanitized fixtures exist, grep for secrets returns nothing, and the notes answer all eight questions. **Do not start Task 5 before this is complete.**

- [ ] **Step 5: Commit**

```bash
git add src/capture/sanitize.ts test/capture-sanitize.test.ts test/fixtures docs/protocol-notes.md fixtures
git commit -m "feat(capture): sanitize traffic and document the protocol"
```

---

### Task 4: Session storage, HTTP client, and browser login

**Files:**
- Create: `src/auth/session.ts`, `src/auth/login.ts`, `src/http/client.ts`
- Modify: `src/cli.ts` (`login`, `status`)
- Test: `test/session.test.ts`, `test/http-client.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export interface Session { token: string; userId: string; displayName?: string; savedAt: string; source: 'playwright' }
  export function readSession(config: Config): Session | undefined
  export function writeSession(config: Config, session: Session): void
  export function clearSession(config: Config): void
  export function assertSession(config: Config): Session
  export async function loginWithBrowser(config: Config): Promise<Session>
  export class KreneHttp {
    constructor(opts: { config: Config; session: Session; fetchImpl?: typeof fetch })
    get<T>(path: string): Promise<T>
    post<T>(path: string, body: unknown): Promise<T>
    memberInfo(): Promise<{ uuid: string; userName?: string; points?: number }>
  }
  ```
- `writeSession` uses `fs.writeFileSync(path, json, { mode: 0o600 })` and `chmod` to `0o600` even when the file exists.
- `KreneHttp` sends `K-Authorization: Bearer <token>`, `krene_user: <userId>`, `X-Language: <lang>`, `Accept: application/json`.
- A 401/403 maps to `UserError('AUTH_EXPIRED: run krene-mcp login')`.
- `loginWithBrowser` opens `https://www.krene.com/design/`, polls `localStorage.getItem('userInfo')` once per second, parses `{ token, uuid, userName }`, maps `uuid` → `userId`, validates with `memberInfo`, and writes the session.

- [ ] **Step 1: Write the failing tests**

```ts
it('writes the session with 0600', () => {
  const config = loadConfig({ KRENE_HOME: tempHome() } as NodeJS.ProcessEnv)
  writeSession(config, { token: 't', userId: 'u', savedAt: '2026-11-04T00:00:00.000Z', source: 'playwright' })
  expect(fs.statSync(config.sessionPath).mode & 0o777).toBe(0o600)
  expect(readSession(config)?.token).toBe('t')
})
```

```ts
it('sends Krene auth headers', async () => {
  const seen: Record<string, string> = {}
  const http = new KreneHttp({
    config: loadConfig({ KRENE_HOME: '/tmp/k' } as NodeJS.ProcessEnv),
    session: { token: 'tok', userId: 'uid', savedAt: 'x', source: 'playwright' },
    fetchImpl: async (_url, init) => {
      Object.assign(seen, init?.headers)
      return new Response(JSON.stringify({ code: 200, data: {} }), { status: 200 })
    },
  })
  await http.get('/krene-backend-ts/api/member/info?userIdentifier=uid')
  expect(seen['K-Authorization']).toBe('Bearer tok')
  expect(seen.krene_user).toBe('uid')
})
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run test/session.test.ts test/http-client.test.ts`
Expected: FAIL — modules do not exist.

- [ ] **Step 3: Implement session, HTTP, and login**

`assertSession` throws `AUTH_REQUIRED` when absent. `memberInfo` calls `/krene-backend-ts/api/member/info?userIdentifier=<id>`. Login uses `playwright` via dynamic import; if it is not installed, throw `UserError('playwright is required for login; install it first')`. Non-2xx responses other than 401/403 throw `UserError('NETWORK: <status> <path>')`.

- [ ] **Step 4: Run tests and manual login**

Run: `pnpm vitest run test/session.test.ts test/http-client.test.ts`
Then: `pnpm tsx src/cli.ts login` (manual captcha) followed by `pnpm tsx src/cli.ts status`.
Expected: tests PASS; status prints the user name.

- [ ] **Step 5: Commit**

```bash
git add src/auth src/http src/cli.ts test/session.test.ts test/http-client.test.ts
git commit -m "feat(auth): add session storage, HTTP client, and browser login"
```

---

### Task 5: Implement the protocol codec from fixtures

**Files:**
- Create: `src/protocol/commands.ts`, `src/protocol/events.ts`, `src/protocol/image-frames.ts`, `src/protocol/schema.ts`
- Test: `test/protocol.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export type OutboundCommand =
    | { command: 'getConversionId'; messageId: string; prompt: string }
    | { command: 'analysis'; messageId: string; params: AnalysisParams }
    | { command: 'executeTask'; messageId: string; params: ExecuteTaskParams }
    | { command: string; messageId: string; [key: string]: unknown }
  export interface AnalysisParams { conversionId: string; prompt: string; imageUuid?: string; parentStepId?: string; chatOn?: boolean; strategyCode?: string; lang: string }
  export interface ExecuteTaskParams { conversionId: string; tasks: unknown[]; parentStepId?: string }
  export function messageId(): string
  export function getConversionId(prompt: string): OutboundCommand
  export function analysis(params: AnalysisParams): OutboundCommand
  export function executeTask(params: ExecuteTaskParams): OutboundCommand
  export function parseServerFrame(raw: string): ServerEvent
  export function parseImagePushFrame(buffer: Buffer): { meta: unknown; bytes: Buffer }
  export type TaskStatus = 'PENDING EXECUTION' | 'EXECUTING' | 'CONFIRMING' | 'PENDING' | 'COMPLETED' | 'CANCELED' | 'EXCEPTION'
  ```
- `parseServerFrame` validates with zod and throws `ProtocolDriftError` on unknown `trigger` or missing fields. `parseImagePushFrame` reads the first 8 bytes as a big-endian length `r`, then metadata of `buffer.length - r - 8`, then `r` image bytes; reject when `r` exceeds the buffer or the metadata is not JSON.
- Exact payload fields come from `docs/protocol-notes.md`; this task pins them in the zod schemas.

- [ ] **Step 1: Write the failing tests**

```ts
it('parses a recorded COMPLETED event', () => {
  const raw = fs.readFileSync('fixtures/generate-image.jsonl', 'utf8')
  const lines = raw.trim().split('\n').map(line => JSON.parse(line))
  const event = parseServerFrame(JSON.stringify(lines.find(line => line.trigger === 'UPDATE_TASK_CARD_STATUS')))
  expect(event.trigger).toBe('UPDATE_TASK_CARD_STATUS')
})

it('rejects an unknown trigger as protocol drift', () => {
  expect(() => parseServerFrame(JSON.stringify({ code: 200, trigger: 'NEW_THING', data: {} })))
    .toThrow(/PROTOCOL_DRIFT/)
})

it('parses the binary image frame layout', () => {
  const meta = Buffer.from(JSON.stringify({ uuid: 'i1' }))
  const image = Buffer.from([1, 2, 3])
  const header = Buffer.alloc(8)
  header.writeBigUInt64BE(BigInt(image.length))
  const parsed = parseImagePushFrame(Buffer.concat([header, meta, image]))
  expect(parsed.meta).toEqual({ uuid: 'i1' })
  expect([...parsed.bytes]).toEqual([1, 2, 3])
})

it('rejects a truncated image frame', () => {
  const header = Buffer.alloc(8)
  header.writeBigUInt64BE(100n)
  expect(() => parseImagePushFrame(Buffer.concat([header, Buffer.from('{}')]))).toThrow(/image frame/i)
})
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run test/protocol.test.ts`
Expected: FAIL — modules do not exist.

- [ ] **Step 3: Implement the codec**

Build commands exactly as recorded. `messageId()` returns `crypto.randomUUID()`. Use zod for `ServerEvent` with `code`, `trigger`, and `data`; preserve unknown `data` fields with `.passthrough()`. Define `ProtocolDriftError` in `schema.ts`.

- [ ] **Step 4: Run tests**

Run: `pnpm vitest run test/protocol.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/protocol test/protocol.test.ts
git commit -m "feat(protocol): encode commands and decode server frames"
```

---

### Task 6: WebSocket client with heartbeat and auth errors

**Files:**
- Create: `src/ws/client.ts`
- Test: `test/ws-client.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export function buildSocketUrl(opts: { base: string; boardId: string; token: string }): string
  export class KreneSocket {
    constructor(opts: {
      url: string
      timeoutMs: number
      onEvent: (event: ServerEvent) => void
      onImage: (push: { meta: unknown; bytes: Buffer }) => void
    })
    open(): Promise<void>
    send(command: OutboundCommand): void
    close(): void
  }
  ```
- `buildSocketUrl` uses the query names recorded in `docs/protocol-notes.md`; if the capture shows no `boardId` parameter, omit it.
- Sends `{ command: 'heartMessage', messageId }` every 30 seconds while open.
- A JSON frame with `code: 4001` rejects with `AUTH_EXPIRED` and does not reconnect.
- Reconnects at most 5 times with exponential backoff (1 s, 2 s, 4 s, 8 s, 16 s, capped at 30 s); `close()` disables reconnect.

- [ ] **Step 1: Write the failing test**

Use `WebSocketServer` from `ws` on `127.0.0.1:0`:

```ts
it('sends commands and delivers events', async () => {
  const wss = new WebSocketServer({ port: 0 })
  const url = `ws://127.0.0.1:${(wss.address() as AddressInfo).port}`
  const events: ServerEvent[] = []
  const socket = new KreneSocket({ url, timeoutMs: 1000, onEvent: e => events.push(e), onImage: () => {} })
  await socket.open()
  wss.clients.forEach(client => client.send(JSON.stringify({ code: 200, trigger: 'CONNECT_SUCCESS', data: {} })))
  await vi.waitFor(() => expect(events[0]?.trigger).toBe('CONNECT_SUCCESS'))
  socket.send(getConversionId('hello'))
  socket.close()
  wss.close()
})

it('maps 4001 to AUTH_EXPIRED without reconnecting', async () => {
  // server replies {code:4001}; assert open() rejects with AUTH_EXPIRED and the server saw one connection
})
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm vitest run test/ws-client.test.ts`
Expected: FAIL — module does not exist.

- [ ] **Step 3: Implement the client**

Accept string and Buffer frames; strings go to `parseServerFrame`, buffers to `parseImagePushFrame`. Resolve `open()` on the first `CONNECT_SUCCESS` or on the socket's `open` event when the server sends no handshake.

- [ ] **Step 4: Run tests**

Run: `pnpm vitest run test/ws-client.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/ws/client.ts test/ws-client.test.ts
git commit -m "feat(ws): add the Krene socket client"
```

---

### Task 7: Boards, task pipeline, image download, path and URL guards

**Files:**
- Create: `src/domain/boards.ts`, `src/domain/tasks.ts`, `src/domain/images.ts`, `src/domain/guards.ts`, `src/domain/lock.ts`
- Test: `test/boards.test.ts`, `test/tasks.test.ts`, `test/guards.test.ts`, `test/lock.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export interface DesignInput { prompt: string; boardId?: string; model?: string; aspectRatio?: string; referenceImages?: string[] }
  export interface ImageInput { prompt: string; boardId?: string; model?: string; aspectRatio?: string; count?: number; referenceImages?: string[] }
  export interface EditInput { imagePath?: string; imageUrl?: string; prompt: string; boardId?: string }
  export interface ExportInput { taskId?: string; urls?: string[]; destDir?: string }
  export interface TaskResult {
    taskId: string
    status: TaskStatus
    text: string
    images: { path: string; url?: string; mimeType: string; width?: number; height?: number }[]
  }
  export interface TaskDeps {
    config: Config
    session: Session
    http: KreneHttp
    openSocket: (handlers: {
      onEvent(event: ServerEvent): void
      onImage(push: { meta: unknown; bytes: Buffer }): void
    }) => Promise<KreneSocket>
  }
  export async function ensureBoard(http: KreneHttp, existing?: string): Promise<{ boardId: string }>
  export async function runDesign(input: DesignInput, deps: TaskDeps): Promise<TaskResult>
  export async function runGenerateImage(input: ImageInput, deps: TaskDeps): Promise<TaskResult>
  export async function runEditImage(input: EditInput, deps: TaskDeps): Promise<TaskResult>
  export async function downloadImages(urls: string[], config: Config, session: Session): Promise<string[]>
  export function resolveWithin(root: string, candidate: string): string
  export function isAllowedKreneUrl(url: string): boolean
  export async function withSessionLock<T>(config: Config, fn: () => Promise<T>): Promise<T>
  ```
- `resolveWithin` throws `UserError('path outside allowed roots')` for `..` or absolute paths outside the root.
- `isAllowedKreneUrl` accepts hosts `www.krene.com`, `ng.krene.com`, and hosts ending in `.krene.com`; rejects `file:`, `data:`, and any other host.
- `withSessionLock` uses `fs.open(lockPath, 'wx')`; on `EEXIST` throws `UserError('BUSY: another krene-mcp process is generating')`; removes the lock in `finally`; treats a lock older than 30 minutes as stale.

- [ ] **Step 1: Write the failing tests**

Imports: `tempHome` from `./helpers.ts`; `loadConfig`, `Config`, `Session`.

```ts
it('rejects path traversal', () => {
  expect(() => resolveWithin('/out', '/out/a/../../etc/passwd')).toThrow(/outside allowed roots/)
})

it('allows only Krene export hosts', () => {
  expect(isAllowedKreneUrl('https://www.krene.com/x')).toBe(true)
  expect(isAllowedKreneUrl('https://cdn.krene.com/x')).toBe(true)
  expect(isAllowedKreneUrl('https://evil.example/x')).toBe(false)
})

it('returns BUSY while the lock is held', async () => {
  const config = loadConfig({ KRENE_HOME: tempHome() } as NodeJS.ProcessEnv)
  const first = withSessionLock(config, async () => {
    await expect(withSessionLock(config, async () => {})).rejects.toThrow(/BUSY/)
  })
  await first
})
```

Task replay test: feed the sanitized `generate-image.json` into a fake socket and HTTP; assert the command order is `getConversionId`, `analysis`, `executeTask`, and the result contains one image path.

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm vitest run test/guards.test.ts test/lock.test.ts test/tasks.test.ts`
Expected: FAIL — modules do not exist.

- [ ] **Step 3: Implement the domain layer**

`ensureBoard` uses `/krene-backend-ts/api/mood/board/create` when no id is supplied, then `/api/mood/board/list` to verify. `runDesign` and the image commands follow the recorded sequence; wait for `COMPLETED`/`CANCELED`/`EXCEPTION` with `config.toolTimeoutMs`. `downloadImages` sends the auth headers, streams to a temp file, renames on completion, and names files `<sha256>.<ext>`.

- [ ] **Step 4: Run tests**

Run: `pnpm vitest run test/guards.test.ts test/lock.test.ts test/tasks.test.ts test/boards.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/domain test/guards.test.ts test/lock.test.ts test/tasks.test.ts test/boards.test.ts
git commit -m "feat(domain): run the Krene task pipeline and guard paths"
```

---

### Task 8: Expose the MCP tools

**Files:**
- Create: `src/mcp/server.ts`, `src/mcp/tools.ts`, `src/mcp/result.ts`
- Modify: `src/cli.ts` (`serve`)
- Test: `test/mcp-tools.test.ts`

**Interfaces:**
- Produces:
  ```ts
  export interface KreneService {
    status(): Promise<{ user: string; points?: number }>
    design(input: DesignInput): Promise<TaskResult>
    generateImage(input: ImageInput): Promise<TaskResult>
    editImage(input: EditInput): Promise<TaskResult>
    task(taskId: string): Promise<TaskResult>
    export(input: ExportInput): Promise<{ paths: string[] }>
  }
  export function createKreneServer(service: KreneService): McpServer
  export function toToolResult(result: TaskResult): CallToolResult
  ```
- Tools: `krene_login` (status only), `krene_status`, `krene_design`, `krene_generate_image`, `krene_edit_image`, `krene_task`, `krene_export`.
- `toToolResult` returns text JSON always, plus `image` blocks for files under 4 MB.
- `krene_export` validates `destDir` with `resolveWithin` and every URL with `isAllowedKreneUrl` before I/O. `krene_edit_image` validates `imagePath` with `resolveWithin`.

- [ ] **Step 1: Write the failing test**

Start the server over an in-memory or stdio transport, connect with the SDK client, and assert:

```ts
const names = (await client.listTools()).tools.map(tool => tool.name).sort()
expect(names).toEqual([
  'krene_design', 'krene_edit_image', 'krene_export',
  'krene_generate_image', 'krene_login', 'krene_status', 'krene_task',
])

const status = await client.callTool({ name: 'krene_status', arguments: {} })
expect(status.content[0]).toMatchObject({ type: 'text' })
```

Also assert `krene_generate_image` advertises `referenceImages` as an array in its `inputSchema`, and that `krene_export` rejects `https://evil.example/x` and `../../etc/passwd` without calling the service.

Add an image case: write a 1×1 PNG to a temp directory, make the fake service return it as `images[0].path`, call `krene_generate_image`, and assert the result contains one `{ type: 'image', mimeType: 'image/png' }` block plus text JSON. A result with a file over 4 MB (fake the size) must return text only.

- [ ] **Step 2: Run test to verify it fails**

Run: `pnpm vitest run test/mcp-tools.test.ts`
Expected: FAIL — module does not exist.

- [ ] **Step 3: Implement the server and tools**

Use the SDK's `McpServer` and `registerTool` with zod shapes. Each tool catches known errors and returns `{ isError: true, content: [{ type: 'text', text: code }] }` for `AUTH_REQUIRED`, `AUTH_EXPIRED`, `QUOTA_EXHAUSTED`, `BUSY`, and `PROTOCOL_DRIFT`. `serve` wires `KreneService` from config + session and connects `StdioServerTransport`.

- [ ] **Step 4: Run tests**

Run: `pnpm vitest run test/mcp-tools.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/mcp src/cli.ts test/mcp-tools.test.ts
git commit -m "feat(mcp): expose Krene design and image tools"
```

---

### Task 9: Live smoke test and client documentation

**Files:**
- Create: `test/live.test.ts`
- Modify: `README.md`
- Test: `test/live.test.ts` (gated by `KRENE_LIVE=1`)

**Interfaces:**
- Live test: load the session, run `runGenerateImage` with a short prompt, assert one image file exists under `config.outputDir`, and print the path. Skip with `it.skipIf(process.env.KRENE_LIVE !== '1')`.

- [ ] **Step 1: Write the gated live test**

Imports: `KreneSocket`, `buildSocketUrl`, `assertSession`, `KreneHttp`, `ensureBoard`, `runGenerateImage`, `loadConfig`, `TaskDeps`.

```ts
it.skipIf(process.env.KRENE_LIVE !== '1')('generates one image with the saved session', async () => {
  const config = loadConfig()
  const session = assertSession(config)
  const http = new KreneHttp({ config, session })
  const board = await ensureBoard(http)
  const deps: TaskDeps = {
    config,
    session,
    http,
    openSocket: async handlers => {
      const socket = new KreneSocket({
        url: buildSocketUrl({ base: config.socketBase, boardId: board.boardId, token: session.token }),
        timeoutMs: config.requestTimeoutMs,
        ...handlers,
      })
      await socket.open()
      return socket
    },
  }
  const result = await runGenerateImage(
    { prompt: 'a simple red circle on white', count: 1, boardId: board.boardId },
    deps,
  )
  expect(result.images.length).toBeGreaterThan(0)
  expect(fs.existsSync(result.images[0].path)).toBe(true)
  console.log(result.images[0].path)
}, 180_000)
```

- [ ] **Step 2: Run the test**

Run: `KRENE_LIVE=1 pnpm vitest run test/live.test.ts`
Expected: PASS and a file path printed. This consumes Krene credits; run it once.

- [ ] **Step 3: Document Claude Code, Cursor, and ds-agent**

README sections: install (local `pnpm build && pnpm link --global`), `krene-mcp login`, session location, tool list, three client config snippets, security boundaries, `KRENE_LIVE` note, the same-process-only task continuation limit, one-generation-at-a-time `BUSY` behavior, and the fact that this uses a private protocol and may break on Krene updates.

- [ ] **Step 4: Run the full gates**

Run: `pnpm typecheck && pnpm test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add test/live.test.ts README.md
git commit -m "test: add the live smoke test and client docs"
```

---

## Self-Review Notes

- Spec §6.1 maps to tasks: scaffold (Task 1), capture (Tasks 2–3), login/session/HTTP (Task 4), protocol (Task 5), socket (Task 6), domain + guards + lock (Task 7), tools (Task 8), live/docs (Task 9).
- Spec §7 Phase 0 is Tasks 2–3 and is a hard gate before Task 5.
- Spec §6.2 is planned separately in `2026-11-04-ds-agent-mcp-client.md`.
- Review Focus tests: `AUTH_EXPIRED` (Task 6), `PROTOCOL_DRIFT` (Task 5), truncated frame (Task 5), path/URL guards (Task 7), `BUSY` (Task 7), `0600` (Task 4).
- Type consistency: `Config` and `Session` are defined in Tasks 1 and 4 and imported by later tasks; `TaskResult` is defined once in Task 7 and used by Task 8.
