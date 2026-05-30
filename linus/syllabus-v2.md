# NanoClaw 两周教学大纲（V2 重构版）

> 项目定位：NanoClaw 是一个多渠道个人 Claude 助手，支持持久化记忆、定时任务和容器隔离的 Agent 执行环境。
> 技术栈：Host 端 Node.js 20+ / pnpm，Container 端 Bun，SQLite（better-sqlite3 / bun:sqlite），Claude Agent SDK，Docker / Apple Container。

---

## 第一周：基础架构与核心机制

---

### Day 1 — 项目全景与环境搭建

**目标：** 建立 NanoClaw 的全局认知，搭建开发环境，理解项目定位与核心设计哲学。

**知识点：**

1. **项目定位与核心价值**
   - NanoClaw 是什么：多渠道个人 AI 助手，支持 WhatsApp / Telegram / Slack / Discord / Gmail 等渠道
   - 核心能力：多渠道接入、持久化记忆、定时任务、容器隔离执行
   - 与普通 ChatBot 的区别：每个 Agent Session 有独立容器，通过 SQLite DB 作为唯一 IO 机制

2. **技术栈概览**
   - Host 端：Node.js 20+、pnpm 10、better-sqlite3、@clack/prompts
   - Container 端：Bun 1.3+、bun:sqlite、@anthropic-ai/claude-agent-sdk
   - 为什么 Host 和 Container 使用不同运行时（参考 `docs/build-and-runtime.md`）
   - 容器运行时：Docker Desktop / Docker Engine / Apple Container

3. **项目目录结构**
   - `src/` — Host 端核心代码（路由、交付、容器管理、渠道、数据库）
   - `container/agent-runner/` — Container 内 Agent Runner（轮询循环、MCP 工具、Provider 抽象）
   - `container/skills/` — 容器内置技能
   - `docs/` — 架构文档（重点阅读 SPEC.md、architecture.md）
   - `setup/` — 安装与配置脚本
   - `groups/` — Agent Group 工作区（CLAUDE.md、技能、日志）
   - `.claude/skills/` — Claude Code 技能系统
   - `bin/ncl` — CLI 入口

4. **开发环境搭建**
   - 安装 Node.js 20+、pnpm 10、Docker
   - `pnpm install` 安装依赖
   - `pnpm run build` 编译 TypeScript
   - `pnpm run dev` 开发模式启动
   - `pnpm run test` 运行测试（vitest）
   - `cd container/agent-runner && bun install` 安装容器端依赖
   - `bun test` 运行容器端测试

5. **关键配置文件**
   - `package.json` — 项目依赖与脚本
   - `tsconfig.json` — TypeScript 配置
   - `vitest.config.ts` — 测试配置（排除 container/agent-runner/）
   - `.mcp.json` — MCP 服务器配置
   - `.env` — 环境变量（ANTHROPIC_API_KEY / CLAUDE_CODE_OAUTH_TOKEN）

**阅读材料：**
- `README.md`
- `docs/SPEC.md`（Architecture 章节）
- `docs/build-and-runtime.md`

**实践任务：**
- 克隆项目，完成环境搭建，确保 `pnpm run typecheck` 和 `pnpm run test` 通过
- 画出项目目录结构思维导图

---

### Day 2 — 架构深潜：Host 与 Container 的双进程模型

**目标：** 深入理解 NanoClaw 的核心架构——Host 进程与 Container 进程的职责划分与通信机制。

**知识点：**

1. **双进程架构**
   - Host（macOS / Linux）：主 Node.js 进程，负责渠道接入、消息路由、容器编排、调度
   - Container（Linux VM）：隔离环境中的 Agent Runner，负责轮询消息、调用 AI Provider、执行 MCP 工具
   - 两者通过挂载的 SQLite DB 通信，无 IPC 文件、无 stdin 管道

2. **核心设计哲学：一切都是消息**
   - `messages_in`（Host → Agent Runner）：chat、task、webhook、system 等类型
   - `messages_out`（Agent Runner → Host）：chat、chat-sdk、system 等类型
   - 路由字段（platform_id、channel_type、thread_id）在 Agent Runner 侧被剥离，Agent 只看到内容

3. **Host 端核心模块**
   - `src/index.ts` — 编排器：初始化 DB、连接渠道、启动消息循环
   - `src/delivery.ts` — 轮询 outbound.db，通过渠道适配器交付消息，处理系统动作
   - `src/host-sweep.ts` — 60 秒扫描：停滞检测、到期消息唤醒、循环任务
   - `src/session-manager.ts` — Session 解析、inbound.db / outbound.db 创建
   - `src/container-runner.ts` — 按 Agent Group 启动容器、OneCLI 凭证注入
   - `src/router.ts` — 消息格式化与出站路由

4. **Container 端核心模块**
   - `container/agent-runner/src/index.ts` — 入口：读取配置、创建 Provider、进入轮询循环
   - `container/agent-runner/src/poll-loop.ts` — 轮询 messages_in、调用 Provider、写入 messages_out
   - `container/agent-runner/src/formatter.ts` — 消息格式化（XML 格式、按 kind 处理）
   - `container/agent-runner/src/db/` — Session DB 访问层（messages-in.ts、messages-out.ts、session-state.ts）

5. **容器挂载结构**
   ```
   /workspace/                 ← 挂载：session 文件夹（读写）
     .claude/                  ← Claude SDK session 数据
     inbound.db                ← Host 写入，Container 只读
     outbound.db               ← Container 写入，Host 轮询
     .heartbeat                ← mtime 触碰作为存活信号
     inbox/<message_id>/       ← 解码的用户附件
     outbox/<message_id>/      ← Agent 产出的附件
     agent/                    ← 挂载：Agent Group 文件夹（嵌套，读写）
       CLAUDE.md               ← Agent 指令
       skills/                 ← Agent 技能
   ```

**阅读材料：**
- `docs/architecture.md`（Core Idea、Two-Level DB、Container Lifecycle 章节）
- `docs/architecture.zh-CN.md`（中文版对照阅读）
- `src/index.ts`、`container/agent-runner/src/index.ts`

