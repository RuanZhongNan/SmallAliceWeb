# 2026-09-05 基于 inkeep/agents 本地源码的 RAG 项目借鉴调研报告

> 报告类型：实现级对标调研报告（重调研）
> 调研对象：本地仓库 `D:\code\store\agents__inkeep`（inkeep/agents @ commit `602e36b`，2026-09-04）
> 服务对象：`packages/ai-rag-api`（Nitro + Neon PostgreSQL 混合检索）与智能客服升级路线
> 关联文档：[主调研报告](./README.md)、[learn-agents-ui spec](./learn-agents-ui/spec.md)、[learn-agents-ui plan](./learn-agents-ui/plan.md)、[docs-assistant 对标报告](./2026-09-05-learn-docs-assistant-report.md)
> 证据档案：[探索笔记 A：core+sdk](./research-notes/A-core-sdk.md)、[探索笔记 B：api 运行时](./research-notes/B-api-runtime.md)、[探索笔记 C：cookbook](./research-notes/C-cookbook-cli.md)

---

## 一、调研背景与执行方式

### 1.1 为什么重调研

此前三份文档（主调研报告、learn-agents-ui spec/plan、docs-assistant 对标报告）的结论来自云端代理的调研与 GitHub 远程抽查，停留在**战略与设计层**。本次拿到本地完整源码（commit `602e36b`，比云端调研时更新一天），目标是把「借鉴什么」落到**实现级**：哪些机制可以逐行对照着抄进我们的升级 1-4，哪些早期结论需要修正，哪些新发现是早期报告完全没有的。

### 1.2 执行方式与诚实披露

按用户要求采用 agent team 蜂群架构（探索/编辑/复核三角色分工）。执行中 Agent 子代理基础设施连续三次返回并发上限错误（并行三发与单发重试均失败），**探索阶段降级为主代理串行执行**，但保留了蜂群的产物结构：三份领域探索笔记落盘为证据档案（见文首链接），报告只做汇总与判断，引用全部可回溯到笔记与源码行号。复核环节在报告完成后重试独立子代理复核，仍被并发限制拦截后降级为「带源码实证的清单式自审」（9 项引用逐条通过 grep/sed 核验）。

### 1.3 路线修订：2026-09-05 用户约束（重要，优先级高于既有路线）

用户在任务台账（`prompts/01.prompts.md`）中给出四条约束，**本报告据此修订 docs-assistant 对标报告提出的升级 1-4 路线**：

1. **我们不做多轮会话** —— 原升级 1（真·多轮对话）整体不采用；`conversationId` 的定位从「多轮历史」改为「单轮问答的分组与追溯标识」。
2. **不接入 MCP** —— 项目是 web 内 RAG 智能客服，无 node 执行环境、不代用户执行操作行为，故不提供 MCP 能力，也不考虑 RAG 项目接入 MCP。原升级 2（检索工具化）失去主要前提（多轮查询改写），**暂缓**，保留为待确认选项。
3. **要做模型更换与切换能力** —— 新增为正式路线项（本报告 4.1），现有双协议注册表是其地基。
4. **升级 4 的会话回流在不做多轮的前提下重新设计** —— 结论是可以做好，重构为「单轮回流评估」，见 4.3 与 3.5 的修订解读。

因此本报告第三章中 3.1/3.2 标注为**备查档案**（设计留存，若未来重启多轮再启用），3.3-3.6 直接服务修订后的路线。

---

## 二、早期结论的本地源码重核

