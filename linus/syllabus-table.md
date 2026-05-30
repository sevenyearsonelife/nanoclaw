# NanoClaw 两周教学大纲（表格版）

> 项目定位：NanoClaw 是一个多渠道个人 Claude 助手，支持持久化记忆、定时任务和容器隔离的 Agent 执行环境。
> 技术栈：Host 端 Node.js 20+ / pnpm，Container 端 Bun，SQLite（better-sqlite3 / bun:sqlite），Claude Agent SDK，Docker / Apple Container。

---

## 第一周：基础架构与核心机制

### Day 1 — 项目全景与环境搭建

| 维度 | 内容 |
|------|------|
| **目标** | 建立 NanoClaw 的全局认知，搭建开发环境，理解项目定位与核心设计哲学 |
| **知识点 1** | **项目定位与核心价值** — 多渠道个人 AI 助手（WhatsApp / Telegram / Slack / Discord / Gmail）；核心能力：多渠道接入、持久化记忆、定时任务、容器隔离执行；与普通 ChatBot 的区别：每个 Agent Session 有独立容器，通过 SQLite DB 作为唯一 IO 机制 |
| **知识点 2** | **技术栈概览** — Host 端：Node.js 20+、pnpm 10、better-sqlite3、@clack/prompts；Container 端：Bun 1.3+、bun:sqlite、@anthropic-ai/claude-agent-sdk；为什么 Host 和 Container 使用不同运行时（参考 `docs/build-and-runtime.md`）；容器运行时：Docker Desktop / Docker Engine / Apple Container |
| **知识点 3** | **项目目录结构** — `src/` Host 端核心代码；`container/agent-runner/` Container 内 Agent Runner；`container/skills/` 容器内置技能；`docs/` 架构文档；`setup/` 安装与配置脚本；`groups/` Agent Group 工作区；`.claude/skills/` Claude Code 技能系统；`bin/ncl` CLI 入口 |
| **知识点 4** | **开发环境搭建** — `pnpm install` 安装依赖；`pnpm run build` 编译 TypeScript；`pnpm run dev` 开发模式启动；`pnpm run test` 运行测试（vitest）；`cd container/agent-runner && bun install` 安装容器端依赖；`bun test` 运行容器端测试 |
| **知识点 5** | **关键配置文件** — `package.json` 项目依赖与脚本；`tsconfig.json` TypeScript 配置；`vitest.config.ts` 测试配置（排除 container/agent-runner/）；`.mcp.json` MCP 服务器配置；`.env` 环境变量（ANTHROPIC_API_KEY / CLAUDE_CODE_OAUTH_TOKEN） |
| **阅读材料** | `README.md`、`docs/SPEC.md`（Architecture 章节）、`docs/build-and-runtime.md` |
| **实践任务** | 克隆项目，完成环境搭建，确保 `pnpm run typecheck` 和 `pnpm run test` 通过；画出项目目录结构思维导图 |

---

### Day 2 — 架构深潜：Host 与 Container 的双进程模型

| 维度 | 内容 |
|------|------|
| **目标** | 深入理解 NanoClaw 的核心架构——Host 进程与 Container 进程的职责划分与通信机制 |
| **知识点 1** | **双进程架构** — Host（macOS / Linux）：主 Node.js 进程，负责渠道接入、消息路由、容器编排、调度；Container（Linux VM）：隔离环境中的 Agent Runner，负责轮询消息、调用 AI Provider、执行 MCP 工具；两者通过挂载的 SQLite DB 通信，无 IPC 文件、无 stdin 管道 |
| **知识点 2** | **核心设计哲学：一切都是消息** — `messages_in`（Host → Agent Runner）：chat、task、webhook、system 等类型；`messages_out`（Agent Runner → Host）：chat、chat-sdk、system 等类型；路由字段在 Agent Runner 侧被剥离，Agent 只看到内容 |
| **知识点 3** | **Host 端核心模块** — `src/index.ts` 编排器；`src/delivery.ts` 轮询 outbound.db 并交付消息；`src/host-sweep.ts` 60 秒扫描；`src/session-manager.ts` Session 解析；`src/container-runner.ts` 容器启动与凭证注入；`src/router.ts` 消息格式化与出站路由 |
| **知识点 4** | **Container 端核心模块** — `container/agent-runner/src/index.ts` 入口；`poll-loop.ts` 轮询循环；`formatter.ts` 消息格式化；`db/` Session DB 访问层（messages-in.ts、messages-out.ts、session-state.ts） |
| **知识点 5** | **容器挂载结构** — `/workspace/` 挂载 session 文件夹（读写）；`.claude/` Claude SDK session 数据；`inbound.db` / `outbound.db` 双向消息；`.heartbeat` mtime 触碰存活信号；`inbox/` / `outbox/` 附件目录；`agent/` 挂载 Agent Group 文件夹（CLAUDE.md、skills） |
| **阅读材料** | `docs/architecture.md`（Core Idea、Two-Level DB、Container Lifecycle 章节）、`docs/architecture.zh-CN.md`、`src/index.ts`、`container/agent-runner/src/index.ts` |
| **实践任务** | 阅读 `src/index.ts` 梳理 Host 启动流程；阅读 `poll-loop.ts` 理解轮询循环逻辑；画出 Host ↔ Container 通信时序图 |

