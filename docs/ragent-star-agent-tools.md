# Ragent 面试话术：Agent 执行与工具治理

整理日期：2026-10-13（Asia/Shanghai）。源码基线：`4d536085`。对应[简历正文](ragent-resume-personalization.md)第 1、2 条职责。

按用户最新说明，项目代码由本人独立完成，以下直接使用“我设计、我实现”的开发者口径。AgentScope、Spring Boot、Redisson 等依赖提供的能力仍明确归于框架。S 描述实际机制要应对的设计场景，不补造线上事故；R 描述已实现的机制与行为，不补造性能、用户规模或实验结果。本次重新读取相关源码及测试，未执行 Java 测试、启动应用或做生产实验。

<a id="point-1"></a>

## 第 1 条：Agent 执行、状态恢复与流生命周期

### 1.1 完整简历条目

> 基于 AgentScope ReAct 统一接入知识检索与 MCP 工具，以 PostgreSQL 持久化会话状态；结合 Redis 运行位、跨节点取消与 SSE 收尾机制，支持多轮任务恢复和流式执行控制。

### 1.2 STAR 拆解

**S — 场景：** 我独立开发 Ragent 时，要让企业知识问答与业务办理共用一套 Agent 入口。用户可能先查规定，再补充条件并调用业务工具，一次请求需要多轮模型和工具交互。与此同时，HTTP 方法返回 SSE emitter 后，后台任务仍在运行；停止、断连和下一次请求都可能与状态保存、资源释放交错。

**T — 任务：** 我要实现一个可持续多轮交互、可保存执行状态、可停止的 Agent 服务，把运行控制覆盖到整条流，并协调同用户的并发启动、取消和收尾。ReAct 推理循环复用 AgentScope，我独立设计实现项目中的工具装配、PG 状态适配和运行生命周期治理。

**A — 行动：** 我做了以下接线与时序控制。

1. 用 `ReActAgentProvider` 构建主 Agent，统一注册 `search_knowledge` 和 `McpToolBridge`。Provider 复用实例，以人设和工具目录指纹判断懒重建；单次请求取得 Agent 与目录快照，避免分别读取造成不一致。
2. 实现 `PgAgentStateStore`，使用 AgentScope codec 将状态按 `userId / sessionId / stateKey` 写入 PG JSONB。业务消息另存 `t_agent_message`，不以聊天正文重放替代框架工作状态。
3. 用 Redis `setIfAbsent + TTL` 抢用户级运行位，值记录本轮 `taskId|conversationId`。位覆盖完整流，释放时 CAS 比较原值，防止旧回调删除新运行占位。
4. 用 `AgentRunHandle` 的同一把生命周期锁协调启动与结算，把订阅和中断能力绑定放进启动交接区。正常、失败和取消汇入一次结算；启动准备失败另行释放运行位并注销任务。
5. 将 SSE 的超时、错误和完成回调接入取消。`StreamTaskManager` 写 Redis 取消标记并广播，执行节点复核属主；取消早于注册时，在注册阶段复核预埋标记。
6. 中断先通知框架，再等待上游终止，超时后 `dispose`。退出时按需补存状态、清本轮实际使用实例的缓存、释放运行位，再调度记忆抽取，避免旧流清理干扰下一轮。

**R — 结果：** 我实现了从入口、工具调用到终态的完整执行治理链路：Agent 状态有 PG 持久化适配，多轮请求可沿框架状态继续；停止请求可通过 Redis 通知执行节点；正常、异常和取消出口共用收尾协议。代码中的 owner 比较和本地启动锁分别处理占位归属与启动交接问题。结果限定为机制已实现，不将其写成零状态丢失、任意网络条件下取消必达或远程副作用可回滚；本轮没有运行数据。

### 1.3 约 90—120 秒连续口述稿

我独立开发 Ragent 时，企业知识问答和业务办理都需要多轮工具交互。一次请求可能先查资料，再根据结果补充条件并继续调用工具。这里最关键的是，Controller 返回 SSE 后任务还在运行，用户停止、关闭页面或者再次提问，都可能和旧任务的状态保存交错。所以我复用 AgentScope 的 ReAct 循环，自己实现工具装配、状态适配和整个流的生命周期控制。

