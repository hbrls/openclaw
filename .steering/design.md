# 项目结构与架构指南

## 项目包结构（Monorepo）

### 主包位置：根目录

**openclaw** 主包位于 **repository 根目录**：

```
.
├── package.json              ← 主包定义 "name": "openclaw"
├── pnpm-workspace.yaml       ← Monorepo 工作区配置
├── openclaw.mjs              ← CLI 入口（命令行工具）
├── src/                      ← 核心功能代码
├── dist/                     ← 编译后的输出
├── extensions/               ← 插件生态系统
├── packages/                 ← 辅助包（兼容性垫片）
├── apps/                     ← 移动应用（iOS/Android）
└── ...
```

### 核心组件

| 组件 | 位置 | 用途 |
|------|------|------|
| **CLI 入口** | `openclaw.mjs` | 命令行工具入口，执行 `openclaw` 命令时调用 |
| **源代码** | `src/` | 核心业务逻辑（gateway、channels、agents 等） |
| **编译输出** | `dist/` | 打包编译后的代码 |
| **扩展系统** | `extensions/` | 15 个扩展插件（Slack、内存、工具等） |
| **网页控制面板** | `ui/` | OpenClaw 控制台（网页界面） |
| **移动应用** | `apps/` | iOS、Android 的原生应用 |

### 辅助包（`packages/`）

这写都是向后兼容性垫片（Compatibility Shim），用于支持项目改名：

| 包名 | 用途 | 说明 |
|------|------|------|
| **moltbot** | 兼容性转发 | 项目旧名 `moltbot` 的转发包，`export * from "openclaw"` |
| **clawdbot** | 兼容性转发 | 项目旧名 `clawdbot` 的转发包，`export * from "openclaw"` |

**特点**：
- 仅包含转发逻辑，没有实际业务代码
- 安装时显示重命名提示
- 支持旧命令 `moltbot` / `clawdbot` 的向后兼容性
- 代码中还保留 `CLAWDBOT_*` 环境变量支持（映射到 `OPENCLAW_*`）

### 工作区配置

**pnpm-workspace.yaml** 定义了多包结构：
```yaml
packages:
  - "packages/*"
  - "extensions/*"
  - "apps/*"
  - "src"  # 可选
```

这使得 pnpm install 可以一次性安装所有依赖，并支持跨包引用。

### 网页控制面板（`ui/`）

**功能**：提供网页版 OpenClaw dashboard，支持：
- **Agent 管理** - 查看、配置、控制 Agent
- **聊天交互** - 与 Agent 对话（支持工具调用和消息流）
- **频道配置** - 配置 Telegram、Discord、Slack 等多种通讯渠道
- **定时任务** - 创建和管理 Cron 工作流
- **配置管理** - 模型、执行规则等系统配置
- **监控面板** - 日志、会话、用量统计

**技术栈**：Lit (Web Components)、Vite、Vitest、多语言 i18n

---

# Agent 相关代码结构

