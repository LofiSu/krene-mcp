# Krene MCP 服务与 ds-agent MCP 客户端 — 设计

- 日期：2026-11-04
- 状态：已确认（2026-11-04）
- 归属：LofiSu
- 涉及仓库：新仓库 `krene-mcp`；`deepseek-super-agent` 新增 M1 MCP 客户端模块

## 1. 意图与为什么

`deepseek-super-agent` 目前只有文本工具，没有生图和设计能力。Krene（https://www.krene.com/design/）提供对话式角色/世界观设计和生图（Nano Banana Pro、Seedream 3.1、GPT Image 2），但没有公开 API。目标是：

1. 做一个独立的 stdio MCP server `krene-mcp`，任何 MCP 客户端（Claude Code、Cursor、ds-agent 等）都能挂载，用**用户自己的 Krene 账号**做设计和生图。
2. 给 ds-agent 增加一个**通用 MCP 客户端模块**，让 worker 能调用任意 MCP server；Krene 只是第一个接入方。

成功标准：在 ds-agent 的一次 worker 任务里调用一个工具，就能拿到 Krene 生成的设计/图片，图片落盘并在回复里给出可打开的路径；同一个 `krene-mcp` 不经过 ds-agent 也能在 Claude Code 里直接使用。

## 2. 目标、非目标、反目标

### 目标

- `krene-mcp` 提供登录、会话校验、对话式设计、生图、改图、任务查询、图片导出。
- 登录自动化：首次 `login` 打开浏览器，用户手动完成验证码；程序自动捕获 token 并保存。之后运行时**不再启动浏览器**。
- 运行时走 Krene 原生 HTTP + WebSocket 协议，不依赖 DOM 操作。
- ds-agent 通过配置挂载 MCP server，worker 获得命名空间化的外部工具；router 保持轻量，不暴露这些工具。
- 协议层用真实抓包 fixture 做离线测试；网络测试显式开关。
- 失败可诊断：会话过期、额度不足、任务失败、协议变更都有明确错误。

### 非目标

- 不做视频生成。Krene 本身不生视频（其官方教程也是转去即梦 Seedance 2.0）；本次不接即梦。
- 不做多用户、凭证托管、托管式 SaaS。
- 不做批量抓取、绕过额度、自动破解验证码。
- 不做图形界面。
- 不追求 Krene 全功能覆盖；只覆盖设计对话和生图/改图所需的最小面。

### 反目标（即使“能跑”也算失败）

- 把账号密码写进配置或仓库。
- 自动识别/绕过阿里云验证码。
- 运行时偷偷退回浏览器自动化却仍声称“原生协议”。
- 把 token、cookie、真实用户标识写进 fixture 或日志。
- 因为接 MCP 而让 router 拿到重型工具，破坏 §2 的“Router stays light”。
- 让一个 MCP server 启动失败拖垮整个 Runtime。

## 3. 约束

- Node.js 22+，TypeScript，pnpm。MCP 用官方 `@modelcontextprotocol/sdk`。
- Krene 无公开 API，协议来自前端 bundle 与真实抓包；属于私有协议，页面改版会变。账号条款风险由仓库所有者承担，仅限本人账号、正常使用频率。
- 登录是手机/邮箱验证码 + 阿里云人机验证，纯代码无法完成，必须保留一次人工登录。
- ds-agent 仓库规则：owner-only、一个 PR 一个关注点、从当前 `main` 开分支、`pnpm typecheck` / `pnpm lint` / `pnpm test` 必须过、按 M0–M11 命名、不新增功能开关。

## 4. 已核实的事实

来自 `https://www.krene.com/design/` 的 `main.13102f61.js` 与公开页面：