---

### Day 3 — 数据库架构：Central DB 与 Session DB

| 维度 | 内容 |
|------|------|
| **目标** | 掌握 NanoClaw 的三级数据库设计，理解数据如何在 Central DB 和 Session DB 之间流转 |
| **知识点 1** | **三级数据库概览** — Central DB（`data/v2.db`）：身份、权限、路由、配线——管理平面；Session Inbound DB（`inbound.db`）：Host 写入，Container 只读；Session Outbound DB（`outbound.db`）：Container 写入，Host 轮询 |
| **知识点 2** | **单写者原则** — 每个 SQLite 文件只有一个写者；Host 写 Central DB 和所有 inbound.db；Container 只写自己的 outbound.db；消除跨 Docker/Apple Container 挂载边界的写冲突 |
| **知识点 3** | **Central DB 核心表** — `agent_groups` Agent 工作区；`messaging_groups` 平台群组/频道；`messaging_group_agents` 配线关系；`users` 用户身份；`user_roles` 角色（owner / admin）；`agent_group_members` 成员关系；`user_dms` DM 解析缓存；`sessions` 会话注册表；`pending_questions` 待处理交互式问题；`pending_approvals` 待审批项；`dropped_messages` 丢弃消息审计 |
| **知识点 4** | **Session DB 核心表** — `messages_in`（inbound.db）：id、kind、status、process_after、recurrence、content；`messages_out`（outbound.db）：id、in_reply_to、kind、content、delivered；`processing_ack` Container 处理状态确认；`session_state` Container 启动时恢复的状态；`agent_destinations` / `session_routing` 投影自 Central DB |
| **知识点 5** | **跨挂载可见性规则** — Session DB 使用 `journal_mode = DELETE`（不是 WAL）；Host 端写入 inbound.db 采用"打开-写入-关闭"模式；Container 以只读模式打开 inbound.db；心跳通过文件 `.heartbeat` 的 mtime 触碰 |
| **知识点 6** | **数据库迁移系统** — Central DB：编号迁移文件（`src/db/migrations/001-initial.ts`...）；Session DB：`IF NOT EXISTS` + 临时 `ALTER TABLE` 辅助函数；迁移运行器通过 `schema_version` 表追踪版本 |
| **知识点 7** | **Seq 编号不变量** — 偶数 = Host，奇数 = Container；不相交的命名空间，Agent 可以仅通过 `seq` 引用任何消息 |
| **阅读材料** | `docs/db.md`、`docs/db-central.md`、`docs/db-session.md`、`src/db/` 目录下各实体文件 |
| **实践任务** | 阅读迁移机制；用 SQLite 客户端查看表结构；阅读 `src/session-manager.ts` 理解 Session 创建和 DB 初始化流程 |

---

### Day 4 — 渠道系统：适配器、注册表与 Chat SDK Bridge

| 维度 | 内容 |
|------|------|
| **目标** | 理解 NanoClaw 的渠道扩展机制，掌握如何添加新渠道 |
| **知识点 1** | **渠道系统设计哲学** — 核心不内置任何渠道——每个渠道通过 Claude Code Skill 安装；渠道自注册模式：模块加载时调用 `registerChannel()`；缺少凭证的渠道发出 WARN 日志并被跳过 |
| **知识点 2** | **ChannelAdapter 接口** — `name` / `channelType` 标识；`setup(config)` 初始化；`teardown()` 清理；`isConnected()` 状态；`deliver(platformId, threadId, message)` 出站交付；`setTyping()` / `syncConversations()` / `updateConversations()` 可选方法 |
| **知识点 3** | **Chat SDK Bridge** — 包装 Chat SDK Adapter + Chat 实例，适配 NanoClaw 的 ChannelAdapter 接口；订阅模型：`onSubscribedMessage`（已订阅线程）、`onNewMention`（新 @提及）、`onDirectMessage`（DM）；平台能力差异：Slack（Block Kit）、Discord（Embeds）、Telegram（内联键盘）、WhatsApp Cloud API（仅 DM） |
| **知识点 4** | **原生渠道（无 Chat SDK）** — WhatsApp/Baileys 适配器：直接实现 ChannelAdapter 接口；自行处理连接、消息接收、触发检查、出站发送 |
| **知识点 5** | **渠道注册表（Registry）** — `src/channels/registry.ts` 工厂注册表；`src/channels/index.ts` Barrel 导入触发自注册；`src/channels/chat-sdk-bridge.ts` Bridge 实现；`src/channels/cli.ts` CLI 渠道 |
| **知识点 6** | **添加新渠道的流程** — 在 `src/channels/` 添加 `<name>.ts` 实现 ChannelAdapter 接口；模块加载时调用 `registerChannel(name, factory)`；凭证缺失时工厂返回 `null`；在 `index.ts` 添加导入行；声明挂载需求 |
| **知识点 7** | **渠道隔离模型** — 三级隔离：Shared Session（多渠道共享会话）、Same Agent Separate Sessions（同 Agent 不同会话）、Separate Agent Groups（完全隔离）；选择依据：信息是否可以跨渠道共享 |
| **阅读材料** | `docs/SPEC.md`（Channel System 章节）、`docs/api-details.md`（Channel Adapter Interface、Chat SDK Bridge 章节）、`docs/isolation-model.md`、`src/channels/` 目录 |
| **实践任务** | 阅读注册流程；追踪一个渠道从消息接收到交付的完整路径；思考添加 Email 渠道需要实现哪些接口 |

