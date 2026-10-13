# Ragent 个性化简历证据核验

核验日期：2026-10-13（Asia/Shanghai）。源码基线：`4667c0bb`。本文件记录模板表述与当前工作区第一手证据的对应关系，供个性化简历与面试准备使用。

## 1. 个人贡献与验证边界

个人贡献：Ragent 全部业务代码由本人独立完成。本文将应用层设计、编码与第三方框架能力分开说明，个人职责使用独立设计实现口径。[贡献说明](source-interview-phase-one.md)与[逐条 STAR 话术](ragent-star-interview-guide.md)采用同一口径。

个人职责中的设计与实现机制已对应源码；性能、质量提升和生产故障经历需要单独记录。本次核验只读取源码、配置和测试文件，没有运行模型、数据库、外部检索服务、端到端评测或生产压测。测试文件证明断言存在，不证明本次运行通过。

## 2. 检索链路

### 2.1 已验证的实现与默认配置

| 模板中的能力 | 当前源码事实 | 简历表述边界 |
| --- | --- | --- |
| 三路混合检索 | 存在向量、关键词、图谱和联网四类通道实现；引擎筛选启用通道后并发执行。 | 可说“构建可配置的多通道检索链路”；不能说默认三路或四路都已部署运行。 |
| 加权 RRF | 使用各通道原始列表中的排名，按 `weight / (k + rank + 1)` 对同一 chunk key 累加；去重后仍保留多路命中投票。 | 可分析分值量纲差异与排名融合；没有同数据消融结果，不能宣称效果优于其他策略。 |
| Rerank | 候选池传给 `RerankService`，按上下文条数预算输出；存在 noop 后备。 | 可分析粗排与精排分工；不能称为自行训练重排模型，或保证每次都由真实重排模型出分。 |
| 拆分检索预算 | `RetrievalBudget` 分离每通道 recall、融合后 candidate limit 和最终 context topK。 | 能说明各预算控制的阶段；条数预算不等于严格 Token 预算或实测 Token 降幅。 |
| 证据相关性闸门 | 读取整批最高有限 `rerankScore`；最高分不足时整批清空，缺分时放行。 | 可讲解无答案场景的批级过滤及降级边界；不能声称逐条过滤噪声或已校准回答可靠性。 |
| 意图定向与补充召回 | KB 意图置信不足或绑定库全部失效时走全局；有效绑定与当前有效库求交，定向时保留未命中库补充范围。 | 可分析路由误判与失效配置的处理；这些范围策略不等于完整的租户授权体系。 |

第一手源码：

