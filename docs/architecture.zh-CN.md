# NanoClaw 架构（草案）

## 核心理念

每个 agent session 都挂载一个 SQLite DB。这个 DB 是 host 与 container 之间唯一的 IO 机制。没有 IPC 文件，也没有 stdin 管道。核心表是两张：`messages_in`（host → agent-runner）和 `messages_out`（agent-runner → host）。一切都是消息。

## 两级 DB

**Central DB（host 进程）：**
- Agent groups、conversations、routing tables
- 映射 platform IDs → agent groups → sessions
- Channel adapters 不直接访问它，由 host 负责查找

**Per-session DB（挂载进 container）：**
- `messages_in`（host 写入，agent-runner 读取）
- `messages_out`（agent-runner 写入，host 读取）
- 一切都是消息：chat、tasks、webhooks、system actions、agent-to-agent 都使用这两张表
- 每个 session 一个 DB，而不是每个 agent group 一个 DB

## Agent Groups 与 Sessions

一个 agent group 拥有自己的文件系统，包括 folder、`CLAUDE.md`、skills、container config。多个 sessions 可以共享同一个 agent group（同一个文件系统、同一组 skills），但每个 session 都会获得自己的 DB，并挂载到一个已知路径。每个 session = 一个独立 container，使用同一个 agent group 的文件系统，但使用不同的 session DB。

## 消息流

```
Platform event
  → Channel adapter（trigger check、ID extraction）
  → 返回：{ platformChannelId, platformThreadId, triggered }
  → Host 映射 platformChannelId + platformThreadId → agent group + session
  → Host 将消息写入 session DB
  → Host 调用 wakeUpAgent(session)
  → Container 启动（或已经在运行）
  → Agent-runner 轮询自己的 session DB，发现新消息
  → Agent-runner 使用 Claude 处理
  → Agent-runner 将响应写入 session DB
  → Host 轮询活跃 session DB 中的响应
  → Host 读取响应、查找 conversation，并通过 channel adapter 投递
```

## Channel Adapters

Channel adapters 负责：
1. 接收 platform events（webhooks、polling、websockets，具体取决于平台）
2. **过滤**：决定哪些消息应转发给 host 处理。这可以是无状态的（regex trigger match），也可以是有状态的（例如：“bot 是否曾在这个 thread 中被提及？如果是，后续所有消息都转发”）。Adapter 接收一条未过滤的平台消息流，并决定哪些消息要传递下去。具体判断方式是实现细节，NanoClaw 不知道也不关心。
3. 提取并标准化两个 ID：
   - **Platform channel ID**：标识 conversation（WhatsApp group、Slack channel、email thread）
   - **Platform thread ID**：可选的子上下文（Slack thread、GitHub PR comment thread）
4. 出站投递，即把响应发回平台

Channel adapter 不知道 agent group IDs 或 session IDs。它只返回平台层面的标识符。Host 将这些标识符映射到实体模型。

两级 ID 方案（channel ID + thread ID）提供了灵活性：
- 希望每个 Slack thread 都是独立 session？返回唯一 thread IDs。
- 希望一个 Slack channel 中所有消息共享 session？返回同一个 thread ID（或 `null`）。
- 这是按 channel 配置的，不是全局配置。

### Channel Adapter 配置

Adapters 是无状态的：它们在 setup 时从 host 接收配置，而不是直接从 DB 读取。

**代码中保存的内容（按 channel type，运行时不变）：**
- Auto-registration 行为（启用/禁用，以及具体方式）
- Sender allowlist 规则
- Allowlisted senders 是否可以自动注册 groups
- 平台特定的连接和消息处理

这些决策是在设置 channel adapter 时做出的。要修改它们，就修改代码。

**DB 中保存的内容（按 group，各 group 可以不同）：**
- 由哪个 agent group 处理
- Trigger / filter 规则（regex、仅 @mention、排除某些 senders 等）
- Response scope（响应所有消息，还是只响应 triggered/allowlisted）
- Session mode（shared 或 per-thread）

Host 从 DB 读取 per-group config，并在 setup 时传给 adapter。如果运行时配置发生变化（admin agent 注册新 group、修改 trigger），host 会调用 adapter 的 update method。

### Auto-Registration

当 adapter 从未知 group 转发消息时，host 需要决定是否为它创建 group 和 session。

**Adapter 控制是否转发未知消息**，依据是它的代码级 auto-registration 规则（sender allowlist、group-add detection 等）。如果 adapter 转发了消息，host 就创建 group + session。

**已知 groups 的 session 创建：**
- Shared session mode：host 查找已有 session，如果是第一条消息则创建一个
- Per-thread session mode：host 按 `threadId` 查找。如果这个 thread 还没有 session，则用同一个 agent group 自动创建一个

**代码级规则是 channel-specific 的：**
- WhatsApp：如果 allowlisted number 将 bot 加入 group → auto-register。如果未知号码发 DM → 取决于 adapter 配置。
- Email：如果 sender 已知 → auto-register 该 thread。如果未知 → drop。
- Slack：如果有人在新 channel 中 @mention bot → adapter 根据自己的规则决定是否转发。

没有 `channel_configs` 表；channel-type 级别的行为固化在 adapter 代码中。

### Chat SDK 集成

Chat SDK adapters 会按 channel 包装：
- 每个 Chat SDK adapter 都有自己的 Chat instance
- Concurrency mode 按 channel 配置（chat 用 concurrent，tasks 用 queue，webhooks 用 debounce）
- Bridge 包装 Chat instance + adapter，使其符合 NanoClaw 的标准 channel interface
- Chat SDK 处理：webhook parsing、dedup、message history、platform API calls、rich content delivery
- NanoClaw 处理：routing、agent lifecycle、session management

**Chat SDK 的 subscription model：**

Chat SDK 有自己的 thread-level subscription 概念（不同于 NanoClaw 的 channel-level registration）：
- `onNewMention` / `onNewMessage(regex)`：首次接触时触发（例如 Slack thread 中的 @mention）
- `thread.subscribe()`：订阅该 thread 中所有未来消息
- `onSubscribedMessage`：对已订阅 threads 中的所有消息触发

这是 sub-channel 粒度。NanoClaw 在 channel 级注册（“监听这个 Discord channel”）。Chat SDK 在 thread 级订阅（“跟踪这个特定 Slack thread”）。Bridge 允许 Chat SDK 在内部管理自己的 subscriptions，NanoClaw 不干预也不复制这套机制。

**平台能力差异：**

