# Ragent 简历面试问题与准备重点

整理日期：2026-10-13（Asia/Shanghai）。材料基线：`45baf7ef566f198574d45b6ed77dbd3eacd74408`。对应[当前简历](ragent-resume-personalization.md)的五条职责，业务代码由本人独立完成。

本清单是针对简历与实现的面试预测，不代表实际面试频率统计。P0 表示建议先练熟，P1 表示后续展开。回答检查点用于检查完整性；连续口述稿和源码文件表见[STAR 总览](ragent-star-interview-guide.md)。

## 准备计划

按简历逐条映射面试问题，重点覆盖设计原因、具体代码、失败时序及验证方法；先准备 P0，再沿追问链深入。每个答案应能从入口讲到关键方法，并明确实际实现、当前局限和后续改进。

本次只新增问题清单，不改业务代码、配置或依赖。问题与回答检查点沿当前源码及本轮已核验的 STAR 资料整理；未将运行配置换算成性能成果。

## 1. 项目整体：开场会问什么

| 编号 | 优先级 | 面试问题 | 回答检查点 |
| --- | --- | --- | --- |
| G1 | P0 | 用两分钟介绍项目：服务什么场景，主要解决哪些问题？ | 企业知识问答与业务工具调用；说明典型任务、平台职责和五个工程问题，规模按真实经历回答 |
| G2 | P0 | 从用户输入到 SSE 输出，完整调用链怎么走？ | Controller → 运行控制 → Provider → ReAct → 知识/MCP 工具 → 事件桥；能画出状态、确认与记忆的位置 |
| G3 | P0 | 为什么同时保留 Workflow 和 ReAct？什么任务适合哪一种？ | 固定流程与动态工具决策的场景差异，确定性、调用次数、调试与迭代约束；结合实际入口解释 |
| G4 | P0 | 你独立开发时最难的设计问题是什么，如何定位和解决？ | 选真实经历，讲触发、定位证据、改动和验证；可围绕启动取消竞争、待确认恢复或过期记忆提交展开 |
| G5 | P1 | 为什么选择 Java / Spring Boot / AgentScope？换框架要改哪些部分？ | 结合既有技术栈与实际开发约束；框架循环、状态 codec、工具契约、业务生命周期各自的边界 |
| G6 | P1 | 如果并发增长十倍，你会先测和改哪里？ | 模型与外部 I/O、线程池排队、Redis/PG、检索候选、限流；先定位瓶颈，再提出改动和验收方法 |

## 2. 第一条：Agent 执行、状态恢复与流生命周期

