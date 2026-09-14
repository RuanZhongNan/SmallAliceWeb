# Plan：ai-vue 子包学习 @inkeep/agents-ui 的落地方案

> 文档类型：plan（实施计划）
> 创建日期：2026-09-05
> 所属报告：`reports/2026-9-5-learn-inkeep-agents-repo/`
> 目标子包：`@ruan-cat-drill-doc/ai-vue`
> 参考对象：`@inkeep/agents-ui` v0.17.8
> 配套 spec：`./spec.md`

---

## 一、实施总览

### 1.1 分阶段路线图

本方案由 P0 前置升级与 4 个实施阶段组成，每个阶段产出可独立验证的增量能力：

| 阶段 | 主题                      | 预计工期 | 核心产出                                                            |
| :--- | :------------------------ | :------- | :------------------------------------------------------------------ |
| P0   | vue-element-plus-x 升级   | 0.5 天   | `Sender`→`XSender` 迁移 + `ConfigProvider` 主题通道（详见第十三章） |
| P1   | 品牌化主题系统            | 3 天     | `useBrandTheme` composable + 色板自动派生                           |
| P2   | Shadow DOM 样式隔离       | 2 天     | `AiShadowRoot` 组件 + variant 配置                                  |
| P3   | 富聊天体验增强            | 4 天     | 自定义渲染器 + 消息操作 + 反馈 + 示例问题                           |
| P4   | 组件形态扩展 + 函数式嵌入 | 3 天     | SidebarChat + ModalChat + `mountAiChat` 函数                        |

### 1.2 文件变更预览

```plain
packages/ai-vue/src/
├── index.ts                          # 修改：导出新组件和 composable
├── composables/
│   ├── useMockAiChat.ts              # 保留
│   ├── useBrandTheme.ts              # 新增：品牌色派生主题
│   └── useChatEvents.ts              # 新增：事件链聚合
├── components/
│   ├── ai-chat/
│   │   ├── AiChat.vue                # 修改：接入主题系统 + 富聊天
│   │   ├── AiChatFloatingButton.vue  # 修改：接入主题系统
│   │   ├── types.ts                  # 修改：扩展 props/emits
│   │   └── parts/                    # 新增：子部件
│   │       ├── AiChatMessageActions.vue
│   │       ├── AiChatFeedback.vue
│   │       ├── AiChatExampleQuestions.vue
│   │       └── AiChatCustomRenderer.vue
│   ├── ai-sidebar-chat/              # 新增
│   │   ├── AiSidebarChat.vue
│   │   ├── index.ts
│   │   └── types.ts
│   ├── ai-modal-chat/                # 新增
│   │   ├── AiModalChat.vue
│   │   ├── index.ts
│   │   └── types.ts
│   └── ai-shadow-root/               # 新增
│       ├── AiShadowRoot.vue
│       └── index.ts
├── theme/
│   ├── color-utils.ts                # 新增：HSL 色板派生
│   ├── default-theme.ts              # 新增：默认主题令牌
│   └── types.ts                      # 新增：主题类型定义
└── styles/
    └── index.scss                    # 修改：改用 CSS 变量驱动

# 注意：不新增独立子包。函数式嵌入（mountAiChat）直接在 ai-vue 包内导出，
# 避免增加 monorepo 的构建配置、版本同步和发布流程复杂度。
# 宿主环境（VitePress）已有 Vue 运行时，无需独立打包 Vue。
```

---

## 二、P1：品牌化主题系统

### 2.1 设计思路

agents-ui 的品牌化核心是：使用者只需提供一个 `primaryBrandColor`，系统自动派生出完整的色板（11 个色阶）和主题令牌。ai-vue 将借鉴这一设计，但用纯 TypeScript 函数实现颜色派生，不引入 `colorjs.io`（体积 30KB+）。

颜色派生算法采用 HSL 空间：将品牌色转换为 HSL，然后通过调整明度（Lightness）生成不同色阶。这种方法简单、可预测、零依赖。

### 2.2 类型定义

```typescript
// packages/ai-vue/src/theme/types.ts

/** 使用者提供的品牌色配置 */
export interface BrandThemeConfig {
	/** 主品牌色，任意 CSS 颜色值（hex/rgb/hsl） */
	primaryBrandColor: string;
	/** 组织展示名称，显示在头部等位置 */
	organizationDisplayName?: string;
	/** 精细覆盖特定色阶 */
	customColorScheme?: Partial<ColorScheme>;
	/** 主题令牌覆盖 */
	theme?: Partial<IkpTheme>;
	/** CSS 变量前缀，默认 'ai-chat' */
	prefix?: string;
	/** 颜色模式 */
	colorMode?: "light" | "dark" | "system";
}

/** 从品牌色派生的色板 */
export interface ColorScheme {
	/** 最浅背景色 */
	lighter: string;
	/** 浅色背景 */
	light: string;
	/** 浅色微妙 */
	lightSubtle: string;
	/** 中等背景 */
	medium: string;
	/** 中等微妙 */
	mediumSubtle: string;
	/** 较强浅色 */
	strongerLight: string;
	/** 强调色（按钮、链接） */
	strong: string;
	/** 最强强调色（悬停态） */
	stronger: string;
	/** 主文字色 */
	textBold: string;
	/** 次要文字色 */
	textSubtle: string;
	/** 主色上的文字色 */
	textColorOnPrimary: string;
}

/** 主题令牌系统 */
export interface IkpTheme {
	colors: Record<string, string>;
	fontFamily: Record<string, string>;
	fontSize: Record<string, string>;
	zIndex: Record<string, string | number>;
}
```

### 2.3 颜色派生实现

```typescript
// packages/ai-vue/src/theme/color-utils.ts

import type { ColorScheme } from "./types";

/** 将 hex/rgb 颜色解析为 RGB 分量 */
function parseToRgb(color: string): { r: number; g: number; b: number } {
	const hexMatch = color.match(/^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i);
	if (hexMatch) {
		return {
			r: parseInt(hexMatch[1], 16),
			g: parseInt(hexMatch[2], 16),
			b: parseInt(hexMatch[3], 16),
		};
	}
	const rgbMatch = color.match(/rgba?\((\d+),\s*(\d+),\s*(\d+)/i);
	if (rgbMatch) {
		return {
			r: parseInt(rgbMatch[1]),
			g: parseInt(rgbMatch[2]),
			b: parseInt(rgbMatch[3]),
		};
	}
	// 兜底：返回默认蓝色
	return { r: 59, g: 130, b: 246 };
}

/** RGB 转 HSL */
function rgbToHsl(r: number, g: number, b: number): { h: number; s: number; l: number } {
	const rNorm = r / 255;
	const gNorm = g / 255;
	const bNorm = b / 255;
	const max = Math.max(rNorm, gNorm, bNorm);
	const min = Math.min(rNorm, gNorm, bNorm);
	const delta = max - min;
	let h = 0;
	if (delta !== 0) {
		if (max === rNorm) h = ((gNorm - bNorm) / delta) % 6;
		else if (max === gNorm) h = (bNorm - rNorm) / delta + 2;
		else h = (rNorm - gNorm) / delta + 4;
		h = Math.round(h * 60);
		if (h < 0) h += 360;
	}
	const l = (max + min) / 2;
	const s = delta === 0 ? 0 : delta / (1 - Math.abs(2 * l - 1));
	return { h, s: Math.round(s * 100), l: Math.round(l * 100) };
}

/** HSL 转 CSS 字符串 */
function hslToString(h: number, s: number, l: number, alpha = 1): string {
	return alpha < 1 ? `hsl(${h} ${s}% ${l}% / ${alpha})` : `hsl(${h} ${s}% ${l}%)`;
}

/**
 * 从单一品牌色派生完整色板。
 * 算法：将品牌色转为 HSL，通过调整明度生成 11 个色阶。
 * @param primaryBrandColor 品牌色（hex 或 rgb）
 * @returns 完整色板
 */
export function deriveColorScheme(primaryBrandColor: string): ColorScheme {
	const { r, g, b } = parseToRgb(primaryBrandColor);
	const { h, s, l } = rgbToHsl(r, g, b);

	// 根据品牌色的明度决定派生方向
	const isLight = l > 60;

	return {
		lighter: hslToString(h, Math.max(s - 5, 10), isLight ? 97 : 95),
		light: hslToString(h, s, isLight ? 92 : 88),
		lightSubtle: hslToString(h, s, isLight ? 88 : 82),
		medium: hslToString(h, s, isLight ? 80 : 72),
		mediumSubtle: hslToString(h, s, isLight ? 72 : 65),
		strongerLight: hslToString(h, s, isLight ? 65 : 58),
		strong: hslToString(h, s, l),
		stronger: hslToString(h, s, Math.max(l - 8, 20)),
		textBold: hslToString(h, Math.min(s + 10, 100), Math.min(l - 35, 15)),
		textSubtle: hslToString(h, s, isLight ? 45 : 60),
		textColorOnPrimary: isLight ? "#1a1a1a" : "#ffffff",
	};
}

/** 将色板转换为 CSS 变量对象 */
export function colorSchemeToCssVars(scheme: ColorScheme, prefix = "ai-chat"): Record<string, string> {
	const vars: Record<string, string> = {};
	const keyMap: Record<keyof ColorScheme, string> = {
		lighter: "surface-lighter",
		light: "surface-light",
		lightSubtle: "surface-light-subtle",
		medium: "surface-medium",
		mediumSubtle: "surface-medium-subtle",
		strongerLight: "surface-stronger-light",
		strong: "primary",
		stronger: "primary-hover",
		textBold: "text",
		textSubtle: "text-muted",
		textColorOnPrimary: "primary-contrast",
	};
	for (const [key, cssKey] of Object.entries(keyMap)) {
		vars[`--${prefix}-${cssKey}`] = scheme[key as keyof ColorScheme];
	}
	return vars;
}
```

### 2.4 useBrandTheme composable

````typescript
// packages/ai-vue/src/composables/useBrandTheme.ts

import { computed, type ComputedRef } from "vue";
import { colorSchemeToCssVars, deriveColorScheme } from "../theme/color-utils";
import { defaultTheme } from "../theme/default-theme";
import type { BrandThemeConfig, ColorScheme, IkpTheme } from "../theme/types";

export interface UseBrandThemeReturn {
	/** 完整色板 */
	colorScheme: ComputedRef<ColorScheme>;
	/** 主题令牌（合并默认值与用户覆盖） */
	theme: ComputedRef<IkpTheme>;
	/** CSS 变量对象，可直接绑定到 style 属性 */
	cssVars: ComputedRef<Record<string, string>>;
	/** CSS 变量前缀 */
	prefix: ComputedRef<string>;
}

/**
 * 从品牌色配置派生主题系统。
 *
 * @example
 * ```ts
 * const { cssVars } = useBrandTheme({
 *   primaryBrandColor: '#3784ff',
 *   organizationDisplayName: '钻头文档',
 * });
 * // 在 template 中：<div :style="cssVars">
 * ```
 */
export function useBrandTheme(config: BrandThemeConfig): UseBrandThemeReturn {
	const prefix = computed(() => config.prefix ?? "ai-chat");

	const colorScheme = computed<ColorScheme>(() => {
		const derived = deriveColorScheme(config.primaryBrandColor);
		// 用户精细覆盖优先
		return { ...derived, ...config.customColorScheme };
	});

	const theme = computed<IkpTheme>(() => ({
		...defaultTheme,
		...config.theme,
	}));

	const cssVars = computed<Record<string, string>>(() => {
		const schemeVars = colorSchemeToCssVars(colorScheme.value, prefix.value);
		const themeVars = flattenThemeToCssVars(theme.value, prefix.value);
		return { ...schemeVars, ...themeVars };
	});

	return { colorScheme, theme, cssVars, prefix };
}

/** 将主题令牌展平为 CSS 变量 */
function flattenThemeToCssVars(theme: IkpTheme, prefix: string): Record<string, string> {
	const vars: Record<string, string> = {};
	for (const [category, tokens] of Object.entries(theme)) {
		for (const [tokenName, value] of Object.entries(tokens)) {
			vars[`--${prefix}-${category}-${tokenName}`] = String(value);
		}
	}
	return vars;
}
````

### 2.5 在 AiChat.vue 中接入

```vue
<!-- packages/ai-vue/src/components/ai-chat/AiChat.vue（修改片段） -->
<script setup lang="ts">
import { useBrandTheme } from "../../composables/useBrandTheme";
import type { BrandThemeConfig } from "../../theme/types";

const props = withDefaults(
	defineProps<
		AiChatProps & {
			/** 品牌主题配置 */
			brandTheme?: BrandThemeConfig;
		}
	>(),
	{
		placeholder: "请输入消息",
		mode: "mock",
	},
);

// 默认品牌色
const brandConfig = computed(
	() =>
		props.brandTheme ?? {
			primaryBrandColor: "#3b82f6",
			prefix: "ai-chat",
		},
);
const { cssVars } = useBrandTheme(brandConfig.value);
</script>

<template>
	<section class="ai-chat" :style="cssVars">
		<!-- 原有内容 -->
	</section>
</template>
```

### 2.6 验收要点

- 输入 `primaryBrandColor: '#3784ff'` 后，按钮、链接、强调色均变为蓝色系
- 输入 `primaryBrandColor: '#22c55e'` 后，整体变为绿色系
- `customColorScheme: { strong: '#ff0000' }` 可单独覆盖强调色
- `prefix: 'my-brand'` 后，CSS 变量变为 `--my-brand-primary` 等

---

## 三、P2：Shadow DOM 样式隔离

### 3.1 设计思路

Shadow DOM 是 Web Components 标准，可将组件的 DOM 和 CSS 封装在独立的 Shadow Root 内，宿主页面的全局样式无法穿透 Shadow 边界。agents-ui 通过 `variant` 配置项控制是否启用 Shadow DOM（`no-shadow` / `container-with-shadow`）。

ai-vue 将实现一个 `AiShadowRoot` 包装组件，内部使用 Vue 3 的 `attachShadow` API。由于 Vue 的模板编译不直接支持 Shadow DOM，需要使用渲染函数或 `Teleport` 到 Shadow Root。

### 3.2 AiShadowRoot 组件

```vue
<!-- packages/ai-vue/src/components/ai-shadow-root/AiShadowRoot.vue -->
<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref, useSlots, watch } from "vue";

const props = withDefaults(
	defineProps<{
		/** Shadow DOM 模式 */
		mode?: "open" | "closed";
		/** 是否启用 Shadow DOM，false 时降级为普通 div */
		enabled?: boolean;
		/** 注入到 Shadow Root 的 CSS 文本 */
		styles?: string;
	}>(),
	{
		mode: "open",
		enabled: true,
	},
);

const slots = useSlots();
const hostRef = ref<HTMLElement | null>(null);
let shadowRoot: ShadowRoot | null = null;

onMounted(() => {
	if (!props.enabled || !hostRef.value) return;
	shadowRoot = hostRef.value.attachShadow({ mode: props.mode });
	// 注入样式
	if (props.styles) {
		const styleEl = document.createElement("style");
		styleEl.textContent = props.styles;
		shadowRoot.appendChild(styleEl);
	}
	// 渲染 slot 内容到 Shadow Root
	// Vue 3.5+ 支持 teleport to shadow root
	forceUpdate();
});

onBeforeUnmount(() => {
	shadowRoot = null;
});

// 强制更新以重新渲染 slot 内容
const updateKey = ref(0);
function forceUpdate() {
	updateKey.value++;
}
</script>

<template>
	<div ref="hostRef" class="ai-shadow-root">
		<template v-if="!enabled">
			<slot />
		</template>
	</div>
</template>
```