不同 adapters 的能力差异很大（见 [Chat SDK adapter docs](https://chat-sdk.dev/docs/adapters)）：
- **Slack**：完整 rich content（Block Kit cards、modals、streaming、reactions、ephemeral messages）
- **Discord**：Embeds、buttons、通过 post+edit 实现 streaming
- **WhatsApp (Cloud API)**：仅 DMs、interactive reply buttons、无 streaming、无 reactions
- **GitHub/Linear**：Markdown comments、无 interactive elements
- **Telegram**：Inline keyboard buttons、通过 post+edit 实现 streaming

Host/bridge 负责 graceful degradation：如果 agent 在不支持 cards 的平台上发布 card，则 fallback 到 text。

非 Chat-SDK channels（通过 Baileys 的 WhatsApp、Gmail、自定义 integrations）直接实现 NanoClaw channel interface，不使用 bridge，也不使用 Chat SDK types。

## Container 生命周期

Host 是 orchestrator：
1. **Spawn**：调用 `wakeUpAgent` 且该 session 没有 container 时
2. **Idle kill**：container 在一段 timeout 内没有未处理消息时
3. **Limits**：`MAX_CONCURRENT_CONTAINERS` 限制活跃 containers 数量

Container 启动后，agent-runner 立即开始轮询它的 session DB。消息已经在那里等待。

## Media Handling

### Inbound

Host 不下载 media。取而代之：
- Messages 包含 download URLs（尽可能使用 signed URLs）
- Agent-runner 在 container 内下载并处理 media
- 对于 signed URLs 不适用的 channels（例如带 buffered streams 的 WhatsApp），channel adapter 下载 media，并通过 container 可访问的本地 URL/server 提供

**Native content blocks（provider-dependent）：**

Agent-runner 检测文件类型，并在 provider 支持时将支持的类型作为 native content blocks 传递：

| Type | Claude | Codex | OpenCode |
|------|--------|-------|----------|
| Images（JPEG、PNG、GIF、WebP） | Native image content block | 保存到磁盘，在 prompt 中引用 | 保存到磁盘，在 prompt 中引用 |
| PDFs | Native document content block | 保存到磁盘 | 保存到磁盘 |
| Audio | Native audio content block | 保存到磁盘 | 保存到磁盘 |
| Other files（code、data、video、archives） | 保存到磁盘 | 保存到磁盘 | 保存到磁盘 |

“保存到磁盘”表示下载到 `/workspace/downloads/{messageId}/`，并在 prompt text 中作为可用文件路径引用。Agent 可以用 tools（Read、Bash）访问它。

Agent-runner 会按 provider 不同构建不同 prompt。对于 Claude，它构造包含 image/document blocks 的 multi-part `MessageParam` content。对于 Codex/OpenCode，所有内容都是 text + 文件路径引用。

### Outbound

Outbound file delivery 基于 tool。Agent 调用一个 tool（例如 `send_file`）并传入文件路径。Agent-runner 将文件移动到 outbox，并写入 `messages_out` 行。

```
/workspace/
  outbox/
    {message_id}/        ← 每个 messages_out row 一个目录
      chart.png
      report.pdf
```

`messages_out` content 只引用文件名：

```json
{ "text": "Here's the chart", "files": ["chart.png", "report.pdf"] }
```

DB 中没有路径；约定就是 contract。Host 从挂载的 session folder 中读取 `outbox/{message_id}/` 下的文件，并通过 adapter 投递（Chat SDK `FileUpload` 携带 buffer data，或 native channels 使用平台特定 upload）。Host 在成功投递后清理 outbox directory。

Outbound files 使用专用的 `send_file` MCP tool（与 `send_message` 分离）。Tool interface 见 [agent-runner-details.md](agent-runner-details.md)。

### Message Deduplication

Dedup 是 channel adapter 的职责。Chat SDK 内部处理这一点。Native adapters 根据需要跟踪 platform message IDs。Host 不做 deduplicate：只要 adapter 转发，host 就写入。

## Session DB Schema

两张表。Content 使用 JSON blobs；schema-free，格式随 `kind` 变化。

```sql
-- Host 写入，agent-runner 读取
CREATE TABLE messages_in (
  id             TEXT PRIMARY KEY,
  kind           TEXT NOT NULL,      -- 'chat' | 'chat-sdk' | 'task' | 'webhook' | 'system'
  timestamp      TEXT NOT NULL,
  status         TEXT DEFAULT 'pending',  -- 'pending' | 'processing' | 'completed' | 'failed'
  status_changed TEXT,               -- 最后一次状态变化的 ISO timestamp
  process_after  TEXT,               -- ISO timestamp。NULL = 立即处理。
  recurrence     TEXT,               -- cron expression。NULL = one-shot。
  tries          INTEGER DEFAULT 0,  -- 处理尝试次数

  -- routing（agent-runner 复制到 messages_out；agent 永远看不到这些字段）
  platform_id    TEXT,
  channel_type   TEXT,
  thread_id      TEXT,

  -- payload（结构取决于 kind）
  content        TEXT NOT NULL        -- JSON blob
);

-- Agent-runner 写入，host 读取
CREATE TABLE messages_out (
  id             TEXT PRIMARY KEY,
  in_reply_to    TEXT,               -- references messages_in.id（可选）
  timestamp      TEXT NOT NULL,
  delivered      INTEGER DEFAULT 0,
  deliver_after  TEXT,               -- ISO timestamp。NULL = 立即投递。
  recurrence     TEXT,               -- cron expression。NULL = one-shot。

  -- routing（默认由 agent-runner 从 messages_in 复制）
  kind           TEXT NOT NULL,      -- 'chat' | 'chat-sdk' | 'task' | 'webhook' | 'system'
  platform_id    TEXT,
  channel_type   TEXT,
  thread_id      TEXT,

  -- payload（格式匹配 kind）
  content        TEXT NOT NULL        -- JSON blob
);

```

### Scheduling

One-shot 和 recurring tasks 使用同一组表；没有单独的 scheduler。

**One-shot：** `process_after`（inbound）或 `deliver_after`（outbound），并且 `recurrence = NULL`。

**Recurring：** 同上，再加一个 `recurrence` cron expression。Host 将一行标记为 handled/delivered 后，如果设置了 `recurrence`，就插入新行，并将 `process_after`/`deliver_after` 推进到下一次 cron occurrence。下一次时间从 scheduled time 计算（不是 wall clock），以避免 drift。

**Host sweep**（大约每 60 秒遍历所有 session DBs）：
- `messages_in WHERE status = 'pending' AND (process_after IS NULL OR process_after <= now())` → wake agent
- `messages_in WHERE status = 'processing' AND status_changed < (now - stale_threshold)` → stale detection，递增 tries，带 backoff 重置为 pending
- `messages_out WHERE delivered = 0 AND (deliver_after IS NULL OR deliver_after <= now())` → deliver
- 完成/投递带有 `recurrence` 的行后，插入下一次 occurrence

**Active container poll**（大约 1 秒）检查同样条件，但只针对有运行中 containers 的 sessions。

**Agent-runner 创建 schedules**：写入 `messages_in`（发给自身）或带 `process_after` 且可选 `recurrence` 的 `messages_out`（reminders/notifications）。

### messages_in content by kind

**`chat`**：简单 NanoClaw format。任何 channel 都可以生成。
```json
{
  "sender": "John",
  "senderId": "user123",
  "text": "Check this PR",
  "attachments": [{ "type": "image", "url": "https://signed-url..." }],
  "isFromMe": false
}
```

**`chat-sdk`**：完整 Chat SDK `SerializedMessage`，由 bridge adapter 透传。包含 `author`、`text`、`formatted`（mdast AST）、`attachments`、`isMention`、`links`、`metadata`。

**`task`**：scheduled task firing。
```json
{ "prompt": "Review open PRs", "script": "scripts/review.sh" }
```

**`webhook`**：raw webhook payload。
```json
{ "source": "github", "event": "pull_request", "payload": { ... } }
```

**`system`**：host action result（对 agent 请求的 system action 的响应）。
```json
{ "action": "register_group", "status": "success", "result": { "agent_group_id": "ag-456" } }
```

### messages_out content by kind

Output `kind` 决定格式和 delivery adapter。默认情况下，agent-runner 会从它响应的 `messages_in` 行复制 `kind` 和 routing fields。

**`chat`**：简单 NanoClaw format。NanoClaw channel 通过 `sendMessage(text)` 投递。
```json
{ "text": "LGTM, merging now" }
```

**`chat-sdk`**：Chat SDK `AdapterPostableMessage`。Bridge adapter 通过 `thread.post()` 投递。可以是 markdown、card 或 raw；adapter 负责平台转换。
```json
{ "markdown": "## Review\n**LGTM**", "attachments": [...] }
```
```json
{ "card": { "type": "card", "title": "Review", "children": [...] }, "fallbackText": "..." }
```

**`task`**：task result。Host 记录日志并可选通知。
```json
{ "result": "3 PRs reviewed", "status": "success" }
```

**`webhook`**：webhook response。Host 发送 HTTP response 或通知。
```json
{ "response": { "status": 200, "body": { ... } } }
```

**`system`**：host action request（register group、reset session 等）。Host 读取、验证权限、执行，并将结果作为 `system` `messages_in` 行写回。
```json
{ "action": "reset_session", "payload": { "session_id": "sess-123" } }
```

### Interactive Operations（Cards、Reactions、Edits）

所有 interactive operations 都通过 `messages_in`/`messages_out` 流转；DB 是 container 唯一的 IO 边界。Agent 使用 MCP tools；agent-runner 将 tool calls 转换为结构化 `messages_out` 行；host 通过合适的 adapter method 投递。

**带用户交互的 Cards（例如 “Ask User Question”）：**

1. Agent 调用 `ask_user_question` tool，传入 question + options
2. Agent-runner 写入带 question card 的 `messages_out`
3. Host 通过 adapter 以 interactive card 形式投递（例如 Slack Block Kit buttons）
4. User 点击某个 option
5. Platform 将 event 发回 adapter → host 写入包含 response 的 `messages_in`
6. Agent-runner 读取 `messages_in`，匹配到 pending tool call，并将 selection 作为 tool result 返回给 agent

Agent-runner 在等待用户通过 `messages_in` 响应时保持 tool call open。完整往返是：agent → `messages_out` → host → platform → user clicks → platform → host → `messages_in` → agent-runner → agent。

**Approvals：**

两种模式，都在 host level 处理：
- **Implicit**：Agent 调用需要 approval 的 tool。Host 拦截，向 admin 发送 approval card，等待响应，然后执行或拒绝。Agent 不知道 approval step。
- **Explicit**：Agent 通过 tool 显式请求 approval。Agent-runner 将 approval request 写入 `messages_out`。流程与 “ask user question” 相同，响应通过 `messages_in` 回来。

两种情况下，approval 和 action execution 都发生在 host side，而不是 agent side。

**Approval routing：** Privilege 是 user-level 概念。`user_roles` 记录 `owner`（仅 global，第一个配对用户成为 owner）和 `admin`（global 或 scoped 到特定 `agent_group_id`）。当 action 需要 approval 时，`pickApprover(agentGroupId)` 按顺序返回候选人：该 agent group 的 scoped admins → global admins → owners（去重）。`pickApprovalDelivery` 再选择第一个可通过 `ensureUserDm` 触达的候选人（带 same-channel-kind tie-break，因此 Discord approval request 会优先选择使用 Discord 的 approver）。Approval card 会落到 approver 的 DM messaging group，而不是来源 chat。对需要 resolution 的 channels（Discord/Slack/…），delivery 通过 Chat SDK 的 `openDM` 解析；对可直接寻址的 channels（Telegram/WhatsApp/…），使用用户 handle，并将映射缓存在 `user_dms` 中供后续请求使用。见 `src/access.ts`、`src/user-dm.ts`。

**编辑已发送消息：**

Agent 调用 `edit_message` tool，传入 message ID 和新内容。Agent-runner 写入带 edit operation 的 `messages_out`。Host 调用 `adapter.editMessage()`。Agent context 中的 messages 包含 integer IDs，因此 agent 可以引用它们。

**Reactions：**

Agent 调用 `add_reaction` tool，传入 message ID 和 emoji。Agent-runner 写入带 reaction operation 的 `messages_out`。Host 调用 `adapter.addReaction()`。

**`messages_out` content 中的 operations：**

```json
// Normal message（默认）
{ "text": "LGTM" }

// Interactive card
{ "operation": "ask_question", "title": "Deploy", "question": "Approve deployment?", "options": ["Yes", "No", "Defer"] }

// Edit existing message
{ "operation": "edit", "messageId": "3", "text": "Updated: LGTM with minor comments" }

// Reaction
{ "operation": "reaction", "messageId": "5", "emoji": "thumbs_up" }
```

Host 读取 `operation` field（如果存在）并调用正确的 adapter method。没有 operation field = 普通消息投递。平台能力各不相同；host/bridge 负责 graceful degradation（例如平台不支持 reaction → skip 或作为 text 发送）。

### Agent-to-Agent Communication

向另一个 agent 发送消息使用与 channel delivery 相同的 routing fields。Agent-runner 设置 `channel_type: 'agent'`，并将 `platform_id` 设置为目标 agent group ID。可选地，`thread_id` 可以指向特定 session（`null` = 查找或创建 default session）。

从发送 agent 的视角看，这与发送到 Slack 或 WhatsApp 是同一种机制，只是 `messages_out` 行使用不同 routing。Host 读取后，检查该 agent group 是否有权限向目标发送消息，解析目标 session，并向目标 session DB 写入一行 `messages_in`。

```json
// messages_out routing fields
{ "kind": "chat", "channel_type": "agent", "platform_id": "pr-worker", "thread_id": null }
// messages_out content
{ "text": "Reset your session and re-review", "sender": "Supervisor", "senderId": "agent:pr-admin" }
```

接收 agent 得到的是普通 chat message。除非上下文相关，否则它不需要知道消息来源是另一个 agent。

### Routing

**默认行为：** Agent-runner 从 `messages_in` 行复制 routing fields（`kind`、`platform_id`、`channel_type`、`thread_id`）到 `messages_out`。响应回到原始来源。

**Host validation：** 投递前，host 检查该 agent group 是否被允许发送到目标。Agent-runner 复制 routing；host 负责验证。

**Multi-destination pattern（customization）：** Agent 可能需要发送到不同于来源的 channel（例如 webhook 触发 Slack notification）。这通过自定义代码支持，不内置于 core：

1. 向 session DB 添加 `destinations` 表，将 logical names 映射到 routing fields
2. Host 设置 session 时填充该表
3. 修改 agent prompt，列出可用 destinations
4. Agent 按名称选择 destination；agent-runner 解析为 routing fields
5. Host 照常验证

这是一个 documented pattern，不是 built-in feature。

## 核心属性
- 通过 filesystem mounts 实现 container isolation
- Credential proxy（OneCLI）
- Per-agent-group workspace（folder、`CLAUDE.md`、skills）
- 基于 polling（不是 event-driven）
- Container startup 时按 agent group 重新编译 agent-runner（agent 可以修改自己的 source、请求 rebuild/restart，变更在 teardown 后仍保留）
- Host ↔ container IO 通过挂载的 session DBs（`messages_in` / `messages_out`）完成，不使用 stdin 管道或 IPC files
- Agent commands 是 `messages_out` 行，且 `kind: 'system'`
- 支持通过 `messages_out` 上的 target-agent routing 实现 agent-to-agent
- Scheduling 在同一组 message tables 上使用 `process_after` / `deliver_after` + `recurrence`
- Media 通过 signed URLs，在 container 内下载
- Channel adapters 使用 Chat SDK bridge + 标准 interface（trunk 只提供 bridge/registry；platform adapters 通过 `/add-<channel>` skills 安装）
- Routing：channel adapter 提取 IDs，host 映射到 entities
- Concurrency：Chat SDK per-channel + container limits
- Session scoping：per-session DB，一个 agent group 可以有多个 sessions

## 设计决策

**Session DB location：** 不放在 agent group folder 中。使用独立目录（例如 `sessions/{session_id}/`）。每个 session 都有自己的 folder，里面包含 `session.db` 和 Claude SDK 的 `.claude/` directory。Session identity 就是 folder，不需要跟踪 Claude SDK session IDs。

**Container mount structure：**

```
/workspace/                 ← mount: session folder（read-write）
  .claude/                  ← Claude SDK session data（auto-created）
  session.db                ← session SQLite DB
  outbox/                   ← agent-runner 在这里写 outbound files
  agent/                    ← mount: agent group folder（nested, read-write）
    CLAUDE.md               ← agent instructions
    skills/                 ← agent skills
    ... working files
```

两个 directory mounts：session folder 挂到 `/workspace`，agent group folder 挂到 `/workspace/agent/`。Agent-runner `cd` 到 `/workspace/agent/` 运行 agent。Claude SDK 将 `.claude/` 写到 `/workspace/.claude/`（workspace root）。Session DB 位于 `/workspace/session.db`。

这同时适用于 Docker（nested bind mounts）和 Apple Container（仅支持 directory mounts，不支持 file-level mounts，但支持 nested directory mounts）。

**Session DB concurrent access：** Host 写 `messages_in`，agent-runner 写 `messages_out`。两者同时访问同一个 SQLite 文件。WAL mode 处理这一点：SQLite 允许 concurrent readers；两边写入不同的表，因此 writer contention 很小。Host 创建 session DB 时启用 WAL mode。

**Session management：** 由 host 管理。Host 创建 session folders 并挂载它们。Container 只能看到自己的 session folder。

**Session creation（无 race condition）：**

1. Message 到达，host 检查 central DB 是否已有匹配该 group + thread 的 session
2. 没有 session → host 在 central DB 中原子创建 session row，创建 session folder，创建 session DB，并写入 message
3. Container 启动前有更多 messages 到达 → host 找到已有 session，写入同一个 session DB
4. Container 启动、挂载 folder，agent-runner 发现等待中的 messages

Central DB session row creation 是 serialization point。无需协调 Claude SDK session ID；SDK 在 agent 运行时会自行发现 `.claude/` 中的 session data。

**System actions：** Agent 使用 MCP tools（register group、reset session、schedule task 等）。Agent-runner 处理这些 tool calls，并写入结构化、确定性的 `messages_out` 行，且 `kind: 'system'`。这不是自然语言，而是 programmatic structured payload，host 会确定性处理。Host 验证权限、执行，并将结果作为 `system` `messages_in` 行写回。

**Container lifecycle：** 没有 warm pool。Containers 按需 spawn（`wakeUpAgent`），并在 idle 时由 host 从外部 teardown。现有 idle detection + teardown 机制延续使用。

## 运行行为

### Output Delivery

NanoClaw 不向用户 stream tokens。Claude Agent SDK 的 `query()` 产出完整结果。Agent-runner 对每个 result 向 `messages_out` 写入一条完整 message。Host 向 channels 投递完整 messages。

Message editing 被支持为显式 operation（agent 调用 `edit_message` tool），不是 streaming 机制。

Typing indicators：host 在某 session 有活跃 container 时设置 typing；当 container 退出或 `messages_out` 中出现响应时清除 typing。

### Message Batching

当 container 关闭时多条 messages 到达，它们会作为 `handled = 0` 行累积在 `messages_in` 中。Container 被唤醒后，agent-runner 查询所有 unhandled messages，并作为 batch 处理；多条 messages 会被格式化为一个 `<messages>` XML block。

### Message Lifecycle

```
pending → processing → completed
                    → failed（超过最大重试次数后）
```

- **pending**：由 host 写入。准备被 picked up（如果 `process_after` 为空或已到期）。
- **processing**：Agent-runner pick up message 时设置。`status_changed` 设置为当前时间。防止其他 polls 重复 pick 同一 message。
- **completed**：Agent-runner 成功处理后设置。
- **failed**：最大重试次数耗尽后设置。

**Stale detection**：如果 message 处于 `processing`，但 `status_changed` 过旧（例如 >10 分钟），host 假定 container 崩溃。它将 message 重置为 `pending`，递增 `tries`，并用 exponential backoff 设置 `process_after`。

### Error Handling and Retries

Retries 使用 `process_after` 和 exponential backoff。每次 retry 都会递增 `tries`，并将 `process_after` 推迟得更远：

- Try 1：立即
- Try 2：+5s
- Try 3：+10s
- Try 4：+20s
- Try 5：+40s
- 超过最大重试次数后：status 设置为 `failed`

由 host 计算这一点，而不是 agent-runner。当 host 检测到 stale `processing` message 或 container 以 error 退出时，它递增 `tries`，计算下一次 `process_after`，并将 status 重置为 `pending`。

**Output-sent protection**：如果 `messages_out` 已经有该 batch 的 delivered rows，则不要 retry（防止向用户发送重复消息）。

### Host Polling

两层：
- **Active containers（约 1 秒）**：轮询 session DBs，查找要投递的新 `messages_out` rows
- **All sessions（约 60 秒）**：扫描所有 session DBs，查找到期的 `process_after` / `deliver_after` timestamps，并处理 recurrence

## 灵活性模型

这个架构是**对代码变更灵活，而不是对一切都可配置**。高级设置（例如下面的 PR Factory）使用自定义 routing logic 和 host-side hooks，而不是数据库配置列。

### 面向 Skill Customization 的代码结构

NanoClaw 通过 skills 自定义；skills 是会 merge 到用户安装中的 branches。不同 skills 添加不同能力（channels、integrations、behaviors）。代码结构必须满足：

1. **不同 customizations 不冲突。** 添加 Slack 和添加 Telegram 不应产生 merge conflicts。添加新 MCP tool 不应与添加 channel 冲突。每类 customization 应该触碰自己的 file(s)。

2. **核心功能块放在独立文件。** Channel registration、message formatting、MCP tools、routing logic、container management 各自位于专门文件。修改消息格式的 skill 不应触碰负责 container spawning 的文件。

3. **Index file 保持轻薄。** 它负责 wiring（init DB、start adapters、start poll loops），但不包含 business logic。所有逻辑都位于 purpose-specific modules，使 skills 可以独立修改。

4. **不要过度拆分。** 简单变更（例如添加新的 message kind）不应要求编辑 5 个文件。相关逻辑放在一起。目标是每个 skill 的核心变更只触碰 1-2 个文件。

5. **Registration patterns 优于 switch statements。** Channels、MCP tools 和 providers 应使用 registration/plugin patterns。Skill 添加 channel 的方式应是添加一个文件和一个 registration call，而不是编辑 central switch statement 并与所有其他 channels 共享冲突点。

**实践示例：** 通过 skill 添加新 channel 应该只需要：
- 一个新文件（channel adapter 或 Chat SDK config）
- Barrel file（`channels/index.ts`）中的一行，用来 import self-registering module
- 对 routing、formatting、delivery 或 container code 零修改

### Conflict Hotspots and Solutions

对 33 个 skill branches 的分析显示，这些文件最容易产生 merge conflicts：

| Hotspot | 为什么冲突 | Solution |
|-----------|-----------------|-------------|
| `src/index.ts`（2000 LOC） | 每个 skill 都 patch main loop、imports、init logic | Thin index 只负责 wiring modules。Logic 位于 purpose-specific files（router、delivery、session-manager、host-sweep）。 |
| `src/config.ts` | 每个 skill 都向 central file 添加 env vars | Config 在使用处声明。每个 module 读取自己的 env vars。不设置每个 skill 都要编辑的 central config registry。 |
| `src/container-runner.ts` | Channel skills 添加 mounts、env vars、credential setup | Declarative mount registration。Channels 在自己的文件声明 mounts。Container runner 从 registry 读取，而不是 hardcoded list。 |
| `src/db.ts`（750 LOC） | Schema、migrations 和全部 CRUD 在一个文件 | 按 entity 拆分。Numbered migrations。Skills 添加 migration file + 编辑一个 entity file。 |
| `container/agent-runner/src/index.ts` | Agent protocol、IPC handling、formatting 全在一个文件 | 拆分为 poll-loop、formatter、providers/、mcp-tools/。Session DB 替代 IPC。 |
| `src/ipc.ts` | 每次添加 MCP tool 都 patch 同一个文件 | `mcp-tools/` directory + barrel。Skills 添加 tool file + barrel line。 |
| `src/channels/index.ts` | 每个 channel 都在同一位置添加 import line | 带 per channel comment slots 的 barrel file（当前 pattern 可用，继续保留）。 |

**Mount registration pattern：** 不让每个 channel skill 都编辑 `buildVolumeMounts()`，而是让 channels 声明自己的 mounts，由 container runner 收集：

```typescript
// channels/gmail.ts
registerChannel('gmail', {
  factory: createGmailAdapter,
  mounts: [
    { hostPath: '~/.gmail-mcp', containerPath: '/home/node/.gmail-mcp', readonly: false }
  ],
  env: ['GMAIL_OAUTH_TOKEN'],
});
```

Container runner 从 channel registry 读取 registered mounts，因此无需编辑 `container-runner.ts`。

**Config pattern：** Skills 不 patch `config.ts` 或 `.env.example`。Skill-specific env vars 记录在该 skill 的 `SKILL.md` 中；setup process 读取这些说明。每个 module 直接读取自己使用的 env vars：

```typescript
// channels/discord.ts
const DISCORD_TOKEN = process.env.DISCORD_BOT_TOKEN;

// channels/gmail.ts
const GMAIL_CREDS = process.env.GMAIL_CREDENTIALS_PATH;
```

Shared config（`DATA_DIR`、`TIMEZONE`、`MAX_CONCURRENT_CONTAINERS`）保留在 `config.ts`。Channel/skill-specific config 留在使用它的 module 中。

### Code Style

**Line width：120 characters。** 大多数 statements 可以在不牺牲可读性的情况下保持一行。

**Concise logging。** 轻量 wrapper 让每个 log call 保持一行：

```typescript
log.info('IPC message sent', { chatJid, sourceGroup });
log.warn('Unauthorized IPC attempt', { chatJid });
log.error('Error processing', { file, err });
```

### DB File Structure

DB layer 按 entity 拆分，而不是保存在单个 monolithic file 中：

```
src/db/
  connection.ts              ← singleton、init、WAL mode
  schema.ts                  ← CREATE TABLE statements（当前状态，供参考）
  migrations/
    index.ts                 ← runner：检查 version，应用 pending
    001-initial.ts           ← initial schema
    002-pending-questions.ts ← 示例：添加 pending_questions table
    ...                      ← skills 追加新的 numbered files
  agent-groups.ts            ← agent_groups 的 CRUD
  messaging-groups.ts        ← messaging_groups + messaging_group_agents 的 CRUD
  sessions.ts                ← sessions + pending_questions 的 CRUD
  index.ts                   ← barrel：re-exports everything
```

**Principles：**
- **按 entity 拆分，不按 layer 拆分。** 每个 entity file 有自己的 CRUD functions（约 50-100 行）。给 `messaging_groups` 添加 column 的 skill 编辑 `messaging-groups.ts`，不触碰 sessions 或 agent groups。
- **Schema 表示当前状态，migrations 表示历史。** `schema.ts` 记录 DB 当前长什么样（理解 schema 时读它）。Migrations 是 append-only numbered files，描述如何演进到当前状态。
- **没有 inline ALTER TABLE。** Migration runner + `schema_version` table 替代 `try { ALTER TABLE } catch { /* exists */ }` blocks。启动时，它检查当前 version，并按顺序应用 pending migrations。每个 migration 是一个函数：`(db: Database) => void`。
- **Skills 添加 migrations。** 需要新 column 的 skill 添加新的 numbered migration file。只要 numbers 不冲突（使用 timestamps 或足够高的 numbers 给 skill branches），就不会与其他 skills 的 migrations 冲突。

**Agent-runner session DB** 使用同一 pattern，但更轻量；不需要 migrations，因为 session DBs 都由 host 新建：

```
container/agent-runner/src/db/
  connection.ts          ← 在固定路径打开 session.db，WAL mode
  messages-in.ts         ← 读取 pending，更新 status
  messages-out.ts        ← 写 results、outbox queries
  index.ts               ← barrel
```

### Base architecture 必须原生支持的能力

这些是 building blocks。它们不需要特殊 abstractions，而是自然来自 per-session DBs、host-managed routing，以及 `kind: 'system'` 的 `messages_out`：

1. **同一个 channel 上多个 agent groups，基于 content routing。** 同一 thread 中的不同 messages 可以按 content route 到不同 agent groups（例如 @mention route 到 supervisor，普通 messages route 到 worker）。由 channel adapter 的 routing logic（custom code）决定。

2. **共享 agent group 的 per-thread sessions。** 多个 sessions 共享同一个 agent group（filesystem、skills、`CLAUDE.md`），但每个 session 有自己的 session DB。Worker pools 的标准做法。

3. **Session reset and replay。** 为同一个 thread 创建新 session。将旧 messages 标记为 unhandled，让 poll 再次 pick。旧 output 仍在平台上可见（例如 Discord thread），便于比较。这是 agent 可以请求的 action，不是自动行为。

4. **Cross-session read access。** 某些 agents 可以查询其他 sessions 的数据。不同访问级别：manager 查看 `messages_in`/`messages_out`（review content）。Supervisor 查看完整 internals（agent logs、tool calls、debug traces）。这只是 filesystem/DB access：mount 或 query 正确路径即可。

5. **Context duplication into new sessions。** 当 supervisor 在 worker 的 thread 中被调用时，会创建一个新 session，并复制 relevant messages。Custom host-side code 处理这一点。

6. **Agent-initiated host actions。** Agent 使用 MCP tools（reset session、update skills 等）。Agent-runner 处理 tool call，并写入结构化 `system` `messages_out` 行。Host 读取并带权限检查执行。Agent 可以请求，但 host 决定。

### 示例：PR Factory

三个 agent groups，一个 Discord channel（PR Factory），再加一个 admin channel：

| Role | Agent Group | Where | Session model |
|------|-------------|-------|---------------|
| **Worker** | pr-worker | PR Factory threads | 每个 thread 一个 session（每个 PR 一个） |
| **Manager** | pr-manager | PR Factory channel | 单个 session，可跨 worker sessions 查询 |
| **Supervisor** | pr-admin | Admin channel + PR Factory（被 @tagged 时） | Admin channel 中的 main session；在 worker threads 中被调用时使用 per-thread session |

**Worker flow：** GitHub PR → Discord thread → worker agent review（triage、review、test plan）。每个 thread 从共享的 pr-worker group 获得一个 session。

**Feedback flow：** User 在 worker threads 中 @tag supervisor → custom routing 发送给 supervisor，并创建一个包含该 thread messages（duplicated）的新 session。Supervisor 将 feedback 收集到 filesystem。Worker 看不到 supervisor messages。

**Iteration flow：** User 在 admin channel 中与 supervisor 讨论 feedback → supervisor 建议 skill changes（以带 diff 的 rich card 展示）→ user approve → supervisor 通过 host action 应用 changes → supervisor 请求 session reset + replay → workers 用更新后的 skills 在相同 threads 中重新 review 同一批 PRs，但使用 fresh sessions → user 并排比较 reviews。

**Manager flow：** User 在 PR Factory main channel（不在 threads 中）与 manager 交谈。Manager 可以搜索所有 worker session DBs（`messages_in`/`messages_out`），回答 “how many PRs today?” 或 “what topics are trending?” 之类的问题。也可以请求 actions（close PR、re-open）。

**哪些是 custom code，哪些是 base architecture：**

| Capability | Base architecture | Custom code（PR Factory） |
|-----------|-------------------|-------------------------|
| Per-thread sessions | ✓ platformThreadId → session | |
| Shared agent group across sessions | ✓ Multiple sessions, one group | |
| Writing messages to session DB | ✓ Standard flow | |
| @mention routing to different agent | | ✓ Channel adapter routing logic |
| Context duplication into supervisor session | | ✓ Host-side hook on supervisor invocation |
| Session reset + replay | ✓ Primitives（new session、mark unhandled） | ✓ Supervisor action triggers it |
| Skill updates | ✓ Filesystem writes | ✓ Supervisor action applies changes |
| Cross-session queries | ✓ DB/filesystem access | ✓ Manager's tools know where to look |
| Rich card output | ✓ Structured output in messages_out | |

## Central DB Schema

Central DB 处理 routing 和 entity management。所有 content 与 execution state 都位于 per-session DBs。

```sql
-- Agent workspaces：folder、skills、CLAUDE.md、container config
CREATE TABLE agent_groups (
  id               TEXT PRIMARY KEY,
  name             TEXT NOT NULL,
  folder           TEXT NOT NULL UNIQUE,
  agent_provider   TEXT,              -- sessions 的默认 provider（null = system default）
  container_config TEXT,              -- JSON: { additionalMounts, timeout }
  created_at       TEXT NOT NULL
);

-- Platform groups/channels（WhatsApp group、Slack channel、Discord channel、email thread 等）
CREATE TABLE messaging_groups (
  id                     TEXT PRIMARY KEY,
  channel_type           TEXT NOT NULL,     -- 'whatsapp', 'slack', 'discord', 'telegram', 'email'
  platform_id            TEXT NOT NULL,     -- platform-specific ID（JID、channel ID 等）
  name                   TEXT,
  is_group               INTEGER DEFAULT 0,
  unknown_sender_policy  TEXT NOT NULL DEFAULT 'strict',  -- 'strict' | 'request_approval' | 'public'
  created_at             TEXT NOT NULL,
  UNIQUE(channel_type, platform_id)
);

-- Users（messaging platform identities，命名空间为 "<channel_type>:<handle>"）
CREATE TABLE users (
  id           TEXT PRIMARY KEY,   -- 例如 'telegram:123456', 'discord:1470...'
  kind         TEXT NOT NULL,      -- mirrors the channel_type prefix
  display_name TEXT,
  created_at   TEXT NOT NULL
);

-- Roles（owner 仅 global；admin 可以是 global 或 scoped 到某个 agent_group）
CREATE TABLE user_roles (
  user_id         TEXT NOT NULL REFERENCES users(id),
  role            TEXT NOT NULL,   -- 'owner' | 'admin'
  agent_group_id  TEXT REFERENCES agent_groups(id),  -- NULL 表示 global
  granted_by      TEXT,
  granted_at      TEXT NOT NULL,
  PRIMARY KEY (user_id, role, agent_group_id)
);
-- owner rows 必须有 agent_group_id = NULL（在 db/user-roles.ts 中强制）

-- Membership（显式非特权访问；admin/owner 隐含 membership）
CREATE TABLE agent_group_members (
  user_id         TEXT NOT NULL REFERENCES users(id),
  agent_group_id  TEXT NOT NULL REFERENCES agent_groups(id),
  added_by        TEXT,
  added_at        TEXT NOT NULL,
  PRIMARY KEY (user_id, agent_group_id)
);

-- DM resolution cache（避免每次 cold DM 都重新解析）
CREATE TABLE user_dms (
  user_id            TEXT NOT NULL REFERENCES users(id),
  channel_type       TEXT NOT NULL,
  messaging_group_id TEXT NOT NULL REFERENCES messaging_groups(id),
  resolved_at        TEXT NOT NULL,
  PRIMARY KEY (user_id, channel_type)
);

-- 哪些 agent groups 用什么规则处理哪些 messaging groups
CREATE TABLE messaging_group_agents (
  id                 TEXT PRIMARY KEY,
  messaging_group_id TEXT NOT NULL REFERENCES messaging_groups(id),
  agent_group_id     TEXT NOT NULL REFERENCES agent_groups(id),
  trigger_rules      TEXT,              -- JSON: { pattern, mentionOnly, excludeSenders, includeSenders }
  response_scope     TEXT DEFAULT 'all',    -- 'all' | 'triggered' | 'allowlisted'
  session_mode       TEXT DEFAULT 'shared', -- 'shared' | 'per-thread'
  priority           INTEGER DEFAULT 0,     -- 多个 agents 匹配时，数值越高越先检查
  created_at         TEXT NOT NULL,
  UNIQUE(messaging_group_id, agent_group_id)
);

-- Sessions：一个 folder = 一个 session = 运行时一个 container
-- Folder path 派生为：sessions/{agent_group_id}/{session_id}/
CREATE TABLE sessions (
  id                 TEXT PRIMARY KEY,
  agent_group_id     TEXT NOT NULL REFERENCES agent_groups(id),
  messaging_group_id TEXT REFERENCES messaging_groups(id),  -- internal/spawned sessions 时为 null
  thread_id          TEXT,              -- platform thread ID（shared session mode 时为 null）
  agent_provider     TEXT,              -- per session override（null = inherit from agent_group）
  status             TEXT DEFAULT 'active',    -- 'active' | 'closed'
  container_status   TEXT DEFAULT 'stopped',   -- 'running' | 'idle' | 'stopped'
  last_active        TEXT,              -- last message activity timestamp
  created_at         TEXT NOT NULL
);
CREATE INDEX idx_sessions_agent_group ON sessions(agent_group_id);
CREATE INDEX idx_sessions_lookup ON sessions(messaging_group_id, thread_id);

-- Pending interactive questions（等待用户响应的 cards）
-- Host 在投递 question card 时写入，在收到 response 时删除
CREATE TABLE pending_questions (
  question_id    TEXT PRIMARY KEY,
  session_id     TEXT NOT NULL REFERENCES sessions(id),
  message_out_id TEXT NOT NULL,     -- 发送 card 的 messages_out row
  platform_id    TEXT,              -- card 被投递到的位置
  channel_type   TEXT,
  thread_id      TEXT,
  created_at     TEXT NOT NULL
);
```

### Pending Question Flow

当 host 投递带 `operation: 'ask_question'` 的 `messages_out` 行时：
1. Host 通过 channel adapter 投递 card
2. Host 写入一行 `pending_questions`，映射 `question_id` → `session_id`

当 Chat SDK `ActionEvent`（button click）到达时：
1. Bridge 从 event 中提取 `actionId`
2. Host 按 `question_id` 查找 `pending_questions`（从 actionId 派生；bridge 维护映射）
3. Host 找到目标 session，并写入带 `questionId` + `selectedOption` 的 `messages_in` 行
4. Host 删除 `pending_questions` 行
5. Agent-runner pick up 该 `messages_in` 行，匹配到 pending tool call，并返回 selection

这样避免扫描 session DBs。Central DB 是 routing lookup，与 message routing 使用同一 pattern。

Host-generated approval cards 也使用它：host 向 admin 的 DM 发送 approval request 时，会写入一行 `pending_questions`。Admin 的响应被 route 回原始 session。

### Container lifecycle states

```
stopped → running → idle → stopped
                  ↗
            idle → running（warm 时收到新消息）
```

- **stopped**：没有 container。每 60 秒 sweep 一次，检查 due scheduled messages。
- **running**：正在处理。每 1 秒 poll `messages_out`。
- **idle**：处理完成，container 仍保持 warm（最长 30 分钟 timeout）。每 1 秒 poll，因此新 messages 可以快速被 picked up。
- Idle timeout 后 → host kill container → stopped。

## Agent-Runner 架构

Agent-runner 是 container 内的进程。它在 session DB 和 Claude SDK 之间居中协调：轮询 work、为 agent 格式化 messages、将 tool calls 转换为 DB rows，并管理 agent lifecycle。

### IO Model

所有 IO 都通过 session DB。没有 stdin、没有 stdout markers、没有 IPC files。

- Initial input 和 follow-ups：poll `messages_in`
- Output：写入 `messages_out` rows
- MCP tools：写入 DB rows（没有 IPC files）
- Shutdown：host 在 idle timeout 时 kill container，或 agent-runner 在没有 pending work 时退出

### Poll Loop

1. 查询 `messages_in WHERE status = 'pending' AND (process_after IS NULL OR process_after <= now())`
2. 如果找到 rows：将每行设置为 `status = 'processing'`，`status_changed = now()`
3. 将 messages batch 成一个 prompt（剥离 routing fields，按 kind 格式化）
4. 推入 Claude SDK 的 MessageStream
5. 处理 agent output → 写入 `messages_out` rows
6. 将已处理 messages 设置为 `status = 'completed'`
7. 回到步骤 1。如果没有 messages，短暂 sleep 后重新 poll（container 在 idle timeout 期间保持 warm）

### Message Formatting by Kind

Agent-runner 在 formatting 前剥离 routing fields（`platform_id`、`channel_type`、`thread_id`）。Agent 永远看不到 routing info，只看到 content。

- **`chat`**：格式化为 `<messages>` XML block
- **`chat-sdk`**：从 serialized message 中提取 text、author、attachments；格式化为 `<messages>` XML
- **`task`**：格式化为 `[SCHEDULED TASK]` prefix + prompt。如有 pre-script，则先运行。
- **`webhook`**：格式化为 `[WEBHOOK: source/event]` + JSON payload
- **`system`**：host action results（例如 “register_group succeeded”）。格式化为 system context，而不是 chat。

Mixed batches（例如 chat message + system result 同时 pending）会合并成一个 prompt，并使用清晰 delimiters。

### MCP Tools

MCP tools 直接写入 session DB。

**Core tools：**

| Tool | 作用 |
|------|------|
| `send_message` | 写入 `messages_out` 行，`kind: 'chat'` |
| `send_file` | 将文件移动到 `outbox/{msg_id}/`，并写入带 filenames 的 `messages_out` |
| `schedule_task` | 写入 `messages_in` 行（发给自身），带 `process_after` + `recurrence`。或者为 outbound reminders 写入带 `deliver_after` 的 `messages_out`。 |
| `list_tasks` | 查询 `messages_in WHERE recurrence IS NOT NULL` |
| `pause_task` / `resume_task` / `cancel_task` | 修改 `messages_in` rows（更新 status，清除/设置 recurrence） |
| `register_agent_group` | 写入 `messages_out`，`kind: 'system'`，`action: 'register_agent_group'` |

**New tools：**

| Tool | 作用 |
|------|------|
| `ask_user_question` | 写入带 question card 的 `messages_out`。保持 tool call open，轮询 `messages_in` 查找匹配 `questionId` 的 response。将 selection 作为 tool result 返回。 |
| `edit_message` | 写入带 `operation: 'edit'` 的 `messages_out` |
| `add_reaction` | 写入带 `operation: 'reaction'` 的 `messages_out` |
| `send_to_agent` | 写入 `messages_out`，其中 `channel_type: 'agent'`，`platform_id: '{target}'` |
| `send_card` | 写入带 card structure 的 `messages_out` |

完整 MCP tool parameter definitions 见 [agent-runner-details.md](agent-runner-details.md)。

### Cards

**Agent-initiated（outbound）：** 基于 tool。Agent 调用 `ask_user_question`（带 options 的 interactive card）或 `send_card`（structured card）。Agent-runner 将 card structure 写入 `messages_out`。Host/adapter 处理 platform-specific rendering（Slack Block Kit、Discord embeds、Telegram inline keyboard、text fallback）。

**Host-initiated（approval cards）：** 当 action 需要 approval 时，host 生成标准化 approval card，并发送到 admin 的 DM。这不是 agent-initiated；agent 不知道 approval step。Card format 固定（action description + approve/deny buttons）。

**Inbound（card responses）：** 不是 card，而是 content 中带 `questionId` + `selectedOption` 的 `messages_in` 行。Agent-runner 匹配 pending `ask_user_question` tool call，并将 selection 作为 tool result 返回。

### Commands

以 `/` 开头的 messages 会按三份列表检查：

**Whitelisted commands（pass-through to agent）：**
- Agent provider 原生处理的标准 slash commands（例如 Claude 的 built-in commands）
- 原样传递，不包裹 `<messages>` XML

**Admin-only commands（require admin sender）：**
- `/remote-control`：remote control session
- `/clear`：clear session context
- `/compact`：force context compaction
- 如果由非 admin user 发送，command 会被拒绝并返回 error message。不会转发给 agent。

**Filtered commands（完全 drop）：**
- 在 NanoClaw context 中没有意义或可能造成问题的 commands
- 静默 drop：没有 error，不转发

Command lists 硬编码在 agent-runner 中。Admin verification 在 host-side 完成，消息进入 container 前已经检查：`src/command-gate.ts` 查询 `user_roles`（owner / global admin / scoped-admin-of-this-agent-group），然后通过、drop 或 route elsewhere。Container 没有 admin identity 概念：没有 env var、没有 DB query、没有 per-message check。

### Recurring Tasks

Agent-runner 像处理其他 `messages_in` 行一样处理 recurring task messages。Agent-runner 将 recurring message 标记为 `completed` 后，**host** 负责插入下一次 occurrence（新的 `messages_in` 行，`process_after` 推进到下一次 cron time）。Agent-runner 不管理 recurrence，只处理它发现的内容。

Pre-scripts：如果 task message 有 `script` field，先运行它。如果 `wakeAgent = false`，则不调用 Claude，直接标记 completed。

### Agent-to-Agent Messaging

**Outbound：** Agent 调用 `send_to_agent` tool → agent-runner 写入 `messages_out`，其中 `channel_type: 'agent'`，`platform_id` = target agent group ID。Host 验证权限，并写入目标 session 的 `messages_in`。

**Inbound：** 来自其他 agents 的 messages 作为普通 `chat` `messages_in` 行到达。Content 包含 `sender` 和 `senderId`（例如 `"senderId": "agent:pr-admin"`）。没有特殊 formatting，agent 会把它看作 chat message。

### Agent-Runner Properties

- AgentProvider interface 包装 SDK-specific query logic（trunk 提供 `claude` provider；OpenCode 等额外 providers 通过 `/add-<provider>` skills 安装）
- 通过 provider-specific mechanisms 实现 session resume
- 从 `CLAUDE.md` files 加载 system prompt
- Transcript archiving 的 PreCompact hook（Claude provider）
- 执行 task-kind messages 的 scripts

## Open Questions

- **Approval routing**：host 如何找到 admin 的 DM conversation？如果没有 DM channel 怎么办？Approval list 是按 agent group 配置，还是 global？
- **MCP server lifecycle**：MCP server process 会在同一个 container 的多次 queries 之间持久存在，还是每次重启？
- **Container startup config**：container launch 时除 env vars 外还传入什么 config（如果有）？Session DB 位于固定 mount path。System prompt 来自 `CLAUDE.md`。Provider name 来自 env。还有什么？
- **Idle detection with pending questions**：当 `ask_user_question` 正等待响应时，container 不应被视为 idle。还需要检测 agent 是否仍在工作（active tool calls、subagents），即使近期没有写入 `messages_out` 也要避免 kill container。

## 相关文档

- **[api-details.md](api-details.md)**：Channel adapter interface（NanoClaw + Chat SDK bridge）、message content examples、host delivery logic
- **[agent-runner-details.md](agent-runner-details.md)**：AgentProvider interface、MCP tools、message formatting、media handling、provider implementations