- 向量通道：[VectorSearchChannel.java:68](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/channel/VectorSearchChannel.java#L68)。
- Elasticsearch 关键词通道：[KeywordSearchChannel.java:63](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/channel/KeywordSearchChannel.java#L63)。
- LightRAG 图谱通道需后端条件装配：[GraphSearchChannel.java:47](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/channel/GraphSearchChannel.java#L47)。
- 联网通道需要开关与 API Key：[WebSearchChannel.java:97](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/channel/WebSearchChannel.java#L97)。
- 启用通道筛选与并发派发：[MultiChannelRetrievalEngine.java:132](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/MultiChannelRetrievalEngine.java#L132)。
- 后处理器按 order 执行；异常处理器会被跳过：[MultiChannelRetrievalEngine.java:201](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/MultiChannelRetrievalEngine.java#L201)。
- 单通道跳过融合，仅截断：[FusionPostProcessor.java:88](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/FusionPostProcessor.java#L88)。
- 加权 RRF 算式与回写：[FusionPostProcessor.java:100](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/FusionPostProcessor.java#L100)。
- 精排及最终条数：[RerankPostProcessor.java:64](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/RerankPostProcessor.java#L64)。
- 三段预算定义：[RetrievalBudget.java:20](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/RetrievalBudget.java#L20)。
- 批级证据闸门及缺分放行：[EvidenceGatePostProcessor.java:75](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/EvidenceGatePostProcessor.java#L75)。
- noop 只截断不打分：[NoopRerankClient.java:37](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/rerank/NoopRerankClient.java#L37)。
- 全局/定向及失效库处理：[RetrievalScopeResolver.java:54](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/channel/RetrievalScopeResolver.java#L54)。

当前仓库默认配置只启用向量通道，关键词、图谱、联网开关均为 false；每通道 recall=20、融合候选上限=40、最终 context topK=10，RRF k=20，权重为向量1.0、关键词1.0、图谱0.8、联网0.5，证据闸门最低精排分0.2。[application.yaml:152](../bootstrap/src/main/resources/application.yaml#L152)

这些数值是配置值。由于 recall 是每通道条数，不能将配置解释成“20 ≥ 40 ≥ 10”的单路单调漏斗。关键词与图谱实际参与还需要后端装配和通道启用同时满足：[RetrievalChannelConfigValidator.java:72](../rag/src/main/java/com/nageoffer/ai/ragent/rag/config/validation/RetrievalChannelConfigValidator.java#L72)。

通道超时以空结果降级，但不会中断底层任务；不能把超时处理描述为完整资源取消：[MultiChannelRetrievalEngine.java:244](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/MultiChannelRetrievalEngine.java#L244)。

### 2.2 测试证据

| 测试文件 | 可核验的断言范围 | 不支持的结论 |
| --- | --- | --- |
| [RetrievalScopeResolverTest.java:41](../rag/src/test/java/com/nageoffer/ai/ragent/rag/core/retrieval/channel/RetrievalScopeResolverTest.java#L41) | 高置信收窄、低置信全局、失效绑定、补充库与意图去重。 | 不能证明分类模型置信度已校准或真实语料召回率改善。 |
| [MultiChannelRetrievalEngineTest.java:54](../rag/src/test/java/com/nageoffer/ai/ragent/rag/core/retrieval/MultiChannelRetrievalEngineTest.java#L54) | 存活 chunk 按库归属、多意图共享库、全局无归属、定向超时降级。 | 不能证明外部服务联调、吞吐或生产稳定性。 |
| [EvidenceGatePostProcessorTest.java:63](../rag/src/test/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/EvidenceGatePostProcessorTest.java#L63) | 无分放行、低分整批清空、批次最高分、阈值边界和非法配置。 | 不能证明真实模型阈值0.2有效，或无答案识别准确率。 |
| [WebSearchChannelLiveTest.java:37](../rag/src/test/java/com/nageoffer/ai/ragent/rag/core/retrieval/channel/WebSearchChannelLiveTest.java#L37) | 有外部 API Key 时才启用的联网检索测试入口。 | 未提供本次执行结果，不能称为当前已联调通过。 |

当前文件清单未见专门的 `FusionPostProcessorTest`。不能宣称已完成完整的 RRF 定量算法回归或检索消融实验。

## 3. 评测与数值声明

### 3.1 已验证的评测出口

`/rag/eval` 在 `ragent.eval.enabled=true` 时注册，执行“无历史的改写拆分 → 意图解析 → 检索”，记录这一段的总耗时。它没有生成最终回答，也没有计算 RAGAS 指标：[EvalController.java:53](../rag/src/main/java/com/nageoffer/ai/ragent/rag/eval/EvalController.java#L53)、[EvalController.java:62](../rag/src/main/java/com/nageoffer/ai/ragent/rag/eval/EvalController.java#L62)。

输出包含召回业务文档 ID、chunk ID、上下文、与上下文逐项对齐的文档 ID、MCP/KB 标记、子问题、意图叶子 ID和耗时；业务文档 ID通过 chunk→文档→去扩展名的 doc_name 对齐。它是为外部评测准备的检索证据接口：[EvalResponse.java:26](../rag/src/main/java/com/nageoffer/ai/ragent/rag/eval/EvalResponse.java#L26)、[EvalController.java:118](../rag/src/main/java/com/nageoffer/ai/ragent/rag/eval/EvalController.java#L118)。

字段注释提及外部脚本可能计算 context precision、context recall 和意图 Top-1，但注释不证明相应脚本或结果存在。[EvalResponse.java:47](../rag/src/main/java/com/nageoffer/ai/ragent/rag/eval/EvalResponse.java#L47)、[EvalResponse.java:73](../rag/src/main/java/com/nageoffer/ai/ragent/rag/eval/EvalResponse.java#L73)

### 3.2 未找到依据的模板内容

已检索源码、配置、测试、资源和包含忽略文件的工作区文件清单；没有找到以下声明对应的可运行脚本、依赖、评测数据与结果报告。仅有备份回答提示词中的“RAGAS 友好回答原则”文字：[answer-chat-kb-bitmall-v2.st:865](../rag/src/main/resources/prompt/buckup/answer-chat-kb-bitmall-v2.st#L865)，它不构成评测实现。

| 模板声明 | 当前结论 | 改写要求 |
| --- | --- | --- |
| “搭建基于 RAGAS 的端到端 RAG 评测体系” | 只确认检索证据接口；当前工作区未见 RAGAS 执行脚本、依赖或回答质量结果。 | 暂不写已经搭建或运行 RAGAS；可写“提供检索证据接口与文档对齐字段”。 |
| “覆盖20+指标” | 未见指标注册、计算逻辑或20+指标清单的执行结果。 | 删除指标数量。 |
| “支持 A/B Diff 与人工校正” | 未见相应程序、人工标注或对比报告。 | 删除已实现、已运行表述。 |
| “Faithfulness +10.3%” | 未见基线、对照、样本、判分模型、原始结果或统计口径。 | 删除提升数值，不把模板数值当项目实测。 |
| “Answer Relevancy +8.0%” | 同上；不能区分相对百分比与百分点。 | 删除提升数值。 |
| “持续验证检索准确性与回答可靠性” | 未见持续评测任务或结果序列。 | 保留检索证据出口，不宣称已持续验证回答质量。 |

上述“未见”仅说明本次可访问工作区中缺少证据，不断言作者或外部仓库从未做过评测。后续只有取得可复现的脚本、固定样本、模型配置、基线与结果后，才能追加真实的个人实验经历。

## 4. 最终五条职责与证据映射

| 职责方向 | 核心代码 | 可写行为及边界 |
| --- | --- | --- |
| Agent 执行与流生命周期 | [ReActAgentProvider:113](../agent/src/main/java/com/nageoffer/ai/ragent/agent/config/ReActAgentProvider.java#L113)、[PgAgentStateStore:48](../agent/src/main/java/com/nageoffer/ai/ragent/agent/state/PgAgentStateStore.java#L48)、[AgentRunGate:53](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentRunGate.java#L53)、[StreamTaskManager:146](../framework/src/main/java/com/nageoffer/ai/ragent/framework/web/StreamTaskManager.java#L146) | 统一工具装配、状态保存、运行位与跨节点取消；ReAct 主循环由 AgentScope 提供，远程业务成功不能由取消协议保证 |
| Skills 与人工确认 | [AgentSkillMaskingMiddleware:79](../agent/src/main/java/com/nageoffer/ai/ragent/agent/skill/AgentSkillMaskingMiddleware.java#L79)、[McpToolBridge:84](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/McpToolBridge.java#L84)、[AgentChatServiceImpl:185](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java#L185) | 工具可见性与执行双校验、服务端参数恢复；不写完整权限系统或远程业务 exactly-once |
| 混合检索 | [MultiChannelRetrievalEngine:132](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/MultiChannelRetrievalEngine.java#L132)、[FusionPostProcessor:100](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/FusionPostProcessor.java#L100)、[EvidenceGatePostProcessor:75](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/EvidenceGatePostProcessor.java#L75) | 可配置多通道、加权 RRF、精排与批级门控；不声称默认多路开启或已有准确率改善 |
| 分层记忆与并发提交 | [AgentContextTrimmer:68](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentContextTrimmer.java#L68)、[AgentContextCompactor:69](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentContextCompactor.java#L69)、[AgentMemoryPipeline:94](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryPipeline.java#L94)、[AgentMemoryRepository:215](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryRepository.java#L215) | 工具裁剪、摘要、事实记忆和 revision / watermark 校验；字符预算不代表 Token 成本实测 |
| RAG 模型路由与流式容错 | [ModelSelector:77](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/model/ModelSelector.java#L77)、[ModelHealthStore:63](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/model/ModelHealthStore.java#L63)、[RoutingLLMService:115](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/chat/RoutingLLMService.java#L115)、[ProbeStreamBridge:71](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/chat/ProbeStreamBridge.java#L71) | 档位候选、三态熔断、首包探测和提交；主 Agent 单模型无此 fallback，首包成功后没有继续跨模型重试 |

## 5. Agent、Skills、HITL 与记忆

ReAct 由 AgentScope 框架提供，项目代码负责模型、工具、技能、记忆与 SSE 生命周期装配。[ReActAgentProvider:113](../agent/src/main/java/com/nageoffer/ai/ragent/agent/config/ReActAgentProvider.java#L113)

Skills 以成功的 `load_skill` 工具结果元数据识别已加载手册。中间件在推理时遮蔽关联工具，`McpToolBridge` 在执行前再次检查；手册退出上下文后重新收回工具。限制仅覆盖技能 `tool_ids` 所关联工具；遮蔽计算异常时会 fail-open。[SkillLoadTool:113](../agent/src/main/java/com/nageoffer/ai/ragent/agent/skill/SkillLoadTool.java#L113)、[AgentSkillMaskingMiddleware:79](../agent/src/main/java/com/nageoffer/ai/ragent/agent/skill/AgentSkillMaskingMiddleware.java#L79)、[McpToolBridge:114](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/McpToolBridge.java#L114)

确认条件为业务节点任一 `requireConfirm=true`，或服务端未明确 `readOnlyHint=true`。明确只读工具也可因业务配置触发确认。[AgentToolCatalog:175](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalog.java#L175)、[AgentToolCatalog:198](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalog.java#L198)

确认请求只提交会话 ID、消息 ID 和批准布尔值；恢复入参来自服务端最后一条 assistant 消息中的待确认 `ToolUseBlock`，客户端不能通过确认接口替换业务参数。确认卡片结算为普通数据库更新，没有业务状态 CAS。[ConfirmRequest:27](../agent/src/main/java/com/nageoffer/ai/ragent/agent/controller/request/ConfirmRequest.java#L27)、[AgentChatServiceImpl:185](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java#L185)、[AgentConversationServiceImpl:203](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentConversationServiceImpl.java#L203)

上下文预算按字符计算，当前配置为 100000；工具裁剪只对白名单旧结果进行替换，当前白名单只有 `search_knowledge`，近期轮次和未闭合调用受到保护。摘要按用户轮次切分，对孤立工具结果进行配对保护，不能理解成完整保存所有历史工具输出。[application.yaml:50](../bootstrap/src/main/resources/application.yaml#L50)、[AgentContextTrimmer:68](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentContextTrimmer.java#L68)、[AgentContextCompactor:152](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentContextCompactor.java#L152)

长期记忆 Judge 与容量合并计划在事务外完成。提交时锁用户控制行，并校验用户 revision 与会话 watermark；不匹配则拒绝旧结果。没有记忆集变更的 NOOP 不推进 revision，但会推进 watermark，因此需要双校验；合并、淘汰等记忆集变更仍推进 revision。该机制不证明事实抽取语义正确或数据库事务时长已降低。[AgentMemoryPipeline:94](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryPipeline.java#L94)、[AgentMemoryRepository:215](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryRepository.java#L215)、[AgentMemoryControlMapper:54](../agent/src/main/java/com/nageoffer/ai/ragent/agent/dao/mapper/AgentMemoryControlMapper.java#L54)

## 6. 执行控制、异步处理与工程设施

用户运行位用 Redis `setIfAbsent` 领取，以本次值 CAS 释放。跨节点取消先写取消标记，再通过 Pub/Sub 广播；任务注册时检查预埋标记，本地 AtomicBoolean 防重复执行取消回调。它治理任务生命周期，不能撤销已经完成的 MCP 写操作。[AgentRunGate:53](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentRunGate.java#L53)、[StreamTaskManager:146](../framework/src/main/java/com/nageoffer/ai/ragent/framework/web/StreamTaskManager.java#L146)、[StreamTaskManager:203](../framework/src/main/java/com/nageoffer/ai/ragent/framework/web/StreamTaskManager.java#L203)

SSE 的超时、错误和完成回调均接取消流程；运行句柄协调订阅、中断、状态保存及释放钩子。网络断开后的取消依赖容器检测，不能保证同步外部操作瞬时终止。[AgentChatServiceImpl:364](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java#L364)、[AgentRunHandle:169](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentRunHandle.java#L169)

RAG 模型调用先筛选档位候选，结合 CLOSED / OPEN / HALF_OPEN 状态进行准入。首包探测结果为成功时提交缓冲；探测结果为超时、错误或无内容完成时尝试下个候选。内容或思考回调先将探测置为成功后，迟到错误不会覆盖成功结果，即使先于 commit 到达也沿当前流转发。首包等待被中断时取消句柄、释放匹配的半开令牌；探测成功后不再切换模型。[RoutingLLMService:124](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/chat/RoutingLLMService.java#L124)、[RoutingLLMService:182](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/chat/RoutingLLMService.java#L182)、[ModelHealthStore:143](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/model/ModelHealthStore.java#L143)、[ProbeStreamBridge:44](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/chat/ProbeStreamBridge.java#L44)

这套 fallback 仅属于 RAG / infra-ai 路径。主 Agent 配置单模型，当前 Agent Langfuse 追踪关闭；RAG 自身追踪开关独立，不据此宣称生产全链路已经完成验证。[application.yaml:42](../bootstrap/src/main/resources/application.yaml#L42)、[application.yaml:55](../bootstrap/src/main/resources/application.yaml#L55)

其他模板能力可确认源码存在，但不纳入五条正文：RocketMQ 事务消息以文档 RUNNING 状态回查，线程池以 `TtlExecutors` 包装，定时刷新有租约、心跳及内容未变化跳过逻辑。[KnowledgeDocumentServiceImpl:207](../rag/src/main/java/com/nageoffer/ai/ragent/knowledge/service/impl/KnowledgeDocumentServiceImpl.java#L207)、[KnowledgeDocumentChunkTransactionChecker:62](../rag/src/main/java/com/nageoffer/ai/ragent/knowledge/mq/KnowledgeDocumentChunkTransactionChecker.java#L62)、[ThreadPoolExecutorConfig:59](../rag/src/main/java/com/nageoffer/ai/ragent/rag/config/ThreadPoolExecutorConfig.java#L59)、[ScheduleLockManager:70](../rag/src/main/java/com/nageoffer/ai/ragent/knowledge/schedule/ScheduleLockManager.java#L70)、[ScheduleRefreshProcessor:159](../rag/src/main/java/com/nageoffer/ai/ragent/knowledge/schedule/ScheduleRefreshProcessor.java#L159)

## 7. 本次交付自检

本次仅修改文档，没有新增依赖，也没有修改源码、配置或测试。自检包含 Markdown 目标存在性、源码行号范围、简历五条职责及差异格式。业务运行无需手动操作；投递前填写真实项目时间。

静态阅读还发现 `AgentToolCatalogTest` 对未声明只读属性工具的确认断言与当前实现不一致；本文不据测试文件宣称当前测试通过，未在本次文档任务中修复或运行该组测试。[当前实现:209](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalog.java#L209)、[旧断言:165](../agent/src/test/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalogTest.java#L165)、[旧断言:244](../agent/src/test/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalogTest.java#L244)

本次交付和 Git 记录见[简历文档](ragent-resume-personalization.md#交付与使用记录)。