- 应用是 React SPA，构建版本写在 `<meta name="build-version">` 和 `buildVersion` cookie。
- 登录接口：`/krene-backend-ts/auth/sendSMS`、`/verifyCaptchaAndSendCode`、`/loginOrRegisterWithCode`。
- 会话保存在 `localStorage.userInfo`，其中含 `token` 和用户标识。
- HTTP 认证头：`K-Authorization: Bearer <token>`、`krene_user: <标识>`、`X-Language`。
- 画布同步 WebSocket：`wss://ng.krene.com/qa_psrooma/room?roomId=<boardId>&token=<token>`（tldraw 同步协议）。
- 设计 agent 的另一个 socket 地址写在配置里：`wss://ng.krene.com/qa_psa/moodWs`。
- 设计 agent 用的是**原生 WebSocket**（不是 socket.io）：
  - 文本帧是 JSON，服务端响应形如 `{ code: 200, trigger, data }`；`code=4001` 表示会话异常。
  - 二进制帧是服务端推送的图片：前 8 字节大端是图片字节数 `r`，随后是 `n-r-8` 字节的 JSON 元数据，最后 `r` 字节是图片本体。
  - 心跳是每 30 秒 `{ command: "heartMessage", messageId }`。
- 客户端命令枚举：`getConversionId`、`sendPrompt`、`analysis`、`executeTask`、`generateTags`、`fetchImages`、`generateImages`、`generatePlan`、`editImages`、`parserTaskCard`、`pushImages`、`chat`、`batch`、`genSuggestions`。
- 已见到的命令载荷：
  - `getConversionId`：`{ command, messageId, prompt }`
  - `analysis`：`{ command, messageId, params: { conversionId, prompt, imageUuid, parentStepId, chatOn, strategyCode, lang } }`
  - `executeTask`：`{ command, messageId, params: { conversionId, tasks, parentStepId } }`
  - `pushImages` / `genSuggestions`：`{ command, messageId, params }`
- 服务端事件枚举：`ADD_TASK_CARD`、`UPDATE_TASK_CARD_STATUS`、`CONVERSION_CREATE`、`UPDATE_CONVERSION_STATUS`、`SERVER_PUSH_IMAGE`、`UPDATE_STEP_STATUS`、`ADD_BATCH`、`CONNECT_SUCCESS`、`ADD_SUGGESTION`、`LOCK`、`UNLOCK`。
- 任务状态：`PENDING EXECUTION`、`EXECUTING`、`CONFIRMING`、`PENDING`、`COMPLETED`、`CANCELED`、`EXCEPTION`。
- 任务模型是“对话 → 分析 → 任务卡片 → 执行”：`krene_generate_image` 不是单一 RPC，而要走 `getConversionId` → `analysis` → `executeTask`，再监听 `UPDATE_TASK_CARD_STATUS`。
- 页面 bundle 里没有找到 `one.connect(...)` 的调用点，说明首次连接 URL/握手可能由别处触发；这是 Phase 0 必须抓到的第一件事。
- 生图模型名：Nano Banana Pro、Seedream 3.1、GPT Image 2。
- 图片上传相关：`/imagehub/api/image/canvas-upload-url`、`/kanvas-api/api/v1/image/temporary-url-from-url`。
- 看板接口：`/krene-backend-ts/api/mood/board/create`、`/api/mood/board/list`。
- 会员/额度：`/krene-backend-ts/api/member/info?userIdentifier=<id>`、`/api/member/points-detail`。

## 5. 决策

### 用户已决定

1. 只做 Krene 的生图和设计，不做视频。
2. 逆向 WebSocket 协议，而不是长期走 UI 自动化。
3. 登录 token 自动获取（用户只手动完成验证码）。
4. ds-agent 接入方式选 A：通用 MCP 客户端，不是 Krene 专用工具。

### 我的判断（可在评审时推翻）

