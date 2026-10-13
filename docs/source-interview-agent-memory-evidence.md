# Agent、MCP 与记忆机制源码证据

核验日期：2026-10-12。工作基线：`98e4217d0f85c92c33b0525a14301efdfdc9abf5`；本次分析在 `feature/source-interview-phase-one` 分支完成。本文只覆盖 Agent 执行、MCP/HITL、技能、上下文压缩和长期记忆，不引用既有简历或面试文档作事实依据。

**个人贡献说明（2026-10-13，Asia/Shanghai）**：用户已明确“所有代码全是我一个人完成的”。本文按本人独立完成业务代码设计与实现的口径整理，AgentScope 等框架能力另行说明。此前贡献判断已更新；具体部署、实验与生产事故仍按实际记录表述。当前材料见[Ragent 简历](ragent-resume-personalization.md)和[逐条 STAR 面试话术](ragent-star-interview-guide.md)。

## 1. 证据口径与生效条件

- **已验证事实**：本次直接读取 Java、配置、SQL、测试和 `git log/show` 得到的静态事实。这里的“验证”不等于程序已经运行。
- **合理技术推演**：可以从实现解释的动机和权衡，不证明作者曾比较这些方案或遇到某次事故。
- **假设性优化**：后续可实施的改造或实验，不能作为既有经历或结果。
- 本次环境没有 Java Runtime，未安装依赖，未运行 Maven/JUnit、模型调用、数据库并发、回归工具或压测。没有本次执行日志与性能/效果指标。
- “默认启用”只指仓库 YAML 与装配条件，不代表服务已部署、MCP Server 可连接、数据库已迁移或模型凭据可用。

| 机制 | 仓库默认接线 / 配置 | 实际边界与依据 |
|---|---|---|
| 主执行引擎 | `ragent.engine.type=agent` | `ConditionalOnAgentEngine` 仅在值为 agent 时装配；YAML 35–54，条件注解 34。[E01][E02] |
| ReAct | AgentScope `2.0.2`；`maxIters=10` | 仓库构建 `io.agentscope.core.ReActAgent`，不自实现 ReAct 循环；最大次数是限制，不是实测性能。[E03][E04] |
| Agent 模型 | deepseek / deepseek-flash；流式 OpenAIChatModel | 单独构建模型，未进入 `infra-ai.ModelRoutingExecutor`；`maxRetries=1` 含首次，即默认不重试，不能声称 Agent 主模型有多模型 fallback。[E01][E05][E06] |
| SSE | 900000 ms | 控制器构建 SseEmitter；运行闸门 TTL 为两倍，即 1800000 ms。[E06][E07][E08] |
| 上下文预算 | YAML `100000` 字符；类兜底 `1200000` 字符 | 50% 裁剪、80% 压缩、20% 保留；默认部署值导出 50000 / 80000 / 20000 字符。不是 token 窗口，也不包含人设、工具 schema、输出预留、长期记忆注入块。[E01][E09][E10] |
| 摘要 / 长期记忆 | YAML 均为 true | 类开关也均为 true；关闭摘要只剩裁剪，关闭长期记忆停抽取与注入、保留已有数据。[E01][E09] |
| 摘要 / 记忆块上限 | YAML 窗口导出摘要 6000、长期记忆块 1500 字符 | 分别按比例导出并夹在 1500–6000；不能把类兜底窗口推导值写成 YAML 当前值。[E09] |
| 可裁剪工具 | `[search_knowledge]` | 不是全部工具结果都可裁；保留最近 2 个已完成循环，额外保护本轮与未闭合循环。[E09][E11] |
| MCP | 连接 `http://localhost:9099/mcp` | 启动时 initialize/listTools；连接失败跳过注册。Agent 只挂意图树工具与实际执行器的交集，存在配置不等于工具可用。[E01][E12][E13] |
| HITL | 节点 requireConfirm，或服务端未明确 `readOnlyHint=true` 时 ask | 是工具级确认规则，不是 RBAC、OAuth 或完整业务授权体系。[E13][E14] |
| Skill | 有启用技能才挂 `load_skill` | 每轮现读技能；实际启用项依赖数据库，不由代码存在证明。[E13][E15][E16] |
| Trace | `agent.trace.enabled=false` | 工具批事实不依赖 trace 开关；不能声称 Langfuse 默认启用或已收集运行数据。[E01][E17] |

## 2. 架构与关键调用链

### 2.1 普通对话与取消

`AgentChatController.chat` → `AgentChatServiceImpl.streamChat` → `AgentRunGate.acquire` → 检查 pending confirm → `startRun` → `ReActAgentProvider.getAgent` → 建/取 Agent 与目录快照 → `touchConversation` → `ensureExtractionBaseline` → `addUserMessage` → `launchStream` → `agent.streamEvents` → `AgentStreamEventBridge` → 助手消息持久化和 SSE 终态。[E04][E07][E08][E18]

`launchStream` 先绑定 emitter 生命周期与 release hooks，再注册取消任务；注册阶段已取消则不启动 Agent。`AgentRunHandle.start` 将确认结算、订阅、绑定 interrupt 放在启动互斥区内。成功、异常、取消共用一次收尾；release hooks 按“必要时补存框架状态 → 清本次实际 Agent 的缓存 → 释放用户闸门 → 调度后台记忆抽取”执行。[E18][E19]

取消先调用 `agent.interrupt`，最多等 2000 ms 让框架收尾存盘，然后 dispose；等待失败标记补存盘需求。这个机制管理本地运行，不代表远端业务写操作能撤回。[E19]

### 2.2 推理与工具

装配顺序：可选追踪 → `AgentUserMemoryMiddleware` → `AgentContextCompactionMiddleware` → `AgentConfirmDenialMiddleware` → `AgentSkillMaskingMiddleware` → `AgentToolBatchMiddleware` → AgentScope 推理 / acting。[E04]