**实践任务：**
- 阅读 `src/index.ts`，梳理 Host 启动流程
- 阅读 `container/agent-runner/src/poll-loop.ts`，理解轮询循环逻辑
- 画出 Host ↔ Container 通信时序图

---

### Day 3 — 数据库架构：Central DB 与 Session DB

**目标：** 掌握 NanoClaw 的三级数据库设计，理解数据如何在 Central DB 和 Session DB 之间流转。

**知识点：**

1. **三级数据库概览**
   - Central DB（`data/v2.db`）：身份、权限、路由、配线——管理平面
   - Session Inbound DB（`inbound.db`）：Host 写入，Container 只读——Host → Container 消息
   - Session Outbound DB（`outbound.db`）：Container 写入，Host 轮询——Container → Host 消息

2. **单写者原则**
   - 每个 SQLite 文件只有一个写者
   - Host 写 Central DB 和所有 inbound.db
   - Container 只写自己的 outbound.db
   - 消除跨 Docker/Apple Container 挂载边界的写冲突

3. **Central DB 核心表**
   - `agent_groups` — Agent 工作区（folder、skills、CLAUDE.md、container_config）
   - `messaging_groups` — 平台群组/频道（channel_type、platform_id）
   - `messaging_group_agents` — 配线关系（session_mode、trigger_rules、priority）
   - `users` — 用户身份（命名空间 `<channel_type>:<handle>`）
   - `user_roles` — 角色（owner 全局、admin 可限定 Agent Group 范围）
   - `agent_group_members` — 成员关系
   - `user_dms` — DM 解析缓存
   - `sessions` — 会话注册表（id、agent_group_id、thread_id、container_status）
   - `pending_questions` — 待处理的交互式问题（卡片等待用户响应）
   - `pending_approvals` — 待审批项
   - `dropped_messages` — 丢弃消息审计

4. **Session DB 核心表**
   - `messages_in`（inbound.db）：id、kind、status、process_after、recurrence、content
   - `messages_out`（outbound.db）：id、in_reply_to、kind、content、delivered
   - `processing_ack`（outbound.db）：Container 处理状态确认
   - `session_state`（outbound.db）：Container 启动时恢复的状态
   - `agent_destinations`（inbound.db）：Agent 间通信目标（投影自 Central DB）
   - `session_routing`（inbound.db）：默认路由信息（投影自 Central DB）

5. **跨挂载可见性规则**
   - Session DB 使用 `journal_mode = DELETE`（不是 WAL），WAL 文件跨挂载不可靠
   - Host 端写入 inbound.db 采用"打开-写入-关闭"模式
   - Container 以只读模式打开 inbound.db
   - 心跳通过文件 `.heartbeat` 的 mtime 触碰，而非 DB 写入

6. **数据库迁移系统**
   - Central DB：编号迁移文件（`src/db/migrations/001-initial.ts`、`002-*.ts`...）
   - Session DB：`IF NOT EXISTS` + 临时 `ALTER TABLE` 辅助函数
   - 迁移运行器通过 `schema_version` 表追踪版本

7. **Seq 编号不变量**
   - 偶数 = Host，奇数 = Container
   - 不相交的命名空间，Agent 可以仅通过 `seq` 引用任何消息

**阅读材料：**
- `docs/db.md`（数据库架构总览）
- `docs/db-central.md`（Central DB 详细 Schema）
- `docs/db-session.md`（Session DB 详细 Schema）
- `src/db/` 目录下的各实体文件

**实践任务：**
- 阅读 `src/db/migrations/` 目录，理解迁移机制
- 用 SQLite 客户端打开 `data/v2.db`（如果存在），查看表结构
- 阅读 `src/session-manager.ts`，理解 Session 创建和 DB 初始化流程

---

### Day 4 — 渠道系统：适配器、注册表与 Chat SDK Bridge

**目标：** 理解 NanoClaw 的渠道扩展机制，掌握如何添加新渠道。

**知识点：**

1. **渠道系统设计哲学**
   - 核心不内置任何渠道——每个渠道通过 Claude Code Skill 安装
   - 渠道自注册模式：模块加载时调用 `registerChannel()`
   - 缺少凭证的渠道发出 WARN 日志并被跳过

2. **ChannelAdapter 接口**
   ```typescript
   interface ChannelAdapter {
     name: string;
     channelType: string;
     setup(config: ChannelSetup): Promise<void>;
     teardown(): Promise<void>;
     isConnected(): boolean;
     deliver(platformId: string, threadId: string | null, message: OutboundMessage): Promise<void>;
     setTyping?(platformId: string, threadId: string | null): Promise<void>;
     syncConversations?(): Promise<ConversationInfo[]>;
     updateConversations?(conversations: ConversationConfig[]): void;
   }
   ```

3. **ChannelSetup 与回调**
   - `onInbound(platformId, threadId, message)` — 接收入站消息
   - `onMetadata(platformId, name, isGroup)` — 接收元数据更新
   - `conversations` — 从 Central DB 传入的对话配置

4. **Chat SDK Bridge**
   - 包装 Chat SDK Adapter + Chat 实例，适配 NanoClaw 的 ChannelAdapter 接口
   - 订阅模型：`onSubscribedMessage`（已订阅线程的所有消息）、`onNewMention`（新 @提及）、`onDirectMessage`（DM）
   - 平台能力差异：Slack（Block Kit 卡片、流式传输）、Discord（Embeds、按钮）、Telegram（内联键盘）、WhatsApp Cloud API（仅 DM、交互式回复按钮）

5. **原生渠道（无 Chat SDK）**
   - WhatsApp/Baileys 适配器：直接实现 ChannelAdapter 接口
   - 自行处理连接、消息接收、触发检查、出站发送

6. **渠道注册表（Registry）**
   - `src/channels/registry.ts` — 工厂注册表
   - `src/channels/index.ts` — Barrel 导入触发自注册
   - `src/channels/chat-sdk-bridge.ts` — Chat SDK Bridge 实现
   - `src/channels/cli.ts` — CLI 渠道

7. **添加新渠道的流程**
   - 在 `src/channels/` 添加 `<name>.ts`，实现 ChannelAdapter 接口
   - 在模块加载时调用 `registerChannel(name, factory)`
   - 凭证缺失时工厂返回 `null`
   - 在 `src/channels/index.ts` 添加导入行
   - 声明挂载需求（`mounts`、`env`）

