# 探索笔记 C：cookbook、cli 与 provider

> 仓库：D:\code\store\agents\_\_inkeep @ commit `602e36b`（2026-09-04）

## 一、模板项目全景表

| 模板                                                                          | 结构要点                                                                 | 场景                 |
| :---------------------------------------------------------------------------- | :----------------------------------------------------------------------- | :------------------- |
| docs-assistant                                                                | 3 文件：index.ts + agents/docs-assistant.ts + tools/inkeep-rag-mcp.ts    | 文档问答（最简样板） |
| customer-support                                                              | 5 文件：index + coordinator/kb/zendesk 三代理 + ticketcard + 双 MCP 工具 | 客服协调             |
| deep-research / meeting-prep / copilot / facts-project / activities-planner×3 | 其余场景模板                                                             | 研究/会议/副驾等     |
| evals/langfuse-dataset-example                                                | OTel trace ↔ 数据集条目链接样例                                          | 评估闭环             |
| template-mcps/                                                                | slack/、vercel-template/、zendesk/                                       | 预置 MCP 服务端      |

## 二、客服可借鉴的 prompt 与结构模式

1. **Coordinator 编号化委派 prompt**（customer-support/agents/customer-support.ts:36-44）：「For each inquiry: 1. First, delegate to the knowledge base agent…… 2. If……cannot provide a satisfactory answer, delegate to the Zendesk agent 3. Ensure smooth handoff……」——步骤化委派指令写法。
2. **升级/转人工话术**（同文件:15-19）：KB 代理 prompt 明示「If you cannot find a satisfactory answer, clearly indicate that you need to escalate to Zendesk support」——把"何时转人工"写进专用代理的 prompt 而非协调者。
3. **会话 ID 模板变量回链**（同文件 zendesk 代理 prompt）：创建工单时写入 `{{$conversation.id}}` 自定义字段，「so support engineers can trace the ticket back to this conversation」——对话与外部系统（工单）双向可追溯。→ 升级 4 会话回流的绝佳启示。
4. **工具子集 + headers 占位符模板**（同文件:26-33）：`zendeskMcpTool.with({ selectedTools: ['create_zendesk_ticket'], headers: { 'zendesk-subdomain': '{{YOUR_ZENDESK_SUBDOMAIN}}'… } })`——最小暴露面 + 配置占位符。
5. **DataComponent 全 describe 契约**（data-components/ticketcard-data.ts）：ticket_id/subject/status/priority/created/url 六字段全部 `z.string().describe(...)`。
6. **两种 prompt 风格按复杂度选用**：docs-assistant 两句话（简单场景），customer-support coordinator 步骤化（复杂场景）——prompt 长度跟随职责复杂度，不堆砌。

## 三、customer-support 深度解剖

- 数据流：用户 → coordinator（defaultSubAgent）→ 委派 KB（先）→ 不满再委派 zendesk（后）→ zendesk 生成工单时产出 zendeskTicketCard DataComponent → 前端按组件名渲染卡片。
- ticketcard 六字段：ticket_id / ticket_subject / ticket_status / ticket_priority / ticket_created / ticket_url。

## 四、evals 样例的 trace→dataset 链路（升级 4 参照）

langfuse-dataset.ts（273 行）关键流：

1. 自建 OTel `NodeTracerProvider` + `tracer = otelTrace.getTracer('dataset-runner')`（:2-10）。
2. 逐条数据集项在 `tracer.startActiveSpan('chat-api-call', …)` 内执行真实调用（:185）。
3. 取 `spanContext?.traceId` 作为本次运行的 trace 标识（:198-200，无 traceId 视为告警）。
4. `context.langfuse.trace({ id: result.traceId })` → `item.link(traceRef, runLabel, …)` 把**数据集条目与真实 trace 双向链接**（:150-158）。

自有化启示：我们不需要 Langfuse——模式是「评估运行必须携带 trace_id 并与数据集条目互链」，落在我们 Neon 的评估表加 trace_id 列即可。

## 五、ai-sdk-provider 与双协议启示

- `packages/ai-sdk-provider/src/` 共 10 文件：inkeep-chat-language-model.ts / inkeep-chat-prompt.ts / inkeep-chat-options.ts / convert-to-inkeep-messages.ts / map-inkeep-finish-reason.ts 等——标准的 AI SDK LanguageModelV1 适配器分层（请求转换、prompt 转换、finish reason 映射各自独立文件）。
- 启示：我们 ai-rag-api 的双协议注册表（Anthropic Messages 激活 / OpenAI Responses 备用）未来若引第三方协议，可按「convert / prompt / finish-reason / error 四件套分文件」组织 adapter。

## 六、manage-ui 的配置驱动理念

- 本次仅核到数据访问层形态：`agentFull.ts`（2177 行）/`projectFull.ts`（1617 行）即"完整配置聚合读写"的载体——配置变更经由聚合对象整体 reconcile。
- 启示：我们的 prompt 配置化（升级 3）先做代码内类型化模块即可，配置上库与聚合 reconcile 是后期形态，不必一步到位。

## 七、与早期调研结论的出入核对

| 早期结论                                              | 核实结果                       | 证据                              |
| :---------------------------------------------------- | :----------------------------- | :-------------------------------- |
| docs-assistant 为 3 文件、prompt 仅两句               | 属实（同 commit 早前已读原文） | template-projects/docs-assistant/ |
| cookbook 顶层为 template-projects/evals/template-mcps | 属实                           | 目录列表                          |
| customer-support 为 Coordinator+KB+Zendesk 三代理     | 属实（5 文件）                 | agents/customer-support.ts        |

## 八、不推荐借鉴

- agents-manage-ui 整体（React 全家桶）：理念可学，代码不搬。
- slack/vercel MCP 模板：暂无集成场景。
- agents-cli push/pull：依赖平台侧，无对应物。