工具目录固定挂 `search_knowledge`，条件挂 `flush_memory` 与 `load_skill`，MCP 工具适配为 `McpToolBridge extends ToolBase`。`search_knowledge` 等内置工具实现 AgentTool，其 `isReadOnly` getter 不等同于 ToolBase 的执行权限检查。[E13][T02]

MCP 业务调用：`McpToolBridge.callAsync` → 检查本轮 masked-tools → boundedElastic 上 `execute` → 从 `RuntimeContext.userId` 生成 `McpCallMeta` → `McpClientToolExecutor.execute` → `McpSyncClient.callTool` → MCP Server 工具。[E14][E20][E21]

### 2.3 确认恢复

框架 `RequireUserConfirmEvent` → Bridge 记录 pending confirm 块及整批 awaiting → 持久化 `AWAITING_CONFIRM` 消息 → 发 confirm SSE。持久化失败不发可点击卡片，提示另建会话。[E22][T03]

`AgentChatController.confirm` → 用户级闸门 → `getPendingConfirm` 校验 user/conversation/message → 从 Agent 状态最后一条 assistant 消息取 ASKING 工具调用 → 空正文 UserMessage 的 `METADATA_CONFIRM_RESULTS` → 启动准备完成 → 锁内 `settlePendingConfirm` → 订阅继续执行。前端不传工具入参，入参从状态重取；状态已无 ASKING 时让旧卡片 expired。[E07][E18][E23]

### 2.4 记忆

短期：框架 AgentState context → 50% 水位裁剪工具正文 / 80% 水位在用户轮边界前缀摘要 → `PgAgentStateStore` 使用框架 JSON codec 写入 PostgreSQL JSONB。业务消息表与框架状态表是两类事实源，不等同于只读聊天记录重放。[E10][E11][E24][E25][E26]

长期：轮次 release → boundedElastic `AgentMemoryPipeline.extract(BACKGROUND)`，或模型主动调用无参 `flush_memory` → 读取控制行/revision、水位与 pending 用户消息 → claim → 事务外 Judge 和可选合并 → `AgentMemoryRepository.commit` 控制行 FOR UPDATE → revision+watermark 双校验 → 受限合并/容量淘汰/应用 ADD、SUPERSEDE、RETRACT → 结算台账。[E18][E27][E28][E29][E30]

注入只读快照：首轮读一次库，存在 RuntimeContext；把记忆块放在人设后、会话 context 前的上行副本。它不写入框架 context。flush 改变记忆集后刷新本轮快照；刷新失败沿用旧视图而不把已写入报为失败。[E31][E32]

## 3. 六个候选工程难点

以下业务触发条件从代码防护对象提炼，没有编造用户规模、QPS、线上事故或提升幅度。比较列中的方案权衡均为**合理技术推演**，只有当前实现与历史表中的变更属于**已验证事实**。

### C1. ReAct 流式生命周期中的启动、取消与状态存盘竞争

**技术问题 / 触发场景**：SSE 同步返回 emitter 后，Agent 仍可能执行多轮工具。若用户关闭页面、启动时取消、正常完成与取消同时到达，仅用请求级防重或最后 finally 不足以协调订阅、状态存盘和闸门释放。[E18][E19]

| 方案 | 原理、复杂度和资源成本 | 正确性、性能、扩展与风险 |
|---|---|---|
| 请求方法 finally / 单机 synchronized | 简单，在返回 emitter 时就释放；成本低 | 只能覆盖同步窗口，无法防同用户两条长流重叠；多节点也不共享锁 |
| 当前用户级 Redis 运行位 + 本地生命周期锁 | setIfAbsent+TTL；taskId\|conversationId 校验释放；运行句柄协调启动与一次收尾，复杂度中等 | 跨节点挡同用户并发、本地锁处理启动/取消竞态；一用户跨会话仍只准一条流，降低自由度。固定 TTL 无续租，Redis 不可用会影响入口 |
| 持久运行状态机 + 下游幂等/任务队列 | 状态转换持久化、可恢复执行，复杂度和运维成本更高 | 适合更长任务/多步骤恢复；须处理租约、版本和业务幂等，无法仅凭框架 interrupt 回滚远端写入 |

**当前实现**：RunGate.acquire 53–59；CAS release 83–85；SSE TTL 派生 92–93。RunHandle.start 126–140 防先取消后订阅；settle 253–274 只收尾一次；interruptUpstream 169–203 先 interrupt 后 dispose。Service release hooks 264–283 先清状态后放行。[E08][E18][E19]

**选择依据**：bffe0e80 的真实差异把 pending 检查移到拿闸门之后，并把确认结算延迟到订阅之前；提交记录明确说明启动/取消及确认延迟结算问题。不能进一步声称作者经历了线上 Redis 事故。[H01]

**效果与局限**：源码与模拟测试支持“已设计相应时序防护”，不支持吞吐提升或零状态丢失。补存盘失败只日志，后续可能从旧状态继续；Redis 闸门 TTL 是崩溃兜底，不是永久线性化保证。PG 状态 upsert 没有 revision CAS，外部旁路写入不受 RunGate 约束。[E18][E25]

**测试与追问**：T01/T03 的 `shouldKeepGateUntilStartupHandoffCompletes`、`shouldNotStartAfterCancellationSettles`、`shouldNotSubscribeWhenConfirmationSettlementFails` 等确实构造了竞争/失败。追问入口：SSE 请求结束为何锁还不能放？为何 CAS 删运行位？十倍并发后 Redis/阻塞调度容量如何限制？超时但远端已成功怎样确认？

**个人贡献**：按最新用户说明，属于独立完成的应用层设计实现；本次仅完成源码核验。**假设性优化**：补 Redis 故障、TTL 过期与迟到 release 的真实集成测试；设计运行状态 save-failed 可观测记录，验证用户重试如何处理。尚未实施。