8. **渠道隔离模型**
   - 三级隔离：Shared Session（多渠道共享会话）、Same Agent Separate Sessions（同 Agent 不同会话）、Separate Agent Groups（完全隔离）
   - 选择依据：信息是否可以跨渠道共享

**阅读材料：**
- `docs/SPEC.md`（Architecture: Channel System 章节）
- `docs/api-details.md`（Channel Adapter Interface、Chat SDK Bridge 章节）
- `docs/isolation-model.md`
- `src/channels/registry.ts`、`src/channels/index.ts`、`src/channels/chat-sdk-bridge.ts`

**实践任务：**
- 阅读 `src/channels/` 目录下所有文件，理解注册流程
- 选择一个已有渠道（如 CLI），追踪从消息接收到交付的完整路径
- 思考：如果要添加一个 Email 渠道，需要实现哪些接口？

---

### Day 5 — 消息流：入站、出站与路由

**目标：** 掌握消息从用户发送到 Agent 响应的完整生命周期。

**知识点：**

1. **入站消息流**
   ```
   用户发送消息 → 渠道适配器接收 → 提取 platformId / threadId →
   Host 查找 Central DB（messaging_group_agents）→ 确定 Agent Group + Session →
   Host 写入 inbound.db → Host 调用 wakeUpAgent(session) →
   Container 启动（或已运行）→ Agent Runner 轮询发现新消息 →
   Agent Runner 格式化消息 → 调用 Provider → 写入 outbound.db →
   Host 轮询 outbound.db → 查找对话 → 通过渠道适配器交付
   ```

2. **消息类型（kind）**
   - `chat` — 简单 NanoClaw 格式（sender、text、attachments）
   - `chat-sdk` — 完整 Chat SDK SerializedMessage（author、formatted、attachments、isMention）
   - `task` — 定时任务触发（prompt、script）
   - `webhook` — Webhook 载荷（source、event、payload）
   - `system` — 系统动作结果（action、status、result）

3. **消息格式化（Agent Runner 侧）**
   - `chat` → `<message sender="..." time="...">text</message>` XML 格式
   - `chat-sdk` → 提取字段，同样格式化为 XML
   - `task` → `[SCHEDULED TASK]` 前缀 + prompt
   - `webhook` → `[WEBHOOK: source/event]` + JSON
   - `system` → `[SYSTEM RESPONSE]` + 动作结果
   - 批量消息合并为 `<messages>` XML 块

4. **路由机制**
   - 默认行为：Agent Runner 从 messages_in 复制路由字段到 messages_out
   - 路由字段：platform_id、channel_type、thread_id
   - Agent 永远看不到路由字段——只看到内容
   - MCP 工具可覆盖路由（如 `send_to_agent`、`send_message` 指定目标）

5. **消息生命周期**
   ```
   pending → processing → completed
                       → failed（超过最大重试次数）
   ```
   - 停滞检测：`processing` 状态超过阈值 → Host 重置为 `pending`，指数退避重试
   - 退避策略：5s → 10s → 20s → 40s → failed

6. **Host 交付逻辑**
   - `kind === 'system'` → Host 内部处理
   - Agent-to-Agent → 写入目标 Session 的 inbound.db
   - 渠道交付 → 委托给适配器的 `deliver()` 方法
   - 操作分发：post / edit / reaction / ask_question

7. **交互式操作**
   - 卡片（Cards）：Agent 调用 `ask_user_question` → 写入 messages_out → Host 通过适配器交付 → 用户点击 → 回到 messages_in → Agent Runner 匹配并返回
   - 编辑消息：`edit_message` 工具 → messages_out with `operation: 'edit'`
   - 表情回应：`add_reaction` 工具 → messages_out with `operation: 'reaction'`
   - 审批路由：权限检查 → 选择审批人 → 发送审批卡片到审批人 DM

8. **触发词匹配**
   - 消息必须以触发模式开头（默认 `@Andy`）
   - 大小写不敏感
   - 非开头位置的触发词被忽略

**阅读材料：**
- `docs/SPEC.md`（Message Flow 章节）
- `docs/architecture.md`（Message Flow、Routing 章节）
- `docs/api-details.md`（Host Delivery Logic 章节）
- `src/delivery.ts`、`src/router.ts`
- `container/agent-runner/src/formatter.ts`

**实践任务：**
- 阅读 `src/delivery.ts`，追踪出站消息的交付逻辑
- 阅读 `container/agent-runner/src/formatter.ts`，理解各 kind 的格式化规则
- 画出完整的消息流转时序图（从用户发送到 Agent 响应）

---

### Day 6 — Agent Runner 核心：轮询循环与消息处理

**目标：** 深入理解 Agent Runner 的核心工作循环——如何轮询消息、调用 Provider、处理结果。

**知识点：**

1. **轮询循环（Poll Loop）**
   ```
   1. 查询 inbound.db 中 pending 且到期的消息
   2. 设置 status = 'processing'，status_changed = now()
   3. 批量格式化消息（剥离路由字段，按 kind 格式化）
   4. 调用 provider.query(prompt)
   5. 处理 Provider 事件 → 写入 outbound.db
   6. 设置已处理消息 status = 'completed'
   7. 回到步骤 1；无消息时短暂休眠后重新轮询
   ```

2. **并发轮询**
   - Provider 运行查询时，Agent Runner 继续以 ~500ms 间隔轮询 inbound.db
   - 新的 pending 消息通过 `provider.push()` 推入活跃查询
   - Claude 原生支持流式输入；Codex/OpenCode 通过 abort+restart 处理

3. **空闲行为**
   - 无消息且无活跃查询时，Agent Runner 休眠 1s 后重新轮询
   - Container 保持温暖直到 Host 杀死（空闲超时）
   - 空闲检测例外：`ask_user_question` 等待中、Agent 主动工作中

4. **消息格式化详解**
   - 路由字段剥离：platform_id、channel_type、thread_id 不包含在 prompt 中
   - 批量格式化：多消息合并为 `<messages>` XML 块
   - 混合类型处理：chat + system 结果用清晰分隔符合并
   - 命令检测：`/` 开头的消息检查命令列表

