# Spec：ai-vue 子包学习 @inkeep/agents-ui 的能力差距与目标

> 文档类型：spec（规范文档）
> 创建日期：2026-09-05
> 所属报告：`reports/2026-9-5-learn-inkeep-agents-repo/`
> 目标子包：`@ruan-cat-drill-doc/ai-vue`（当前版本 0.0.1）
> 参考对象：`@inkeep/agents-ui`（当前版本 0.17.8）

---

## 一、背景与动机

### 1.1 当前 ai-vue 子包的定位

`@ruan-cat-drill-doc/ai-vue` 是 SmallAliceWeb monorepo 内的 Vue 3 组件库，为 VitePress 文档站提供 AI 聊天能力。当前它包含两个组件：

- `AiChat.vue`：嵌入式聊天面板，支持消息列表、来源链接、流式 Markdown 渲染、停止生成
- `AiChatFloatingButton.vue`：悬浮按钮 + 侧边抽屉，点击展开 `AiChat`

配套一个 `useMockAiChat` composable 用于本地无网络的模拟对话。实际的 RAG 聊天由 `ai-vitepress-plugins` 包内的 `useKnowledgeChat` composable 驱动，它基于 `@ai-sdk/vue` 的 `useChat` 与后端 Nitro API 的 `/v1/chat` 通信。

### 1.2 为什么选择 @inkeep/agents-ui 作为学习对象

`@inkeep/agents-ui` 是 Inkeep 公司开源的 React 聊天组件库（独立发布到 npm，当前 0.17.8 版本），是经过生产验证的企业级 AI 聊天 UI 方案。它具备以下值得学习的特质：

1. **多形态部署**：提供 EmbeddedChat（嵌入式）、SidebarChat（侧边栏）、ChatButton（悬浮按钮）、ModalChat（弹窗）、SearchBar（搜索栏）等多种组件形态，覆盖文档站、SaaS 产品、营销页面等不同场景
2. **深度品牌化**：通过 `primaryBrandColor` 单一入口自动生成完整色板，支持 `UserProvidedColorScheme` 精细覆盖、`IkpTheme` 主题令牌系统、CSS 变量前缀定制
3. **Shadow DOM 隔离**：通过 Shadow DOM 将组件样式与宿主页面隔离，避免 CSS 冲突，确保在任何宿主环境下视觉一致
4. **富聊天体验**：支持自定义消息渲染（`ComponentsConfig`）、工具调用审批 UI（`IkpTool`）、表单收集（`openForm`）、反馈机制、消息操作菜单、示例问题、引导消息等
5. **事件驱动架构**：通过 `InkeepCallbackEvent` 暴露完整的用户行为事件链，便于埋点分析和质量优化
6. **双模式嵌入**：既支持 React 组件直接引入，也支持通过 `<script>` 标签 + `Inkeep.EmbeddedChat()` 函数在非 React 环境嵌入

### 1.3 核心矛盾

ai-vue 当前是一个**最小可用**的聊天组件，仅满足"能对话、能看来源"的基本需求。而 SmallAliceWeb 正在向"AI 智能客服"方向演进，需要更丰富的交互能力、更强的品牌定制、更可靠的样式隔离。直接引入 `@inkeep/agents-ui` 不可行（它是 React 组件库，SmallAliceWeb 使用 Vue），但其设计理念和架构模式值得深度借鉴。

---

## 二、能力差距分析

### 2.1 组件形态丰富度

| 能力            | @inkeep/agents-ui             | ai-vue（当前）         | 差距评估 |
| --------------- | ----------------------------- | ---------------------- | -------- |
| 嵌入式聊天      | `InkeepEmbeddedChat`          | `AiChat`               | 基本对齐 |
| 悬浮按钮 + 抽屉 | `InkeepChatButton`            | `AiChatFloatingButton` | 基本对齐 |
| 侧边栏聊天      | `InkeepSidebarChat`           | ❌ 无                  | 缺失     |
| 弹窗聊天        | `InkeepModalChat`             | ❌ 无                  | 缺失     |
| 搜索栏          | `InkeepSearchBar`             | ❌ 无                  | 缺失     |
| 搜索 + 聊天组合 | `InkeepEmbeddedSearchAndChat` | ❌ 无                  | 缺失     |

**差距说明**：ai-vue 只有 2 种形态，agents-ui 有 6+ 种。侧边栏和弹窗形态在 SaaS 产品集成场景中需求很高。

### 2.2 品牌化与主题系统

| 能力         | @inkeep/agents-ui                                 | ai-vue（当前）         | 差距评估 |
| ------------ | ------------------------------------------------- | ---------------------- | -------- |
| 品牌色入口   | `primaryBrandColor` 自动生成色板                  | ❌ 无自动色板          | 重大差距 |
| 色板精细覆盖 | `UserProvidedColorScheme`（11 个色阶）            | 仅 CSS 变量覆盖        | 重大差距 |
| 主题令牌系统 | `IkpTheme`（colors/fontFamily/fontSize/zIndex）   | ❌ 无                  | 重大差距 |
| CSS 前缀定制 | `prefix` 配置（默认 `ikp`）                       | ❌ 固定 `ai-chat` 前缀 | 中等差距 |
| 暗色模式同步 | `ColorModeProviderProps`（system/forced/storage） | 仅 CSS 媒体查询        | 中等差距 |
| 组织名展示   | `organizationDisplayName`                         | ❌ 无                  | 低差距   |

**差距说明**：ai-vue 的品牌化能力非常薄弱。当前 `styles/index.scss` 虽然定义了 CSS 变量（如 `--ai-chat-primary-color`），但需要使用者手动覆盖每个变量，没有"输入一个品牌色，自动生成完整色板"的能力。agents-ui 通过 `colorjs.io` 库从 `primaryBrandColor` 自动派生出 11 个色阶（lighter/light/medium/strong 等），极大降低了品牌定制成本。