---

### Day 5 — 消息流：入站、出站与路由

| 维度 | 内容 |
|------|------|
| **目标** | 掌握消息从用户发送到 Agent 响应的完整生命周期 |
| **知识点 1** | **入站消息流** — 用户发送 → 渠道适配器接收 → 提取 platformId / threadId → Host 查找 Central DB → 确定 Agent Group + Session → 写入 inbound.db → wakeUpAgent → Container 启动 → Agent Runner 轮询 → 格式化 → 调用 Provider → 写入 outbound.db → Host 轮询 → 渠道适配器交付 |
| **知识点 2** | **消息类型（kind）** — `chat` 简单格式（sender、text、attachments）；`chat-sdk` 完整 SerializedMessage；`task` 定时任务触发（prompt、script）；`webhook` Webhook 载荷；`system` 系统动作结果 |
| **知识点 3** | **消息格式化** — `chat` → XML `<message>` 格式；`chat-sdk` → 提取字段同样 XML；`task` → `[SCHEDULED TASK]` 前缀；`webhook` → `[WEBHOOK: source/event]` + JSON；`system` → `[SYSTEM RESPONSE]`；批量合并为 `<messages>` XML 块 |
| **知识点 4** | **路由机制** — 默认：Agent Runner 从 messages_in 复制路由字段到 messages_out；路由字段：platform_id、channel_type、thread_id；Agent 永远看不到路由字段；MCP 工具可覆盖路由 |
| **知识点 5** | **消息生命周期** — `pending → processing → completed → failed`；停滞检测：processing 超阈值 → 重置为 pending + 指数退避（5s → 10s → 20s → 40s → failed） |
| **知识点 6** | **Host 交付逻辑** — `kind === 'system'` → Host 内部处理；Agent-to-Agent → 写入目标 Session inbound.db；渠道交付 → 委托适配器 `deliver()`；操作分发：post / edit / reaction / ask_question |
| **知识点 7** | **交互式操作** — 卡片：ask_user_question → messages_out → 适配器交付 → 用户点击 → messages_in → Agent Runner 匹配返回；编辑：edit_message → operation: 'edit'；表情：add_reaction → operation: 'reaction'；审批路由：权限检查 → 选择审批人 → DM 审批卡片 |
| **知识点 8** | **触发词匹配** — 必须以触发模式开头（默认 `@Andy`）；大小写不敏感；非开头位置忽略 |
| **阅读材料** | `docs/SPEC.md`（Message Flow 章节）、`docs/architecture.md`（Message Flow、Routing 章节）、`docs/api-details.md`（Host Delivery Logic 章节）、`src/delivery.ts`、`src/router.ts`、`container/agent-runner/src/formatter.ts` |
| **实践任务** | 阅读 `src/delivery.ts` 追踪出站交付逻辑；阅读 `formatter.ts` 理解格式化规则；画出完整消息流转时序图 |

---

### Day 6 — Agent Runner 核心：轮询循环与消息处理