### C2. MCP 的能力契约、登录身份与 Human-in-the-Loop 恢复

**技术问题 / 触发场景**：模型能生成工具入参，写操作不能直接放行；客户端传回展示入参可被篡改；确认卡片结算过早会造成“卡片已处理但工具没订阅”；异步线程拿不到 ThreadLocal 登录主体。[E13][E14][E18]

| 方案 | 原理、复杂度和资源成本 | 可靠性、性能、扩展与风险 |
|---|---|---|
| 模型自己判断危险性、在 prompt 中口头确认 | 接入便宜，额外模型交互可能少 | 模型不确定性，无法保证写工具前真的停止；用户身份若进 schema 会变成模型控制值 |
| 当前服务端工具契约 + 框架 ASKING + 持久卡片 | readOnlyHint/requireConfirm 决策；状态恢复参数；RuntimeContext → _meta，复杂度中等 | 判断可解释；缺声明默认确认；卡片失败可恢复边界明确。依赖框架状态与业务卡片同步，仍无通用远端 exactly-once |
| 业务命令审核表 + 网关授权 + 幂等键 | 审批绑定命令摘要/版本，独立鉴权与幂等，开发运维成本高 | 适合公网、多租户、高风险写入；审批时到执行时业务条件变化仍须服务器重校验 |

**当前实现**：工具 schema 保留 required、additionalProperties、defs；工具重名保护内置工具。同工具多个意图节点任一要求确认即需确认。needsConfirm=`confirmConfigured || !Boolean.TRUE.equals(readOnlyHint)`。工具继承 ToolBase，checkPermissions 返回 ask/allow。确认参数仅从状态取，前端只有 conversationId/messageId/approved。[E13][E14][E18]

**身份与异常**：`RuntimeContext.getUserId` 跨 boundedElastic 显式传递；`_meta` 只在非空时发。服务端默认绑定 127.0.0.1，源码明确 `_meta` 是可信内部数据范围提示、不能当公网认证凭据。MCP 初始化失败跳过注册；Bridge 只抽 TextContent，非文本/structuredContent 不完整保留。ClientExecutor catch Exception 返回普通 isError；因此传输失败、取消和“远端可能已提交”没有通用明确结算协议，不能声称 exactly-once。[E12][E14][E20][E21][E33]

**选择依据**：根 POM 注释记录 AgentScope MCP SDK 与项目 SDK 版本冲突约束，配置明确走既有 rag 连接桥接；这是有记录的兼容性理由，不是本次实际运行验证。da32db76 明确记录 RuntimeContext 身份、_meta 及回环网络边界。bffe0e80 明确记录确认结算延迟。[E03][E05][H01][H02]

**测试与追问**：T02 的 readOnlyHint / additionalProperties /保留工具名断言；T04 的 meta 透传；T03 的准备失败不结卡、结卡失败不订阅。追问入口：readOnlyHint 是否可信？工具确认是否等于业务授权？卡片批准后价格/库存变了怎么办？网络超时远端写入成功后重试是否重复？

**个人贡献**：按最新用户说明，属于独立完成的应用层设计实现。**假设性优化**：用模拟 MCP 服务构造“提交后断响应”，补 commandId 幂等和结果查询；细分 cancellation/unknown-outcome，保留对外可解释状态。需要先决定具体业务工具，不可声称现有桥已经保证幂等。

### C3. Skill 渐进披露与单例 Agent 的会话隔离

**技术问题 / 触发场景**：把所有手册塞进 prompt 增加负载；永久激活写工具会在手册被压缩后继续可用；共享 Toolkit 工具组状态可能被另一个会话覆盖。[E15][E16][T05]

| 方案 | 原理、复杂度和资源成本 | 可靠性、性能、扩展与风险 |
|---|---|---|
| 全量手册 + 全工具可见 | 简单；输入负载随手册数线性增长 | 模型可绕过必要步骤；容易混淆技能，适合很小清单 |
| 原生工具组持久激活 | 框架接入成本低，激活保存在 ToolContext | 仓库复现测试断言原生组状态跨会话污染、正文压缩后仍激活；这是框架复现，不是当前自有遮蔽运行结果 |
| 当前上下文推导 G−L + 执行前检查 | 手册由 load_skill 动态取；每轮扫描上下文，成本随上下文/技能数增加 | 不修改共享工具组；手册不在则收回工具；一个工具多手册任一加载可放行。但异常路径 fail-open，不能当硬权限保障 |

**当前实现**：`load_skill` 描述挂 enabled manifest，schema enum 动态读清单，成功结果 metadata 写 `ragent_loaded_skill`。Masking 每轮只认成功且未裁掉的 TOOL result，计算启用技能 tool_ids 并集减去已加载技能 tool_ids 并集，过滤本轮 schema，并把映射放 RuntimeContext。Bridge 再检查 masked map，模型根据旧调用硬闯也不会执行。[E14][E15][E16]

**效果与局限**：没有 per-session 字段写在共享中间件，适合单例 Agent；启停和手册热更新依赖 registry/cache 的当前值。Masking 81–84 计算异常直接使用全量 tools；该轮还可能沿用 RuntimeContext 上一轮 masked 属性，错误语义需补测试。metadata 仅记录 code，不记录手册版本；旧正文和最新 toolIds 可以发生漂移。修改手册内容不重建 Agent 不等于正在运行的旧手册同步更新。[E15][E16]

**历史支持**：c71f0750 新增自有遮蔽；7e522563 后来新增原生 Skill 生命周期/并发复现。时间顺序是自有实现先于这两个复现测试，不能把后来测试写成早先设计时的已记录决策过程。[H03][H04]

**测试与追问**：T05 用 CountDownLatch、假模型、真实框架对象复现错误；生命周期测试真调用项目 compactor，但 mock summarizer/mapper。它们没有验证当前自有 G−L 隔离在多用户并发下已通过。追问入口：为何激活应依赖上下文而非永久状态？多手册共享工具怎么办？手册被裁掉为何 metadata 还不能证明加载？缓存版本变化如何保证同轮一致？