> **注意**：Vue 3 的模板语法不直接支持将 slot 内容渲染到 Shadow Root。实际实现需要使用渲染函数（`h()`）配合 `Teleport`，或使用 `vue-custom-element` 等库。上面的代码展示的是设计思路，完整实现见 P2 任务清单。

### 3.3 在 AiChat 中接入 Shadow DOM

```typescript
// AiChat.vue 新增 variant 配置
interface AiChatProps {
	// ... 原有 props
	/** 样式隔离模式 */
	variant?: "no-shadow" | "container-with-shadow";
}

// 当 variant === 'container-with-shadow' 时，使用 AiShadowRoot 包装
```

### 3.4 SSR 降级策略

VitePress 在构建时会执行 SSR，Shadow DOM 在 SSR 环境中不可用。降级策略：

```typescript
const isSSR = typeof window === "undefined";
const shouldUseShadow = computed(() => props.variant === "container-with-shadow" && !isSSR);
```

---

## 四、P3：富聊天体验增强

### 4.1 自定义消息渲染器

借鉴 agents-ui 的 `ComponentsConfig`，允许使用者按名称注册自定义渲染器。

```typescript
// packages/ai-vue/src/components/ai-chat/types.ts（扩展）

/** 自定义渲染器函数签名 */
export type CustomComponentRenderer = (
	props: Record<string, unknown> & { messageId: string },
	context: { renderMarkdown: (text: string) => string },
) => string | void;

/** 自定义组件注册表 */
export type CustomComponents = Record<string, CustomComponentRenderer>;

export interface AiChatProps {
	// ... 原有 props
	/** 自定义消息渲染器注册表 */
	customComponents?: CustomComponents;
	/** 示例问题列表 */
	exampleQuestions?: string[];
	/** 引导消息 */
	introMessage?: string;
	/** 消息操作菜单配置 */
	messageActions?: MessageAction[];
	/** 反馈配置 */
	feedbackOptions?: FeedbackOptions;
}

export interface MessageAction {
	label: string;
	icon?: string;
	handler: (message: AiChatMessage) => void;
}

export interface FeedbackOptions {
	enabled: boolean;
	onSubmit?: (feedback: { type: "positive" | "negative"; messageId: string; details?: string }) => void;
}
```

### 4.2 消息内容扩展

当前 `AiChatMessage.content` 是纯字符串。为支持富组件，需要扩展消息结构：

```typescript
export interface AiChatMessage {
	id: string;
	role: AiChatRole;
	content: string;
	sources?: AiChatSource[];
	/** 新增：富组件渲染指令 */
	component?: {
		/** 渲染器名称，对应 customComponents 的 key */
		name: string;
		/** 传递给渲染器的 props */
		props: Record<string, unknown>;
	};
}
```

### 4.3 自定义渲染器组件

```vue
<!-- packages/ai-vue/src/components/ai-chat/parts/AiChatCustomRenderer.vue -->
<script setup lang="ts">
import { computed, h } from "vue";
import type { CustomComponents } from "../types";

const props = defineProps<{
	componentName: string;
	componentProps: Record<string, unknown> & { messageId: string };
	customComponents?: CustomComponents;
	markdownText?: string;
}>();

const renderer = computed(() => props.customComponents?.[props.componentName]);

const renderedContent = computed(() => {
	if (!renderer.value) return null;
	return renderer.value(props.componentProps, {
		renderMarkdown: (text: string) => text, // 简化，实际接入 markstream-vue
	});
});
</script>

<template>
	<div v-if="renderedContent" class="ai-chat__custom-component" v-html="renderedContent" />
	<div v-else class="ai-chat__custom-component ai-chat__custom-component--missing">未知组件：{{ componentName }}</div>
</template>
```

### 4.4 反馈组件

```vue
<!-- packages/ai-vue/src/components/ai-chat/parts/AiChatFeedback.vue -->
<script setup lang="ts">
import { ref } from "vue";
import type { FeedbackOptions } from "../types";

const props = defineProps<{
	messageId: string;
	options?: FeedbackOptions;
}>();

const emit = defineEmits<{
	(e: "submit", feedback: { type: "positive" | "negative"; messageId: string; details?: string }): void;
}>();

const selected = ref<"positive" | "negative" | null>(null);
const showDetail = ref(false);
const detail = ref("");

function submit(type: "positive" | "negative") {
	selected.value = type;
	if (type === "negative") {
		showDetail.value = true;
	} else {
		emit("submit", { type, messageId: props.messageId });
	}
}

function submitDetail() {
	emit("submit", {
		type: "negative",
		messageId: props.messageId,
		details: detail.value,
	});
	showDetail.value = false;
}
</script>

<template>
	<div v-if="options?.enabled" class="ai-chat__feedback">
		<button
			v-if="!selected"
			type="button"
			class="ai-chat__feedback-btn"
			:aria-label="'有帮助'"
			@click="submit('positive')"
		>
			👍
		</button>
		<button
			v-if="!selected"
			type="button"
			class="ai-chat__feedback-btn"
			:aria-label="'无帮助'"
			@click="submit('negative')"
		>
			👎
		</button>
		<span v-else class="ai-chat__feedback-done">已反馈</span>
		<div v-if="showDetail" class="ai-chat__feedback-detail">
			<textarea v-model="detail" placeholder="请告诉我们哪里可以改进" />
			<button type="button" @click="submitDetail">提交</button>
		</div>
	</div>
</template>
```

### 4.5 示例问题组件

```vue
<!-- packages/ai-vue/src/components/ai-chat/parts/AiChatExampleQuestions.vue -->
<script setup lang="ts">
defineProps<{
	questions: string[];
}>();

const emit = defineEmits<{
	(e: "select", question: string): void;
}>();
</script>

<template>
	<div v-if="questions.length" class="ai-chat__examples">
		<p class="ai-chat__examples-title">试试这些问题：</p>
		<div class="ai-chat__examples-list">
			<button v-for="q in questions" :key="q" type="button" class="ai-chat__example-item" @click="emit('select', q)">
				{{ q }}
			</button>
		</div>
	</div>
</template>
```

### 4.6 事件系统

```typescript
// packages/ai-vue/src/composables/useChatEvents.ts

import type { AiChatMessage } from "../components/ai-chat/types";

/** 聊天事件类型 */
export type ChatEventType =
	| "user_message_submitted"
	| "assistant_answer_displayed"
	| "chat_clear_clicked"
	| "chat_share_clicked"
	| "feedback_submitted"
	| "message_action_clicked";

export interface ChatEvent {
	type: ChatEventType;
	conversationId?: string;
	messageId?: string;
	tags: string[];
	properties?: Record<string, unknown>;
}

export type ChatEventHandler = (event: ChatEvent) => void;

/**
 * 聚合聊天事件，统一通过 onChatEvent 回调输出。
 */
export function useChatEvents(onChatEvent?: ChatEventHandler) {
	function emit(event: ChatEvent) {
		onChatEvent?.(event);
	}

	function emitUserMessage(message: AiChatMessage) {
		emit({
			type: "user_message_submitted",
			messageId: message.id,
			tags: ["chat", "user"],
			properties: { contentLength: message.content.length },
		});
	}

	function emitAssistantDisplayed(message: AiChatMessage) {
		emit({
			type: "assistant_answer_displayed",
			messageId: message.id,
			tags: ["chat", "assistant"],
			properties: { sourceCount: message.sources?.length ?? 0 },
		});
	}

	function emitFeedback(type: "positive" | "negative", messageId: string, details?: string) {
		emit({
			type: "feedback_submitted",
			messageId,
			tags: ["chat", "feedback"],
			properties: { feedbackType: type, hasDetails: !!details },
		});
	}

	return { emit, emitUserMessage, emitAssistantDisplayed, emitFeedback };
}
```

---

## 四点五、P3.5：DataComponent 结构化卡片渲染 [新增]

> 本阶段是对 P3 富聊天体验增强的补充，专门解决结构化数据（搜索结果、工单等）的卡片渲染问题。
> 建议在 P3 之后实施，因为需要 P3 的 `customComponents` 机制作为基础。

### 4.5.1 设计原则：不重复造轮子

**核心结论**：SmallAliceWeb 当前依赖的 `vue-element-plus-x`（v2.0.3）已经提供了完整的富聊天 UI 组件库，包括 BubbleList（消息列表）、Bubble（消息气泡）、FilesCard（文件卡片）、Prompts（提示卡片）、Welcome（欢迎卡片）、ThoughtChain（思维链）等。**不需要自己实现消息列表和气泡组件，也不需要引入新的 UI 库。**

DataComponent 结构化卡片渲染的核心实现思路是：**利用 `BubbleList` 的 `#item` slot + `itemType` 机制**，按消息类型分发到不同的渲染组件。这与 inkeep/agents 的 `ComponentsConfig` 设计理念一致，但利用了 Vue 原生的 slot 机制，无需自行实现组件注册和查找逻辑。

### 4.5.2 类型定义扩展

```typescript
// packages/ai-vue/src/components/ai-chat/types.ts（扩展）

export type AiChatRole = "user" | "assistant";

/** 内置的结构化消息类型 */
export type BuiltinItemType =
	| "text" // 默认纯文本 + Markdown
	| "search-result" // 搜索结果卡片
	| "source-list"; // 来源列表卡片

/** 消息数据结构 */
export interface AiChatMessage {
	id: string;
	role: AiChatRole;
	content: string;
	sources?: AiChatSource[];

	/** 消息类型，用于 BubbleList 的 #item slot 分发渲染 */
	itemType?: BuiltinItemType | string;

	/** 结构化数据，传给自定义渲染器 */
	data?: Record<string, unknown>;
}

/** 自定义渲染器映射：itemType → Vue 组件 */
export type CustomRendererMap = Record<string, import("vue").Component>;

export interface AiChatProps {
	initialMessages?: AiChatMessage[];
	messages?: AiChatMessage[];
	isResponding?: boolean;
	errorMessage?: string;
	mode?: "mock" | "external";
	placeholder?: string;
	mockDelay?: number;

	/** 自定义渲染器：按 itemType 注册 Vue 组件 */
	customRenderers?: CustomRendererMap;
}
```

### 4.5.3 内置卡片组件

#### 搜索结果卡片

```vue
<!-- packages/ai-vue/src/components/ai-chat/cards/SearchResultCard.vue -->
<script setup lang="ts">
import { Card, Tag } from "element-plus";

export interface SearchResultData {
	title: string;
	snippet: string;
	sourceUrl: string;
	sourceLabel: string;
	score?: number;
	headingPath?: string[];
}

const props = defineProps<{ data: SearchResultData }>();
</script>

<template>
	<Card class="search-result-card" shadow="hover">
		<template #header>
			<div class="search-result-card__header">
				<span class="search-result-card__title">{{ data.title }}</span>
				<Tag v-if="data.score" size="small" type="info"> 相关度 {{ Math.round(data.score * 100) }}% </Tag>
			</div>
		</template>
		<p class="search-result-card__snippet">{{ data.snippet }}</p>
		<div v-if="data.headingPath?.length" class="search-result-card__path">
			<span v-for="(h, i) in data.headingPath" :key="i">
				{{ h }}<span v-if="i < data.headingPath.length - 1"> / </span>
			</span>
		</div>
		<a :href="data.sourceUrl" class="search-result-card__link" target="_blank">
			{{ data.sourceLabel }}
		</a>
	</Card>
</template>
```

#### 来源列表卡片

```vue
<!-- packages/ai-vue/src/components/ai-chat/cards/SourceListCard.vue -->
<script setup lang="ts">
import type { AiChatSource } from "../types";

defineProps<{ sources: AiChatSource[] }>();
</script>

<template>
	<nav class="source-list-card">
		<span class="source-list-card__label">参考来源：</span>
		<a
			v-for="(source, index) in sources"
			:key="source.id"
			:href="source.sourceHref"
			class="source-list-card__link"
			target="_blank"
			rel="noopener"
		>
			[{{ index + 1 }}] {{ source.label }}
		</a>
	</nav>
</template>
```

### 4.5.4 AiChat.vue 的 #item slot 分发逻辑

修改 `AiChat.vue`，利用 `BubbleList` 的 `#item` slot 实现 itemType 分发：

```vue
<!-- packages/ai-vue/src/components/ai-chat/AiChat.vue（修改部分） -->
<script setup lang="ts">
import MarkdownRender from "markstream-vue";
import { computed } from "vue";
import { Bubble, BubbleList, Sender } from "vue-element-plus-x";
import SearchResultCard from "./cards/SearchResultCard.vue";
import SourceListCard from "./cards/SourceListCard.vue";
import type { AiChatEmits, AiChatMessage, AiChatProps, CustomRendererMap } from "./types";

const props = withDefaults(defineProps<AiChatProps>(), {
	placeholder: "请输入消息",
	mode: "mock",
});

/** 内置渲染器映射 */
const builtinRenderers = {
	"search-result": SearchResultCard,
	"source-list": SourceListCard,
} as const;

/** 合并内置渲染器和自定义渲染器 */
const allRenderers = computed<CustomRendererMap>(() => ({
	...builtinRenderers,
	...props.customRenderers,
}));

/** 根据 itemType 解析渲染器组件 */
function resolveRenderer(item: AiChatMessage) {
	const type = item.itemType ?? "text";
	return allRenderers.value[type];
}
</script>

<template>
	<BubbleList :list="displayedMessages">
		<!-- #item slot：按 itemType 分发渲染 -->
		<template #item="{ item }">
			<Bubble :placement="item.placement" :loading="item.loading" variant="filled">
				<!-- 有 itemType 且注册了渲染器：渲染结构化卡片 -->
				<component v-if="resolveRenderer(item)" :is="resolveRenderer(item)" :data="item.data" :sources="item.sources" />
				<!-- 无 itemType 或未注册渲染器：回退到默认 Markdown 渲染 -->
				<MarkdownRender v-else :content="item.content" :is-done="!displayedResponding" />
				<!-- 来源链接（默认行为，保持向后兼容） -->
				<SourceListCard v-if="!item.itemType && item.sources?.length" :sources="item.sources" />
			</Bubble>
		</template>
	</BubbleList>
</template>
```

### 4.5.5 使用示例