### 2.3 样式隔离

| 能力             | @inkeep/agents-ui   | ai-vue（当前） | 差距评估 |
| ---------------- | ------------------- | -------------- | -------- |
| Shadow DOM 隔离  | ✅ 内置 Shadow 组件 | ❌ 无          | 重大差距 |
| 样式冲突风险     | 极低                | 高（全局 CSS） | 重大差距 |
| 第三方嵌入可行性 | 高                  | 低             | 重大差距 |

**差距说明**：ai-vue 当前使用全局 SCSS，当被嵌入到第三方页面时，宿主页面的 CSS 可能污染聊天组件的样式（如 `button`、`input`、`a` 等元素的全局样式）。agents-ui 通过 Shadow DOM 彻底隔离了组件样式，这是"可嵌入"能力的基础。

### 2.4 富聊天体验

| 能力           | @inkeep/agents-ui                      | ai-vue（当前）    | 差距评估 |
| -------------- | -------------------------------------- | ----------------- | -------- |
| Markdown 渲染  | ✅ react-markdown + remark-gfm + prism | ✅ markstream-vue | 基本对齐 |
| 来源引用展示   | ✅ citation 组件                       | ✅ 来源链接列表   | 基本对齐 |
| 自定义消息渲染 | `ComponentsConfig`（按名称注册渲染器） | ❌ 无             | 重大差距 |
| 工具调用 UI    | `IkpTool`（含审批按钮）                | ❌ 无             | 重大差距 |
| 表单收集       | `openForm(formSettings)`               | ❌ 无             | 中等差距 |
| 反馈机制       | `InkeepFeedback`（正/负面 + 详情）     | ❌ 无             | 中等差距 |
| 消息操作菜单   | `CustomMessageAction[]`                | ❌ 无             | 中等差距 |
| 示例问题       | `exampleQuestions`                     | ❌ 无             | 低差距   |
| 引导消息       | `introMessage`                         | ❌ 无             | 低差距   |
| 文件附件       | `FileUIPart`                           | ❌ 无             | 中等差距 |

**差距说明**：ai-vue 的聊天体验停留在"文本 + 来源链接"阶段。agents-ui 支持通过 `ComponentsConfig` 注册自定义渲染器，实现工单卡片、订单状态、数据图表等富组件渲染，这是"智能客服"区别于"文档问答"的关键能力。

### 2.5 事件与可观测性

| 能力         | @inkeep/agents-ui                       | ai-vue（当前）              | 差距评估 |
| ------------ | --------------------------------------- | --------------------------- | -------- |
| 用户行为事件 | `InkeepCallbackEvent`（15+ 事件类型）   | ❌ 仅 send/stop/clear-error | 重大差距 |
| 反馈事件     | `AssistantPositiveFeedbackSubmitted` 等 | ❌ 无                       | 重大差距 |
| 消息完成事件 | `AssistantAnswerDisplayed`              | `onResponseComplete`        | 部分对齐 |
| 用户升级事件 | `UserEscalationIndicatedEvent`          | ❌ 无                       | 中等差距 |

**差距说明**：ai-vue 的事件系统非常简单，只有 3 个 emit 事件。agents-ui 暴露了完整的用户行为事件链，包括消息提交、回复展示、反馈提交、清除点击、分享点击、升级指示等，为数据分析和质量优化提供了基础。

### 2.6 嵌入方式

| 能力            | @inkeep/agents-ui                     | ai-vue（当前） | 差距评估    |
| --------------- | ------------------------------------- | -------------- | ----------- |
| Vue 组件引入    | ❌（React）                           | ✅             | ai-vue 优势 |
| npm 包安装      | ✅                                    | ✅             | 对齐        |
| script 标签嵌入 | ✅（`@inkeep/agents-ui-js-cloud`）    | ❌             | 重大差距    |
| 函数式挂载      | `Inkeep.EmbeddedChat(target, config)` | ❌             | 重大差距    |

**差距说明**：agents-ui 提供了独立的 JS Cloud 包，允许非 React 环境（如纯 HTML 页面、WordPress、Shopify）通过 `<script>` 标签 + 函数调用的方式嵌入聊天。这极大降低了集成门槛。ai-vue 目前只能在 Vue 环境使用。

---

## 三、目标与非目标

### 3.1 目标

1. **建立品牌化主题系统**：实现"输入一个品牌色，自动生成完整色板"的能力，支持精细覆盖和主题令牌系统
2. **实现 Shadow DOM 样式隔离**：使 ai-vue 组件可安全嵌入任意宿主页面而不受 CSS 污染
3. **扩展组件形态**：新增 SidebarChat 和 ModalChat 两种形态，覆盖更多集成场景
4. **增强富聊天体验**：支持自定义消息渲染器（DataComponent）、消息操作菜单、反馈机制、示例问题
5. **完善事件系统**：暴露完整的用户行为事件链，为埋点分析提供基础
6. **提供函数式嵌入能力**：支持非 Vue 环境通过 script 标签 + 函数调用嵌入聊天

### 3.2 非目标

1. **不直接移植 agents-ui 的 React 代码**：ai-vue 是 Vue 组件库，所有实现基于 Vue 3 Composition API
2. **不引入 Radix UI / Zag.js 等无障碍基础库**：保持 ai-vue 的依赖精简，使用 element-plus 和 vue-element-plus-x 作为 UI 基础
3. **不实现搜索能力**：搜索栏（SearchBar）形态暂不实现，聚焦聊天能力增强
4. **不实现可视化构建器**：agents-ui 的拖拽画布能力不在范围内
5. **不引入 colorjs.io 依赖**：使用更轻量的颜色计算方案（如纯函数或 colord 库）

---

## 四、验收标准