**个人贡献**：按最新用户说明，属于独立完成的应用层设计实现。**假设性优化**：添加当前 Masking 的双会话确定性测试、异常 fail-closed 或可审计降级策略、loadedSkillVersion 与目录快照绑定。只读搜索和高风险写工具应明确分级后再确定异常策略。

### C4. 在工具协议约束下控制上下文体量

**技术问题 / 触发场景**：检索结果与多轮工具轨迹可膨胀；直接删最老 N 条消息可能留下无配对 tool_result；无条件摘要会引入同步 LLM 成本、丢掉当前问题或未执行完的工具循环。[E11][E24]

| 方案 | 原理、复杂度和资源成本 | 可靠性、性能、扩展与风险 |
|---|---|---|
| 固定滑窗截 N 条消息 | O(n)，没有模型成本，容易实现 | 忽略每条体量与工具配对；重要早期约束丢失，适合纯文本短聊天 |
| 每轮摘要历史 | 语义保留较灵活；模型调用成本和尾延迟高 | 摘要幻觉逐代累积，故障影响主回答；依旧需保护切点与输出格式 |
| 当前白名单裁剪 + 高水位前缀摘要 | 先清可再查的旧结果，再在用户轮起点替换前缀；实现复杂度较高 | 降低无必要摘要调用；保持调用/结果关系；字符代理和逐代摘要仍有质量/硬窗口缺口 |

**当前实现**：Trimmer 先累计字符，分工具循环，保护本轮/未闭合/最近 2 个已完成循环，只替换白名单 tool_result 正文；可回收不足上下文 20% 即放弃；保留 id/name/metadata/state，先构造完整替换映射再修改 context。Middleware 同步替换上行列表引用。[E09][E11][E34]

Compaction 在末条 USER 且超 80% 时尝试；先按引用验证上行列表的前缀与 context 对齐。Compactor 留够最近 20% 字符后向前找真实 USER 切点，可替换素材少于总量一半、只有旧摘要、保留段孤儿工具结果均放弃。旧摘要+新素材生成新摘要，完整构造后一次改 context。[E24][E34]

Summarizer 使用 STANDARD 档阻塞调用放在 boundedElastic，不是并行跳过等待；thinking 不进笔录，单工具结果头 800/尾 400 保留；返回须至少 3 个 `## ` 小节，超长在小节边界截断且至少 2 节。nonce 围栏和用户数据身份降低注入风险，不能证明完全防注入。审计失败不回滚压缩。[E24][E35]

**效果与局限**：保证机制有静态依据；字符口径对非文本块计 0，人设/schema/记忆块不纳入预算，所以不能承诺绝不超过模型上下文。单个巨型本轮/未闭合循环可能无可裁内容；摘要失败退裁剪，裁剪失败原样推理，没有硬拒绝或终极 token 限制。代码先判高水位摘要再回退裁剪，不能描述成“同一次推理必先裁剪、然后再重新计量判断摘要”。[E10][E34]

**历史支持**：33399fc2 引入裁剪，ed059271 添加分层压缩、引用校验和水位；记录支持演进功能，但不证明某个线上 400 故障、成本节省或摘要保真率。[H05]

**测试与追问**：T05 生命周期测试覆盖一种 compactor 真实路径；未发现专门 Trimmer/Compactor/Summarizer JUnit 类。现成回归工具存在，不代表已运行。追问入口：如何定义工具循环闭合？多工具同一 assistant 消息如何配对？当前输入超长为何不压缩？字符与 token 如何校准？如何证明摘要没有丢关键约束？

**个人贡献**：按最新用户说明，属于独立完成的应用层设计实现。**假设性优化**：补参数化工具轨迹测试和性质测试；加入真实 tokenizer 预算与超限策略；用固定对话集对比无压缩/裁剪/摘要的输入 token、首包/全程耗时、关键事实保留率，失败与未知结果保留在分母。

### C5. 异步长期记忆的 claim、版本和水位一致性

**技术问题 / 触发场景**：后台抽取与 flush 可能重叠；同用户多个会话更新同一长期记忆；Judge 慢调用期间已有事实被修改；NOOP 也已处理一批消息，单靠 revision 无法阻止重复处理。[E27][E28][E29]

| 方案 | 原理、复杂度和资源成本 | 可靠性、性能、扩展与风险 |
|---|---|---|
| 每轮直接抽取后写库 | 简单；每轮模型成本高 | 重复抽取、过期快照覆盖、工具幻觉污染；缺恢复台账 |
| 全流程锁住用户 | 串行正确性易理解；锁持续到 LLM 返回 | 同用户全部会话等待慢网络，占连接/锁时间；进程故障与锁续租复杂 |
| 当前无锁 Judge + 会话 claim + 短事务双校验 | 部分唯一索引、控制行 FOR UPDATE、revision+watermark，复杂度高于简单写库 | 慢调用不占行锁；同用户提交串行且跨会话过期可拒绝；冲突后需下一轮触发，非即时自旋重试；无持久队列保障 |

**当前实现**：先在用户消息落库前 ensureControl，控制行创建时刻是抽取下界；失败不阻断对话但可能漏首消息。后台 3 条待处理用户消息才调用 Judge，flush 绕过门槛；每批最多 40 条。claim 先回收 >10 分钟 PROCESSING，再靠 `(user_id,conversation_id) WHERE status='PROCESSING'` 唯一索引抢占。[E27][E28][E29][E36]

Judge 只读用户消息、单条最多 1200 字符；返回 NOOP/ADD/SUPERSEDE/RETRACT。LLM 调用在事务外。commit 锁用户控制行后双检 revision+会话水位；目标 id 必须属于当前 user 的 active 集，正文上限 500；操作/结算同事务；结算不是 PROCESSING 或已被回收则抛错回滚。[E28][E29][E37]