> **详细信息**：关于 `src/agents/`、`src/context-engine/` 等核心模块的完整分析，请参考 [design-core.md](design-core.md)：
> - [服务与网关 - agents/](design-core.md#agents---ai-代理和会话生命周期) - AI 代理生命周期管理和运行时
> - [AI 代理核心 - context-engine/](design-core.md#context-engine---上下文管理引擎) - 上下文管理接口层

本文档只保留与其他系统（Extensions、应用端）相关的架构内容。

## Extensions 概览

### 当前启用的扩展 (15 个)

#### 📱 社交平台
- `slack/` - Slack 集成

#### 🧠 内存与上下文
- `memory-core/` - 文件支持的 memory search 工具
- `memory-lancedb/` - LanceDB 向量存储的长期 memory（auto-recall/capture）

#### 📱 手机控制
- `phone-control/` - 手机控制功能

#### 🛠️ 开发工具
- `diffs/` - 代码差异查看器
- `device-pair/` - 设备配对
- `copilot-proxy/` - Copilot 代理
- `diagnostics-otel/` - OpenTelemetry 诊断
- `llm-task/` - LLM 任务处理
- `lobster/` - Lobster 功能
- `open-prose/` - 文本处理

#### ⚙️ 基础设施
- `shared/` - 跨扩展共享库
- `test-utils/` - 测试工具
- `thread-ownership/` - 线程所有权管理
- `acpx/` - ACP 扩展

## 总结

| 目录/文件 | 作用 |
|-----------|------|
| `context-engine/` | **接口层** - 定义契约，实际逻辑在别处 |
| `pi-embedded-runner/compact.ts` | **实际压缩逻辑** - session.compact() 调用 |
| `pi-embedded-runner/history.ts` | 历史消息管理（limitHistoryTurns） |
| `pi-embedded-runner/system-prompt.ts` | 系统 prompt 构建 |
| `pi-embedded-runner/run/attempt.ts` | 上下文组装 pipeline（sanitize → validate → limit → repair） |

---

## Android 节点相关架构

### Android WebSocket 连接拓扑

**核心原则**: Android 只作为 WebSocket 客户端连接到网关，不启动任何 WebSocket 服务器。

#### 架构拓扑

```
┌─────────────────────────────────────────┐
│         Gateway (Master Machine)        │
│      (macOS/Linux/Windows WebSocket     │
│            Server @ :18789)             │
└─────────────────────────────────────────┘
          △              △
          │              │
    (WS客户端)      (WS客户端)
          │              │
   ┌──────┴──┐      ┌────┴──────┐
   │ role:   │      │ role:      │
   │operator │      │"node"      │
   │(UI/控制)│      │(执行命令)   │
   └─────────┘      └────────────┘

   统一运行在 Android 手机上的两个 WebSocket 连接
```

#### 两个连接的职责

1. **operator 连接** (role: "operator")
   - scopes: `["operator.read", "operator.write", "operator.talk.secrets"]`
   - 用途: UI 操作、聊天交互
   - 代码: `apps/android/app/src/main/java/ai/openclaw/app/node/ConnectionManager.kt` → `buildOperatorConnectOptions()`

2. **node 连接** (role: "node")
   - capabilities: 相机、位置、Canvas、语音等
   - commands: 支持的执行命令（camera.snap, canvas.eval 等）
   - 用途: 接收网关推送的命令，执行后回复结果
   - 代码: `apps/android/app/src/main/java/ai/openclaw/app/node/ConnectionManager.kt` → `buildNodeConnectOptions()`

#### 关键限制

| 特性 | 说明 |
|------|------|
| **连接方向** | 单向：Android → 网关（客户端→服务器） |
| **服务器启动** | Android 不启动任何 WebSocket 服务器 |
| **外部连接** | 不可以。其他客户端无法直接连接到 Android |
| **中转通信** | 所有其他设备要和 Android 通信，都必须通过网关中转 |
| **事件推送** | 网关推送 `node.invoke.request` 事件到 Android 的 node 连接 |

#### WebSocket 消息流向

```
1. Android operator 连接 → 发送 chat.send 请求 → 网关
2. 网关处理后 → 推送 chat 事件 → Android operator 连接
3. 网关推送命令 → node.invoke.request 事件 → Android node 连接
4. Android 执行命令 → 发送 node.invoke.result 请求 → 网关
```

#### 相关代码位置

| 模块 | 文件 | 功能 |
|------|------|------|
| **连接管理** | `apps/android/app/src/main/java/ai/openclaw/app/gateway/GatewaySession.kt` | WebSocket 连接、消息收发 |
| **连接配置** | `apps/android/app/src/main/java/ai/openclaw/app/node/ConnectionManager.kt` | 两个 role 的连接选项构建 |
| **运行时控制** | `apps/android/app/src/main/java/ai/openclaw/app/NodeRuntime.kt` | connect/disconnect 生命周期 |
| **聊天功能** | `apps/android/app/src/main/java/ai/openclaw/app/chat/ChatController.kt` | operator 连接的聊天操作 |
| **命令执行** | `apps/android/app/src/main/java/ai/openclaw/app/node/InvokeDispatcher.kt` | node 连接的命令处理 |

---

## Windows/Linux Node-Host 对应架构

### 现状

| 平台 | GUI 应用 | Node 功能 | 说明 |
|------|----------|-----------|------|
| **macOS** | ✅ 菜单栏应用 | ✅ 通过 app 内置 | 完整功能 |
| **iOS** | ✅ iOS 应用 | ✅ 内置 | 完整功能 |
| **Android** | ✅ Android 应用 | ✅ 内置 | 完整功能 |
| **Windows** | ❌ 无 | ✅ CLI (`node-host`) | 计划中 |
| **Linux** | ❌ 无 | ✅ CLI (`node-host`) | 计划中 |

### Windows/Linux Node 实现

Windows 和 Linux 没有独立的 GUI 伴侣应用，但可以通过 **CLI 方式**提供 node 功能：

- **实现文件**: `src/node-host/runner.ts`
- **运行命令**: `openclaw node-host`
- **连接方式**: 与 Android/iOS 相同的 WebSocket 客户端模式，以 `role: "node"` 连接到 Gateway
- **支持的能力**:
  - `system` - 系统命令执行
  - `browser` - 浏览器代理（可选）
  - `system.run` - 命令执行
  - `exec approvals` - 执行审批

```typescript
// src/node-host/runner.ts:145-155
const client = new GatewayClient({
  role: "node",
  scopes: [],
  caps: ["system", ...(browserProxyEnabled ? ["browser"] : [])],
  commands: [
    ...NODE_SYSTEM_RUN_COMMANDS,
    ...NODE_EXEC_APPROVALS_COMMANDS,
    ...(browserProxyEnabled ? [NODE_BROWSER_PROXY_COMMAND] : []),
  ],
  // ...
});
```

### 官方建议

- Windows 用户推荐使用 **WSL2** 运行 Gateway + node-host
- 独立的 Windows GUI 伴侣应用正在计划中（见 `docs/platforms/index.md`）
