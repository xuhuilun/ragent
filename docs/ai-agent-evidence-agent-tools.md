# Agent 编排与 MCP 工具治理源码证据

本文为本地源码静态研究结果；没有运行测试、调用模型或连接业务数据库。测试条目表示仓库有相应断言，不表示本轮已通过。路径均相对仓库根目录，行号对应本轮源码。核心循环由 AgentScope 依赖提供，不能把框架能力写成完全自研。

## 候选核心点一：ReAct 编排、技能渐进加载与流式生命周期

**建议简历原文：** 基于 AgentScope 构建 ReAct 执行链，将知识检索与业务工具统一编排；按会话上下文渐进加载技能并动态遮蔽工具，结合 PostgreSQL 状态恢复和 SSE 生命周期治理支持多轮任务、暂停续跑与取消收尾。

**真实性：A（代码直接证明），其中“统一编排”“生命周期治理”为对实现的合理归纳。** 无实际吞吐、成功率或延迟指标。不能写 LangGraph、多智能体协作、独立规划器、Reflection 反思、自研 ReAct 框架、主 Agent 多模型熔断切换。

### 证据索引

|文件:行号|类/方法|直接证据|
|---|---|---|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/controller/AgentChatController.java:46](../agent/src/main/java/com/nageoffer/ai/ragent/agent/controller/AgentChatController.java#L46)|`chat/confirm/stop`|GET SSE 首问、POST 确认续跑、停止入口|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java:83](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java#L83)|`streamChat`|用户运行闸门、等待确认时禁止新问|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java:138](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java#L138)|`startRun`|解析 Agent、建立会话、写用户消息、构建输入|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java:221](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java#L221)|`launchStream`|RuntimeContext、任务注册、streamEvents 订阅与取消句柄|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/config/ReActAgentProvider.java:71](../agent/src/main/java/com/nageoffer/ai/ragent/agent/config/ReActAgentProvider.java#L71)|`getAgent/buildAgent`|persona 与目录指纹懒重建，Agent 与目录成对快照|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/config/ReActAgentProvider.java:113](../agent/src/main/java/com/nageoffer/ai/ragent/agent/config/ReActAgentProvider.java#L113)|`buildAgent`|ReAct 模型、工具、迭代与重试上限、PG Store、中间件|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/config/AgentEngineConfiguration.java:45](../agent/src/main/java/com/nageoffer/ai/ragent/agent/config/AgentEngineConfiguration.java#L45)|`agentChatModel`|独立 OpenAIChatModel、DeepSeek formatter、流式及结构化兼容选项|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalog.java:79](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalog.java#L79)|`resolve/buildToolkit`|知识检索、记忆刷写、技能加载、MCP 注册|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/skill/SkillLoadTool.java:113](../agent/src/main/java/com/nageoffer/ai/ragent/agent/skill/SkillLoadTool.java#L113)|`execute/success`|调用时获取手册正文，成功结果 metadata 标记技能|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/skill/AgentSkillMaskingMiddleware.java:77](../agent/src/main/java/com/nageoffer/ai/ragent/agent/skill/AgentSkillMaskingMiddleware.java#L77)|`onReasoning/maskedTools`|推理前过滤工具 schema；映射存于 RuntimeContext|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/skill/AgentSkillMaskingMiddleware.java:122](../agent/src/main/java/com/nageoffer/ai/ragent/agent/skill/AgentSkillMaskingMiddleware.java#L122)|`loadedSkillCodes`|只认成功且未被裁剪的 TOOL 结果标记|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/state/PgAgentStateStore.java:49](../agent/src/main/java/com/nageoffer/ai/ragent/agent/state/PgAgentStateStore.java#L49)|`save/get/getList`|框架 State JSON 编解码与持久化|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/dao/mapper/AgentStateMapper.java:36](../agent/src/main/java/com/nageoffer/ai/ragent/agent/dao/mapper/AgentStateMapper.java#L36)|`upsert`|以 user_id/session_id/state_key 冲突键保存 JSONB|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentRunGate.java:53](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentRunGate.java#L53)|`acquire/release`|Redis setIfAbsent、TTL 与按值比较释放|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentStreamEventBridge.java:151](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentStreamEventBridge.java#L151)|`onEvent`|正文、思考、工具、提示、确认、终答事件适配|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentRunHandle.java:169](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentRunHandle.java#L169)|`interruptUpstream/settle`|先框架中断等存盘，再强制 dispose；唯一收尾|

配置：[pom.xml:31](../pom.xml#L31) MCP SDK 1.1.2，`:32` AgentScope 2.0.2；[bootstrap/src/main/resources/application.yaml:43](../bootstrap/src/main/resources/application.yaml#L43) 主 Agent 固定 DeepSeek 模型、max-iters=10、max-retries=1、SSE 超时 900000ms。`AgentProperties.java:25` 明确单模型无 fallback；`:40` 说明重订阅会重放已发送文本。RAG 子系统的模型路由不能移植为主 Agent 特性。

### 1. 解决什么问题

复杂问题往往需要先检索、再查业务状态、再操作，单次 LLM 文本回答无法实际办事；同时工具多而描述长，全部暴露会增加误选与跳过前置步骤的机会。

### 2. 为什么项目需要

知识问答与购物、请假、预订等业务工具共享入口。Agent 需要根据观察结果决定下一步，也必须把等待确认、取消、部分失败展示为可信业务状态。

### 3. 整体架构

Spring MVC API → AgentChatServiceImpl → ReActAgentProvider → AgentScope ReActAgent；模型侧经过记忆、压缩、拒绝解释、技能遮蔽中间件，动作侧通过 Toolkit 调用知识检索/MCP；框架事件进入 AgentStreamEventBridge → SSE 与业务消息库。框架内部状态单独保存于 t_agent_state。

### 4. 完整执行流程

1. 从登录上下文读取 userId，生成 conversationId/taskId，取得用户级运行位。
2. 读取一次 prompt/目录快照，按指纹复用或重建 Agent；写入用户消息。
3. RuntimeContext 明确携带 userId/sessionId/taskId 及工具事实源。
4. 先发送 meta、登记任务取消回调，再订阅 `agent.streamEvents`。
5. 每次推理前注入用户记忆、压缩上下文、解释历史拒绝、重新计算技能遮蔽。
6. LLM 输出正文或工具调用；load_skill 返回操作手册，下一轮解锁相关工具；search_knowledge 返回检索证据；MCP 返回业务观察。
7. 框架把工具结果纳入后续推理，直到终答、要求确认或达到迭代边界。
8. Bridge 固化正文/思考/工具块，保存业务消息并发送终态；框架保存状态。释放时必要补存、清本轮 Agent 缓存、释放闸门、异步记忆抽取。

### 5. 对应文件

见证据索引；主链重点为 Controller、ServiceImpl、Provider、Catalog、SkillMaskingMiddleware、EventBridge、RunHandle、PgAgentStateStore。框架 ReAct 内部循环不在本仓业务源码中。

### 6. 核心类和函数

`getAgent()` 管构建快照一致性；`onReasoning()` 管当前推理可见工具；`launchStream()` 管启动交接；`onEvent()` 管协议适配；`settle()` 管唯一收尾。把这些职责分别解释，比泛称“搭建 Agent”更能体现个人工程设计。

### 7. 输入输出

用户输入 question/conversationId；框架输入 UserMessage 与 RuntimeContext；模型输入 messages/tools/options。工具结果以 ToolResultBlock 关联 toolCallId。用户输出是 meta/message/tool/confirm/finish/cancel/done 等事件，而非只有字符串。

### 8. 调用关系

`chat → streamChat → startRun → getAgent → launchStream → streamEvents → Toolkit → ToolResult → next reasoning → onComplete`。具体知识检索算法隐藏在 `KnowledgeSearchTool → KnowledgeSearchFacade` 之后，不应把搜索工具说成另一个自主 Agent。

### 9. 数据流转

Prompt/目录决定实例配置；技能正文成为 TOOL 消息；成功标记决定可见工具；框架上下文保存推理与工具配对；业务消息表保存前端回放块。两套持久化服务不同读者，不是双写同一张表。

### 10. 核心原理

ReAct 是推理—动作—观察的迭代控制，而非一次性写死 DAG。这里依据框架结构化 ToolUse/ToolResult 工作，不依赖正则解析“Action:”文本。技能遮蔽定义为 G−L：G 是所有启用技能绑定工具并集，L 是当前有效手册解锁并集；一个工具被多个手册绑定时任一加载即可解锁。遮蔽修改本次 ReasoningInput，不修改共享 Toolkit，避免会话间全局开关污染。

### 11. 采用当前方案的理由

复用 Java/Spring 技术栈与 AgentScope 循环、状态接口；保留本仓知识搜索与 MCP 门面；使用 RuntimeContext 代替跨线程 ThreadLocal；共享 Agent 减少反复组装但以会话状态和调用上下文隔离运行数据。以上是对代码结构的合理技术归纳，不声称作者做过特定性能实验。

### 12. 替代方案

固定工作流适合审批链明确且可验证的任务，但动态问题灵活性较弱；每请求创建 Agent 简化隔离却重复组装；一次性全量工具适合极少工具，规模扩大时增加提示开销；外部工作流引擎可提供持久化调度，但当前未实现。

### 13. 优缺点

优点是职责分层、动态目录快照、跨轮手册约束与完整回放。代价是多轮延迟和模型决策不确定性、框架升级适配成本、两套状态的一致性维护。

### 14. 异常处理

技能不存在返回 ERROR 观察；遮蔽计算异常会全量放行（不是安全级 fail-closed）；缺失关键 persona/description 直接拒绝启动；启动失败独立释放闸门和注销；流失败保留部分内容与工具留痕；确认落库失败提示新建会话。细节见 `AgentSkillMaskingMiddleware.java:82`、`AgentChatServiceImpl.java:125`、`AgentStreamEventBridge.java:206/363/628`。

### 15. 性能问题

一次任务可能串联多次 LLM 和同步 MCP，耗时随轮次增长；同步工具迁移 boundedElastic 只避免阻塞事件线程，不降低远端延迟。每轮扫描历史计算技能加载情况也受上下文长度影响。用户级闸门意味着同一用户不同会话也不能并行；无压测证据不能写高 QPS。

### 16. 生产问题

SSE 断线会触发取消，不是断线自动续传；Redis TTL 无续租，极端停顿可能突破原有互斥假设；状态缓存驱逐当前仅本节点有效（Provider:94）；状态 JSON 随框架升级需要迁移验证；业务消息保存失败不一定阻止终端事件，不能称零丢失。

### 17. 优化方向

为模型/工具/整轮分别设置超时预算；增加恢复版本与 CAS；工具延迟和回合数建立分位监控；异常遮蔽改为按业务风险关闭；多节点缓存失效广播；在明确需求后引入持久化任务恢复。均为建议，非已实现能力。

### 18. 直接知识

ReAct、Function Calling、JSON Schema、Reactor Flux/Mono、SSE 与事件流、线程上下文、状态机、Redis SET NX 与比较释放、工具结果配对、上下文压缩。

### 19. 延伸知识

工作流与自主规划的边界、耐久执行、至少一次与恰好一次语义、背压与缓冲、checkpoint 版本、租约 fencing token、提示注入防护、Agent 任务成功率评测。

### 20. 面试追问与回答要点

1. **为什么这叫 Agent？** LLM 根据工具观察继续选择动作，而非固定检索后生成；循环由 AgentScope 提供，项目实现工具与上下文治理。
2. **是否自研规划器？** 没有独立 planner；工具选择隐含于模型每轮推理，不写 Planning DAG/Reflection。
3. **为什么不用全局工具组激活？** 单例 Agent 下全局可变 Toolkit 状态会跨会话污染；按请求过滤 schema 并在执行侧复核。
4. **压缩后技能还有效吗？** 成功 ToolResult 正文被移除/裁剪后不再算加载，工具重新遮蔽；摘要提到技能不等于加载凭据。
5. **热更新如何不影响旧流？** ActiveAgent 绑定目录与实例；新请求懒重建，旧流继续使用旧实例；不主动关闭旧实例。
6. **停止为何不能直接 dispose？** 框架需机会保存状态；先 interrupt 并等待最多 2 秒，上游未结束才强断和补存。
7. **流式重试为何默认关闭？** 已发文本无法回滚，重订阅会重复输出；maxRetries=1 表示一次尝试。
8. **一份 Agent 怎么隔离用户？** RuntimeContext user/session 与 PG 三元键隔离状态；运行参数不放全局可变字段；同一用户再加 Redis 运行位。
9. **SSE 就有背压吗？** 不能这样说。Flux 和 Servlet SseEmitter 跨边界，还需研究发送缓存、慢客户端及线程占用。
10. **PG 保存意味着一定能恢复吗？** 只保证保存过的框架状态可再加载；进程崩溃前未存状态、业务写入不确定性、schema 升级都需要额外处理。

### 独立知识树

```text
ReAct 编排
├── 模型循环：messages → tool call → observation → next reasoning
├── 上下文：persona / user memory / compaction / skill manual
├── 技能：load_skill → metadata → G−L schema masking → execution guard
├── 状态：RuntimeContext / PG JSONB / business replay blocks
├── 流：Flux → event bridge → SSE → terminal event
└── 生命周期：Redis gate / startup handoff / interrupt / single settlement
```

## 候选核心点二：MCP 工具桥接与人工确认闭环

**建议简历原文：** 设计 MCP 工具目录与 AgentScope 适配层，基于 JSON Schema、读写声明和服务端状态实现高风险操作人工确认；贯通用户身份透传、确认续跑及业务结果回写，降低模型越权代填与误执行业务操作的风险。

**真实性：A/B。** 代码证明确认、身份传递和工具结果；“降低风险”是机制推论，不能量化。不能写 OAuth 已接入、任意第三方 MCP 安全接入、所有入参严格服务端 Schema 验证、分布式 exactly-once、取消即回滚、支付网关已集成。

### 证据索引

|文件:行号|类/方法|直接证据|
|---|---|---|
|[rag/src/main/java/com/nageoffer/ai/ragent/rag/core/mcp/McpClientAutoConfiguration.java:53](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/mcp/McpClientAutoConfiguration.java#L53)|`init/registerRemoteTools`|启动 initialize/listTools、Streamable HTTP、注册远端工具|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalog.java:138](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalog.java#L138)|`resolveMcpTools`|意图树配置与实际执行器取交集，排除内置重名|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalog.java:198](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalog.java#L198)|`McpToolBinding.of`|needsConfirm=意图确认 OR readOnlyHint 非 true|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/McpToolBridge.java:76](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/McpToolBridge.java#L76)|`checkPermissions/callAsync`|PermissionDecision.ask、技能执行前复核|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/McpToolBridge.java:137](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/McpToolBridge.java#L137)|`execute`|同步调用移 boundedElastic，RuntimeContext 身份、结果标准化|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentStreamEventBridge.java:278](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentStreamEventBridge.java#L278)|`onRequireUserConfirm`|参数全量可见的确认块，schema title 标签|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentStreamEventBridge.java:341](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentStreamEventBridge.java#L341)|`settleAwaitingConfirm`|AWAITING_CONFIRM 消息入库与 confirm 终态|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java:161](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java#L161)|`startConfirmRun`|读取确认卡、构建 metadata 确认消息|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java:187](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java#L187)|`resolveConfirmResultsOrExpire`|从框架状态末条 assistant 中提取 ASKING 调用|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentConversationServiceImpl.java:220](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentConversationServiceImpl.java#L220)|`selectPendingConfirm`|message/conversation/user 同时约束，检查待确认状态|
|[agent/src/main/java/com/nageoffer/ai/ragent/agent/confirm/AgentConfirmDenialMiddleware.java:59](../agent/src/main/java/com/nageoffer/ai/ragent/agent/confirm/AgentConfirmDenialMiddleware.java#L59)|`onReasoning`|仅改写用户拒绝文案、保持 ID 和状态|
|[rag/src/main/java/com/nageoffer/ai/ragent/rag/core/mcp/McpClientToolExecutor.java:50](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/mcp/McpClientToolExecutor.java#L50)|`execute/buildRequest`|arguments 和 _meta 分离；异常标准化|
|[mcp-server/src/main/java/com/nageoffer/ai/ragent/mcp/config/McpServerConfig.java:43](../mcp-server/src/main/java/com/nageoffer/ai/ragent/mcp/config/McpServerConfig.java#L43)|`mcpServlet/mcpServer`|独立 /mcp 传输，缺 readOnlyHint 拒绝启动|
|[mcp-server/src/main/java/com/nageoffer/ai/ragent/mcp/executor/McpToolResults.java:53](../mcp-server/src/main/java/com/nageoffer/ai/ragent/mcp/executor/McpToolResults.java#L53)|`userId`|信任域内身份读取；明确 _meta 不是认证|
|[mcp-server/src/main/java/com/nageoffer/ai/ragent/mcp/executor/McpToolSchema.java:78](../mcp-server/src/main/java/com/nageoffer/ai/ragent/mcp/executor/McpToolSchema.java#L78)|`build`|additionalProperties=false 仅模型约束，并非入参自动验证|
|[mcp-server/src/main/java/com/nageoffer/ai/ragent/mcp/executor/bit/BitOrderCreateMcpExecutor.java:115](../mcp-server/src/main/java/com/nageoffer/ai/ragent/mcp/executor/bit/BitOrderCreateMcpExecutor.java#L115)|`handleCall/placeOrder`|用户身份要求、TransactionTemplate、业务参数校验及数据库写入|

### 1. 解决什么问题

让 LLM 调用实际业务能力时，参数生成、身份、权限和执行结果必须分开管理；模型不应自行提供当前登录身份，也不应直接执行具有副作用的动作。

### 2. 为什么需要

项目不止知识问答，还包含订单、工单、会议、请假等读写工具；“回答正确”和“操作获用户同意”是不同约束。

### 3. 整体架构

LLM Function Calling → AgentScope ToolBase → McpToolBridge → 既有 McpToolExecutor → McpSyncClient → HTTP /mcp → SyncToolSpecification → 业务 Mapper/外部 API；PermissionDecision.ask 在执行前切出确认分支。

### 4. 完整执行流程

1. 应用启动发现 MCP tools 并注册执行器；Agent Catalog 仅挂载意图树配置且可用工具。
2. LLM 根据工具 description/schema 生成 tool name、arguments、callId。
3. Bridge 权限检查：明确只读且未要求确认才放行；其他要求用户确认。
4. 框架发 REQUIRE_USER_CONFIRM；Bridge 把工具参数和中文标签持久化成确认卡，结束本条 SSE。
5. 用户 POST conversationId/messageId/approved；服务校验归属，读取 Agent 状态 ASKING 工具原始参数。
6. 新运行准备完成后先结算旧卡片，再订阅携带 ConfirmResult 的框架输入。
7. 同意进入执行；拒绝成为 DENIED 观察并用中间件解释“用户取消”。
8. RuntimeContext userId 放入 _meta，模型 arguments 不承担身份职责；远端业务工具检查身份与资源范围。
9. MCP TextContent/isError 转为 ToolResultBlock，模型继续推理，SSE/消息库记录结果。

### 5. 对应代码

见索引。SDK 传输与主 Agent 桥之间跨 `agent`、`rag`、`mcp-server` 三模块；`AgentEngineConfiguration.java:35` 说明复用既有连接是为避免 SDK 版本冲突。

### 6. 核心类函数

Catalog 计算生效策略，Bridge 消费策略而非重复推导；Service 管确认恢复；EventBridge 管卡片展示；MCP Server 工具处理业务校验与事务。这些边界让新增工具可复用确认与事件机制。

### 7. 输入输出

模型输入工具 schema；工具输入业务 arguments 与可信应用上下文透传的 _meta；确认输入只有标识与布尔结果；执行输出 CallToolResult，再映射为带原 callId 的 ToolResultBlock。

### 8. 调用关系

`checkPermissions → ASKING → RequireUserConfirmEvent → AWAITING_CONFIRM → confirmPendingTool → getAgentState → ConfirmResult → streamEvents → callAsync → executor.execute → client.callTool → handleCall`。

### 9. 数据流转

确认卡持久化供用户查看；Agent state 保存真正续跑参数；_meta 与 arguments 分离避免模型控制用户 ID；工具结果串回原调用。确认卡结算、Agent state 存盘、远端业务事务不属于同一事务。

### 10. 核心原理

Function Calling 是模型输出结构化调用意图，MCP 是工具发现/调用传输协议；两者通过适配层衔接，不能等同。HITL 是执行前挂起与后续新请求恢复，不是服务器线程阻塞等待人。Schema 提供参数约束/可读字段，但不能替代业务授权与状态机。

### 11. 为什么这样设计

统一 MCP 执行器减少连接和协议重复实现；未知读写属性默认要求确认；后端从保存状态取参数，避免批准时改价改账号；确认结束释放流资源，适应人类不确定响应时间。

### 12. 替代方案

直接 Java 方法工具适合单体内部能力但耦合模型层与业务；REST 工具同样可实现但需自行维护统一发现协议；固定审批工作流适合高合规确定性流程。当前方案适合已有 MCP 服务和混合知识业务场景。

### 13. 优缺点

优点：读写策略集中、身份不暴露为模型自由参数、确认内容透明、可接入异构工具。缺点：依赖外部注解真实性、确认与业务提交间仍有竞态、远端网络失败会造成执行结果不确定。

### 14. 异常处理

启动远端连接失败跳过该服务工具；不存在的工具不挂载。远端异常转标准错误观察；无 ASKING 状态则使卡片失效并提示重问。确认准备失败保留待确认卡。注意 McpClientToolExecutor 捕获全部 Exception 并转 isError，可能掩盖 Bridge 对取消异常的分类，不能宣称所有取消均精确识别。

### 15. 性能问题

当前 MCP 为同步客户端，转 boundedElastic 仍占工作线程；工具目录每请求解析，元数据增长有成本；远端工具只在初始化 listTools，不能说已实现实时订阅 tools/list_changed 和动态重连。

### 16. 生产问题

[mcp-server/src/main/resources/application.yml:2](../mcp-server/src/main/resources/application.yml#L2) 绑定 127.0.0.1；_meta 用户信息只在受信应用调用链成立，不是 OAuth。网络超时不能证明远端没提交；停止不会撤销已写订单。additionalProperties=false 不代表服务端拒绝所有额外参数。批量批准对应当时末条 assistant 所有 ASKING 调用，未见逐项编辑参数审批。

### 17. 优化方向

引入外部认证与工具级授权；高风险动作批准时重校验最新业务状态；幂等业务键与结果查询；工具参数服务端统一校验；确认记录绑定 callId/参数哈希及版本；隔离各远端超时与并发预算。以上均为延伸建议，不能写进已有成果。

### 18. 直接知识

JSON Schema、Function Calling、MCP tools/list 与 tools/call、Streamable HTTP、人工确认状态机、用户数据范围、同步转异步适配、业务事务。

### 19. 延伸知识

TOCTOU（确认后资源状态变化）、OAuth 与服务间认证、ABAC/RBAC、幂等键、Outbox、Saga 补偿、未知提交结果处理、工具供应链与提示注入。

### 20. 面试追问与回答要点

1. **MCP 等于 Function Calling 吗？** 前者承载能力发现与远程调用，后者是模型生成结构化调用；Bridge 连接二者。
2. **怎么防模型冒充其他用户？** 模型 schema 不用 userId 决定身份；RuntimeContext 的登录用户独立进入 _meta；服务端按该身份限定业务数据。
3. **_meta 能认证吗？** 不能。当前仅本机可信应用链；开放第三方必须独立认证授权。
4. **默认是否全部直接调用？** 不是。仅 readOnlyHint=true 且无意图确认标记可直接运行，其他默认 ask。
5. **用户批准时能篡改参数吗？** 该 API 不接收工具参数；服务端从 Agent 状态取原 ToolUseBlock。
6. **用户拒绝为何还需要模型？** 模型需看到准确观察后解释取消；拒绝中间件避免把用户意愿误读为系统权限故障。
7. **如何保证一次执行？** 当前用户运行位及卡片状态降低重复续跑，但没有跨远端业务的全局 exactly-once；应补幂等键与查询机制。
8. **为什么取消不等于回滚？** HTTP 请求可能已触达并提交，客户端只能停止等待；业务补偿需独立设计。
9. **Schema 能保证参数正确吗？** 不能保证语义、资源授权或业务状态；本仓还明确没有统一入参自动 Schema 校验。
10. **事务能覆盖 Agent 和远端吗？** 订单工具本地事务保障该业务数据库原子性，不覆盖 Agent 状态、确认卡和远端 HTTP。

### 独立知识树

```text
MCP 工具治理
├── 模型接口：description / JSON Schema / ToolUseBlock
├── 发现：initialize → listTools → registry → intent intersection
├── 策略：readOnlyHint / requireConfirm / reserved names / skills guard
├── HITL：ASKING → persisted card → approve/deny → ConfirmResult
├── 身份：RuntimeContext → _meta → scoped business query
├── 执行：boundedElastic → MCP HTTP → business transaction
└── 可靠性：error observation / uncertain commit / idempotency / audit
```

## 测试证据与验证边界

本轮仅阅读测试与断言，未运行 Maven。不得表述“测试全部通过”。

|测试文件:行号（均位于 agent/src/test/java/com/nageoffer/ai/ragent/agent）|断言/目的|不证明|
|---|---|---|
|`config/ReActAgentProviderTest.java:111`|一次解析目录和 prompts|真实并发全部时序|
|`config/ReActAgentProviderTest.java:129`|目录不变复用同一 Agent|生产内存开销|
|`config/ReActAgentProviderTest.java:139`|工具出现后重建|实时远端自动发现|
|`config/ReActAgentProviderTest.java:167`|工具默认单次尝试|业务全局幂等|
|`config/ReActAgentProviderTest.java:182`|模型 maxRetries=1|模型永不失败|
|`tool/AgentToolCatalogTest.java:156`|意图漏配时写工具仍确认|第三方声明真实|
|`tool/AgentToolCatalogTest.java:197`|additionalProperties 透传|服务端自动 Schema 验证|
|`tool/McpToolBridgeTest.java:73`|身份经 _meta 传递|OAuth 认证|
|`service/impl/AgentChatServiceImplTest.java:490`|准备完才结算卡，订阅前结算|跨库原子事务|
|`service/impl/AgentChatServiceImplTest.java:527`|取消赢得准备阶段竞争时不续跑|远端提交可撤销|
|`service/handler/AgentRunHandleTest.java:363`|超时强断标记|所有网络终止即时生效|

特别注意：`skill/AgentScopeToolGroupConcurrencyTest.java:80` 的测试名是 `shouldReproduceCrossSessionToolGroupPollution`，是在复现上游工具组污染；`skill/AgentScopeSkillActivationLifecycleTest.java` 研究框架技能生命周期。不能把这两份框架行为探针直接当成当前自实现遮蔽“所有并发隔离均已验证”的证明。