| 早期结论                                                                                     | 核实结果                                                 | 证据                                                                                                                                                                            |
| :------------------------------------------------------------------------------------------- | :------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| ContextConfig 支持 `timeout` / `requiredToFetch`（必需变量缺失则跳过该次 fetch）             | 属实                                                     | `agents-core/src/context/ContextConfig.ts:63-64、440-447`                                                                                                                       |
| manage/runtime 两库分离                                                                      | 属实，且评估配置/运行也分属两库                          | `db/manage/manage-schema.ts`（1603 行）、`db/runtime/runtime-schema.ts`（1300 行）、`data-access/manage/evalConfig.ts`（1381 行）与 `data-access/runtime/evalRuns.ts`（858 行） |
| AgentSession 约 2197 行                                                                      | 属实（实测 2196 行）                                     | `agents-api/src/domains/run/session/AgentSession.ts`                                                                                                                            |
| 对话历史压缩存在（compression 事件）                                                         | 属实且**比早期认知重得多**：独立压缩模块四文件共 1493 行 | `domains/run/compression/{BaseCompressor,ConversationCompressor,MidGenerationCompressor,reconcileToolPairs}.ts`                                                                 |
| 工具审批机制（PendingToolApprovalManager）                                                   | 属实：审批实现 + 双路由消费                              | `domains/run/agents/tools/tool-approval.ts`，`routes/conversations.ts`、`routes/chatDataStream.ts`                                                                              |
| docs-assistant 三文件、prompt 两句话                                                         | 属实                                                     | `template-projects/docs-assistant/`（同 commit 早前已读原文）                                                                                                                   |
| cookbook 顶层为 template-projects / evals / template-mcps                                    | 属实（template-mcps 含 slack/zendesk/vercel 三套）       | 目录列表                                                                                                                                                                        |
| customer-support 为 Coordinator+KB+Zendesk 三代理                                            | 属实（5 文件）                                           | `template-projects/customer-support/agents/customer-support.ts`                                                                                                                 |
| （早期未提及）**TemplateEngine**：prompt 模板变量引擎，JMESPath 取值                         | 新发现                                                   | `agents-core/src/context/TemplateEngine.ts:86-94`                                                                                                                               |
| （早期未提及）**MidGenerationCompressor + reconcileToolPairs**：生成中途压缩与工具对守恒修复 | 新发现                                                   | `domains/run/compression/`                                                                                                                                                      |
| （早期未提及）**sandbox 执行器家族**与 **ttft-recorder**                                     | 新发现                                                   | `domains/run/tools/{Native,Vercel}SandboxExecutor.ts`、`domains/run/stream/ttft-recorder.ts`                                                                                    |

---

## 三、实现级借鉴清单（映射升级路线）

以下六项按「能直接抄进我们代码」的标准筛选，每项含源码证据与落地归属。原升级路线定义见 [docs-assistant 对标报告](./2026-09-05-learn-docs-assistant-report.md)，**现按 1.3 的用户约束执行**：升级 1 不做、升级 2 暂缓、升级 3 保留、升级 4 重构为单轮回流，并新增模型更换与切换能力。

### 3.1 历史压缩三件套（备查档案：当前路线不做多轮，见 1.3）

- **触发公式**：`hardLimit - contextSize <= safetyBuffer` 即压缩（`BaseCompressor.ts:854` 的 `triggerAt = hardLimit - safetyBuffer`；`ConversationCompressor.isCompressionNeeded` 同式）。比「按消息条数截断」精确，且 `getModelAwareCompressionConfig(summarizerModel, 0.5)` 按 summarizer 模型窗口比例折算 hardLimit（`BaseCompressor.ts:985`）。
- **摘要是一个类型化工具**：`distillConversationHistory` 产出 `ConversationHistorySummary` 结构体，且支持传入 `priorSummary` 做**链式增量摘要**（`ConversationCompressor.ts:5-31`）——旧摘要不重算，增量滚动。
- **生成中途也能压**：`MidGenerationCompressor.ts`（217 行）处理长回答生成过程中的上下文膨胀。
- **落地启示（备查）**：若未来重启多轮，历史注入按「hardLimit/safetyBuffer 双参数 + 摘要工具 + priorSummary 链式」实现；摘要模型用注册表里的轻模型即可。当前路线不实施。

### 3.2 reconcileToolPairs：工具对守恒安全网（备查档案：多轮 + 工具循环场景的前置坑）

`reconcileToolPairs.ts:1-13` 注释原文记录了一个真实事故类别：压缩重写消息数组会切断 tool-call/tool-result 配对，上游以 `Tool results are missing for tool calls` 拒绝整个请求。其修复策略是**只删除未配对侧、绝不合成占位**（对比 SDK 的 `pruneMessages` 会把工具部分整体剥掉）。

