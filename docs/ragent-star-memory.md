# 第 4 条简历：分层记忆与并发提交 STAR

整理日期：2026-10-13（Asia/Shanghai）。材料基线：`4d536085`。个人贡献按用户本轮确认的“全部业务代码由本人独立完成”整理。本文直接核查业务源码、配置、SQL 与测试文件；没有运行数据库、模型或业务测试，不将测试源码当成本轮通过报告。

## 原简历条目

> 构建工具结果裁剪、会话摘要与跨会话事实记忆，保护近期消息及工具调用配对；采用事务外模型判定、短事务内版本号与处理水位双校验，防止过期抽取结果覆盖新事实。

这条包含两个独立问题：会话内上下文如何压缩，以及跨会话事实如何并发更新。前者处理消息结构和上下文容量，后者处理模型调用期间数据库状态已经变化的提交窗口。

## STAR 展开

### S：问题背景

我在设计多轮 Agent 时，需要同时处理两类信息：商品或知识检索的工具结果往往很长，却未必需要在后续每轮完整发送；用户的稳定偏好与约束又需要跨会话保留。如果只按消息条数删除历史，可能割裂 `tool_use` 与 `tool_result`；如果把全部历史长期保留，上下文会持续增长。另一处设计风险是后台事实抽取：模型判定期间，同一用户的其他会话可能已经修改记忆，较早启动的抽取反而较晚提交，不能直接覆盖新状态。这是系统设计要处理的场景，不描述为未经记录的线上事故。

### T：我的任务

我独立设计并实现记忆模块，目标是把会话内信息压缩与跨会话事实沉淀分开治理：压缩时保留近期交互和工具调用结构；事实更新时避免慢模型调用长期占用数据库行锁，并在提交前确认模型判定依据仍有效。容量预算、模型失败、并发冲突和没有新事实的情况都需要有明确出口。

### A：我的具体行动

1. 我用字符数作为上下文体量的工程代理，设置裁剪与摘要两道水位。较低水位只处理白名单中较老的工具结果，不动 `tool_use`，并保留结果的 ID、工具名、元数据及状态；本轮、未闭合循环和前两个已完成循环受到保护。可回收字符不足当前上下文的 20% 时，本次裁剪直接跳过。
2. 我把摘要安排在用户轮次边界：最后一条上下文消息必须是用户消息，切点向前对齐到真实用户轮次起点，保留近期原文。摘要替换前检查保留段是否有找不到对应 `tool_use` 的结果；没有安全切点、上行引用不匹配或摘要不合格时放弃摘要并退回裁剪。模型调用放到 `boundedElastic`，避免占用推理线程。
3. 我把跨会话事实放入按用户隔离的事实表。每次 Agent 调用首轮读取 active 条目的快照，放入本次 `RuntimeContext`，再注入上行消息副本。后台抽取只读取水位之后、控制行创建时间之后的用户原文；模型生成新增、取代或撤回决策。无参 `flush_memory` 只负责触发服务端抽取，模型不能直接提交要写入的事实内容。
4. 我把模型判定与数据库提交拆开。先通过抽取台账和部分唯一索引抢占同会话处理权，在事务外调用 Judge；容量预演超限时也在事务外规划合并。提交才开启短事务，锁用户控制行，同时比较用户记忆集版本 `revision` 和当前会话处理水位 `watermark`。任一个变化就拒绝本批旧结果，不继续应用原决策。
5. 我在提交阶段重验目标条目归属、有效状态与长度，按容量需要执行受限合并、淘汰或整批拒收。成功、无新决策以及重试耗尽分别结算为 `WRITTEN`、`NOOP`、`DROPPED`；这些状态推进处理水位。纯粹没有记忆集变更的 `NOOP` 不增版本号，合并或淘汰已经改变记忆集时，即使最终为 `NOOP`，仍会递增版本号。

### R：可据源码说明的结果

我交付了工具结果裁剪、历史摘要和跨会话事实三个层次的记忆机制，以及容量、失败与并发提交的处理流程。裁剪保留工具调用结构，摘要对保留段的孤立工具结果设检查出口；模型判定期间产生的过期快照会在事务提交阶段被拒绝，模型调用位于数据库提交事务之外。源码还保留压缩前后字符、摘要审计事件和抽取台账，便于核对实际触发与结算。

