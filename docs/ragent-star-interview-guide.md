# Ragent 五条简历职责 STAR 面试话术与源码总览

日期：2026-10-13（Asia/Shanghai）。核查基线：`4d5360857a22671ddb30ce544cd86b42af1860be`。对应[当前简历五条职责](ragent-resume-personalization.md#简历正文)。

## 个人职责与准备方式

Ragent 的业务代码由本人独立完成，话术统一使用“我设计、我实现”。AgentScope 的 ReAct 循环及其他第三方库由对应框架提供，项目贡献是应用层接入、约束和执行治理。

每条都包含 S（场景）、T（任务）、A（技术行动）、R（交付结果），并另附约 90–120 秒连续口述稿。口述时不念 S/T/A/R 字母，先用场景说明问题，再把时间主要留给实现动作和关键取舍。时长仅为准备稿长度估算。

场景围绕实现要处理的设计约束；结果陈述已交付的机制与行为，没有补造生产事故、准确率或性能提升数字。运行效果可在有记录时另行补充。

## 实施计划与材料索引

本次按五条简历原文追踪源码与配置，核对关键类和方法位置；逐条整理 STAR、连续话术、核心实现及支撑文件、调用顺序和追问；同步更新个人独立开发口径，完成链接与内容检查后提交合并。仅修改文档，不修改业务源码或新增依赖。

| 简历条目 | 详细材料 | 建议先读的核心源码 |
| --- | --- | --- |
| 1. Agent 执行、状态恢复与流生命周期 | [第 1 条完整话术](ragent-star-agent-tools.md#point-1) | [ReActAgentProvider](../agent/src/main/java/com/nageoffer/ai/ragent/agent/config/ReActAgentProvider.java#L113)、[AgentChatServiceImpl](../agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java#L158)、[StreamTaskManager](../framework/src/main/java/com/nageoffer/ai/ragent/framework/web/StreamTaskManager.java#L146) |
| 2. Skills 双重校验与 MCP 人工确认 | [第 2 条完整话术](ragent-star-agent-tools.md#point-2) | [AgentSkillMaskingMiddleware](../agent/src/main/java/com/nageoffer/ai/ragent/agent/skill/AgentSkillMaskingMiddleware.java#L79)、[McpToolBridge](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/McpToolBridge.java#L84)、[AgentToolCatalog](../agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalog.java#L198) |
| 3. 可配置混合检索与分阶段预算 | [第 3 条完整话术](ragent-star-retrieval.md) | [MultiChannelRetrievalEngine](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/MultiChannelRetrievalEngine.java#L132)、[FusionPostProcessor](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/FusionPostProcessor.java#L100)、[EvidenceGatePostProcessor](../rag/src/main/java/com/nageoffer/ai/ragent/rag/core/retrieval/postprocessor/EvidenceGatePostProcessor.java#L75) |
| 4. 分层记忆与并发一致性 | [第 4 条完整话术](ragent-star-memory.md) | [AgentContextCompactor](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentContextCompactor.java#L69)、[AgentMemoryPipeline](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryPipeline.java#L94)、[AgentMemoryRepository](../agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryRepository.java#L215) |
| 5. RAG 模型路由与流式容错 | [第 5 条完整话术](ragent-star-model-routing.md) | [ModelSelector](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/model/ModelSelector.java#L77)、[RoutingLLMService](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/chat/RoutingLLMService.java#L115)、[ModelHealthStore](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/model/ModelHealthStore.java#L143) |

## 1. Agent 执行、状态恢复与流生命周期

**S：** 企业问答与业务办理需要多次检索和工具调用，任务可能跨轮次执行，用户也可能随时停止或断开流。

**T：** 我需要把 Agent 循环接入 Web 会话，统一管理状态恢复、运行互斥、取消与资源释放。

**A：** 我基于 AgentScope 装配 ReAct 与统一工具目录，以 PostgreSQL 保存会话状态；通过 Redis 用户运行位控制并发，结合跨节点取消和 SSE 生命周期回调协调启动、中断及收尾。

**R：** 我把多步骤执行、连续追问、状态恢复和流式停止纳入同一条执行链路，旧任务释放运行位时必须匹配自身占位，避免误释放后来的任务。

完整口述与源码表见[第 1 条](ragent-star-agent-tools.md#point-1)。

## 2. Skills 双重校验与 MCP 人工确认

**S：** 技能手册全部进入上下文会占用预算；模型在缺少规程时也可能直接调用业务工具，写操作还需要明确的用户批准。

**T：** 我需要按需提供操作规程，把工具可见性、执行检查和人工确认接到同一链路。

**A：** 我通过 `load_skill` 加载手册，在模型推理阶段遮蔽关联工具，执行前再次检查；对于需确认的 MCP 调用保存待执行状态，确认恢复时从服务端读取原始工具参数。

**R：** 关联工具的可见性跟随当前上下文中的手册变化，需确认操作可以暂停并恢复，确认接口不允许客户端替换待执行参数。

完整口述与源码表见[第 2 条](ragent-star-agent-tools.md#point-2)。

## 3. 可配置混合检索与分阶段预算

**S：** 企业资料查询同时涉及语义表达、精确术语和关联信息，不同召回通道的原始分数不能直接比较。

**T：** 我需要统一融合通道结果，并分别控制召回、重排和回答上下文的候选规模。

**A：** 我实现可配置通道与并行召回，用加权 RRF 按排名融合，再进行 Rerank 和批级证据门控；通过独立预算控制每通道召回、融合候选与最终上下文条数。

**R：** 检索得到统一排序的候选链路，各阶段规模可分别配置，通道超时可按空结果降级；当前默认只启用向量，其他通道按配置接入。

完整口述与源码表见[第 3 条](ragent-star-retrieval.md)。

## 4. 分层记忆与并发一致性

**S：** 长对话中工具结果持续积累，用户事实还要跨会话保存；异步抽取完成时，原先读取的记忆可能已经被其他会话更新。

**T：** 我需要压缩上下文并保护工具协议，同时让慢模型调用避免长时间持有数据库事务。

**A：** 我实现旧工具结果裁剪、会话摘要和跨会话事实记忆，保护近期轮次与工具调用配对；模型判定和合并规划放在事务外，提交时锁用户控制行并校验 revision 与 watermark。

**R：** 长对话上下文按字符水位管理，事实可以跨会话注入；过期抽取结果被拒绝提交，未修改记忆的处理也通过水位记录进度。

完整口述与源码表见[第 4 条](ragent-star-memory.md)。

## 5. RAG 模型路由与流式容错

**S：** 多 Provider 调用可能启动失败或首包超时，流式内容发送后难以撤回，取消等待还可能占住半开探测名额。

**T：** 我需要明确候选切换和输出提交的边界，同时让熔断探测可以继续恢复。

**A：** 我按档位组织候选，以三态熔断控制准入；流式回调先缓冲，首包成功再提交，探测失败则换候选。等待中断时取消请求，并用令牌校验释放当前半开名额。

**R：** RAG 调用实现首包探测失败时的候选切换，提交后保持同一输出来源；旧请求不能误释放新的半开探测许可。

完整口述与源码表见[第 5 条](ragent-star-model-routing.md)。

## 文档交付与使用

新增本总览以及 Agent / 工具、检索、记忆、模型路由四份详细稿；同步更新简历、证据与三份原面试文档的独立开发口径。原有四份未跟踪文档保持原状。

自检通过：五条简历原文与章节引用一致，五份新增文档的 200 个本地链接及源码行号范围有效，章节定位和 STAR 四要素完整，`git diff --check` 无格式错误。没有运行模型、Java 测试、数据库或多节点实验，相关测试表只说明测试源码覆盖的场景。

不需要安装依赖或执行部署。准备时先阅读核心源码，按自己的真实设计动机调整口语，再补有记录的实验结果；不要把默认配置数值读成测量结果。

Git 工作分支为 `feature/ragent-star-interview`；自检后提交、合并到 main，并沿用此前授权目标 `xuhuilun/ragent` 普通推送。本次没有业务代码改动，运行行为不受影响。