具体来说，我把知识检索和 MCP 工具放进统一目录，用 PostgreSQL 保存框架编码的会话状态，同时单独保存面向前端的消息记录。并发控制采用用户级 Redis 运行位，覆盖整条流，释放时比较本轮标识，避免旧回调误删新占位。启动和收尾共用生命周期锁，防止取消已经完成后，后台又开始订阅执行。跨节点停止先写取消标记再广播，执行节点检查属主，注册时也检查提前到达的标记。SSE 断连和超时进入同一取消路径。

中断时我先给框架保存状态的机会，再强制断流；最后按需补存状态、清本轮实例缓存，然后释放运行位。这样就实现了多轮恢复和流式任务控制。我也明确区分这些保证：TTL 没有续租，CAS 保护的是占位释放，不能保证远程业务只执行一次，已经成功的工具操作也不会因为停止而回滚。

### 1.4 关键源码文件

相对链接从本文所在的 `docs` 目录定位，`#L` 为实际源码行号。先读“核心必读”，再沿支撑、配置和测试展开。

| 层级 | 文件与行号 | 类 / 方法 | 面试时要讲清的职责 |
| --- | --- | --- | --- |
| 核心必读 | [AgentChatController.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/controller/AgentChatController.java#L46) | `chat`、`confirm`、`stop` | Agent SSE、确认续跑与停止三个入口，emitter 超时来自配置 |
| 核心必读 | [ReActAgentProvider.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/config/ReActAgentProvider.java#L71) | `getAgent`；L113 `buildAgent` | 复用/懒重建实例；装配第三方 ReAct 循环、目录、中间件与 StateStore |
| 核心必读 | [AgentToolCatalog.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalog.java#L89) | `buildToolkit` | 知识检索、记忆、技能与 MCP 工具统一注册 |
| 核心必读 | [AgentChatServiceImpl.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java#L83) | `streamChat`；L221 `launchStream` | 抢运行位、pending 校验、创建运行句柄、注册取消与订阅事件流 |
| 核心必读 | [AgentRunGate.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentRunGate.java#L53) | `acquire`；L83 `release` | 用户级 SETNX 占位；owner 值 CAS 清理，TTL 是 SSE 超时的两倍 |
| 核心必读 | [AgentRunHandle.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentRunHandle.java#L126) | `start`；L169 `interruptUpstream`；L253 `settle` | 启动/结算互斥；框架中断→等待→dispose；一次收尾与释放钩子 |
| 核心必读 | [StreamTaskManager.java](../framework/src/main/java/com/nageoffer/ai/ragent/framework/web/StreamTaskManager.java#L101) | `register`；L146 `cancelByUser`；L203 `cancelLocal` | Redis 属主与预埋标记，广播跨节点取消，执行端属主复核和本地 CAS |
| 核心必读 | [AgentStreamEventBridge.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentStreamEventBridge.java#L174) | `onComplete`、`onError`、`finishCancelledStream` | Agent 事件转业务消息、SSE 终态；待确认与正常结束分开结算 |
| 核心必读 | [PgAgentStateStore.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/state/PgAgentStateStore.java#L49) | `save`、`get`、`getList` | AgentScope codec 序列化/反序列化，按用户/会话/状态键存取 |
| 支撑 | [AgentChatServiceImpl.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java#L264) | `bindReleaseHooks`；L364 `bindEmitterLifecycle` | 失败补存、清本轮实例、放运行位的顺序；SSE 三类回调触发取消 |
| 支撑 | [KnowledgeSearchTool.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/KnowledgeSearchTool.java#L85) | `callAsync`、`execute` | 在 boundedElastic 调知识入口，取消返回 `INTERRUPTED`，避免当普通失败重试 |
| 支撑 | [KnowledgeSearchFacade.java](../rag/src/main/java/com/nageoffer/ai/ragent/rag/service/KnowledgeSearchFacade.java#L65) | `search` | 知识工具内部的改写、意图、检索与证据合成；不再传整段会话历史 |
| 支撑 | [AgentStateMapper.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/dao/mapper/AgentStateMapper.java#L33) | `upsert`、`selectPayload` | JSONB 状态写入与读取；upsert 没有 revision/fencing 条件 |
| 配置 | [application.yaml](../bootstrap/src/main/resources/application.yaml#L35)；[AgentEngineConfiguration.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/config/AgentEngineConfiguration.java#L46) | `ragent.engine.type`、`agent.*`；`agentChatModel`、`agentStateStore` | 缺省 Agent 档、10 次迭代配置、尝试次数 1、900000 ms SSE 超时；主 Agent 独立单模型 |
| SQL | [260812_agent_engine.sql](../resources/database/upgrades/v2.0.0/260812_agent_engine.sql#L23) | `t_agent_message`、L40 `t_agent_state` | 业务消息与框架状态分表；状态联合主键和 JSONB payload |
| 静态测试 | [AgentRunGateTest.java](../agent/src/test/java/com/nageoffer/ai/ragent/agent/service/handler/AgentRunGateTest.java#L58)；[AgentRunHandleTest.java](../agent/src/test/java/com/nageoffer/ai/ragent/agent/service/handler/AgentRunHandleTest.java#L130) | `shouldReleaseOnlyOwnSlot`、`shouldNotStartAfterCancellationSettles`、`shouldKeepGateUntilStartupHandoffCompletes` | mock Redis 与 latch 固定竞争窗口；说明测试意图，不能充当真实 Redis/PG 集成结果 |
| 静态测试 | [AgentChatServiceImplTest.java](../agent/src/test/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImplTest.java#L268) | `shouldSaveStateBeforeEvictWhenForcedDisposal`；L352 `shouldClearStateCacheBeforeReleasingGate` | 补存与清理顺序、取消与完成竞争、SSE 生命周期回调断言 |
| 静态测试 | [StreamTaskManagerTest.java](../framework/src/test/java/com/nageoffer/ai/ragent/framework/web/StreamTaskManagerTest.java#L87) | `shouldRejectCancelFromNonOwner`；L123 `shouldHonorOwnMarkerPlantedBeforeRegister`；L187 `shouldNotResurrectTaskAfterUnregister` | 属主、注册前标记与晚绑定测试；没有本轮网络分区/广播丢失验证 |

### 1.5 可追踪调用链

```text
首问 / 继续提问
AgentChatController.chat
  → AgentChatServiceImpl.streamChat
  → AgentRunGate.acquire（SETNX，userId 维度）
  → pending 校验 → startRun
  → ReActAgentProvider.getAgent（Agent + 目录快照）
  → launchStream → bindReleaseHooks / bindEmitterLifecycle
  → StreamTaskManager.register（属主落地、复核预埋取消）
  → ReActAgent.streamEvents（AgentScope 框架循环）
  → AgentRunHandle.start → subscribe + bindStream + bindHandle
  → AgentStreamEventBridge.onEvent
  → search_knowledge / McpToolBridge → 工具结果进入后续推理
  → 框架状态保存 → PgAgentStateStore → AgentStateMapper → t_agent_state

主动停止 / 断连
AgentChatController.stop → AgentChatServiceImpl.stopTask → cancelByUser
或 emitter timeout / error / completion → StreamTaskManager.cancel
  → publishCancel（Redis 标记 → Pub/Sub）
  → 运行节点 cancelLocal（属主复核、本地 CAS）
  → AgentRunHandle.interruptUpstream（interrupt → 等待 → dispose）
  → 事件桥取消收尾 → AgentRunHandle.settle
  → unregister → 按需补存状态 → 清本轮实例缓存
  → AgentRunGate.release（CAS）→ 后台记忆抽取
```

`subscribe` 可能同步触发事件或完成；启动锁协调的是本句柄的交接与结算，不将上述步骤变成 PG、Redis 和远端业务的共同事务。状态 Store 与业务消息表也不是同一个原子提交。

### 1.6 三个高频追问

**追问 1：既然用了 AgentScope，你自己到底实现了什么？**

我复用的是框架的推理、工具交互循环与状态协议。我独立实现了模型/工具装配、知识检索工具和 MCP 桥接、PG StateStore、用户运行位、跨节点取消、SSE 事件适配以及启动、失败和收尾的交接逻辑。换框架时，要重新适配状态 codec、事件和工具权限接口；生命周期治理和业务工具边界仍是项目工程工作。源码从 `ReActAgentProvider.buildAgent` 和 `AgentChatServiceImpl.launchStream` 开始定位。

**追问 2：Redis 运行位加 CAS，是否就保证了并发安全和工具只执行一次？**

不是完整保证。`setIfAbsent` 在运行位有效时拒绝同用户的其他流，CAS 释放只防旧任务误删新占位；本地锁另行保护启动/结算交接。固定 TTL 没有续租，旧任务超过 TTL 仍可能继续，PG 状态 upsert 也没有 fencing 条件。更重要的是，这些控制没有为远程业务建立操作幂等账本，因此不能保证订单、提交等副作用 exactly-once。强副作用业务要单独设计稳定操作 ID、下游幂等和未知结果对账。

**追问 3：停止请求落在另一个节点、甚至早于任务注册，怎么办？**

我把属主和取消标记放进 Redis；正常停止先校验属主，再写携带发起人的标记并广播，执行节点再次校验后触发中断与收尾。属主尚未注册时，标记保留发起人，在注册时再复核，避免用可预测 taskId 抢跑取消他人任务。CAS 避免同节点重复进入取消；晚绑定时检查已取消状态。这个方案覆盖注册窗口，但 Pub/Sub 不保证可靠投递，也不能证明远端请求一定停止；取消动作异常和活动任务广播丢失仍需要额外故障验证与治理。

<a id="point-2"></a>

## 第 2 条：Skills 双重校验与 MCP 人工确认

### 2.1 完整简历条目

> 设计 Skills 按需加载与工具双重校验，随技能手册退出上下文收回关联工具；为 MCP 写操作及需确认工具接入 HITL，恢复时从服务端状态读取原始参数，约束未确认调用及参数篡改。

### 2.2 STAR 拆解

**S — 场景：** 我设计业务办理工具时，既要允许模型根据需求选择动作，也要让它遵循场景手册。例如提交申请前应先查询规则并澄清必要字段；如果所有写工具始终暴露，模型可能直接提交。长会话压缩又可能移走手册正文，但工具仍保持激活。另一类问题是用户批准操作时，确认接口不能接受一份被客户端替换的工具参数。

**T — 任务：** 我要实现按场景加载手册的工具编排机制，让手册与关联工具的有效期绑定；对 MCP 需确认操作挂起、展示参数并在用户决定后恢复。技能约束业务顺序，HITL 控制是否执行，两者分别实现，不把它们当成完整业务权限系统。

**A — 行动：** 我将能力拆成目录、上下文判定、调用兜底和确认恢复四层。

1. `SkillLoadTool` 的工具说明只展示启用技能清单，模型选中后才读取手册正文。成功结果写入 `ragent_loaded_skill` 元数据，正文与加载标记一起进入工具结果。
2. 推理前根据当前消息列表，计算启用技能关联工具并集 G 与已加载技能解锁工具并集 L，遮蔽 `G − L`。成功且未被裁剪成占位块的加载结果才算有效；同工具关联多份手册时，任一手册有效即可解锁。
3. 遮蔽映射保存在本轮 `RuntimeContext`，`McpToolBridge.callAsync` 执行前再次检查，防止模型依据历史调用直接选择本轮隐藏的工具。上下文压缩或裁剪移走手册后，下一次推理自动收回工具。
4. 在目录里合并意图节点确认配置和服务端 `readOnlyHint`：任一节点要求确认，或者服务端未明确声明只读，工具就返回 ASK。AgentScope 挂起为 `ASKING`，事件桥将参数封装成待确认卡片。
5. 确认请求只接收会话 ID、消息 ID 和批准布尔值。服务端校验用户/会话/消息及 pending 状态，再从 Agent 状态最后一条 assistant 消息取 `ASKING` 调用，按原参数构造确认结果恢复；在启动互斥段中先结算卡片再订阅。
6. 身份从 `RuntimeContext` 传入 MCP `_meta`，不让模型自行填写身份；Agent 档关闭知识检索内部的旧 MCP 执行支路，保持工具治理入口一致。

**R — 结果：** 我实现了“加载手册→关联工具可见→调用前复查”和“待确认→服务端参数恢复→继续执行”两条链路。工具可见性随上下文变化，确认接口不提供替换参数的通道。正常链路可约束未加载或未确认调用；仅 `tool_ids` 关联工具受遮蔽，计算异常存在 fail-open，确认和远端执行也不共享事务。结果不扩展为完整鉴权、防注入或远端 exactly-once。

### 2.3 约 90—120 秒连续口述稿

我独立实现这部分时，主要解决两个问题：模型调用业务工具前需要理解办理步骤，而产生副作用的操作需要用户明确确认。比如提交申请，模型应该先加载手册、查规则并补齐字段。把全部工具一直交给模型，容易跳过前置步骤；手册被长对话压缩掉以后，也不能继续依赖以前的激活状态。

所以我设计了按需加载。模型先看到技能名称和适用场景，选中后再取得正文，成功结果带上技能标记。每次推理都从当前上下文判断哪些手册仍有效，隐藏尚未解锁的关联工具；执行前 MCP 桥再检查同一份遮蔽结果，避免模型照着历史调用直接闯入。手册退出上下文后，工具跟着收回；没有配置到技能 tool_ids 的工具则始终可见。

对于需要确认的 MCP 操作，我根据业务配置和服务端只读声明返回 ASK，让框架挂起并展示工具参数。用户只提交同意或拒绝，原参数从服务端 Agent 状态读取，不接收前端重新传参。恢复启动时，先结算确认卡再订阅执行。这就形成了可恢复的工具治理闭环。我也保留了清楚的边界：技能计算异常会放开本轮工具，它不是鉴权系统；确认不等于远端幂等，工具执行后响应丢失仍需要业务层对账。

### 2.4 关键源码文件

| 层级 | 文件与行号 | 类 / 方法 | 面试时要讲清的职责 |
| --- | --- | --- | --- |
| 核心必读 | [SkillLoadTool.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/skill/SkillLoadTool.java#L75) | `getDescription`；L113 `execute`；L130 `success` | 清单进工具说明、正文调用时读取、成功结果写技能 metadata |
| 核心必读 | [AgentSkillMaskingMiddleware.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/skill/AgentSkillMaskingMiddleware.java#L77) | `onReasoning`；L101 `maskedTools`；L122 `loadedSkillCodes` | 本轮 `G − L` 遮蔽；只认成功且未 evict 的加载结果；异常 fail-open |
| 核心必读 | [McpToolBridge.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/McpToolBridge.java#L76) | `checkPermissions`；L84 `callAsync`；L114 `maskedBySkill` | ToolBase 权限 ASK/ALLOW；执行前按 RuntimeContext 遮蔽映射再次拒绝 |
| 核心必读 | [AgentToolCatalog.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalog.java#L164) | `toBinding`；L198 `McpToolBinding.of` | 多意图节点任一配置确认生效；`confirmConfigured || !Boolean.TRUE.equals(readOnlyHint)` |
| 核心必读 | [AgentStreamEventBridge.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentStreamEventBridge.java#L278) | `onRequireUserConfirm`；L305 `toConfirmCall`；L341 `settleAwaitingConfirm` | 工具参数展示、pending 卡片、`AWAITING_CONFIRM` 消息保存与 confirm 事件 |
| 核心必读 | [ConfirmRequest.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/controller/request/ConfirmRequest.java#L27)；[AgentChatController.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/controller/AgentChatController.java#L54) | `ConfirmRequest`、`confirm` | 入参只有 conversationId、messageId、approved；确认接口不接受工具参数 |
| 核心必读 | [AgentChatServiceImpl.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java#L161) | `startConfirmRun`；L187 `resolveConfirmResultsOrExpire`；L203 `askingToolCalls` | 从服务端最后 assistant 的 ASKING 工具恢复；无 ASKING 时将卡片 expired |
| 核心必读 | [AgentConversationServiceImpl.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentConversationServiceImpl.java#L165) | `getPendingConfirm`；L203 `settleConfirmBlock`；L220 `selectPendingConfirm` | 用户/会话/消息和 pending 校验；更新卡片与消息状态，使用普通 updateById |
| 核心必读 | [AgentChatServiceImpl.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java#L245)；[AgentRunHandle.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentRunHandle.java#L126) | `launchStream`、`settleConfirmCard`、`start` | 启动互斥段里结卡→订阅；结卡失败不订阅，仍有崩溃窗口 |
| 支撑 | [AgentSkillRegistry.java](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/skill/AgentSkillRegistry.java#L43) | `listEnabled`、`findByCode` | Redis 缓存 / 数据库读取启用技能，正文及 toolIds 一并提供 |
| 支撑 | [McpToolBridge.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/McpToolBridge.java#L132)；[McpClientToolExecutor.java](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/mcp/McpClientToolExecutor.java#L50) | `userId`、`execute`、`buildRequest` | 从 RuntimeContext 取登录身份，以 MCP `_meta` 透传；不是独立公网身份认证 |
| 支撑 | [AgentConfirmDenialMiddleware.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/confirm/AgentConfirmDenialMiddleware.java#L59) | `onReasoning`、`rewrite` | 将框架用户拒绝文案解释为“用户取消，操作没有执行”，保留调用 ID 与状态 |
| 支撑 | [RetrievalEngine.java](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/RetrievalEngine.java#L175) | `buildSubQuestionContext` | Agent 模式不走 RAG 内部 MCP 支路，避免绕过主 Agent 确认/技能治理 |
| 配置 / 契约 | [application.yaml](../bootstrap/src/main/resources/application.yaml#L35)；[McpToolAnnotations.java](../mcp-server/src/main/java/com/nageoffer/ai/ragent/mcp/config/McpToolAnnotations.java#L30) | Agent 模式；`READ_ONLY`、`WRITE` | 模式开关；服务端读写 annotation 与客户端业务确认配置是不同信息 |
| SQL | [260903_agent_skill.sql](../resources/database/upgrades/v2.0.0/260903_agent_skill.sql#L8)；[260830_agent_tool_confirm.sql](../resources/database/upgrades/v2.0.0/260830_agent_tool_confirm.sql#L7) | `t_agent_skill`、`require_confirm`、`message_status` | 技能正文/JSONB tool_ids；确认配置与等待确认状态；旧 SQL 注释不覆盖当前 Java 确认规则 |
| 静态测试 | [AgentChatServiceImplTest.java](../agent/src/test/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImplTest.java#L490) | `shouldSettleConfirmationAfterPreparationAndBeforeSubscription`；L506 `shouldNotSubscribeWhenConfirmationSettlementFails` | mock 确认续跑交接与结卡失败不订阅；没有本轮远端业务验证 |
| 静态测试 | [McpToolBridgeTest.java](../agent/src/test/java/com/nageoffer/ai/ragent/agent/tool/McpToolBridgeTest.java#L73) | `shouldPassLoginUserThroughMeta`、`shouldNeverFallBackToSingleArgumentOverload` | executor 捕获身份 metadata，不能证明公网身份可信或权限完备 |
| 静态测试 / 缺口 | [AgentToolCatalogTest.java](../agent/src/test/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalogTest.java#L156)；[同文件 L239](../agent/src/test/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalogTest.java#L239) | `shouldAskConfirmForDeclaredWriteToolWhenIntentNodeMisses`、`shouldCountReadOnlyAndConfirmSeparatelyIntoFingerprint` | 当前仍期待无只读声明工具 ALLOW、其确认值等于只读工具；与当前 Java 的保守 ASK 规则不一致，需修订测试；本次未运行 |
| 静态测试 / 对照 | [AgentScopeSkillActivationLifecycleTest.java](../agent/src/test/java/com/nageoffer/ai/ragent/agent/skill/AgentScopeSkillActivationLifecycleTest.java#L83) | `shouldKeepSkillToolActiveAfterLoadResultIsCompactedOut` | 原生 Skill 在手册压缩后仍激活的确定性对照，不证明当前自定义 Masking 已通过端到端回归 |

### 2.5 可追踪调用链

```text
按需加载与关联工具控制
ReActAgentProvider.buildAgent
  → AgentToolCatalog.buildToolkit（注册 load_skill、知识工具、MCP 桥）
  → SkillLoadTool.getDescription（技能清单）
  → 模型选择 load_skill → SkillLoadTool.execute
  → AgentSkillRegistry.findByCode → success（正文 + ragent_loaded_skill）
  → 工具结果进入上下文
  → 下一次推理：上下文压缩/裁剪先发生
  → AgentSkillMaskingMiddleware.onReasoning
  → loadedSkillCodes → maskedTools（G − L）
  → 遮蔽 schema，并将结果写本轮 RuntimeContext
  → McpToolBridge.callAsync → maskedBySkill 复查 → executor.execute

人工确认与恢复
AgentToolCatalog.McpToolBinding.of → needsConfirm
  → McpToolBridge.checkPermissions 返回 ASK
  → AgentScope 挂起 ASKING 工具并提供 RequireUserConfirmEvent
  → AgentStreamEventBridge：pending 卡片 → AWAITING_CONFIRM 消息 → SSE confirm
  → 用户提交 conversationId / messageId / approved
  → AgentChatController.confirm → AgentRunGate.acquire
  → getPendingConfirm（user / conversation / message / pending 校验）
  → resolveConfirmResultsOrExpire
  → Agent.getAgentState → 最后 assistant 消息中的 ASKING ToolUseBlock
  → 原 ToolUseBlock + approved → METADATA_CONFIRM_RESULTS
  → launchStream → AgentRunHandle.start
  → settleConfirmCard → subscribe
  → AgentScope 处理确认结果 → 批准后继续工具执行 / 拒绝后不执行
```

加载后的工具可见性仍由当前 `tool_ids` 映射决定；这里没有把所有工具都绑定到技能。确认目前是对本轮取出的 ASKING 调用列表统一批准或拒绝，不是逐个工具字段编辑。卡片展示参数不作为恢复执行的数据来源。

### 2.6 三个高频追问

**追问 1：为什么既遮蔽工具 schema，又在执行时检查？会不会把所有工具都锁住？**

schema 遮蔽减少模型误选，执行检查处理模型照着历史调用硬闯的情况，二者共用本轮 RuntimeContext 里的遮蔽映射。只限制启用技能 `tool_ids` 的并集；未关联任何技能的工具始终可见，共享工具加载任一关联手册即可解锁。手册加载结果必须成功且未被裁剪成占位，metadata 残留本身不够。当前遮蔽计算异常会直接放行原推理输入，是 fail-open；没有可靠映射就不能承诺执行兜底仍能拒绝，所以它是流程约束，业务服务还必须独立鉴权。

**追问 2：哪些工具需要确认？用户能不能批准时把金额或订单号改掉？**

规则是 `confirmConfigured || readOnlyHint != true`，其中 configured 来自同工具的意图节点任一确认配置。明确写工具、未声明只读的工具都会确认；明确只读工具也可因业务配置而确认。确认 DTO 只有会话 ID、消息 ID 和布尔决定，服务端先检查所属用户与 pending 状态，再从框架状态读取原 ASKING 调用，客户端不能通过这个接口换金额或订单号。业务数据若在等待期间变化，仍应由远端执行器检查资源状态和授权，不能把参数来源可信等同业务条件仍有效。

**追问 3：重复点确认是否一定不会重复执行？结卡成功后进程崩溃呢？**

正常链路由用户运行位挡住重叠请求，卡片终态校验挡住已处理卡片再次续跑，启动锁确保先结卡再订阅，结卡失败不订阅。卡片更新本身是普通 `updateById`，并非数据库 CAS；运行位 CAS 只用于释放原占位。PG 卡片、Agent 状态和 MCP 远程写入没有共同事务，结卡后崩溃可能尚未执行，远端成功后响应丢失也可能无法判断结果。因此不能保证 exactly-once；需要操作 ID、下游幂等结果与查询/对账，未知结果下不能盲目重新写入。

## 3. 文档自检与使用边界

本文只新增面试准备文档，没有修改业务源码、配置、SQL 或测试，没有新增依赖。核心引用来自本次实际读取的文件，测试条目描述静态断言与覆盖缺口。练习时先完整讲出连续口述，再按文件表解释关键方法；遇到效果追问，给出已有机制和实际报告，不用配置值代替吞吐、延迟或成本数字。