5. **状态管理**
   - Agent Runner 管理 messages_in 的 `status` 和 `status_changed`
   - 错误处理：Agent Runner 不设置 `failed`——留给 Host 侧的停滞检测
   - 处理确认：通过 outbound.db 的 `processing_ack` 表同步

6. **Session 恢复**
   - `sessionId` — 从 `ProviderEvent { type: 'init' }` 捕获
   - `resumeAt` — Claude 特有（最后 assistant 消息 UUID）
   - Container 重启时，Host 从 Central DB 的 sessions 表传入存储的 sessionId

7. **媒体处理**
   - 入站：Agent Runner 检测附件类型，Claude 原生支持图片/PDF/音频，其他保存到磁盘
   - 出站：通过 `send_file` MCP 工具，移动到 outbox 目录
   - 内容块构建：Claude 使用多部分 MessageParam，Codex/OpenCode 使用纯文本

8. **Pre-Agent 脚本（Tasks）**
   - task 类型消息有 `script` 字段时，先执行脚本
   - 脚本输出最后一行解析为 JSON：`{ wakeAgent: boolean, data?: unknown }`
   - `wakeAgent === false`：标记完成，不调用 Provider
   - `wakeAgent === true`：用脚本输出丰富 prompt，然后调用 Provider

**阅读材料：**
- `docs/agent-runner-details.md`（Poll Loop、Message Formatting、Status Management、Media Handling 章节）
- `container/agent-runner/src/poll-loop.ts`
- `container/agent-runner/src/formatter.ts`
- `container/agent-runner/src/db/messages-in.ts`、`messages-out.ts`

**实践任务：**
- 精读 `container/agent-runner/src/poll-loop.ts`，画出轮询循环状态机
- 阅读 `container/agent-runner/src/formatter.ts`，对比各 kind 的格式化差异
- 运行 `bun test`（在 container/agent-runner/ 目录下），理解测试用例

---

### Day 7 — MCP 工具系统

**目标：** 掌握 Agent Runner 的 MCP 工具体系——Agent 如何通过工具与外部世界交互。

**知识点：**

1. **MCP 工具架构**
   - Agent Runner 运行一个 MCP Server，暴露工具给 Agent
   - 所有工具通过 Session DB 通信——写入 messages_out 或查询 messages_in
   - MCP Server 接收 Session DB 路径通过环境变量，以只读方式打开第二个连接

2. **核心工具详解**

   | 工具 | 功能 | 实现方式 |
   |------|------|----------|
   | `send_message` | 发送聊天消息 | 写入 messages_out，kind: 'chat' |
   | `send_file` | 发送文件 | 移动文件到 outbox，写入 messages_out |
   | `send_card` | 发送结构化卡片 | 写入 messages_out，kind: 'chat-sdk' |
   | `ask_user_question` | 交互式提问（阻塞） | 写入 messages_out + 轮询 messages_in 等待响应 |
   | `edit_message` | 编辑已发送消息 | 写入 messages_out，operation: 'edit' |
   | `add_reaction` | 添加表情回应 | 写入 messages_out，operation: 'reaction' |
   | `send_to_agent` | 发送消息给其他 Agent | 写入 messages_out，channel_type: 'agent' |
   | `schedule_task` | 调度任务 | 写入 messages_in（给自己），设置 process_after + recurrence |
   | `list_tasks` | 列出任务 | 查询 messages_in |
   | `cancel_task` / `pause_task` / `resume_task` | 管理任务 | 修改 messages_in 行 |
   | `register_agent_group` | 注册新 Agent Group | 写入 messages_out，kind: 'system' |

3. **ask_user_question 的阻塞机制**
   - 生成 questionId
   - 写入 messages_out with `operation: 'ask_question'`
   - 轮询 messages_in 匹配 questionId
   - 超时返回错误
   - Agent 执行在此工具调用处暂停，Provider 的查询保持运行

4. **Agent-to-Agent 通信**
   - 发送方：`send_to_agent` → messages_out with `channel_type: 'agent'`
   - Host 验证权限 → 写入目标 Session 的 inbound.db
   - 接收方：作为普通 chat 消息到达，包含 sender 和 senderId

5. **系统动作（System Actions）**
   - Agent 通过 MCP 工具请求 Host 执行操作（注册 Group、重置 Session 等）
   - Agent Runner 写入 messages_out with `kind: 'system'`
   - Host 读取、验证权限、执行、将结果写回 messages_in with `kind: 'system'`
   - 这是程序化的结构化载荷，不是自然语言

6. **MCP 工具代码结构**
   - `container/agent-runner/src/mcp-tools/` 目录
   - `core.ts` — 核心工具（send_message、send_file 等）
   - `agents.ts` — Agent 管理工具
   - `interactive.ts` — 交互式工具（ask_user_question）
   - `scheduling.ts` — 调度工具
   - `self-mod.ts` — 自修改工具
   - `server.ts` — MCP Server 启动
   - `types.ts` — 类型定义
   - `index.ts` — Barrel 导出

7. **命令系统**
   - 白名单命令：传递给 Agent Provider 原生处理
   - 管理员命令（`/remote-control`、`/clear`、`/compact`）：需要 admin 身份
   - 过滤命令：静默丢弃
   - 命令验证在 Host 侧 `src/command-gate.ts` 完成

**阅读材料：**
- `docs/agent-runner-details.md`（MCP Tools 章节）
- `container/agent-runner/src/mcp-tools/` 目录下所有文件
- `src/command-gate.ts`

**实践任务：**
- 精读 `container/agent-runner/src/mcp-tools/core.ts`，理解 send_message 和 send_file 的实现
- 阅读 `container/agent-runner/src/mcp-tools/interactive.ts`，理解 ask_user_question 的阻塞等待机制
- 阅读 `container/agent-runner/src/mcp-tools/scheduling.ts`，理解任务调度如何通过 DB 实现

---

## 第二周：高级主题与实战应用

---

### Day 8 — Provider 系统：Claude、Codex 与 OpenCode

**目标：** 理解 Agent Runner 的 Provider 抽象层，掌握如何支持不同的 AI 后端。

**知识点：**