### 4.1 品牌化主题系统

- [ ] 提供 `primaryBrandColor` 配置项，输入一个 CSS 颜色值
- [ ] 从品牌色自动派生出至少 8 个色阶（lighter/light/medium/strong/stronger 等）
- [ ] 支持 `customColorScheme` 精细覆盖任意色阶
- [ ] 支持 `theme` 主题令牌系统（colors/fontFamily/fontSize/zIndex）
- [ ] 支持 `prefix` 配置项自定义 CSS 变量前缀
- [ ] 暗色模式可通过 `colorMode` 配置项控制（light/dark/system）

### 4.2 Shadow DOM 样式隔离

- [ ] 提供 `ShadowRoot` 包装组件，将聊天组件渲染在 Shadow DOM 内
- [ ] Shadow DOM 模式下，宿主页面 CSS 不影响组件内部样式
- [ ] 提供 `variant` 配置项（`no-shadow` / `container-with-shadow`）控制是否启用 Shadow DOM
- [ ] Shadow DOM 内的字体、图标等资源正确加载

### 4.3 组件形态扩展

- [ ] 新增 `AiSidebarChat` 组件，从右侧滑入的侧边栏聊天
- [ ] 新增 `AiModalChat` 组件，居中弹窗式聊天
- [ ] 两种新形态均支持品牌化主题系统和 Shadow DOM 隔离

### 4.4 富聊天体验

- [ ] 支持 `customComponents` 配置项，按名称注册自定义消息渲染器
- [ ] 自定义渲染器接收 `props` 和 `renderMarkdown` 函数
- [ ] 支持 `messageActions` 配置项，为每条消息添加操作菜单（复制、分享、反馈等）
- [ ] 支持 `feedbackOptions` 配置项，启用正/负面反馈按钮
- [ ] 支持 `exampleQuestions` 配置项，在空状态展示示例问题
- [ ] 支持 `introMessage` 配置项，展示引导消息

### 4.5 事件系统

- [ ] 暴露 `onChatEvent` 回调，接收完整的用户行为事件
- [ ] 事件类型至少包含：消息提交、回复展示、反馈提交、清除点击、分享点击
- [ ] 每个事件携带 `conversationId`、`messageId`、`tags` 等上下文信息

### 4.6 函数式嵌入

- [ ] 提供独立的 JS 包（如 `@ruan-cat-drill-doc/ai-vue-embed`），支持 script 标签引入
- [ ] 提供 `mountAiChat(target, config)` 函数，将聊天挂载到指定 DOM 节点
- [ ] 函数式嵌入同样支持品牌化、Shadow DOM、事件回调等全部能力

---

## 五、约束与风险

### 5.1 技术约束

1. **Vue 3 Composition API**：所有组件基于 `<script setup>` 语法，不使用 Options API
2. **TypeScript 严格模式**：所有新增类型必须完整定义，禁止 `any`
3. **peerDependencies 精简**：不新增重型依赖，品牌色计算使用纯函数或轻量库（< 5KB）
4. **构建产物兼容**：保持 Vite 构建产物的 ESM + CJS 双格式，支持 SSR

### 5.2 兼容性约束

1. **向后兼容**：现有 `AiChat` 和 `AiChatFloatingButton` 的 API 不能破坏性变更
2. **VitePress 集成**：`ai-vitepress-plugins` 的 `useKnowledgeChat` 必须无缝对接增强后的组件
3. **element-plus 共存**：不能与宿主页面的 element-plus 版本冲突

### 5.3 风险

1. **Shadow DOM 的 SSR 兼容性**：Shadow DOM 在服务端渲染时需要特殊处理，VitePress 的 SSR 模式可能需要降级为非 Shadow 模式
2. **品牌色自动派生的准确性**：从单一颜色派生完整色板需要颜色空间转换（HSL/OKLCH），不同色相的派生效果可能不一致
3. **函数式嵌入包的体积**：独立 JS 包需要包含 Vue 运行时，体积可能较大（> 100KB），需评估 tree-shaking 和按需加载策略
4. **自定义渲染器的类型安全**：`ComponentsConfig` 的动态注册机制在 TypeScript 中难以实现完全类型安全，需要权衡灵活性与类型严格性

---

## 六、术语表

| 术语                             | 含义                                                                                                             |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| 品牌色（primaryBrandColor）      | 使用者提供的单一 CSS 颜色值，作为主题色板生成的种子                                                              |
| 色板（ColorScheme）              | 从品牌色派生出的多个色阶，用于不同 UI 层级（背景、边框、文字、强调等）                                           |
| 主题令牌（Theme Token）          | 结构化的主题配置对象，包含 colors/fontFamily/fontSize/zIndex 四类令牌                                            |
| Shadow DOM                       | Web Components 标准，将组件的 DOM 和 CSS 封装在 Shadow Root 内，与宿主页面隔离                                   |
| 自定义渲染器（Custom Component） | 使用者注册的渲染函数，用于在消息流中渲染特定类型的富组件（如工单卡片）                                           |
| 函数式嵌入                       | 通过 JavaScript 函数（如 `mountAiChat(target, config)`）将组件挂载到 DOM，无需框架环境                           |
| 事件链（Event Chain）            | 用户在聊天过程中的完整行为事件序列，用于埋点分析                                                                 |
| Teek 主题                        | `vitepress-theme-teek`，SmallAliceWeb 文档站使用的 VitePress 主题，通过 `@ruan-cat/vitepress-preset-config` 引入 |
| 主题色传递链路                   | Teek 主题色 → VitePress CSS 变量 → ai-vitepress-plugins 桥接 → ai-vue 消费的完整颜色传递路径                     |
| CSS 变量桥接                     | 在中间层将上游 CSS 变量（如 `--vp-c-brand-1`）映射为下游 CSS 变量（如 `--ai-chat-primary-color`）的机制          |