- **落地启示（备查）**：若未来实施历史裁剪或工具循环，必须内置「tool-call/result 配对校验」，把该文件的策略（配对图遍历 + 只删不造 + 只动被修改消息）作为规格写进 openspec change。当前路线不实施。

### 3.3 ContextConfig + TemplateEngine（服务升级 3）

- `fetchDefinition` 声明式配置（`ContextConfig.ts:396-447`）：`timeout`、`requiredToFetch`（必需变量解析失败则**跳过该次 fetch 而非失败**）——上下文获取的降级语义已经内置。
- `TemplateEngine.ts:86-94`：`{{variable.path}}` 用 **JMESPath** 从类型化上下文取值渲染 prompt，专用 `PromptRenderOptions`。
- **落地启示**：升级 3 的「页面上下文注入」直接采用 `fetchDefinition` 形状（URL + zod schema + timeout + requiredToFetch）；prompt 模板用 `{{context.currentPagePath}}` 风格变量，模板引擎我们自己写 30 行正则替换即可（无需 JMESPath 全量）。

### 3.4 会话 ID 模板变量回链（服务单轮回流与未来外部回链）

customer-support 的 Zendesk 代理在创建工单时注入 `{{$conversation.id}}` 自定义字段，注释明言「so support engineers can trace the ticket back to this conversation」（`customer-support/agents/customer-support.ts:33`，自审实证）。

- **落地启示**：这是「对话与外部系统双向可追溯」的模式样板。不做多轮**不影响**此模式——conversationId 作为单轮问答的追溯标识照样成立：每次 `/v1/chat` 请求携带（或由服务端生成）conversationId，写入日志、评估记录与将来的工单/邮件回链，同一模板变量机制复用。

### 3.5 conversationEvaluation：触发式评估 + trace 双向链接（服务单轮回流评估）

- 会话结束时触发：`triggerConversationEvaluation({tenantId, projectId, conversationId, resolvedRef})` 读取 manage 库中激活的 `EvaluationRunConfigsWithSuiteConfigs`（isActive 过滤）→ 按套件的 agentIds 过滤 → 从 run 库取会话 → 启动 `evaluateConversationWorkflow`（`domains/evals/services/conversationEvaluation.ts:34-90`）。
- **评估运行与 trace 双向链接**：cookbook evals 样例在 `startActiveSpan('chat-api-call')` 内跑真实调用，取 OTel `traceId`，再 `item.link(traceRef, runLabel)` 把数据集条目与真实 trace 互链（`evals/langfuse-dataset-example/langfuse-dataset.ts:150-158、185-200`）。
- **落地启示（单轮版重构）**：不做多轮时该模式**完整可用**——① 单轮 Q/A 落库（一张 `qa_records` 表即可，无需 conversations/messages 双表与历史链）；② 每条记录落库后触发评估钩子（Nitro 事件，替代其会话结束钩子）；③ 评估运行携带 `conversation_id` + `trace_id`，与数据集条目互链；④ 不引 Langfuse 与 durable workflow，钩子内同步或队列执行。

### 3.6 流式可观测：TTFT 记录与增量解析（补充长板）

`domains/run/stream/` 除增量解析器（`IncrementalStreamParser.ts`，含 DataComponent 解析）外，还有专用 **ttft-recorder**（time-to-first-token 记录）。

- **落地启示**：我们的 chat-api spec 已把 120 秒/420 秒作为上游时间线验证点（openspec Requirement 9），但未常态化记录 TTFT。在 `/v1/chat` 管线加 TTFT 打点（首字节到达时间入日志/表）成本极低，是可观测性长板的第一块砖。

---

## 四、修订后的路线执行建议