```vue
<!-- 使用者在 VitePress 或 Vue 应用中 -->
<script setup lang="ts">
import { AiChat } from "@ruan-cat-drill-doc/ai-vue";
import type { AiChatMessage } from "@ruan-cat-drill-doc/ai-vue";

// 自定义工单卡片渲染器
const TicketCard = defineComponent({
	props: { data: Object },
	template: `
    <div class="ticket-card">
      <h4>工单 #{{ data.id }}</h4>
      <p>状态：{{ data.status }}</p>
      <p>优先级：{{ data.priority }}</p>
    </div>
  `,
});

const messages: AiChatMessage[] = [
	// 内置搜索结果卡片
	{
		id: "1",
		role: "assistant",
		itemType: "search-result",
		data: {
			title: "如何配置 VitePress",
			snippet: "VitePress 是基于 Vite 的静态站点生成器...",
			sourceUrl: "https://vitepress.dev/guide/",
			sourceLabel: "VitePress 官方文档",
			score: 0.95,
			headingPath: ["指南", "快速开始"],
		},
	},
	// 自定义工单卡片
	{
		id: "2",
		role: "assistant",
		itemType: "ticket-card",
		data: { id: "42", status: "处理中", priority: "高" },
	},
	// 默认纯文本（无 itemType）
	{
		id: "3",
		role: "assistant",
		content: "这是普通文本消息",
		sources: [{ id: "s1", label: "来源1", sourceHref: "#" }],
	},
];
</script>

<template>
	<AiChat :messages="messages" :custom-renderers="{ 'ticket-card': TicketCard }" />
</template>
```

### 4.5.6 实施任务清单

| #   | 任务                                                | 文件                                                       | 优先级 |
| --- | --------------------------------------------------- | ---------------------------------------------------------- | ------ |
| 1   | 扩展 AiChatMessage 类型，新增 itemType 和 data 字段 | `ai-vue/src/components/ai-chat/types.ts`                   | P0     |
| 2   | 新增 customRenderers prop                           | `ai-vue/src/components/ai-chat/types.ts`                   | P0     |
| 3   | 实现 SearchResultCard 卡片组件                      | `ai-vue/src/components/ai-chat/cards/SearchResultCard.vue` | P0     |
| 4   | 实现 SourceListCard 卡片组件                        | `ai-vue/src/components/ai-chat/cards/SourceListCard.vue`   | P0     |
| 5   | 修改 AiChat.vue 接入 #item slot 分发逻辑            | `ai-vue/src/components/ai-chat/AiChat.vue`                 | P0     |
| 6   | 导出内置卡片组件供使用者引用                        | `ai-vue/src/index.ts`                                      | P1     |
| 7   | 编写 DataComponent 单元测试                         | `ai-vue/src/tests/data-component.test.ts`                  | P1     |

---

## 五、P4：组件形态扩展 + 函数式嵌入

### 5.1 AiSidebarChat

```vue
<!-- packages/ai-vue/src/components/ai-sidebar-chat/AiSidebarChat.vue -->
<script setup lang="ts">
import { ref } from "vue";
import AiChat from "../ai-chat/AiChat.vue";
import type { AiChatProps, AiChatEmits } from "../ai-chat/types";
import type { BrandThemeConfig } from "../../theme/types";

const props = defineProps<
	AiChatProps & {
		brandTheme?: BrandThemeConfig;
		/** 侧边栏宽度 */
		width?: string;
		/** 是否默认展开 */
		defaultOpen?: boolean;
	}
>();

const emit = defineEmits<AiChatEmits>();
const isOpen = ref(props.defaultOpen ?? false);

function toggle() {
	isOpen.value = !isOpen.value;
}
</script>

<template>
	<div class="ai-sidebar-chat" :class="{ 'ai-sidebar-chat--open': isOpen }">
		<button class="ai-sidebar-chat__trigger" @click="toggle">
			{{ isOpen ? "关闭" : "AI 对话" }}
		</button>
		<transition name="ai-sidebar-slide">
			<aside v-show="isOpen" class="ai-sidebar-chat__panel" :style="{ width: width ?? '24rem' }">
				<AiChat v-bind="props" @send="emit('send', $event)" @stop="emit('stop')" />
			</aside>
		</transition>
	</div>
</template>
```

### 5.2 AiModalChat

```vue
<!-- packages/ai-vue/src/components/ai-modal-chat/AiModalChat.vue -->
<script setup lang="ts">
import { ref } from "vue";
import AiChat from "../ai-chat/AiChat.vue";
import type { AiChatProps, AiChatEmits } from "../ai-chat/types";
import type { BrandThemeConfig } from "../../theme/types";

const props = defineProps<
	AiChatProps & {
		brandTheme?: BrandThemeConfig;
	}
>();

const emit = defineEmits<AiChatEmits>();
const isOpen = ref(false);

function open() {
	isOpen.value = true;
}
function close() {
	isOpen.value = false;
}
</script>

<template>
	<div>
		<button class="ai-modal-chat__trigger" @click="open">AI 对话</button>
		<transition name="ai-modal-fade">
			<div v-if="isOpen" class="ai-modal-chat__overlay" @click.self="close">
				<div class="ai-modal-chat__dialog">
					<button class="ai-modal-chat__close" @click="close">✕</button>
					<AiChat v-bind="props" @send="emit('send', $event)" @stop="emit('stop')" />
				</div>
			</div>
		</transition>
	</div>
</template>
```

### 5.3 函数式嵌入（在 ai-vue 包内导出，不新增子包）

**设计决策**：不创建独立的 `ai-vue-embed` 子包。理由如下：

1. **monorepo 复杂度**：每新增一个子包，就增加一套 package.json、构建配置、tsconfig、版本号、发布流程、依赖管理。SmallAliceWeb 已有 5 个 packages，不应为单一函数增加包。
2. **Vue 运行时已存在**：inkeep/agents-ui 之所以有独立 JS 包，是因为 React 组件库需要在非 React 环境打包 React 运行时。而 ai-vue 的宿主环境（VitePress）已有 Vue 运行时，`mountAiChat` 只需 `import { createApp } from 'vue'`，由宿主环境的 Vue 提供。
3. **函数足够轻量**：`mountAiChat` 本质是 `createApp(AiChat).mount(target)` 的封装，加上 props 传递和卸载清理，总共不超过 50 行代码，不值得独立成包。
4. **tree-shaking 友好**：放在 ai-vue 包内，通过 `exports` 字段单独导出，未使用 `mountAiChat` 的项目不会打包这部分代码。

````typescript
// packages/ai-vue/src/mount.ts
// 直接在 ai-vue 包内实现，不新增子包

import { createApp, h, type App as VueApp } from "vue";
import AiChat from "./components/ai-chat/AiChat.vue";
import type { AiChatProps } from "./components/ai-chat/types";
import type { BrandThemeConfig } from "./theme/types";

export interface MountAiChatOptions extends AiChatProps {
	brandTheme?: BrandThemeConfig;
	variant?: "no-shadow" | "container-with-shadow";
}

export interface MountResult {
	/** 卸载并清理 DOM */
	unmount: () => void;
	/** 获取 Vue 应用实例（高级用法） */
	app: VueApp;
}

/**
 * 将 AI 聊天组件挂载到指定 DOM 节点。
 * 用于非 Vue 环境（纯 HTML、WordPress、Shopify 等）。
 *
 * @example
 * ```html
 * <div id="ai-chat-target"></div>
 * <script type="module">
 *   import { mountAiChat } from '@ruan-cat-drill-doc/ai-vue';
 *   const { unmount } = mountAiChat('#ai-chat-target', {
 *     brandTheme: { primaryBrandColor: '#3784ff' },
 *     mode: 'external',
 *   });
 * </script>
 * ```
 */
export function mountAiChat(target: string | HTMLElement, options: MountAiChatOptions): MountResult {
	const el = typeof target === "string" ? document.querySelector(target) : target;
	if (!el) throw new Error(`Target not found: ${target}`);

	const app = createApp({
		render() {
			return h(AiChat, {
				...options,
				onSend: options.onSend,
				onStop: options.onStop,
			});
		},
	});

	app.mount(el);

	return {
		app,
		unmount: () => {
			app.unmount();
		},
	};
}
````

在 `package.json` 的 `exports` 中新增入口：

```jsonc
{
	"exports": {
		".": {
			/* 主入口 */
		},
		"./styles": {
			/* 样式入口 */
		},
		"./mount": {
			"types": "./dist/mount.d.ts",
			"import": "./dist/mount.js",
			"require": "./dist/mount.cjs",
		},
	},
}
```

使用者按需引入：

```typescript
// 方式1：只引入 mount 函数（tree-shaking 友好）
import { mountAiChat } from "@ruan-cat-drill-doc/ai-vue/mount";

// 方式2：从主入口引入（与组件一起打包）
import { mountAiChat, AiChat } from "@ruan-cat-drill-doc/ai-vue";
```

---

## 六、任务清单

### P0：vue-element-plus-x 升级（0.5 天）

> 2026-09-05 新增前置任务：P3.5 依赖的 BubbleList `#item`/`itemType` 仅 v2 提供（1.3.98 dist 类型实测无此 API），必须先升级。详见第十三章与 spec 第九章。

| 编号 | 任务                              | 产出                                            | 验收                                 |
| :--- | :-------------------------------- | :---------------------------------------------- | :----------------------------------- |
| P0-1 | 升级 vue-element-plus-x 至 ^2.0.3 | `packages/ai-vue/package.json` + pnpm-lock.yaml | 构建与类型检查通过                   |
| P0-2 | Sender 迁移为 XSender             | 修改 `AiChat.vue`                               | 发送/停止/禁用行为不变，vitest 全绿  |
| P0-3 | 接入 ConfigProvider 主题通道      | 修改 `AiChat.vue`                               | themeOverrides 生效，为 P1/P1.5 铺路 |

### P1：品牌化主题系统（3 天）

| 编号 | 任务                      | 产出                            | 验收                                   |
| ---- | ------------------------- | ------------------------------- | -------------------------------------- |
| P1-1 | 定义主题类型              | `theme/types.ts`                | 类型完整，无 any                       |
| P1-2 | 实现颜色派生函数          | `theme/color-utils.ts`          | hex/rgb 输入均能派生 11 色阶           |
| P1-3 | 定义默认主题令牌          | `theme/default-theme.ts`        | 覆盖 colors/fontFamily/fontSize/zIndex |
| P1-4 | 实现 useBrandTheme        | `composables/useBrandTheme.ts`  | 返回 cssVars 可直接绑定                |
| P1-5 | AiChat 接入主题系统       | 修改 `AiChat.vue`               | brandTheme prop 生效                   |
| P1-6 | AiChatFloatingButton 接入 | 修改 `AiChatFloatingButton.vue` | 品牌色一致                             |
| P1-7 | 编写主题单元测试          | `tests/use-brand-theme.test.ts` | 覆盖率 > 90%                           |

### P2：Shadow DOM 样式隔离（2 天）

| 编号 | 任务                   | 产出                         | 验收                                   |
| ---- | ---------------------- | ---------------------------- | -------------------------------------- |
| P2-1 | 实现 AiShadowRoot 组件 | `components/ai-shadow-root/` | Shadow DOM 正确挂载                    |
| P2-2 | 样式注入机制           | AiShadowRoot 内 style 注入   | 组件样式在 Shadow 内生效               |
| P2-3 | variant 配置接入       | AiChat 新增 variant prop     | no-shadow / container-with-shadow 切换 |
| P2-4 | SSR 降级处理           | isSSR 检测                   | VitePress 构建不报错                   |
| P2-5 | 隔离效果测试           | 手动验证                     | 宿主页面 CSS 不污染组件                |

### P3：富聊天体验增强（4 天）

| 编号 | 任务                  | 产出                               | 验收                 |
| ---- | --------------------- | ---------------------------------- | -------------------- |
| P3-1 | 扩展消息类型          | `types.ts` 新增 component 字段     | 支持富组件指令       |
| P3-2 | 实现自定义渲染器      | `parts/AiChatCustomRenderer.vue`   | 按名称渲染注册的组件 |
| P3-3 | 实现反馈组件          | `parts/AiChatFeedback.vue`         | 正/负面反馈 + 详情   |
| P3-4 | 实现示例问题          | `parts/AiChatExampleQuestions.vue` | 空状态展示可点击问题 |
| P3-5 | 实现消息操作菜单      | `parts/AiChatMessageActions.vue`   | 复制/分享/反馈按钮   |
| P3-6 | 实现事件系统          | `composables/useChatEvents.ts`     | 6 种事件类型正确触发 |
| P3-7 | AiChat 集成全部子部件 | 修改 `AiChat.vue`                  | 各部件按配置显示     |
| P3-8 | 编写富聊天单元测试    | `tests/`                           | 覆盖率 > 85%         |

### P4：组件形态扩展 + 函数式嵌入（3 天）

| 编号 | 任务                        | 产出                             | 验收                         |
| ---- | --------------------------- | -------------------------------- | ---------------------------- |
| P4-1 | 实现 AiSidebarChat          | `components/ai-sidebar-chat/`    | 侧边栏滑入动画正常           |
| P4-2 | 实现 AiModalChat            | `components/ai-modal-chat/`      | 弹窗 + 遮罩正常              |
| P4-3 | 实现 mountAiChat 函数       | `ai-vue/src/mount.ts`            | 非 Vue 环境可挂载            |
| P4-4 | 新增 `./mount` exports 入口 | `ai-vue/package.json`            | 子路径导入正常               |
| P4-5 | 更新 index.ts 导出新组件    | 修改 `ai-vue/src/index.ts`       | SidebarChat/ModalChat 可导入 |
| P4-6 | 编写 mountAiChat 单元测试   | `ai-vue/src/tests/mount.test.ts` | 挂载/卸载/事件传递正常       |

### FC：路线配套任务（随后端修订路线分期，详见第十四章）

| 编号 | 任务                         | 产出                           | 验收                               |
| :--- | :--------------------------- | :----------------------------- | :--------------------------------- |
| FC-1 | 页面上下文采集与透传         | 修改 `ai-vitepress-plugins`    | 请求体携带 pageContext，后端可注入 |
| FC-2 | provider 无关契约回归矩阵    | vitest 矩阵 + 第十二章场景扩展 | 切换 provider 后前端全绿           |
| FC-3 | 客户端 TTFT 与响应元数据事件 | 扩展 4.6 事件系统              | response-metadata 含 ttftMs        |
| FC-4 | 反馈载荷关联 conversationId  | 扩展 4.4 反馈组件              | 载荷可关联后端 qa_records          |
| FC-5 | conversationId 追溯语义固化  | 修改 `useKnowledgeChat`        | 向后兼容、无多轮语义               |

### CC：ContextConfig 动态上下文改造任务（后端为主，详见第十五章）