---

## 七、Teek 主题色传递链路需求 [新增]

### 7.1 背景与现状

SmallAliceWeb 文档站的颜色体系并非单一来源，而是由三层主题叠加构成：

```plain
┌─────────────────────────────────────────────────────────┐
│  第1层：VitePress 默认主题变量                            │
│  --vp-c-brand-1/2/3/soft, --vp-c-bg, --vp-c-text-1 等   │
│  定义在 docs/.vitepress/theme/style.css                  │
│  当前映射：--vp-c-brand-1 → --vp-c-indigo-1              │
└──────────────────────────┬──────────────────────────────┘
                           │ extends
┌──────────────────────────▼──────────────────────────────┐
│  第2层：Teek 主题变量（vitepress-theme-teek）             │
│  --tk-theme-color, --tk-color-primary,                   │
│  --tk-el-color-primary-light-3/5/7/8/9,                  │
│  --tk-bg-color, --tk-text-color 等                       │
│  Teek 的 --tk-theme-color 直接引用 --vp-c-brand-1        │
│  支持 html[theme-color=tk-primary] 动态主题色切换         │
└──────────────────────────┬──────────────────────────────┘
                           │ 桥接 style.css
┌──────────────────────────▼──────────────────────────────┐
│  第3层：ai-vitepress-plugins 桥接层                      │
│  --ai-chat-primary-color → var(--vp-c-brand-1)           │
│  --ai-chat-surface-color → var(--vp-c-bg-elv)            │
│  --ai-chat-text-color → var(--vp-c-text-1)               │
│  ... 共 15 个桥接变量                                    │
└──────────────────────────┬──────────────────────────────┘
                           │ 消费
┌──────────────────────────▼──────────────────────────────┐
│  第4层：ai-vue 组件库                                    │
│  .ai-chat { --ai-chat-primary: var(--ai-chat-primary-    │
│  color, #3b82f6); }                                      │
│  当前使用固定深色面板作为 fallback                        │
└─────────────────────────────────────────────────────────┘
```

**当前问题**：

1. **桥接不完整**：ai-vitepress-plugins 的 `style.css` 只桥接了 VitePress 的 `--vp-c-*` 变量，**没有桥接 Teek 主题的 `--tk-*` 变量**。当 Teek 主题切换主题色（通过 `html[theme-color=tk-primary]` 属性选择器）时，AI 聊天组件无法感知。

2. **fallback 不适配**：ai-vue 的 `index.scss` 使用固定深色面板色（`#111318`、`#171a21` 等）作为 fallback，这些 fallback 在浅色主题下会显示为不协调的深色块。

3. **缺少运行时感知**：当前桥接是纯 CSS 变量映射，无法在 JavaScript 运行时获取当前主题色值，限制了品牌色派生（P1 阶段的 `useBrandTheme`）无法获取真实的 Teek 主题色作为种子。

4. **暗色模式不同步**：Teek 主题有自己的暗色模式切换逻辑（`.dark` class + `--tk-*` 变量覆盖），ai-vue 组件没有同步机制。

### 7.2 目标：完整的主题色传递链路

#### 目标 7.2.1：Teek 主题色变量桥接

- [ ] ai-vitepress-plugins 的 `style.css` 新增 Teek 主题变量桥接，将 `--tk-theme-color`、`--tk-color-primary` 等变量映射为 `--ai-chat-*` 变量
- [ ] 桥接优先级为：Teek 变量 > VitePress 变量 > 固定 fallback（Teek 优先，因为 Teek 是最终用户可见的主题层）
- [ ] 支持 Teek 的 `html[theme-color=tk-primary]` 动态主题色切换，确保 AI 聊天组件跟随切换

#### 目标 7.2.2：运行时主题色获取

- [ ] 提供 `useThemeColor()` composable，在运行时通过 `getComputedStyle(document.documentElement)` 获取当前生效的 `--tk-theme-color` / `--vp-c-brand-1` 值
- [ ] 该 composable 返回响应式 ref，当 Teek 主题色切换时自动更新
- [ ] 为 P1 阶段的 `useBrandTheme` 提供真实的主题色种子，而非要求使用者手动传入 `primaryBrandColor`

#### 目标 7.2.3：暗色模式同步

- [ ] ai-vue 组件库感知 Teek/VitePress 的暗色模式切换（监听 `html.dark` class 变化）
- [ ] 暗色模式下自动切换 ai-vue 的 surface/text/border 色阶，无需使用者手动配置
- [ ] 与 P2 阶段的 Shadow DOM 方案兼容（Shadow DOM 内部也能感知外部暗色模式）

#### 目标 7.2.4：品牌色派生与主题色获取的协同

- [ ] `useBrandTheme` composable 优先使用运行时获取的 Teek 主题色作为种子
- [ ] 当运行时获取失败（如 SSR 环境或非 VitePress 嵌入场景）时，降级为使用者传入的 `primaryBrandColor`
- [ ] 当两者都不可用时，降级为默认品牌色 `#3b82f6`

### 7.3 验收标准

- [ ] 在 Teek 主题色为默认 indigo 时，AI 聊天组件的主色调与文档站导航栏主色调视觉一致
- [ ] 通过 Teek 的主题色切换功能（如切换为 green/purple）后，AI 聊天组件的主色调跟随切换
- [ ] 在暗色模式下，AI 聊天组件的背景色、文字色、边框色自动适配为暗色色阶
- [ ] `useThemeColor()` 返回的颜色值与 `getComputedStyle` 获取的值一致
- [ ] 在非 VitePress 环境（如纯 Vue 应用）中，`useBrandTheme` 降级为手动传入的品牌色，不报错

### 7.4 约束

