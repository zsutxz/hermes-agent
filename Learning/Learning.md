# Hermes 架构深度解读

本文按真实代码路径分析 Hermes（v0.13.0, Nous Research）的完整架构，覆盖从请求到响应的全链路、核心类层次、插件系统、压缩管线、内存子系统、MCP 集成和容错模式。

三条入口（CLI / TUI / Gateway）最终汇聚到同一个核心：`run_agent.py` → `AIAgent.run_conversation()`。

---

## 导航目录

### 上篇：请求到响应流程分析

| 章节 | 主题 | 关键词 |
|------|------|--------|
| [一、整体分层](#一先看整体分层) | 6 层架构概览 | 入口层 → 会话层 → 上下文层 → Agent loop → 持久化层 → 回传层 |
| [二、工具装配](#二工具与能力是如何在请求开始前装配好的) | tool discovery / toolset / AIAgent.\_\_init\_\_ | `ToolRegistry`, `resolve_toolset()`, `get_tool_definitions()` |
| [三、CLI 流程](#三入口一经典-cli-流程) | HermesCLI.chat → \_init\_agent → run\_conversation | `cli.py:9144`, interrupt 队列, callback 绑定 |
| [四、TUI 流程](#四入口二tui-流程) | JSON-RPC 双进程模型 | `session.create`, `prompt.submit`, `message.delta` |
| [五、Gateway 流程](#五入口三gateway-平台消息流程) | 平台 adapter → MessageEvent → GatewayRunner | 鉴权, busy session, hygiene compression, transcript |
| [六、run\_conversation()](#六统一核心aiagentrun_conversation) | 主循环状态机 | system prompt 缓存, API messages 标准化, tool loop, 持久化 |
| [七、持久化层](#七持久化层sessiondb-和-sessionstore) | SessionDB + SessionStore | SQLite WAL, FTS5, session\_key vs session\_id |
| [八、三条入口对比](#八三条入口的共同点与差异) | CLI / TUI / Gateway 异同 | 单进程 vs 双进程 vs 平台适配 |
| [九、完整请求串讲](#九按时间顺序串一次完整请求) | Gateway 普通消息 17 步全链路 | adapter → agent → result → 回传 |
| [十、函数索引](#十最关键的函数索引) | 按模块分类的关键函数定位 | 入口与分发, session, agent 核心, tool |
| [十一、设计观察](#十一个设计上的关键观察) | 4 个核心洞察 | 可恢复工具循环, session\_id 变化, 持久化即主流程 |
| [十二、结论](#十二结论) | 一句话总结 | 同步 agent runtime + 强会话管理 |

### 下篇：架构审计（Agent Architecture Audit）

| 章节 | 主题 | 关键词 |
|------|------|--------|
| [十三、类层次与继承](#十三核心类层次与继承关系) | 类图 + AIAgent 组合关系 | `ContextEngine`, `MemoryProvider`, `ToolRegistry`, 组合优于继承 |
| [十四、插件系统](#十四插件系统架构) | 四级发现 + 七种插件类型 + Hook | `PluginManager`, `PluginLlm` 门面, `HookRegistry` |
| [十五、压缩管线](#十五上下文压缩管线) | 双压缩器 + 三触发点 + 副作用 | `ContextCompressor`, `TrajectoryCompressor`, `focus_topic` |
| [十六、内存子系统](#十六内存子系统设计) | 1+1 提供者 + 生命周期 + 容错 | `MemoryManager`, `MemoryStore`, 非阻塞容错 |
| [十七、MCP 集成](#十七mcp-集成层) | 三传输协议 + 线程安全 + Sampling | stdio / HTTP / SSE, 后台事件循环, 凭证脱敏 |
| [十八、错误与容错](#十八错误处理与容错模式) | Fallback 链 + 错误分类 + 溢出处理 | retryable vs non-retryable, context overflow, 插件隔离 |
| [十九、设计模式](#十九关键设计模式) | 10 个模式 + 同步/异步桥接 | Strategy, Registry, Observer, Facade, Snapshot, Sentinel |
| [二十、12 层栈审计](#二十12-层栈审计结果) | 按 12 层模型的风险评估 | 总体健康度: 良好 (Low-to-Medium Risk) |
| [二十一、接口速查](#二十一关键抽象接口速查) | 11 个核心 ABC/接口的文件和核心方法 | `ContextEngine`, `MemoryProvider`, `ToolRegistry` 等 |

---

## 一、先看整体分层

Hermes 的消息处理大体可拆成 6 层：

1. **入口层**
   CLI 读取终端输入；TUI 读取 JSON-RPC `prompt.submit`；Gateway adapter 从 Telegram/Discord/Slack/Webhook 收到平台消息。

2. **会话层**
   把消息归一化为 Hermes 自己的 session / history / session_key / session_id。

3. **上下文构建层**
   组装 system prompt、历史消息、技能内容、memory、reply context、图片/语音转文本、`@file` 上下文等。

4. **Agent 主循环**
   `AIAgent.run_conversation()` 调模型；如果模型发出 tool calls，就执行工具并把结果继续塞回消息序列，再次调模型，直到得到最终回答。

5. **持久化层**
   通过 `SessionDB` / `SessionStore` 把 session、message、tool_call、reasoning、token usage 等写入 SQLite 和兼容性的 JSON/JSONL。

6. **回传层**
   CLI 直接打印；TUI 发 `message.delta` / `message.complete` 事件；Gateway 经 adapter 发回 Telegram/Discord/Slack 等。

理解 Hermes 的关键是：

- **入口不一样，核心 loop 一样**
- **session 管理不一样，消息格式最终都统一成 OpenAI 风格的 `messages`**
- **真正“做事”的地方是 `AIAgent.run_conversation()`**

---

## 二、工具与能力是如何在请求开始前装配好的

在看消息流程前，要先看 agent 初始化时做了什么，因为这决定了本次消息“模型能调用哪些工具、用什么 system prompt、会不会自动压缩上下文、有没有 memory”。

### 2.1 tool discovery 和 toolset 解析

核心在：

- `model_tools.py`
- `toolsets.py`
- `tools/registry.py`

`model_tools.py` 顶部导入时就会：

- 调 `discover_builtin_tools()`，扫描 `tools/*.py`
- 每个 tool 文件在 import 时通过 `registry.register(...)` 注册 schema、handler、check_fn
- 再尝试 `discover_plugins()`，把插件提供的工具也加入 registry

关键函数：

- `model_tools.get_tool_definitions(...)`
- `toolsets.resolve_toolset(...)`
- `registry.get_definitions(...)`

流程是：

1. 根据 `enabled_toolsets` / `disabled_toolsets` 决定本次暴露哪些工具
2. 把 toolset 展开成真实工具名集合
3. `registry.get_definitions(...)` 再根据每个工具的 `check_fn` 和环境条件过滤不可用工具
4. 生成最终给模型看的 tool schema 列表

注意：

- `toolsets.py` 中 `_HERMES_CORE_TOOLS` 是默认核心工具集合
- `TOOLSETS` 则定义了逻辑分组，如 `web`、`file`、`browser`、`memory`、`delegation`
- 一个工具就算注册成功，如果没有被某个启用的 toolset 引入，也不会暴露给模型

### 2.2 `AIAgent.__init__()` 做的初始化

关键类：

- `run_agent.py` -> `class AIAgent`

关键初始化逻辑：

- `self.tools = get_tool_definitions(...)`
- `self.valid_tool_names = {...}`
- 初始化 `SessionDB` 相关状态
- 初始化 `TodoStore`
- 初始化内建 memory store：`tools.memory_tool.MemoryStore`
- 初始化 memory provider plugin：`agent.memory_manager.MemoryManager`
- 初始化 context engine / context compressor
- 缓存 `_cached_system_prompt`

对应代码段在 `run_agent.py` 约 1613 行之后。

这一步的重要影响：

1. 本次会话“模型能看到哪些工具”在这里确定
2. 外部 memory provider 和 context engine 也会把自己的 tool schema 注入进来
3. 上下文压缩器 `ContextCompressor` 在这里就准备好了，后续每个 turn 都会用它判断是否需要压缩

---

## 三、入口一：经典 CLI 流程

入口类：

- `cli.py` -> `class HermesCLI`

### 3.1 用户输入进入 CLI

经典 CLI 下，用户在终端输入消息，最终会进入：

- `HermesCLI.chat(self, message, images=None)`

关键位置：

- `cli.py:9144`

这个函数是 CLI 的一层 orchestration，不是真正的模型循环。

它先做几件事：

1. 刷新运行时 credentials：`_ensure_runtime_credentials()`
2. 根据本轮消息决定是否需要重新建 agent：`_resolve_turn_agent_config()`
3. 如果 agent 还没建，调用 `_init_agent(...)`
4. 处理图片输入：
   - vision model：直接构造成多模态 content parts
   - 非 vision model：先走 vision 分析，转成文字
5. 处理 `@file` / `@diff` / `@folder` 这类 context references
6. 对用户消息做 surrogate 清理
7. 先把用户消息加入 `self.conversation_history`

### 3.2 `_init_agent()` 如何恢复会话

关键函数：

- `cli.py:3565` -> `HermesCLI._init_agent(...)`

它负责：

1. 创建 `SessionDB`
2. 如果是 `--resume` 场景，从 SQLite 取 session metadata 和历史消息
3. 如果 resume 的 session 因为压缩被 fork 过，会调用：
   - `SessionDB.resolve_resume_session_id(...)`
   找到真正持有消息的 descendant session
4. 用这些参数实例化 `AIAgent(...)`

`AIAgent` 被创建时，CLI 会把大量 callback 传进去，例如：

- `clarify_callback`
- `reasoning_callback`
- `tool_progress_callback`
- `tool_start_callback`
- `tool_complete_callback`
- `stream_delta_callback`

所以 CLI 的 spinner、thinking、inline diff、approval prompt，本质上不是 agent 自己“会显示”，而是 agent 在关键事件点回调 CLI。

### 3.3 CLI 如何真正调用主循环

仍在 `HermesCLI.chat(...)` 中，真正执行时会开一个后台线程：

- 线程里调用 `self.agent.run_conversation(...)`

调用大致是：

- `user_message=agent_message`
- `conversation_history=self.conversation_history[:-1]`
- `stream_callback=stream_callback`
- `task_id=self.session_id`
- `persist_user_message=message if _voice_prefix else None`

这里有个细节很重要：

- `conversation_history` 传的是“之前的历史”
- 当前这一轮用户消息已经先被 append 到 `self.conversation_history`
- 但传给 agent 时是 `[:-1]`，因为当前消息会由 `run_conversation()` 自己插入

### 3.4 CLI 中断机制

CLI 不是简单同步阻塞等待。

`HermesCLI.chat(...)` 在 agent 线程运行期间，会循环监听 `_interrupt_queue`：

- 如果用户又发新消息，会调用 `self.agent.interrupt(interrupt_msg)`

这会让 `run_conversation()` 在下一次检查中断位时退出 tool loop，或尽量中断正在执行的工具。

### 3.5 CLI 输出如何回到用户

CLI 最终拿到 `result = self.agent.run_conversation(...)` 后：

1. 更新 `self.conversation_history = result["messages"]`
2. 如果压缩导致 session_id 变更，同步 `self.session_id = self.agent.session_id`
3. 展示 final response
4. 可选自动生成 title：`agent.title_generator.maybe_auto_title(...)`

因此 CLI 的显示层和 agent 状态是紧耦合的，但真正消息语义还是由 `result["messages"]` 这个统一消息列表决定。

---

## 四、入口二：TUI 流程

TUI 是两进程模型：

- 前端：Node/Ink
- 后端：Python `tui_gateway/server.py`

前端不会直接调用 `AIAgent`，而是通过 JSON-RPC。

### 4.1 TUI session 创建

关键方法：

- `tui_gateway/server.py:2023` -> `@method("session.create")`

流程：

1. 为 TUI session 生成一个短 `sid`
2. 再生成 Hermes session key：`_new_session_key()`
3. 在 `_sessions[sid]` 中保存：
   - `history`
   - `history_lock`
   - `running`
   - `session_key`
   - `transport`
   - `slash_worker`
   - UI 相关状态
4. 用 timer 延迟触发 `_start_agent_build(...)`

这里的设计关键点是：

- `session.create` 立即返回，保证 UI 先显示出来
- `AIAgent` 的真实构建是延迟的，不阻塞首屏

### 4.2 `_make_agent()` 如何构造 TUI agent

关键函数：

- `tui_gateway/server.py:1821` -> `_make_agent(...)`

它本质也是在构造 `AIAgent(...)`，但参数是 TUI 版本：

- `platform="tui"`
- `session_id=session_id or key`
- `session_db=_get_db()`
- `enabled_toolsets=_load_enabled_toolsets()`
- `ephemeral_system_prompt=system_prompt`
- 再通过 `**_agent_cbs(sid)` 传入一组 TUI 专用 callback

所以 TUI 与 CLI 的共同点是：

- 都是本地直接 new `AIAgent`

不同点是：

- CLI 把显示输出送到 prompt_toolkit / Rich
- TUI 把所有事件变成 JSON-RPC event 发给 Ink

### 4.3 用户提交消息：`prompt.submit`

关键方法：

- `tui_gateway/server.py:2929` -> `@method("prompt.submit")`

流程：

1. 根据 `session_id` 取出 TUI session
2. 上锁检查 `running`
3. 如果 agent 还没准备好，先 `_start_agent_build(...)`
4. 新开线程执行 `_run_prompt_submit(...)`

### 4.4 `_run_prompt_submit()` 的主要工作

关键函数：

- `tui_gateway/server.py:2963`

它做的事跟 CLI 类似，但输出形态完全事件化：

1. 复制当前 `history`
2. 复制附带图片列表
3. 发送 `message.start` 事件
4. 处理 `@` 上下文引用
5. 处理图片路由：
   - native 多模态
   - text vision enrichment
6. 定义 `_stream(delta)`，每到一个 token/delta 就 emit：
   - `message.delta`
7. 调用：
   - `agent.run_conversation(run_message, conversation_history=list(history), stream_callback=_stream)`
8. 结果返回后，更新 `session["history"] = result["messages"]`
9. 发 `message.complete`

### 4.5 TUI 与 CLI 的核心区别

CLI 和 TUI 走的是同一个 `AIAgent.run_conversation()`，但 TUI 额外多了一个 RPC transport 层：

1. 前端发 `prompt.submit`
2. 后端线程跑 agent
3. agent 的 delta / reasoning / tool 事件通过 callback 转成：
   - `message.delta`
   - `message.complete`
   - `tool.start`
   - `approval.request`
   - `review.summary`
4. Ink 再把这些事件渲染成界面

因此 TUI 不是“另一套 agent”，只是“另一套 I/O 管道”。

---

## 五、入口三：Gateway 平台消息流程

这是最复杂的一条，因为它多了：

- 平台 adapter
- 鉴权
- 会话分流
- busy session / interrupt / queue
- transcript load/save
- 平台级 typing / message edit / ephemeral reply / media send

### 5.1 平台 adapter 先把消息归一化成 `MessageEvent`

统一数据结构定义在：

- `gateway/platforms/base.py:870` -> `class MessageEvent`

里面包含：

- `text`
- `message_type`
- `source: SessionSource`
- `message_id`
- `media_urls`
- `reply_to_message_id`
- `reply_to_text`
- `auto_skill`
- `channel_prompt`
- `internal`

这意味着：

- Telegram、Discord、Slack、Webhook 的原始消息对象都不会直接传给 agent
- 它们先被平台 adapter 转成 Hermes 自己的标准事件模型

典型接入方式：

- `GatewayRunner.start()` 时对每个 adapter 调：
  - `adapter.set_message_handler(self._handle_message)`
- 见 `gateway/run.py:2934`

然后 adapter 在各自平台收到消息后，最终都会：

- `await self._message_handler(event)`

例如：

- `gateway/platforms/telegram.py` 用 `add_handler(...)` 注册文本/命令/媒体回调
- `gateway/platforms/discord.py:_handle_message(...)` 负责从 Discord message 构造/清理内容
- `gateway/platforms/base.py` 的公共处理逻辑最终 `await self._message_handler(event)`

### 5.2 GatewayRunner 的总入口：`_handle_message`

关键函数：

- `gateway/run.py:4645` -> `GatewayRunner._handle_message(event)`

这是 gateway 消息总调度器。

它的前半段主要是“分流”和“拦截”，还没到 agent：

1. `pre_gateway_dispatch` plugin hook
2. 鉴权：`_is_user_authorized(source)`
3. pairing code 逻辑
4. `/update`、`/reload-mcp` 等 pending prompt 的拦截
5. 如果本 session 已有运行中的 agent：
   - `/status`、`/restart`、`/stop`、`/new` 等命令优先处理
   - `/queue` 会排队而不打断
   - 普通消息可能触发 interrupt / queue / busy session 分支
6. skill slash command 预处理
7. unknown slash command 拦截

这一段的重要性在于：

- 很多平台消息根本不会进入 agent
- command、approval、busy-session、pairing 都是在 gateway 层消化掉的

### 5.3 `_running_agents`：gateway 的并发保护

在真正进入 agent 前，gateway 会先给当前 session 立一个 sentinel：

- `self._running_agents[_quick_key] = _AGENT_PENDING_SENTINEL`

然后生成：

- `_run_generation = self._begin_session_run_generation(_quick_key)`

这一步的目的：

1. 防止同一 session 同时有两个请求并发跑
2. 防止在异步 `await` 期间第二条消息“溜进去”再建一个新 agent
3. 后续如果 session 被 reset / interrupt / stale eviction，可以用 generation 判断旧结果是否该丢弃

### 5.4 真正进入 gateway agent 处理：`_handle_message_with_agent`

关键函数：

- `gateway/run.py:5837`

这是 gateway 的“普通消息 -> agent 结果”主流程。

#### 步骤 1：取或创建 session

调用：

- `self.session_store.get_or_create_session(source)`

关键类型：

- `gateway/session.py` -> `SessionStore`
- `hermes_state.py` -> `SessionDB`

`SessionStore.get_or_create_session(...)` 做的事：

1. 用 `SessionSource` 算出 `session_key`
2. 查看这个 key 当前是否已有活动会话
3. 根据 reset policy 判断是否需要自动 reset
4. 如需新 session，则生成新的 `session_id`
5. 把 session 记录写入 SQLite：
   - `SessionDB.create_session(...)`

所以：

- `session_key` 是“逻辑会话 key”，例如某个 DM / thread / user lane
- `session_id` 是“具体一次会话实例”

前者用于定位“当前聊天槽位”，后者用于定位“这一轮会话的 transcript 和统计”

#### 步骤 2：构建 session context

调用：

- `build_session_context(source, self.config, session_entry)`
- `build_session_context_prompt(context, redact_pii=...)`

这会把平台、用户、chat、thread、home channel、shared session 等信息转成一段上下文提示。

如果 session 是自动 reset 出来的，还会在 context prompt 前额外插入 system note，告诉 agent：

- 上一会话因 idle / daily / suspended 被重置
- 这是一段全新对话

#### 步骤 3：自动加载技能、历史 transcript、预清洗上下文

主要包括：

1. 新 session 时根据 channel/topic binding 自动加载 skill
2. `history = self.session_store.load_transcript(session_entry.session_id)`
3. 预检查 transcript 是否过大

`load_transcript(...)` 很关键：

- 优先从 SQLite `SessionDB.get_messages_as_conversation(...)` 取
- 同时读 legacy JSONL
- 谁消息更多用谁，避免迁移期间历史丢失

#### 步骤 4：session hygiene / 自动压缩

在真正调 agent 前，gateway 会先做一次“卫生压缩”：

- 位于 `gateway/run.py` 约 6039 行后

逻辑大致是：

1. 根据最近一次真实 `prompt_tokens` 或 rough token estimate 判断 transcript 是否太大
2. 如果超过阈值，临时创建一个轻量 `AIAgent`
3. 调它的 `_compress_context(...)`
4. 压缩后可能生成新的 `session_id`
5. 用 `SessionStore.rewrite_transcript(...)` 把压缩后的消息写入新 session
6. evict cached agent，下次重建

这一步和 CLI/TUI 的差异很大：

- CLI/TUI 主要是在 agent loop 内部感知压缩
- gateway 会在“进主循环前”就先抢救一遍超长 transcript

#### 步骤 5：准备最终 `message_text`

调用：

- `_prepare_inbound_message_text(...)`

它会统一处理：

1. shared multi-user session 时，给消息前缀 `[username]`
2. 图片：
   - native multimodal：暂存到 per-session buffer
   - 否则调用 vision，先转文本
3. 音频：
   - 先跑 STT，把转录文本并入消息
4. document：
   - 给出文件说明或直接注入文本
5. reply-to：
   - 注入 `[Replying to: "..."]`
6. `@file` / `@folder`：
   - 异步展开

到这一步，gateway 入口层才真正拿到了本轮准备送进 agent 的用户文本。

### 5.5 gateway 如何调用 agent：`_run_agent`

关键函数：

- `gateway/run.py:12695`

这不是 `AIAgent.run_conversation()` 本体，而是 gateway 的一层包装器，负责：

1. 读取 gateway config
2. 决定启用哪些 toolsets
3. 决定 tool progress 的展示级别
4. 构造 progress callback / step callback / stream consumer
5. 构造或复用当前 session 对应的 `AIAgent`
6. 在线程池里调用 `agent.run_conversation(...)`
7. 同步 interrupt、queue、streaming、progress message edit 等平台行为

这个包装层的重要工作是把 agent 的同步 loop 转成“适合聊天平台”的异步体验。

比如：

- tool progress 通过编辑消息方式显示
- typing indicator 在后台持续刷新
- interrupt 通过 adapter pending queue 监控
- 如果用户在 agent 运行过程中又发了一条消息，gateway 会把它作为 pending_event 或 interrupt_message，当前轮结束后递归调用 `_run_agent(...)` 继续处理

### 5.6 gateway 得到 agent 结果后的落库与回传

回到 `_handle_message_with_agent(...)` 后半段。

拿到 `agent_result` 后，gateway 会做：

1. 停止 typing indicator
2. 丢弃 stale generation 的结果
3. 提取：
   - `final_response`
   - `messages`
   - `api_calls`
   - `last_reasoning`
4. 如配置允许，把 reasoning prepend 给 response
5. 追加 runtime footer
6. 分类失败：
   - context overflow failure：不写 transcript，避免坏 session 越写越大
   - transient failure：只写 user message，保留上下文
7. 如果压缩耗尽：
   - `session_store.reset_session(session_key)`
   - evict cached agent
8. 把本轮新增消息写回 transcript

注意 transcript 写入分两种：

- agent 已经通过自己的 `SessionDB` 路径写过 SQLite
- gateway 这里只补 JSONL，避免 duplicate-write

对应逻辑：

- `session_store.append_to_transcript(..., skip_db=agent_persisted)`

最后：

- 如果平台支持 streaming，可能 response body 早就已经流式发出，此时只需补媒体或 footer
- 否则 `_handle_message_with_agent(...)` 返回 response string，adapter 再统一发送

---

## 六、统一核心：`AIAgent.run_conversation()`

前面三条入口都只是“送消息进来”的方式不同。真正的 Hermes agent loop 在：

- `run_agent.py:10569` -> `AIAgent.run_conversation(...)`

这是整个系统最关键的函数。

可以把它理解成一个“同步状态机”：

1. 准备本轮消息、system prompt、memory、上下文
2. 发起模型调用
3. 如果模型要调用工具，则执行工具、把工具结果 append 回消息序列
4. 回到第 2 步
5. 如果模型给出最终回复，则结束

### 6.1 进入 `run_conversation()` 的前置准备

函数一开始做的事情非常多，主要是为了把这轮 conversation 放到一个可恢复、可中断、可持久化的状态里。

关键步骤：

1. `_ensure_db_session()`
   - 首次需要时为该 `session_id` 在 SQLite 中建 `sessions` 记录

2. `set_session_context(self.session_id)`
   - 给日志打上 session 上下文

3. 恢复 primary runtime / fallback runtime 状态

4. 清洗用户输入和 `persist_user_message`

5. 生成本轮 `effective_task_id`

6. 重置本轮重试计数器、tool guardrail 状态

7. 初始化 `messages = list(conversation_history) if conversation_history else []`

8. 如果有历史但 todo store 为空，从历史中 hydrate todo 状态

9. 记录 `original_user_message`

10. 如果 `_cached_system_prompt` 为空，则构建 system prompt
    - 否则复用缓存
    - 并在必要时写入 `SessionDB.update_system_prompt(...)`

11. 预压缩检查：
    - 如果当前 messages 估算 token 太大，在进入 loop 前就先压缩

12. memory provider：
    - `on_turn_start(...)`
    - `prefetch_all(...)`

最终这一步的核心产物有三个：

1. `messages`
2. `active_system_prompt`
3. 当前 turn 的 clean user message / persist message 语义

### 6.2 system prompt 是怎么来的

`AIAgent` 不会每次都重建 system prompt。

关键变量：

- `self._cached_system_prompt`

构建逻辑大致在 `_build_system_prompt(...)`。

system prompt 里会汇入：

1. Hermes 自身身份与行为规则
2. `SOUL.md`、`AGENTS.md`、`.cursorrules` 等上下文文件
3. memory summary / memory provider system block
4. skills 相关指示
5. platform hints
6. 当前可用 tool 的摘要信息

缓存的意义是：

- 多轮对话不必每次重建完整大 prompt
- 只有在压缩、session 切换、model 切换等时才失效

### 6.3 主循环本体

真正的循环在：

- `run_agent.py:10953`

```python
while (api_call_count < self.max_iterations and self.iteration_budget.remaining > 0) or self._budget_grace_call:
```

这不是简单的 `while True`，它受以下因素约束：

1. `max_iterations`
2. `iteration_budget`
3. `_budget_grace_call`
4. 用户 interrupt
5. tool guardrail
6. context compression

每轮 iteration 大体做以下事：

#### 第一步：检查 interrupt / budget / step callback

如果用户中断：

- `self._interrupt_requested == True`
- 直接 break，形成 interrupted result

然后更新：

- `api_call_count`
- `self._api_call_count`
- `iteration_budget.consume()`

并触发：

- `step_callback(api_call_count, prev_tools)`

Gateway 就靠这个 step callback 发 `agent:step` 之类的状态事件。

#### 第二步：构造本轮 API messages

这里会把内部 `messages` 转成真正送给模型的 `api_messages`。

关键工作：

1. 修复损坏的 tool_call 参数：`_sanitize_tool_call_arguments(...)`
2. 把 memory prefetch 和 plugin pre-llm context 注入“当前轮 user message”
3. 复制 reasoning 内容到 provider 需要的字段
4. 删除内部字段：
   - `reasoning`
   - `finish_reason`
   - `_thinking_prefill`
5. 对严格 provider 清理 `tool_calls` 中的兼容性字段
6. 插入 system message
7. 插入 `prefill_messages`
8. 应用 Anthropic prompt cache metadata
9. `_sanitize_api_messages(...)`
   - 删除 orphaned tool result
   - 补 tool result stub
10. `_drop_thinking_only_and_merge_users(...)`
    - 去掉只有 thinking 没有 visible content 的 assistant turn

这是 Hermes 很重要的一层：

- 内部持久化消息结构和实际 API payload 结构不是完全一致的
- Hermes 在这里做了大量 provider compatibility adaptation

#### 第三步：发模型调用

根据 `api_mode` 和 provider 走不同 transport。

但对主流程来说，可以抽象成：

1. 调模型
2. 得到 `assistant_message`
3. 解析 finish_reason、usage、reasoning、tool_calls

模型返回后，Hermes 会调用：

- `_build_assistant_message(assistant_message, finish_reason)`

这个函数非常关键。

### 6.4 `_build_assistant_message()`：把 provider 返回标准化

关键函数：

- `run_agent.py:8762`

它负责把 provider 的原始返回整理成 Hermes 内部消息格式。

主要做：

1. 提取 reasoning
   - 结构化 reasoning 字段
   - 或从 `<think>...</think>` 中提取

2. 清洗 content
   - surrogate sanitize
   - 去掉 `<think>` 标签本身，只把 reasoning 留在单独字段

3. 组装 assistant 消息：
   - `role`
   - `content`
   - `reasoning`
   - `reasoning_content`
   - `reasoning_details`
   - `finish_reason`

4. 标准化 tool_calls
   - 给每个 tool_call 确定稳定 id / call_id / response_item_id
   - 保存 `function.name` 与 `function.arguments`

这个标准化层的意义是：

- 不同 provider 返回结构不一致
- Hermes 需要一个统一内部格式，后续 persistence、resume、compression、tool replay 都依赖它

### 6.5 如果模型要求调工具，会发生什么

当 `assistant_message.tool_calls` 存在时，`run_conversation()` 不会结束，而是进入工具执行路径。

关键代码：

- `self._execute_tool_calls(...)`

在此之前会先：

1. 对 `delegate_task` 做数量上限控制：`_cap_delegate_task_calls(...)`
2. 做 tool call 去重：`_deduplicate_tool_calls(...)`
3. 先把 assistant message 自己 append 到 `messages`

为什么要先 append assistant tool-call message？

因为后续的每个 tool result 都必须能和一个先前的 assistant tool_call 对应上；否则很多 provider 在 replay 时会报错。

### 6.6 工具执行总入口：`_execute_tool_calls()`

关键函数：

- `run_agent.py:9427`

它只做一件事：

- 判断是并发执行还是串行执行

依据：

- `_should_parallelize_tool_batch(tool_calls)`

读类工具、彼此路径不冲突的文件工具，可以并发；否则串行。

#### 并发路径：`_execute_tool_calls_concurrent(...)`

关键职责：

1. 先解析每个 tool_call 的 arguments
2. 为文件修改 / destructive terminal command 建 checkpoint
3. 跑 plugin pre-tool hook
4. 跑 tool guardrail `before_call`
5. 通过线程池并发执行 `_invoke_tool(...)`
6. 保证结果按原 tool_call 顺序 append 回 `messages`

#### 串行路径：`_execute_tool_calls_sequential(...)`

职责类似，但逐个执行，并在每个工具之间：

- 允许 interrupt 更快生效
- 打印更细粒度 CLI status
- 在 tool result 后马上更新消息序列

### 6.7 单个工具调用如何落地：`_invoke_tool()` -> `handle_function_call()`

关键链路：

1. `AIAgent._invoke_tool(...)`
2. `model_tools.handle_function_call(...)`
3. `registry.dispatch(...)`
4. 真实工具 handler

`_invoke_tool(...)` 先处理几类 agent 内部工具：

- `todo`
- `session_search`
- `memory`
- memory provider tool
- `clarify`
- `delegate_task`
- context engine tool

其余普通工具才走：

- `handle_function_call(function_name, function_args, ...)`

`handle_function_call(...)` 做的事情：

1. `coerce_tool_args(...)`
   - 根据 schema 把字符串参数转回 int/bool/list/dict

2. 跑 plugin `pre_tool_call` hook，可直接 block

3. 记录 latency

4. 调 `registry.dispatch(...)`

5. 跑 plugin `post_tool_call` hook

6. 跑 plugin `transform_tool_result` hook

7. 返回 JSON string

也就是说：

- Hermes 规定 tool handler 的最终返回值是 JSON 字符串
- agent 并不直接理解 Python object，它只把工具结果当成下一轮模型输入中的 `tool` role content

### 6.8 tool result 如何回到对话里

无论并发还是串行，工具执行完后都会 append 类似消息到 `messages`：

- `role="tool"`
- `tool_call_id=<对应 tool call id>`
- `name=<tool name>`
- `content=<JSON string result>`

这样下一轮模型请求时，messages 中会是：

1. user
2. assistant(tool_calls=...)
3. tool(result for call 1)
4. tool(result for call 2)
5. ...

然后模型再根据这些 tool results 继续思考并回答。

### 6.9 什么时候会压缩上下文

Hermes 不只在 gateway 入口做 hygiene compression。

`AIAgent` 自己在 tool loop 内也会做动态压缩。

在 tool 执行后，会拿真实 `prompt_tokens` 或 rough estimate 评估当前上下文大小，若超阈值：

- 调 `_compress_context(messages, system_message, approx_tokens=...)`

压缩的影响不只是缩短消息列表，还可能：

1. 结束当前 `session_id`
2. 新建 continuation session
3. 更新 `self.session_id`
4. 重建 system prompt
5. 重置 DB flush cursor / history origin

因此调用方（CLI/TUI/gateway）都会在 turn 结束后检查：

- `agent.session_id` 是否变化

### 6.10 什么时候持久化

这是一个非常关键的点。

Hermes 不是“最终回答出来以后才一次性存库”，而是多次增量持久化。

关键函数：

- `_persist_session(messages, conversation_history=None)`
- `_flush_messages_to_session_db(...)`

对应位置：

- `run_agent.py:3786`

持久化内容包括：

1. session log JSON
2. SQLite `messages` 表

`_flush_messages_to_session_db(...)` 会利用：

- `self._last_flushed_db_idx`

只写“本次新增的消息”，避免重复写入。

写入实际由：

- `SessionDB.append_message(...)`

完成，字段包括：

- `role`
- `content`
- `tool_calls`
- `tool_call_id`
- `tool_name`
- `finish_reason`
- `reasoning`
- `reasoning_content`
- `reasoning_details`
- `codex_reasoning_items`
- `codex_message_items`

所以 Hermes 的会话恢复不是只恢复 user/assistant 文本，而是能恢复：

- assistant tool-call turn
- tool result
- reasoning 相关字段
- finish_reason

这也是为什么它能在 resume 后继续较稳定地 replay 工具链。

### 6.11 什么时候结束 loop

loop 结束的几类典型原因：

1. 模型返回无 tool_calls 的最终内容
2. 用户 interrupt
3. max_iterations / budget exhausted
4. tool guardrail halt
5. fatal / non-retryable error
6. compression exhausted

结束时 `run_conversation()` 会返回一个 result dict，通常含有：

- `final_response`
- `messages`
- `api_calls`
- `completed`
- `interrupted`
- `failed`
- `error`
- `last_reasoning`
- `last_prompt_tokens`
- `context_length`
- `session_id`
- `tools`

这个结构就是 CLI/TUI/gateway 三个入口统一消费的核心结果。

---

## 七、持久化层：`SessionDB` 和 `SessionStore`

Hermes 的存储是两层结构：

1. **底层事实存储**：`SessionDB`（SQLite）
2. **gateway 的逻辑会话映射层**：`SessionStore`

### 7.1 `SessionDB`

定义在：

- `hermes_state.py` -> `class SessionDB`

它负责：

1. `create_session(...)`
2. `append_message(...)`
3. `replace_messages(...)`
4. `get_messages(...)`
5. `get_messages_as_conversation(...)`
6. `update_token_counts(...)`
7. `update_system_prompt(...)`
8. `end_session(...)`
9. `reopen_session(...)`

特点：

- SQLite WAL
- 写事务有 jitter retry
- FTS5 / trigram search
- 可存结构化 content、tool_calls、reasoning details

这是 Hermes 事实上的单一真相源。

### 7.2 `SessionStore`

定义在：

- `gateway/session.py`

它不是替代 `SessionDB`，而是 gateway 的会话路由层。

它维护：

1. `session_key -> SessionEntry` 的当前映射
2. reset policy
3. auto reset / resume_pending / suspended 状态
4. legacy transcript JSONL

关键函数：

- `get_or_create_session(...)`
- `reset_session(...)`
- `switch_session(...)`
- `append_to_transcript(...)`
- `load_transcript(...)`
- `update_session(...)`

理解关系时可以这样看：

- `SessionDB` 记录“历史上发生过什么”
- `SessionStore` 记录“当前这个 chat/thread 应该指向哪一个 session_id”

---

## 八、三条入口的共同点与差异

### 8.1 共同点

CLI、TUI、Gateway 最终都会：

1. 准备 user message
2. 准备 conversation history
3. 准备 session_id / session context
4. 调 `AIAgent.run_conversation(...)`
5. 消费统一的 `result` dict

所以如果你要分析“为什么某个模型行为异常 / 为什么 tool loop 卡住 / 为什么 context 被压缩”，核心永远先看：

- `run_agent.py`

### 8.2 差异

#### CLI

- 单进程
- UI 回调直接绑定到 Rich / prompt_toolkit
- `conversation_history` 常驻内存
- 中断通过本地队列和线程实现

#### TUI

- Node/Ink + Python gateway 双进程
- 通过 JSON-RPC 事件流输出
- session 状态维护在 `tui_gateway/server.py` 的 `_sessions`
- 仍然本地直接 new `AIAgent`

#### Gateway

- 平台 adapter -> `MessageEvent` -> `GatewayRunner`
- 有授权、pairing、session_key、queue、interrupt、typing、message edit、media send 等聊天平台特有逻辑
- transcript 由 `SessionStore + SessionDB` 协同维护
- 比 CLI/TUI 多了一层会话路由与平台交互适配

---

## 九、按时间顺序串一次“完整请求”

下面以 gateway 普通文本消息为例，串一次最典型的真实流程：

1. Telegram/Discord/Slack adapter 收到原始平台消息。
2. adapter 把它归一化成 `MessageEvent`，并调用 `GatewayRunner._handle_message(event)`。
3. `_handle_message()` 先做 plugin pre-dispatch、鉴权、pending prompt、slash command、busy session 等分流。
4. 如果是普通消息，runner 先在 `_running_agents` 里占坑，防止并发重入。
5. `_handle_message_with_agent()` 调 `SessionStore.get_or_create_session(source)`，拿到 `session_key` 和 `session_id`。
6. 构建 session context prompt。
7. 通过 `SessionStore.load_transcript(session_id)` 取历史消息。
8. 如 transcript 太大，先做 hygiene compression。
9. `_prepare_inbound_message_text()` 处理图片、音频、文档、reply-to、`@file` 等。
10. `_run_agent()` 根据平台配置准备 tool progress callback、stream consumer、typing 等包装逻辑。
11. `_run_agent()` 构造或复用 `AIAgent`。
12. `AIAgent.run_conversation()` 开始执行：
    - 确保 `SessionDB` 中存在当前 session 行
    - 构建 / 复用 system prompt
    - 组装 API messages
    - 调模型
13. 如果模型返回 tool_calls：
    - `_build_assistant_message()` 标准化 assistant turn
    - 把 assistant tool-call message append 到 `messages`
    - `_execute_tool_calls()` 执行工具
    - 每个工具结果以 `role="tool"` 消息 append 回 `messages`
    - 增量持久化
    - 再次调模型
14. 如果模型返回最终文本：
    - 构建 final assistant message
    - 更新 token usage / compressor 状态
    - 持久化 messages
    - 返回统一 result dict
15. Gateway 对 result 做后处理：
    - reasoning 显示
    - footer
    - 媒体投递
    - transcript 写入 / 更新 session metadata
16. adapter 把 response 发回平台。
17. 用户收到消息。

CLI/TUI 也是同样的第 12~14 步，只是第 1~11 步和第 15~16 步的 I/O 方式不同。

---

## 十、最关键的函数索引

如果你要顺着代码继续深入，优先看下面这些函数。

### 入口与分发

- `cli.py:9144` `HermesCLI.chat`
- `cli.py:3565` `HermesCLI._init_agent`
- `tui_gateway/server.py:2023` `@method("session.create")`
- `tui_gateway/server.py:2929` `@method("prompt.submit")`
- `tui_gateway/server.py:2963` `_run_prompt_submit`
- `gateway/run.py:4645` `GatewayRunner._handle_message`
- `gateway/run.py:5837` `GatewayRunner._handle_message_with_agent`
- `gateway/run.py:12695` `GatewayRunner._run_agent`

### session / transcript

- `gateway/session.py:850` `SessionStore.get_or_create_session`
- `gateway/session.py:1249` `SessionStore.append_to_transcript`
- `gateway/session.py:1303` `SessionStore.load_transcript`
- `hermes_state.py:546` `SessionDB.create_session`
- `hermes_state.py:1261` `SessionDB.append_message`
- `hermes_state.py:1427` `SessionDB.get_messages`
- `hermes_state.py:1514` `SessionDB.get_messages_as_conversation`

### agent 核心

- `run_agent.py:885` `AIAgent`
- `run_agent.py:2225` `AIAgent._ensure_db_session`
- `run_agent.py:3786` `AIAgent._persist_session`
- `run_agent.py:3796` `AIAgent._flush_messages_to_session_db`
- `run_agent.py:8762` `AIAgent._build_assistant_message`
- `run_agent.py:9427` `AIAgent._execute_tool_calls`
- `run_agent.py:9469` `AIAgent._invoke_tool`
- `run_agent.py:10569` `AIAgent.run_conversation`
- `run_agent.py:14169` `AIAgent.chat`

### tool schema 与 tool dispatch

- `model_tools.py:271` `get_tool_definitions`
- `model_tools.py:679` `handle_function_call`
- `toolsets.py:31` `_HERMES_CORE_TOOLS`
- `toolsets.py:73` `TOOLSETS`

---

## 十一、几个设计上的关键观察

### 11.1 Hermes 的核心不是“单次问答”，而是“可恢复的工具循环”

很多系统把一次请求当成：

- user -> model -> assistant

Hermes 不是。它更接近：

- user -> model -> assistant(tool call) -> tool -> model -> tool -> model -> assistant(final)

再加上：

- 中间任何一步都可能被中断、压缩、持久化、恢复

### 11.2 session_id 会变化，session_key 通常不变

在 gateway/TUI/CLI resume/压缩链路里，这是最容易搞混的点。

- `session_key` 代表逻辑槽位
- `session_id` 代表某次具体 transcript 实例

压缩、reset、switch、resume 都可能改变 `session_id`。

### 11.3 持久化不是附属功能，而是主流程的一部分

Hermes 的 resume、retry、undo、compress、session_search、gateway 恢复能力，全都建立在 message 级别持久化之上。

不是“顺手记个日志”，而是“主循环每一步都在维护可恢复状态”。

### 11.4 Gateway 比 CLI/TUI 多的不只是平台适配

Gateway 额外承担了：

- 认证
- pairing
- busy session 排队与打断
- transcript hygiene
- typing / message edit / media send
- slash command 在聊天平台中的特殊语义

所以你分析平台问题时，通常不能只看 `run_agent.py`，还必须联动看 `gateway/run.py` 和对应 adapter。

---

## 十二、结论

Hermes 从“收到用户消息”到“返回消息”的真实流程，可以概括为：

1. **入口层收消息并归一化**
2. **会话层确定当前 session 与 transcript**
3. **上下文层把技能、memory、附件、reply context、`@file` 等并入当前轮输入**
4. **`AIAgent.run_conversation()` 作为统一核心，驱动模型与工具的多轮循环**
5. **每一步关键状态都被写入 `SessionDB` / transcript，保证可恢复**
6. **最后由 CLI/TUI/Gateway 各自的显示/投递层把结果送回用户**

如果只保留一句话来描述 Hermes 的内部机制：

**Hermes 不是一个”聊天壳”，而是一个带强会话管理、强工具调度、强持久化恢复能力的同步 agent runtime；CLI、TUI、Gateway 只是它的三种入口和输出面。**

---

# 下篇：架构审计（Agent Architecture Audit）

基于 `ecc:agent-architecture-audit` 12 层栈模型对 Hermes 进行的系统性审计。聚焦类层次、插件系统、压缩管线、内存子系统、MCP 集成和容错模式。

---

## 十三、核心类层次与继承关系

### 13.1 类图概览

```
ContextEngine (ABC)                     MemoryProvider (ABC)
├── ContextCompressor (默认)            ├── BuiltinMemoryProvider
└── Plugin ContextEngine (LCM 等)       └── ExternalProvider (Honcho/Hindsight/Mem0)
                                              ↑
MemoryManager (编排器，1+1 提供者)  ←── MemoryProvider 实例

ToolRegistry (单例, 线程安全)
├── ToolEntry (schema + handler + check_fn)
└── generation counter (缓存失效)

AIAgent (核心 agent)
├── 持有 ContextEngine
├── 持有 MemoryManager
├── 持有 SessionDB
├── 持有 TodoStore
└── 持有 PluginManager 引用

PluginManager (插件发现与加载)
├── PluginManifest
├── LoadedPlugin
├── PluginContext → PluginLlm (受信任的 LLM facade)
└── HookRegistry (gateway)

SessionDB (SQLite WAL)
SessionStore (gateway 会话路由层)
TrajectoryCompressor (轨迹压缩)
```

### 13.2 AIAgent 的组合关系

`AIAgent` 不是继承式设计，而是组合式设计。它在 `__init__` 中装配所有子系统：

| 属性 | 类型 | 职责 |
|------|------|------|
| `self.context_engine` | `ContextEngine` | 上下文压缩策略 |
| `self.memory_manager` | `MemoryManager` | 跨会话记忆 |
| `self.session_db` | `SessionDB` | SQLite 持久化 |
| `self.todo_store` | `TodoStore` | 任务追踪 |
| `self.tools` | `list[dict]` | 最终暴露给模型的 tool schema |
| `self.valid_tool_names` | `set[str]` | 快速查找合法工具名 |
| `self._cached_system_prompt` | `str` | 缓存的 system prompt |

关键设计决策：**所有子系统在 `__init__` 时确定，运行时不再动态更换**（fallback provider 除外）。

---

## 十四、插件系统架构

### 14.1 四级插件发现源

`PluginManager.discover_and_load()` 按优先级从四个源扫描：

| 优先级 | 源 | 路径 | 特点 |
|--------|-----|------|------|
| 1 (低) | Bundled | `<repo>/plugins/<name>/` | 随代码库发布 |
| 2 | User | `~/.hermes/plugins/` | 用户自定义 |
| 3 | Project | `./.hermes/plugins/` | 需 `HERMES_ENABLE_PROJECT_PLUGINS=1` |
| 4 (高) | Pip entry-point | `setuptools` 入口点 | 通过 pip 安装 |

**高优先级覆盖低优先级**：同 key 冲突时，user 覆盖 bundled，project 覆盖 user。

### 14.2 插件类型

Hermes 插件不是单一形态，而是通过 `plugin.yaml` manifest 声明能力：

- **工具插件**：注册新 tool schema + handler 到 `ToolRegistry`
- **平台适配器**：`plugins/platforms/<name>/` 注册新的聊天平台
- **内存提供者**：`plugins/memory/<name>/` 注册 `MemoryProvider`
- **上下文引擎**：`plugins/context_engine/<name>/` 注册 `ContextEngine`
- **图片生成后端**：`plugins/image_gen/<name>/` 注册图片生成实现
- **模型提供者**：`providers/<name>/` 注册 LLM API 后端
- **Hook 插件**：监听 `pre_tool_call`、`post_tool_call`、`pre_gateway_dispatch` 等事件

### 14.3 PluginLlm：受信任的 LLM Facade

`PluginLlm`（`agent/plugin_llm.py:598`）是插件调用 LLM 的安全门面：

- **信任边界**：插件不能直接调 API，必须通过 `PluginLlm`
- **策略控制**：可限制调用次数、模型范围、token 预算
- **错误隔离**：插件 LLM 调用失败不影响主 agent 循环
- **审计**：所有插件 LLM 调用都经过日志记录

### 14.4 Hook 分发机制

Gateway 中的 `HookRegistry`（`gateway/hooks.py:35`）管理事件钩子：

```
事件触发 → HookRegistry.fire(event_name, payload)
         → 遍历注册的 handler
         → 每个 handler 可阻断或修改事件
         → 异常不中断主流程（隔离保证）
```

关键 hook 点：

| Hook | 触发时机 | 用途 |
|------|---------|------|
| `pre_gateway_dispatch` | Gateway 收到消息后、分发前 | 拦截/修改消息 |
| `pre_tool_call` | 工具执行前 | 权限检查、日志 |
| `post_tool_call` | 工具执行后 | 结果转换、审计 |
| `transform_tool_result` | 工具结果返回模型前 | 格式化、截断 |
| `pre_llm_call` | 模型调用前 | prompt 注入 |
| `on_session_end` | 会话结束时 | 清理、归档 |

---

## 十五、上下文压缩管线

### 15.1 双压缩器架构

Hermes 有两个独立的压缩组件：

```
                    ┌─────────────────────┐
                    │   ContextEngine ABC  │
                    │  (策略接口层)         │
                    └──────────┬──────────┘
                               │ implements
                    ┌──────────▼──────────┐
                    │  ContextCompressor   │ ← 默认，run_agent.py 使用
                    │  (上下文压缩器)       │
                    └─────────────────────┘

                    ┌─────────────────────┐
                    │ TrajectoryCompressor │ ← 独立，gateway/session 层使用
                    │  (轨迹压缩器)         │
                    └─────────────────────┘
```

### 15.2 ContextCompressor 工作流程

`ContextCompressor.compress()`（`context_compressor.py:1354`）的核心逻辑：

```
输入: messages, current_tokens, focus_topic
  │
  ├─ 1. 计算压缩边界
  │     protect_first_n (默认 3) — 保留最早的 n 条
  │     protect_last_n (默认 6)  — 保留最近的 n 条
  │     中间部分为可压缩区段
  │
  ├─ 2. 检查是否已有历史摘要
  │     如果有，从摘要位置之后开始
  │
  ├─ 3. 提取可压缩 turns
  │     turns_to_summarize = messages[compress_start:compress_end]
  │
  ├─ 4. 调用 LLM 生成摘要
  │     _generate_summary(turns_to_summarize, focus_topic)
  │     ├─ 计算摘要 token 预算: _compute_summary_budget()
  │     ├─ 序列化为文本: _serialize_for_summary()
  │     └─ 调模型生成结构化摘要
  │
  ├─ 5. 工具结果预处理
  │     _summarize_tool_result() — 大型工具输出先内联摘要
  │
  └─ 6. 返回新 messages 列表
        [protected_head] + [summary_message] + [protected_tail]
```

### 15.3 压缩触发点

Hermes 在**三个独立位置**触发压缩：

| 触发点 | 位置 | 条件 | 使用组件 |
|--------|------|------|---------|
| Gateway hygiene | `gateway/run.py:~6039` | 调 agent 前 transcript 过大 | 临时 `AIAgent._compress_context()` |
| Agent loop 动态 | `run_agent.py:~10194` | 工具执行后 token 超阈值 | `ContextEngine.should_compress()` |
| 用户手动 | `/compress` 命令 | 用户显式触发 | 支持 `focus_topic` 参数 |

### 15.4 压缩的副作用

压缩不只是缩短消息列表。它会导致：

1. 旧 `session_id` 被结束（`SessionDB.end_session()`）
2. 新 `session_id` 被创建
3. 压缩后的消息写入新 session
4. `self.session_id` 更新
5. System prompt 可能需要重建
6. DB flush cursor 重置

**调用方必须在 turn 结束后检查 `agent.session_id` 是否变化。**

### 15.5 ContextEngine 可扩展点

第三方可通过 `plugins/context_engine/<name>/` 替换默认压缩器：

- `should_compress_preflight()` — 快速预检（不需要真实 token 计数）
- `compress()` — 自定义压缩策略（DAG、LCM 等）
- `get_tool_schemas()` — 暴露引擎专用工具（如 `lcm_grep`）
- `handle_tool_call()` — 处理引擎专用工具调用

---

## 十六、内存子系统设计

### 16.1 分层架构

```
                    ┌─────────────────────┐
                    │   MemoryManager      │ ← 编排器
                    │   (1 内置 + 1 外部)  │
                    └──────────┬──────────┘
                               │ manages
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────▼──────┐  ┌─────▼──────┐  ┌──────▼────────┐
    │ BuiltinProvider │  │  Honcho    │  │  Hindsight    │
    │ (memory_tool)  │  │ (external) │  │  (external)   │
    └────────────────┘  └────────────┘  └───────────────┘
```

### 16.2 MemoryProvider 生命周期

`MemoryProvider`（`agent/memory_provider.py:42`）定义了完整的生命周期接口：

```
Agent 初始化
  │
  ├─ initialize(session_id, **kwargs)
  │    建立连接、创建资源
  │
  ├─ 每个 Turn:
  │    ├─ on_turn_start(turn_number, message)
  │    ├─ prefetch(query) → 背景召回
  │    ├─ handle_tool_call(name, args) → 模型主动查询
  │    └─ sync_turn(user_content, assistant_content) → 异步写入
  │
  ├─ 压缩时:
  │    └─ on_pre_compress(messages) → 压缩前提取关键信息
  │
  ├─ 会话切换:
  │    └─ on_session_switch(new_session_id)
  │
  └─ 会话结束:
       └─ on_session_end(messages) → 最终提取和归档
```

### 16.3 1+1 提供者限制

`MemoryManager` 强制 **最多一个外部提供者**：

- 内置提供者（`MemoryStore`，基于文件）永远存在
- 只允许注册一个外部提供者（Honcho / Hindsight / Mem0 三选一）
- 第二个外部提供者注册会被拒绝并记录 warning
- 通过 `memory.provider` 配置项选择

这个限制的原因是：**避免工具 schema 膨胀和提供者间的记忆冲突**。

### 16.4 内置 MemoryStore

`tools/memory_tool.py:107` 的 `MemoryStore` 是文件系统实现：

- 存储路径：`~/.hermes/memory/`
- 操作：`add`、`replace`、`remove`、`snapshot`
- 每条记忆有 `id`、`content`、`metadata`
- 模型通过 `memory` 工具主动管理

### 16.5 MemoryManager 的容错设计

`MemoryManager` 的所有方法都遵循**非阻塞容错**模式：

```python
for provider in self._providers:
    try:
        provider.prefetch(query, session_id=session_id)
    except Exception as e:
        logger.debug(“Memory provider '%s' prefetch failed (non-fatal): %s”, ...)
```

核心原则：**一个提供者的失败永远不阻塞另一个提供者或主流程。**

---

## 十七、MCP 集成层

### 17.1 架构概览

`tools/mcp_tool.py` 实现了完整的 MCP 客户端：

```
config.yaml (mcp_servers)
  │
  ├─ discover_mcp_tools() ─→ 连接 MCP 服务器，发现工具
  │                         └─ 注册到 ToolRegistry (toolset=”mcp-<name>”)
  │
  ├─ 后台事件循环 (_mcp_loop)
  │    └─ 每个 MCP 服务器运行为独立 asyncio Task
  │
  ├─ ToolRegistry.dispatch(“mcp_tool_name”, args)
  │    └─ run_coroutine_threadsafe() → 后台线程执行
  │
  └─ shutdown_mcp_servers() ─→ 优雅关闭
```

### 17.2 三种传输协议

| 传输 | 配置 | 场景 |
|------|------|------|
| **stdio** | `command` + `args` | 本地子进程（npx、python 脚本） |
| **HTTP/StreamableHTTP** | `url` | 远程 MCP 服务器 |
| **SSE** | `url` + `transport: sse` | 旧版 SSE 协议服务器 |

### 17.3 线程安全设计

MCP 工具调用涉及两个线程：

- **主线程**：`run_conversation()` 同步执行 agent loop
- **MCP 后台线程**：`_mcp_loop` 运行 asyncio 事件循环

桥梁：`asyncio.run_coroutine_threadsafe()` + `concurrent.futures`

所有共享状态通过 `_lock`（`threading.RLock`）保护，兼容 Python 3.13+ free-threading。

### 17.4 Sampling 支持

MCP 服务器可以主动请求 LLM 完成（`sampling/createMessage`）：

- 配置项：`sampling.enabled`、`sampling.model`、`sampling.max_tokens_cap`
- 频率限制：`max_rpm`
- 模型白名单：`allowed_models`
- 工具循环限制：`max_tool_rounds`

这使得 MCP 服务器不只是被动的工具提供者，还能发起自己的 LLM 调用。

### 17.5 安全措施

- 环境变量过滤：stdio 子进程不继承父进程所有环境变量
- 凭证脱敏：错误消息中的 API key 等会被剥离
- 工具名冲突保护：MCP 工具不能覆盖内置工具

---

## 十八、错误处理与容错模式

### 18.1 Fallback Provider 链

`AIAgent` 支持多级 provider 回退：

```python
# run_agent.py:1756
# Provider fallback chain — ordered list of backup providers
self._fallback_chain = config.get(“fallback_providers”, [])
```

回退触发条件：

| 场景 | 行为 |
|------|------|
| 主 provider 认证失败 | 立即切换 fallback |
| 429 Rate Limit | 切换 fallback + 状态提示 |
| 重试耗尽（可重试错误） | 尝试 fallback |
| 非 retryable 客户端错误 | 尝试 fallback |
| Fallback 也失败 | 放弃并返回错误 |

`_try_activate_fallback()`（`run_agent.py:8535`）切换后，**后续所有 turn 都使用 fallback provider**，不会自动切回。

### 18.2 错误分类

Hermes 将 API 错误分为三类：

| 分类 | 示例 | 处理 |
|------|------|------|
| **Retryable** | 429、500、502、503 | 指数退避重试，耗尽后 fallback |
| **Non-retryable client** | 400、401、403 | 直接 fallback 或放弃 |
| **Context overflow** | context_length_exceeded | 触发压缩，不写坏 transcript |

### 18.3 上下文溢出的特殊处理

Context overflow 是最危险的错误类型——如果不处理，会话会进入”每次都溢出”的死循环。

Hermes 的处理：

1. 捕获 `context_length_exceeded` 错误
2. 尝试压缩上下文
3. 如果压缩成功，重试当前请求
4. 如果压缩也失败（内容太短无法再压缩）：
   - Gateway：`session_store.reset_session()` 重置会话
   - CLI/TUI：通知用户
5. **不把溢出的请求写入 transcript**，避免坏 session 越来越大

### 18.4 工具执行的容错

`ToolRegistry.dispatch()` 的错误处理：

```python
try:
    return entry.handler(args, **kwargs)
except Exception as e:
    return json.dumps({“error”: f”Tool execution failed: {type(e).__name__}: {e}”})
```

关键设计：**工具异常永远不中断 agent loop**。错误作为 JSON 字符串返回给模型，模型可以决定如何处理。

### 18.5 Plugin 隔离

插件系统有两层隔离保证：

1. **Plugin LLM 调用隔离**：插件的 LLM 调用失败不影响主循环
2. **Hook handler 隔离**：hook handler 抛异常只记日志，不阻断事件链

---

## 十九、关键设计模式

### 19.1 模式清单

| 模式 | 应用位置 | 说明 |
|------|---------|------|
| **Strategy** | `ContextEngine` → `ContextCompressor` / Plugin | 压缩策略可插拔替换 |
| **Provider** | `MemoryProvider` → 内置 / Honcho / Hindsight | 记忆后端可插拔 |
| **Registry** | `ToolRegistry` 单例 | 工具注册、发现、分发 |
| **Observer/Callback** | CLI/TUI/Gateway → `AIAgent` callbacks | 事件驱动的 UI 更新 |
| **Chain of Responsibility** | Fallback provider chain | 逐级尝试备选 provider |
| **Template Method** | ABC 生命周期方法 | `ContextEngine`、`MemoryProvider` |
| **Facade** | `PluginLlm` | 插件通过受控门面访问 LLM |
| **Snapshot** | `ToolRegistry._snapshot_state()` | 读写分离，线程安全 |
| **Generation Counter** | `ToolRegistry._generation` | 缓存失效的无锁检测 |
| **Sentinel** | `_AGENT_PENDING_SENTINEL` | gateway 并发保护占位符 |

### 19.2 同步核心 + 异步桥接

Hermes 的核心 agent loop 是同步的（`run_conversation` 是同步函数），但多个子系统是异步的：

| 同步 | 异步 |
|------|------|
| `run_conversation()` | MCP 工具调用 |
| `ToolRegistry.dispatch()` | Memory provider prefetch |
| SessionDB 写入 | TrajectoryCompressor |
| | Gateway adapter 消息收发 |

桥梁机制：
- `_run_async()` — 在 agent 的后台事件循环中执行协程
- `asyncio.run_coroutine_threadsafe()` — MCP 专用
- `threading.Thread` — CLI/TUI 的 agent 执行线程

### 19.3 消息标准化层

Hermes 内部消息格式和 API payload 格式不一致。`run_conversation()` 在每次调模型前都会做标准化：

```
内部 messages (Hermes 格式)
  │
  ├─ _sanitize_tool_call_arguments() — 修复损坏参数
  ├─ 删除内部字段 (reasoning, finish_reason, _thinking_prefill)
  ├─ 插入 system message + prefill_messages
  ├─ 应用 Anthropic prompt cache metadata
  ├─ _sanitize_api_messages() — 清理孤儿 tool result
  └─ _drop_thinking_only_and_merge_users()
  │
  ▼
api_messages (Provider 格式)
```

这一层使得 Hermes 可以在保持内部一致性的同时，适配不同 provider 的 API 要求。

---

## 二十、12 层栈审计结果

基于 `ecc:agent-architecture-audit` 的 12 层栈模型，对 Hermes 的快速诊断：

| # | 层 | 风险等级 | 说明 |
|---|-----|---------|------|
| 1 | System prompt | **低** | 有缓存机制，构建路径清晰，支持 SOUL.md 注入 |
| 2 | Session history | **低** | SessionDB 是单一真相源，resume 路径明确 |
| 3 | Long-term memory | **中** | 1+1 提供者限制防冲突，但跨 session 污染路径存在 |
| 4 | Distillation | **中** | 压缩摘要可能丢失细节，`focus_topic` 是可选的补偿 |
| 5 | Active recall | **低** | prefetch + queue_prefetch 分离，非阻塞 |
| 6 | Tool selection | **低** | toolset 分组 + check_fn 代码级过滤，不依赖 prompt 约束 |
| 7 | Tool execution | **低** | Registry.dispatch 有异常兜底，MCP 有重连机制 |
| 8 | Tool interpretation | **低** | 工具结果以 JSON string 返回模型，无二次变换 |
| 9 | Answer shaping | **低** | 内部格式到 API 格式有标准化层 |
| 10 | Platform rendering | **中** | Gateway 的 message edit、media send、typing 可能引入延迟 |
| 11 | Hidden repair loops | **低** | 无隐藏 LLM 修复层，fallback 是显式 provider 切换 |
| 12 | Persistence | **低** | 增量持久化，有 flush cursor 避免重复写入 |

### 总体健康度：**良好（Low-to-Medium Risk）**

Hermes 的架构在 agent runtime 领域属于高成熟度设计。主要优势：
- 工具约束是代码级的（check_fn + toolset），不依赖 prompt 约定
- 没有隐藏的 LLM 修复层
- 压缩、持久化、错误恢复都是主流程的一等公民
- 插件系统有清晰的信任边界

需要关注的风险点：
- 压缩摘要的保真度（第 4 层）—— 当 focus_topic 未使用时
- 跨 session 的记忆污染（第 3 层）—— 外部提供者的质量依赖具体实现
- Gateway 平台渲染延迟（第 10 层）—— 消息编辑/typing 的时序复杂性

---

## 二十一、关键抽象接口速查

| 接口/ABC | 文件 | 核心方法 |
|----------|------|---------|
| `ContextEngine` | `agent/context_engine.py:32` | `should_compress()`, `compress()`, `update_from_response()` |
| `MemoryProvider` | `agent/memory_provider.py:42` | `initialize()`, `prefetch()`, `sync_turn()`, `handle_tool_call()` |
| `ToolRegistry` | `tools/registry.py:151` | `register()`, `dispatch()`, `get_definitions()` |
| `PluginManager` | `hermes_cli/plugins.py:672` | `discover_and_load()`, `fire_hook()` |
| `PluginLlm` | `agent/plugin_llm.py:598` | `complete()`, `structured()` |
| `SessionDB` | `hermes_state.py:309` | `create_session()`, `append_message()`, `get_messages_as_conversation()` |
| `SessionStore` | `gateway/session.py:662` | `get_or_create_session()`, `load_transcript()`, `reset_session()` |
| `MemoryManager` | `agent/memory_manager.py:190` | `add_provider()`, `prefetch_all()`, `handle_tool_call()` |
| `ContextCompressor` | `agent/context_compressor.py:346` | `compress()`, `_generate_summary()`, `_compute_summary_budget()` |
| `TrajectoryCompressor` | `trajectory_compressor.py:332` | `compress_trajectory()`, `_generate_summary()` |
| `HookRegistry` | `gateway/hooks.py:35` | 事件钩子注册与分发 |
