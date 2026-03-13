# OpenClaw Core Design - src 目录功能分析

## 整体架构概览

OpenClaw 的核心代码位于 `src/` 目录，实现了一个多通道、AI-驱动的消息网关系统。主要分为以下几大层次：

1. **通信层** - 多消息通道集成与路由
2. **应用层** - CLI、命令处理、代理管理
3. **服务层** - 网关服务器、计算服务
4. **数据层** - 配置、会话、内存存储
5. **支撑层** - 日志、媒体、安全、基础设施

---

## 核心通信层

### channels/ - 聊天通道系统
**职责**: 管理多个消息通道集成

- **集成通道**: Telegram、WhatsApp、Discord、IRC、Google Chat、Slack、Signal、iMessage、Line
- **核心功能**:
  - `registry.ts` - 通道注册及元数据管理
  - `session.ts` - 会话（conversation）生命周期管理
  - `targets.ts` - 消息目标地址解析
- **特性**:
  - `allowlist/` - 消息过滤和允许列表
  - `typing-lifecycle/` - 输入指示器生命周期（"对方正在输入"）
  - `draft-stream-controls/` - 草稿流控制和部分消息处理
  - `sender-identity/` - 发送者身份跟踪

### routing/ - 消息路由引擎
**职责**: 智能路由消息到对应的代理和处理器

- **核心功能**:
  - `resolve-route.ts` - 根据上下文解析消息路由目标
  - `bindings.ts` - 账户到通道的多对多绑定管理
  - `session-key.ts` - 会话密钥和账户关联
- **特性**:
  - 多账户支持
  - 默认账户提醒机制
  - 账户查询和验证

---

## 命令与CLI系统

### cli/ - 命令行接口框架
**职责**: OpenClaw 完整的 CLI 工具集实现

- **核心模块**:
  - `program.ts` - CLI 主程序框架
  - `route.ts` - CLI 路由分发
  - `gateway-cli/` - 网关相关命令组
- **命令分类**:
  - `channels-cli/` - 通道管理
  - `models-cli/` - 模型配置
  - `plugins-cli/` - 插件管理
  - `nodes-cli/` - 节点管理
  - `daemon-cli/` - 守护进程控制
  - `pairing-cli/` - 设备配对
  - `config-cli/` - 配置管理
  - `skills-cli/` - 技能管理
- **工具模块**:
  - `parser/` - 参数解析和验证
  - 输出格式化（JSON、表格等）
  - `progress/` - 进度显示

### commands/ - 高级命令和业务逻辑
**职责**: CLI 命令的业务实现层

- **核心命令模块**:
  - `agents.ts` - AI 代理管理
  - `channels.ts` - 通道操作
  - `models.ts` - 模型配置
  - `sessions.ts` - 会话管理
  - `webhook/` - 网络钩子处理
- **初始化系统** (onboarding/):
  - `onboard-auth.ts` - 认证配置向导
  - `onboard-channels.ts` - 通道接入向导
  - `onboard-models.ts` - 模型选择向导
- **特性**:
  - 命令绑定和聚合
  - 交互式配置步骤
  - 状态验证

---

## 服务与网关

### gateway/ - 消息网关服务器（核心后端）
**职责**: 完整的 WebSocket + HTTP 服务实现，OpenClaw 的心脏

- **核心实现**:
  - `server.impl.ts` - 服务器主体实现
  - `server-methods/` - RPC 方法库
- **主要模块**:
  - `server-chat.ts` - 聊天消息处理和流转
  - `server-channels.ts` - 通道生命周期管理
  - `server-plugins.ts` - 动态插件加载和执行
  - `server-cron.ts` - 定时任务调度
  - `server-sessions.ts` - 会话状态管理
- **协议层**:
  - `auth.ts` - 认证和授权
  - `protocol/` - 网关通信协议
- **特性**:
  - 连接池管理
  - 健康检查
  - 广播消息分发

### agents/ - AI 代理和会话生命周期
**职责**: AI 代理实例的创建、管理和运行