这些是实现结果，不是实测效果：目前这份材料没有 Token 成本降幅、数据库锁等待降幅、事实抽取准确率或生产故障改善数据；摘要和抽取本身会新增模型调用。面试中可说明“建立了上下文容量与并发提交的控制机制”，不能补写不存在的实验数字。

## 90—120 秒连续口述稿

这块我独立设计和实现的是分层记忆，主要解决长对话信息越来越多，以及跨会话事实并发更新的问题。工具结果经常很长，但直接删除历史消息可能破坏工具调用和返回结果的配对。我先按字符预算分两道水位，超过一半裁剪白名单中的旧工具结果，保留调用和结果的标识，同时保护当前轮、未完成调用和近期循环；超过八成，再把较早的完整轮次压成摘要，保留近期原文，并检查保留段有没有孤立工具结果。第三层是用户的跨会话事实，我只从用户原文抽取，按用户保存，下一次会话注入事实快照。这里最关键的是并发窗口：模型判定期间，其他会话可能已经更新记忆，所以我把模型调用放在事务外，提交时才锁用户控制行，同时检查记忆版本号和会话处理水位。版本号保护记忆集，水位保护已处理区间，任一变化就拒绝旧结果。没有新事实也要推进水位；如果先做了合并，即使最后没有新增决策，版本号仍要更新。最终我把长对话压缩、跨会话事实沉淀和并发提交控制接成了完整流程，并把慢模型调用与短数据库提交分开。

## 三条链路的实际调用顺序

### 1. 工具结果裁剪

1. `ReActAgentProvider.buildAgent` 将 `AgentUserMemoryMiddleware` 放在外层，再进入 `AgentContextCompactionMiddleware.onReasoning`。事实注入块属于上行副本，不写进会话 context。
2. `AgentContextCompactionMiddleware.dispatch` 解析本轮可变 context；不满足摘要条件、摘要关闭或引用不匹配时进入 `trim`。
3. `AgentContextTrimmer.trimInPlace` 统计字符，超过 50% 水位才继续；`splitCycles` 按 assistant 的工具调用切分循环，`protectedCycles` 保护当前轮、未闭合和近期循环。
4. `collectCandidates` 只选择允许裁剪的旧工具结果，按调用 ID 找对应入参。可回收量至少达到上下文的 20% 才执行 `apply`。
5. `evict` 以占位文本替换工具输出，保留原 ID、name、metadata、state。`trim` 用替换映射同步本次上行消息，接着交给后续推理。

这里保留的是工具协议结构，不是所有历史工具输出正文。实际原文会被有意裁剪。

### 2. 会话摘要压缩

1. `dispatch → shouldCompact`：摘要开启、上下文超过 80% 水位且末消息为 USER，才尝试压缩；`resolvePrefix` 先按对象引用检查上行列表与 context 是否对应。
2. `compact → AgentContextCompactor.compactInPlace → findSafeCutoff`：从尾部保留至少 20% 字符预算，再把切点向前对齐真实用户轮次起点。可替换原文不足原上下文一半时跳过。
3. `hasOrphanToolResult` 检查保留段里的每个结果 ID 是否已经出现对应调用；发现孤立结果就放弃本次摘要。
4. `AgentConversationSummarizer.summarize` 在已有摘要基础上生成下一代摘要，使用带 nonce 的素材围栏和长度、分节校验。旧工具结果进入摘要素材时还会留头留尾，不保证完整保留其全部正文。
5. 摘要通过后，`compactInPlace` 一次性将早期前缀换成摘要消息，保留尾部原文；`audit` 写压缩审计表，`rebuild` 重建上行消息。摘要不可用时退回裁剪；审计写入失败只留日志，不回滚已经完成的压缩。

摘要属于有损压缩。配对检查约束消息结构，不能证明模型完整保留全部事实或没有语义漂移。

### 3. 长期事实抽取、提交与注入