- `krene-mcp` 是独立仓库，TypeScript，单包，不塞进 ds-agent 的 monorepo。
- MCP 传输先只做 stdio；HTTP 传输留到有远程需求时。
- 工具只暴露给 worker；router 不挂 MCP 工具。
- 工具命名用 `mcp__<server>__<tool>`，避免与内置工具冲突。
- 会话文件放 `~/.krene-mcp/session.json`，权限 0600；可用 `KRENE_HOME` 覆盖。
- 生成结果默认落盘到 `~/.krene-mcp/output/<yyyy-mm-dd>/<taskId>/`。
- 长任务默认阻塞等待 120 秒；超时返回 `taskId`，由 `krene_task` 继续查。
- 协议抓包和实现分两个阶段；没有真实 fixture 不写协议实现。
- **登录只走 CLI**。MCP 工具不弹浏览器、不等待人工输入；worker 是后台回合，没人能应。
- **图片以路径交付**。ds-agent 的模型消息和通道都是纯文本，v1 不内联图片；把图片发到 IM 是独立的通道媒体项目。
- **MCP 参数 schema 全量透传**。ds-agent 的 `ToolSpec.parameters` 放宽为完整 JSON Schema，保证"任何 MCP server"这个说法成立。
- **输入边界默认收紧**：本地路径限定在允许根目录，导出 URL 限定 Krene 域名。
- **子进程环境用白名单**，不是"什么都不传"。
- **任务续查与并发**：v1 只保证同进程续查；同一会话同时只允许一个生成（文件锁）。
- **v1 用本地构建 + link**，npm 发布等你决定；Playwright 只在登录/抓包时需要。

## 6. 架构

### 6.1 `krene-mcp`

```text
krene-mcp/
  src/
    cli.ts                 # login / capture / status / serve
    auth/
      session.ts           # 读写、校验、过期判断
      login.ts             # Playwright 一次性登录与 token 捕获
    http/
      client.ts            # fetch 封装、认证头、错误映射
    ws/
      client.ts            # socket 连接、重连、帧编解码
    protocol/
      commands.ts          # 命令枚举与 payload 构造
      events.ts            # 事件解析、任务状态机
      schema.ts            # zod 校验，协议漂移尽早报错
    domain/
      boards.ts            # 看板创建/选择
      tasks.ts             # 任务提交、轮询、结果
      images.ts            # 下载、去重、落盘
    mcp/
      server.ts            # MCP server 装配
      tools/               # 每个工具一个文件
    config.ts              # 路径、超时、日志级别
    log.ts                 # 结构化日志与脱敏
  test/                    # fixture 回放，默认无网络
  fixtures/                # 脱敏后的抓包
  docs/superpowers/specs/  # 本文件
```

**登录流程**

1. `krene-mcp login` 启动 Playwright Chromium（有头，走系统代理设置）。
2. 打开 `https://www.krene.com/design/`，用户在页面里完成手机/邮箱验证码。
3. 程序每秒读取 `localStorage.userInfo`；拿到 `token` 和用户标识后，再用 `/api/member/info` 校验一次。
4. 写入 `~/.krene-mcp/session.json`，包含 `{ token, userId, displayName?, savedAt, source }`；不保存密码、验证码、cookie 全文。
5. 关闭浏览器。

**运行时流程（设计/生图）**

1. `krene_status` 校验 token；401 则提示重新 `login`。
2. 选择或创建 board，得到 `boardId`。
3. 连接设计 agent socket（地址与握手以 Phase 0 抓包为准；候选是 `wss://ng.krene.com/qa_psa/moodWs`）。
4. `getConversionId` 创建会话，拿 `conversionId`。
5. `analysis` 发送用户提示，拿回分析和任务卡片；必要时 `executeTask` 执行选中的任务。
6. 监听 `UPDATE_TASK_CARD_STATUS` / `UPDATE_CONVERSION_STATUS`，直到 `COMPLETED`、`CANCELED` 或 `EXCEPTION`。
7. 结果可能以 JSON 里的图片 URL 返回，也可能以二进制 `SERVER_PUSH_IMAGE` 帧返回；两种都要处理。
8. 图片落盘到输出目录，返回本地路径、原始 URL、尺寸和任务 ID。
9. 连接空闲超过阈值后关闭；同一进程内复用。

**会话文件 schema**

```json
{
  "token": "...",
  "userId": "...",
  "displayName": "...",
  "savedAt": "2026-11-04T12:00:00.000Z",
  "source": "playwright"
}
```

**任务生命周期**：v1 只保证**同一进程内**可续查。超过等待窗口返回的 `taskId` 在该 `krene-mcp` 进程存活期间可用；进程退出后不保证恢复。若 Phase 0 发现任务能只靠 `taskId` 从服务端拉取，则再加持久化恢复；否则文档明确写“重启后请重新发起”。不做跨进程恢复，避免为不确定的协议承诺大量存储工作。