| 维度 | 内容 |
|------|------|
| **目标** | 深入理解 Agent Runner 的核心工作循环——如何轮询消息、调用 Provider、处理结果 |
| **知识点 1** | **轮询循环** — 1. 查询 inbound.db pending 且到期的消息；2. 设置 status = 'processing'；3. 批量格式化消息；4. 调用 provider.query(prompt)；5. 处理 Provider 事件 → 写入 outbound.db；6. 设置 status = 'completed'；7. 回到步骤 1 |
| **知识点 2** | **并发轮询** — Provider 运行查询时，Agent Runner 继续以 ~500ms 间隔轮询 inbound.db；新消息通过 `provider.push()` 推入活跃查询；Claude 原生支持流式输入；Codex/OpenCode 通过 abort+restart 处理 |
| **知识点 3** | **空闲行为** — 无消息且无活跃查询时休眠 1s 后重新轮询；Container 保持温暖直到 Host 杀死（空闲超时）；空闲检测例外：ask_user_question 等待中、Agent 主动工作中 |
| **知识点 4** | **消息格式化详解** — 路由字段剥离；批量格式化为 `<messages>` XML 块；混合类型用清晰分隔符合并；命令检测：`/` 开头检查命令列表 |
| **知识点 5** | **状态管理** — Agent Runner 管理 messages_in 的 status 和 status_changed；错误处理：不设置 failed——留给 Host 侧停滞检测；处理确认：通过 outbound.db 的 processing_ack 表同步 |
| **知识点 6** | **Session 恢复** — `sessionId` 从 ProviderEvent init 捕获；`resumeAt` Claude 特有（最后 assistant 消息 UUID）；Container 重启时 Host 从 Central DB 传入存储的 sessionId |
| **知识点 7** | **媒体处理** — 入站：检测附件类型，Claude 原生支持图片/PDF/音频，其他保存到磁盘；出站：通过 send_file MCP 工具移动到 outbox；内容块构建：Claude 多部分 MessageParam，Codex/OpenCode 纯文本 |
| **知识点 8** | **Pre-Agent 脚本** — task 类型有 script 字段时先执行脚本；输出最后一行解析为 JSON `{ wakeAgent, data }`；`wakeAgent === false` 标记完成不调用 Provider；`wakeAgent === true` 用脚本输出丰富 prompt |
| **阅读材料** | `docs/agent-runner-details.md`（Poll Loop、Message Formatting、Status Management、Media Handling 章节）、`poll-loop.ts`、`formatter.ts`、`db/messages-in.ts`、`db/messages-out.ts` |
| **实践任务** | 精读 `poll-loop.ts` 画出轮询循环状态机；阅读 `formatter.ts` 对比各 kind 格式化差异；运行 `bun test` 理解测试用例 |

---

### Day 7 — MCP 工具系统

| 维度 | 内容 |
|------|------|
| **目标** | 掌握 Agent Runner 的 MCP 工具体系——Agent 如何通过工具与外部世界交互 |
| **知识点 1** | **MCP 工具架构** — Agent Runner 运行 MCP Server 暴露工具给 Agent；所有工具通过 Session DB 通信；MCP Server 接收 Session DB 路径通过环境变量，以只读方式打开第二个连接 |
| **知识点 2** | **核心工具详解** — 见下表 |

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

| 维度 | 内容 |
|------|------|
| **知识点 3** | **ask_user_question 的阻塞机制** — 生成 questionId → 写入 messages_out with operation: 'ask_question' → 轮询 messages_in 匹配 questionId → 超时返回错误 → Agent 执行在此工具调用处暂停 |
| **知识点 4** | **Agent-to-Agent 通信** — 发送方：send_to_agent → messages_out with channel_type: 'agent'；Host 验证权限 → 写入目标 Session inbound.db；接收方：作为普通 chat 消息到达 |
| **知识点 5** | **系统动作** — Agent 通过 MCP 工具请求 Host 执行操作；Agent Runner 写入 messages_out with kind: 'system'；Host 读取、验证权限、执行、结果写回 messages_in with kind: 'system'；程序化结构化载荷，非自然语言 |
| **知识点 6** | **MCP 工具代码结构** — `core.ts` 核心工具；`agents.ts` Agent 管理工具；`interactive.ts` 交互式工具；`scheduling.ts` 调度工具；`self-mod.ts` 自修改工具；`server.ts` MCP Server 启动；`types.ts` 类型定义；`index.ts` Barrel 导出 |
| **知识点 7** | **命令系统** — 白名单命令：传递给 Agent Provider 原生处理；管理员命令（/remote-control、/clear、/compact）：需要 admin 身份；过滤命令：静默丢弃；命令验证在 Host 侧 `src/command-gate.ts` 完成 |
| **阅读材料** | `docs/agent-runner-details.md`（MCP Tools 章节）、`container/agent-runner/src/mcp-tools/` 目录、`src/command-gate.ts` |
| **实践任务** | 精读 `core.ts` 理解 send_message 和 send_file 实现；阅读 `interactive.ts` 理解 ask_user_question 阻塞等待机制；阅读 `scheduling.ts` 理解任务调度 DB 实现 |

---

## 第二周：高级主题与实战应用

### Day 8 — Provider 系统：Claude、Codex 与 OpenCode