1. `AgentChatServiceImpl.startRun` 在写入用户消息之前调用 `ensureExtractionBaseline`，预建用户控制行。它的创建时间兼抽取下界；失败不拦对话，但本轮可能因此漏出下界。
2. 运行释放钩子调用 `scheduleMemoryExtraction`，在 `boundedElastic` 异步触发 `AgentMemoryPipeline.extract(BACKGROUND)`。`flush_memory → extract(FLUSH)` 使用同一管线，绕过后台最少三条待处理用户消息的门槛。
3. `extract → ensureControl/currentWatermark/loadPending` 读取控制快照、水位及待处理用户原文，再通过 `claim` 抢占同会话 PROCESSING 台账。抢不到返回 BUSY；一次最多取 40 条，旧 PROCESSING 超过 10 分钟可被回收。
4. `runExtraction → AgentMemoryJudge.judge` 在事务外执行模型判定；`planConsolidation` 预演成品记忆块，超限才在事务外调用 `AgentMemoryConsolidator.plan`。
5. `AgentMemoryRepository.commit` 才开启事务：`selectForUpdate(userId) → selectWatermark → 比较 expectedRevision / expectedWatermark`。不一致则结算 CONFLICT 并拒绝旧决策；这是乐观快照判定与短事务行锁提交的结合。
6. 校验通过后重读 active 条目，复核合并计划与有效决策，必要时淘汰；仍超限抛异常回滚。随后应用 ADD/SUPERSEDE/RETRACT，推进记忆集版本，条件结算台账。结算必须从 PROCESSING 更新一行，否则抛异常回滚，避免已被回收的迟到结果提交。
7. `WRITTEN`、`NOOP`、`DROPPED` 推进会话水位；CONFLICT 不推进。失败到第三次会被 DROPPED，意味着本批可能被放弃，不是保证所有事实最终都写入。快照冲突不消耗同一区间的失败次数，后续触发再尝试，当前没有独立持久重试调度器。
8. 后续 Agent 调用由 `AgentUserMemoryMiddleware` 首轮加载新快照；同一调用内成功 flush 后，`MemoryFlushTool` 按 `outcome.mutated()` 刷新本轮快照。即使 applied=0，合并已经改变记忆集也需要刷新。

## 核心源码文件

以下链接均相对本文件定位，`#L` 为上述基线中的一基行号。

