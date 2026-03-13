# Context Engineering 设计文档

## 概述

ContextEngine 是一个插件化框架，负责 agent 运行时的消息生命周期管理。支持消息重排、摘要、系统提示增强、批处理等高级上下文操作。

## 核心架构

**关键文件**：
- `src/context-engine/types.ts` - 核心接口定义
- `src/context-engine/registry.ts` - 引擎注册与解析
- `src/context-engine/legacy.ts` - 默认实现
- `src/context-engine/init.ts` - 初始化保证
- `src/agents/pi-embedded-runner/run/attempt.ts` - 集成调用

---

## 消息处理流程（简化模型）

### 阶段流转

```
阶段 1: Bootstrap (初始化)
    输入: session.json (历史存储)
    操作: contextEngine.bootstrap()
    输出: 引擎初始化完成

        ↓

阶段 2: 消息准备 + Assemble (组装)
    输入: activeSession.messages
         [user, assistant, tool_result, ...]

    准备流程:
    - sanitizeSessionHistory()  ← 清理无效消息
    - validateTurns()           ← 验证结构一致性
    - limitHistoryTurns()       ← 按配置截断历史

    组装流程:
    - contextEngine.assemble(messages)
      * 可重排消息序列
      * 可摘要旧消息
      * 返回 assembled.messages

    系统提示增强:
    - assembled.systemPromptAddition
      * 可追加到 baseSystemPrompt

    输出: 新消息序列 + 增强系统提示

        ↓

阶段 3: Prompt 执行
    输入: systemPrompt + messages + userPrompt
    操作: activeSession.prompt()
    输出: assistant message (可能带 tool_use)

        ↓

阶段 4: Tool Response 摄入
    输入: tool_result message

    摄入流程:
    - contextEngine.ingest(message)
      * 逐条处理新消息
      * 可能触发更多 tool_call

    输出: 消息摄入确认

        ↓

阶段 5: AfterTurn 后处理
    输入: 完整消息快照 + 新增消息范围

    处理流程 (三层回退):
    1. contextEngine.afterTurn()        ← 统一处理所有新消息
    2. contextEngine.ingestBatch()      ← 批量摄入回退
    3. contextEngine.ingest() 逐条      ← 兜底方案

    输出: 持久化 + 后期处理完成
```

---

## 时序图

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Attempt   │    │ ContextEngine│    │  Session     │    │    Model     │
└──────┬──────┘    └──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │                   │
       │ bootstrap()       │                   │                   │
       ├──────────────────>│                   │                   │
       │                   │ init from session │                   │
       │                   │<──────────────────┤                   │
       │                   │                   │                   │
       │ sanitize/validate/limit               │                   │
       ├──────────────────────────────────────>│                   │
       │                   │                   │                   │
       │ assemble(messages)│                   │                   │
       ├──────────────────>│                   │                   │
       │<─messages + system_addition──────────┤                   │
       │                   │                   │                   │
       │                   │                   │ replaceMessages() │
       ├──────────────────────────────────────>│                   │
       │                   │                   │                   │
       │                   │ applySystemPrompt │                   │
       ├─────────────────────────────────────>│                   │
       │                   │                   │                   │
       │                   │                   │ prompt()          │
       │                   │                   ├──────────────────>│
       │                   │                   │<── assistant msg──┤
       │                   │                   │   (+ tool_use)    │
       │                   │                   │                   │
       │    [Tool Execution Loop]             │                   │
       │                   │                   │                   │
       │    tool_result added to session       │                   │
       ├──────────────────────────────────────>│                   │
       │                   │                   │                   │
       │ ingest(tool_result)                   │                   │
       ├──────────────────>│                   │                   │
       │                   │ (摄入新消息)       │                   │
       │                   │                   │                   │
       │    [可能继续 Tool Call]               │                   │
       │                   │                   │                   │
       │ afterTurn()       │                   │                   │
       ├──────────────────>│                   │                   │
       │                   │ (处理全量新消息)   │                   │
       │                   │ (持久化 + 后期逻辑)│                   │
       │                   │                   │                   │
       │<── 处理完成 ──────│                   │                   │
       │                   │                   │                   │
```

---

## 数据流向详解

### 消息序列变化

```
bootstrap() 后
  ↓
raw_messages = [msg1, msg2, ...]

sanitize() → validate() → limit()
  ↓
limited_messages = [adjusted_seq]

assemble()
  ↓
assembled.messages = [reordered/summarized]
assembled.systemPromptAddition = "..."

需要替换到 activeSession
  ↓
activeSession.replaceMessages(assembled.messages)

prompt() 执行
  ↓
新增: assistant + possible tool_use

tool 执行
  ↓
新增: tool_result

ingest() 逐条摄入
  ↓
引擎记录新消息

afterTurn() 统一处理
  ↓
messagesSnapshot = 完整快照 (从 prePromptMessageCount 开始)
引擎执行持久化+后期逻辑
```

---

## 四个核心方法

| 方法 | 何时调用 | 输入 | 输出 | 职责 |
|------|---------|------|------|------|
| **bootstrap()** | attempt 开始，有旧 session | sessionFile | - | 从历史恢复引擎状态 |
| **assemble()** | prompt 前 | 历史消息 | messages[] + systemPromptAddition | 消息重排/摘要 + 系统提示增强 |
| **ingest()** | 每个 tool_result 后 | 单条消息 | - | 逐条摄入新消息 |
| **afterTurn()** | 运行完成后 | 消息快照 + 范围 | - | 统一处理运行结果 (最优) |

---

## 系统提示增强机制

### 阶段演变

```
构建阶段:
  buildEmbeddedSystemPrompt()
  → baseSystemPrompt = "You are an assistant. Tools: [...]"

