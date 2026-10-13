# 第 5 条：RAG 模型路由与流式容错 STAR 面试稿

日期：2026-10-13（Asia/Shanghai）。源码基线：`4d5360857a22671ddb30ce544cd86b42af1860be`。本章按本人独立完成项目业务代码的口径编写。

## 简历原文

> 为 RAG 模型调用实现档位路由、候选降级与三态熔断，在流式首包探测成功后提交缓冲，隔离首包探测失败候选的输出；首包等待中断时释放半开名额，避免取消阻塞后续探测。

## STAR 拆解

**S — 场景：** RAG 链路里的问题改写、摘要和最终回答对模型能力与等待时间的需求不同。接入多个 Provider 后，还需要处理启动失败、首包超时、空流和用户主动取消。流式输出一旦发送给前端，就无法通过切换模型撤回前面的内容。

**T — 任务：** 我需要把任务选模型、故障时换候选、熔断准入和流式输出控制收敛到统一调用层，明确允许切换模型的阶段，同时避免中断请求长期占住半开探测名额。

**A — 行动：** 我用 `ModelSelector` 根据档位构造有序候选，处理 preferred、启用状态与思考能力过滤；再由 `ModelHealthStore` 的 CLOSED、OPEN、HALF_OPEN 状态控制实际准入。同步调用交给 `ModelRoutingExecutor`，真实模型故障尝试下一候选，取消异常直接退出。流式调用增加 `ProbeStreamBridge`：先暂存回调，首包探测成功后才提交给前端；探测结果为超时、错误或无内容完成时，取消当前句柄并尝试下一候选。内容或思考回调先将探测置为成功后，后来的错误不会改变这一结果。半开准入返回带令牌的 `CallPermit`，首包等待被中断时先取消句柄，再在 finally 中释放当前令牌持有的名额。

**R — 结果：** 我实现了档位选型、首包探测失败时的候选切换和熔断恢复控制；探测失败候选产生的回调不会被转发给用户，迟到的旧许可也无法释放新的半开名额。首包成功后继续由当前候选输出，后续错误不自动拼接其他模型的答案。

以上 S/T 是设计约束的叙述，R 是代码实现的能力；没有加入未记录的线上事故、可用性提升或延迟数字。

## 约 90–120 秒连续口述稿

这一块是我独立实现的 RAG 模型调用层。问题改写、摘要和最终回答的模型需求不同，接入多个供应商以后，我还需要处理启动失败、首包超时和用户停止。尤其是流式回答，前端已经收到一段内容后再换模型，容易拼出语义不连续的答案。

所以我先把模型选择和调用执行拆开。选择层按 fast、standard、deep 档位生成有序候选，并过滤未启用或不支持思考的模型；调用层再用关闭、打开、半开三种状态决定是否放行。同步调用中，模型故障可以尝试下一个候选，但主动取消直接退出，不继续降级。

流式调用的重点是首包提交边界。我先用桥接器暂存内容、思考、完成和错误回调，只有首包探测成功才提交缓冲；如果探测结果是超时、错误或无内容完成，就取消当前请求，尝试下一个候选。内容或思考先到后，后来的错误沿当前流转发，不再换模型。这样探测失败候选的回调不会污染前端输出。

我还给半开探测发放带令牌的许可。等待首包被中断时，先取消请求，再在 finally 中释放匹配令牌的名额，避免旧请求释放新探测的许可。最终交付的是探测失败可切换、探测成功后保持同一输出来源的容错机制。这套机制用于 RAG 调用层，主 Agent 的独立单模型配置不经过它。

时长为准备稿长度估算，可按实际语速调整。

## 核心源码文件：按这个顺序阅读