| 编号 | 任务                                        | 产出                                 | 验收                |
| :--- | :------------------------------------------ | :----------------------------------- | :------------------ |
| CC-0 | openspec change 修订 chat-api Requirement 1 | openspec 变更包                      | spec 修订先行合入   |
| CC-1 | PageContext 共享契约                        | `ai-rag-core` page-context schema    | 两端 typecheck 通过 |
| CC-2 | ChatContext 与来源归一化                    | `ai-rag-api` context/types + sources | 归一化三态单测通过  |
| CC-3 | 函数式 prompt 模板                          | `context/prompt-template.ts`         | 渲染两态快照通过    |
| CC-4 | 接线 contracts/chat.ts                      | 修改 `chat.ts:129` 硬编码段          | 既有 chat 用例全绿  |
| CC-5 | vitest 覆盖降级路径                         | context 模块测试文件                 | 全量通过            |

---

## 七、风险与缓解

### 7.1 Shadow DOM 的 Vue 渲染难题

**风险**：Vue 3 的模板编译器不原生支持将 slot 内容渲染到 Shadow Root。直接使用 `<slot>` 只会将内容渲染到 light DOM。

**缓解**：使用渲染函数（`h()`）+ 手动 `attachShadow` + `Teleport` 到 Shadow Root 的方案。或使用 `@vue/web-component-wrapper` 库。如果实现成本过高，P2 可降级为 CSS Scoping（使用 `:where()` 和高优先级选择器模拟隔离），后续再实现完整 Shadow DOM。

### 7.2 品牌色派生的色相偏差

**风险**：HSL 空间的明度调整在不同色相下视觉效果不一致。例如黄色的高明度区域变化不明显，蓝色则变化显著。

**缓解**：P1 阶段先使用 HSL 方案快速验证，如果效果不佳，在 P1-7 测试中引入 OKLCH 色彩空间（通过 `culori` 库，体积约 8KB）作为可选增强。

### 7.3 向后兼容性

**风险**：现有 `AiChat` 和 `AiChatFloatingButton` 的 API 变更可能破坏 `ai-vitepress-plugins` 的 `useKnowledgeChat` 集成。

**缓解**：所有新增 props 设置默认值，确保不传时行为与当前一致。`useKnowledgeChat` 无需修改即可继续工作。新增能力通过可选 props 暴露。

### 7.4 函数式嵌入的 Vue 运行时依赖

**风险**：`mountAiChat` 依赖 `vue` 的 `createApp`，在非 Vue 环境使用时需要宿主页面自行引入 Vue。

**缓解**：已在文档中说明 `mountAiChat` 要求宿主环境提供 Vue 3.5+。通过 `peerDependencies` 声明 Vue 依赖，构建时将 Vue 设为 external。如需在完全无 Vue 的环境使用，使用者需自行通过 CDN 引入 Vue。

---

## 八、验收检查清单

### 8.1 功能验收

- [ ] 输入 `primaryBrandColor: '#3784ff'`，聊天界面整体呈蓝色系
- [ ] 输入 `customColorScheme: { strong: '#ff0000' }`，仅强调色变红
- [ ] `variant: 'container-with-shadow'` 时，宿主页面 CSS 不影响组件
- [ ] `customComponents: { 'ticket-card': (props) => '<div>工单</div>' }` 时，消息流中渲染工单卡片
- [ ] `feedbackOptions: { enabled: true }` 时，每条助手消息下方出现反馈按钮
- [ ] `exampleQuestions: ['怎么使用？']` 时，空状态展示可点击问题
- [ ] `onChatEvent` 回调能接收到 6 种事件类型
- [ ] `AiSidebarChat` 从右侧滑入，`AiModalChat` 居中弹窗
- [ ] `mountAiChat('#target', config)` 在纯 HTML 页面正常工作

### 8.2 非功能验收

- [ ] ai-vue 包体积增量 < 15KB（gzip，不含 Vue 运行时）
- [ ] `mountAiChat` 通过 `./mount` 子路径导入时 tree-shaking 正常
- [ ] TypeScript 严格模式无报错
- [ ] VitePress SSR 构建无报错
- [ ] 单元测试覆盖率 > 85%
- [ ] 现有 `useKnowledgeChat` 集成无需修改

---

## 九、Vitest 测试用例设计

> 本章节详细定义各阶段的 vitest 测试用例，确保每个能力都有可验证的测试覆盖。
> 测试文件统一放在 `packages/ai-vue/src/tests/` 目录，遵循现有项目的 vitest 配置。

### 9.1 测试环境配置

现有 `packages/ai-vue` 已配置 vitest + jsdom，测试文件放在 `src/tests/` 目录。新增测试沿用此约定：

```typescript
// vitest.config.ts（已有，无需修改）
import { defineConfig } from "vitest/config";
import vue from "@vitejs/plugin-vue";

export default defineConfig({
	plugins: [vue()],
	test: {
		environment: "jsdom",
		globals: true,
	},
});
```

### 9.2 P1 品牌化主题系统测试

#### 9.2.1 `tests/color-utils.test.ts` — 颜色派生函数

```typescript
import { describe, expect, it } from "vitest";
import { deriveColorScheme, hexToHsl, hslToHex } from "../theme/color-utils";

describe("deriveColorScheme", () => {
	it("从 hex 颜色派生 11 个色阶", () => {
		const scheme = deriveColorScheme("#3b82f6");
		expect(scheme).toHaveProperty("lighter");
		expect(scheme).toHaveProperty("light");
		expect(scheme).toHaveProperty("medium");
		expect(scheme).toHaveProperty("strong");
		expect(scheme).toHaveProperty("stronger");
		expect(scheme).toHaveProperty("textColorOnPrimary");
		// 共 11 个字段
		expect(Object.keys(scheme)).toHaveLength(11);
	});

	it("深色品牌色派生的 textColorOnPrimary 为白色", () => {
		const scheme = deriveColorScheme("#1a1a2e");
		expect(scheme.textColorOnPrimary).toBe("#ffffff");
	});

	it("浅色品牌色派生的 textColorOnPrimary 为黑色", () => {
		const scheme = deriveColorScheme("#fbbf24");
		expect(scheme.textColorOnPrimary).toBe("#000000");
	});

	it("rgb 格式输入也能正确派生", () => {
		const scheme = deriveColorScheme("rgb(59, 130, 246)");
		expect(scheme.strong).toMatch(/^#/);
	});

	it("无效颜色输入抛出错误", () => {
		expect(() => deriveColorScheme("not-a-color")).toThrow();
	});
});

describe("hexToHsl / hslToHex", () => {
	it("hex → hsl → hex 往返转换保持一致", () => {
		const original = "#3b82f6";
		const hsl = hexToHsl(original);
		const result = hslToHex(hsl.h, hsl.s, hsl.l);
		// 允许极小误差
		expect(result.toLowerCase()).toBe(original.toLowerCase());
	});
});
```

#### 9.2.2 `tests/use-brand-theme.test.ts` — 品牌主题 composable

```typescript
import { describe, expect, it, vi } from "vitest";
import { useBrandTheme } from "../composables/useBrandTheme";

describe("useBrandTheme", () => {
	it("使用传入的 primaryBrandColor 作为种子色", () => {
		const { seedColor } = useBrandTheme({ primaryBrandColor: "#ff0000" });
		expect(seedColor.value).toBe("#ff0000");
	});

	it("未传入 primaryBrandColor 时使用默认值 #3b82f6", () => {
		const { seedColor } = useBrandTheme();
		expect(seedColor.value).toBe("#3b82f6");
	});

	it("colorSchemeOverrides 能覆盖派生的色阶", () => {
		const { colorScheme } = useBrandTheme({
			primaryBrandColor: "#3b82f6",
			colorSchemeOverrides: { strong: "#ff0000" },
		});
		expect(colorScheme.value.strong).toBe("#ff0000");
		// 其他色阶不受影响
		expect(colorScheme.value.lighter).not.toBe("#ff0000");
	});

	it("cssVars 返回可直接绑定到 style 的对象", () => {
		const { cssVars } = useBrandTheme({ primaryBrandColor: "#3b82f6" });
		expect(cssVars.value).toHaveProperty("--ai-chat-primary");
		expect(cssVars.value["--ai-chat-primary"]).toMatch(/^#/);
	});
});
```

### 9.3 P1.5 Teek 主题色传递测试

#### 9.3.1 `tests/use-theme-color.test.ts` — 运行时主题色获取

```typescript
import { describe, expect, it, beforeEach, vi } from "vitest";
import { useThemeColor } from "../composables/useThemeColor";

// 模拟 document 环境
function mockComputedStyle(props: Record<string, string>) {
	return {
		getPropertyValue: (name: string) => props[name] ?? "",
	};
}

describe("useThemeColor", () => {
	beforeEach(() => {
		// 重置 DOM 状态
		document.documentElement.className = "";
		document.documentElement.removeAttribute("theme-color");
	});

	it("优先读取 --tk-theme-color", () => {
		vi.spyOn(window, "getComputedStyle").mockReturnValue(
			mockComputedStyle({ "--tk-theme-color": "#ff6600", "--vp-c-brand-1": "#3b82f6" }) as any,
		);

		const { primary, isReady } = useThemeColor();
		expect(primary.value).toBe("#ff6600");
		expect(isReady.value).toBe(true);
	});

	it("Teek 变量不存在时降级到 --vp-c-brand-1", () => {
		vi.spyOn(window, "getComputedStyle").mockReturnValue(mockComputedStyle({ "--vp-c-brand-1": "#3b82f6" }) as any);

		const { primary } = useThemeColor();
		expect(primary.value).toBe("#3b82f6");
	});

	it("两个变量都不存在时使用默认值", () => {
		vi.spyOn(window, "getComputedStyle").mockReturnValue(mockComputedStyle({}) as any);

		const { primary } = useThemeColor();
		expect(primary.value).toBe("#3b82f6");
	});

	it("检测暗色模式（html.dark class）", () => {
		document.documentElement.classList.add("dark");
		vi.spyOn(window, "getComputedStyle").mockReturnValue(mockComputedStyle({ "--tk-theme-color": "#3b82f6" }) as any);

		const { isDark } = useThemeColor();
		expect(isDark.value).toBe(true);
	});

	it("非暗色模式时 isDark 为 false", () => {
		vi.spyOn(window, "getComputedStyle").mockReturnValue(mockComputedStyle({ "--tk-theme-color": "#3b82f6" }) as any);

		const { isDark } = useThemeTheme();
		expect(isDark.value).toBe(false);
	});
});
```

#### 9.3.2 `ai-vitepress-plugins/src/tests/theme-style.test.ts` — 增强 Teek 变量桥接测试

在现有测试文件中追加：

```typescript
describe("Teek 主题变量桥接", () => {
	test("桥接 --tk-theme-color 到 --ai-chat-primary-color", () => {
		expect(styleSource).toContain("var(--tk-theme-color");
	});

	test("桥接 --tk-color-primary-light-3 到 --ai-chat-primary-hover-color", () => {
		expect(styleSource).toContain("var(--tk-color-primary-light-3");
	});

	test("桥接 --tk-bg-color 到 --ai-chat-surface-color", () => {
		expect(styleSource).toContain("var(--tk-bg-color");
	});

	test("桥接 --tk-text-color 到 --ai-chat-text-color", () => {
		expect(styleSource).toContain("var(--tk-text-color");
	});

	test("Teek 变量优先于 VitePress 变量（嵌套 var）", () => {
		// 验证优先级：--tk-theme-color 在 --vp-c-brand-1 之前
		const primaryLine = styleSource.match(/--ai-chat-primary-color:[^;]+;/)?.[0] ?? "";
		expect(primaryLine).toContain("--tk-theme-color");
		expect(primaryLine).toContain("--vp-c-brand-1");
		// tk 在 vp 之前
		expect(primaryLine.indexOf("--tk-theme-color")).toBeLessThan(primaryLine.indexOf("--vp-c-brand-1"));
	});

	test("暗色模式覆盖存在", () => {
		expect(styleSource).toContain("html.dark");
		expect(styleSource).toMatch(/html\.dark.*--ai-chat-surface-color/s);
	});
});
```

### 9.4 P2 Shadow DOM 测试

#### 9.4.1 `tests/ai-shadow-root.test.ts` — Shadow DOM 隔离

```typescript
import { describe, expect, it } from "vitest";
import { mount } from "@vue/test-utils";
import AiShadowRoot from "../components/ai-shadow-root/AiShadowRoot.vue";

describe("AiShadowRoot", () => {
	it('variant="container-with-shadow" 时创建 Shadow Root', () => {
		const wrapper = mount(AiShadowRoot, {
			props: { variant: "container-with-shadow" },
			slots: { default: '<div class="test-content">内容</div>' },
		});

		const el = wrapper.element as HTMLElement;
		expect(el.shadowRoot).not.toBeNull();
	});

	it('variant="no-shadow" 时不创建 Shadow Root', () => {
		const wrapper = mount(AiShadowRoot, {
			props: { variant: "no-shadow" },
			slots: { default: '<div class="test-content">内容</div>' },
		});

		const el = wrapper.element as HTMLElement;
		expect(el.shadowRoot).toBeNull();
	});

	it("默认 variant 为 no-shadow", () => {
		const wrapper = mount(AiShadowRoot, {
			slots: { default: "内容" },
		});

		expect(wrapper.props("variant")).toBe("no-shadow");
	});
});
```

### 9.5 P3 富聊天体验测试

#### 9.5.1 `tests/ai-chat-custom-renderer.test.ts` — 自定义渲染器

```typescript
import { describe, expect, it, vi } from "vitest";
import { mount } from "@vue/test-utils";
import AiChat from "../components/ai-chat/AiChat.vue";

describe("AiChat 自定义渲染器", () => {
	it("customComponents 中注册的组件在消息流中渲染", () => {
		const customComponents = {
			"ticket-card": (props: { id: string }) => `工单 #${props.id}`,
		};

		const wrapper = mount(AiChat, {
			props: {
				messages: [
					{
						id: "1",
						role: "assistant",
						content: "",
						customType: "ticket-card",
						customProps: { id: "42" },
					},
				],
				customComponents,
			},
		});

		expect(wrapper.text()).toContain("工单 #42");
	});

	it("未注册的 customType 回退到纯文本渲染", () => {
		const wrapper = mount(AiChat, {
			props: {
				messages: [
					{
						id: "1",
						role: "assistant",
						content: "普通文本",
						customType: "unknown-type",
					},
				],
			},
		});

		expect(wrapper.text()).toContain("普通文本");
	});
});
```

#### 9.5.2 `tests/ai-chat-feedback.test.ts` — 反馈按钮

```typescript
import { describe, expect, it, vi } from "vitest";
import { mount } from "@vue/test-utils";
import AiChat from "../components/ai-chat/AiChat.vue";