1. **不修改 Teek 主题源码**：所有桥接在 ai-vitepress-plugins 和 ai-vue 层完成，不修改 `vitepress-theme-teek` 包
2. **CSS 变量优先**：颜色传递以 CSS 自定义属性为主要机制，JavaScript 运行时获取仅用于派生场景
3. **SSR 安全**：`useThemeColor()` 在服务端渲染时返回默认值，不访问 `document`
4. **性能无感知**：主题色监听使用 `MutationObserver` 而非轮询，对页面性能无影响

---

## 八、DataComponent 结构化卡片渲染需求 [新增]

### 8.1 背景与现状

当前 `AiChat.vue` 的消息渲染能力非常基础：

- **纯文本 + Markdown**：助手消息通过 `markstream-vue` 渲染为流式 Markdown
- **来源链接**：助手消息底部展示 RAG 检索来源的链接列表
- **无结构化卡片**：当 Agent 返回搜索结果、工单信息、订单状态等结构化数据时，只能序列化为纯文本嵌入 Markdown，无法以卡片形式展示

inkeep/agents 的 `DataComponent` 模式展示了更优的方案：Agent 返回结构化数据（JSON），前端根据数据类型注册对应的渲染器，渲染为带标题、摘要、操作按钮的富卡片。例如搜索结果卡片包含标题、摘要、来源链接、相关度评分；工单卡片包含工单号、状态、优先级、处理人。

### 8.2 现有依赖的富 UI 能力盘点 [关键]

**结论：不需要重复造轮子。** SmallAliceWeb 当前依赖的 `vue-element-plus-x` 已经提供了完整的富聊天 UI 组件库，足以支撑 DataComponent 结构化卡片渲染。

#### 8.2.1 vue-element-plus-x 已提供的组件

| 组件              | 用途                           | 可用于 DataComponent 的场景                       |
| ----------------- | ------------------------------ | ------------------------------------------------- |
| **BubbleList**    | 消息列表（虚拟滚动、自动跟随） | 消息容器，通过 `itemType` + `#item` slot 分发渲染 |
| **Bubble**        | 单条消息气泡                   | 卡片的外层容器（头像、placement、variant）        |
| **FilesCard**     | 文件卡片（16 种文件类型）      | 文档搜索结果卡片、附件展示                        |
| **Prompts**       | 提示卡片（分组、图标、描述）   | 示例问题、推荐操作                                |
| **Welcome**       | 欢迎卡片                       | 空状态引导                                        |
| **ThoughtChain**  | 思维链展示                     | Agent 推理过程展示                                |
| **Conversations** | 会话列表                       | 多轮对话历史                                      |
| **XSender**       | 输入框（提及、触发器）         | 增强输入体验                                      |

#### 8.2.2 BubbleList 的自定义渲染机制

`BubbleList` 提供了完整的 slot 系统用于自定义渲染：

```typescript
// BubbleList 的 slot 定义（来自类型声明）
slots: {
  // 完全自定义整条消息（最高优先级，可按 itemType 分发）
  item?(ctx: { item: T; index: number; itemType?: string }): any;
  // 自定义消息内容区域（保留气泡外壳）
  content?(ctx: { item: T }): any;
  // 自定义消息头部
  header?(ctx: { item: T }): any;
  // 自定义消息尾部（来源链接放这里）
  footer?(ctx: { item: T }): any;
  // 自定义头像
  avatar?(ctx: {}): any;
  // 自定义加载状态
  loading?(ctx: { item: T }): any;
}
```

**`itemType` 机制**是关键：每条消息可以携带 `itemType` 字段（如 `"search-result"`、`"ticket-card"`），`BubbleList` 的 `#item` slot 可以根据 `itemType` 值分发到不同的渲染组件。这正是 inkeep/agents `ComponentsConfig` 的 Vue 版等价物。

#### 8.2.3 markstream-vue 的自定义渲染能力

`markstream-vue` 提供了 `placeholder` slot，可以在 Markdown 渲染过程中对特定节点（如 `image`、`html_inline`）进行自定义渲染。这适用于在 Markdown 流中插入富组件，但不如 `BubbleList` 的 `itemType` 机制直观。

### 8.3 目标

#### 目标 8.3.1：结构化消息类型定义

- [ ] 扩展 `AiChatMessage` 类型，新增 `itemType` 字段标识消息的数据类型
- [ ] 定义内置的结构化消息类型：`search-result`（搜索结果卡片）、`source-list`（来源列表卡片）
- [ ] 支持使用者自定义 `itemType` 和对应的渲染组件

#### 目标 8.3.2：基于 BubbleList slot 的卡片渲染

- [ ] 利用 `BubbleList` 的 `#item` slot + `itemType` 机制实现按类型分发渲染
- [ ] 内置 `search-result` 卡片：标题、摘要、来源链接、相关度评分
- [ ] 内置 `source-list` 卡片：来源列表（替代当前的纯链接 nav）
- [ ] 未注册的 `itemType` 回退到默认的 Markdown 渲染

#### 目标 8.3.3：自定义渲染器注册

- [ ] 提供 `customRenderers` prop，允许使用者注册 `itemType → Vue组件` 的映射
- [ ] 注册的渲染器接收 `item` 数据作为 props，可完全自定义卡片样式
- [ ] 与 inkeep/agents 的 `ComponentsConfig` 设计理念一致，但利用 Vue 的 slot 机制实现

#### 目标 8.3.4：与现有 RAG 管线兼容

- [ ] 后端 `/v1/chat` 返回的流式响应中可携带结构化数据标记
- [ ] 前端解析流式响应时，识别结构化数据并构造对应的 `itemType` 消息
- [ ] 现有的纯文本 + 来源链接模式作为默认行为不受影响

### 8.4 验收标准