**为什么两把版本尺**：revision 判断全用户事实集是否变；watermark 判断该会话消息段是否处理。没有合并或淘汰的纯 NOOP 不改 revision 却推进水位；只查 revision 会重复处理同一批。快照冲突不消耗内容失败次数；真正模型/提交/容量失败最多 3 次，之后 DROPPED 推进水位。[E28][E29][E30]

**效果与局限**：静态机制针对重复/过期提交，非已证明 exactly-once 或事实准确。3 次后丢弃是明确数据损失策略；“三次”按 `to_message_id` 归并，新增消息改变批尾可能改变统计，不能描述为所有失败内容无条件只尝试三次。10 分钟回收依赖 DB 时钟；用户首基线用应用时钟；多节点钟差/雪花 ID 排序与消息时间应实测。后台是 boundedElastic fire-and-forget，没有持久消息队列，进程退出/用户不再对话后不会保证最终重试或清空超过 40 条积压。[E18][E28][E29]

**历史支持**：1cb684ed 提交记录直接支持基线先于消息、异步后台及开关；双校验动机还见代码注释和 SQL。个人独立开发归属已确认，但不能由这些记录补写具体事故经历。[H06]

**测试与追问**：T06 仅 3 个 ensureBaseline mock 测试；T03 覆盖异步请求线程隔离与基线时序；T08 提供 M1 revision、M2 watermark、M3 flush 门槛、M4 NOOP 等机制回归，需运行服务/模型/PG 且可能 UNCOVERED。追问入口：为何事务外读快照仍正确？NOOP 为什么也推水位？跨会话冲突重试能否饿死？claim 超时后旧 Judge 迟到能否写入？

**个人贡献**：按最新用户说明，属于独立完成的应用层设计实现。**假设性优化**：先用真实 PG、可控制的假 Judge、CountDownLatch 做不同会话同用户竞态、NOOP、僵尸回收后迟到提交测试；再补持久重试/积压调度，记录冲突率、丢弃率和重复条目数。未运行前只能描述为测试设计。

### C6. 不确定模型输出下的长期记忆容量和可追溯更正

**技术问题 / 触发场景**：跨会话全量注入时记忆不可无限增长；用户改口不能并存旧值；盲目重新总结全部记忆容易丢事实，模型还可能给出不存在 ID 或非法合并。[E28][E32][E37][E38]

| 方案 | 原理、复杂度和资源成本 | 可靠性、性能、扩展与风险 |
|---|---|---|
| 无限追加 / 全量注入 | 写入简单，读成本随用户数据增长 | 同义重复/冲突、窗口成本不断增长；缺事实更正链 |
| 向量记忆按需 topK | 记忆量大时输入固定；需 embedding/index/query 和质量评估 | 检索遗漏稳定约束；跨会话更新/撤回也需一致性，适合大量非结构事实 |
| 当前小容量全量注入 + 受限合并/淘汰/拒收 | active事实追加新行，旧行失效并指向 superseded_by；全量块硬限，复杂度中等 | 小集无需检索遗漏；合并后仍满可淘汰或整批拒收，明确牺牲完整性；500 字符上限和全量注入限制规模 |

**当前实现**：容量核算共用 `AgentMemoryBlock.projectedChars`，按完整渲染块核账而不只累加正文。预演顶上限才模型规划合并；事务内重验组至少 2 条、无重复/跨组占用、不是本批决策目标、产物 <=500 且比旧正文总量短。有效 merge 一条新行取代多条旧行。[E28][E32][E38]

合并仍不足时非 FLUSH 来源优先、同档最旧先淘；保护本批目标；纯存量不得跌破记忆块上限 75% 的下限，越界便停止淘。仍放不下抛 CapacityException，淘汰随事务回滚。容量失败走失败结算阶梯。[E28][E27]

**模型边界**：nonce 与围栏中和、仅用户原文抽取、无参 flush、正文/ID 复核降低污染面；代码不能验证模型合并确实保留语义，也不做同义去重算法保障。每条的事实真假信任用户陈述，原始提示词明说不判真假、整体清空要求按 NOOP。记忆块使用 USER 角色和数据说明，仍受模型遵循程度影响。用户撤回与容量淘汰共用 retract，事实表不记录撤回原因，区分需依赖日志，不能称完整事件溯源。[E32][E37][E38][E36][E39]

**选择依据**：全量块容量、受限合并和来源优先是代码体现的取舍；未找到作者比较向量记忆的历史记录，这部分只能合理推演。1cb684ed 证明机制由上游引入。[H06]

**测试与追问**：T07 检查 flush 后 snapshot 刷新和失败反馈；T08 有容量回归 R5、改口 R3、撤回 R2；本次未运行。追问入口：为何 1500 字符够用？合并语义怎么验？容量满时为何可能拒收新事实？用户请求删除全部记忆为何没有执行？为何 flush 不能接 content 参数？

**个人贡献**：按最新用户说明，属于独立完成的应用层设计实现。**假设性优化**：固定事实集测重复、改口、撤回、容量拒收与语义保留；给 DROPPED/容量拒收明确管理界面和重跑路径；大量事实时才比较向量/结构化混合记忆，不应未经实验替换当前小集方案。

## 4. 评分与建议进入五大亮点的方式

每维 0–5 分：深度、业务价值、岗位匹配、**个人开发贡献可信度**、可追问性。下表保留 2026-10-12 的历史分析评分；**原个人贡献 0 分与依赖它的合计已因 2026-10-13 贡献更正失效，不作为当前贡献结论，不在本次重新打分**。其他评分是选题判断，不是测量值。

