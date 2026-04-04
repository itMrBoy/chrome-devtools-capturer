---
name: chrome-devtools-capturer
description: >
  EXCLUSIVE browser debugging skill. Invoke FIRST for any browser-side issue: white screen, JS error, network error,
  console exception, performance problem. Orchestrates capture-analyze-cleanup pipeline via MCP.
allowed-tools: mcp__chrome-devtools-capturer__prepare_capture_session, mcp__chrome-devtools-capturer__wait_for_capture_result, mcp__chrome-devtools-capturer__analyze_capture_results, mcp__chrome-devtools-capturer__cleanup_vibe_workspace, Agent
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

共 4 个 MCP Tools，形成**自动闭环 + 阅后即焚**的工作流：

| Tool                       | 何时调用                                      |
| -------------------------- | ----------------------------------------- |
| `prepare_capture_session`  | 需要浏览器数据之前，第一步调用                           |
| `wait_for_capture_result`  | prepare 之后立即调用，自动阻塞等待数据到达（替代人工确认）          |
| `analyze_capture_results`  | 手动模式下使用：用户确认完成浏览器操作后调用（自动模式下由 wait_for 替代）|
| `cleanup_vibe_workspace`   | 分析完毕，立即清理，防止旧数据污染下一轮                      |

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

### Step 2 — 自动等待捕获数据

**在 Step 1 给出操作引导后，立即调用 `wait_for_capture_result`：**

```json
{
  "timeout_seconds": 600
}
```

此 Tool 会自动阻塞等待 Chrome 扩展上报数据（每 2 秒轮询一次），**无需等待用户回来确认**。
它通过 MCP progress notification 定期重置超时计时器，支持长时间等待。

**三种返回结果：**

| 结果 | 含义 | 后续动作 |
|---|---|---|
| 返回 JSON 数据 | 捕获成功，数据已就绪 | 进入 Step 3 分析 |
| 返回"部分数据"警告 | 超时但文件已存在（Tracing 可能仍在处理） | 仍可进入 Step 3 分析已有数据 |
| 返回"超时无数据" | 扩展未录制或连接异常 | 向用户展示排查建议 |

---

### Step 3 — 委托 subAgent 分析并清理

**触发时机：** Step 2 返回了捕获数据后，立即执行。

**为什么用 subAgent：** 捕获数据（trace JSON）可能非常大，直接在主对话中读取会消耗大量 token 并污染上下文。委托 subAgent 在隔离上下文中完成分析，只将精简结论返回主对话。

**注意：** 由于 `wait_for_capture_result` 已经返回了数据，subAgent **不需要再调用 `analyze_capture_results`**，直接分析 Step 2 返回的数据即可。

使用 Agent tool 派出 subAgent，prompt 模板：

```
你是浏览器调试分析助手。请分析以下捕获数据并执行清理：

<capture_data>
{这里粘贴 wait_for_capture_result 返回的数据}
</capture_data>

1. **数据质量检查**（在分析前必须先执行）：

   读取 meta.stats 中的 network_count 和 console_count，按以下规则判断：

   a) **数据极少**（network_count == 0 且 console_count == 0）：
      → 判定：高度疑似捕获失败（正常页面至少有 document 请求）
      → 返回结论时标记 data_quality: "empty"

   b) **数据偏少**（network_count < 3 或仅有 console_count 而用户报告的是网络问题）：
      → 判定：可能是录制时间太短、未操作目标功能、或扩展 attach 晚于页面加载
      → 返回结论时标记 data_quality: "sparse"

   c) **数据正常**：
      → 返回结论时标记 data_quality: "ok"

2. 按以下维度分析数据（即使数据偏少也尽量分析已有内容）：

   针对 network_logs：
   - 状态码异常（4xx/5xx）— 列出 URL、status、statusText
   - 耗时过长（durationMs > 1000）— 列出 URL 和耗时
   - 请求头/响应头问题（CORS、认证）
   - 未完成的请求（status: null）

   针对 console_logs：
   - level: "error" 的条目 — 列出 text、url、line
   - source: "javascript" 的异常
   - level: "warning" 中的弃用警告

   针对 performance_logs（如果存在）：
   - 长任务数量和总阻塞时间
   - 耗时最长的子任务及其调用信息

3. 分析完成后，调用 mcp__chrome-devtools-capturer__cleanup_vibe_workspace 清理数据

4. 返回精简的分析报告，必须包含：
   - data_quality 字段（empty / sparse / ok）
   - 如果 data_quality 不是 ok，附带可能原因说明
   - 分析结论（直接引用原始数据字段值作为证据，避免模糊结论）
```

**主 agent 收到 subAgent 结论后，根据 data_quality 给出用户提示：**

- `data_quality: "empty"` → 告知用户录制期间未捕获到任何数据，建议：
  1. 确认录制时扩展图标是否变为红色 "REC"
  2. 确认操作的是目标页面（非其他 Tab）
  3. 尝试使用 `action_mode: "reload"` 从页面加载开始捕获

- `data_quality: "sparse"` → 告知用户数据较少，分析结论可能不完整，建议：
  1. 延长录制时间，确保完整执行了目标操作
  2. 如果问题发生在页面加载阶段，改用 `action_mode: "reload"`
  3. 如果当前结论已足够定位问题，可以直接使用

- `data_quality: "ok"` → 正常展示分析结论

**如果 Step 2 返回超时无数据（未进入 Step 3）：** 直接向用户展示 wait_for_capture_result 返回的排查建议，询问是否重试。

主 agent 收到 subAgent 返回的分析结论后，根据结果向用户提出修复建议或下一步行动。

---

## 数据结构速查（供 subAgent 分析参考）

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
├── console_logs[]
│   ├── level             ← verbose | info | warning | error
│   ├── source            ← javascript | network | console-api | ...
│   ├── text / url / line
│   └── timestamp
└── performance_logs（可选）
    ├── summary
    │   ├── totalLongTasks / totalBlockingTimeMs
    │   └── mainThread { pid, tid }
    └── longTasks[]
        ├── durationMs / blockingTimeMs
        └── heavySubTasks[] { name, durationMs, callInfo }
```

---

## 常见场景示例

### 场景 A：接口报错

```
用户：页面加载后 /api/user 一直 500，帮我看看
→ prepare_capture_session(target="http://...", types=["network"], action_mode="manual")
→ 引导用户操作后，立即调用 wait_for_capture_result(timeout_seconds=120)
→ 数据自动到达 → 派出 subAgent 分析 + 清理
→ 主 agent 根据结论给出修复建议
```

### 场景 B：JS 异常定位

```
用户：点击按钮后白屏了，控制台应该有报错
→ prepare_capture_session(target="http://...", types=["console"], action_mode="manual")
→ wait_for_capture_result 自动等待录制完成
→ subAgent 分析 → 找 level:"error" + url + line 定位源码
→ 主 agent 定位代码并修复
```

### 场景 C：不确定问题类型

```
用户：这个功能感觉怪怪的，帮我捕获一下
→ prepare_capture_session(target="...", types=["network","console"], action_mode="manual")
→ wait_for_capture_result 自动等待
→ subAgent 全面分析后返回精简结论
→ 主 agent 综合判断问题根因
```