| 维度 | 内容 |
|------|------|
| **目标** | 理解 Agent Runner 的 Provider 抽象层，掌握如何支持不同的 AI 后端 |
| **知识点 1** | **AgentProvider 接口** — `query(input: QueryInput): AgentQuery`；QueryInput：prompt、sessionId、resumeAt、cwd、mcpServers、systemPrompt、env；AgentQuery：push()、end()、abort()、events（AsyncIterable<ProviderEvent>） |
| **知识点 2** | **ProviderEvent 类型** — `init` 会话建立/恢复，携带 sessionId；`result` Agent 产出完整响应（可能多次）；`error` 失败，retryable 标记 + classification（quota / auth / transport）；`progress` 可选，用于日志 |
| **知识点 3** | **Claude Provider** — 包装 `@anthropic-ai/claude-agent-sdk` 的 `query()`；MessageStream 异步可迭代输入；resumeSessionAt 精确恢复；PreCompact Hook 对话归档；PreToolUse Hook Bash 环境变量清理；完整工具白名单 + bypassPermissions |
| **知识点 4** | **Codex Provider** — 包装 `@openai/codex-sdk`；不支持流式输入——abort+restart 模式处理 follow-up；developer_instructions 加载系统提示；需要 git init 在工作区；sandboxMode / approvalPolicy / networkAccessEnabled 配置 |
| **知识点 5** | **OpenCode Provider** — 包装 `@opencode-ai/sdk`；运行本地 gRPC/HTTP 服务器；SSE 事件流输出；不支持 resume；系统提示通过 `<system>` 前缀注入 |
| **知识点 6** | **Provider 工厂** — `factory.ts` 根据 AGENT_PROVIDER 环境变量创建 Provider；`provider-registry.ts` Provider 注册表；额外 Provider 通过 `/add-<provider>` Skill 安装 |
| **知识点 7** | **接口边界** — Agent Runner 决定**发送什么**和**如何处理结果**；Provider 决定**如何与 SDK 通信**；消息格式化、Hook、工具白名单、Session 持久化都是 Provider 内部实现 |
| **阅读材料** | `docs/agent-runner-details.md`（AgentProvider Interface、Provider Implementations 章节）、`providers/claude.ts`、`providers/factory.ts`、`providers/types.ts` |
| **实践任务** | 精读 `claude.ts` 理解 Claude SDK 集成方式；对比三种 Provider 实现差异，总结接口边界；思考添加 OpenAI GPT Provider 需要实现哪些方法 |

---

### Day 9 — Session 管理与 Container 生命周期

| 维度 | 内容 |
|------|------|
| **目标** | 掌握 Session 的创建、恢复、销毁流程，以及 Container 的完整生命周期管理 |
| **知识点 1** | **Session 概念** — Session = 一个文件夹 = 一个容器实例（运行时）；路径：`data/v2-sessions/<agent_group_id>/<session_id>/`；包含：inbound.db、outbound.db、.heartbeat、inbox/、outbox/、.claude/ |
| **知识点 2** | **Session 创建（无竞态条件）** — 消息到达 → Host 查找 Central DB → 无 session 则原子创建 session 行 → 创建文件夹和 DB → 写入消息 → Container 启动前更多消息到达 → 写入同一个 inbound.db → Container 启动发现等待消息；Central DB session 行创建是序列化点 |
| **知识点 3** | **Session 模式** — `shared` 每个 messaging_group 一个 session；`per-thread` 每个 platform thread 一个 session；`agent-shared` 多个 messaging_group 共享一个 session |
| **知识点 4** | **Container 生命周期状态** — `stopped`（无容器，60s 扫描）→ `running`（活跃处理，1s 轮询）→ `idle`（处理完毕容器温暖，最长 30 分钟）→ `stopped`（空闲超时后 Host 杀死）；idle → running（温暖期间新消息到达） |
| **知识点 5** | **Container 启动流程** — `container-runner.ts` 使用 `--entrypoint bash -c 'exec bun run /app/src/index.ts'`；挂载 session 文件夹 → /workspace、agent group 文件夹 → /workspace/agent/；环境变量：AGENT_PROVIDER、NANOCLAW_ADMIN_USER_ID、TZ、API Key |
| **知识点 6** | **Container 配置** — `container_config` 列（JSON）：additionalMounts、timeout；额外挂载出现在 `/workspace/extra/{containerPath}`；只读挂载使用 `--mount type=bind,source=...,target=...,readonly` |
| **知识点 7** | **空闲检测与心跳** — `.heartbeat` 文件的 mtime 是存活信号；Host 检查心跳判断 Container 是否存活；空闲检测例外：ask_user_question 等待中、Agent 主动工作中 |
| **知识点 8** | **Session 恢复与重置** — 恢复：Host 传入 sessionId，Provider 负责恢复上下文；重置：创建新 session，标记旧消息为未处理，重新触发处理；Agent 可通过 MCP 工具请求 session 重置 |
| **阅读材料** | `docs/architecture.md`（Agent Groups vs Sessions、Container Lifecycle 章节）、`src/session-manager.ts`、`src/container-runner.ts`、`container/Dockerfile`、`container/entrypoint.sh` |
| **实践任务** | 精读 `session-manager.ts` 理解 session 创建和路径管理；精读 `container-runner.ts` 理解容器启动和挂载配置；阅读 `Dockerfile` 理解镜像构建过程 |

---

### Day 10 — 调度系统与循环任务