| 候选 | 深度 | 业务 | 匹配 | 历史个人贡献（已失效） | 追问 | 历史合计 /25（已失效） | 筛选建议 |
|---|---:|---:|---:|---:|---:|---:|---|
| C1 生命周期竞争 | 5 | 5 | 5 | 0 | 5 | 20 | 适合 Java 后端核心工程题；可并入 Agent 执行亮点 |
| C2 MCP/HITL/身份 | 5 | 5 | 5 | 0 | 5 | 20 | 独立核心问题；确认、认证、幂等边界需讲透 |
| C3 技能隔离 | 4 | 4 | 5 | 0 | 5 | 18 | 适合附在 Agent 工具编排；单独写需补自有遮蔽测试 |
| C4 配对安全压缩 | 5 | 4 | 5 | 0 | 5 | 19 | 可和 C5 合成一个记忆亮点，但口述时必须区分短/长期一致性 |
| C5 异步长期记忆一致性 | 5 | 5 | 5 | 0 | 5 | 20 | 个人可补真实 PG 并发实验的优先入口 |
| C6 记忆容量/不确定输出 | 4 | 4 | 5 | 0 | 5 | 18 | 深入追问或补强主题，避免与 C4/C5 三条简历重复 |

与 RAG/模型路由领域合并时，建议形成“Agent 生命周期与技能编排”“MCP 的可恢复确认和身份边界”“上下文安全压缩与异步长期记忆”三个问题族，再由根分析筛选五个。按最新用户说明，可作为独立完成的应用层设计实现亮点；实测效果与后续优化仍需独立记录。

## 5. 静态测试证据与缺口

| 索引 | 文件与案例 | 本次实际证明范围 |
|---|---|---|
| T01 | [AgentRunHandleTest](/Users/alexander/LLM/ragent/agent/src/test/java/com/nageoffer/ai/ragent/agent/service/handler/AgentRunHandleTest.java:129)：shouldNotStartAfterCancellationSettles、shouldKeepGateUntilStartupHandoffCompletes、shouldDisposeSubscriptionWhenStartupFailsAfterSubscribe；[AgentRunGateTest](/Users/alexander/LLM/ragent/agent/src/test/java/com/nageoffer/ai/ragent/agent/service/handler/AgentRunGateTest.java:57)：shouldHoldSlotWithTtlFallback、shouldRejectSecondRunOfSameUser、shouldReleaseOnlyOwnSlot | 测试源码使用假对象/线程竞争；RunGate 的 Redis 行为是 mock 验证，未实测 Redis |
| T02 | [AgentToolCatalogTest](/Users/alexander/LLM/ragent/agent/src/test/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalogTest.java:91)：shouldTreatMcpToolWithoutReadOnlyHintAsWritable、shouldPassThroughMcpReadOnlyHint、shouldAskConfirmForDeclaredWriteToolWhenIntentNodeMisses、shouldPassThroughAdditionalPropertiesToModelSchema、shouldNotLetMcpToolOverrideReservedToolName | 工具契约存在；测试 getter 不等于执行权限认证或端到端批准执行 |
| T03 | [AgentChatServiceImplTest](/Users/alexander/LLM/ragent/agent/src/test/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImplTest.java:418)：shouldGetAgentBeforeWritingConversationOrQuestion、shouldKeepConfirmPendingWhenRegistrationFails、shouldSettleConfirmationAfterPreparationAndBeforeSubscription、shouldNotSubscribeWhenConfirmationSettlementFails；[AgentStreamEventBridgeTest](/Users/alexander/LLM/ragent/agent/src/test/java/com/nageoffer/ai/ragent/agent/service/handler/AgentStreamEventBridgeTest.java:348)：shouldNotSendConfirmCardWhenPersistFails | 模拟失败时序有明确断言，没有真实多节点/网络端到端证据 |
| T04 | [McpToolBridgeTest](/Users/alexander/LLM/ragent/agent/src/test/java/com/nageoffer/ai/ragent/agent/tool/McpToolBridgeTest.java:72)：shouldPassLoginUserThroughMeta、shouldPassEmptyMetaWithoutIdentity、shouldNeverFallBackToSingleArgumentOverload | mock executor 捕获 _meta；没有公网身份可信性或 OAuth 证据 |
| T05 | [AgentScopeToolGroupConcurrencyTest](/Users/alexander/LLM/ragent/agent/src/test/java/com/nageoffer/ai/ragent/agent/skill/AgentScopeToolGroupConcurrencyTest.java:60)：shouldReproduceCrossSessionToolGroupPollution、shouldExecuteSkillToolWithoutConcurrentSession；[AgentScopeSkillActivationLifecycleTest](/Users/alexander/LLM/ragent/agent/src/test/java/com/nageoffer/ai/ragent/agent/skill/AgentScopeSkillActivationLifecycleTest.java:66)：shouldKeepSkillToolActiveAfterLoadResultIsCompactedOut | 测试明确断言原生工具组的错误/激活残留行为，不是当前自有遮蔽已经修复并通过测试；本次未运行 |
| T06 | [AgentMemoryPipelineTest](/Users/alexander/LLM/ragent/agent/src/test/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryPipelineTest.java:45)：shouldEnsureControlWhenLongTermEnabled、shouldSkipBaselineWhenLongTermDisabled、shouldSwallowBaselineFailure | 只有基线开关/异常 mock 测试；不覆盖仓库真实事务与双校验 |
| T07 | [MemoryFlushToolTest](/Users/alexander/LLM/ragent/agent/src/test/java/com/nageoffer/ai/ragent/agent/tool/MemoryFlushToolTest.java:62)：shouldRefreshSnapshotAfterWrite、shouldRefreshSnapshotWhenSettledEmptyButMutated、shouldReportFailureForEveryNonWritingOutcome、shouldRejectCallWithoutIdentity | mock pipeline 结局和快照反馈，不证明 Judge 质量或容量算法语义正确 |
| T08 | [AgentMemoryMechanismMain](/Users/alexander/LLM/ragent/resources/regression/agent-memory/AgentMemoryMechanismMain.java:67)：M1版本号、M2水位、M3门槛、M4 NOOP、M6撤回、M7失效目标、M8首消息、R1跨会话、R2撤回持久、R3改口、R4台账、R5容量 | 独立 main 回归工具存在；需要 live 服务/模型/PG。统计 FAIL/UNCOVERED，SUCCESS 可带未覆盖项，不能把它等同全部案例通过 |
| T09 | [ReActAgentProviderTest](/Users/alexander/LLM/ragent/agent/src/test/java/com/nageoffer/ai/ragent/agent/config/ReActAgentProviderTest.java:110)：shouldResolveToolCatalogOncePerRequest、shouldBuildToolkitFromTheSnapshotItsFingerprintCameFrom、shouldRebuildWhenMcpToolAppears、shouldKeepModelRetriesOffSoHalfStreamFailureIsNotReplayed | 目录/实例快照与重试配置断言存在；未提供真实模型半流失败/重建压力结果 |