- [ ] 当消息 `itemType` 为 `search-result` 时，渲染为带标题、摘要、来源链接的卡片
- [ ] 当消息 `itemType` 为 `source-list` 时，渲染为来源列表卡片（替代当前 nav 链接）
- [ ] 当消息无 `itemType` 或 `itemType` 未注册时，回退到默认 Markdown 渲染
- [ ] 使用者可通过 `customRenderers` prop 注册自定义 `itemType` 渲染器
- [ ] 卡片样式跟随主题色（与 P1 品牌化主题系统协同）
- [ ] 现有 `useKnowledgeChat` 的流式 Markdown 渲染不受影响

### 8.5 约束

1. **不重复造轮子**：卡片渲染基于 `vue-element-plus-x` 的 `BubbleList` slot 机制，不自行实现消息列表和气泡组件
2. **不新增重型依赖**：利用现有 `vue-element-plus-x` 和 `element-plus` 组件，不引入新的 UI 库
3. **向后兼容**：现有 `AiChatMessage` 类型的 `content` 和 `sources` 字段保持不变，`itemType` 为可选字段
4. **流式安全**：结构化卡片渲染需要等待完整数据到达后再渲染，不能在流式过程中部分渲染卡片

---

## 九、vue-element-plus-x 升级评估与组件复用策略 [新增]

### 9.1 调研问题与方法

用户提出三个问题：

1. vue-element-plus-x 要不要升级到最新版？
2. 最新版是否提供了更多组件可供合理使用？
3. ai-vue 接下来如何增加组件，在学习 @inkeep/agents-ui、不重复造轮子的前提下复用 vue-element-plus-x？

调研方法：不依赖 README 口径，直接对 npm 产物做 dist 实测 —— `npm pack vue-element-plus-x@2.0.3` 解包读取 `types/` 类型声明，与仓库实际安装的 1.3.98 类型逐项对照。

### 9.2 版本现状

| 对比项        | 当前（仓库安装）                          | 最新（npm latest）                                                   |
| :------------ | :---------------------------------------- | :------------------------------------------------------------------- |
| 版本          | 1.3.98（package.json 声明 ^1.3.2）        | 2.0.3（2026-05-15 发布）                                             |
| 组件目录数    | 17                                        | 11                                                                   |
| Markdown 渲染 | 内置 XMarkdown 系列（捆绑 shiki，体积大） | 已移除，交给业务侧（本仓库用 markstream-vue）                        |
| 输入组件      | Sender / EditorSender / MentionSender     | 统一为 XSender（基于独立 x-sender 包）                               |
| 主题系统      | 无官方入口                                | ConfigProvider（namespace/theme/themeOverrides/applyTo）+ useTheme() |
| BubbleList    | 无虚拟滚动、无 #item slot                 | 虚拟滚动（virtua）+ #item slot + itemType 解析器 + loadMore 分页     |

peer 依赖对照：v2 要求 `element-plus ^2.9.7`、`vue ^3.5.17`；仓库为 `element-plus ^2.11.8`、`vue ^3.5.28`，满足，无版本冲突。

### 9.3 组件盘点与可用性评估（v2.0.3 dist 实测）

| 组件                         | v2 状态                  | 用途                                       | 对 ai-vue 的价值                                    |
| :--------------------------- | :----------------------- | :----------------------------------------- | :-------------------------------------------------- |
| Bubble / BubbleList          | 保留（API 有破坏性演进） | 气泡与消息列表                             | 核心承载，#item slot 是 P3.5 的基础                 |
| XSender                      | 新增（替代 Sender）      | 聊天输入框                                 | 必须迁移，AiChat.vue 当前 import 的 Sender 已被移除 |
| ConfigProvider               | 增强                     | 主题注入（namespace/theme/themeOverrides） | P1 品牌化的官方通道，替代直接改内部 CSS 变量        |
| useTheme / useNamespace      | 新增 hooks               | 读取主题态、生成 CSS 变量                  | 供 useBrandTheme 桥接 Teek 主题色                   |
| Prompts / Welcome            | 保留                     | 示例问题 / 空状态引导                      | 对应 agents-ui 的 exampleQuestions / introMessage   |
| ThoughtChain / Thinking      | 保留                     | 推理过程与思考态                           | 对应 StatusComponent 的过程可见性                   |
| Conversations                | 保留                     | 会话历史列表                               | 多轮对话管理（后续阶段）                            |
| FilesCard / Attachments      | 保留                     | 文件卡片 / 附件                            | 知识库文档引用展示                                  |
| Typewriter / XMarkdown\*     | 移除                     | 打字机 / Markdown 渲染                     | 无损失：markstream-vue 已承担                       |
| EditorSender / MentionSender | 移除（能力并入 XSender） | 富文本 / @提及输入                         | 暂无诉求                                            |

### 9.4 升级决策：升级到 2.0.3，并作为 P0 前置任务

**决策**：升级，且必须先于 P3.5 实施（落地方案见 plan 第十三章）。

**理由**：

1. P3.5 DataComponent 依赖的 `#item` slot 与 `itemType` 解析器只在 v2 存在 —— 1.3.98 的 BubbleList 类型实测无此 API，不升级则 P3.5 方案是空中楼阁。
2. v2 ConfigProvider + useTheme 是官方主题注入通道，P1/P1.5 应搭车而不是逆向 CSS 变量。
3. v2 移除的 Typewriter/XMarkdown 与仓库现状零冲突（markdown 归 markstream-vue）。
4. shiki 捆绑移除后安装体积显著下降。

**破坏点与对策**：

- `Sender` 移除 → AiChat.vue 改用 XSender（无 v-model，取值走 `getModelValue()`；`auto-size`/`submit-btn-disabled` 无对应，见 plan 13.2 对照表）。
- BubbleList `complete`/`triggerIndices`/TypewriterInstance 移除 → AiChat 未使用，无影响。
- Bubble `typing/isMarkdown/isFog` 移除 → AiChat 的 Bubble 仅做空状态壳，无影响。