| 顺序 | 文件与关键入口 | 作用及面试关注点 |
| --- | --- | --- |
| 1 | [ModelSelector.java:77](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/model/ModelSelector.java#L77)，`selectChatCandidates` / `resolveTierName` / `buildTierTargets` | 任务到档位、有序候选、preferred 去重及思考能力过滤；thinking 对应 deep 配置优先于显式 override |
| 2 | [ModelHealthStore.java:63](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/model/ModelHealthStore.java#L63)，`allowCall` / `markSuccess` / `markFailure` | 原子更新健康状态，半开时只允许一个探测；状态保存在进程内，不是跨节点共享熔断器 |
| 3 | [ModelRoutingExecutor.java:42](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/model/ModelRoutingExecutor.java#L42)，`executeWithFallback` | 同步候选循环；识别取消后不标失败、不继续降级，释放匹配半开许可 |
| 4 | [RoutingLLMService.java:115](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/chat/RoutingLLMService.java#L115)，`streamChat` | 逐候选准入、启动 Provider、探测首包、取消失败候选及终止降级 |
| 5 | [ProbeStreamBridge.java:44](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/chat/ProbeStreamBridge.java#L44)，`onContent` / `onThinking` / `awaitFirstPacket` / `commit` | 首包可以是内容或思考回调；提交前缓冲，成功后转发，不等于已经校验答案质量 |
| 6 | [RoutingLLMService.java:182](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/chat/RoutingLLMService.java#L182)，`awaitFirstPacket` | 中断标志恢复，先取消句柄、finally 释放半开许可；不保证所有启动阶段异常都按取消分类 |
| 7 | [ModelHealthStore.java:143](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/model/ModelHealthStore.java#L143)，`releaseHalfOpenPermit` | 仅释放状态和令牌均匹配的当前许可，防止旧请求误释放新探测 |

## 支撑文件与配置

| 文件 | 作用 |
| --- | --- |
| [LlmFirstPacketProbe.java:35](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/chat/LlmFirstPacketProbe.java#L35) | 独立 Spring Bean 承接首包等待，避免 self-call 绕开追踪 AOP |
| [ChatClient.java:28](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/chat/ChatClient.java#L28) | 统一 Provider 同步与流式调用契约 |
| [AbstractOpenAIStyleChatClient.java:166](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/chat/AbstractOpenAIStyleChatClient.java#L166) | OpenAI 兼容 Provider 的 HTTP 流读取、响应回调和取消句柄 |
| [ModelTarget.java](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/model/ModelTarget.java) | 路由后的候选、Provider、端点及超时预算 |
| [TaskCancellation.java](../framework/src/main/java/com/nageoffer/ai/ragent/framework/cancellation/TaskCancellation.java) | 区分中断、取消与模型错误，处理异常链和线程中断标志 |
| [AIModelProperties.java](../infra-ai/src/main/java/com/nageoffer/ai/ragent/infra/config/AIModelProperties.java) | Provider、候选、档位、失败阈值和打开时长配置模型 |
| [application.yaml:230](../bootstrap/src/main/resources/application.yaml#L230) | 熔断阈值 2 次、打开时长 30000 ms；这是配置，不是实测效果 |
| [application.yaml:260](../bootstrap/src/main/resources/application.yaml#L260) | 默认 standard，思考 deep；fast / standard / deep 等待预算分别 5000 / 120000 / 180000 ms，流式路径用作首包预算 |
| [application.yaml:42](../bootstrap/src/main/resources/application.yaml#L42) | 主 Agent 独立单模型且无此 fallback，防止把 RAG 路由扩展成整个 Agent 的降级机制 |

本条不依赖新增业务数据库表。健康状态存于 `ConcurrentHashMap`，多个节点各自维护。

## 调用链

同步路径：

```text
RoutingLLMService.chat
→ ModelSelector.selectChatCandidates
→ ModelRoutingExecutor.executeWithFallback
→ ModelHealthStore.allowCall
→ ChatClient.chat
→ 成功 markSuccess；模型失败 markFailure 后尝试下一候选
→ 取消则 releaseHalfOpenPermit 并退出
```

流式路径：

```text
RoutingLLMService.streamChat
→ ModelSelector.selectChatCandidates
→ ModelHealthStore.allowCall：取得 CallPermit
→ ChatClient.streamChat：回调先进入 ProbeStreamBridge
→ LlmFirstPacketProbe.awaitFirstPacket
  ├─ 首包成功：commit → markSuccess → 返回当前句柄
  ├─ 超时 / 错误 / 空流：markFailure → cancel → 下一候选
  └─ 等待被中断：恢复中断标志 → cancel → finally 释放匹配许可
```

候选选择不是模型健康的最终准入；实际调用还必须经过 `allowCall`。不要把首包等待预算说成完整回答的端到端耗时上限。

## 相关测试文件

| 文件 | 源码中的断言范围 |
| --- | --- |
| [ModelSelectorTest.java:103](../rag/src/test/java/com/nageoffer/ai/ragent/infra/model/ModelSelectorTest.java#L103) | 默认档位、thinking 优先、override、preferred 去重、能力过滤、熔断候选排除 |
| [ModelRoutingExecutorTest.java:42](../infra-ai/src/test/java/com/nageoffer/ai/ragent/infra/model/ModelRoutingExecutorTest.java#L42) | 中断和取消不继续降级；包装异常识别；真实模型失败仍降级 |
| [RoutingLLMServiceHalfOpenRecoveryTest.java:93](../rag/src/test/java/com/nageoffer/ai/ragent/infra/chat/RoutingLLMServiceHalfOpenRecoveryTest.java#L93) | 中断释放名额、旧许可不误释放、先取消后释放，以及取消句柄抛错仍释放名额 |

本次文档任务没有执行这些测试，不将静态断言写成运行通过。

## 三个高频追问

**为什么只在首包前切换？**

当前切换条件是首包探测结果，而不是检查提交之前发生的所有事件。若首个回调是错误、无内容完成，或等不到首包，可以丢弃缓冲并重试；若内容或思考先将探测置为成功，随后即使错误先于 commit 到达，也不会覆盖探测结果，仍提交当前流的内容与错误。用户可见输出一旦开始，另一个模型无法保证续写同一答案，所以当前实现不在流中自动切换。

**为什么半开许可需要令牌，而不只用布尔值？**

布尔值只能表达“有人占用”。旧请求可能在新一轮半开探测已经开始后才结束，若它直接清布尔值，会释放新请求占用的名额。令牌标识这次占用，只有当前令牌持有者才能释放。

**取消为什么不能直接算失败？**

同步调用中的用户停止或线程中断并不说明模型故障，误计会触发无意义熔断和继续降级。首包等待中断同样应取消并释放名额。实现上要区分这两条已处理路径与 Provider 启动阶段的普通异常，不能笼统声称所有取消来源都已统一覆盖。

返回[五条 STAR 总览](ragent-star-interview-guide.md)。