- **代理生命周期**:
  - `acp-spawn.ts` - 代理进程生成
  - `agent-scope.ts` - 代理上下文隔离
- **认证和配置**:
  - `auth-profiles/` - 认证配置文件管理
- **运行时实现**:
  - `cli-runner/` - CLI 后端运行时
  - `pi-embedded-runner/` - Pi 嵌入式运行时
  - `subagent/` - 子代理生成和管理机制
- **会话管理**:
  - 会话创建、恢复、清理
  - 并发控制
  - 资源生命周期

---

## 插件和扩展系统

### plugins/ - 插件管理框架
**职责**: 动态插件加载、生命周期管理和 hook 系统

- **核心功能**:
  - `registry.ts` - 全局插件注册表
  - `manifest.ts` - 插件清单解析和验证
  - `loader.ts` - 插件动态加载
- **运维功能**:
  - `discovery.ts` - 插件自动发现（本地、npm）
  - `install.ts/uninstall.ts` - 插件安装卸载
  - `enable-disable.ts` - 插件启用禁用
- **扩展机制**:
  - `hooks.ts` - 统一 hook 系统（命令注入、UI 扩展等）
  - `provider-discovery.ts` - 动态提供商发现
  - HTTP 路由注入
- **安全性**:
  - 沙盒隔离
  - 权限验证

### providers/ - 提供商和连接器
**职责**: 第三方服务集成和 API 管理

- **模型提供商** - 对接多个 LLM 服务
- **认证提供商** - OAuth、API Key 管理
- **集成验证** - 连接测试和有效性检查
- **工具库** - `provider-shared/` 共享工具

---

## 数据和存储

### memory/ - 向量数据库和嵌入式搜索
**职责**: 完整的向量搜索和语义记忆管理系统

- **存储引擎**:
  - `manager.ts` - 内存管理器
  - `search-manager.ts` - 搜索引擎
  - 支持 sqlite-vec、LanceDB 等后端
- **嵌入模型**:
  - `embeddings.ts` - 嵌入模型接口
  - 支持 OpenAI、Mistral、Voyage、Ollama、Gemini 等
- **高级搜索**:
  - `batch-*.ts` - 批量嵌入处理（OpenAI、Gemini、Voyage）
  - `hybrid.ts` - 混合搜索（向量 + 关键词）
  - `mmr.ts` - 最大边际相关性算法
  - `temporal-decay.ts` - 时间衰减和新鲜度加权
- **查询能力**:
  - 范围管理（QMD）
  - 时间范围过滤
  - 相关性排序

### sessions/ - 会话持久化
**职责**: 会话数据存储和恢复

- **功能**:
  - 会话序列化/反序列化
  - 历史管理和清理
  - 成本使用追踪（token/API costs）
  - 备份恢复
  - 文件锁定机制（防止并发冲突）

### config/ - 全局配置管理
**职责**: 统一的配置参数管理

- **核心文件**:
  - `config.ts` - 配置主体实现
  - `schema.ts` - 配置 JSON Schema 定义
  - `types.ts` - TypeScript 类型
- **管理范围**:
  - 通道配置
  - 代理和模型配置
  - 插件配置
  - 守护进程配置
  - 网关配置
  - 日志配置
  - 沙盒配置
- **特性**:
  - 配置验证和迁移
  - 环境变量替换
  - 路径规范化
  - 缺省值管理

---

## AI 代理核心

### context-engine/ - 上下文管理引擎
**职责**: 代理运行时的上下文和对话历史管理

- **功能**:
  - `registry.ts` - 上下文提供商注册
  - 代理执行上下文初始化
  - 对话历史追踪
  - 上下文窗口优化管理

---

## 媒体和文件处理

### media/ - 媒体处理系统
**职责**: 多种媒体格式的获取、解析和处理

- **核心接口**:
  - `fetch.ts` - 媒体获取（URL、本地文件）
  - `store.ts` - 媒体存储管理
  - `parse.ts` - 内容解析