1. **AgentProvider 接口**
   ```typescript
   interface AgentProvider {
     query(input: QueryInput): AgentQuery;
   }
   ```
   - `QueryInput`：prompt、sessionId、resumeAt、cwd、mcpServers、systemPrompt、env
   - `AgentQuery`：push()、end()、abort()、events（AsyncIterable<ProviderEvent>）

2. **ProviderEvent 类型**
   - `init` — 会话建立/恢复，携带 sessionId
   - `result` — Agent 产出完整响应（可能多次）
   - `error` — 失败，retryable 标记 + classification（quota / auth / transport）
   - `progress` — 可选，用于日志

3. **Claude Provider**
   - 包装 `@anthropic-ai/claude-agent-sdk` 的 `query()`
   - 使用 `MessageStream` 实现异步可迭代输入
   - 支持 `resumeSessionAt` 精确恢复
   - PreCompact Hook：对话归档
   - PreToolUse Hook：Bash 环境变量清理
   - 完整工具白名单 + `permissionMode: 'bypassPermissions'`
   - 事件映射：system/init → init、result → result、api_retry → error(retryable)、rate_limit → error(quota)

4. **Codex Provider**
   - 包装 `@openai/codex-sdk`
   - 不支持流式输入——使用 abort+restart 模式处理 follow-up
   - `developer_instructions` 加载系统提示
   - 需要 `git init` 在工作区
   - `sandboxMode`、`approvalPolicy`、`networkAccessEnabled` 配置

5. **OpenCode Provider**
   - 包装 `@opencode-ai/sdk`
   - 运行本地 gRPC/HTTP 服务器
   - SSE 事件流输出
   - 不支持 resume（会话总是新的或通过 ID 复用）
   - 系统提示通过 `<system>` 前缀注入

6. **Provider 工厂**
   - `container/agent-runner/src/providers/factory.ts` — 根据 `AGENT_PROVIDER` 环境变量创建 Provider
   - `container/agent-runner/src/providers/provider-registry.ts` — Provider 注册表
   - 额外 Provider 通过 `/add-<provider>` Skill 安装

7. **接口边界**
   - Agent Runner 决定**发送什么**和**如何处理结果**
   - Provider 决定**如何与 SDK 通信**
   - 消息格式化、Hook、工具白名单、Session 持久化都是 Provider 内部实现

**阅读材料：**
- `docs/agent-runner-details.md`（AgentProvider Interface、Provider Implementations 章节）
- `container/agent-runner/src/providers/claude.ts`
- `container/agent-runner/src/providers/factory.ts`
- `container/agent-runner/src/providers/types.ts`

**实践任务：**
- 精读 `container/agent-runner/src/providers/claude.ts`，理解 Claude SDK 的集成方式
- 对比三种 Provider 的实现差异，总结接口边界
- 思考：如果要添加一个 OpenAI GPT Provider，需要实现哪些方法？

---

### Day 9 — Session 管理与 Container 生命周期

**目标：** 掌握 Session 的创建、恢复、销毁流程，以及 Container 的完整生命周期管理。

**知识点：**

1. **Session 概念**
   - Session = 一个文件夹 = 一个容器实例（运行时）
   - 文件夹路径：`data/v2-sessions/<agent_group_id>/<session_id>/`
   - 包含：inbound.db、outbound.db、.heartbeat、inbox/、outbox/、.claude/

2. **Session 创建（无竞态条件）**
   ```
   1. 消息到达 → Host 查找 Central DB 中匹配 group + thread 的 session
   2. 无 session → Host 原子创建 session 行 → 创建文件夹 → 创建 DB → 写入消息
   3. Container 启动前更多消息到达 → Host 找到已有 session → 写入同一个 inbound.db
   4. Container 启动 → 挂载文件夹 → Agent Runner 发现等待中的消息
   ```
   - Central DB session 行创建是序列化点

3. **Session 模式**
   - `shared` — 每个 messaging_group 一个 session
   - `per-thread` — 每个 platform thread 一个 session
   - `agent-shared` — 多个 messaging_group 共享一个 session

4. **Container 生命周期状态**
   ```
   stopped → running → idle → stopped
                 ↗
           idle → running（温暖期间新消息到达）
   ```
   - `stopped`：无容器，60s 扫描检查到期调度消息
   - `running`：活跃处理中，1s 轮询 outbound.db
   - `idle`：处理完毕，容器仍温暖（最长 30 分钟超时），1s 轮询
   - 空闲超时后 → Host 杀死容器 → stopped

5. **Container 启动流程**
   - `src/container-runner.ts` — 使用 `--entrypoint bash -c 'exec bun run /app/src/index.ts'`
   - 挂载：session 文件夹 → /workspace、agent group 文件夹 → /workspace/agent/
   - 环境变量：AGENT_PROVIDER、NANOCLAW_ADMIN_USER_ID、TZ、API Key
   - Agent Runner 读取配置 → 创建 Provider → 进入轮询循环

6. **Container 配置**
   - `container_config` 列（JSON）：additionalMounts、timeout
   - 额外挂载出现在 `/workspace/extra/{containerPath}`
   - 只读挂载使用 `--mount type=bind,source=...,target=...,readonly`

7. **空闲检测与心跳**
   - `.heartbeat` 文件的 mtime 是存活信号
   - Host 检查心跳判断 Container 是否存活
   - 空闲检测例外：ask_user_question 等待中、Agent 主动工作中

8. **Session 恢复与重置**
   - 恢复：Host 传入 sessionId，Provider 负责恢复上下文
   - 重置：创建新 session，标记旧消息为未处理，重新触发处理
   - Agent 可以通过 MCP 工具请求 session 重置

**阅读材料：**
- `docs/architecture.md`（Agent Groups vs Sessions、Container Lifecycle 章节）
- `src/session-manager.ts`
- `src/container-runner.ts`
- `container/Dockerfile`、`container/entrypoint.sh`

**实践任务：**
- 精读 `src/session-manager.ts`，理解 session 创建和路径管理
- 精读 `src/container-runner.ts`，理解容器启动和挂载配置
- 阅读 `container/Dockerfile`，理解镜像构建过程

---

### Day 10 — 调度系统与循环任务