**并发**：v1 不支持同一会话文件的并发生成。用 `~/.krene-mcp/lock` 做进程级文件锁；第二个实例在锁被占用时返回 `BUSY`，而不是创建重复 board 或并发扣额度。多个 MCP 客户端同时使用属于未支持场景，文档写明。

**安装与查找**：v1 用本地构建 + `pnpm link --global`（或 `npm link`）提供 `krene-mcp` 命令；是否发布 npm 由用户决定。Playwright 是**可选**依赖，只有 `login` / `capture` 需要；未安装时给出明确提示，运行时不需要浏览器。README 分别给出 Claude Code、Cursor、ds-agent 的配置片段，命令用绝对路径或 `npx`，不假设 `PATH` 一定包含全局 bin。

**错误分类**

- `AUTH_REQUIRED`：无会话或 401 → 提示运行 `krene-mcp login`。
- `AUTH_EXPIRED`：token 过期 → 同上。
- `QUOTA_EXHAUSTED`：额度不足 → 原样返回 Krene 文案。
- `PROTOCOL_DRIFT`：schema 校验失败 → 记录 buildVersion 和未知帧摘要。
- `TASK_FAILED`：任务失败 → 返回任务 ID 和错误消息。
- `TIMEOUT`：等待超时 → 返回 `taskId`。
- `NETWORK`：连接/下载失败 → 可重试。

**MCP 工具（初版，字段以抓包后为准）**

| 工具 | 输入 | 输出 |
|------|------|------|
| `krene_login` | 无 | **只报告会话状态**；未登录时返回 `AUTH_REQUIRED` 和“在主机运行 `krene-mcp login`”的提示，不启动浏览器 |
| `krene_status` | 无 | 登录状态、用户、额度 |
| `krene_design` | `prompt`、可选 `boardId`、`model`、`aspectRatio`、`referenceImages[]` | 设计回复文本、图片列表、`taskId` |
| `krene_generate_image` | `prompt`、可选 `model`、`aspectRatio`、`count`、`referenceImages[]`、`boardId` | 图片列表或 `taskId` |
| `krene_edit_image` | `imagePath` 或 `imageUrl`、`prompt`、可选 `boardId` | 图片列表或 `taskId` |
| `krene_task` | `taskId` | 状态、文本、图片列表 |
| `krene_export` | `taskId` 或 `urls[]`、可选 `destDir` | 本地文件路径 |

图片结果同时返回 MCP `image` content（小于 4 MB 时）和 `text` JSON（路径、URL、尺寸）。

**登录职责**：启动浏览器只发生在 CLI 的 `krene-mcp login`。MCP 工具永远不会弹出浏览器或等待人工输入——ds-agent 的 worker 是后台 ReAct 回合，没有人可以应。`krene_login` 因此是状态/提示工具，不是交互工具。

**结果交付**：ds-agent 的模型消息和现有通道都是纯文本，v1 不把图片内联进回复。状态与图片都以**本地文件路径 + 元数据**返回；真正把图片发到 Feishu/IM 属于通道媒体能力，另立项目。MCP server 本身仍可按客户端能力返回 `image` content；ds-agent 客户端会把图片块落盘并转成路径。

**输入边界（安全）**：
- `krene_edit_image.imagePath` 只允许 `KRENE_INPUT_ROOTS`（默认 `~/.krene-mcp/output` 和 worker 工作目录）内的文件；越界直接拒绝。
- `krene_export.destDir` 默认且只能落在 `~/.krene-mcp/output` 或显式允许的根目录。
- `krene_export.urls[]` 只允许 Krene 结果域名（`www.krene.com`、`ng.krene.com` 及其对象存储域名），拒绝任意 URL，避免被提示注入当成 SSRF/上传跳板。
- 不读取 `~/.ssh`、环境变量文件等敏感路径；错误信息不回显绝对路径以外的内容。

### 6.2 ds-agent MCP 客户端（M1 Harness）

新增 `src/harness/mcp.ts`，职责：