- **处理能力**:
  - **图像**: `image-ops.ts` - 图像缩放、转换、优化
  - **音频**: `audio.ts`、`ffmpeg-exec.ts` - 音频转码、提取
  - **PDF**: `pdf-extract.ts` - 文本提取
  - **编码**: `base64.ts` - Base64 编解码
  - **类型检测**: `mime.ts` - MIME 类型识别
- **安全性**:
  - 文件大小限制检查
  - 策略验证
  - 病毒/恶意软件检查接口

### pairing/ - 设备配对系统
**职责**: 跨设备配对和设置流程

- **核心模块**:
  - `pairing-challenge.ts` - 配对挑战
  - `setup-code.ts` - 设置代码生成和验证
  - `pairing-messages.ts` - 配对协议消息
  - `pairing-store.ts` - 已配对设备存储
- **功能**:
  - 安全配对流程
  - 代码校验
  - 设备标识

---

## 支撑和工具模块

### logging/ - 日志系统
**职责**: 统一的应用日志管理

- **功能**:
  - `logger.ts` - 日志核心接口
  - `console.ts` - 控制台日志和重定向
  - `diagnostic.ts` - 诊断日志
- **特性**:
  - 日志级别管理（DEBUG、INFO、WARN、ERROR）
  - 文件日志持久化
  - 控制台输出捕获
  - 时间戳和上下文

### terminal/ - 终端 UI 组件库
**职责**: CLI 用户界面组件

- **核心组件**:
  - `table.ts` - 表格渲染
  - `palette.ts` - 颜色和样式管理
  - `ansi.ts` - ANSI 控制码处理
  - `prompt-select-styled.ts` - 样式化选择提示
- **特性**:
  - ANSI 256 色和真色支持
  - 主题管理
  - 响应式布局

### tts/ - 文本转语音
**职责**: TTS 服务集成

- **功能**:
  - 多语言 TTS 处理
  - `prepare-text.ts` - 文本预处理
  - 声音合成和流式输出

### infra/ - 基础设施和系统工具集
**职责**: 底层系统交互和通用工具集（200+ 模块）

- **进程管理**:
  - 进程执行和监控
  - 子进程通信
  - 资源管理
- **网络功能**:
  - SSH 隧道
  - Bonjour 服务发现
  - 网络连接检测
- **文件系统**:
  - 路径操作
  - 文件监听
  - 权限管理
- **执行管理**:
  - `exec-approvals/` - 执行批准系统
  - `bash-tools/` - Bash 脚本执行
- **系统监控**:
  - `heartbeat/` - 保活监控
  - 状态检查
- **设备管理**:
  - `device-identity/` - 设备标识
  - 配对工具
- **更新机制**:
  - 版本检查
  - 自动更新

### 其他支撑模块

