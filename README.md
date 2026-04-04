# chrome-devtools-capturer

> 本仓库是 [chrome-devtools-capturer-repo](https://github.com/itMrBoy/chrome-devtools-capturer-repo) 的子模块，作为 Claude Code Plugin 独立发布。
>
> 主仓库包含 Plugin Marketplace 配置和 Chrome 浏览器扩展，完整安装请参考主仓库 README。

## 这是什么

Claude Code 的浏览器调试 Plugin —— 通过 MCP Server + Chrome 扩展，让 Claude 能够直接捕获和分析浏览器运行时数据（网络请求、控制台日志、性能 Tracing）。

v1.1 新增了 `wait_for_capture_result` MCP Tool，实现 prepare → wait → analyze → cleanup 自动闭环。用户在浏览器中完成录制后，数据自动流入 Claude 分析，无需手动确认。

本仓库包含：

- **Skill 定义**（`skills/chrome-devtools-capturer/SKILL.md`）— Claude 识别浏览器调试场景时自动触发
- **MCP Server**（`scripts/mcp-server/`）— 通过 WebSocket 桥接 Claude 与 Chrome 扩展
- **Plugin 清单**（`.claude-plugin/plugin.json`）— Claude Code 插件元信息
- **MCP 配置**（`.mcp.json`）— MCP Server 自动注册

## 项目结构

```
chrome-devtools-capturer/          ← 本仓库（Plugin）
├── .claude-plugin/
│   └── plugin.json
├── .mcp.json
├── skills/
│   └── chrome-devtools-capturer/
│       └── SKILL.md
├── scripts/
│   └── mcp-server/
│       ├── start.js               # 入口，自动安装依赖
│       └── index.js               # MCP Server 主逻辑
├── LICENSE
└── README.md
```

## 自动化改造

### 新增 MCP Tool: `wait_for_capture_result`

- **作用**：在 `prepare_capture_session` 之后自动阻塞等待，直到 Chrome 扩展上报的捕获数据到达 MCP Server
- **原因**：之前用户需要在浏览器操作完成后回到 Claude 手动确认，中断了工作流。新 Tool 通过轮询 + MCP Progress Notification 保活机制，实现最长 10 分钟的自动等待
- **技术实现**：每 2 秒轮询 `latest_trace.json`，同时通过 `sendProgress` 重置 MCP 客户端超时计时器

### 新增 `pendingConfig` 配置暂存机制

- **作用**：MCP Server 暂存最新配置，当 Chrome 扩展的 Service Worker 重连时自动下发
- **原因**：Chrome Manifest V3 的 Service Worker 可能被挂起，醒来重连后原先的配置已丢失

## License

MIT — 见 [LICENSE](LICENSE)。
