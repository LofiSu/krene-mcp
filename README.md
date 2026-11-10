# krene-mcp

Krene 设计/生图的 MCP server，以及把它接入 `deepseek-super-agent` 的通用 MCP 客户端。

- **状态**：设计阶段，代码尚未开始。
- **设计文档**：[docs/superpowers/specs/2026-11-04-krene-mcp-design.md](docs/superpowers/specs/2026-11-04-krene-mcp-design.md)

## 范围

- `krene-mcp`：独立的 stdio MCP server，任何 MCP 客户端都能挂载；首次用浏览器登录 Krene，之后走原生 HTTP + WebSocket 协议做设计对话、生图、改图和图片导出。
- `deepseek-super-agent`：新增 M1 通用 MCP 客户端模块，让 worker 调用任意 MCP server。

不在本次范围：视频生成、Krene 画布协作/支付、远程托管 MCP、多账号轮换。

## 注意

- 仅使用你自己的 Krene 账号，正常使用频率。
- Krene 没有公开 API；协议来自前端 bundle 与真实抓包，页面改版会影响可用性。
- Token 存在 `~/.krene-mcp/session.json`，不会进入仓库。

## License

MIT，见 [LICENSE](LICENSE)。
