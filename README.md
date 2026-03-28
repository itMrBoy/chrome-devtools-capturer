# chrome-devtools-capturer

> 本仓库是 [chrome-devtools-capturer-repo](https://github.com/itMrBoy/chrome-devtools-capturer-repo) 的子模块，作为 Claude Code Plugin 独立发布。
>
> 主仓库包含 Plugin Marketplace 配置和 Chrome 浏览器扩展，完整安装请参考主仓库 README。

## 这是什么

Claude Code 的浏览器调试 Plugin —— 通过 MCP Server + Chrome 扩展，让 Claude 能够直接捕获和分析浏览器运行时数据（网络请求、控制台日志、性能 Tracing）。

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

## License

MIT — 见 [LICENSE](LICENSE)。