验收标准：

- [ ] 升级后构建与类型检查通过
- [ ] Sender→XSender 迁移后发送/停止/禁用行为不变
- [ ] 空状态 Bubble 与消息列表渲染回归通过（vitest 现有用例全绿）
- [ ] agent-browser 视觉验证（plan 第十二章流程）确认消息列表与输入框外观正常

### 9.5 组件复用策略：三层分工，不重复造轮子

**分层原则**：

1. **原子组件层 = vue-element-plus-x**：气泡、列表、输入、提示、欢迎、思维链等原子能力直接复用，禁止在 ai-vue 内重写同类组件。
2. **ai-vue 包装层 = 项目增值所在**：品牌化主题（useBrandTheme + ConfigProvider 桥接）、Teek 主题色传递（useThemeColor）、类型系统（AiChatMessage/itemType）、事件链（useChatEvents）、Shadow DOM 隔离（AiShadowRoot）、组件形态（Sidebar/Modal 容器）。学习 @inkeep/agents-ui 的产出全部落在这一层。
3. **自研边界**：仅当 vepx 无对应组件时自研（如 AiShadowRoot、色板派生 color-utils）。

**映射表**：

| @inkeep/agents-ui 能力      | ai-vue 落地                     | 复用的 vepx 组件              | ai-vue 自研部分            |
| :-------------------------- | :------------------------------ | :---------------------------- | :------------------------- |
| EmbeddedChat                | AiChat                          | BubbleList / Bubble / XSender | 主题系统、事件链、消息类型 |
| ComponentsConfig 自定义渲染 | customRenderers + itemType 分发 | BubbleList `#item` slot       | 渲染器注册类型与回退逻辑   |
| exampleQuestions            | 示例问题区                      | Prompts                       | 事件上报                   |
| introMessage / 空状态       | 引导消息                        | Welcome / Bubble              | 文案配置                   |
| InkeepFeedback              | 反馈按钮                        | Bubble `#footer` slot         | 反馈事件与上报协议         |
| StatusComponent             | 生成过程可见性                  | Thinking / ThoughtChain       | 状态事件映射               |
| SidebarChat / ModalChat     | AiSidebarChat / AiModalChat     | 复用 AiChat 内核              | 容器交互与定位             |
| 会话切换                    | useConversation                 | Conversations                 | 历史存储与压缩             |

### 9.6 约束

1. 版本锁定策略：package.json 声明 `^2.0.3`，锁文件已入库（.gitignore 已移除 pnpm-lock.yaml 忽略规则），升级必须走 lock 文件 diff review。
2. 不 fork vepx 源码：定制一律通过 wrapper 组件、slot、ConfigProvider themeOverrides、CSS 变量前缀实现。
3. 升级与功能解耦：升级本身是独立 P0 任务，回归通过后才允许叠加新功能开发。
4. tree-shaking：ai-vue 构建保持按需引入 vepx 组件，避免全量注册。

---

## 十、后端路线修订的前端配套需求 [新增]

### 10.1 背景与职责边界

[重调研报告](../2026-09-05-inkeep-agents-local-research-report.md) 1.3 节记录了用户的四条路线约束（不做多轮、不接 MCP、模型切换 P0、单轮回流 P2）。修订后路线中落在**前端侧**的配套义务归入本章，范围覆盖 `@ruan-cat-drill-doc/ai-vue` 与 `ai-vitepress-plugins` 两包：

- **前端职责**：页面上下文采集与透传、provider 无关的契约稳定性、客户端响应元数据事件、反馈与回流关联。
- **前端不做什么**：不做历史注入语义与多轮 UI、不接 MCP、不实现任何工具执行能力；后端事项（模型注册表泛化、qa_records、评估钩子）在 `ai-rag-api` 侧另行立项。

### 10.2 需求一：页面上下文采集与透传

- [ ] `ai-vitepress-plugins` 的 `useKnowledgeChat` 新增页面上下文采集（基于 VitePress `useData`/`useRoute`），随 `/v1/chat` 请求体发送 `pageContext` 字段（至少含 `pagePath`、`title`）
- [ ] 采集契约与后端 zod schema 对齐；字段缺失或为空时后端跳过注入（对齐 inkeep `requiredToFetch` 的降级语义），前端不因采集失败阻断提问
- [ ] SSR 安全：服务端渲染期间返回 `undefined`，不访问 `document`
- [ ] **ai-vue 零改动判定**：聊天传输由 `useKnowledgeChat`（`@ai-sdk/vue` useChat）驱动，`AiChat.vue` 仅 emit 消息，故 `AiChatProps` 不新增上下文 prop，避免职责越界

验收标准：

- [ ] 在文档页 A 提问「这个怎么配」，后端收到的请求体携带页面 A 的 pageContext
- [ ] 非文档环境（纯 Vue）下采集函数返回空，请求照常发送

### 10.3 需求二：provider 无关的前端契约稳定性

- [ ] 后端切换模型 provider（修订路线 P0）时，前端渲染、来源数据帧、事件行为完全不变
- [ ] 前端消费面清单化：梳理 AiChat 依赖的全部流式帧与事件类型，vitest 断言不依赖任何上游 SSE 事件名（与 openspec chat-api Requirement 8 的「下游流格式保持稳定」对齐）
- [ ] 视觉验证矩阵（plan 第十二章）新增「切换 provider 后回归」场景

验收标准：

- [ ] mock 双 provider 场景下 vitest 全绿；agent-browser 视觉验证在 provider 切换前后截图判读一致

### 10.4 需求三：客户端响应元数据与 TTFT 事件