**目标：** 掌握 NanoClaw 的调度机制——如何实现定时任务和循环任务。

**知识点：**

1. **调度设计哲学**
   - 没有独立的调度器——使用同一组消息表
   - One-shot：`process_after`（入站）或 `deliver_after`（出站），`recurrence = NULL`
   - Recurring：同上 + `recurrence` cron 表达式

2. **调度类型**
   - `cron` — Cron 表达式（如 `0 9 * * 1` = 每周一 9 点）
   - `interval` — 毫秒间隔（如 `3600000` = 每小时）
   - `once` — ISO 时间戳一次性执行

3. **Host 扫描机制**
   - 活跃容器（~1s）：轮询 Session DB 中的新 messages_out 行
   - 全量扫描（~60s）：扫描所有 Session DB 的到期 process_after / deliver_after
   - 完成/交付带 recurrence 的行后，插入下一次出现

4. **循环任务的实现**
   - Agent Runner 处理循环任务消息与普通消息相同
   - Agent Runner 标记 `completed` 后，**Host** 负责插入下一次出现
   - 下次时间从计划时间计算（非墙钟时间），防止漂移

5. **Pre-Agent 脚本**
   - task 类型消息的 `script` 字段
   - 执行流程：写入临时文件 → bash 执行（30s 超时）→ 解析最后一行 JSON
   - `wakeAgent: false` → 不唤醒 Agent，标记完成
   - `wakeAgent: true` → 用脚本输出丰富 prompt

6. **任务管理工具**
   - `schedule_task` — 创建任务（写入 messages_in 给自己）
   - `list_tasks` — 列出活跃任务
   - `pause_task` / `resume_task` — 暂停/恢复
   - `cancel_task` — 取消
   - `update_task` — 修改 prompt / recurrence / processAfter / script

7. **调度模块代码结构**
   - `src/modules/scheduling/` — Host 端调度模块
     - `actions.ts` — 调度动作
     - `db.ts` — 调度数据库操作
     - `recurrence.ts` — 循环规则计算
   - `container/agent-runner/src/mcp-tools/scheduling.ts` — Agent Runner 侧调度工具
   - `container/agent-runner/src/scheduling/task-script.ts` — Pre-Agent 脚本执行

**阅读材料：**
- `docs/SPEC.md`（Scheduled Tasks 章节）
- `docs/architecture.md`（Scheduling 章节）
- `src/modules/scheduling/` 目录
- `container/agent-runner/src/mcp-tools/scheduling.ts`

**实践任务：**
- 阅读 `src/modules/scheduling/recurrence.ts`，理解 cron 表达式的解析和下次时间计算
- 阅读 `container/agent-runner/src/scheduling/task-script.ts`，理解 pre-agent 脚本执行
- 阅读 `src/host-sweep.ts`，理解 Host 扫描如何触发到期任务

---

### Day 11 — 安全模型：隔离、权限与审批

**目标：** 理解 NanoClaw 的安全架构——容器隔离、权限系统、审批流程。

**知识点：**

1. **容器隔离**
   - 文件系统隔离：Agent 只能访问挂载的目录
   - 安全的 Bash 访问：命令在容器内运行，不在 Host 上
   - 网络隔离：可按容器配置
   - 进程隔离：容器进程不能影响 Host
   - 非 root 用户：容器以 `node` 用户（uid 1000）运行
   - `tini` 作为 init：回收 Chromium 僵尸进程，转发信号

2. **权限系统**
   - 角色：`owner`（全局唯一，首个配对用户）、`admin`（全局或限定 Agent Group 范围）
   - 成员关系：`agent_group_members` 表
   - 用户身份命名空间：`<channel_type>:<handle>`（如 `telegram:123456`）
   - 权限检查：`src/access.ts`、`src/modules/permissions/`

3. **审批流程**
   - 隐式审批：Agent 调用需要审批的工具 → Host 拦截 → 发送审批卡片给 admin → 等待响应 → 执行或拒绝
   - 显式审批：Agent 主动请求审批 → messages_out → 同样的流程
   - 审批路由：`pickApprover(agentGroupId)` → 范围内 admin → 全局 admin → owner
   - `pickApprovalDelivery` → 选择可达的审批人 → 同渠道类型优先
   - 审批卡片发送到审批人的 DM

4. **凭证存储与注入**
   - Claude CLI 认证：`data/sessions/{group}/.claude/` → 挂载到 `/home/node/.claude/`
   - WhatsApp Session：`store/auth/`
   - OneCLI 凭证代理：Host 通过 OneCLI 注入凭证到容器
   - `.env` 中只有认证变量被提取并写入 `data/env/env`，挂载到容器

5. **Prompt 注入防护**
   - 容器隔离限制爆炸半径
   - 只处理已注册 Group 的消息
   - 需要触发词（减少意外处理）
   - Agent 只能访问其 Group 的挂载目录
   - Claude 内置安全训练

6. **挂载安全**
   - `src/modules/mount-security/` — 挂载白名单验证
   - `config-examples/mount-allowlist.json` — 挂载白名单示例
   - 额外挂载需要 admin 审批

7. **发送者审批**
   - `src/modules/permissions/sender-approval.ts` — 未知发送者审批
   - `unknown_sender_policy`：`strict`（拒绝）/ `request_approval`（请求审批）/ `public`（允许）

8. **文件权限**
   - `groups/` 文件夹包含个人记忆，应设置 `chmod 700`

**阅读材料：**
- `docs/SPEC.md`（Security Considerations 章节）
- `docs/SECURITY.md`
- `src/access.ts`
- `src/modules/permissions/` 目录
- `src/modules/mount-security/` 目录

**实践任务：**
- 阅读 `src/access.ts`，理解权限检查逻辑
- 阅读 `src/modules/permissions/sender-approval.ts`，理解发送者审批流程
- 阅读 `src/modules/permissions/channel-approval.ts`，理解渠道审批流程

---

### Day 12 — Skills 系统与自定义扩展

**目标：** 掌握 NanoClaw 的 Skill 定制机制，理解如何通过 Skill 扩展项目功能。

**知识点：**