describe("AiChat 反馈功能", () => {
	it("feedbackOptions.enabled=true 时助手消息下方出现反馈按钮", () => {
		const wrapper = mount(AiChat, {
			props: {
				messages: [{ id: "1", role: "assistant", content: "回答" }],
				feedbackOptions: { enabled: true },
			},
		});

		const feedbackBtns = wrapper.findAll('[data-testid="feedback-positive"], [data-testid="feedback-negative"]');
		expect(feedbackBtns).toHaveLength(2);
	});

	it("feedbackOptions.enabled=false 时不渲染反馈按钮", () => {
		const wrapper = mount(AiChat, {
			props: {
				messages: [{ id: "1", role: "assistant", content: "回答" }],
				feedbackOptions: { enabled: false },
			},
		});

		expect(wrapper.find('[data-testid="feedback-positive"]').exists()).toBe(false);
	});

	it("点击赞触发 onChatEvent 事件", async () => {
		const onChatEvent = vi.fn();
		const wrapper = mount(AiChat, {
			props: {
				messages: [{ id: "1", role: "assistant", content: "回答" }],
				feedbackOptions: { enabled: true },
				onChatEvent,
			},
		});

		await wrapper.find('[data-testid="feedback-positive"]').trigger("click");
		expect(onChatEvent).toHaveBeenCalledWith(
			expect.objectContaining({
				type: "positive_feedback_submitted",
				messageId: "1",
			}),
		);
	});
});
```

#### 9.5.3 `tests/ai-chat-example-questions.test.ts` — 示例问题

```typescript
import { describe, expect, it, vi } from "vitest";
import { mount } from "@vue/test-utils";
import AiChat from "../components/ai-chat/AiChat.vue";

describe("AiChat 示例问题", () => {
	it("消息列表为空时展示示例问题", () => {
		const wrapper = mount(AiChat, {
			props: {
				messages: [],
				exampleQuestions: ["怎么使用？", "常见问题"],
			},
		});

		expect(wrapper.text()).toContain("怎么使用？");
		expect(wrapper.text()).toContain("常见问题");
	});

	it("消息列表非空时不展示示例问题", () => {
		const wrapper = mount(AiChat, {
			props: {
				messages: [{ id: "1", role: "user", content: "你好" }],
				exampleQuestions: ["怎么使用？"],
			},
		});

		expect(wrapper.text()).not.toContain("怎么使用？");
	});

	it("点击示例问题触发 send 事件", async () => {
		const onSend = vi.fn();
		const wrapper = mount(AiChat, {
			props: {
				messages: [],
				exampleQuestions: ["怎么使用？"],
				onSend,
			},
		});

		await wrapper.find('[data-testid="example-question-0"]').trigger("click");
		expect(onSend).toHaveBeenCalledWith(expect.objectContaining({ content: "怎么使用？" }));
	});
});
```

### 9.5.5 P3.5 DataComponent 结构化卡片测试

#### `tests/data-component.test.ts` — 结构化卡片渲染

```typescript
import { describe, expect, it, vi } from "vitest";
import { mount } from "@vue/test-utils";
import { defineComponent } from "vue";
import AiChat from "../components/ai-chat/AiChat.vue";
import SearchResultCard from "../components/ai-chat/cards/SearchResultCard.vue";
import SourceListCard from "../components/ai-chat/cards/SourceListCard.vue";

describe("DataComponent 结构化卡片渲染", () => {
	describe("内置卡片", () => {
		it('itemType="search-result" 时渲染 SearchResultCard', () => {
			const wrapper = mount(AiChat, {
				props: {
					messages: [
						{
							id: "1",
							role: "assistant",
							itemType: "search-result",
							data: {
								title: "测试标题",
								snippet: "测试摘要",
								sourceUrl: "https://example.com",
								sourceLabel: "来源",
								score: 0.95,
							},
						},
					],
				},
			});

			expect(wrapper.findComponent(SearchResultCard).exists()).toBe(true);
			expect(wrapper.text()).toContain("测试标题");
			expect(wrapper.text()).toContain("测试摘要");
			expect(wrapper.text()).toContain("95%");
		});

		it('itemType="source-list" 时渲染 SourceListCard', () => {
			const wrapper = mount(AiChat, {
				props: {
					messages: [
						{
							id: "1",
							role: "assistant",
							itemType: "source-list",
							sources: [
								{ id: "s1", label: "来源1", sourceHref: "#1" },
								{ id: "s2", label: "来源2", sourceHref: "#2" },
							],
						},
					],
				},
			});

			expect(wrapper.findComponent(SourceListCard).exists()).toBe(true);
			expect(wrapper.text()).toContain("来源1");
			expect(wrapper.text()).toContain("来源2");
		});
	});

	describe("回退逻辑", () => {
		it("无 itemType 时回退到 Markdown 渲染", () => {
			const wrapper = mount(AiChat, {
				props: {
					messages: [{ id: "1", role: "assistant", content: "普通文本消息" }],
				},
			});

			expect(wrapper.findComponent(SearchResultCard).exists()).toBe(false);
			expect(wrapper.text()).toContain("普通文本消息");
		});

		it("itemType 未注册时回退到 Markdown 渲染", () => {
			const wrapper = mount(AiChat, {
				props: {
					messages: [
						{
							id: "1",
							role: "assistant",
							content: "回退文本",
							itemType: "unknown-type",
						},
					],
				},
			});

			expect(wrapper.text()).toContain("回退文本");
		});
	});

	describe("自定义渲染器", () => {
		it("customRenderers 注册的 itemType 正确渲染", () => {
			const TicketCard = defineComponent({
				props: { data: Object },
				template: '<div class="ticket-card">工单 #{{ data.id }}</div>',
			});

			const wrapper = mount(AiChat, {
				props: {
					messages: [
						{
							id: "1",
							role: "assistant",
							itemType: "ticket-card",
							data: { id: 42 },
						},
					],
					customRenderers: { "ticket-card": TicketCard },
				},
			});

			expect(wrapper.find(".ticket-card").exists()).toBe(true);
			expect(wrapper.text()).toContain("工单 #42");
		});

		it("自定义渲染器覆盖同名内置渲染器", () => {
			const CustomSearchCard = defineComponent({
				props: { data: Object },
				template: '<div class="custom-search">自定义搜索卡片</div>',
			});

			const wrapper = mount(AiChat, {
				props: {
					messages: [
						{
							id: "1",
							role: "assistant",
							itemType: "search-result",
							data: { title: "测试" },
						},
					],
					customRenderers: { "search-result": CustomSearchCard },
				},
			});

			expect(wrapper.find(".custom-search").exists()).toBe(true);
			expect(wrapper.findComponent(SearchResultCard).exists()).toBe(false);
		});
	});

	describe("向后兼容", () => {
		it("无 itemType 的消息仍展示来源链接", () => {
			const wrapper = mount(AiChat, {
				props: {
					messages: [
						{
							id: "1",
							role: "assistant",
							content: "回答内容",
							sources: [{ id: "s1", label: "来源1", sourceHref: "#1" }],
						},
					],
				},
			});

			// 来源链接应通过 SourceListCard 渲染
			expect(wrapper.findComponent(SourceListCard).exists()).toBe(true);
			expect(wrapper.text()).toContain("来源1");
		});

		it("有 itemType 的消息不重复展示来源链接", () => {
			const wrapper = mount(AiChat, {
				props: {
					messages: [
						{
							id: "1",
							role: "assistant",
							itemType: "search-result",
							data: { title: "测试", sourceUrl: "#", sourceLabel: "来源" },
							sources: [{ id: "s1", label: "来源1", sourceHref: "#1" }],
						},
					],
				},
			});

			// itemType 消息不展示额外的来源链接
			const sourceListCards = wrapper.findAllComponents(SourceListCard);
			expect(sourceListCards).toHaveLength(0);
		});
	});
});
```

### 9.6 P4 函数式嵌入测试

#### 9.6.1 `tests/mount.test.ts` — mountAiChat 函数

```typescript
import { describe, expect, it, afterEach } from "vitest";
import { mountAiChat } from "../mount";

describe("mountAiChat", () => {
	let container: HTMLDivElement;

	afterEach(() => {
		container?.remove();
	});

	it("通过 CSS 选择器挂载到目标节点", () => {
		container = document.createElement("div");
		container.id = "test-target";
		document.body.appendChild(container);

		const { unmount } = mountAiChat("#test-target", { mode: "mock" });

		expect(container.querySelector(".ai-chat")).toBeTruthy();
		unmount();
	});

	it("通过 HTMLElement 挂载到目标节点", () => {
		container = document.createElement("div");
		document.body.appendChild(container);

		const { unmount } = mountAiChat(container, { mode: "mock" });

		expect(container.querySelector(".ai-chat")).toBeTruthy();
		unmount();
	});

	it("目标节点不存在时抛出错误", () => {
		expect(() => mountAiChat("#nonexistent", { mode: "mock" })).toThrow("Target not found: #nonexistent");
	});

	it("unmount 后清理 DOM", () => {
		container = document.createElement("div");
		document.body.appendChild(container);

		const { unmount } = mountAiChat(container, { mode: "mock" });
		expect(container.querySelector(".ai-chat")).toBeTruthy();

		unmount();
		expect(container.querySelector(".ai-chat")).toBeFalsy();
	});

	it("brandTheme 配置传递到组件", () => {
		container = document.createElement("div");
		document.body.appendChild(container);

		const { unmount } = mountAiChat(container, {
			mode: "mock",
			brandTheme: { primaryBrandColor: "#ff0000" },
		});

		const aiChatEl = container.querySelector(".ai-chat") as HTMLElement;
		const styles = getComputedStyle(aiChatEl);
		// 验证 CSS 变量已注入
		expect(styles.getPropertyValue("--ai-chat-primary")).toBe("#ff0000");

		unmount();
	});

	it("返回的 app 实例可访问", () => {
		container = document.createElement("div");
		document.body.appendChild(container);

		const { app, unmount } = mountAiChat(container, { mode: "mock" });

		expect(app).toBeDefined();
		expect(typeof app.unmount).toBe("function");

		unmount();
	});

	it("事件回调正常传递", async () => {
		container = document.createElement("div");
		document.body.appendChild(container);

		const onSend = vi.fn();
		const { unmount } = mountAiChat(container, {
			mode: "mock",
			onSend,
		});

		// 模拟用户输入并发送
		const input = container.querySelector("textarea") as HTMLTextAreaElement;
		const sendBtn = container.querySelector('[data-testid="send-btn"]') as HTMLButtonElement;

		input.value = "测试消息";
		await sendBtn.click();

		expect(onSend).toHaveBeenCalledWith(expect.objectContaining({ content: "测试消息" }));

		unmount();
	});
});
```

### 9.7 测试覆盖率目标

| 测试文件                            | 覆盖目标 | 关键覆盖点                                      |
| ----------------------------------- | -------- | ----------------------------------------------- |
| `color-utils.test.ts`               | > 95%    | 所有颜色格式输入、边界值、往返转换              |
| `use-brand-theme.test.ts`           | > 90%    | 种子色优先级、覆盖逻辑、cssVars 生成            |
| `use-theme-color.test.ts`           | > 90%    | Teek/VitePress 变量读取、暗色模式检测、SSR 安全 |
| `theme-style.test.ts`               | > 85%    | Teek 变量桥接、优先级、暗色模式覆盖             |
| `ai-shadow-root.test.ts`            | > 85%    | Shadow Root 创建/不创建、variant 切换           |
| `ai-chat-custom-renderer.test.ts`   | > 85%    | 注册组件渲染、未注册回退                        |
| `ai-chat-feedback.test.ts`          | > 85%    | 按钮显示/隐藏、事件触发                         |
| `ai-chat-example-questions.test.ts` | > 85%    | 空状态展示、点击触发                            |
| `data-component.test.ts`            | > 90%    | 内置卡片渲染、回退逻辑、自定义渲染器、向后兼容  |
| `mount.test.ts`                     | > 90%    | 选择器/元素挂载、卸载清理、配置传递、事件回调   |

---

## 十、参考资源

### 10.1 @inkeep/agents-ui 关键源码位置

| 文件                                | 作用                                                   |
| ----------------------------------- | ------------------------------------------------------ |
| `dist/types/config/base.d.ts`       | `InkeepBaseSettings`（品牌色、组织名、颜色模式）       |
| `dist/types/config/ai.d.ts`         | `InkeepAIChatSettings`（聊天配置、上下文、文件、表单） |
| `dist/types/config/components.d.ts` | `ComponentsConfig`（自定义渲染器注册）                 |
| `dist/types/theme.d.ts`             | `IkpTheme` / `UserProvidedColorScheme`（主题令牌）     |
| `dist/types/shadow.d.ts`            | Shadow DOM 配置类型                                    |
| `dist/types/events.d.ts`            | `InkeepCallbackEvent`（事件链）                        |
| `dist/types/color-mode.d.ts`        | `ColorModeProviderProps`（暗色模式）                   |
| `dist/react/embedded-chat.d.ts`     | `InkeepEmbeddedChatProps`（组件 props）                |

### 10.2 实际使用示例

| 文件                                                | 作用                                                         |
| --------------------------------------------------- | ------------------------------------------------------------ |
| `agents-manage-ui/.../chat-widget.tsx`              | 完整的生产环境使用示例（品牌色、主题覆盖、事件、自定义组件） |
| `agents-manage-ui/.../chat-ui-code.tsx`             | 代码生成器（React / JS 两种嵌入方式）                        |
| `agents-manage-ui/.../snippets/js-embedded-chat.ts` | JS 函数式嵌入示例                                            |
| `agents-manage-ui/.../snippets/react-component.ts`  | React 组件嵌入示例                                           |
| `agents-ui-demo/src/App.tsx`                        | 最简使用示例                                                 |

### 10.3 SmallAliceWeb 现有代码

| 文件                                                                       | 作用                              |
| -------------------------------------------------------------------------- | --------------------------------- |
| `packages/ai-vue/src/components/ai-chat/AiChat.vue`                        | 当前聊天组件（待增强）            |
| `packages/ai-vue/src/components/ai-chat/types.ts`                          | 当前类型定义（待扩展）            |
| `packages/ai-vue/src/styles/index.scss`                                    | 当前样式（CSS 变量驱动）          |
| `packages/ai-vue/src/composables/useMockAiChat.ts`                         | Mock 对话 composable              |
| `packages/ai-vitepress-plugins/src/client/composables/useKnowledgeChat.ts` | RAG 聊天 composable（需保持兼容） |

---

## 十一、P1.5 Teek 主题色传递链路方案 [新增]

> 本阶段是对 P1 品牌化主题系统的补充，解决 Teek 主题色 → ai-vitepress-plugins → ai-vue 的完整传递问题。
> 建议在 P1 之后、P2 之前实施，因为 P2 的 Shadow DOM 隔离需要先解决主题色传递。

### 11.1 Teek 主题色变量调研结果

通过分析 `vitepress-theme-teek@1.6.2` 的源码，确认 Teek 主题提供以下关键颜色变量：

| Teek 变量                            | 引用关系                  | 用途                                      |
| ------------------------------------ | ------------------------- | ----------------------------------------- |
| `--tk-theme-color`                   | `var(--vp-c-brand-1)`     | Teek 主主题色，直接引用 VitePress brand-1 |
| `--tk-color-primary`                 | Teek 自定义               | Element Plus 兼容主色                     |
| `--tk-color-primary-light-3/5/7/8/9` | Teek 派生                 | Element Plus 兼容色阶（light-9 最浅）     |
| `--tk-el-color-primary`              | `var(--tk-color-primary)` | Element Plus primary 色别名               |
| `--tk-bg-color`                      | Teek 自定义               | Teek 背景色                               |
| `--tk-text-color`                    | Teek 自定义               | Teek 文字色                               |
| `--tk-fill-color-dark`               | Teek 自定义               | 骨架屏填充色                              |

**动态主题色切换机制**：Teek 通过 `html[theme-color="tk-primary"]` 等属性选择器实现主题色切换，每个 theme-color 值对应一组不同的 `--tk-color-primary` 色板。

### 11.2 方案一：CSS 变量桥接增强（ai-vitepress-plugins 层）

在 `packages/ai-vitepress-plugins/src/client/style.css` 中增强桥接，优先使用 Teek 变量：

```css
/* === Teek 主题色桥接（优先于 VitePress 变量） === */