| 文件与定位 | 类 / 方法 | 与简历条目的关系 |
| --- | --- | --- |
| [AgentContextCompactionMiddleware.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentContextCompactionMiddleware.java#L62) | `dispatch`、`shouldCompact`、`trim`、`resolvePrefix` | 推理前分派裁剪或摘要，检查上下文与上行引用，同步改写结果 |
| [AgentContextTrimmer.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentContextTrimmer.java#L68) | `trimInPlace`、`splitCycles`、`protectedCycles`；[evict](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentContextTrimmer.java#L237) | 工具白名单、保护范围、裁剪门槛及协议字段保留 |
| [AgentContextCompactor.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentContextCompactor.java#L69) | `compactInPlace`；[findSafeCutoff / hasOrphanToolResult](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentContextCompactor.java#L152) | 用户轮次切点、近期原文和保留段配对检查、一次性替换与审计 |
| [AgentConversationSummarizer.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentConversationSummarizer.java#L87) | `summarize` | 摘要模型调用、输出结构与长度校验；生成失败放弃摘要 |
| [AgentUserMemoryMiddleware.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentUserMemoryMiddleware.java#L65) | `dispatch`、`load`、`inject` | 用户 active 事实的调用级快照、上行副本注入及旧块去重 |
| [AgentMemoryPipeline.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryPipeline.java#L67) | `extract`；[runExtraction / planConsolidation](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryPipeline.java#L94) | 抽取门槛、claim、事务外模型判断与合并规划、结局转换 |
| [AgentMemoryJudge.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryJudge.java#L78) | `judge`、`parse`、`toDecision` | 用户原文与现有事实的模型判定，结构化 ADD/SUPERSEDE/RETRACT |
| [AgentMemoryRepository.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryRepository.java#L215) | `commit`；[claim](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryRepository.java#L169)；[settleOrThrow](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryRepository.java#L485) | 短事务行锁、revision/watermark 双校验、容量与目标复核、迟到提交回滚 |

## 支撑源码文件

| 文件与定位 | 类 / 方法 | 支撑职责 |
| --- | --- | --- |
| [ReActAgentProvider.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/config/ReActAgentProvider.java#L113) | `buildAgent` | 挂载事实注入与上下文压缩中间件，明确执行顺序 |
| [AgentChatServiceImpl.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java#L138) | `startRun`；[释放钩子](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java#L283)；[scheduleMemoryExtraction](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java#L343) | 首条消息前建基线、轮次收尾后异步抽取 |
| [AgentContextChars.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentContextChars.java#L38) | `total`、`of`、`ofOutput` | 统一字符统计，非文本块按零计；不是精确 tokenizer |
| [AgentMemoryBlock.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryBlock.java#L54) | `render`、`projectedChars` | 注入与容量校验共用成品记忆块口径，计入围栏和说明开销 |
| [AgentMemoryConsolidator.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryConsolidator.java#L77) | `plan`、`call` | 事务外模型规划受限合并，失败返回空计划 |
| [MemoryFlushTool.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/MemoryFlushTool.java#L68) | `getParameters`、`execute`；[refreshSnapshot](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/MemoryFlushTool.java#L118) | 无参触发、身份取自 RuntimeContext、按记忆集实际变更刷新快照 |
| [AgentMemoryControlMapper.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/dao/mapper/AgentMemoryControlMapper.java#L54) | `selectForUpdate`、`bumpRevision` | PostgreSQL 用户级行锁与版本递增 |
| [AgentMemoryExtractionMapper.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/dao/mapper/AgentMemoryExtractionMapper.java#L35) | `selectWatermark`、`settle`、`recycleStale` | 终态水位、PROCESSING 条件结算与超时回收 |
| [AgentMemoryMapper.java](../agent/src/main/java/com/nageoffer/ai/ragent/agent/dao/mapper/AgentMemoryMapper.java#L34) | `supersede`、`retract` | 更新谓词同时限制用户、条目 ID 与 active 状态；旧行失效，新行追加 |

## 配置、SQL 与测试索引

### 配置口径

| 项目 | 当前有效值 / 规则 | 定位 |
| --- | --- | --- |
| 上下文字符预算 | YAML 为 100000，覆盖配置类默认 1200000；不包含人设、工具 Schema、输出预留及单独注入的事实块 | [application.yaml:50](../bootstrap/src/main/resources/application.yaml#L50)、[AgentMemoryProperties:88](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryProperties.java#L88) |
| 裁剪 / 摘要 / 保留段 | 超过 50000 / 80000 字符触发；摘要保留原文目标至少 20000 字符，再对齐用户轮次 | [AgentMemoryProperties:38](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryProperties.java#L38)、[派生方法:115](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryProperties.java#L115) |
| 工具裁剪 | 白名单当前仅 `search_knowledge`；保护前两个已完成循环，当前轮与未闭合循环额外保护；可回收量需达到上下文的 20% | [application.yaml:54](../bootstrap/src/main/resources/application.yaml#L54)、[AgentMemoryProperties:56](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryProperties.java#L56) |
| 开关与容量 | 摘要、长期记忆均开启；当前摘要正文上限 6000 字符，长期记忆成品块上限 1500 字符 | [application.yaml:52](../bootstrap/src/main/resources/application.yaml#L52)、[AgentMemoryProperties:138](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryProperties.java#L138) |
| 抽取与失败 | 后台至少三条待处理用户消息，flush 绕过门槛；最多 40 条一批，PROCESSING 超时回收为 10 分钟，失败三次可放弃本批，单条正文最多 500 字符 | [AgentMemoryProperties:81](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryProperties.java#L81)、[AgentMemoryRepository:68](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryRepository.java#L68) |

这些数字是配置与算法常量，不是吞吐、成本或质量实验结果。

### SQL 索引

| 文件与定位 | 对应结构 |
| --- | --- |
| [260823_agent_context_compaction.sql:11](../resources/database/upgrades/v2.0.0/260823_agent_context_compaction.sql#L11) | `AGENT_CONTEXT_COMPACTION` 摘要提示词 |
| [260823_agent_context_compaction.sql:53](../resources/database/upgrades/v2.0.0/260823_agent_context_compaction.sql#L53) | `t_agent_context_compaction` 追加审计表，包含代次、摘要、压缩前后字符 |
| [260827_agent_long_term_memory.sql:5](../resources/database/upgrades/v2.0.0/260827_agent_long_term_memory.sql#L5) | `t_agent_memory` 事实表、active 部分索引、失效和后继字段 |
| [260827_agent_long_term_memory.sql:24](../resources/database/upgrades/v2.0.0/260827_agent_long_term_memory.sql#L24) | 抽取台账；[第 39 行](../resources/database/upgrades/v2.0.0/260827_agent_long_term_memory.sql#L39)部分唯一索引约束同用户同会话仅一条 PROCESSING |
| [260827_agent_long_term_memory.sql:54](../resources/database/upgrades/v2.0.0/260827_agent_long_term_memory.sql#L54) | 用户控制行，保存 revision 与抽取下界创建时间 |
| [260827_agent_long_term_memory.sql:67](../resources/database/upgrades/v2.0.0/260827_agent_long_term_memory.sql#L67)、[第 112 行](../resources/database/upgrades/v2.0.0/260827_agent_long_term_memory.sql#L112) | 事实抽取与容量合并提示词；仅提示词要求不构成语义正确性证明 |

### 测试与回归入口

| 文件与定位 | 当前可核查范围 | 使用时的证据边界 |
| --- | --- | --- |
| [AgentMemoryPipelineTest.java:45](../agent/src/test/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryPipelineTest.java#L45) | 控制行预建、长期记忆关闭跳过、预建异常不阻塞对话 | mock 单元测试；不覆盖真实 PG 提交竞态 |
| [MemoryFlushToolTest.java:62](../agent/src/test/java/com/nageoffer/ai/ragent/agent/tool/MemoryFlushToolTest.java#L62)、[第 99 行](../agent/src/test/java/com/nageoffer/ai/ragent/agent/tool/MemoryFlushToolTest.java#L99) | 写入后刷新快照；NOOP 且记忆集已变化时也刷新；无变更时不重读 | mock 管线结局；不证明模型抽取准确率或真实事务正确性 |
| [AgentMemoryRegressionMain.java:39](../resources/regression/agent-memory/AgentMemoryRegressionMain.java#L39) | 多轮脚本、上下文结构、跨会话约束、裁剪与摘要触发等回归检查 | 独立 main 工具，需 live 服务 / 模型 / PG；SUCCESS 仍可能附带 UNCOVERED，不能算所有项已覆盖 |
| [AgentMemoryMechanismMain.java:111](../resources/regression/agent-memory/AgentMemoryMechanismMain.java#L111) | M1 revision 竞态、M2 watermark 竞态、M3 flush 门槛、M4 NOOP 水位、M6 撤回、M7 失效目标、M8 首消息；R1—R5 跨会话、持久撤回、改口、台账与容量 | 机制回归工具存在；本轮未执行，不能报告通过率 |

## 三个常见追问

### 1. 为什么不直接按最近十条消息截断？你怎么保证工具配对？

我按消息内容统计字符，并按工具循环和用户轮次保护结构。十条消息可能是一条很大的工具结果，也可能截在工具调用和返回之间，条数不能代表体量。裁剪阶段只换结果正文，调用本身不删，结果 ID、name、metadata、state 都保留；摘要阶段先对齐用户轮次切点，再检查保留段是否有孤立结果，没有安全切点就跳过。这里保护的是协议结构和近期上下文；历史输出与摘要仍可能丢信息，我不会把它称作无损记忆。

### 2. 有 revision 为什么还要 watermark？NOOP 会不会改变 revision？

revision 表示整个用户记忆集是否变化，watermark 表示当前会话哪一段用户消息已经处理。没有记忆集变化的 NOOP 不增 revision，但这批已判断完，仍会推进水位。如果只验版本号，就识别不了“别人已经处理了同一段，但没写新事实”的过期快照。反过来，同用户另一会话更新了事实，当前会话水位可能不变，所以也不能只验水位。提交在用户行锁内双校验。NOOP 不是绝对不改版本：本批先做了合并或淘汰，随后有效决策为空，最终仍可结算 NOOP；记忆集已经变化，版本号仍会推进。

### 3. 模型判定放在事务外是不是会有竞态？冲突以后是否立即重试？

会有，所以我没有把事务外快照当权威状态。模型调用期间不持有数据库行锁，最终提交才锁控制行并重读版本和水位；任一个失配就结算 CONFLICT，不应用旧决策。快照冲突不计入内容失败次数，后续对话收尾或 flush 再触发，当前不是一个独立持久任务重试队列。Judge/解析等失败到三次可能 DROPPED 并推进水位，容量也可能拒收，因此不能承诺全部事实最终保存。现有机制保证的是该提交协议内过期结果被拒绝；它不验证模型事实判断的语义正确性，也没有这次真实 PG 并发实验的通过报告。

## 本文件自检与使用

本轮只创建此 Markdown 文件，未修改业务源码、配置、SQL、测试或 Git。已核对关键方法、配置常量和源码链接；没有执行数据库、模型或业务测试。面试时可直接使用第一人称口述稿，结果部分保持实现层面的表述；如要补充成本、准确率或锁等待改善，应另附固定样本、配置、对照方法与原始结果。