| 维度 | 内容 |
|------|------|
| **目标** | 掌握 NanoClaw 的调度机制——如何实现定时任务和循环任务 |
| **知识点 1** | **调度设计哲学** — 没有独立的调度器——使用同一组消息表；One-shot：process_after / deliver_after，recurrence = NULL；Recurring：同上 + recurrence cron 表达式 |
| **知识点 2** | **调度类型** — `cron` Cron 表达式（如 `0 9 * * 1` = 每周一 9 点）；`interval` 毫秒间隔（如 `3600000` = 每小时）；`once` ISO 时间戳一次性执行 |
| **知识点 3** | **Host 扫描机制** — 活跃容器（~1s）：轮询 Session DB 中新 messages_out 行；全量扫描（~60s）：扫描所有 Session DB 的到期 process_after / deliver_after；完成/交付带 recurrence 的行后，插入下一次出现 |
| **知识点 4** | **循环任务的实现** — Agent Runner 处理循环任务消息与普通消息相同；Agent Runner 标记 completed 后，Host 负责插入下一次出现；下次时间从计划时间计算（非墙钟时间），防止漂移 |
| **知识点 5** | **Pre-Agent 脚本** — task 类型消息的 script 字段；执行流程：写入临时文件 → bash 执行（30s 超时）→ 解析最后一行 JSON；`wakeAgent: false` → 不唤醒 Agent；`wakeAgent: true` → 用脚本输出丰富 prompt |
| **知识点 6** | **任务管理工具** — `schedule_task` 创建任务；`list_tasks` 列出活跃任务；`pause_task` / `resume_task` 暂停/恢复；`cancel_task` 取消；`update_task` 修改 prompt / recurrence / processAfter / script |
| **知识点 7** | **调度模块代码结构** — `src/modules/scheduling/` Host 端调度模块（actions.ts、db.ts、recurrence.ts）；`container/agent-runner/src/mcp-tools/scheduling.ts` Agent Runner 侧调度工具；`container/agent-runner/src/scheduling/task-script.ts` Pre-Agent 脚本执行 |
| **阅读材料** | `docs/SPEC.md`（Scheduled Tasks 章节）、`docs/architecture.md`（Scheduling 章节）、`src/modules/scheduling/` 目录、`container/agent-runner/src/mcp-tools/scheduling.ts` |
| **实践任务** | 阅读 `recurrence.ts` 理解 cron 解析和下次时间计算；阅读 `task-script.ts` 理解 pre-agent 脚本执行；阅读 `host-sweep.ts` 理解 Host 扫描如何触发到期任务 |

---

### Day 11 — 安全模型：隔离、权限与审批

| 维度 | 内容 |
|------|------|
| **目标** | 理解 NanoClaw 的安全架构——容器隔离、权限系统、审批流程 |
| **知识点 1** | **容器隔离** — 文件系统隔离：Agent 只能访问挂载目录；安全 Bash：命令在容器内运行；网络隔离：可按容器配置；进程隔离：容器进程不影响 Host；非 root 用户：容器以 node 用户（uid 1000）运行；tini 作为 init：回收僵尸进程、转发信号 |
| **知识点 2** | **权限系统** — 角色：owner（全局唯一，首个配对用户）、admin（全局或限定 Agent Group 范围）；成员关系：agent_group_members 表；用户身份命名空间：`<channel_type>:<handle>`；权限检查：`src/access.ts`、`src/modules/permissions/` |
| **知识点 3** | **审批流程** — 隐式审批：Agent 调用需审批工具 → Host 拦截 → 发送审批卡片给 admin → 等待响应 → 执行或拒绝；显式审批：Agent 主动请求审批 → 同样流程；审批路由：pickApprover → 范围内 admin → 全局 admin → owner；审批卡片发送到审批人 DM |
| **知识点 4** | **凭证存储与注入** — Claude CLI 认证：`data/sessions/{group}/.claude/` → 挂载到 `/home/node/.claude/`；WhatsApp Session：`store/auth/`；OneCLI 凭证代理：Host 通过 OneCLI 注入凭证；.env 中只有认证变量被提取并写入 `data/env/env` |
| **知识点 5** | **Prompt 注入防护** — 容器隔离限制爆炸半径；只处理已注册 Group 的消息；需要触发词；Agent 只能访问其 Group 挂载目录；Claude 内置安全训练 |
| **知识点 6** | **挂载安全** — `src/modules/mount-security/` 挂载白名单验证；`config-examples/mount-allowlist.json` 白名单示例；额外挂载需要 admin 审批 |
| **知识点 7** | **发送者审批** — `src/modules/permissions/sender-approval.ts` 未知发送者审批；`unknown_sender_policy`：strict（拒绝）/ request_approval（请求审批）/ public（允许） |
| **知识点 8** | **文件权限** — `groups/` 文件夹包含个人记忆，应设置 `chmod 700` |
| **阅读材料** | `docs/SPEC.md`（Security Considerations 章节）、`docs/SECURITY.md`、`src/access.ts`、`src/modules/permissions/` 目录、`src/modules/mount-security/` 目录 |
| **实践任务** | 阅读 `access.ts` 理解权限检查逻辑；阅读 `sender-approval.ts` 理解发送者审批流程；阅读 `channel-approval.ts` 理解渠道审批流程 |