## 6. 上游历史证据

下列 commit 的 author/committer 记录为 `ding.ma`；个人独立开发归属按用户全部业务代码独立完成的最新说明确认，不再因作者名不同否定个人贡献。提交时间反映仓库演进顺序，不等于故障发生时间；独立开发说明不能补出未记录的生产事故或实验结果。

| 索引 | commit / 时间 | 直接支持的变更 | 不支持的说法 |
|---|---|---|---|
| H01 | bffe0e80 / 2026-09-18 | Git show 真实改动：闸门后检查 pending、先获取 Agent 后写用户消息、确认卡片延迟结算、release 顺序与测试 | 用户本人遇到并修复线上故障；消灭所有重复业务执行 |
| H02 | da32db76 / 2026-09-14 | _meta 透传登录身份、RuntimeContext 取身份、MCP 回环绑定；正文声明开放网需认证方案 | 实现 OAuth、多租户权限治理或公网认证 |
| H03 | c71f0750 / 2026-09-03 | 新增 SkillLoadTool 与 Masking 源码，技能清单/目录注册 | 作者在实现前已用后来测试复现框架缺陷 |
| H04 | 7e522563 / 2026-09-18 | 新增原生 Skill 激活周期和共享 Toolkit 并发复现测试，含假模型/latch断言 | 本次运行通过；当前自有遮蔽修复有完整回归覆盖 |
| H05 | 33399fc2 / 2026-08-23；ed059271 / 2026-08-26 | 裁剪引入→分层压缩、引用校验、模型调度和回归工具演进 | 提示词成本降低百分比、特定供应商线上 400 次数下降 |
| H06 | 1cb684ed / 2026-08-30 | 新增长期记忆、异步后台、首条基线时序、开关、记忆工具和数据库资源 | 记忆准确率、实测吞吐、生产稳定性 |

复核命令：`git log --format='%h %ad %an <%ae> %s' --date=short -- <scope>`；`git show --format=fuller --stat <commit> -- <scope>`；H01 另读取 AgentChatServiceImpl 的实际 diff，H03 另读取 Masking 的新增 diff。未读 README 或既有面试文档作证。

## 7. 源码定位索引

链接指向当前工作区绝对路径；行号为本次基线实际位置，后续源码改变需要刷新。每项列出类/方法及相关范围，不能只凭索引标题归纳超出实现的能力。