- **shared/** - 跨模块共享工具和类型
- **utils/** - 通用工具函数库（数组、对象、字符串、日期）
- **types/** - 全局类型定义
- **security/** - 安全相关（加密、授权等）
- **hooks/** - 全局 hook 系统（生命周期钩子）
- **link-understanding/** - 链接识别和内容提取
- **markdown/** - Markdown 解析和处理
- **browser/** - 浏览器自动化（Puppeteer 等）
- **process/** - 进程管理工具
- **daemon/** - 守护进程相关
- **cron/** - 定时任务系统
- **secrets/** - 凭证和密钥管理
- **i18n/** - 国际化提供商

---

## 数据流和关键路径

### 消息接收和处理流程
1. 消息到达 **channels/** 中的某个通道（如 Telegram）
2. **routing/** 根据账户、会话等信息解析路由目标
3. **gateway/** 的 server-chat.ts 接收并分发
4. **agents/** 中对应的 AI 代理处理
5. 响应消息通过 **channels/** 发回

### 插件加载流程
1. **plugins/registry.ts** 发现本地和 npm 插件
2. **plugins/loader.ts** 动态加载插件代码
3. 插件注入 hooks 到 **gateway/** 和其他核心模块
4. 插件可访问 **providers/** 和通道系统

### 会话和数据持久化
1. **agents/** 创建会话，**sessions/** 持久化
2. **memory/** 存储向量嵌入和语义记忆
3. **config/** 存储全局配置
4. 启动时从磁盘恢复状态

---

## 守护进程和系统集成

### daemon/ - 跨平台守护进程管理
**职责**: 将 Gateway 注册为系统服务，支持开机自启和后台运行

#### 统一接口层
[src/daemon/service.ts](src/daemon/service.ts) 定义了 `GatewayService` 抽象接口：

```typescript
type GatewayService = {
  install: (args: GatewayServiceInstallArgs) => Promise<void>;
  uninstall: (args: GatewayServiceManageArgs) => Promise<void>;
  stop: (args: GatewayServiceControlArgs) => Promise<void>;
  restart: (args: GatewayServiceControlArgs) => Promise<GatewayServiceRestartResult>;
  isLoaded: (args: GatewayServiceEnvArgs) => Promise<boolean>;
  readCommand: (env: GatewayServiceEnv) => Promise<GatewayServiceCommandConfig | null>;
  readRuntime: (env: GatewayServiceEnv) => Promise<GatewayServiceRuntime>;
};
```

根据平台动态选择实现：
```typescript
const GATEWAY_SERVICE_REGISTRY = {
  darwin: { /* launchd 实现 */ },
  linux: { /* systemd 实现 */ },
  win32: { /* schtasks 实现 */ },
};
```

#### macOS - launchd 实现

**文件位置**：`src/daemon/launchd.ts`、`src/daemon/launchd-plist.ts`

**plist 生成** ([launchd-plist.ts:96-120](src/daemon/launchd-plist.ts#L96)):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" ...>
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>ai.openclaw.gateway</string>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>ThrottleInterval</key>
    <integer>1</integer>
    <key>ProgramArguments</key>
    <array>
      <string>openclaw</string>
      <string>gateway</string>
      <string>run</string>
    </array>
    <key>StandardOutPath</key>
    <string>~/.openclaw/gateway.log</string>
    <key>StandardErrorPath</key>
    <string>~/.openclaw/gateway.err.log</string>
    <key>EnvironmentVariables</key>
    <dict><!-- 自定义环境变量 --></dict>
  </dict>
</plist>
```

**关键特性**：
- **位置**：`~/.LaunchAgents/ai.openclaw.gateway.plist`（或自定义标签）
- **KeepAlive**：设为 `true`，macOS 会自动重启失败的服务
- **ThrottleInterval**：1 秒，防止快速重启产生延迟
- **RunAtLoad**：启用时自动运行，无需手动启动