---

### Day 12 — Skills 系统与自定义扩展

| 维度 | 内容 |
|------|------|
| **目标** | 掌握 NanoClaw 的 Skill 定制机制，理解如何通过 Skill 扩展项目功能 |
| **知识点 1** | **Skills 设计哲学** — 通过 Skills（分支合并到用户安装中）进行定制；不同 Skill 添加不同能力；代码结构必须确保不同定制不冲突 |
| **知识点 2** | **Skill 冲突热点与解决方案** — 见下表 |

| 热点 | 冲突原因 | 解决方案 |
|------|----------|----------|
| `src/index.ts` | 每个 Skill 修补主循环 | 薄 index，逻辑在专用模块 |
| `src/config.ts` | 每个 Skill 添加环境变量 | 配置在使用处声明 |
| `src/container-runner.ts` | 渠道 Skill 添加挂载 | 声明式挂载注册 |
| `src/db.ts` | Schema + 迁移 + CRUD 一体 | 按实体拆分 + 编号迁移 |
| `agent-runner/src/index.ts` | 协议 + IPC + 格式化一体 | 拆分为 poll-loop、formatter、providers/、mcp-tools/ |
| `src/ipc.ts` | 每个 MCP 工具修补 | mcp-tools/ 目录 + barrel |
| `src/channels/index.ts` | 每个渠道添加导入行 | Barrel 文件 + 注释槽位 |

| 维度 | 内容 |
|------|------|
| **知识点 3** | **注册模式优于 switch 语句** — 渠道、MCP 工具、Provider 使用注册/插件模式；Skill 添加一个文件 + 一个注册调用——不编辑中央 switch 语句 |
| **知识点 4** | **添加新渠道的 Skill 流程** — 一个新文件（渠道适配器或 Chat SDK 配置）；`src/channels/index.ts` 中一行导入；零修改路由、格式化、交付、容器代码 |
| **知识点 5** | **挂载注册模式** — `registerChannel('gmail', { factory, mounts, env })`；Container Runner 从注册表读取挂载——不需要编辑 container-runner.ts |
| **知识点 6** | **配置模式** — Skill 不修补 config.ts 或 .env.example；Skill 特定环境变量在 SKILL.md 中文档化；每个模块直接读取自己的环境变量；共享配置留在 config.ts |
| **知识点 7** | **DB 文件结构** — 按实体拆分不按层拆分；每个 Skill 添加编号迁移文件；迁移是只追加的编号文件 |
| **知识点 8** | **Skill 目录结构** — `.claude/skills/add-<name>/SKILL.md` Skill 定义；`.claude/skills/add-<name>/` 脚本和资源；容器内 Skill：`container/skills/<name>/SKILL.md` |
| **知识点 9** | **代码风格** — 行宽 120 字符；简洁日志：薄包装保持每个日志调用在一行；无内联 ALTER TABLE——使用迁移运行器 |
| **阅读材料** | `docs/architecture.md`（Flexibility Model 章节）、`docs/skills-as-branches.md`、`.claude/skills/` 典型 Skill、`container/skills/` 目录 |
| **实践任务** | 阅读完整渠道 Skill（如 add-telegram）理解结构；阅读 `container/skills/welcome/SKILL.md` 理解容器内 Skill；思考创建自定义 Skill 需要哪些文件和修改 |

---

### Day 13 — Setup 流程与部署

| 维度 | 内容 |
|------|------|
| **目标** | 理解 NanoClaw 的安装配置流程和部署方案 |
| **知识点 1** | **Setup 入口** — `nanoclaw.sh` 顶层包装器：Phase 1（bootstrap）+ Phase 2（setup:auto）；`setup.sh` Phase 1 bootstrap：Node、pnpm、原生模块验证；`setup/auto.ts` Phase 2 驱动：clack UI、步骤执行、用户提示 |
| **知识点 2** | **三级输出契约** — Level 1（用户面）：clack 渲染，简洁品牌化；Level 2（进展日志）：`logs/setup.log`，结构化步骤块；Level 3（原始日志）：`logs/setup-steps/NN-step-name.log`，完整子进程输出 |
| **知识点 3** | **Setup 步骤** — Bootstrap：安装 Node.js、pnpm、验证原生模块；Environment：检测 Docker / Apple Container 运行时；Container：构建容器镜像；Auth：Anthropic 凭证注册（交互式例外）；Channel：选择并配置渠道；Service：安装 launchd 服务（macOS） |
| **知识点 4** | **新步骤的契约** — 接收 raw-log 路径，所有 stdout + stderr 写入该文件；结束时发出终端状态块 `STATUS: success|skipped|failed`；长时间运行步骤可发出子状态块；硬失败时非零退出 |
| **知识点 5** | **Anthropic 例外** — `claude setup-token` 打开浏览器、运行 OAuth 提示；唯一允许的中断视觉流的地方；clack 流程显式暂停，子进程继承 stdio |
| **知识点 6** | **部署方案** — macOS：launchd 服务（`launchd/com.nanoclaw.plist`）；Linux：systemd 或直接运行；环境变量配置：ASSISTANT_NAME、CONTAINER_TIMEOUT、MAX_CONCURRENT_CONTAINERS |
| **知识点 7** | **CI 流程** — `.github/workflows/ci.yml`；安装 Node + Bun；pnpm install --frozen-lockfile（Host）；bun install --frozen-lockfile（Container）；格式检查、类型检查（Host + Container）、测试（Host + Container） |
| **知识点 8** | **Setup 代码结构** — `setup/index.ts` 入口；`setup/auto.ts` 自动化驱动；`setup/environment.ts` 环境检测；`setup/container.ts` 容器构建；`setup/service.ts` 服务安装；`setup/channels/` 渠道安装脚本；`setup/lib/` 共享工具函数 |
| **阅读材料** | `docs/setup-flow.md`、`docs/setup-wiring.md`、`nanoclaw.sh`、`setup/auto.ts`、`.github/workflows/ci.yml` |
| **实践任务** | 阅读 `nanoclaw.sh` 理解两阶段安装流程；阅读 `setup/auto.ts` 理解步骤编排逻辑；阅读 CI 流程 |

