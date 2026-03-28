---
name: chrome-devtools-capturer
description: >
  EXCLUSIVE browser debugging skill. Invoke FIRST for any browser-side issue: white screen, JS error, network error,
  console exception, performance problem. Orchestrates capture-analyze-cleanup pipeline via MCP.
allowed-tools: mcp__chrome-devtools-capturer__prepare_capture_session, mcp__chrome-devtools-capturer__analyze_capture_results, mcp__chrome-devtools-capturer__cleanup_vibe_workspace
---

# chrome-devtools-capturer — MCP 工具使用指南

## 触发场景

当用户描述以下任何情况时，**主动调用**此 skill，无需等用户指名 MCP 工具：

- 白屏、页面崩溃、按钮点击没反应
- 接口报错（404、500、CORS、超时）
- JS 异常、控制台报错
- 性能问题（加载慢、接口慢）
- "帮我抓一下网络请求"、"看看控制台有没有报错"、"capture network logs"、"check console errors"
- Vibe coding 过程中需要浏览器运行时数据作为证据

## MCP Server 依赖

本 Skill 依赖一个 MCP Server，启动信息：

- **command**: `node`
- **args**: `./scripts/mcp-server/start.js`

> 如果通过 plugin 安装，MCP Server 会通过 `.mcp.json` 自动注册。
> 如果工具调用失败，提示用户检查 IDE 的 MCP 配置中 `chrome-devtools-capturer` 是否已连接。

## 工具概览

共 3 个 MCP Tools，形成**阅后即焚**的闭环：

| Tool                      | 何时调用                 |
| ------------------------- | -------------------- |
| `prepare_capture_session` | 需要浏览器数据之前，第一步调用      |
| `analyze_capture_results` | 用户完成浏览器操作、数据已上报之后    |
| `cleanup_vibe_workspace`  | 分析完毕，立即清理，防止旧数据污染下一轮 |

---

## 标准工作流

### Step 1 — 调用 `prepare_capture_session`

**参数填写策略：**

```json
{
  "target": "页面 URL 或 'extension'（调试扩展本身时）",
  "types": ["network", "console"],
  "action_mode": "manual"
}
```

- `target`：从用户描述或当前代码推断目标页面 URL；不确定时询问
- `types`：根据问题类型选择
  - 网络问题 → `["network"]`
  - JS 报错 → `["console"]`
  - 不确定 → `["network", "console"]`（默认两者都抓）
  - 性能 → `["network", "console", "performance"]`
- `action_mode`：捕获触发模式
  - `"manual"` → 用户手动按快捷键开始/停止录制（默认推荐）
  - `"reload"` → attach 后自动刷新页面，从 navigationStart 起完整捕获

**调用后，给用户明确的操作引导：**

```
配置已下发给 Chrome 扩展。请在浏览器中：
1. 确认扩展图标显示 "RDY"（琥珀色徽标）
2. 切换到目标页面
3. 按 Alt+Shift+C 开始录制（图标变红 "REC"）
4. 执行你想捕获的操作
5. 再次按 Alt+Shift+C 停止录制
6. 回来告诉我已完成操作
```

---

### Step 2 — 等待用户操作浏览器

这一步不调用任何工具。等待用户说"好了"、"操作完了"、"录制结束"等信号。

如果用户说扩展没有反应：

- 提醒检查 MCP Server 是否在运行：`node mcp-server/index.js`
- 提醒扩展是否已加载（`chrome://extensions/` 开发者模式）
- 扩展图标应显示 "RDY" 才能录制

---

### Step 3 — 调用 `analyze_capture_results`

**触发时机：** 用户确认已完成浏览器操作后，立即调用。

无参数，直接读取 `.vibeDevtools/latest_trace.json`。

**拿到数据后的分析重点：**

针对 **network_logs**：

- 找状态码异常（4xx/5xx）
- 找耗时过长（`durationMs > 1000`）
- 找请求头/响应头问题（CORS、认证）
- 找未完成的请求（`status: null`）

针对 **console_logs**：

- 优先看 `level: "error"` 的条目
- 关注 `source: "javascript"` 的异常，结合 `url` 和 `line` 定位源码
- 看 `level: "warning"` 是否有预期外的弃用警告

分析时直接引用原始数据中的字段值作为证据，避免模糊结论。

---

### Step 4 — 调用 `cleanup_vibe_workspace`

**触发时机：** 分析完成后，紧接着调用，无需用户要求。

> 这是"阅后即焚"原则的执行点。不清理旧数据会导致下一轮 `analyze_capture_results` 读到过期结果，让 LLM 产生错误分析。

---

## 数据结构速查

```
latest_trace.json
├── meta
│   ├── capturedAt        ← 捕获时间戳
│   ├── config            ← 本次下发的配置
│   └── stats             ← { network_count, console_count }
├── network_logs[]
│   ├── url / method / type
│   ├── status / statusText / mimeType
│   ├── startTime / durationMs
│   └── headers           ← Authorization/Cookie 已自动脱敏为 [MASKED]
└── console_logs[]
    ├── level             ← verbose | info | warning | error
    ├── source            ← javascript | network | console-api | ...
    ├── text / url / line
    └── timestamp
```

---

## 常见场景示例

### 场景 A：接口报错

```
用户：页面加载后 /api/user 一直 500，帮我看看
→ prepare_capture_session(target="http://...", types=["network"], action_mode="manual")
→ 引导用户录制页面加载
→ analyze_capture_results() → 找 /api/user 的 status 和响应头
→ cleanup_vibe_workspace()
```

### 场景 B：JS 异常定位

```
用户：点击按钮后白屏了，控制台应该有报错
→ prepare_capture_session(target="http://...", types=["console"], action_mode="manual")
→ 引导用户录制点击操作
→ analyze_capture_results() → 找 level:"error" + url + line 定位源码
→ cleanup_vibe_workspace()
```

### 场景 C：不确定问题类型

```
用户：这个功能感觉怪怪的，帮我捕获一下
→ prepare_capture_session(target="...", types=["network","console"], action_mode="manual")
→ 两类数据都抓，全面分析
→ cleanup_vibe_workspace()
```
