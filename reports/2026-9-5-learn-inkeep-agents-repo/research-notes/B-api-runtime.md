# 探索笔记 B：agents-api 运行时与评估

> 仓库：D:\code\store\agents\_\_inkeep @ commit `602e36b`（2026-09-04）

## 一、AgentSession 结构解剖

- `agents-api/src/domains/run/session/AgentSession.ts` = **2196 行**（早期报告称 2197，差 1 行，视为属实）。
- domains/run 全景子域：`a2a / agents / artifacts / compression / context / data / errors / handlers / routes / services / session / stream / tools / types / utils / workflow`；另有顶层域 `evals / manage / mcp / credential-gateway`。
- run/tools 内含 **sandbox 执行器家族**：NativeSandboxExecutor / VercelSandboxExecutor / SandboxExecutorFactory（代码执行沙箱，早期报告未提及）。

## 二、对话历史压缩专题（升级 1 直接参照）

- 独立模块 `domains/run/compression/`：BaseCompressor.ts（995）、ConversationCompressor.ts（163）、**MidGenerationCompressor.ts**（217，生成中途压缩）、reconcileToolPairs.ts（118）。
- **触发公式**：`hardLimit - contextSize <= safetyBuffer` 时压缩（BaseCompressor.ts:854 `triggerAt = hardLimit - safetyBuffer`；ConversationCompressor.isCompressionNeeded 同式）。
- **模型感知配置**：`getModelAwareCompressionConfig(summarizerModel, 0.5)`（BaseCompressor.ts:985）——按 summarizer 模型上下文窗口比例算 hardLimit。
- **摘要是一个工具**：`distillConversationHistory`（run/tools/distill-conversation-history-tool.ts，另有 distill-conversation-tool.ts 与 distill-utils.ts）产出类型化 `ConversationHistorySummary`；ConversationCompressor 支持传入 `priorSummary` 做**链式增量摘要**（ConversationCompressor.ts:5-12、31）。
- **工具对守恒**：reconcileToolPairs.ts:1-13 注释明言——压缩重写消息数组会切断 tool-call/tool-result 配对，上游以 "Tool results are missing for tool calls" 拒绝；修复策略是**只删除未配对侧、绝不合成占位**（SPEC D1/D2/D3 安全网）。
- 压缩全程 OTel 打点：`SPAN_KEYS.COMPRESSION_TRIGGER_AT / COMPRESSION_OVERAGE`（BaseCompressor.ts:870-872）。

## 三、工具调用循环与审批专题（升级 2 参照）

- 审批实证：`domains/run/agents/tools/tool-approval.ts` 定义，`routes/conversations.ts` 与 `routes/chatDataStream.ts` 消费（grep -l 证据）。
- 沙箱执行：工具代码可跑在 Native/Vercel 两种沙箱（工厂切换）。

## 四、评估框架专题（升级 4 直接参照）

- evals 域：services/{EvaluationService.ts 536、**conversationEvaluation.ts 210**、datasetRun.ts 500、evaluationJob.ts 71} + workflow/{routes.ts 150、steps/、functions/}。
- **conversation 触发式评估数据流**（conversationEvaluation.ts:34-90）：会话结束时 `triggerConversationEvaluation({tenantId, projectId, conversationId, resolvedRef})` → 读 manage 库激活的 `EvaluationRunConfigsWithSuiteConfigs`（isActive 过滤）→ 按 suite 的 agentIds 过滤 → 从 **run 库**取 conversation → `workflow/api` 的 `start(evaluateConversationWorkflow)`。
- 双库在运行时并用的实证：同文件内 `manageDbPool`（配置）与 `runDbClient`（会话）并存。
- core 侧配套：`data-access/manage/evalConfig.ts`（1381）+ `data-access/runtime/evalRuns.ts`（858）。

## 五、可观测性专题

- SDK 自定义 `TelemetrySpan` 接口（telemetry-provider.ts:39-55）抽象 OTel，provider 无关。
- stream 域有 **ttft-recorder**（首 token 延迟记录）——与我们 120s/420s 上游时间线验证实践同构。
- 压缩、评估均有 span 属性打点。

## 六、借鉴设计汇总

| #   | 设计                              | 证据                            | 对应升级    | 成本       |
| :-- | :-------------------------------- | :------------------------------ | :---------- | :--------- |
| 1   | 压缩触发公式 + 模型感知 hardLimit | BaseCompressor.ts:854、985      | 升级 1      | 低         |
| 2   | 摘要即工具 + priorSummary 链式    | ConversationCompressor.ts:5-31  | 升级 1      | 中         |
| 3   | reconcileToolPairs 只删不造       | reconcileToolPairs.ts:1-40      | 升级 1+2    | 低         |
| 4   | 会话结束触发对话评估              | conversationEvaluation.ts:34-90 | 升级 4      | 中         |
| 5   | TTFT 记录器                       | run/stream/ttft-recorder.ts     | 流式可观测  | 低         |
| 6   | 工具审批暂停/恢复                 | tool-approval.ts                | 升级 2 远期 | 高（暂缓） |

## 七、与早期调研结论的出入核对

| 早期结论                         | 核实结果                                                      | 证据                            |
| :------------------------------- | :------------------------------------------------------------ | :------------------------------ |
| AgentSession 约 2197 行          | 属实（2196）                                                  | wc -l                           |
| compression 事件存在             | 属实且更重——是独立压缩模块而非单点事件                        | domains/run/compression/ 四文件 |
| PendingToolApprovalManager 存在  | 审批机制属实（tool-approval.ts + 双路由消费）；类名细节未展开 | grep -l 证据                    |
| Scheduler/Trigger/Webhook 三服务 | services 目录存在（本次未逐一展开）                           | domains/run/services/           |

## 八、不推荐借鉴

- `workflow/api` durable workflow 引擎：重依赖，我们用 Nitro 后台任务/队列即可。
- a2a 域、Dolt 分支管理、credential-gateway：当前无场景。