---

### Day 14 — 综合复习与实战演练

| 维度 | 内容 |
|------|------|
| **目标** | 整合两周所学知识，通过实战项目巩固理解，形成完整的知识体系 |
| **知识点 1** | **架构全景回顾** — 双进程模型：Host（Node.js）+ Container（Bun）；三级数据库：Central DB + Session Inbound DB + Session Outbound DB；消息驱动：一切都是消息，DB 是唯一 IO 边界；渠道自注册：工厂模式 + Barrel 导入；Provider 抽象：统一接口，多后端支持；Skill 定制：分支合并，最小冲突 |
| **知识点 2** | **核心数据流回顾** — 用户 → 渠道适配器 → Host 路由 → inbound.db → Agent Runner 轮询 → 格式化 → Provider → outbound.db → Host 交付 → 渠道适配器 → 用户 |
| **知识点 3** | **关键设计决策回顾** — 为什么用 SQLite 而不是消息队列：简单、可靠、无外部依赖；为什么用轮询而不是事件驱动：跨挂载边界的事件不可靠；为什么 Host 和 Container 用不同运行时：Baileys 原生绑定 vs Bun 性能；为什么用 DELETE 模式而不是 WAL：跨挂载 WAL 可见性问题；为什么单写者原则：消除跨挂载写冲突 |
| **知识点 4** | **实战演练：追踪一个完整请求** — 从用户在 Telegram 发送 `@Andy 今天天气如何？` 开始；追踪消息经过的每一个模块和函数；记录每一步的数据格式变化；理解每个设计决策的影响 |
| **知识点 5** | **实战演练：添加一个新功能** — 设计一个简单的 MCP 工具（如 `get_current_time`）；在 `container/agent-runner/src/mcp-tools/` 添加工具文件；更新 barrel 导出；编写测试 |
| **知识点 6** | **知识体系自检清单** — 能画出完整架构图；能解释消息从入站到出站的完整流程；能说明 Central DB 和 Session DB 的职责划分；能描述渠道自注册的实现机制；能解释 Agent Runner 的轮询循环逻辑；能列出所有 MCP 工具及其功能；能对比三种 Provider 的实现差异；能说明 Container 的生命周期状态转换；能解释调度系统的实现原理；能描述安全模型的各个层面；能说明 Skill 系统的冲突避免策略；能追踪 Setup 流程的完整步骤 |
| **知识点 7** | **进阶学习方向** — 深入 Chat SDK 适配器开发；自定义 Provider 实现；复杂 Agent 编排（如 PR Factory 模式）；性能优化与监控；多租户与权限扩展 |
| **阅读材料** | 回顾所有 `docs/` 下的文档、`docs/architecture.md`（PR Factory 示例章节）、`docs/SDK_DEEP_DIVE.md` |
| **实践任务** | 完成知识体系自检清单；实现一个简单的 MCP 工具并测试；撰写个人学习笔记 |

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

| 序号 | 建议 |
|------|------|
| 1 | **先读文档，再读代码**：`docs/` 下的文档质量很高，先建立概念模型再深入源码 |
| 2 | **画图辅助理解**：架构图、时序图、状态机图能极大帮助理解复杂系统 |
| 3 | **从测试入手**：测试用例是最好的代码文档，理解预期行为后再看实现 |
| 4 | **追踪一个完整请求**：从用户发送消息到 Agent 响应，追踪每一步 |
| 5 | **动手实践**：尝试添加一个简单的 MCP 工具或渠道，巩固理解 |
| 6 | **关注设计决策**：理解"为什么"比理解"怎么做"更重要 |
