# 探索笔记 A：agents-core 与 agents-sdk

> 仓库：D:\code\store\agents\_\_inkeep @ commit `602e36b`（2026-09-04）

## 一、包结构与核心文件清单

| 路径                                                              | 行数           | 职责                           |
| :---------------------------------------------------------------- | :------------- | :----------------------------- |
| `packages/agents-core/src/validation/schemas.ts`                  | 4109           | 全实体 zod/openapi schema 工厂 |
| `packages/agents-core/src/data-access/manage/agentFull.ts`        | 2177           | Agent 完整配置聚合读写         |
| `packages/agents-core/src/db/manage/manage-schema.ts`             | 1603           | 管理库 Drizzle schema          |
| `packages/agents-core/src/db/runtime/runtime-schema.ts`           | 1300           | 运行库 Drizzle schema          |
| `packages/agents-core/src/data-access/manage/evalConfig.ts`       | 1381           | 评估配置数据访问               |
| `packages/agents-core/src/data-access/runtime/evalRuns.ts`        | 858            | 评估运行数据访问               |
| `packages/agents-core/src/context/ContextConfig.ts`               | 456            | 动态上下文声明式配置           |
| `packages/agents-core/src/context/TemplateEngine.ts`              | 317            | prompt 模板变量引擎            |
| `packages/agents-core/src/external-fetch/`                        | 4 文件         | 外部资源下载安全套件           |
| `packages/agents-sdk/src/agent.ts` / `subAgent.ts` / `project.ts` | 1244/1163/1013 | 构建器三件套                   |
| `packages/agents-sdk/src/telemetry-provider.ts`                   | 626            | 自定义 TelemetrySpan 抽象      |
| `packages/agents-sdk/src/tool.ts`                                 | 274            | 工具构建器                     |

core 顶层目录含：`api-client auth constants context credential-stores credential-stuffer data-access data-reconciliation db(dolt) evaluation external-fetch middleware retry setup text-attachments types utils validation`。

## 二、值得借鉴的设计

1. **ContextConfig 声明式 fetch 定义**（ContextConfig.ts:53-71、396-447）：`fetchDefinition` 构建器携带 `timeout?` 与 `requiredToFetch?: Array<string>`，注释明确「required 变量无法解析时跳过该次 fetch」——非必需上下文失败不阻断主流程。→ 升级 3（页面上下文注入），成本**低**。
2. **TemplateEngine JMESPath 模板变量**（TemplateEngine.ts:86-94）：`template.replace(/\{\{([^}]+)\}\}/g, ...)` 按 JMESPath 从上下文取值替换 `{{variable.path}}`，另有 `PromptRenderOptions` 专门服务 prompt 渲染。→ 升级 3 的 prompt 模板化直接参照，成本**低**。
3. **manage/runtime 双库分离**（manage-schema.ts / runtime-schema.ts）：配置类实体（agent/工具/评估配置）与运行类实体（conversations/messages/evalRuns）分库分访问层。→ 升级 1 的 `conversations`/`messages` 表设计与命名参照，成本**低**（我们单库，但 schema 分域可照搬）。
4. **dataComponent = zod props 声明**（cookbook/ticketcard-data.ts：六字段全部 `z.string().describe(...)`）：组件契约即 schema，描述即文档。→ ai-vue P3.5 卡片契约参照，成本**低**。
5. **`tool.with({ selectedTools })` 工具子集裁剪**（customer-support/agents/customer-support.ts:26-33）：同一 MCP 工具可裁剪出只含 `create_zendesk_ticket` 的暴露面并注入 headers。→ 升级 2 的工具暴露面最小化原则，成本**低**。
6. **credential-stores 多后端抽象**（memory/keychain/nango/composio + credential-stuffer）：凭证与工具解耦、调用时注入。→ 暂缓（环境变量已够用），列出备查。
7. **external-fetch 安全套件**（file-url-security.ts 120 行 / file-content-security.ts 159 行 / constants / errors 四件分层）：URL 校验、内容校验、错误分层独立成文件。→ 未来附件/外部抓取场景备用，成本**中**。

## 三、ContextConfig 专题

- 配置面（字段级）：`fetchDefinition{ url, method?, headers?(headers() 构建器 + zod), body?, timeout?, requiredToFetch? }`，`contextConfig<CV>` 泛型携带各变量 zod schema（ContextConfig.ts:16-28 类型推导 `ExtractSchemasFromCV`）。
- 执行接线：声明在 core/context；执行与缓存位于 agents-api `domains/run/context/` 与 `data-access/contextCache`（runtime 侧有 contextCache 数据访问与 scoping 测试）。
- 校验：全部走 zod（`@hono/zod-openapi`），失败按真实错误暴露。

## 四、DataComponent 事件协议专题

- SDK 侧：`dataComponent({ id, name, description, props: z.object({...}) })`——纯 schema 声明。
- 流侧：agents-api `domains/run/stream/IncrementalStreamParser.ts` 负责从流中增量解析（含 DATA_COMPONENT 标记，streaming-integration.test.ts 覆盖）；stream 域另有 ResponseFormatter、durable-stream-helper、stream-buffer-registry、ttft-recorder。
- 前端渲染：由宿主组件按名称注册渲染器（我们 ai-vue P3.5 的 itemType 方案对齐此设计）。

## 五、与早期调研结论的出入核对

| 早期结论                                    | 核实结果                                                                | 证据                                                     |
| :------------------------------------------ | :---------------------------------------------------------------------- | :------------------------------------------------------- |
| ContextConfig 支持 requiredToFetch/timeout  | 属实                                                                    | ContextConfig.ts:63-64、440-447                          |
| manage/runtime 两库分离                     | 属实                                                                    | db/manage/manage-schema.ts、db/runtime/runtime-schema.ts |
| data-reconciliation 用 diff+effect handlers | 目录属实（src/data-reconciliation/），effect handler 细节本次未逐行核实 | 目录存在性确认                                           |

## 六、不推荐借鉴的部分

- DoltgreSQL/dolt 目录（版本化数据库）：运维过重，Neon 即可。
- auth（SpiceDB 细粒度授权）：超出单租户文档客服需要。
- 4109 行 validation/schemas.ts 全家桶：我们只需按需定义自己的 zod 契约。