**安装流程** ([launchd.ts:422-488](src/daemon/launchd.ts#L422)):
1. 清理旧的遗留 plist 文件
2. 创建目录结构：`~/.LaunchAgents/`
3. 生成 plist 文件
4. 执行 `launchctl bootout` 和 `launchctl bootstrap`

```bash
launchctl bootout gui/{uid}/ai.openclaw.gateway
launchctl bootstrap gui/{uid} ~/.LaunchAgents/ai.openclaw.gateway.plist
```

#### Linux - systemd 实现

**文件位置**：`src/daemon/systemd.ts`、`src/daemon/systemd-unit.ts`

**unit 文件生成** ([systemd-unit.ts:41-66](src/daemon/systemd-unit.ts#L41)):
```ini
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=openclaw gateway run
Restart=always
RestartSec=5
TimeoutStopSec=30
TimeoutStartSec=30
SuccessExitStatus=0 143
KillMode=control-group
WorkingDirectory=/home/user/.openclaw  # 如果指定

[Install]
WantedBy=default.target
```

**关键特性**：
- **位置**：`~/.config/systemd/user/openclaw-gateway.service`
- **Restart=always**：服务挂掉后自动重启，延迟 5 秒
- **SuccessExitStatus=0 143**：0 正常退出，143 是 SIGTERM 信号（也认为成功）
- **After=network-online.target**：确保网络就绪后启动
- **UserLinger**：自动启用，允许用户会话离线时后台运行

**安装流程**：
1. 启用 systemd 用户 lingering：`systemctl --user enable-linger`
2. 创建目录：`~/.config/systemd/user/`
3. 生成 unit 文件
4. 执行 `systemctl --user daemon-reload` 和 `systemctl --user enable openclaw-gateway`

```bash
systemctl --user daemon-reload
systemctl --user enable openclaw-gateway.service
systemctl --user start openclaw-gateway.service
```

#### Windows - Task Scheduler 实现

**文件位置**：`src/daemon/schtasks.ts`

**任务脚本生成** ([schtasks.ts:235-269](src/daemon/schtasks.ts#L235)):
```batch
@echo off
rem OpenClaw Gateway Service
cd /d %USERPROFILE%\.openclaw
set OPENCLAW_PROFILE=default
openclaw gateway run
```

**关键特性**：
- **位置**：`%AppData%\Microsoft\Windows\Start Menu\Programs\Startup\` + 备用任务脚本
- **触发条件**：`ONLOGON`（登录时运行）
- **权限等级**：`LIMITED`（非管理员）
- **降级方案**：若 schtasks 不可用，退而启用启动文件夹快捷方式

**安装流程**：
1. 生成 `~\.openclaw\gateway.cmd` 脚本
2. 执行 `schtasks /create` 创建计划任务

```powershell
schtasks /create /tn "OpenClaw Gateway" /tr "cmd.exe /c %USERPROFILE%\.openclaw\gateway.cmd" /sc ONLOGON /rl LIMITED
```

若失败，创建启动文件夹快捷方式：
```
%AppData%\Microsoft\Windows\Start Menu\Programs\Startup\OpenClawGateway.cmd
```

### CLI 命令入口

**文件**：[src/cli/daemon-cli/register.ts](src/cli/daemon-cli/register.ts)

**支持的命令**：
```bash
openclaw daemon install         # 安装系统服务
openclaw daemon uninstall       # 卸载系统服务
openclaw daemon start           # 启动服务
openclaw daemon stop            # 停止服务
openclaw daemon restart         # 重启服务
openclaw daemon status          # 显示状态和诊断
```

**安装入口** ([daemon-cli/install.ts:18](src/cli/daemon-cli/install.ts#L18)):
1. 解析端口和运行时参数
2. 检查服务是否已安装
3. 生成安装令牌（如需要）
4. 调用 `resolveGatewayService().install(args)`
5. 输出安装结果和配置

### 数据结构

| 类型 | 位置 | 作用 |
|------|------|------|
| `GatewayServiceInstallArgs` | [service-types.ts](src/daemon/service-types.ts) | 安装参数：命令行、工作目录、环境变量、描述 |
| `GatewayServiceControlArgs` | [service-types.ts](src/daemon/service-types.ts) | 控制参数：输出流、环境 |
| `GatewayServiceRuntime` | [service-runtime.ts](src/daemon/service-runtime.ts) | 运行时状态：PID、激活状态、启动时间 |

### 环境变量配置

支持通过环境变量自定义行为：

| 变量 | 平台 | 作用 |
|------|------|------|
| `OPENCLAW_LAUNCHD_LABEL` | macOS | 自定义 plist 标签（默认：`ai.openclaw.gateway`） |
| `OPENCLAW_PROFILE` | 全部 | 配置文件名前缀 |
| `OPENCLAW_WINDOWS_TASK_NAME` | Windows | 任务计划任务名称 |
| `OPENCLAW_TASK_SCRIPT` | Windows | 自定义脚本路径 |

---

## 关键设计原则

1. **多通道一致性** - 所有通道通过统一的 channels/ 和 routing/ 接口
2. **插件优先** - 核心功能通过 hook 系统供插件扩展
3. **会话隔离** - 每个代理有独立的会话和上下文
4. **配置集中化** - 全局配置通过 config/ 统一管理
5. **异步和流式** - 支持流式消息、部分响应和长操作
6. **媒体处理完整** - 内置图像、音频、PDF 等处理能力
7. **向量记忆系统** - 通过向量数据库实现语义搜索和上下文记忆
8. **跨平台守护进程** - 统一接口+ 平台特定实现（launchd/systemd/schtasks）
