# Ragent 简历第 3 条：混合检索 STAR 与源码索引

整理日期：2026-10-13（Asia/Shanghai）。核查源码基线：`4d536085`。用户已确认业务代码由本人独立完成，以下按独立设计与实现组织第一人称话术。场景说明采用设计需求，不编造线上事故、部署规模或性能实验。

## 原简历条目

> 构建可配置的向量、关键词与图谱混合检索链路，以加权 RRF 融合多路排名，串联 Rerank 和证据相关性门控，并拆分召回、精排候选及最终上下文预算。

原条目见[个性化简历第 3 条](ragent-resume-personalization.md#L27)。这里的“可配置”必须保留：当前仓库默认只启用向量通道；关键词、图谱与额外的联网通道均默认关闭。

## STAR 拆解

### S：设计场景

我在设计企业知识问答的检索模块时，需要处理语义描述、精确术语和关联知识三类查询需求。向量、关键词和图谱证据各有作用，但分数尺度不同；简单拼接候选会把重复证据和更多精排输入一起交给下游。另外，即使库里没有答案，相似度检索也可能返回一批“最像”的资料。这些是我的设计考虑，不代表实际发生过某次线上故障。

### T：我的任务

我要独立实现一条能按部署条件启用通道的检索链路：统一同一子问题的检索范围，融合不同检索器的排名，分别约束召回深度、精排候选量和上下文条数，并在精排后对证据批次做相关性判断。新增通道和修改预算应尽量不改动主流程。

### A：我的动作

我定义 `SearchChannel` 与 `SearchResultPostProcessor` 扩展点，在 `MultiChannelRetrievalEngine` 中统一解析作用域，并发派发启用通道，再按处理器顺序执行后处理。向量通道对接 pgvector，关键词通道对接 Elasticsearch，图谱通道接入 LightRAG；关键词和图谱分别通过后端装配条件与检索开关控制。

我先按 chunk key 去重，再从各通道原始排名中累加加权 RRF 分，保留同一证据被多路命中的投票信息。融合结果按候选预算截断后调用 Rerank，独立保留 `rerankScore` 给证据闸门使用。我将三个阶段的预算放进 `RetrievalBudget`，在配置启动时校验预算与最终条数的关系。最后，我按整批最高精排分决定批次去留，同时明确 noop 无分降级时放行，以及门控不等于逐条过滤。

### R：交付结果

我交付了可配置、多通道并发、排名融合、候选限量、模型精排和批级证据判断串联的检索模块。主流程使用统一接口接收通道结果，预算与通道开关分别配置；默认配置可以保持单向量运行，具备后端条件时再启用其他通道。相关配置、通道行为与门控分支有测试代码。这一结果是机制与代码交付，本次没有运行测试、真实模型或检索服务，也没有可用于宣称召回率、延迟或回答质量提升的实验记录。

## 90–120 秒连续口述稿

下面约 460 字，按正常语速讲述。数字均为配置值。

这条检索链路是我独立设计和实现的。我的设计场景是企业知识问答既有语义表达，也有精确术语和关联知识，单一路径不容易兼顾；同时，各检索器的分数尺度不同，直接拼接候选会增加重复证据和精排输入。我先把检索拆成通道和后处理两个扩展点，统一解析同一子问题的知识库范围，再并发执行已启用的向量、关键词和图谱通道。结果先按证据标识去重，但保留各路原始排名，再用加权RRF累加多路命中的贡献，这样不需要直接比较余弦分和关键词分。融合后只取候选池前段送给Rerank，并单独保存精排分供门控读取。预算方面，我把每通道召回、精排候选和最终上下文条数拆开，当前配置分别是二十、四十和十，避免一个topK牵动所有阶段。门控按整批最高精排分判断，低于阈值就丢弃整批；它不是逐条过滤，noop降级没有分数时会放行。我还加了预算和后端开关的启动校验，以及候选分布日志。最终交付的是能独立配置、排序和过滤证据的检索模块。默认只开启向量，其他通道需要后端和开关配齐，外部模型及检索引擎能力由相应服务提供。

## 核心源码文件

| 文件与定位 | 关键类 / 方法 | 在这条职责中的作用 |
| --- | --- | --- |
| [RetrievalEngine.java:87](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/RetrievalEngine.java#L87) | `RetrievalEngine.retrieve` | 一次构造三段预算，并行构建各子问题上下文。 |
| [RetrievalEngine.java:217](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/RetrievalEngine.java#L217) | `retrieveAndRerank` | 进入多通道引擎；接收存活证据和意图归属，再格式化知识上下文。 |
| [MultiChannelRetrievalEngine.java:80](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/MultiChannelRetrievalEngine.java#L80) | `retrieveKnowledgeChannels` | 串联上下文构建、通道执行、后处理与结果归属。 |
| [MultiChannelRetrievalEngine.java:132](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/MultiChannelRetrievalEngine.java#L132) | `executeSearchChannels` | 筛选启用通道，使用 `CompletableFuture` 与专用 Executor 并发执行。 |
| [MultiChannelRetrievalEngine.java:201](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/MultiChannelRetrievalEngine.java#L201) | `executePostProcessors` | 筛选处理器并按 order 排序；顺序执行，单个处理器异常时跳过。 |
| [MultiChannelRetrievalEngine.java:274](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/MultiChannelRetrievalEngine.java#L274) | `buildSearchContext` | 同一子问题的查询和作用域在这里统一产生，供通道只读。 |
| [VectorSearchChannel.java:74](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/channel/VectorSearchChannel.java#L74) | `VectorSearchChannel.search` | 根据统一作用域执行全局或定向向量召回。 |
| [KeywordSearchChannel.java:68](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/channel/KeywordSearchChannel.java#L68) | `KeywordSearchChannel.search` | 分配命中库与补充库配额，接收关键词后端结果。 |
| [GraphSearchChannel.java:76](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/channel/GraphSearchChannel.java#L76) | `GraphSearchChannel.search` | 调用 LightRAG，并按匹配/未匹配证据配额组装结果。 |
| [FusionPostProcessor.java:81](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/FusionPostProcessor.java#L81) | `FusionPostProcessor.process` | 多通道做 RRF；单通道保留原序；之后截断精排候选池。 |
| [FusionPostProcessor.java:100](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/FusionPostProcessor.java#L100) | `fuseByRrf` | 对原始通道排名按 `weight/(k+rank+1)` 累加，再回写融合分并排序。 |
| [RerankPostProcessor.java:64](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/RerankPostProcessor.java#L64) | `RerankPostProcessor.process` | 将融合候选及 `contextTopK` 交给精排服务，并记录精排分布与通道归因。 |
| [EvidenceGatePostProcessor.java:76](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/EvidenceGatePostProcessor.java#L76) | `EvidenceGatePostProcessor.process` | 批次最高有限精排分未过线则清空；没有可读精排分则放行。 |
| [RetrievalBudget.java:34](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/RetrievalBudget.java#L34) | `RetrievalBudget` | 三段不可变预算：`recallBudget`、`candidateLimit`、`contextTopK`。 |
| [SearchChannelProperties.java:86](../rag/src/main/java/com/nageoffer/ai/ragent/rag/config/SearchChannelProperties.java#L86) | `afterPropertiesSet` | 启动校验最终条数为正，以及有效召回/候选预算不少于最终条数等约束。 |

## 支撑源码文件与依赖边界

| 文件与定位 | 关键类 / 方法 | 作用及能力归属 |
| --- | --- | --- |
| [SearchChannel.java:32](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/channel/SearchChannel.java#L32) | `SearchChannel` | 我定义的检索通道扩展接口；检索引擎依赖接口而非各后端。 |
| [SearchResultPostProcessor.java:37](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/SearchResultPostProcessor.java#L37) | `SearchResultPostProcessor` | 我定义的后处理扩展接口与顺序契约。 |
| [RetrievalScopeResolver.java:54](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/channel/RetrievalScopeResolver.java#L54) | `resolve` | 低置信全局、高置信定向；有效库与意图绑定求交，并产生补充库范围。 |
| [DeduplicationPostProcessor.java:56](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/DeduplicationPostProcessor.java#L56) | `process` | 按 chunk key 保留首次证据对象；原始通道列表仍供 RRF 读取。 |
| [RetrievedChunkKey.java:28](../framework/src/main/java/com/nageoffer/ai/ragent/framework/convention/RetrievedChunkKey.java#L28) | `RetrievedChunkKey.of` | 优先 ID，缺 ID 时用全文 SHA-256；这是确定性去重，不是语义去重。 |
| [PgVectorRetrieverService.java:71](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/vector/PgVectorRetrieverService.java#L71) | `queryByCollections` | 我编写 SQL 与库范围适配；余弦距离运算和 HNSW 检索由 PostgreSQL/pgvector 提供，embedding 由模型服务提供。 |
| [EsKeywordRetrieverService.java:57](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/keyword/EsKeywordRetrieverService.java#L57) | `search` | 我实现 content match 查询和 collection filter 接入；分词、倒排检索与打分来自 Elasticsearch。 |
| [LightRagClient.java:87](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/graph/LightRagClient.java#L87) | `retrieveByScope` | 我实现 LightRAG 请求与证据解析适配；图谱检索算法由外部 LightRAG 服务提供。 |
| [RoutingRerankService.java:54](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/rerank/RoutingRerankService.java#L54) | `rerank` | 我实现客户端路由与候选失败降级接入。 |
| [BaiLianRerankClient.java:63](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/rerank/BaiLianRerankClient.java#L63) | `rerank` | 我实现供应商 HTTP 适配；候选少于 topN 仍调用模型以取得精排分。模型推理来自供应商服务。 |
| [BaiLianRerankClient.java:160](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/rerank/BaiLianRerankClient.java#L160) | `doRerank` 的回填段 | 将供应商相关性分同时写入 score 与 rerankScore，保留供门控使用的分数。 |
| [NoopRerankClient.java:37](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/rerank/NoopRerankClient.java#L37) | `rerank` | 后备客户端只按条数截断，不生成真实精排分。 |
| [MetadataEnrichmentPostProcessor.java:66](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/MetadataEnrichmentPostProcessor.java#L66) | `process` | 对存活证据补齐文档 ID、序号、标题，保持相关性顺序。 |
| [DefaultContextFormatter.java:51](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/prompt/DefaultContextFormatter.java#L51) | `formatKbContext` | 将存活证据按意图分支和文档组装上下文。 |
| [RetrievalChannelConfigValidator.java:72](../rag/src/main/java/com/nageoffer/ai/ragent/rag/config/validation/RetrievalChannelConfigValidator.java#L72) | `SPECS` / `validate` | 检查关键词/图谱后端未装配却打开检索开关的矛盾配置。 |

Spring 的容器装配、Java 的异步执行、HTTP 客户端及后端检索/模型推理属于依赖能力；我的职责是业务检索接口、查询适配、编排、预算、融合、门控、降级及测试。RRF 是已有算法，我实现了它在本项目中的加权版本，不称为发明算法。

## 调用顺序与关键分支

1. `RetrievalEngine.retrieve` 用配置创建预算，将每个 `SubQuestionIntent` 派发给 `buildSubQuestionContext`，经 `retrieveAndRerank` 调用多通道引擎。
2. `retrieveKnowledgeChannels` 先调用 `buildSearchContext`；`RetrievalScopeResolver.resolve` 解析一次作用域，避免同一子问题各通道独立判断库范围。
3. `executeSearchChannels` 只执行已装配且已启用的通道，并发取得 `SearchChannelResult`。`withTimeout` 用超时异常转空结果，放弃该通道结果，不会中断底层任务：[MultiChannelRetrievalEngine.java:251](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/MultiChannelRetrievalEngine.java#L251)。
4. `executePostProcessors` 按 order 串联 `Deduplication(1) → Fusion(5) → Rerank(10) → EvidenceGate(15) → MetadataEnrichment(20)`，各处理器仍受自身开关控制。Fusion 单通道跳过排名融合；候选限制仍执行。
5. 引擎按最终存活 chunk 的库归属计算意图归属，返回 `KnowledgeRetrievalResult`；`RetrievalEngine.retrieveAndRerank` 再格式化上下文。[deriveAttribution:104](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/MultiChannelRetrievalEngine.java#L104)

注意：后处理器异常会跳过而继续，因此不能宣称所有失败都严格阻止证据进入上下文。`contextTopK` 在单子问题检索与格式化中控制条数；多个子问题汇总时不是整个回答的统一 Token 硬上限。

## 配置值与默认行为

| 配置 | 当前值 / 行为 | 源码定位 |
| --- | --- | --- |
| 向量后端 | `rag.vector.type=pg` | [application.yaml:85](../bootstrap/src/main/resources/application.yaml#L85) |
| 关键词 / 图谱后端 | `none` / `none`；启用需分别设为 `es` / `lightrag` | [application.yaml:88](../bootstrap/src/main/resources/application.yaml#L88) |
| 检索通道 | 向量 true，关键词/图谱/联网 false | [application.yaml:162](../bootstrap/src/main/resources/application.yaml#L162) |
| 三段预算 | 每通道召回基数20，融合后候选上限40，最终上下文条数10 | [application.yaml:152](../bootstrap/src/main/resources/application.yaml#L152) |
| RRF | strategy=rrf，k=20；向量1.0、关键词1.0、图谱0.8、联网0.5 | [application.yaml:175](../bootstrap/src/main/resources/application.yaml#L175) |
| 精排与后备 | enabled=true；qwen3-rerank 优先，noop 后备 | [application.yaml:112](../bootstrap/src/main/resources/application.yaml#L112)、[application.yaml:293](../bootstrap/src/main/resources/application.yaml#L293) |
| 门控 | `min-rerank-score=0.2`；0关闭，最高有限精排分低于门槛整批丢弃 | [application.yaml:184](../bootstrap/src/main/resources/application.yaml#L184) |
| 作用域 | 最低意图分0.4，收窄置信阈值0.6，补充比例0.25 | [application.yaml:158](../bootstrap/src/main/resources/application.yaml#L158) |
| 通道超时 | 15000毫秒，超时结果降为空 | [application.yaml:163](../bootstrap/src/main/resources/application.yaml#L163) |

三段预算并不是 `20 ≥ 40 ≥ 10` 的单路数值漏斗；20是每通道基数，多通道汇总后才由40限制候选池。启动校验要求有效 recall 不低于 contextTopK，且正数 candidateLimit 不低于 contextTopK；candidateLimit<=0 不截断。[SearchChannelProperties.java:86](../rag/src/main/java/com/nageoffer/ai/ragent/rag/config/SearchChannelProperties.java#L86)

还有两处召回细节：向量定向深度可被意图节点 topK 覆盖，再受候选上限钳制：[VectorSearchChannel.java:145](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/channel/VectorSearchChannel.java#L145)；图谱定向请求为对冲结果侧过滤，会将请求 topK 放大后再按配额截回：[GraphSearchChannel.java:88](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/channel/GraphSearchChannel.java#L88)。因此“20”也不是所有底层请求的固定条数。

这些配置来自设计取值，当前材料没有消融结果证明它们是最优参数。LightRAG 当前是单实例全图查询后按 file_path 分库归属，不能称为物理子图隔离：[GraphSearchChannel.java:41](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/channel/GraphSearchChannel.java#L41)。

## 相关测试与本次验证范围

下面均为重新读取的测试代码，本次未运行 Maven/JUnit、数据库、Elasticsearch、LightRAG 或模型服务。不要把测试存在改写成“本次测试全部通过”。

| 测试文件 | 已有覆盖点 |
| --- | --- |
| [SearchChannelPropertiesTest.java:35](../rag/src/test/java/com/nageoffer/ai/ragent/rag/config/SearchChannelPropertiesTest.java#L35) | 召回回退、预算不变式、无限候选、作用域阈值及门控上限的配置校验。 |
| [VectorSearchChannelTest.java:72](../rag/src/test/java/com/nageoffer/ai/ragent/rag/core/retrieval/channel/VectorSearchChannelTest.java#L72) | 补充配额、意图 topK 覆盖与钳制、共享 query vector、去重库范围、补充失败保主路及通道出口排序。 |
| [KeywordSearchChannelTest.java:75](../rag/src/test/java/com/nageoffer/ai/ragent/rag/core/retrieval/channel/KeywordSearchChannelTest.java#L75) | 定向/全局配额、补充失败保主路、空作用域、失败转空结果。 |
| [GraphSearchChannelTest.java:65](../rag/src/test/java/com/nageoffer/ai/ragent/rag/core/retrieval/channel/GraphSearchChannelTest.java#L65) | 图谱排名合并、两侧配额截断、有效库过滤和定向请求深度放大。 |
| [DeduplicationPostProcessorTest.java:61](../rag/src/test/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/DeduplicationPostProcessorTest.java#L61) | 字符串 hash 碰撞不会误合并；无分证据保留首次出现。 |
| [BaiLianRerankClientTest.java:72](../infra-ai/src/test/java/com/nageoffer/ai/ragent/infra/rerank/BaiLianRerankClientTest.java#L72) | 少候选仍请求模型、score/rerankScore双字段、回填和缺失相关性分处理。使用本地 HTTP stub，不证明真实供应商模型效果。 |
| [EvidenceGatePostProcessorTest.java:65](../rag/src/test/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/EvidenceGatePostProcessorTest.java#L65) | 无分放行、最高分门槛、低分整批丢弃、边界值、非有限分以及门控开但精排关的启动拒绝。 |
| [MultiChannelRetrievalEngineTest.java:55](../rag/src/test/java/com/nageoffer/ai/ragent/rag/core/retrieval/MultiChannelRetrievalEngineTest.java#L55) | 最终证据库归属、多意图共享库、全局无归属和定向超时保留范围。 |

当前测试文件清单未见专门的 `FusionPostProcessorTest`；已有通道/引擎测试不能替代加权 RRF 的手算分值回归。当前材料没有同语料消融、阈值校准或性能对照结果，结果表述停留在机制交付。

## 三个高频追问

### 1. 为什么用 RRF？去重后如何保留多路投票？

我选择排名融合，是因为不同检索器的原始分数尺度不统一，直接相加需要额外校准。我的实现按通道原始名次累加 `w/(k+rank+1)`；去重只产生最终代表对象列表，原始 `SearchChannelResult` 还在，所以同一 key 的多路命中贡献不会丢失。代价是丢弃绝对分差，权重和 k 仍要用同一批标注语料验证。k=20与图谱权重0.8是当前配置，不是已证明最优。[fuseByRrf:100](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/FusionPostProcessor.java#L100)

### 2. 为什么预算是20、40、10？“最终上下文预算”就是 Token 上限吗？

我把三个不同问题分开：每路召回需要多深、精排最多处理多少候选、每个子问题保留多少证据。多通道可合计超过20，所以候选上限40不与召回20矛盾。三段可以独立调整，并在启动时防止召回或正数候选上限少于最终条数。这里按证据条数计量，证据长度和子问题数量会变化，因此它不是全回答的严格 Token 上限；这组值也没有消融证明最优。[预算构造:94](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/RetrievalEngine.java#L94)、[预算校验:86](../rag/src/main/java/com/nageoffer/ai/ragent/rag/config/SearchChannelProperties.java#L86)

### 3. 门控为什么看最高精排分？noop 时怎么处理？能保证不出现幻觉吗？

我实现的是批级去留策略：整批连最高分都没达到门槛，就认为缺少足够相关的证据，清空这一批；最高分过线时保留整批，不逐条过滤。读取独立 `rerankScore` 可以避免把 RRF 分当精排相关性分。noop 只截断不出分，当前实现缺分会告警并放行，选择保留可用证据的降级路径。门槛依赖具体 reranker，应通过标注集校准；而且后处理器异常会被跳过，这些机制不能保证答案一定可靠，也不等于幻觉率已下降。[门控:76](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/EvidenceGatePostProcessor.java#L76)、[noop:37](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/rerank/NoopRerankClient.java#L37)

## 交付说明

本文件只补充简历第 3 条的第一人称 STAR、口述稿与源码定位，没有修改业务代码、配置或测试，没有新增依赖。使用时可直接口述主体，追问时按源码索引展开；涉及项目运行、真实模型效果或线上指标的回答仍需实际记录支持。