1. **Skills 设计哲学**
   - NanoClaw 通过 Skills（分支合并到用户安装中）进行定制
   - 不同的 Skill 添加不同的能力（渠道、集成、行为）
   - 代码结构必须确保不同定制不冲突

2. **Skill 冲突热点与解决方案**

   | 热点 | 冲突原因 | 解决方案 |
   |------|----------|----------|
   | `src/index.ts` | 每个 Skill 修补主循环 | 薄 index，逻辑在专用模块 |
   | `src/config.ts` | 每个 Skill 添加环境变量 | 配置在使用处声明 |
   | `src/container-runner.ts` | 渠道 Skill 添加挂载 | 声明式挂载注册 |
   | `src/db.ts` | Schema + 迁移 + CRUD 一体 | 按实体拆分 + 编号迁移 |
   | `agent-runner/src/index.ts` | 协议 + IPC + 格式化一体 | 拆分为 poll-loop、formatter、providers/、mcp-tools/ |
   | `src/ipc.ts` | 每个 MCP 工具修补 | mcp-tools/ 目录 + barrel |
   | `src/channels/index.ts` | 每个渠道添加导入行 | Barrel 文件 + 注释槽位 |

3. **注册模式优于 switch 语句**
   - 渠道、MCP 工具、Provider 使用注册/插件模式
   - Skill 添加一个文件 + 一个注册调用——不编辑中央 switch 语句

4. **添加新渠道的 Skill 流程**
   - 一个新文件（渠道适配器或 Chat SDK 配置）
   - `src/channels/index.ts` 中一行导入
   - 零修改路由、格式化、交付、容器代码

5. **挂载注册模式**
   ```typescript
   registerChannel('gmail', {
     factory: createGmailAdapter,
     mounts: [{ hostPath: '~/.gmail-mcp', containerPath: '/home/node/.gmail-mcp', readonly: false }],
     env: ['GMAIL_OAUTH_TOKEN'],
   });
   ```
   - Container Runner 从注册表读取挂载——不需要编辑 container-runner.ts

6. **配置模式**
   - Skill 不修补 `config.ts` 或 `.env.example`
   - Skill 特定的环境变量在 SKILL.md 中文档化
   - 每个模块直接读取自己的环境变量
   - 共享配置（DATA_DIR、TIMEZONE 等）留在 config.ts

7. **DB 文件结构**
   - 按实体拆分，不按层拆分
   - 每个 Skill 添加编号迁移文件
   - 迁移是只追加的编号文件

8. **Skill 目录结构**
   - `.claude/skills/add-<name>/SKILL.md` — Skill 定义
   - `.claude/skills/add-<name>/` — Skill 脚本和资源
   - 容器内 Skill：`container/skills/<name>/SKILL.md`

9. **代码风格**
   - 行宽 120 字符
   - 简洁日志：薄包装保持每个日志调用在一行
   - 无内联 ALTER TABLE——使用迁移运行器

**阅读材料：**
- `docs/architecture.md`（Flexibility Model、Code Structure for Skill Customization 章节）
- `docs/skills-as-branches.md`
- `.claude/skills/` 目录下几个典型 Skill（如 `add-telegram/SKILL.md`、`add-slack/SKILL.md`）
- `container/skills/` 目录

**实践任务：**
- 阅读一个完整的渠道 Skill（如 `add-telegram`），理解 Skill 的完整结构
- 阅读 `container/skills/welcome/SKILL.md`，理解容器内 Skill
- 思考：如果要创建一个自定义 Skill，需要哪些文件和修改？

---

### Day 13 — Setup 流程与部署

**目标：** 理解 NanoClaw 的安装配置流程和部署方案。

**知识点：**

1. **Setup 入口**
   - `nanoclaw.sh` — 顶层包装器：Phase 1（bootstrap）+ Phase 2（setup:auto）
   - `setup.sh` — Phase 1 bootstrap：Node、pnpm、原生模块验证
   - `setup/auto.ts` — Phase 2 驱动：clack UI、步骤执行、用户提示

2. **三级输出契约**
   - Level 1（用户面）：clack 渲染，简洁品牌化
   - Level 2（进展日志）：`logs/setup.log`，结构化步骤块
   - Level 3（原始日志）：`logs/setup-steps/NN-step-name.log`，完整子进程输出

3. **Setup 步骤**
   - Bootstrap：安装 Node.js、pnpm、验证原生模块
   - Environment：检测 Docker / Apple Container 运行时
   - Container：构建容器镜像
   - Auth：Anthropic 凭证注册（交互式例外）
   - Channel：选择并配置渠道（Telegram / Slack / Discord / WhatsApp 等）
   - Service：安装 launchd 服务（macOS）

4. **新步骤的契约**
   - 接收 raw-log 路径，所有 stdout + stderr 写入该文件
   - 结束时发出终端状态块：`STATUS: success|skipped|failed`
   - 长时间运行的步骤可发出子状态块
   - 硬失败时非零退出

5. **Anthropic 例外**
   - `claude setup-token` 打开浏览器、运行 OAuth 提示
   - 唯一允许的中断视觉流的地方
   - clack 流程显式暂停，子进程继承 stdio

6. **部署方案**
   - macOS：launchd 服务（`launchd/com.nanoclaw.plist`）
   - Linux：systemd 或直接运行
   - 环境变量配置：ASSISTANT_NAME、CONTAINER_TIMEOUT、MAX_CONCURRENT_CONTAINERS

7. **CI 流程**
   - `.github/workflows/ci.yml`
   - 安装 Node + Bun
   - `pnpm install --frozen-lockfile`（Host）
   - `bun install --frozen-lockfile`（Container）
   - 格式检查、类型检查（Host + Container）、测试（Host + Container）

8. **Setup 代码结构**
   - `setup/index.ts` — 入口
   - `setup/auto.ts` — 自动化驱动
   - `setup/environment.ts` — 环境检测
   - `setup/container.ts` — 容器构建
   - `setup/service.ts` — 服务安装
   - `setup/channels/` — 渠道安装脚本
   - `setup/lib/` — 共享工具函数

**阅读材料：**
- `docs/setup-flow.md`
- `docs/setup-wiring.md`
- `nanoclaw.sh`
- `setup/auto.ts`
- `.github/workflows/ci.yml`