/* 主色调：Teek > VitePress > 固定 fallback */
.ai-chat-vitepress-shell {
	/* 主色：优先取 Teek 主题色，降级到 VitePress brand-1 */
	--ai-chat-primary-color: var(--tk-theme-color, var(--vp-c-brand-1, #3b82f6));
	--ai-chat-primary-hover-color: var(--tk-color-primary-light-3, var(--vp-c-brand-2, #60a5fa));
	--ai-chat-primary-soft-color: var(--tk-color-primary-light-9, var(--vp-c-brand-soft, rgb(59 130 246 / 12%)));
	--ai-chat-primary-contrast-color: var(--vp-c-white, #ffffff);

	/* 背景层：优先取 Teek 背景色 */
	--ai-chat-surface-color: var(--tk-bg-color, var(--vp-c-bg-elv, var(--vp-c-bg, #ffffff)));
	--ai-chat-surface-muted-color: var(--vp-c-bg-soft, var(--vp-c-bg-alt, #f6f6f7));
	--ai-chat-surface-elevated-color: var(--vp-c-bg, #ffffff);

	/* 文字层：优先取 Teek 文字色 */
	--ai-chat-text-color: var(--tk-text-color, var(--vp-c-text-1, #213547));
	--ai-chat-text-muted-color: var(--vp-c-text-2, #676e7b);

	/* 边框层 */
	--ai-chat-border-color: var(--vp-c-divider, rgb(60 60 67 / 12%));
	--ai-chat-border-strong-color: var(--vp-c-divider, rgb(60 60 67 / 29%));

	/* 状态色 */
	--ai-chat-focus-color: var(--tk-color-primary-light-9, var(--vp-c-brand-soft, rgb(59 130 246 / 32%)));
	--ai-chat-success-color: var(--vp-c-green-1, #16a34a);
	--ai-chat-danger-color: var(--vp-c-danger-1, #b91c1c);
}

/* === Teek 动态主题色切换支持 === */
/* 当 Teek 切换主题色时，html 元素会获得 theme-color 属性 */
/* 我们需要确保 AI 聊天组件在这些属性变化时自动跟随 */

/* 暗色模式覆盖 */
html.dark .ai-chat-vitepress-shell {
	--ai-chat-surface-color: var(--tk-bg-color, var(--vp-c-bg-elv, #1a1a1a));
	--ai-chat-surface-muted-color: var(--vp-c-bg-soft, #252525);
	--ai-chat-surface-elevated-color: var(--vp-c-bg, #1a1a1a);
	--ai-chat-text-color: var(--tk-text-color, var(--vp-c-text-1, #ffffff));
	--ai-chat-text-muted-color: var(--vp-c-text-2, #a0a0a0);
}
```

### 11.3 方案二：运行时主题色获取 composable（ai-vue 层）

新增 `useThemeColor` composable，在运行时获取当前生效的主题色：

```typescript
// packages/ai-vue/src/composables/useThemeColor.ts
import { onBeforeUnmount, onMounted, ref, type Ref } from "vue";

export interface ThemeColorState {
	/** 当前主色调，如 #3b82f6 */
	primary: Ref<string>;
	/** 当前是否为暗色模式 */
	isDark: Ref<boolean>;
	/** 是否已初始化（SSR 安全） */
	isReady: Ref<boolean>;
}

/**
 * 运行时获取 Teek/VitePress 主题色。
 *
 * 通过 getComputedStyle 读取 CSS 变量的实际计算值，
 * 并通过 MutationObserver 监听 html 元素的 class 和 attribute 变化，
 * 实现主题色切换时自动更新。
 *
 * SSR 安全：在服务端渲染时返回默认值，不访问 document。
 */
export function useThemeColor(): ThemeColorState {
	const primary = ref("#3b82f6");
	const isDark = ref(false);
	const isReady = ref(false);

	let observer: MutationObserver | null = null;

	/** 从 DOM 读取当前主题色 */
	function readThemeColor() {
		if (typeof document === "undefined") return;

		const root = document.documentElement;
		const computed = getComputedStyle(root);

		// 优先读取 Teek 主题色，降级到 VitePress brand-1
		const tkColor = computed.getPropertyValue("--tk-theme-color").trim();
		const vpColor = computed.getPropertyValue("--vp-c-brand-1").trim();

		if (tkColor) {
			primary.value = tkColor;
		} else if (vpColor) {
			primary.value = vpColor;
		}

		// 检测暗色模式
		isDark.value = root.classList.contains("dark");
		isReady.value = true;
	}

	onMounted(() => {
		readThemeColor();

		// 监听 html 元素的 class 和 attribute 变化
		// Teek 切换主题色时会修改 html 的 theme-color 属性
		// VitePress 切换暗色模式时会切换 html 的 dark class
		observer = new MutationObserver((mutations) => {
			for (const mutation of mutations) {
				if (
					mutation.type === "attributes" &&
					(mutation.attributeName === "class" ||
						mutation.attributeName === "theme-color" ||
						mutation.attributeName === "data-theme")
				) {
					readThemeColor();
					break;
				}
			}
		});

		observer.observe(document.documentElement, {
			attributes: true,
			attributeFilter: ["class", "theme-color", "data-theme"],
		});
	});

	onBeforeUnmount(() => {
		observer?.disconnect();
		observer = null;
	});

	return { primary, isDark, isReady };
}
```

### 11.4 方案三：useBrandTheme 与 useThemeColor 的协同

修改 P1 阶段的 `useBrandTheme`，优先使用运行时获取的 Teek 主题色：

```typescript
// packages/ai-vue/src/composables/useBrandTheme.ts（修改版）
import { computed, type Ref } from "vue";
import { useThemeColor } from "./useThemeColor";

export interface UseBrandThemeOptions {
	/** 手动传入的品牌色，作为运行时获取失败时的降级 */
	primaryBrandColor?: string;
	/** 自定义色阶覆盖 */
	colorSchemeOverrides?: Partial<UserProvidedColorScheme>;
}

export function useBrandTheme(options: UseBrandThemeOptions = {}) {
	// 尝试从运行时获取 Teek 主题色
	const { primary: runtimePrimary, isDark, isReady } = useThemeColor();

	// 种子色优先级：运行时 Teek 主题色 > 手动传入 > 默认值
	const seedColor = computed(() => {
		if (isReady.value && runtimePrimary.value) {
			return runtimePrimary.value;
		}
		return options.primaryBrandColor ?? "#3b82f6";
	});

	// 从种子色派生完整色板（P1 阶段的色板派生逻辑）
	const colorScheme = computed(() => {
		const base = deriveColorScheme(seedColor.value);
		return { ...base, ...options.colorSchemeOverrides };
	});

	// 生成 CSS 变量对象
	const cssVars = computed(() => ({
		"--ai-chat-primary": colorScheme.value.strong,
		"--ai-chat-primary-hover": colorScheme.value.stronger,
		"--ai-chat-primary-soft": colorScheme.value.lighter,
		"--ai-chat-surface": isDark.value ? colorScheme.value.medium : colorScheme.value.lightSubtle,
		// ... 其他变量
	}));

	return { seedColor, colorScheme, cssVars, isDark, isReady };
}
```

### 11.5 实施任务清单

| #   | 任务                                    | 文件                                                 | 优先级 |
| --- | --------------------------------------- | ---------------------------------------------------- | ------ |
| 1   | 增强 style.css 桥接 Teek 变量           | `ai-vitepress-plugins/src/client/style.css`          | P0     |
| 2   | 新增暗色模式覆盖                        | `ai-vitepress-plugins/src/client/style.css`          | P0     |
| 3   | 新增 useThemeColor composable           | `ai-vue/src/composables/useThemeColor.ts`            | P0     |
| 4   | 修改 useBrandTheme 协同 useThemeColor   | `ai-vue/src/composables/useBrandTheme.ts`            | P1     |
| 5   | 更新 theme-style.test.ts 覆盖 Teek 变量 | `ai-vitepress-plugins/src/tests/theme-style.test.ts` | P1     |
| 6   | 新增 useThemeColor 单元测试             | `ai-vue/src/tests/use-theme-color.test.ts`           | P1     |
| 7   | 修改 AiChat.vue 接入 useBrandTheme      | `ai-vue/src/components/ai-chat/AiChat.vue`           | P2     |

---

## 十二、Agent-Browser 视觉验证流程 [新增]

> 本章节提供基于 Windows 系统 + agent-browser CLI 的可复现视觉验证流程，
> 供其他 AI agent 完成自主视觉验证。

### 12.1 环境准备

#### 11.1.1 安装 agent-browser

```bash
# 在 Windows PowerShell 中执行
npm install -g agent-browser
agent-browser install
agent-browser install --with-deps
```

#### 11.1.2 启动本地文档站

```bash
# 在 SmallAliceWeb 仓库根目录
pnpm install
pnpm run docs:dev
# 文档站启动在 http://localhost:5173
```

#### 11.1.3 确认验证目标

验证目标清单：

| #   | 验证场景                                 | 预期结果                     |
| --- | ---------------------------------------- | ---------------------------- |
| V1  | 默认主题色（indigo）下 AI 聊天组件主色调 | 与导航栏主色调视觉一致       |
| V2  | 切换 Teek 主题色后 AI 聊天组件主色调     | 跟随切换为新主题色           |
| V3  | 暗色模式下 AI 聊天组件配色               | 背景变暗、文字变亮、边框适配 |
| V4  | AI 聊天组件悬浮按钮颜色                  | 与主题色一致                 |
| V5  | AI 聊天组件来源链接颜色                  | 使用主题色                   |

### 12.2 视觉验证流程

#### 步骤 1：打开文档站并截图基线

```bash
# 打开文档站
agent-browser open http://localhost:5173

# 等待页面加载完成
agent-browser wait --load networkidle

# 截图作为基线
agent-browser screenshot --full baseline-default-light.png
```

#### 步骤 2：验证默认主题色（V1）

```bash
# 获取页面交互元素快照
agent-browser snapshot -i

# 查找 AI 聊天悬浮按钮（通常在右下角）
# 假设快照显示按钮 ref 为 @e1

# 点击打开 AI 聊天面板
agent-browser click @e1

# 等待面板展开动画完成
agent-browser wait --time 1000

# 截图 AI 聊天面板
agent-browser screenshot ai-chat-panel-default.png

# 通过 JavaScript 获取当前主题色值
agent-browser eval "
  const root = document.documentElement;
  const tk = getComputedStyle(root).getPropertyValue('--tk-theme-color').trim();
  const vp = getComputedStyle(root).getPropertyValue('--vp-c-brand-1').trim();
  const aiChat = getComputedStyle(document.querySelector('.ai-chat')).getPropertyValue('--ai-chat-primary-color').trim();
  JSON.stringify({ tkThemeColor: tk, vpBrand1: vp, aiChatPrimary: aiChat });
"

# 预期输出：三个值应该一致或 aiChatPrimary 应引用 tk/vp 的值
# 记录输出到验证报告
```

#### 步骤 3：验证 Teek 主题色切换（V2）

```bash
# 查找 Teek 主题色切换器（通常在导航栏设置中）
agent-browser snapshot -i

# 假设主题色切换按钮 ref 为 @e2
# 点击打开主题色选择面板
agent-browser click @e2

# 等待面板展开
agent-browser wait --time 500

# 重新快照找到主题色选项
agent-browser snapshot -i

# 假设绿色主题选项 ref 为 @e3
agent-browser click @e3

# 等待主题色切换完成
agent-browser wait --time 1000

# 截图切换后的 AI 聊天面板
agent-browser screenshot ai-chat-panel-green-theme.png

# 再次获取主题色值，验证是否已切换
agent-browser eval "
  const root = document.documentElement;
  const tk = getComputedStyle(root).getPropertyValue('--tk-theme-color').trim();
  const aiChat = getComputedStyle(document.querySelector('.ai-chat')).getPropertyValue('--ai-chat-primary-color').trim();
  JSON.stringify({ tkThemeColor: tk, aiChatPrimary: aiChat, themeAttr: root.getAttribute('theme-color') });
"

# 预期：tkThemeColor 和 aiChatPrimary 应变为绿色系值
```

#### 步骤 4：验证暗色模式（V3）

```bash
# 查找暗色模式切换按钮（通常在导航栏）
agent-browser snapshot -i

# 假设暗色模式切换按钮 ref 为 @e4
agent-browser click @e4

# 等待暗色模式切换完成
agent-browser wait --time 1000

# 截图暗色模式下的 AI 聊天面板
agent-browser screenshot ai-chat-panel-dark-mode.png

# 获取暗色模式下的颜色值
agent-browser eval "
  const root = document.documentElement;
  const isDark = root.classList.contains('dark');
  const surface = getComputedStyle(document.querySelector('.ai-chat')).getPropertyValue('--ai-chat-surface-color').trim();
  const text = getComputedStyle(document.querySelector('.ai-chat')).getPropertyValue('--ai-chat-text-color').trim();
  JSON.stringify({ isDark, surfaceColor: surface, textColor: text });
"

# 预期：isDark 为 true，surfaceColor 为深色，textColor 为浅色
```

#### 步骤 5：验证悬浮按钮颜色（V4）

```bash
# 关闭聊天面板，回到只有悬浮按钮的状态
agent-browser click @e1
agent-browser wait --time 500

# 截图悬浮按钮
agent-browser screenshot ai-chat-floating-button.png

# 获取悬浮按钮的颜色
agent-browser eval "
  const btn = document.querySelector('.ai-chat-floating-button__trigger');
  const bg = getComputedStyle(btn).backgroundColor;
  const color = getComputedStyle(btn).color;
  JSON.stringify({ backgroundColor: bg, color: color });
"

# 预期：backgroundColor 应为主题色
```

#### 步骤 6：关闭浏览器

```bash
agent-browser close
```

### 12.3 验证报告模板

每次视觉验证后，填写以下报告：

```markdown
## 视觉验证报告

- **日期**：YYYY-MM-DD
- **验证人**：AI Agent / 人类
- **文档站版本**：commit hash
- **agent-browser 版本**：x.x.x

### 验证结果

| #   | 场景       | 状态  | 截图                          | 备注                                |
| --- | ---------- | ----- | ----------------------------- | ----------------------------------- |
| V1  | 默认主题色 | ✅/❌ | ai-chat-panel-default.png     | tkThemeColor=xxx, aiChatPrimary=xxx |
| V2  | 主题色切换 | ✅/❌ | ai-chat-panel-green-theme.png | 切换后 tkThemeColor=xxx             |
| V3  | 暗色模式   | ✅/❌ | ai-chat-panel-dark-mode.png   | isDark=true, surfaceColor=xxx       |
| V4  | 悬浮按钮   | ✅/❌ | ai-chat-floating-button.png   | backgroundColor=xxx                 |
| V5  | 来源链接   | ✅/❌ | —                             | color=xxx                           |

### 失败分析（如有）

对于每个 ❌ 项，记录：

1. 预期值 vs 实际值
2. 可能的根因
3. 修复建议
```

### 12.4 自动化验证脚本

将上述流程封装为可重复执行的脚本：

```bash
#!/bin/bash
# scripts/verify-ai-chat-theme.sh
# 在 Windows Git Bash 或 WSL 中执行

set -e

BASE_URL="${1:-http://localhost:5173}"
OUTPUT_DIR="${2:-./verify-screenshots}"

mkdir -p "$OUTPUT_DIR"

echo "=== AI 聊天主题色视觉验证 ==="
echo "目标: $BASE_URL"
echo "输出: $OUTPUT_DIR"

# 步骤1：打开页面
agent-browser open "$BASE_URL"
agent-browser wait --load networkidle
agent-browser screenshot --full "$OUTPUT_DIR/baseline.png"

# 步骤2：验证默认主题色
agent-browser snapshot -i > "$OUTPUT_DIR/snapshot-1.txt"
# AI 按钮通常包含 "AI 对话" 文本
AI_BTN_REF=$(grep -oP '@e\d+' "$OUTPUT_DIR/snapshot-1.txt" | head -1)

if [ -z "$AI_BTN_REF" ]; then
  echo "❌ 未找到 AI 聊天按钮"
  agent-browser close
  exit 1
fi

echo "找到 AI 聊天按钮: $AI_BTN_REF"
agent-browser click "$AI_BTN_REF"
agent-browser wait --time 1000
agent-browser screenshot "$OUTPUT_DIR/ai-chat-default.png"

# 获取颜色值
agent-browser eval "
  const r = document.documentElement;
  const c = getComputedStyle(r);
  const ai = document.querySelector('.ai-chat');
  const aiC = ai ? getComputedStyle(ai) : null;
  JSON.stringify({
    tkThemeColor: c.getPropertyValue('--tk-theme-color').trim(),
    vpBrand1: c.getPropertyValue('--vp-c-brand-1').trim(),
    aiChatPrimary: aiC ? aiC.getPropertyValue('--ai-chat-primary-color').trim() : 'N/A',
    isDark: r.classList.contains('dark')
  });
" > "$OUTPUT_DIR/colors-default.json"

echo "默认主题色颜色值已保存到 colors-default.json"

# 步骤3：验证暗色模式
agent-browser snapshot -i > "$OUTPUT_DIR/snapshot-2.txt"
DARK_BTN_REF=$(grep -i "dark\|暗\|夜" "$OUTPUT_DIR/snapshot-2.txt" | grep -oP '@e\d+' | head -1)

if [ -n "$DARK_BTN_REF" ]; then
  agent-browser click "$DARK_BTN_REF"
  agent-browser wait --time 1000
  agent-browser screenshot "$OUTPUT_DIR/ai-chat-dark.png"
  echo "暗色模式截图已保存"
else
  echo "⚠️ 未找到暗色模式切换按钮，跳过暗色验证"
fi

# 清理
agent-browser close

echo "=== 验证完成 ==="
echo "截图和颜色数据保存在: $OUTPUT_DIR"
echo "请人工检查截图，确认 AI 聊天组件颜色与文档站主题色一致"
```

### 12.5 注意事项

1. **Windows 路径**：在 Windows PowerShell 中执行时，路径使用反斜杠 `\` 或正斜杠 `/` 均可，但建议统一使用正斜杠
2. **端口冲突**：如果 5173 端口被占用，VitePress 会自动切换到 5174 等，需确认实际端口
3. **SSR 注意**：VitePress 的 SSR 模式下 `agent-browser eval` 可能无法获取到客户端注入的 CSS 变量，需确保在客户端渲染完成后执行
4. **截图对比**：建议使用工具（如 `pixelmatch`）对比基线截图和验证截图，实现自动化视觉回归
5. **CI 集成**：可将 `verify-ai-chat-theme.sh` 集成到 CI 流程，在每次 PR 时自动执行视觉验证

---

## 十三、vue-element-plus-x 升级与组件复用实施方案 [新增]

> 2026-09-05 新增。依据 spec 第九章的 dist 实测结论：当前安装的 1.3.98 没有 BubbleList `#item`/`itemType` API，P3.5 方案只有升级到 v2 才能落地。本章任务作为 P0 前置，先于 P1-P4 执行。

### 13.1 升级决策回顾

| 问题                       | 结论                                                                                                                                                                                                           |
| :------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 要不要升级到最新版         | 要。升级至 `^2.0.3`，作为 P0 前置任务                                                                                                                                                                          |
| 最新版是否提供更多可用组件 | 提供的是「更聚焦」的组件集：新增 `XSender`、`ConfigProvider` 主题系统、`useTheme`，BubbleList 获得虚拟滚动与 `#item`/`itemType`；移除 `Sender`/`Typewriter`/`XMarkdown*`（markdown 归 markstream-vue，零损失） |
| 如何复用而不重复造轮子     | 三层分工：原子组件复用 vepx，ai-vue 只做主题/类型/事件/隔离等包装层（详见 spec 9.5 映射表）                                                                                                                    |

### 13.2 Sender → XSender 迁移对照（AiChat.vue）

v2 的 `XSender` 基于 `x-sender` 包重构，与 1.3.98 `Sender` 的 API 差异以 dist 类型实测为准：

| 1.3.98 `Sender` 现状             | 2.0.3 `XSender` 对应                     | 说明                                                   |
| :------------------------------- | :--------------------------------------- | :----------------------------------------------------- |
| `v-model="input"`（string）      | 无 `v-model`（实测无 `modelValue` prop） | 经模板 ref 暴露的 `getModelValue()` 读取 `{html,text}` |
| `:auto-size="{minRows,maxRows}"` | 无对应                                   | 高度自适应，`variant: 'default'\|'updown'` 控制布局    |
| `:submit-btn-disabled`           | 无直接对应                               | 由 `:loading` 态与 `handleSend` 内部守卫承接           |
| `:loading`                       | `:loading`                               | 一致，loading 态下发送按钮转为停止                     |
| `:placeholder`                   | `:placeholder`                           | 一致                                                   |
| `@submit` 负载为 string          | `@submit` 无负载                         | 取值改走 `getModelValue().text`                        |
| `@cancel`                        | `@cancel`                                | 一致                                                   |

迁移代码示例（目标形态，实施时以 `vue-element-plus-x/types/XSender` 类型为准核对）：

```vue
<template>
	<XSender
		ref="senderRef"
		:loading="displayedResponding"
		:placeholder="placeholder"
		:submit-type="'enter'"
		@submit="handleXSenderSubmit"
		@cancel="handleStop"
	/>
</template>
```

```ts
import { ref } from "vue";
import { XSender } from "vue-element-plus-x";
import type { ModelValue } from "vue-element-plus-x/types/XSender";

const senderRef = ref<InstanceType<typeof XSender> | null>(null);

/** 读取 XSender 当前输入并清空：XSender 不提供 v-model，值经 expose 的 getModelValue 获取。 */
function readAndClearSender(): string {
	const value: ModelValue | undefined = senderRef.value?.getModelValue();
	senderRef.value?.onClear();
	return value?.text?.trim() ?? "";
}

/** 处理 XSender 提交：submit 事件不携带负载。 */
function handleXSenderSubmit() {
	handleSend(readAndClearSender());
}
```

### 13.3 ConfigProvider 主题通道接入（服务 P1/P1.5）

v2 提供官方主题入口（dist 实测类型）：`ConfigProviderProps { namespace?: string; theme?: 'light'\|'dark'; themeOverrides?: { common?: Record<string, string>; components?: Record<string, Record<string, string>> }; applyTo?: 'root'\|'self' }`。

在 `AiChat.vue` 根部包一层：

```vue
<template>
	<ConfigProvider :theme="isDark ? 'dark' : 'light'" :theme-overrides="brandOverrides" apply-to="self">
		<!-- 现有 AiChat 内容 -->
	</ConfigProvider>
</template>
```

与既有章节的协同关系：

- P1 的 `useBrandTheme` 派生色板 → 输出 `themeOverrides.common`（替代直接覆盖 vepx 内部变量）。
- P1.5 的 `useThemeColor` 运行时读取 Teek 主题色 → 作为派生种子色；`html.dark` 监听 → `ConfigProvider.theme` 切换。
- vepx 的 `useTheme()` hook（namespace/theme/isDark）可用于调试与暗色同步验证。

### 13.4 实施步骤（小步提交，每步可验证）

- [ ] **步骤 1：升级依赖。** 修改 `packages/ai-vue/package.json` 将 `vue-element-plus-x` 置为 `^2.0.3`，执行 `pnpm install`；核对已入库的 `pnpm-lock.yaml` diff 仅包含 vepx 相关条目与新增 `x-sender`、`virtua` 依赖。验证：`pnpm --filter @ruan-cat-drill-doc/ai-vue run build` 通过。
- [ ] **步骤 2：Sender 迁移。** 按 13.2 对照表改写 `AiChat.vue` 输入区；移除 `auto-size`、`submit-btn-disabled`，值读写改走 `getModelValue()`。验证：`pnpm --filter @ruan-cat-drill-doc/ai-vue run test` 全绿（现有用例的 vepx mock 需同步补 `getModelValue`）。
- [ ] **步骤 3：接入 ConfigProvider。** 按 13.3 包裹根部并接临时品牌色常量。验证：浏览器中 vepx 组件颜色随 overrides 变化。
- [ ] **步骤 4：BubbleList 回归核对。** 确认 `:auto-scroll="false"`、`#content`、`#footer` 行为不变；`complete`/`triggerIndices` 未被 AiChat 使用，无需处理。验证：消息流式渲染与来源链接显示正常。
- [ ] **步骤 5：视觉验证收尾。** 按第十二章 agent-browser 流程完成默认主题/主题色切换/暗色模式三场景截图判读。
- [ ] **步骤 6：提交。** 按修改职责拆分：依赖升级（lock）一个提交、Sender 迁移一个提交、ConfigProvider 接入一个提交。

### 13.5 分阶段组件复用清单

| 阶段 | ai-vue 新增能力            | 复用的 vepx 组件                        | 自研部分                                         |
| :--- | :------------------------- | :-------------------------------------- | :----------------------------------------------- |
| P0   | 升级 + 输入迁移 + 主题通道 | `XSender`、`ConfigProvider`、`useTheme` | 迁移适配代码                                     |
| P1   | `useBrandTheme` 色板派生   | `ConfigProvider.themeOverrides`         | `color-utils` 派生算法                           |
| P1.5 | Teek 主题色传递            | `useTheme`（调试/验证）                 | `useThemeColor`、CSS 桥接                        |
| P2   | Shadow DOM 隔离            | 不变                                    | `AiShadowRoot`                                   |
| P3   | 反馈/消息操作/事件链       | `Bubble` `#footer`、`Thinking`          | `useChatEvents`、反馈组件                        |
| P3.5 | DataComponent 卡片         | `BubbleList` `#item` + `itemType`       | `SearchResultCard`、`SourceListCard`、渲染器注册 |
| P4   | Sidebar/Modal/函数式嵌入   | 复用 AiChat 全量                        | 容器组件、`mountAiChat`                          |

### 13.6 验收标准

- [ ] `vue-element-plus-x` 锁定 `^2.0.3`，`pnpm-lock.yaml` 入库且 diff 干净
- [ ] `Sender`→`XSender` 迁移后发送、停止、loading 禁用逻辑与升级前行为一致
- [ ] 空状态 `Bubble`、消息列表 `#content`/`#footer` 渲染回归通过
- [ ] `ConfigProvider` 主题通道生效，P1 落地时色板直接可注入
- [ ] vitest 全量通过（更新 vepx mock 以匹配 v2 导出面）

---

## 十四、后端路线修订的前端配套实施方案 [新增]

> 2026-09-05 新增。依据[重调研报告](../2026-09-05-inkeep-agents-local-research-report.md)第四章修订路线与 spec 第十章：前端只做配套义务（采集/透传/契约稳定/反馈关联），后端事项（模型注册表泛化、qa_records、评估钩子）在 `ai-rag-api` 侧另行立项 openspec change。

### 14.1 分工边界

| 后端路线项（ai-rag-api）             | 前端配套任务                                 | 所在包                              | 优先级 |
| :----------------------------------- | :------------------------------------------- | :---------------------------------- | :----- |
| 模型更换与切换（P0）                 | FC-2 provider 无关契约回归矩阵               | ai-vue + ai-vitepress-plugins 测试  | P0     |
| prompt 配置化 + 页面上下文注入（P1） | FC-1 页面上下文采集与透传                    | ai-vitepress-plugins                | P1     |
| TTFT 可观测（P1 横切）               | FC-3 客户端 TTFT 与 response-metadata 事件   | ai-vitepress-plugins + 4.6 事件系统 | P1     |
| 单轮回流评估（P2）                   | FC-4 反馈载荷关联 + FC-5 conversationId 语义 | 4.4 反馈组件 + useKnowledgeChat     | P2     |

### 14.2 FC-1：页面上下文采集与透传

`useKnowledgeChat` 现签名（实测 `ai-vitepress-plugins/src/client/composables/useKnowledgeChat.ts:92`）：`useKnowledgeChat(conversationId = "knowledge-chat", options)`。扩展示意：

```ts
import type { PageContext } from "../types";

/** 采集当前文档页上下文；SSR 或非文档环境返回 undefined，请求照常发送。 */
function collectPageContext(): PageContext | undefined {
	if (typeof window === "undefined") return undefined;
	const route = useRoute();
	const { frontmatter, title } = useData();
	if (!route?.path) return undefined;
	return {
		pagePath: route.path,
		title: typeof title.value === "string" ? title.value : undefined,
		keywords: Array.isArray(frontmatter.value?.keywords) ? frontmatter.value.keywords : undefined,
	};
}
```

实施要点：

- [ ] 在 `ai-vitepress-plugins` 定义 `PageContext` 类型（`pagePath` 必填，`title`/`keywords` 可选），与后端 zod schema 字段一一对应
- [ ] 请求体新增可选 `pageContext` 字段；后端校验失败或字段为空时跳过注入（`requiredToFetch` 降级语义），前端不重试、不阻断
- [ ] **ai-vue 零改动**：传输在 plugins 层，`AiChatProps` 不新增上下文 prop（spec 10.2 判定）
- 验证：`pnpm --filter @ruan-cat-drill-doc/ai-vitepress-plugins run test` 全绿；真实文档页提问「这个怎么配」，后端日志可见 pageContext

### 14.3 FC-2：provider 无关契约回归矩阵

- [ ] 清单化 AiChat 消费的全部流式帧与事件（文本流、来源帧、错误态、abort），形成「前端消费面」表格
- [ ] vitest 断言：mock 传输层在两种 provider 假设（不同上游事件命名）下，前端解析结果一致 —— 对齐 openspec chat-api Requirement 8「下游流格式保持稳定」
- [ ] 第十二章视觉验证流程的场景表新增一行「切换 provider 后回归」：同一 URL 在 provider A/B 下分别执行默认主题、主题色切换、暗色模式三场景截图判读
- 验证：`pnpm --filter @ruan-cat-drill-doc/ai-vue run test` 全绿；视觉验证报告（12.3 模板）记录两次判读一致

### 14.4 FC-3：客户端 TTFT 与 response-metadata 事件

- [ ] `useKnowledgeChat` 内以自定义 fetch 包装记录 `firstChunkAt - requestStart = ttftMs`
- [ ] 4.6 事件系统（`useChatEvents`）新增 `response-metadata` 事件类型：

```ts
/** 响应元数据事件：由后端 data-stream 元数据帧与客户端 TTFT 测量合成。 */
export interface AiChatResponseMetadataEvent {
	type: "response-metadata";
	provider?: string;
	model?: string;
	/** 客户端感知的首 chunk 延迟（毫秒）；与后端分 provider TTFT 交叉验证。 */
	ttftMs: number;
	conversationId: string;
}
```

- [ ] 现有 emit 契约不变：`response-metadata` 为新增事件类型，旧消费方忽略即可
- 验证：事件在 mock 与真实传输下均触发；`ttftMs` 数值合理（> 0）

### 14.5 FC-4：反馈载荷关联 conversationId

- [ ] 4.4 反馈组件（AiChatFeedback）emit 载荷扩展为：

```ts
/** 反馈事件载荷：可唯一定位一条问答记录，供后端 qa_records 关联。 */
export interface AiChatFeedbackPayload {
	conversationId: string;
	messageId: string;
	rating: "positive" | "negative";
	comment?: string;
}
```

- [ ] 后端回流评估（P2）将用户反馈作为评估信号之一；前端只保证载荷完备
- 验证：反馈用例断言载荷字段完整

### 14.6 FC-5：conversationId 追溯语义固化

- [ ] `useKnowledgeChat` 保持默认值 `"knowledge-chat"` 向后兼容；文档注释明确「仅作追溯分组标识，不携带历史注入语义」
- [ ] 宿主可传页面级会话 ID 提升回流粒度（示例：`docs/getting-started#s-3f9a`，即 `pagePath + 会话种子`）；种子由宿主自行管理（sessionStorage 或路由状态），ai-vue/plugins 不强制实现
- [ ] 全链路检查：该 ID 出现在请求体、日志、后端 qa_records 与评估运行中，形成 3.4 所述的「对话 ↔ 外部系统」回链
- 验证：向后兼容用例（不传 ID 行为不变）+ 文档注释就位

### 14.7 实施顺序

| 顺序 | 任务                     | 依赖              | 对应后端项                 |
| :--- | :----------------------- | :---------------- | :------------------------- |
| 1    | FC-2 契约回归矩阵        | 无（纯测试侧）    | 模型切换 P0 先行验收的一半 |
| 2    | FC-1 页面上下文采集      | 后端 zod 契约定稿 | prompt 配置化 P1           |
| 3    | FC-3 TTFT 与元数据事件   | 4.6 事件系统实施  | TTFT 横切 P1               |
| 4    | FC-5 conversationId 语义 | 无                | 单轮回流 P2                |
| 5    | FC-4 反馈载荷            | 4.4 反馈组件实施  | 单轮回流 P2                |

### 14.8 验收标准

- [ ] 页面上下文用例通过：文档页提问「这个怎么配」回答针对当前页
- [ ] 切换 provider 后 vitest 与视觉验证全绿（12.3 报告模板记录）
- [ ] `response-metadata` 事件含 `ttftMs` 且旧消费方零感知
- [ ] 反馈载荷含 `conversationId`/`messageId`，可与后端 `qa_records` 关联
- [ ] 全部新增字段可选，现有 `AiChatProps`/`AiChatEmits`/`useKnowledgeChat` 签名向后兼容，无多轮 UI、无 MCP 依赖

---

## 十五、ContextConfig 动态上下文改造实施方案 [新增]

> 2026-09-05 新增。落地 spec 第十一章：借鉴 inkeep ContextConfig/TemplateEngine 的机制形状（证据：探索笔记 A），按「单轮、无 MCP、web RAG」裁剪。前端采集侧（FC-1，plan 14.2）已就绪，本章以后端 ai-rag-api 为主。

### 15.1 文件变更预览

```plain
packages/ai-rag-core/src/
└── page-context.ts                  # 新增：PageContext zod schema（两端共用契约）
packages/ai-rag-api/server/
├── context/
│   ├── types.ts                     # 新增：ChatContext / ContextSource / ServerFetchDefinition（类型预留）
│   ├── sources.ts                   # 新增：来源归一化与装配
│   └── prompt-template.ts           # 新增：buildSystemPrompt 五段式函数式模板
└── contracts/chat.ts                # 修改：第 129 行硬编码 prompt → buildSystemPrompt 调用；请求 schema 增加可选 pageContext
```

### 15.2 前置步骤：openspec change 修订 chat-api spec

- [ ] 以 openspec change 修订 chat-api Requirement 1 的描述：由「基于检索到的上下文（Top-5）组装 system prompt」扩为「由类型化上下文与模板模块组装 system prompt，页面上下文可选注入、缺失时降级」
- [ ] 行为验收（来源帧、`[来源N]`、拒答文案、流式契约）保持不变
- 纪律：spec 是唯一事实源，spec 修订合入后才允许动 `contracts/chat.ts`

### 15.3 任务 CC-1：PageContext 共享契约（ai-rag-core）

```ts
// packages/ai-rag-core/src/page-context.ts
import { z } from "zod";

/** 客户端页面上下文契约：ai-vitepress-plugins 采集、ai-rag-api 校验共用同一份 schema。 */
export const pageContextSchema = z.object({
	/** 用户当前浏览的文档页路径，如 /guide/install。 */
	pagePath: z.string().min(1).max(512),
	/** 页面标题（可选）。 */
	title: z.string().max(256).optional(),
});

export type PageContext = z.infer<typeof pageContextSchema>;
```

- [ ] `ai-vitepress-plugins` 增加 `"@ruan-cat-drill-doc/ai-rag-core": "workspace:*"` 依赖，FC-1 的采集类型改为引用该 schema 推导类型
- 验证：两包子包 `typecheck` 通过

### 15.4 任务 CC-2：ChatContext 容器与来源归一化（types.ts + sources.ts）

```ts
// packages/ai-rag-api/server/context/types.ts
import type { PageContext } from "@ruan-cat-drill-doc/ai-rag-core";

/** 单轮聊天的动态上下文容器：任一来源可选，缺失仅降级不阻断。 */
export interface ChatContext {
	/** 客户端页面上下文（FC-1 透传）。 */
	page?: PageContext;
	/** 静态站点信息。 */
	site: { name: string };
}

/**
 * 服务端拉取式上下文定义（v2 预留，v1 不实现执行器）。
 * 对齐 inkeep fetchDefinition 形状：url + zod 校验 + timeout + requiredToFetch 跳过语义。
 * 第一个真实拉取场景出现时实现，接口不变。
 */
export interface ServerFetchDefinition {
	url: string;
	schema: import("zod").ZodTypeAny;
	timeoutMs?: number;
	requiredToFetch?: boolean;
}
```

```ts
// packages/ai-rag-api/server/context/sources.ts
import { pageContextSchema } from "@ruan-cat-drill-doc/ai-rag-core";
import type { ChatContext } from "./types";

/** 归一化客户端页面上下文：合法返回值，非法或缺失返回 undefined（永不抛错）。 */
export function normalizeClientContext(input: unknown): PageContext | undefined {
	const parsed = pageContextSchema.safeParse(input);
	return parsed.success ? parsed.data : undefined;
}

/** 装配单轮聊天上下文：来源逐个归一化，失败跳过。 */
export function assembleChatContext(rawPageContext: unknown, siteName: string): ChatContext {
	const context: ChatContext = { site: { name: siteName } };
	const page = normalizeClientContext(rawPageContext);
	if (page) context.page = page;
	return context;
}
```

- 验证：归一化三态单测（合法 / 非法 / 缺失）

### 15.5 任务 CC-3：函数式 prompt 模板（prompt-template.ts）

```ts
// packages/ai-rag-api/server/context/prompt-template.ts
import type { PageContext } from "@ruan-cat-drill-doc/ai-rag-core";
import type { ChatContext } from "./types";

/** 角色设定段。 */
export const ROLE_SEGMENT = "你是知识库问答助手。根据以下参考资料回答问题。";

/** 检索引导段。 */
export const RETRIEVAL_GUIDE_SEGMENT = "回答必须依据参考资料；资料未覆盖的内容不要编造。";

/** 引用格式段。 */
export const CITATION_FORMAT_SEGMENT = "回答中每个观点标注来源 [来源N]，N 对应参考资料编号。";

/** 拒答策略段。 */
export const REFUSAL_POLICY_SEGMENT = "如果资料不足，说明「根据现有资料无法回答」。";

/** 页面上下文注入段：独立段落 + 防误引声明，防止模型把页面信息误标为 [来源N]。 */
export function buildPageContextSegment(page: PageContext): string {
	const lines = [
		"【页面上下文】以下信息仅作语境参考，不得作为来源引用，不计入 [来源N] 编号：",
		`- 用户当前浏览页面：${page.pagePath}`,
	];
	if (page.title) lines.push(`- 页面标题：${page.title}`);
	return lines.join("\n");
}

/** 组装 system prompt：五段式，页面上下文存在时才渲染注入段。 */
export function buildSystemPrompt(context: ChatContext, sources: string[]): string {
	const segments = [
		ROLE_SEGMENT,
		RETRIEVAL_GUIDE_SEGMENT,
		CITATION_FORMAT_SEGMENT,
		context.page ? buildPageContextSegment(context.page) : null,
		`参考资料：\n${sources.map((source, index) => `[${index + 1}] ${source}`).join("\n\n")}`,
		REFUSAL_POLICY_SEGMENT,
	].filter((segment): segment is string => segment !== null);
	return segments.join("\n\n");
}
```

- [ ] 五个段落常量独立导出，便于单测与未来按渠道差异化
- [ ] 对外行为与现实现逐字对齐（除新增的页面上下文段外），保证 openspec 行为验收不变
- 验证：有/无 pageContext 两态渲染快照

### 15.6 任务 CC-4：接线 contracts/chat.ts

- [ ] `chatRequestSchema` 增加可选 `pageContext` 字段（引用 ai-rag-core 的 schema）
- [ ] 第 129 行硬编码段替换为：

```ts
const context = assembleChatContext(parsed.data.pageContext, SITE_NAME);
const system = buildSystemPrompt(
	context,
	sources.map((source) => source.content),
);
```

- [ ] 来源帧、abort 传播、错误映射逻辑零改动
- 验证：现有 chat 用例全绿；新增「携带合法 pageContext 的集成用例」断言 system 含页面路径

### 15.7 任务 CC-5：vitest 覆盖

- [ ] `prompt-template` 渲染两态快照（有/无 pageContext）
- [ ] `sources` 归一化三态（合法 / 非法 / 缺失）
- [ ] 降级路径：非法 pageContext → 基础模板输出、无异常抛出
- 运行：`pnpm --filter @ruan-cat-drill-doc/ai-rag-api run test`

### 15.8 蓝军拷问记录（grill-me 自审：推荐默认已执行，均可推翻）

| 拷问                                           | 结论                                                                                                     | 推翻成本                                               |
| :--------------------------------------------- | :------------------------------------------------------------------------------------------------------- | :----------------------------------------------------- |
| Q1 为何 v1 不实现服务端 fetchDefinition 执行器 | 无真实拉取场景（文档元数据已在检索结果内，页面信息由客户端透传），YAGNI；类型接口已预留                  | 场景出现后约 1 天，接口不变                            |
| Q2 为何 TS 函数式模板而非字符串模板 + 渲染器   | prompt 不入库、无可视化编辑器；函数模板类型安全、零解析层。inkeep 用字符串模板是因 prompt 可被构建器存库 | 未来 prompt 上库时切换载体，buildSystemPrompt 签名不变 |
| Q3 为何字段最小集（pagePath + title）          | 足够支撑「这个怎么配」指代消解；字段越多校验与隐私面越大                                                 | 加字段 = schema + 采集 + 模板三处小改                  |
| Q4 为何独立注入段 + 防误引声明                 | 防止模型把页面信息当作来源标注 [来源 N]，污染引用编号体系                                                | 换注入方式 = 改一个段落函数                            |
| D3 为何 PageContext schema 放 ai-rag-core      | api 已依赖该包（workspace:\*，chat.ts:1 已在用）；plugins 加一条 workspace 依赖换取两端契约不漂移        | 挪包 = 移文件 + 改 import                              |
| 上下文会不会演化成「隐形多轮」                 | 不会：单轮、无状态、全部可选、任何来源失败只降级不阻塞                                                   | —                                                      |

### 15.9 实施顺序与验收标准

实施顺序：openspec change（15.2）→ CC-1 → CC-2 → CC-3 → CC-4 → CC-5；前端 FC-1（plan 14.2）与 CC-1 的 schema 定稿并行。

验收标准：

- [ ] openspec chat-api spec 修订先行合入
- [ ] 文档页 A 提问「这个怎么配」，回答针对页面 A（FC-1 联动联调用例）
- [ ] pageContext 缺失或非法时基础模板回答、无报错
- [ ] 提示词修改只触碰模板模块，`contracts/chat.ts` 请求处理逻辑不变
- [ ] `pnpm --filter @ruan-cat-drill-doc/ai-rag-api run test` 全量通过
- [ ] 四项拍板决策（Q1-Q4）在 15.8 有记录，用户可随时推翻重拍