- [ ] `useKnowledgeChat` 记录首 chunk 到达时间（客户端感知 TTFT），作为后端分 provider TTFT 记录的用户侧交叉验证
- [ ] 并入 plan 4.6 事件系统（`useChatEvents`）：新增 `response-metadata` 事件（携带 `provider`/`model`/`ttftMs`，字段以后端 data-stream 元数据帧实际提供为准），现有消费方不受影响

验收标准：

- [ ] `response-metadata` 事件包含 `ttftMs` 且在现有用例中不破坏既有 emit 契约

### 10.5 需求四：conversationId 追溯语义与反馈关联

- [ ] `conversationId` 语义固化为「单轮问答的追溯分组标识」，仅用于日志、回流评估与外部系统回链，不携带历史注入语义
- [ ] 默认值保持向后兼容（现默认 `"knowledge-chat"`）；宿主可传入页面级会话 ID（如 `pagePath + 会话种子`）提升回流粒度
- [ ] plan 4.4 反馈组件（AiChatFeedback）的 emit 载荷增加 `conversationId` 与 `messageId`，供后端 `qa_records` 关联用户反馈信号

验收标准：

- [ ] 反馈事件载荷可唯一定位一条问答记录；现有 API 无破坏性变更

### 10.6 约束

1. 不新增运行时依赖；上下文采集使用 VitePress/Vue 已有 API。
2. 所有新增字段均为可选，向后兼容现有 `AiChatProps`/`AiChatEmits` 与 `useKnowledgeChat` 签名。
3. SSR 安全：任何新增采集不得在服务端访问浏览器 API。
4. 不引入多轮会话语义、不接 MCP（用户约束，见 10.1）。

---

## 十一、ContextConfig 动态上下文系统改造需求 [新增]

### 11.1 背景与借鉴对象

重调研报告第三章 3.3 节确认了 inkeep `agents-core` 的动态上下文三件套（证据见探索笔记 A）：

1. **ContextConfig**（`agents-core/src/context/ContextConfig.ts:396-447`）：`fetchDefinition` 声明式拉取配置，携带 `timeout` 与 `requiredToFetch`（必需变量无法解析则**跳过该次拉取**，非必需上下文失败不阻断主流程）。
2. **类型化上下文变量**：每个上下文变量绑定独立 zod schema，类型安全直达 prompt。
3. **TemplateEngine**（`agents-core/src/context/TemplateEngine.ts:86-94`）：`{{variable.path}}` 变量渲染进 prompt。

我们的现状：system prompt 硬编码在 `ai-rag-api/server/contracts/chat.ts:129`；无任何动态上下文；客户端来源（pageContext）的采集与透传已在 spec 10.2 / plan 14.2（FC-1）设计完毕。**本章定义动态上下文的完整改造需求，以后端（ai-rag-api）为主**，定位裁剪遵循 1.3 约束：单轮、无 MCP、web RAG——采用 inkeep 的机制形状，不引入其平台。

### 11.2 目标形态：三层结构

| 层  | 名称                            | v1 内容                                                                                                                                       | 来源拍板  |
| :-- | :------------------------------ | :-------------------------------------------------------------------------------------------------------------------------------------------- | :-------- |
| 1   | 上下文来源（ContextSource）     | `client`（pageContext，由 FC-1 透传）+ `static`（站点名/版本常量）；服务端 `fetchDefinition` 执行器**只定义类型接口不实现**（Q1 拍板，YAGNI） | plan 15.8 |
| 2   | 类型化上下文容器（ChatContext） | 全字段 zod 校验；`PageContext` schema 放 `@ruan-cat-drill-doc/ai-rag-core` 两端共用（D3 拍板）                                                | plan 15.3 |
| 3   | 函数式 prompt 模板              | `buildSystemPrompt(ctx, sources)` 五段式：角色设定 / 检索引导 / 引用格式 / 页面上下文注入段（条件渲染）/ 拒答策略（Q2/D2a 拍板）              | plan 15.5 |

**降级语义（borrow 自 requiredToFetch）**：任一上下文来源缺失、非法或失败时，跳过该来源并回退基础模板，**问答永不因上下文问题而失败**。v1 所有来源皆可选，无「必需上下文」场景，`requiredToFetch` 等价物仅在类型注释中预留。

### 11.3 与既有章节及规格的关系

- 客户端采集在前端侧（spec 10.2 / plan 14.2），本章不重复；本章只定义后端的容器、模板与接线。
- prompt 段落化后，`[来源N]` 引用格式与「根据现有资料无法回答」的对外行为**保持不变**（openspec chat-api Requirement 1 行为不变、组装方式变）。
- **spec 纪律前置**：chat-api spec Requirement 1 的组装描述需先以 openspec change 修订合入，才允许动 `contracts/chat.ts` 代码。

### 11.4 验收标准

- [ ] 在文档页 A 提问「这个怎么配」，回答针对页面 A（FC-1 采集 + 注入段联动）
- [ ] pageContext 缺失或非法时，使用基础模板正常回答，无报错无重试
- [ ] 修改提示词只需改模板模块，`contracts/chat.ts` 的请求处理逻辑无需改动
- [ ] vitest 覆盖：模板渲染两态快照（有/无 pageContext）、来源归一化三态（合法/非法/缺失）、既有 chat 用例全绿
- [ ] 来源数据帧契约与流式行为零破坏

### 11.5 约束

1. 不做多轮会话语义、不接 MCP（沿用 1.3 用户约束）；上下文为单轮、无状态、全部可选。
2. 不新增运行时依赖（zod 已有；不引入 JMESPath——函数式模板不需要）。
3. 所有新增请求字段可选，向后兼容现有 `/v1/chat` 契约。
4. 四项关键决策（Q1 范围 / Q2 载体 / Q3 字段最小集 / Q4 独立注入段）按推荐默认拍板，备选方案与推翻成本记录于 plan 15.8 蓝军拷问记录，用户可随时推翻。