**实践任务：**
- 阅读 `nanoclaw.sh`，理解两阶段安装流程
- 阅读 `setup/auto.ts`，理解步骤编排逻辑
- 阅读 `.github/workflows/ci.yml`，理解 CI 流程

---

### Day 14 — 综合复习与实战演练

**目标：** 整合两周所学知识，通过实战项目巩固理解，形成完整的知识体系。

**知识点：**

1. **架构全景回顾**
   - 双进程模型：Host（Node.js）+ Container（Bun）
   - 三级数据库：Central DB + Session Inbound DB + Session Outbound DB
   - 消息驱动：一切都是消息，DB 是唯一 IO 边界
   - 渠道自注册：工厂模式 + Barrel 导入
   - Provider 抽象：统一接口，多后端支持
   - Skill 定制：分支合并，最小冲突

2. **核心数据流回顾**
   ```
   用户 → 渠道适配器 → Host 路由 → inbound.db →
   Agent Runner 轮询 → 格式化 → Provider →
   outbound.db → Host 交付 → 渠道适配器 → 用户
   ```

3. **关键设计决策回顾**
   - 为什么用 SQLite 而不是消息队列：简单、可靠、无外部依赖
   - 为什么用轮询而不是事件驱动：跨挂载边界的事件不可靠
   - 为什么 Host 和 Container 用不同运行时：Baileys 原生绑定 vs Bun 性能
   - 为什么用 DELETE 模式而不是 WAL：跨挂载 WAL 可见性问题
   - 为什么单写者原则：消除跨挂载写冲突

4. **实战演练：追踪一个完整请求**
   - 从用户在 Telegram 发送 `@Andy 今天天气如何？` 开始
   - 追踪消息经过的每一个模块和函数
   - 记录每一步的数据格式变化
   - 理解每个设计决策的影响

5. **实战演练：添加一个新功能**
   - 设计一个简单的 MCP 工具（如 `get_current_time`）
   - 在 `container/agent-runner/src/mcp-tools/` 添加工具文件
   - 更新 barrel 导出
   - 编写测试

6. **知识体系自检清单**
   - [ ] 能画出完整的架构图
   - [ ] 能解释消息从入站到出站的完整流程
   - [ ] 能说明 Central DB 和 Session DB 的职责划分
   - [ ] 能描述渠道自注册的实现机制
   - [ ] 能解释 Agent Runner 的轮询循环逻辑
   - [ ] 能列出所有 MCP 工具及其功能
   - [ ] 能对比三种 Provider 的实现差异
   - [ ] 能说明 Container 的生命周期状态转换
   - [ ] 能解释调度系统的实现原理
   - [ ] 能描述安全模型的各个层面
   - [ ] 能说明 Skill 系统的冲突避免策略
   - [ ] 能追踪 Setup 流程的完整步骤

7. **进阶学习方向**
   - 深入 Chat SDK 适配器开发
   - 自定义 Provider 实现
   - 复杂 Agent 编排（如 PR Factory 模式）
   - 性能优化与监控
   - 多租户与权限扩展

**阅读材料：**
- 回顾所有 `docs/` 下的文档
- `docs/architecture.md`（PR Factory 示例章节）
- `docs/SDK_DEEP_DIVE.md`

**实践任务：**
- 完成知识体系自检清单
- 实现一个简单的 MCP 工具并测试
- 撰写个人学习笔记，记录关键理解

---

## 附录：关键文件速查表

| 类别 | 文件 | 用途 |
|------|------|------|
| 入口 | `src/index.ts` | Host 主进程入口 |
| 入口 | `container/agent-runner/src/index.ts` | Agent Runner 入口 |
| 配置 | `src/config.ts` | Host 配置常量 |
| 类型 | `src/types.ts` | 核心类型定义 |
| 路由 | `src/router.ts` | 消息路由 |
| 交付 | `src/delivery.ts` | 出站消息交付 |
| 容器 | `src/container-runner.ts` | 容器生命周期管理 |
| 会话 | `src/session-manager.ts` | Session 管理 |
| 扫描 | `src/host-sweep.ts` | Host 定期扫描 |
| 渠道 | `src/channels/registry.ts` | 渠道注册表 |
| 渠道 | `src/channels/chat-sdk-bridge.ts` | Chat SDK Bridge |
| 数据库 | `src/db/` | Central DB 访问层 |
| 权限 | `src/access.ts` | 权限检查 |
| 命令 | `src/command-gate.ts` | 命令门控 |
| 轮询 | `container/agent-runner/src/poll-loop.ts` | Agent 轮询循环 |
| 格式化 | `container/agent-runner/src/formatter.ts` | 消息格式化 |
| 工具 | `container/agent-runner/src/mcp-tools/` | MCP 工具集 |
| Provider | `container/agent-runner/src/providers/` | AI Provider 实现 |
| Session DB | `container/agent-runner/src/db/` | Session DB 访问层 |
| 调度 | `src/modules/scheduling/` | 调度模块 |
| 权限 | `src/modules/permissions/` | 权限模块 |
| 审批 | `src/modules/approvals/` | 审批模块 |
| 交互 | `src/modules/interactive/` | 交互模块 |
| 自修改 | `src/modules/self-mod/` | 自修改模块 |
| CLI | `src/cli/` | 命令行接口 |
| Setup | `setup/` | 安装配置 |
| 文档 | `docs/` | 架构与设计文档 |
| 技能 | `.claude/skills/` | Claude Code Skills |
| 容器技能 | `container/skills/` | 容器内 Skills |
| 构建 | `container/Dockerfile` | 容器镜像构建 |
| CI | `.github/workflows/ci.yml` | CI 流程 |

---

## 学习建议

1. **先读文档，再读代码**：`docs/` 下的文档质量很高，先建立概念模型再深入源码
2. **画图辅助理解**：架构图、时序图、状态机图能极大帮助理解复杂系统
3. **从测试入手**：测试用例是最好的代码文档，理解预期行为后再看实现
4. **追踪一个完整请求**：从用户发送消息到 Agent 响应，追踪每一步
5. **动手实践**：尝试添加一个简单的 MCP 工具或渠道，巩固理解
6. **关注设计决策**：理解"为什么"比理解"怎么做"更重要