源码与完整话术：[Agent 执行详细稿](ragent-star-agent-tools.md#point-1)。

| 编号 | 优先级 | 面试问题 | 回答检查点 |
| --- | --- | --- | --- |
| A1 | P0 | AgentScope 已提供 ReAct，你自己实现了哪些部分？ | 工具目录、知识/MCP 桥、PG 状态适配、运行位、SSE 和生命周期；框架推理循环单独说明 |
| A2 | P0 | PG 保存了什么状态？为什么不能只保存聊天消息？ | AgentScope codec JSON、用户/会话/状态键；业务消息与框架状态分表，二者不共用原子提交 |
| A3 | P0 | 为什么按用户抢运行位？SETNX、CAS、TTL 各解决什么？TTL 先过期怎么办？ | SETNX 管准入，CAS 管释放归属，TTL 管崩溃兜底；固定 TTL 无续租，过期不停止旧任务，也不阻止迟到业务写入 |
| A4 | P0 | 停止请求落到其他节点，或早于任务注册，如何处理？ | Redis 属主与取消标记、标记后广播、注册复核、执行端属主校验；Pub/Sub 不是可靠投递队列 |
| A5 | P1 | 为什么检查 cancelled 标志还不够？订阅和取消之间如何避免竞态？ | 检查后再执行存在窗口；RunHandle 同锁协调启动与结算，订阅可能同步回调，中断能力晚绑定需补检查 |
| A6 | P0 | SSE 断连后，保存状态、清缓存和释放运行位的顺序能否交换？ | interrupt → 等待 → dispose；按需补存 → 清本轮实际实例 → 放运行位；先放位可能让旧流干扰新流 |

追问链：Controller finally 为什么不够 → 为什么锁整条流 → 为什么按用户 → TTL 过期旧任务继续怎么办 → CAS 能否防迟到写 → Provider 重建后清哪个实例 → PG 补存失败怎样处理。

关键定位：[AgentChatServiceImpl:221](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java#L221)、[AgentRunGate:53](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentRunGate.java#L53)、[AgentRunHandle:126](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentRunHandle.java#L126)、[StreamTaskManager:101](../framework/src/main/java/com/nageoffer/ai/ragent/framework/web/StreamTaskManager.java#L101)。

## 3. 第二条：Skills、MCP 与人工确认

源码与完整话术：[工具治理详细稿](ragent-star-agent-tools.md#point-2)。

| 编号 | 优先级 | 面试问题 | 回答检查点 |
| --- | --- | --- | --- |
| B1 | P0 | 如何判断技能仍然加载？为什么不永久保存 activated 状态？ | 当前 context 中成功且未被裁剪占位的加载结果；手册退出后收回工具；同工具多手册任一有效可解锁 |
| B2 | P0 | 工具 Schema 已隐藏，为什么执行时还检查？遮蔽计算异常会怎样？ | 推理侧减少误选，执行侧处理历史调用；共享本轮 RuntimeContext 映射；异常 fail-open，不能代替业务鉴权 |
| B3 | P1 | MCP 如何接入？异步线程上的用户身份从哪里来？模型能否伪造身份？ | 复用 MCP executor 桥，身份取 RuntimeContext 并走 `_meta`；模型不提供身份，metadata 本身不是认证协议 |
| B4 | P0 | 哪些工具必须确认？只读工具会弹确认卡吗？ | 任一意图节点要求确认，或服务端未明确 readOnlyHint=true；明确只读也可因业务配置确认 |
| B5 | P0 | 用户批准时能否修改订单号或金额？实际执行参数从哪里恢复？ | DTO 只接会话、消息、决定；所属与 pending 校验；原 ASKING 调用来自服务端状态，展示卡片不是权威参数来源 |
| B6 | P0 | 重复确认如何处理？卡片已结算但进程崩溃，或远端成功后响应丢失呢？ | 运行位、终态检查、先结卡后订阅；卡片普通 updateById，无跨 PG/MCP 事务；业务操作 ID、幂等结果和对账需另行设计 |

追问链：手册怎样解锁工具 → 压缩后工具为什么收回 → metadata 仍在但正文不在怎么办 → 模型硬调隐藏工具怎么办 → fail-open 时的边界 → ASK 条件 → 原参数谁保存 → 确认后崩溃/远端结果未知怎么办。

关键定位：[AgentSkillMaskingMiddleware:77](../agent/src/main/java/com/nageoffer/ai/ragent/agent/skill/AgentSkillMaskingMiddleware.java#L77)、[McpToolBridge:84](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/McpToolBridge.java#L84)、[AgentToolCatalog:198](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalog.java#L198)、[确认恢复:185](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java#L185)。

## 4. 第三条：混合检索与三段预算

源码与完整话术：[检索详细稿](ragent-star-retrieval.md)。

| 编号 | 优先级 | 面试问题 | 回答检查点 |
| --- | --- | --- | --- |
| C1 | P0 | 为什么选加权 RRF？能手算同一 chunk 在两路不同名次的融合分吗？ | 原始分量纲不同；按权重与名次累加，说明 k 和权重影响；排名融合丢弃绝对分差，需评测校准 |
| C2 | P0 | 先去重会不会丢掉多路投票？按什么去重？ | 代表对象去重，原始通道列表保留；key 优先 chunk ID，无 ID 用正文 SHA-256，不是语义去重 |
| C3 | P0 | 召回、候选、上下文预算分别控制什么？20、40、10 为什么不矛盾？ | 20 是每通道召回基数，多路合计后截到候选上限；10 是每子问题上下文条数，条数不是全回答 Token 上限 |
| C4 | P1 | RRF 已排序，为何仍需要 Rerank？降到 noop 后会怎样？ | RRF 综合通道排名，重排模型重新判断文本相关性；noop 只截断、不产生精排分，不等价于真实模型精排 |
| C5 | P0 | 一条高分、九条低分会被门控如何处理？缺分、换模型会怎样？ | 最高有限 rerankScore 低于阈值才整批丢弃；一条达标保留整批；缺分放行；阈值依赖重排模型，需校准 |
| C6 | P1 | 当前默认开了几路？通道超时和后处理异常如何影响结果？ | 默认仅向量，其他通道需开关与后端装配；超时转空但不停止底层任务；后处理器异常会跳过继续 |

追问链：为什么 RRF → 手算融合分 → 去重后原始排名在哪 → k/权重如何选 → 候选截断后图谱证据为何消失 → noop 后门控怎么走 → 一条高分为什么放行整批 → 用同一批数据如何验证。

关键定位：[FusionPostProcessor:100](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/FusionPostProcessor.java#L100)、[RetrievalBudget:34](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/RetrievalBudget.java#L34)、[EvidenceGatePostProcessor:75](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/EvidenceGatePostProcessor.java#L75)。

## 5. 第四条：上下文压缩与长期记忆

源码与完整话术：[记忆详细稿](ragent-star-memory.md)。

| 编号 | 优先级 | 面试问题 | 回答检查点 |
| --- | --- | --- | --- |
| D1 | P0 | 为什么分三层？摘要和长期记忆有什么区别？ | 工具正文裁剪、当前会话摘要、跨会话稳定事实；用户 active 事实有限容量注入，不是向量 topK 记忆 |
| D2 | P0 | 裁剪和摘要怎样保护 tool_use / tool_result？摘要切点如何选？ | 裁剪保留调用与结果标识；保护当前、未闭合及近期循环；摘要按用户轮次切点，并检查保留段孤立结果，不安全就跳过 |
| D3 | P0 | 已经有用户行锁，为什么还要 revision 和 watermark 双校验？ | 行锁保护提交顺序，revision 保护事务外读取的用户记忆集，watermark 保护本会话已处理区间；单靠行锁不验证旧快照 |
| D4 | P0 | NOOP 为什么仍推进水位？它一定不增加 revision 吗？ | 消息已判完需推进水位；纯无变更不增版本，合并/淘汰改变记忆集后即使 NOOP 仍会增版本 |
| D5 | P1 | 为什么按字符水位？如何处理摘要丢失约束和实际模型窗口？ | 字符是工程代理；实际预算还要计人设、Schema、事实块与输出预留；协议结构安全不等于摘要语义无损 |
| D6 | P1 | 冲突会立即重试吗？模型迟到、进程退出、容量满怎样处理？ | 后续收尾/flush 再触发，非持久重试队列；PROCESSING 回收与条件结算，迟到提交可回滚；合并/淘汰/拒收及 DROPPED 边界 |

追问链：为什么模型不放事务内 → 事务外快照如何验证 → 已有行锁为何要版本号 → 没写新事实但别人处理过同一区间怎么办 → NOOP 与水位 → 合并后 NOOP → claim 回收后旧结果回来 → 谁触发重试 → 怎么验证这些时序。

关键定位：[AgentContextCompactor:152](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentContextCompactor.java#L152)、[AgentMemoryPipeline:94](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryPipeline.java#L94)、[AgentMemoryRepository:215](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryRepository.java#L215)、[AgentMemoryExtractionMapper:35](../agent/src/main/java/com/nageoffer/ai/ragent/agent/dao/mapper/AgentMemoryExtractionMapper.java#L35)。

## 6. 第五条：模型路由、熔断与首包边界

源码与完整话术：[模型路由详细稿](ragent-star-model-routing.md)。

| 编号 | 优先级 | 面试问题 | 回答检查点 |
| --- | --- | --- | --- |
| E1 | P0 | fast / standard / deep 如何选？preferred 和 thinking 如何影响候选？ | 任务档位、有序候选、去重、启用与思考能力过滤；thinking 命中 deep 配置优先于 override，实际准入还要过健康状态 |
| E2 | P0 | 为什么先探测首包再提交？内容先到、错误随后到但尚未 commit，如何处理？ | 首个内容/思考回调将探测置 SUCCESS；迟到错误不覆盖结果，沿当前流转发；仅探测超时/错误/无内容完成才换候选 |
| E3 | P0 | 三态熔断如何转移？半开为何只放一个探测？多节点状态共享吗？ | 连续失败打开，时间到半开，成功关闭、失败重新打开；限制探测占用，当前健康表是进程内存，不是共享熔断状态 |
| E4 | P0 | 半开许可为什么带令牌？旧请求晚结束会发生什么？ | 布尔值无法区分占用代次；释放需状态和令牌匹配，旧许可不能释放新探测名额；等待中断先取消再 finally 释放 |
| E5 | P1 | 为什么用户取消不应计模型失败？所有取消入口都一样吗？ | 同步执行识别取消后不降级/不计失败；流式首包等待中断释放许可；Provider 启动阶段普通异常分类要另行核对 |
| E6 | P0 | 这套路由覆盖主 Agent 吗？首包之后失败会切换模型吗？ | 当前只属于 RAG / infra-ai 调用；主 Agent 独立单模型；探测成功后保持当前流，不自动拼接其他模型答案 |

追问链：为什么首包前可以换 → 如何阻止失败候选回调污染前端 → 首个事件为何决定探测结果 → 内容先到后错误还切不切 → 首包成功是不是模型全程健康 → 半开名额何时释放 → 旧许可如何识别 → 多节点状态与重启怎么办。

关键定位：[RoutingLLMService:115](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/chat/RoutingLLMService.java#L115)、[ProbeStreamBridge:44](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/chat/ProbeStreamBridge.java#L44)、[ModelHealthStore:143](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/model/ModelHealthStore.java#L143)。

## 7. 验证与 Java 工程：面试官如何判断你做过实现

| 编号 | 优先级 | 面试问题 | 回答检查点 |
| --- | --- | --- | --- |
| F1 | P0 | 如何确定性复现取消、确认和旧记忆提交竞争？为什么只 sleep 不够？ | latch/屏障控制交错、可控制模型、调用计数与归属断言；mock 单元测试、真实 PG/Redis 集成与生产记录分别说明 |
| F2 | P0 | 如何证明混合检索和 Rerank 有价值？实验如何保证可比？ | 固定语料、问题、标注与模型配置，分阶段指标与消融；保留失败样本，检索证据接口不等于完整回答质量评测 |
| F3 | P0 | 如何计算长对话总 Token 成本？压缩多调用一次模型是否划算？ | 同时计主 Agent、知识工具内部生成、摘要、Judge 和重试调用；对比关键约束保留与成本，字符水位不能直接换算降幅 |
| F4 | P1 | 为什么选择 pgvector？如何处理索引、过滤、排序和检索规模增长？ | 结合真实数据与部署约束回答，能沿 SQL 和库范围适配讲距离、近似索引、结果排序与过滤；不要泛称数据库一定更快 |
| F5 | P1 | RocketMQ 事务消息如何衔接文档状态和分块？重复消费怎样处理？ | 生产侧本地状态事务与 Broker 回查，RUNNING 回查规则；消费者重试和业务幂等单独解释，不把事务消息当 exactly-once |
| F6 | P1 | 为什么拆线程池、使用 boundedElastic？饱和和取消时会怎样？ | 阻塞 I/O 隔离、队列与拒绝策略、CallerRuns 执行线程、上下文透传和句柄清理；异步并不代表底层任务自动取消 |

验证与工程定位：[测试及运行入口](ragent-star-interview-guide.md)、[检索评测接口](../rag/src/main/java/com/nageoffer/ai/ragent/rag/eval/EvalController.java#L62)、[文档事务消息:207](../rag/src/main/java/com/nageoffer/ai/ragent/knowledge/service/impl/KnowledgeDocumentServiceImpl.java#L207)、[事务回查](../rag/src/main/java/com/nageoffer/ai/ragent/knowledge/mq/KnowledgeDocumentChunkTransactionChecker.java#L62)、[线程池配置](../rag/src/main/java/com/nageoffer/ai/ragent/rag/config/ThreadPoolExecutorConfig.java#L46)。

## 推荐演练顺序

先讲 G1 / G2 项目与全链路，再练 A1 / A3 / A4、B1 / B5 / B6、C1 / C3 / C5、D2 / D3 / D4、E2 / E4 / E6，最后练 F1 / F2 / F3 的验证方法。每次回答以一个具体时序或例子结束，避免只列技术名词。

我判断最容易拉开回答深度的六个切入点是：Redis CAS 的实际作用、确认后崩溃窗口、去重后 RRF 多路投票、工具消息配对、revision 与 watermark 的不同职责、首包探测与缓冲提交的先后关系。

## 交付与自检

本次仅新增本文，保留原有未跟踪文档；没有新增依赖或修改业务行为。自检覆盖 42 个唯一问题编号、五条职责对应关系、Markdown 链接及源码行号范围、差异格式。本次没有启动项目、运行 Java 测试、模型或数据库实验。

无需安装或部署。准备答案时补充你实际完成的测试、实验或部署记录；当前材料中的机制和配置不替代效果测量。本次文档变更不影响运行代码。

Git 工作分支为 `feature/ragent-interview-questions`；自检后提交合并到 main，并沿用授权目标 `xuhuilun/ragent` 普通推送。