| 索引 | 文件 / 类 / 方法 | 核心行范围 |
|---|---|---|
| E01 | [application.yaml](/Users/alexander/LLM/ragent/bootstrap/src/main/resources/application.yaml:35) | 35–64 Agent；147–150 MCP |
| E02 | [ConditionalOnAgentEngine](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/config/ConditionalOnAgentEngine.java:34) | 34 配置条件 |
| E03 | [pom.xml](/Users/alexander/LLM/ragent/pom.xml:203) | 32 AgentScope版本；203–214 SDK兼容记录与依赖 |
| E04 | [ReActAgentProvider#getAgent/buildAgent](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/config/ReActAgentProvider.java:71) | 71–110 快照缓存；113–135 装配 |
| E05 | [AgentEngineConfiguration#agentChatModel](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/config/AgentEngineConfiguration.java:45) | 45–81 独立模型与formatter |
| E06 | [AgentProperties](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/config/AgentProperties.java:35) | 35–49 迭代/尝试/SSE参数 |
| E07 | [AgentChatController#chat/confirm/stop](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/controller/AgentChatController.java:46) | 46–65 REST入口 |
| E08 | [AgentRunGate#acquire/release/ttl](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentRunGate.java:53) | 53–59 抢位；83–93 CAS/TTL |
| E09 | [AgentMemoryProperties](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryProperties.java:38) | 38–110 常量/默认值；115–159 派生 |
| E10 | [AgentContextChars#total/of](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentContextChars.java:31) | 31–86 字符代理 |
| E11 | [AgentContextTrimmer#trimInPlace/protectedCycles/apply](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentContextTrimmer.java:68) | 68–92 门槛；98–184循环；210–245 原位替换 |
| E12 | [McpClientAutoConfiguration#registerRemoteTools](/Users/alexander/LLM/ragent/rag/src/main/java/com/nageoffer/ai/ragent/rag/core/mcp/McpClientAutoConfiguration.java:65) | 65–95 客户端发现/失败 |
| E13 | [AgentToolCatalog#resolve/buildToolkit/resolveMcpTools/McpToolBinding.of](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/AgentToolCatalog.java:79) | 79–105 注册；138–177 交集；198–239 契约；267–296 指纹 |
| E14 | [McpToolBridge#checkPermissions/callAsync/execute](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/McpToolBridge.java:75) | 75–95 权限/遮蔽；128–164 身份/结果；181–189文本 |
| E15 | [SkillLoadTool#getDescription/getParameters/execute/success](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/skill/SkillLoadTool.java:74) | 74–97 manifest/schema；113–143取正文/metadata |
| E16 | [AgentSkillMaskingMiddleware#onReasoning/maskedTools/loadedSkillCodes](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/skill/AgentSkillMaskingMiddleware.java:76) | 76–95过滤/异常；101–136 G−L |
| E17 | [AgentToolBatchMiddleware#onActing](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/AgentToolBatchMiddleware.java:56) | 56–73 批边界；84–95 前执行短路 |
| E18 | [AgentChatServiceImpl#streamChat/startRun/startConfirmRun/launchStream/bindReleaseHooks](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentChatServiceImpl.java:83) | 83–118闸门；138–155基线；161–215确认；221–283启动/释放；343–375异步/断链 |
| E19 | [AgentRunHandle#start/interruptUpstream/settle](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentRunHandle.java:126) | 43等待常量；126–140启动；169–217取消；253–294收尾 |
| E20 | [McpClientToolExecutor#execute/buildRequest](/Users/alexander/LLM/ragent/rag/src/main/java/com/nageoffer/ai/ragent/rag/core/mcp/McpClientToolExecutor.java:49) | 49–82远端调用/异常/_meta |
| E21 | [McpCallMeta#ofUser](/Users/alexander/LLM/ragent/rag/src/main/java/com/nageoffer/ai/ragent/rag/core/mcp/McpCallMeta.java:25) | 25–48身份信任边界 |
| E22 | [AgentStreamEventBridge#onRequireUserConfirm/settleAwaitingConfirm](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/service/handler/AgentStreamEventBridge.java:275) | 275–303 awaiting；341–368持久卡片 |
| E23 | [AgentConversationServiceImpl#getPendingConfirm/settleConfirmBlock](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/service/impl/AgentConversationServiceImpl.java:164) | 164–254归属/状态校验与卡片改写 |
| E24 | [AgentContextCompactor#compactInPlace/findSafeCutoff/hasOrphanToolResult](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentContextCompactor.java:69) | 69–121前提与提交；127–146审计；152–192配对切点 |
| E25 | [AgentStateMapper#upsert](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/dao/mapper/AgentStateMapper.java:33) | 33–51 JSONB无版本upsert |
| E26 | [PgAgentStateStore#save/get/getList](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/state/PgAgentStateStore.java:48) | 48–78框架codec；105–106匿名哨兵 |
| E27 | [AgentMemoryPipeline#extract/runExtraction/planConsolidation](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryPipeline.java:67) | 53–61基线；67–122管道；130–139合并预演 |
| E28 | [AgentMemoryRepository#claim/commit/apply/consolidate/evict](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryRepository.java:169) | 68–85限额；154–203claim/失败；215–279提交；285–309应用；330–402合并；409–488淘汰/结算 |
| E29 | [AgentMemoryExtractionMapper#selectWatermark/settle/recycleStale/selectSpentAttempts](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/dao/mapper/AgentMemoryExtractionMapper.java:32) | 32–85台账SQL |
| E30 | [AgentMemoryControlMapper#ensureExists/selectForUpdate/bumpRevision](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/dao/mapper/AgentMemoryControlMapper.java:34) | 34–70控制行SQL |
| E31 | [AgentUserMemoryMiddleware#dispatch/inject](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentUserMemoryMiddleware.java:59) | 59–84快照；90–128上行注入 |
| E32 | [AgentMemoryBlock#render/projectedChars](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryBlock.java:34)；[MemoryFlushTool#execute/refreshSnapshot](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/tool/MemoryFlushTool.java:65) | Block34–92渲染核账；Flush65–73无参；82–143刷新/反馈 |
| E33 | [McpToolResults#userId](/Users/alexander/LLM/ragent/mcp-server/src/main/java/com/nageoffer/ai/ragent/mcp/executor/McpToolResults.java:47)；[MCP application.yml](/Users/alexander/LLM/ragent/mcp-server/src/main/resources/application.yml:1) | Results47–56信任边界；yml1–3回环 |
| E34 | [AgentContextCompactionMiddleware#dispatch/compact/resolvePrefix](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentContextCompactionMiddleware.java:62) | 62–108压缩/回退；115–154同步上行 |
| E35 | [AgentConversationSummarizer#summarize/validate](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentConversationSummarizer.java:87) | 55–71素材/结构；87–128调用；135–168围栏；174–208结构校验 |
| E36 | [长期记忆 SQL](/Users/alexander/LLM/ragent/resources/database/upgrades/v2.0.0/260827_agent_long_term_memory.sql:5) | 5–64表/部分唯一索引；67–109抽取规则；112–141合并规则 |
| E37 | [AgentMemoryJudge#judge/parse/toDecision/renderTurns](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryJudge.java:78) | 54素材限额；78–103调用；109–147解析；174–208用户素材/围栏 |
| E38 | [AgentMemoryConsolidator#plan/call/parse](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/memory/AgentMemoryConsolidator.java:77) | 77–108计划/降级；114–158输出预算/解析 |
| E39 | [AgentMemoryMapper#supersede/retract](/Users/alexander/LLM/ragent/agent/src/main/java/com/nageoffer/ai/ragent/agent/dao/mapper/AgentMemoryMapper.java:31) | 31–51用户/生效条件更新与撤回原因边界 |

## 8. 完成检查

历史源码分析只新增本文；本次更正贡献口径，无业务源码、配置或测试修改，无新增第三方依赖。Git交付与最终候选筛选由主报告统一管理；本附录的六个候选是细分问题，与主报告的十二项编号分别使用，不做直接合计。运行验证需要可用 Java、PG、Redis、MCP Server 和模型端点，后续实验必须使用独立测试账号/数据并保存失败与未覆盖记录。个人独立开发归属已确认，仍须避免把静态实现、复现测试源码或设计意图误写成实测结果或具体生产事故经历。