组装阶段 (Context Engine 参与):
  contextEngine.assemble()
  → assembled.systemPromptAddition = "Use this context: ..."

融合阶段:
  systemPromptText = baseSystemPrompt + "\n" + systemPromptAddition
  → applySystemPromptOverrideToSession()

发送阶段:
  model.prompt(finalSystemPrompt, finalMessages, userPrompt)
```

### 关键特性

- **动态注入**：在 prompt 发送前动态追加系统指令
- **引擎建议**：assemble() 返回的 systemPromptAddition 是 context engine 的建议
- **不覆盖**：使用 prepend（追加）而非覆盖，保留原始系统提示完整性

---

## 调用点代码位置

### attempt.ts 中的集成

**Bootstrap** (L2388-2395)
```
htmlspecialchars
if (hadSessionFile && params.contextEngine?.bootstrap) {
  await params.contextEngine.bootstrap({ sessionId, sessionKey, sessionFile });
}
```

**Assemble** (L2099-2126)
```
const assembled = await params.contextEngine.assemble({
  sessionId, sessionKey,
  messages: activeSession.messages,
  tokenBudget: params.contextTokenBudget,
});

if (assembled.messages !== activeSession.messages) {
  activeSession.agent.replaceMessages(assembled.messages);
}
if (assembled.systemPromptAddition) {
  systemPromptText = prependSystemPromptAddition({
    systemPrompt: systemPromptText,
    systemPromptAddition: assembled.systemPromptAddition,
  });
}
```

**Ingest** (L2649-2668)
```
for (const msg of newMessages) {
  await params.contextEngine.ingest({
    sessionId, sessionKey, message: msg
  });
}
```

**AfterTurn** (L2670-2710)
```
if (typeof params.contextEngine.afterTurn === 'function') {
  await params.contextEngine.afterTurn({
    sessionId, sessionKey, sessionFile,
    messages: messagesSnapshot,
    prePromptMessageCount,
    runtimeContext: afterTurnRuntimeContext,
  });
} else {
  // 三层回退到 ingestBatch() → ingest()
}
```

---

## 关键设计决策

1. **三层 afterTurn 回退**：
   - 优先级高的引擎实现 afterTurn() 享受方便
   - 降级到 ingestBatch() 避免逐条开销
   - 兜底到 ingest() 保证最大兼容性

2. **系统提示追加而非替换**：
   - 保留原始系统提示的完整性和上下文
   - 引擎建议以补充形式加入
   - 便于问题诊断和审计

3. **消息快照 + 范围标记**：
   - afterTurn 接收完整快照 + `prePromptMessageCount` 指针
   - 引擎自行计算新增消息: `messagesSnapshot.slice(prePromptMessageCount)`
   - 支持引擎灵活处理（批处理、去重、重排等）

4. **运行时上下文传递**：
   - runtimeContext 是灵活的 Record<string, unknown>
   - 允许引擎按需访问运行时数据
   - 避免预定义固定接口导致的扩展困难

---

## 插件化架构

### 注册与解析

```typescript
// 注册新引擎
registerContextEngine('my-engine', () => new MyContextEngine());

// 解析引擎
const engine = await resolveContextEngine(config);
// 按优先级：
//   1. config.plugins.slots.contextEngine (显式指定)
//   2. "legacy" (默认)
```

### 初始化保证

```typescript
// 一次性初始化
ensureContextEnginesInitialized();
// 确保 'legacy' 总是可用的安全后备方案
```

---

## 典型引擎实现要点

### LegacyContextEngine

- `ingest()` → no-op (SessionManager 负责)
- `assemble()` → pass-through (返回原消息 + 空系统提示)
- `compact()` → 委托给 compactEmbeddedPiSessionDirect
- `afterTurn()` → no-op (运行时直接持久化)

### 新引擎可选能力

- `bootstrap()` - 从历史导入
- `ingestBatch()` - 批量优化
- `afterTurn()` - 统一后处理
- `prepareSubagentSpawn()` - 子代理前准备
- `onSubagentEnded()` - 子代理结束通知
- `dispose()` - 资源清理

---

## 性能考虑

1. **Assemble 开销**：
   - 只在 prompt 前一次性调用
   - 消息重排/摘要成本被分摊

2. **Ingest 开销**：
   - Tool response 后逐条调用
   - 鼓励实现 ingestBatch() 以批处理

3. **AfterTurn 统一**：
   - 运行完成后的统一处理
   - 避免大量逐条 ingest 调用
   - 允许引擎做持久化、后期决策等

---

## 调试与诊断

- 查看 systemPrompt 是否被增强：`log.debug()` 输出 systemPromptAddition 长度
- 检查消息是否被重排：比对 assembled.messages 与原始消息序列
- 追踪 ingest 失败：catch() 块记录警告，不中断流程
- 审计 afterTurn：运行时上下文包含完整操作信息