| 路线项                                     | 从 inkeep 抄什么                                                                                                                                                 | 我们的落地方案                                                                                                                                                                      | 优先级 | 核心验收                                                                        |
| :----------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :----- | :------------------------------------------------------------------------------ |
| 模型更换与切换（新增）                     | ai-sdk-provider 的 adapter 分文件分层（convert/prompt/finish-reason/error 四件套，见探索笔记 C 第五章）；`getModelAwareCompressionConfig` 的「按模型参数化」思路 | 双协议注册表泛化：类型化 provider 注册表（protocol/baseUrl/model/参数）+ 可切换 `activeProvider` + 每 provider 独立超时与 TTFT 观测；以 openspec change 修订 chat-api Requirement 8 | **P0** | 切换 provider 无需改业务代码；切换后流式与来源帧行为不变；TTFT 分 provider 可查 |
| prompt 配置化 + 页面上下文注入（原升级 3） | ContextConfig fetchDefinition 形状、TemplateEngine 变量渲染、`requiredToFetch` 降级语义                                                                          | 30 行自写模板替换；zod 校验页面上下文；上下文获取失败跳过不阻断                                                                                                                     | P1     | 页面上下文用例通过；提示词修改免发版                                            |
| 单轮回流评估（原升级 4 重构）              | conversationEvaluation 触发链路、trace↔dataset 双向链接、`{{$conversation.id}}` 回链模式                                                                         | 单轮 `qa_records` 落库 + 评估钩子；评估运行带 conversation_id/trace_id；gold-set 扩展单轮真实问题                                                                                   | P2     | 真实问答回流为评估集；trace 可回查                                              |
| 横切：可观测                               | ttft-recorder                                                                                                                                                    | /v1/chat 首字节打点（随模型切换项一起做，分 provider 记录）                                                                                                                         | P1     | TTFT 出现在日志与验证报告                                                       |
| 检索工具化（暂缓）                         | `tool.with({selectedTools})` 最小暴露面                                                                                                                          | 前提（多轮改写）已移除；若答案质量评估显示单次检索不足再议                                                                                                                          | 暂缓   | —                                                                               |
| 多轮会话机制（备查）                       | 3.1/3.2 全部设计                                                                                                                                                 | 设计留存于本报告与探索笔记，不立项                                                                                                                                                  | 备查   | —                                                                               |
| 并行：ai-vue P0-P4                         | DataComponent zod 契约                                                                                                                                           | 按 learn-agents-ui plan 第十三章执行                                                                                                                                                | 并行   | 见该 plan 验收清单                                                              |

## 五、与既有文档的关系

三份既有文档与本报告**分层互补**：主调研报告（README.md）= 战略层；learn-agents-ui spec/plan = 前端组件层；docs-assistant 对标报告 = 架构对标层；**本报告 = 实现级层，并按 1.3 的用户约束对 docs-assistant 报告的升级 1-4 路线作出修订**（升级 1 不做、升级 2 暂缓、升级 4 重构为单轮回流、新增模型切换项）。后续立项以本报告第四章为准。

## 六、边界与不引入清单

- **不引入**：DoltgreSQL/dolt、SpiceDB 授权、durable workflow 引擎、agents-manage-ui、agents-cli 平台同步、Langfuse（模式自实现）、**MCP 协议栈与一切工具执行能力（用户明确约束：web RAG 无 node 执行环境、不代用户操作，见 1.3）**、多轮会话机制（用户明确约束，设计备查）。
- **理由**：均为平台级重依赖或与产品定位冲突，与我们「自有栈 + 零平台依赖 + ELv2 规避」的既定边界一致（主调研报告已论证）。
- **暂列备查**：credential-stores 多后端抽象、external-fetch 安全套件、sandbox 执行器、多轮压缩三件套与 reconcileToolPairs（重启多轮时启用）。

## 七、结论

本地源码重调研把「学习 inkeep/agents」推进到实现级施工图，并按用户当日约束完成了路线收敛：**模型更换与切换能力**升格为 P0（双协议注册表泛化 + ai-sdk-provider 式 adapter 分层 + 分 provider TTFT 观测）；**prompt 配置化与页面上下文注入**（ContextConfig/TemplateEngine 形状）为 P1；**单轮回流评估**（conversationEvaluation 触发链路 + trace 双向链接，单轮形态完整可用）为 P2；多轮与 MCP 相关设计全部转入备查档案。三份探索笔记证实早期战略结论无一被推翻，并补充 TemplateEngine、MidGenerationCompressor、reconcileToolPairs、sandbox 家族、ttft-recorder 五项早期盲区。下一步以第四章「模型更换与切换」立项 openspec change，本报告第三章作为其 design 输入。
