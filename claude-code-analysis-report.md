# Claude Code 源码框架结构化分析报告

> 分析对象：`claude-code-main/src/`
> 文件总量：1904 个
> 报告日期：2026-03-31
> 分析范围：完整 `src/` 目录，含所有子模块

---

## 目录

1. [项目概览](#1-项目概览)
2. [整体架构](#2-整体架构)
3. [模块详解](#3-模块详解)
   - 3.1 启动与引导层
   - 3.2 核心引擎层
   - 3.3 工具系统
   - 3.4 命令系统
   - 3.5 状态管理
   - 3.6 UI 渲染层
   - 3.7 服务层
   - 3.8 Bridge / 远程执行
   - 3.9 多 Agent 系统
   - 3.10 Plugin & Skill 系统
   - 3.11 基础设施与工具库
4. [数据流分析](#4-数据流分析)
5. [关键设计模式](#5-关键设计模式)
6. [功能门控系统](#6-功能门控系统)
7. [扩展点地图](#7-扩展点地图)
8. [关键文件速查表](#8-关键文件速查表)

---

## 1. 项目概览

Claude Code 是 Anthropic 的官方命令行 AI 助手，基于 **TypeScript + Bun + React（Ink）** 构建。其核心定位是一个可扩展的 **终端 AI Agent 框架**，具备：

- 交互式 REPL 对话界面
- 结构化工具执行系统（文件操作、Bash、搜索、Web 等）
- 多 Agent 协调与任务并发
- 本地 / 远程双模执行
- Plugin / Skill / MCP 三层扩展机制
- 企业级权限控制与合规管理

**技术栈：**

| 层次 | 技术 |
|------|------|
| 运行时 | Bun（含 `bun:bundle` 功能门控） |
| 语言 | TypeScript（严格模式） |
| UI 框架 | React + 自定义 Ink 终端渲染引擎 |
| 布局引擎 | Yoga（Flexbox for Terminal） |
| 状态管理 | React Context + 不可变状态树 |
| API | @anthropic-ai/sdk |
| 模式验证 | Zod v4 |
| CLI 解析 | commander-js |

---

## 2. 整体架构

### 2.1 顶层模块图

```
src/
├── 【启动层】    main.tsx · bootstrap/ · entrypoints/
├── 【引擎层】    QueryEngine.ts · query.ts · query/
├── 【工具层】    Tool.ts · tools.ts · tools/(42个工具)
├── 【命令层】    commands.ts · commands/(101个命令)
├── 【状态层】    state/ · Task.ts · tasks/
├── 【UI层】      screens/ · components/ · ink/ · vim/ · voice/
├── 【服务层】    services/(20个服务) · context/
├── 【远程层】    bridge/ · remote/ · assistant/ · coordinator/
├── 【扩展层】    plugins/ · skills/ · hooks/
└── 【基础层】    utils/(329个工具函数) · types/ · schemas/ · constants/
```

### 2.2 系统架构图（纵向分层）

```
┌─────────────────────────────────────────────────────────┐
│                      用户界面层                          │
│  screens/REPL.tsx(5005行)  ·  components/(38个)         │
│  ink/(自定义终端渲染引擎)   ·  vim/ · voice/             │
├─────────────────────────────────────────────────────────┤
│                      应用逻辑层                          │
│  commands/(101个)  ·  QueryEngine.ts  ·  query.ts        │
│  hooks/            ·  state/AppStateStore.ts             │
├─────────────────────────────────────────────────────────┤
│                      工具执行层                          │
│  Tool.ts(接口)  ·  tools.ts(注册表)  ·  tools/(42个)    │
│  tasks/ · Task.ts · TaskManager                          │
├─────────────────────────────────────────────────────────┤
│                      服务与扩展层                        │
│  services/mcp/  ·  services/api/  ·  services/analytics/ │
│  plugins/  ·  skills/  ·  memdir/                        │
├─────────────────────────────────────────────────────────┤
│                    远程 / 多Agent层                       │
│  bridge/  ·  remote/  ·  coordinator/  ·  assistant/     │
├─────────────────────────────────────────────────────────┤
│                      基础设施层                          │
│  utils/(329个)  ·  types/  ·  schemas/  ·  constants/    │
│  bootstrap/  ·  upstreamproxy/                           │
└─────────────────────────────────────────────────────────┘
```

---

## 3. 模块详解

### 3.1 启动与引导层

#### `main.tsx`（4683 行）

程序唯一入口，执行顺序严格优化：

```
① profileCheckpoint('main_tsx_entry')   ← 启动性能计时
② startMdmRawRead()                     ← MDM 配置并行预读（macOS: plutil / Win: reg query）
③ startKeychainPrefetch()               ← 并行预取 Keychain OAuth + API Key（节省 65ms）
④ 特殊模式快速路径
   ├── --dump-system-prompt              ← 打印系统提示词后退出
   ├── --chrome-native-host              ← Chrome 扩展 Native Messaging 模式
   ├── --daemon-workers                  ← 后台 daemon 工作进程模式
   └── 正常 CLI 模式                    → entrypoints/cli.tsx
⑤ 死代码消除门控（bun:bundle feature()）
   ├── COORDINATOR_MODE → coordinator/coordinatorMode.js
   ├── KAIROS          → assistant/index.js
   └── 其他门控模块...
```

**关键性能优化：**
- MDM 读取和 Keychain 预取在模块评估阶段就并发触发（import 副作用）
- 非交互模式（`--print`）和 SDK 模式绕过完整 React 初始化

#### `bootstrap/state.ts`（56KB）

全局不可变状态的最早初始化点，设置：
- `originalCwd` — 进程启动时的工作目录
- `isRemote` — 是否为远程 Bridge 模式
- MDM 策略限制
- session 元信息（sessionId、agentId）
- 环境变量规范化

#### `entrypoints/cli.tsx`（302 行可见 + 完整 CLI 定义）

使用 `commander-js` 构建完整 CLI，处理：
- 全局选项（`--model`, `--permission-mode`, `--debug`）
- 子命令路由（`claude config`, `claude mcp` 等）
- 工具注册（`getTools()`）
- 命令注册（`getCommands()`）
- 初始消息构造与 REPL 启动

---

### 3.2 核心引擎层

#### `QueryEngine.ts`（1295 行）

**框架最核心模块**，负责一次完整 AI 交互的全生命周期：

```
processUserInput()
    ↓
fetchSystemPromptParts()   ← 构建系统提示词（CLAUDE.md + 记忆 + 技能）
    ↓
loadMemoryPrompt()         ← 加载 memdir/ 持久化记忆
    ↓
selectMessages()           ← 消息历史窗口管理 + 压缩决策
    ↓
query()                    ← 调用 Claude API（流式）
    ↓
工具调用循环
    ├── tool_use → 查找工具 → canUseTool() → execute()
    ├── 结果注入回消息历史
    └── 继续对话直到 stop_reason = "end_turn"
    ↓
recordTranscript()         ← 会话持久化
    ↓
flushSessionStorage()      ← 磁盘刷新
```

**核心职责：**
- 系统提示词动态构建（注入 CLAUDE.md、记忆、技能描述）
- 上下文窗口管理（token 计数 + 历史压缩触发）
- 工具调用编排（权限检查 → 执行 → 结果处理）
- 错误重试（分类 API 错误，区分可重试/不可重试）
- 投机执行（speculation）支持

#### `query.ts`

底层 API 调用层，封装 `@anthropic-ai/sdk` 的流式调用：
- Token 计数与 budget 控制
- 流式事件解析
- 速率限制处理
- Beta 功能标志管理（thinking、extended context 等）

---

### 3.3 工具系统

#### 接口定义（`Tool.ts`）

工具系统的核心契约，定义了 4 个关键类型：

```typescript
// 工具使用上下文 —— 传入每个工具执行调用
type ToolUseContext = {
  options: {
    commands, debug, mainLoopModel, tools,
    verbose, thinkingConfig, mcpClients, mcpResources,
    isNonInteractiveSession, agentDefinitions, maxBudgetUsd,
    customSystemPrompt, appendSystemPrompt
  }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f): void
  setAppStateForTasks?: (f) => void   // 会话级基础设施变更
  handleElicitation?: (...)           // MCP URL 触发处理
  setToolJSX?: SetToolJSXFn          // UI 嵌入渲染
  addNotification?: (notif) => void
  appendSystemMessage?: (msg) => void
  updateFileHistoryState: (updater) => void
  agentId?: AgentId                  // 仅子 Agent 设置
  agentType?: string
  // ... 约 20 个回调字段
}

// 工具权限上下文（DeepImmutable）
type ToolPermissionContext = {
  mode: PermissionMode                    // 'default' | 'plan' | 'auto' | 'bypass'
  additionalWorkingDirectories: Map<...>
  alwaysAllowRules, alwaysDenyRules, alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
}
```

#### 工具注册表（`tools.ts`）

条件加载 42 个工具，按功能分组：

**文件操作类（核心）：**
| 工具 | 功能 |
|------|------|
| `FileReadTool` | 读取文件内容（含 PDF、图片） |
| `FileEditTool` | 精确字符串替换编辑 |
| `FileWriteTool` | 写入/创建文件 |
| `GlobTool` | 文件模式匹配搜索 |
| `GrepTool` | 内容正则搜索（ripgrep） |
| `NotebookEditTool` | Jupyter Notebook 编辑 |

**执行类：**
| 工具 | 功能 |
|------|------|
| `BashTool` | Shell 命令执行（沙箱 + 超时） |
| `PowerShellTool` | Windows PowerShell 执行 |
| `REPLTool` | 持久化 REPL 会话（Node/Python） |

**智能类：**
| 工具 | 功能 |
|------|------|
| `AgentTool` | 启动子 Agent（递归调用 QueryEngine） |
| `SkillTool` | 执行预定义技能 |
| `ToolSearchTool` | 延迟工具的 schema 搜索与加载 |

**任务管理类：**
| 工具 | 功能 |
|------|------|
| `TaskCreateTool` | 创建后台任务 |
| `TaskGetTool` | 查询任务状态 |
| `TaskListTool` | 列出所有任务 |
| `TaskOutputTool` | 读取任务输出 |
| `TaskUpdateTool` | 更新任务状态 |
| `TaskStopTool` | 停止任务 |

**MCP 类：**
| 工具 | 功能 |
|------|------|
| `MCPTool` | 调用 MCP 工具 |
| `ListMcpResourcesTool` | 列出 MCP 资源 |
| `ReadMcpResourceTool` | 读取 MCP 资源 |
| `McpAuthTool` | MCP 服务器认证 |

**工作流类：**
| 工具 | 功能 |
|------|------|
| `EnterPlanModeTool` | 进入计划模式（只读权限） |
| `ExitPlanModeTool` | 退出计划模式 |
| `EnterWorktreeTool` | 进入 Git worktree 隔离环境 |
| `ExitWorktreeTool` | 退出 worktree |
| `TodoWriteTool` | 写入任务清单 |
| `AskUserQuestionTool` | 向用户提问 |

**多 Agent 类（功能门控）：**
| 工具 | 功能 | 门控 |
|------|------|------|
| `TeamCreateTool` | 创建 Agent 团队 | AGENT_SWARMS |
| `TeamDeleteTool` | 解散 Agent 团队 | AGENT_SWARMS |
| `SendMessageTool` | Agent 间消息传递 | AGENT_SWARMS |
| `RemoteTriggerTool` | 触发远程任务 | AGENT_TRIGGERS |
| `ScheduleCronTool` | 创建定时触发器 | AGENT_TRIGGERS |
| `SleepTool` | Agent 休眠等待 | KAIROS |
| `BriefTool` | 生成摘要 | 内部 |

#### 任务类型（`Task.ts`）

```typescript
type TaskType =
  | 'local_bash'         // 本地 Shell 任务
  | 'local_agent'        // 本地子 Agent
  | 'remote_agent'       // 远程 Agent（CCR）
  | 'in_process_teammate'// 进程内队友 Agent
  | 'local_workflow'     // 本地工作流
  | 'monitor_mcp'        // MCP 监控任务
  | 'dream'              // 自动探索任务

type TaskStatus = 'pending' | 'running' | 'completed' | 'failed' | 'killed'
```

任务 ID 使用类型前缀 + 36进制随机字节（`b-`, `a-`, `r-`, `t-`, `w-`, `m-`, `d-`），生成约 2.8 万亿组合，防止符号链接攻击。

---

### 3.4 命令系统

#### 命令注册（`commands.ts`）

101 个命令按领域组织，部分重要命令：

**开发工作流：**
- `commit` / `commit-push-pr` — 生成提交信息并提交
- `diff` — 查看 git diff
- `branch` — 分支管理
- `autofix-pr` — 自动修复 PR 问题

**会话管理：**
- `clear` — 清空对话历史
- `compact` — 压缩历史（触发 AI 摘要）
- `context` — 上下文可视化
- `resume` / `exit` — 会话恢复/退出
- `export` — 导出对话记录

**配置与权限：**
- `config` — 查看/修改配置
- `hooks` — 配置 hooks
- `keybindings` — 快捷键管理
- `permissions` — 权限规则管理

**高级功能：**
- `plan` — 进入计划模式
- `ultraplan` (66KB) — 超级计划模式
- `debug-tool-call` — 工具调用调试
- `doctor` — 系统诊断

**集成：**
- `install-github-app` — GitHub App 安装
- `install-slack-app` — Slack 集成
- `mcp` — MCP 服务器管理
- `ide` — IDE 插件集成

---

### 3.5 状态管理

#### `state/AppStateStore.ts`（569 行）

框架唯一数据源（Single Source of Truth），采用 **DeepImmutable 不可变状态树**：

```typescript
// 状态树主要字段（简化）
type AppState = {
  // 对话相关
  messages: Message[]
  conversationId: UUID

  // 权限系统
  toolPermissionContext: ToolPermissionContext
  pendingPermissions: PendingPermission[]

  // 任务系统
  tasks: Map<string, TaskState>

  // 多 Agent
  teammates: Map<AgentId, TeammateState>

  // UI 状态
  theme: Theme
  footerSelections: FooterSelection[]
  isSpeculationRunning: boolean

  // 文件历史
  fileHistoryState: FileHistoryState
  attributionState: AttributionState

  // 设置
  settings: Settings
}
```

**状态更新模式：**
```typescript
// 函数式更新，始终返回新状态
setAppState(f: (prev: AppState) => AppState): void

// 任务级更新（穿透子 Agent 隔离边界）
setAppStateForTasks?: (f: (prev: AppState) => AppState) => void
```

#### `state/store.ts`

基于 Zustand 的 store 工厂，包装 DeepImmutable 状态。

---

### 3.6 UI 渲染层

#### 自定义 Ink 终端渲染引擎（`ink/`）

这是项目中最独特的模块——一个**完整的终端 UI 框架**，非直接依赖第三方 `ink` 包，而是深度定制版：

```
ink/
├── reconciler.ts        ← React Fiber Reconciler（将 React 树→终端节点）
├── renderer.ts          ← 渲染管线协调器
├── render-to-screen.ts  ← 差异渲染到终端输出
├── render-node-to-output.ts ← 节点→字符矩阵
├── render-border.ts     ← 边框渲染
├── layout/
│   ├── engine.ts        ← Flexbox 布局计算
│   └── yoga.ts          ← Yoga 布局引擎绑定
├── termio/              ← 终端 I/O 层
│   ├── parser.ts        ← ANSI 序列解析
│   ├── tokenize.ts      ← 字节流 tokenizer
│   ├── csi.ts           ← CSI 序列处理
│   ├── osc.ts           ← OSC 序列处理
│   └── dec.ts           ← DEC 私有序列
├── events/              ← 事件系统
│   ├── dispatcher.ts    ← 事件分发
│   ├── keyboard-event.ts
│   ├── click-event.ts
│   ├── focus-event.ts
│   └── input-event.ts
├── components/          ← 基础 UI 原语
│   ├── Box.tsx          ← 布局容器（Flexbox）
│   ├── Text.tsx         ← 文本渲染
│   ├── ScrollBox.tsx    ← 滚动容器
│   └── Button.tsx
├── hooks/               ← React 钩子
│   ├── use-input.ts     ← 键盘输入
│   ├── use-selection.ts ← 文本选择
│   └── use-terminal-viewport.ts
└── bidi.ts              ← 双向文本（BiDi）支持
```

**渲染管线：**
```
React 组件树
    ↓ reconciler.ts（React Fiber）
虚拟节点树（ink 节点）
    ↓ layout/engine.ts（Yoga Flexbox）
带坐标的节点树
    ↓ render-node-to-output.ts
字符矩阵（Output 对象）
    ↓ render-to-screen.ts（差异比较）
ANSI 转义序列
    ↓ termio/
终端显示
```

#### `screens/REPL.tsx`（5005 行）

主交互界面，是整个应用最大的单文件。管理：
- 多行文本输入（含 Vim 模式）
- 消息列表渲染（流式更新）
- 工具调用进度展示
- 权限请求对话框
- 自动补全与建议
- 搜索高亮
- 底部状态栏

#### `components/`（38 个子目录）

业务 UI 组件，部分重要组件：
- `App.tsx` — 根应用组件
- `CoordinatorAgentStatus.tsx` — 多 Agent 状态显示
- `ContextVisualization.tsx` — 上下文可视化
- `DevBar.tsx` — 开发调试栏
- `diff/` — diff 渲染组件
- `design-system/` — 设计系统基础组件

#### `vim/`

完整 Vim 键位模拟：
- `motions.ts` — 移动命令（`hjkl`, `w`, `b`, `e`, `0`, `$` 等）
- `operators.ts` — 操作符（`d`, `c`, `y`, `>`, `<`）
- `textObjects.ts` — 文本对象（`iw`, `aw`, `i"`, `a(` 等）
- `transitions.ts` — 模式切换（Normal/Insert/Visual/Command）

---

### 3.7 服务层

#### `services/api/`（Claude API 交互）
- `claude.ts` — API 客户端，usage 统计
- `bootstrap.ts` — 启动数据预取（feature flags、tips 等）
- `filesApi.ts` — Files API（上传/下载附件）
- `errors.ts` — API 错误分类（可重试 vs. 不可重试）

#### `services/mcp/`（Model Context Protocol）
- MCP 服务器注册、连接管理
- 权限审批流程
- 工具/资源发现与代理
- 官方 MCP 注册表集成

#### `services/analytics/`（可观测性）
- `growthbook.ts` — GrowthBook 功能门控（A/B 测试）
- 事件日志（Datadog / 内部系统）
- 诊断追踪

#### `services/oauth/`（认证）
- OAuth 2.0 流程（claudeai.com + Bedrock + GCP）
- token 刷新与 keychain 存储
- AWS/GCP 凭据管理

#### `services/compact/`（历史压缩）
- 基于 AI 的消息历史摘要
- pre_compact / post_compact hooks 触发
- token budget 监控与自动触发

#### `services/lsp/`（语言服务器）
- LSP 协议集成
- 代码高亮与符号解析
- IDE 集成支持

#### `services/SessionMemory/`（会话记忆）
- 跨会话记忆提取与注入
- CLAUDE.md 记忆文件管理
- 结构化记忆 vs. 自由文本记忆

#### 其他服务
| 服务 | 功能 |
|------|------|
| `AgentSummary` | Agent 执行摘要生成 |
| `autoDream` | 自动探索/梦境任务 |
| `MagicDocs` | 文档自动生成 |
| `PromptSuggestion` | 提示词建议 |
| `toolUseSummary` | 工具调用结果摘要 |
| `tips` | 用户提示管理 |
| `settingsSync` | 设置云同步 |
| `teamMemorySync` | 团队记忆同步 |
| `policyLimits` | 企业策略限制 |

---

### 3.8 Bridge / 远程执行

#### Bridge 架构（`bridge/`）

Bridge 模式是 Claude Code 的**云端执行模式（CCR, Cloud Claude Runtime）**，33 个文件：

```
本地 Claude Code CLI
    ↓ bridge/initReplBridge.ts（检测远程环境）
    ↓
bridge/bridgeMain.ts（2999 行，主协调器）
    ├── 创建本地 REPL 进程
    ├── 建立与 CCR 的双向通信
    ├── bridgeMessaging.ts — 消息序列化/反序列化
    ├── jwtUtils.ts — JWT token 刷新（轮询）
    ├── flushGate.ts — 背压控制
    └── capacityWake.ts — 容量管理

bridge/replBridge.ts（2406 行，传输层）
    ├── replBridgeHandle.ts — 连接句柄
    ├── replBridgeTransport.ts — HTTP/WebSocket 传输
    └── remoteBridgeCore.ts — 远程核心逻辑

bridge/codeSessionApi.ts — 会话 API 客户端
bridge/sessionRunner.ts  — 子进程生命周期管理
bridge/bridgeUI.ts       — Bridge 模式 UI 覆盖层
bridge/bridgePermissionCallbacks.ts — 权限回调桥接
```

**通信协议：**
- 上行：本地工具执行结果 → HTTP POST → CCR
- 下行：CCR 指令 → HTTP 长轮询 → 本地执行
- 认证：JWT（自动刷新）
- 附件：Files API（`inboundAttachments.ts`）

#### Remote 模式（`remote/`）

查看远程 Agent 运行状态的只读模式：

```
remote/SessionsWebSocket.ts   — WebSocket 连接管理
remote/RemoteSessionManager.ts — 会话生命周期
remote/sdkMessageAdapter.ts   — SDK 消息格式适配
remote/remotePermissionBridge.ts — 权限请求转发
```

#### Assistant 模式（`assistant/`）

KAIROS（云端 Assistant）的历史记录分页获取，通过 Claude API events 接口拉取分页历史。

---

### 3.9 多 Agent 系统

#### Coordinator 模式（`coordinator/coordinatorMode.ts`，19KB）

多 Agent 任务协调的主控 Agent，职责：
- 将复杂任务分解为子任务
- 创建并管理多个 Worker Agent
- 控制 Worker 可用工具集（限制危险工具）
- 汇总 Worker 结果
- Scratchpad（工作草稿）访问控制

#### Agent Swarms（`utils/swarm/`）

并发 Agent 团队机制：
```
utils/swarm/
├── teammatePromptAddendum.ts — 队友系统提示附录
├── reconnection.ts           — 断线重连逻辑
└── backends/
    └── teammateModeSnapshot.ts — 队友状态快照
```

**Agent 生命周期：**
```
AgentTool.execute()
    ↓
createSubagentContext()       ← 创建隔离上下文
    ↓
QueryEngine（子 Agent 实例）  ← 递归调用
    ↓
setAppStateForTasks()         ← 任务注册到根 AppState
    ↓
结果返回父 Agent
```

**关键隔离机制：**
- 子 Agent 的 `setAppState` 是 no-op（状态隔离）
- `setAppStateForTasks` 穿透隔离（任务基础设施共享）
- 每个 Agent 有独立 `AgentId` 和 `abortController`

---

### 3.10 Plugin & Skill 系统

#### Plugin 系统（`plugins/` + `services/plugins/`）

```
plugins/
├── builtinPlugins.ts    ← 内置插件注册
└── bundled/             ← 打包进发行版的插件

services/plugins/
├── validation.ts        ← 插件 manifest 验证（Zod schema）
└── pluginLoader.ts（in utils）← 动态加载 ~/.claude/plugins/
```

**Plugin 能力：**
- 注册新工具
- 注册新命令
- 注入系统提示词片段
- MCP 服务器配置

#### Skill 系统（`skills/`）

Skill 是**结构化的提示词模板**，比 Plugin 更轻量：

```
skills/
├── bundledSkills.ts     ← 注册 19 个内置 Skill
├── loadSkillsDir.ts     ← 动态加载 ~/.claude/skills/（34KB）
└── bundled/             ← 19 个内置 Skill 实现
    ├── batch.ts         ← 批量操作
    ├── claudeApi.ts     ← Claude API 调用
    ├── commit.ts        ← 生成提交
    ├── debug.ts         ← 调试助手
    ├── keybindings.ts   ← 快捷键配置
    ├── loop.ts          ← 循环执行
    ├── remember.ts      ← 记忆保存
    ├── scheduleRemoteAgents.ts ← 调度远程 Agent
    ├── simplify.ts      ← 代码简化
    ├── skillify.ts      ← 将对话转为 Skill
    ├── stuck.ts         ← 卡住时求助
    ├── updateConfig.ts  ← 更新配置
    ├── verify.ts        ← 验证结果
    └── ...（其他）
```

**Skill 注入流程：**
```
loadSkillsDir() → 验证 skill.md frontmatter
    ↓
SkillTool 注册（with 延迟加载 schema）
    ↓
QueryEngine.fetchSystemPromptParts() → 注入 skill 列表到系统提示
    ↓
Claude 选择调用 SkillTool
    ↓
skill 内容展开 → 替换 {{variables}} → 执行
```

---

### 3.11 基础设施与工具库

#### `utils/`（329 个文件）

规模最大的模块，按功能分组：

**认证与安全：**
- `auth.ts` — OAuth token 管理、订阅类型检测
- `secureStorage/` — Keychain 抽象（macOS/Win/Linux）
- `authFileDescriptor.ts` — 文件描述符认证传递
- `sessionIngressAuth.ts` — 进入会话认证

**文件系统：**
- `attachments.ts`（127KB，最大工具文件）— 附件处理（图片、PDF、文档）
- `fileHistory.ts` — 文件编辑历史追踪
- `fileStateCache.ts` — 文件状态 LRU 缓存
- `claudemd.ts` — CLAUDE.md 文件发现与解析

**消息处理：**
- `messages.ts` — 消息创建工厂（`createUserMessage`, `createSystemMessage`）
- `api.ts`（26KB）— API 消息规范化
- `processUserInput/` — 用户输入预处理管线

**权限系统：**
- `permissions/` — 权限规则引擎
  - `denialTracking.ts` — 拒绝记录与学习
  - 自动审批分类器集成

**模型管理：**
- `model/model.ts` — 模型选择逻辑
- `model/deprecation.ts` — 过期模型处理
- `fastMode.ts` — Fast Mode 状态管理
- `thinking.ts` — Extended Thinking 配置

**Shell 执行：**
- `bash/` — Bash 工具实现（超时、信号处理、输出捕获）
- `Shell.ts` — 工作目录管理

**启动性能：**
- `startupProfiler.js` — 精细化启动计时（多个 checkpoint）
- `headlessProfiler.ts` — 无头模式性能分析

**设置管理：**
- `settings/settings.ts` — JSON 配置文件读写
- `settings/mdm/` — 企业 MDM 策略（macOS plist / Windows Registry）
- `settings/changeDetector.ts` — 文件变更监听

**其他：**
- `computerUse/` — 计算机视觉/控制能力
- `agentId.ts` — Agent ID 生成与管理
- `codeIndexing.ts` — 代码库索引
- `commitAttribution.ts` — commit 归因追踪

#### `types/`

核心类型定义（避免循环依赖的集中位置）：
```
types/
├── message.ts      ← 消息联合类型（AssistantMessage | UserMessage | SystemMessage | ...）
├── permissions.ts  ← 权限相关类型（PermissionMode, PermissionResult, ...）
├── tools.ts        ← 工具进度类型（BashProgress, AgentToolProgress, ...）
├── hooks.ts        ← Hooks 类型（PromptRequest, PromptResponse, HookProgress）
├── ids.ts          ← ID 类型（AgentId, SessionId）
├── logs.ts         ← 日志类型
└── utils.ts        ← 工具类型（DeepImmutable, ...）
```

#### `memdir/`（记忆系统）

```
memdir/
├── memdir.ts    ← 记忆加载与格式化（注入系统提示词）
└── paths.ts     ← 记忆文件路径解析（~/.claude/memory/）
```

支持：
- 基于文件的持久化记忆
- 自动记忆提取（`services/extractMemories/`）
- 跨会话记忆同步（`services/teamMemorySync/`）

---

## 4. 数据流分析

### 4.1 用户消息完整处理流程

```
[用户键入输入]
    │
    ▼
screens/REPL.tsx
  onSubmit(input)
    │
    ├─→ commands.ts.matchCommand(input)
    │     如果匹配：执行命令，结束
    │
    └─→ QueryEngine.processUserInput()
          │
          ├─ 1. 预处理
          │     processUserInput() → 解析 @file 引用, 图片附件
          │
          ├─ 2. 构建系统提示词
          │     fetchSystemPromptParts()
          │       ├── CLAUDE.md 加载（递归向上查找）
          │       ├── memdir/ 记忆注入
          │       ├── Skills 列表
          │       └── 工具描述
          │
          ├─ 3. 消息选择
          │     selectMessages() → token budget 检查 → 可能触发压缩
          │
          ├─ 4. API 调用
          │     query.ts → @anthropic-ai/sdk（流式）
          │       ├── 模型选择（main loop model / fast mode）
          │       ├── Beta flags（thinking, computer-use, etc.）
          │       └── 流式事件处理
          │
          ├─ 5. 工具调用循环（可能多轮）
          │     for each tool_use in response:
          │       ├── 找到工具实现（tools.ts 注册表）
          │       ├── canUseTool() → 权限检查
          │       │     ├── alwaysAllow 规则？→ 直接执行
          │       │     ├── alwaysDeny 规则？→ 拒绝
          │       │     └── 需要确认？→ UI 弹窗 → 等待用户
          │       ├── tool.execute(input, context)
          │       └── 结果注入消息历史 → 继续对话
          │
          └─ 6. 结果持久化
                recordTranscript() → ~/.claude/projects/<hash>/session.jsonl
                flushSessionStorage()
```

### 4.2 工具权限决策流程

```
canUseTool(toolName, input, context)
    │
    ├─ hooks 检查（pre-tool hooks）
    │     ├── 外部进程 hooks（~/.claude/settings.json hooks 配置）
    │     └── 内部分类器（AI 安全分类）
    │
    ├─ 规则匹配
    │     ├── alwaysAllow 规则 → approve（跳过 UI）
    │     ├── alwaysDeny 规则 → deny
    │     └── alwaysAsk 规则 → 强制弹窗
    │
    ├─ PermissionMode 决策
    │     ├── 'default' → 逐个确认危险操作
    │     ├── 'plan' → 只读，禁止写操作
    │     ├── 'auto' → 自动审批（配置的规则内）
    │     └── 'bypass' → 完全跳过权限（开发模式）
    │
    └─ UI 交互（必要时）
          → REPL.tsx 渲染权限对话框
          → 等待用户 approve/deny/always-allow/always-deny
          → 结果写回 ToolPermissionContext（持久化到 settings.json）
```

### 4.3 Bridge 模式数据流

```
本地 Claude Code（用户端）          CCR 云端
        │                               │
        │◄─── HTTP 长轮询 ─────────────│
        │     指令：执行工具X            │
        │                               │
  本地执行工具 X                        │
  (Bash/File/等)                        │
        │                               │
        │──── HTTP POST 结果 ──────────►│
        │                               │
        │     (CCR 继续推理)            │
        │                               │
        │◄─── 新指令 ──────────────────│
        │                               │
```

---

## 5. 关键设计模式

### 5.1 不可变状态 + 函数式更新

```typescript
// 永远不直接修改状态，而是返回新对象
setAppState((prev) => ({
  ...prev,
  messages: [...prev.messages, newMessage],
}))

// DeepImmutable<T> 在类型层面强制不可变
type AppState = DeepImmutable<{...}>
```

### 5.2 循环依赖打破：集中类型定义 + 懒加载

```typescript
// types/ 作为无依赖的类型集中地
import type { PermissionMode } from './types/permissions.js'  // ✓
// 而非从 Tool.ts 导入（会引入循环）

// 懒加载打破运行时循环依赖
const getTeammateUtils = () => require('./utils/teammate.js')
```

### 5.3 功能门控死代码消除

```typescript
// Bun 构建时消除未启用的 feature 分支
const coordinatorModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js')
  : null  // ← 构建时整个分支被移除
```

### 5.4 Tool 权限三层分离

```
Layer 1: ToolPermissionContext（运行时规则）
    ↓ 被
Layer 2: canUseTool()（决策函数）
    ↓ 查询
Layer 3: hooks（外部 + AI 分类器）
```

### 5.5 上下文传播（非全局状态）

```typescript
// ToolUseContext 贯穿整个调用栈，避免全局变量
async function execute(input, context: ToolUseContext) {
  const state = context.getAppState()
  context.setAppState(prev => {...})
  context.addNotification?.(...)
  // ...
}
```

### 5.6 子 Agent 状态隔离

```typescript
function createSubagentContext(parentCtx): ToolUseContext {
  return {
    ...parentCtx,
    // 子 Agent 的状态变更不影响父 Agent
    setAppState: (_f) => {},  // no-op
    // 但任务基础设施变更穿透到根
    setAppStateForTasks: parentCtx.setAppStateForTasks ?? parentCtx.setAppState,
  }
}
```

---

## 6. 功能门控系统

使用 `bun:bundle` 的 `feature()` 函数，在**构建时**决定是否包含代码（Dead Code Elimination）：

| Feature Flag | 功能 | 影响范围 |
|---|---|---|
| `COORDINATOR_MODE` | 多 Agent 协调模式 | `coordinator/`, `AgentTool`, `TeamCreateTool` |
| `KAIROS` | 云端 Assistant 模式 | `assistant/`, `bridge/`, `SleepTool`, `BriefTool` |
| `AGENT_SWARMS` | Agent 团队并发 | `TeamCreateTool`, `TeamDeleteTool`, `SendMessageTool`, `utils/swarm/` |
| `AGENT_TRIGGERS` | 定时/远程触发器 | `RemoteTriggerTool`, `ScheduleCronTool` |
| `VOICE_MODE` | 语音输入输出 | `voice/`, `voiceStreamSTT.ts` |
| `DAEMON` | 后台 daemon 进程 | `main.tsx` daemon 分支 |
| `PROACTIVE` | 主动任务执行 | `autoDream/`, 自动探索功能 |
| `COMPUTER_USE` | 计算机视觉控制 | `utils/computerUse/` |
| `WORKTREE_MODE` | Git Worktree 隔离 | `EnterWorktreeTool`, `ExitWorktreeTool` |

---

## 7. 扩展点地图

```
Claude Code 扩展点
├── 1. MCP 服务器（最强大）
│      位置：~/.claude/mcp-servers/ 或 settings.json
│      能力：注册任意工具/资源，独立进程通信
│      协议：Model Context Protocol (JSON-RPC 2.0)
│
├── 2. Plugin（中等）
│      位置：~/.claude/plugins/
│      能力：注册工具、命令、系统提示词片段
│      格式：plugin.json manifest + TS/JS 代码
│
├── 3. Skill（轻量）
│      位置：~/.claude/skills/
│      能力：结构化提示词模板，可含变量替换
│      格式：skill.md（YAML frontmatter + Markdown）
│
├── 4. Hooks（事件驱动）
│      位置：~/.claude/settings.json → hooks
│      事件：PreToolUse, PostToolUse, Notification
│             PreCompact, PostCompact, SessionStart
│      能力：外部进程干预工具调用（可批准/拒绝/修改）
│
├── 5. CLAUDE.md（上下文注入）
│      位置：项目根目录 / 任意父目录 / ~/.claude/
│      能力：注入项目特定指令到系统提示词
│      特性：递归向上合并多层 CLAUDE.md
│
└── 6. Agent 定义（~/.claude/agents/）
       能力：定义具名子 Agent，带自定义系统提示词
       使用：AgentTool 的 subagent_type 参数
```

---

## 8. 关键文件速查表

| 场景 | 文件路径 | 说明 |
|------|---------|------|
| 理解启动流程 | `src/main.tsx` | 全局入口，4683 行 |
| 理解 AI 交互核心 | `src/QueryEngine.ts` | 查询引擎，1295 行 |
| 添加新工具 | `src/Tool.ts` + `src/tools.ts` | 接口定义 + 注册表 |
| 添加新命令 | `src/commands.ts` | 命令注册表 |
| 修改 UI | `src/screens/REPL.tsx` | 主界面，5005 行 |
| 理解状态管理 | `src/state/AppStateStore.ts` | 状态树定义 |
| 理解权限系统 | `src/Tool.ts` + `src/types/permissions.ts` | 权限接口 |
| 理解消息格式 | `src/types/message.ts` | 消息类型定义 |
| 理解任务系统 | `src/Task.ts` | 任务类型定义 |
| 理解 Bridge | `src/bridge/bridgeMain.ts` | Bridge 主协调器，2999 行 |
| 理解多 Agent | `src/coordinator/coordinatorMode.ts` | 协调模式，19KB |
| 理解记忆系统 | `src/memdir/memdir.ts` | 记忆加载 |
| 理解 Skill 系统 | `src/skills/loadSkillsDir.ts` | Skill 加载器，34KB |
| 理解终端渲染 | `src/ink/reconciler.ts` + `renderer.ts` | 渲染管线 |
| 配置与设置 | `src/utils/config.js` + `utils/settings/` | 配置读写 |
| 文件操作工具 | `src/tools/FileEditTool/` | 编辑工具实现 |
| Bash 执行工具 | `src/tools/BashTool/` | Shell 工具实现 |
| Web 搜索工具 | `src/tools/WebSearchTool/` | 搜索工具实现 |

---