- 从 `RuntimeOptions.mcpServers` 或 `DS_AGENT_MCP_SERVERS`（JSON 数组）读取配置：
  ```json
  [{ "name": "krene", "command": "/Users/me/.local/bin/krene-mcp", "args": ["serve"], "timeoutMs": 130000 }]
  ```
  `env` 可选，只用于补充白名单之外需要的变量；`PATH`、`HOME`、`LANG`、`TZ` 和代理变量由客户端自动继承。
- `Runtime.create` 时在后台启动连接；连接失败只记日志和指标，不阻塞 Runtime。
- 首次需要工具列表时等待连接完成（有超时）；`ToolRegistry.specsFor` 改为异步，`McpToolSource.specs()` 在连接未完成时等待。
- 工具名映射为 `mcp__<server>__<tool>`；描述原样透传，前缀注明来源。
- **参数 schema 全量透传**：把 `ToolSpec.parameters` 从扁平的 `properties: Record<string, {type, description}>` 放宽为完整的 JSON Schema 对象（仍是 `type: 'object'`）。provider 目前已经把 `parameters` 原样交给模型，所以这是类型放宽，不是协议改造；`krene_*` 的数组/枚举/required 才能表达。
- 只加入 `worker` 角色的工具列表；router/watchdog 不暴露。
- `invoke` 带超时；结果走现有 `redactUnknown`；超过阈值的结果写入 spill 表，只把引用交给模型。
- **内容归一化**：`text` 直接进入工具消息；`image` / 二进制 `resource` 落盘到 worker 工作目录，消息里只放路径、MIME 和尺寸；单条结果超过 64 KB 走 spill。绝不把 base64 塞进 `JSON.stringify` 的工具结果。
- **环境白名单**：子进程只继承 `PATH`、`HOME`、`LANG`、`TZ` 和代理变量（`HTTP_PROXY`/`HTTPS_PROXY`/`NO_PROXY`），再加上配置里显式写的 `env`。不是"什么都不传"——否则找不到命令、也找不到 `~/.krene-mcp`。密钥不进入日志。
- `Runtime.finish()` 关闭所有 MCP 连接和子进程。
- 指标：`mcp.connect`、`mcp.connect_error`、`mcp.call`、`mcp.call_error`、`mcp.call_timeout`。

模块边界（实现时可调整命名，但职责不变）：

```ts
interface McpServerConfig {
  name: string
  command: string
  args?: string[]
  env?: Record<string, string>
  timeoutMs?: number
}

interface McpToolSource {
  specs(): Promise<ToolSpec[]>          // 连接未完成时等待，超时返回已连上的部分
  invoke(name: string, args: unknown): Promise<unknown>
  close(): Promise<void>
}
```

`ToolRegistry` 只做两件事：`specsFor` 把静态工具和 `mcp.specs()` 合并（仅 `worker`），`invoke` 把 `mcp__` 前缀的调用转给 `mcp.invoke`。MCP 连接失败不抛到 Runtime 构造；调用失败按现有工具错误路径返回给模型。

## 7. Phase 0：协议抓包（必须先做）

目的：把“猜”变成“已知”。没有这一步不进入协议实现。

交付物：

1. `krene-mcp capture`：打开有头浏览器，记录 HTTP 请求/响应和 WebSocket 帧到 `captures/<timestamp>/`。
2. 用户手动完成登录，并在 Krene 里做**一次**设计对话和**一次**生图。
3. 脱敏脚本：去掉 token、cookie、用户标识、手机号/邮箱、图片 URL 中的签名参数。
4. `fixtures/` 下按场景保存：`design-session.jsonl`、`generate-image.jsonl`、`edit-image.jsonl`（一行一帧）。
5. `docs/protocol-notes.md`：记录 socket 选择、握手、消息 schema、任务状态、结果结构、错误码。

验收：能离线回放 fixture，解析出一次完整生图任务的输入、状态流转和结果图片 URL，且不依赖网络。

**技术风险与回退**：设计 agent 命令可能必须经过 tldraw 同步房间；如果是这样，手写同步协议会很重。Phase 0 要确认命令是走独立的 `moodWs` socket，还是必须挂在 tldraw room 上。如果必须完整同步 tldraw 状态，回退方案是：同一个 `krene-mcp` 工具接口下，使用 Playwright 有头/无头驱动页面，而不是手写同步协议。回退会记录在文档里，不会静默切换。

## 8. 阶段与验收

**Phase 0 — 抓包**：见上。需要你在本机手动登录一次，并在 Krene 里做一次设计和一次生图；产出 fixture 和协议笔记。这一步不做完，Phase 2 不开始。

**Phase 1 — `krene-mcp` 骨架与登录**：
- `login` 自动捕获 token；`status` 校验；会话文件 0600。
- 验收：`krene-mcp login` 后 `krene-mcp status` 返回用户信息；没有网络测试。

**Phase 2 — 协议客户端与工具**：
- HTTP/WS 客户端、命令、事件、下载、MCP 工具。
- 验收：fixture 回放测试全绿；`KRENE_LIVE=1` 时用真实账号生成一张图并落盘。

**Phase 3 — ds-agent MCP 客户端**：
- `src/harness/mcp.ts`、配置解析、异步工具列表、完整 JSON Schema 透传、worker 可见、超时与关闭。
- 验收：`pnpm typecheck`、`pnpm lint`、`pnpm test` 通过；新增测试用假 MCP server（本地脚本）验证连接、数组参数、调用、失败隔离。

**Phase 4 — 文档与端到端**：
- README：Claude Code、Cursor、ds-agent 三种挂载示例。
- 一次端到端：ds-agent worker 触发 → 调用 `mcp__krene__krene_generate_image` → 图片落盘 → 回复里给出路径。
- 验收：命令输出和生成文件路径作为证据。

## 9. 风险

| 风险 | 影响 | 缓解 |
|------|------|------|
| Krene 条款不允许自动化 | 账号风险 | 仅本人账号、正常频率、可随时停用；文档明示 |
| 协议/页面改版 | 工具失效 | 协议集中在 `protocol/`；fixture 测试；buildVersion 记录 |
| 登录/验证码改版 | 无法自动拿 token | `login` 失败时给出明确提示，回退手动粘贴 token |
| tldraw 同步协议过重 | 无法原生驱动 | Phase 0 判定；回退 Playwright 驱动，接口不变 |
| 生成消耗额度 | 测试成本 | 默认只跑 fixture；live 测试显式开关并提示额度 |
| token 过期 | 运行中断 | 401 立即报 `AUTH_EXPIRED`，不自动弹窗 |
| MCP 子进程挂起 | worker 卡住 | 连接/调用超时、失败隔离、指标 |
| 外部工具输出泄密 | 隐私 | 沿用 redact；大结果走 spill；不传无关环境变量 |

## 10. Phase 0 要回答的问题（不需要用户回答）

1. 设计 agent 的首次连接究竟打到哪个 URL、带哪些 query/header（bundle 里没找到调用点）。
2. 连接握手是否需要 `boardId`、`krene_user`、共享密钥或额外的 `CONNECT_SUCCESS` 应答。
3. `analysis` 的 `tasks` 结构、`executeTask` 的 `tasks` 结构，以及任务卡片如何选择。
4. `sendPrompt` 与 `getConversionId`/`analysis` 的关系：是否两套入口。
5. 结果图片是 JSON URL、二进制 push，还是两者都有；URL 是否临时签名、下载要不要认证头。
6. `krene_user` 期望的是 uuid、手机号还是邮箱。
7. 生图是否必须先有 board，board 与 conversion 的关系。
8. 额度扣减发生在哪个命令。

## 11. 留给实现者决定

- 具体文件划分以外的内部抽象、重试/退避策略、日志字段。
- MCP SDK 的具体版本与 stdio 装配细节。
- fixture 脱敏脚本的实现方式。
- 图片去重和命名规则。

## 12. 明确不在本次范围

- 视频生成、即梦接入。
- Krene 的画布协作、分享、支付、会员任务。
- 远程托管 MCP（HTTP/SSE）。
- 多 Krene 账号轮换。
