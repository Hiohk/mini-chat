# DeepSeek chat-demo 源码导读

> 基于 DeepSeek 官方开源聊天示例 [chat-demo](https://github.com/deepseek-ai/deepseek-chat-demo) 的架构梳理与学习指南。
> 这是一个**纯前端、零后端**项目：浏览器直连 DeepSeek API，数据全部保存在本地。

---

## 一、技术栈总览

| 层面 | 技术选型 | 说明 |
|---|---|---|
| 构建工具 | Rspack 2.x + swc + lightningcss | Rust 版 webpack，构建速度极快 |
| UI 框架 | React 18 | **零 UI 组件库依赖**，Button/Modal/Toast/Menu 全部自研 |
| 状态管理 | Zustand 4 | 4 个 store，**不用 persist 中间件，持久化手写** |
| 本地存储 | IndexedDB（idb 库）+ localStorage | 会话与附件 Blob 存 IndexedDB；设置与 API Key 存 localStorage |
| API 通信 | fetch 直连 Anthropic 兼容协议 | `/anthropic/v1` 前缀，**手写 SSE 流式解析**，无 SDK 依赖 |
| Markdown | mdast-util-from-markdown + micromark | 解析成 AST 树渲染，**不用 dangerouslySetInnerHTML** |
| 代码高亮 | Prism.js | 语言包按需动态 `import()` |
| 数学公式 | KaTeX（+ mhchem） | 单例按需加载，启动时预热 |
| 图表 | Mermaid | 全局串行渲染队列 + strict 安全模式 |
| 语音 | Web Audio API | AudioContext 采集 PCM + 手写 WAV 编码器；TTS 裸 PCM 流式播放 |
| 导入导出 | fflate | ZIP 流式读写，含 ZIP64 校验与路径穿越防护 |
| 工程化 | TypeScript strict + oxlint + oxfmt | Rust 工具链，强制具名导出、禁循环依赖、禁 any |

依赖极简：运行时依赖只有 React、Zustand、idb、clsx、fflate 和 Markdown/渲染相关的几个包，没有任何"大而全"的框架。

---

## 二、项目结构与模块职责

```
src/
  main.tsx                 入口：预热 KaTeX → setupTheme → 挂载 <App />
  app/                     应用壳：App.tsx（布局与副作用编排）、theme、viewport
  components/
    base/                  零依赖自研 UI 基础件（Button/Modal/Toast/Menu/Tooltip + 浮层定位 anchored.ts）
    messages/              消息列表、用户/助手消息、分支切换器、智能滚动
    markdown/              Markdown 组件层：AST 递归渲染、CodeBlock、MathFormula、MermaidDiagram
    chatInput/             输入区：草稿附件、语音输入、频谱动画、拖拽上传
    session/ sider/ settings/ search/ files/   会话视图、侧边栏、设置弹窗、搜索、文件卡片
  i18n/                    极简自研国际化（zh.ts / en.ts 两个同构字典）
  services/
    api/                   API 客户端、SSE 解析、协议类型、语音接口
    chat/                  会话数据层（最重模块）：store、分支树、IndexedDB、历史导入导出
    markdown/              Markdown 解析、高亮、KaTeX、Mermaid
    speech/                录音（recorder.ts）与朗读（readAloud.ts）
  utils/                   纯工具函数（id、format、clipboard）
```

分层思想非常清晰：

- **components 只做视图**，所有业务逻辑下沉到 services 层的**纯函数**（branches/reply/view 均可独立单测）；
- UI 开关类状态（弹窗、Toast、搜索面板）用**独立小 store**，不挂在业务 store 上，避免 props 层层传递；
- `App.tsx` 是唯一做全局副作用编排的地方。

---

## 三、核心机制详解

### 1. 状态管理：4 个 Zustand store + 双轨持久化

| Store | 位置 | 职责 | 持久化 |
|---|---|---|---|
| `useChatStore` | `src/services/chat/store.ts` | 会话列表、详情 LRU 缓存、当前会话、流式 AbortController | IndexedDB |
| `useSettings` | `src/services/settings.ts` | API Key、语言、主题、模型开关等 | localStorage（每项一个 key） |
| `useDrafts` | `src/services/chat/drafts.ts` | 每个会话的输入草稿 | 仅内存 |
| `useReadAloud` | `src/services/speech/readAloud.ts` | 全局朗读状态 | 仅内存 |

值得学的细节：

- **详情 LRU 缓存**（`SESSION_CACHE_LIMIT = 5`）：列表页只存 `SessionSummary`（3 个字段），打开详情才从 IndexedDB 加载，内存里最多留 5 个；
- **模块级 `Map/Set` 存运行时细节**：`sessionAccess`、`unsavedSessions`、`pendingFiles` 等"不进 React 状态"的东西放在 store.ts 模块作用域，避免 React 重渲染开销；
- **每个 localStorage 读写都 try/catch 兜底**：存储不可用时降级为"只在内存生效"，不打断使用。

### 2. 数据模型：带选中指针的消息分支树

定义在 `src/services/chat/types.ts`：

- `Session.messages` 是**扁平数组，包含所有分支的消息**；
- `rootMessageId` + 每条消息的 `parentId` / `selectedChildId` 构成一棵"带选中指针的树"；
- `AssistantMessage.blocks: ReplyBlock[]`（thinking/text/工具块），`status: pending | streaming | done | stopped | error`。

分支操作在 `src/services/chat/branches.ts`（纯函数）：

- `getVisibleMessages()`：沿 `selectedChildId` 走出当前可见路径；
- `appendBranch()`：新回答挂到父节点并自动切换选中，**旧版本保留**；
- **编辑提问、重新生成都是"新增同父版本"而非覆盖**——这就是聊天软件"消息版本切换"的实现原理；
- 版本列表未变时**复用旧对象**，避免流式更新时重绘历史消息。

### 3. API 层：手写 SSE 流式解析

`src/services/api/sse.ts`（约 230 行）+ `client.ts`：

- 不用 EventSource，用 **fetch + `response.body.getReader()`** 手写解析器；
- `ServerSentEventDecoder` 按 WHATWG 规范逐行累积，支持 CRLF/CR/LF、`:` keep-alive 注释、结尾悬 CR；
- `TextDecoder({ stream: true })` 防止多字节字符（如中文）跨网络分片被截断；
- async generator 模式逐事件产出，abort 时 `reader.cancel()` 释放连接；
- 自定义 `ApiError` 分 5 类（http/stream/network/interrupted/protocol），**错误信息本身可翻译**（i18n key 而非硬编码文案）。

端点全景：`POST /anthropic/v1/messages`（SSE 对话）、`/anthropic/v1/files`（附件）、`GET /models`（验证 Key）、`/v1/tools/asr|tts`（语音）。联网搜索和读链接是**服务端工具**（`deepseek_web_search`），无独立端点。

### 4. 流式 Markdown 渲染：AST 树 + 自适应节流

- 解析期"简化"：`[citation:N]` 替换为自定义 citation 节点，并用 `declare module 'mdast'` 扩展类型；
- `MarkdownAstRenderer.tsx` 纯递归组件渲染 AST，各节点 `memo` 化；
- **自适应节流**（`useThrottleMarkdownRender.ts`）：按上次渲染耗时 ×2（上限 2 秒）动态决定下次更新时间点——**渲染越卡，更新越稀**，简单两行逻辑解决流式输出卡顿；
- 代码高亮按需 `import()` 语言包（`highlight.ts` 的 `loaders` 表 + 依赖链处理如 tsx→jsx+ts）；
- Mermaid 用全局串行渲染队列避免并发覆盖配置，`securityLevel: 'strict'`。

### 5. 消息列表：智能跟随滚动

`MessageList.tsx`（无虚拟化，靠 memo + 版本缓存控制重绘）：

- sticky 滚动：距底部 40px 以内才跟随新内容，用户上滚立即停止跟随、回到底部自动恢复；
- `ResizeObserver` 驱动高度变化检测；
- 切换消息版本时用 `data-message-version` 锚点保持滚动位置。

### 6. 语音：手写 WAV 编码 + PCM 流播放

`src/services/speech/recorder.ts`：

- **不用 MediaRecorder**，而是 `AudioContext({ sampleRate: 16000 })` 采集原始 PCM（浏览器负责重采样）；
- `stop()` 时手写 WAV 编码器（44 字节头 + 16-bit 小端）；
- `AnalyserNode` 串入链路供频谱动画使用；iOS suspended 状态、逐节点容错释放均有注释。

`readAloud.ts`：TTS 返回**裸 PCM 字节流**（24kHz），边收边 `toFloat32` → `createBuffer` → 排程播放；网络慢于播放时**不补播过去时间**（`nextTime = Math.max(nextTime, currentTime + ...)`）。

### 7. 历史导入导出：两阶段事务

`src/services/chat/history/`：

- fflate 的 `Zip/ZipPassThrough` 实现 **STORE 流式写入**，带 ZIP64 哨兵校验、文件名 NFC 规范化和路径穿越校验；
- 导入用 **staging 暂存区**：全部数据先写入 `sessionsStaging/filesStaging`，校验全通过后才在一个事务里 `commitStaging` 原子替换正式数据——**校验失败不会污染现有数据**。

---

## 四、工程化亮点

### rspack.config.ts（93 行，值得整篇精读）

- `createConfig()` 工厂导出，支持自定义 `apiOrigin` 和 devServer 代理——**换 API 地址只改一个参数**；
- `DefinePlugin` 注入 `__API_ORIGIN__` 构建期常量；
- lightningcss 按 `targets: ['chrome>=105', 'safari>=16', ...]` 自动降级 CSS；
- `publicPath: 'auto'` + `scriptLoading: 'defer'` 支持 **file:// 协议直接打开产物**（双击 index.html 即可用）；
- `CopyRspackPlugin` 把 LICENSE 复制进 dist，版权合规意识；
- 分包不靠 splitChunks，靠动态 `import()`（Prism 语言包、Mermaid、历史归档模块）天然产生异步 chunk。

### TypeScript 与 Lint

- tsconfig 全 strict 家族（`noImplicitOverride` / `useUnknownInCatchVariables` / `noUnusedLocals` 等）+ `module: NodeNext` + 相对导入显式带 `.js` 后缀（运行时与打包双友好）；
- `.oxlintrc.json`：`import/no-default-export`（全项目强制具名导出）、`import/no-cycle`（禁循环依赖）、`no-explicit-any`、`no-non-null-assertion` 全开。

### i18n：极简自研

- `zh.ts` / `en.ts` 两个同构字典，`TranslationKey = keyof typeof zh` 让**缺 key 直接编译报错**；
- `useTranslate()` 组件订阅 + `translate()` 普通函数两种形态；
- `{param}` 占位符替换仅 5 行。不需要引入 i18next 这类重库。

### CSS 方案

CSS Modules（`*.module.css` + 手写 `.d.ts` 类型声明）+ `theme.css` 设计令牌 + `clsx` 拼类名。没有 Tailwind / styled-components，证明中小型项目纯 CSS Modules 完全够用。

### 错误处理（全项目最大亮点）

- `ApiError` 五类 kind 分类，错误信息用 i18n key；
- 存储失败不打断对话，只 console.error 并提供重试入口；
- 上传失败区分"可重试 / 不可重试"；
- 所有 store 的 localStorage 读写 try/catch 兜底。

---

## 五、初学者精读路线（按学习价值排序）

### 第 1 站：`src/services/api/sse.ts` + `client.ts`

手写 SSE 解码器是前端进阶最浓缩的一课：状态机、AsyncGenerator、ReadableStream、TextDecoder 流式解码、abort 清理，全部在 ~230 行内。**AI 时代前端开发者的看家本领。**

### 第 2 站：`src/services/chat/store.ts`（前 540 行）+ `branches.ts` + `types.ts`

学习"React 之外的状态层"怎么写：单 store + 模块级运行时 Map/Set、详情 LRU 缓存、原子 IndexedDB 事务、分支树数据结构。理解"编辑重发不丢历史"如何用数据结构实现。

### 第 3 站：`src/services/chat/reply.ts`

全是纯函数：流式事件 → 不可变数据合并 → 请求上下文构造的完整链路。最适合练 TS 类型推导的阅读能力，也是"逻辑与视图分离"的示范。

### 第 4 站：`rspack.config.ts` + `tsconfig.json` + `.oxlintrc.json`

真实项目的最小可读工程化样板，每行配置都有注释说明动机。

### 第 5 站：`useThrottleMarkdownRender.ts` + `MessageList.tsx`

两堂小而精的性能课：自适应节流与 sticky 滚动状态机。

### 其他值得看的细节

- `src/components/base/anchored.ts`：浮层定位怎么写（下拉菜单、Tooltip 的通用方案）；
- `src/services/markdown/highlight.ts`：动态 import + 依赖链解析（tsx 需要 jsx+ts 先加载）；
- `src/utils/id.ts`：`crypto.randomUUID` 的降级兜底；
- 全项目高密度中文注释，几乎每条非平凡逻辑都写了"为什么"，本身就极适合阅读。

---

## 六、二次开发改造方向

### 低成本练手（改常量 / 小文件即可）

| 改造 | 位置 | 说明 |
|---|---|---|
| 垂直领域 AI 助手 | `store.ts` 的 `MODEL` / `SYSTEM_PROMPT` | 加上行业系统提示词即成"XX 领域顾问" |
| 快捷指令 / 预设 Prompt | `ChatInput.tsx` + `i18n/zh.ts` / `en.ts` | 类型系统会强制补全两份语言 |
| 主题 / 气泡皮肤 | `app/theme.css` + 各 `*.module.css` | 纯 CSS，零风险 |
| 高亮语言 / Mermaid 主题 | `markdown/highlight.ts`、`mermaid.ts` | 改两张表即可 |

### 中等成本（有真实产品价值）

| 改造 | 位置 | 说明 |
|---|---|---|
| 多模型聚合客户端 | `api/client.ts` + `parseEvent.ts` | 抽象协议适配层，接入 OpenAI 兼容端点，做成"一个 Key 管多家"的本地客户端 |
| 会话全文搜索 | `SearchPanel.tsx` + `searchPanelState.ts` | 现在只搜标题，可遍历 IndexedDB 做内容检索，或接 minisearch 建本地索引 |
| Prompt 模板分享 | 复用 `chat/history/` 的 ZIP 机制 | 把系统提示词做成可导入导出的模板包 |
| TTS 音色自定义 | `api/speech.ts` + `readAloud.ts` | 改请求体与音色选择 UI |

### 高价值产品方向（组合多项能力）

1. **团队知识库聊天**：附件上传（Files API）+ 引用渲染（citation 节点）已实现，扩展为"上传团队文档 → 回答带引用"的内部工具。
2. **本地优先的 AI 笔记**：基于"消息树分支"这一独特数据模型，把对话改造成"可分支探索的思考笔记"——一个问题的多种回答就是思维的分支，这是其他聊天产品没有的特性。
3. **离线兜底 / 组织自部署版**：项目本就支持 `file://` 打开，打包成 Electron 或 PWA，利用 rspack 的 `devServer` 代理扩展点加一层服务端，规避用户各自管理 API Key 的麻烦。

### 建议的学习路径

先按第五节精读源码（尤其 SSE 和 store），然后从"低成本"表格里挑 1-2 个改造建立信心，再挑战"多模型聚合"或"AI 笔记"这类综合方向，做出能写进简历的作品。

---

## 七、逐文件功能详解

> 以下按目录逐个说明每个源文件的功能与实现要点。`*.module.css.d.ts` 为 CSS Module 的手写类型声明（无运行时逻辑，不单独列出）；各目录的 CSS 文件在末尾统一概述。

### 7.1 根目录与 app/

- `src/main.tsx`：应用入口；渲染前预载 KaTeX（`ensureKatex`）、同步主题（`setupTheme`），再挂载 `<App />` 并引入主题/全局样式。
- `src/env.d.ts`：全局类型声明；为 CSS Module 提供类名映射类型，声明构建注入的 `__API_ORIGIN__` 常量及 Prism/KaTeX 的副作用扩展模块。
- `src/index.html`：SPA 静态壳；仅含 viewport/theme-color meta 与 `#root` 挂载点，其余全部由 JS 接管。
- `src/app/App.tsx`：应用根组件；组装侧边栏/会话视图/欢迎页/搜索面板/设置弹窗/Toast，处理加载与保存失败重试、会话切换副作用（关面板、停朗读、收抽屉）与首次 `ensureApiKey` 校验。
- `src/app/theme.ts`：深浅色主题管理；用 `useSyncExternalStore` 订阅系统 `prefers-color-scheme`，手动偏好优先，切主题帧禁用过渡动画，同步 html 背景、colorScheme 与 meta theme-color。
- `src/app/viewport.ts`：移动端视口工具；`useIsMobile` 按 768px 断点，`usePreventPageScroll` 逐层探测可滚动容器后再拦截 touchmove，`useMobileViewport` 借 visualViewport 写 CSS 变量，键盘弹出时布局跟随且不干预双指缩放。
- CSS：`App.module.css` 定义外层布局与保存失败条，移动端用 visualViewport 变量定位；`theme.css` 是设计令牌大本营（字体、控件高度、全套明暗色板变量）；`global.css` 定义 body 基础样式、滚动条与消息列表宽度变量。

### 7.2 components/base/ — 零依赖自研 UI 基础件

- `Button.tsx`：通用按钮；variant/size/block 组合 CSS Module 类名，默认 `type="button"`，其余属性透传原生 button。
- `IconButton.tsx`：圆形图标按钮；三档尺寸（28/34/44px），tooltip 同时充当无障碍名（aria-label）。
- `Menu.tsx`：点击展开菜单；children 作 render-prop 暴露 open/toggle，浮层经 Portal 挂 body、anchored 定位，点外部/Esc 关闭。
- `Modal.tsx`：模态弹窗；遮罩+居中卡片经 Portal 渲染，**模块级 openModals 栈保证叠放时 Esc 只关最上层**，onClose 存 ref 避免监听器重建。
- `Select.tsx`：泛型下拉选择器；胶囊触发器+Portal 菜单，`captureEscape` 让 Esc 只关菜单不冒泡到外层弹窗，支持 placeholder。
- `Toast.tsx`：轻提示展示组件；订阅 toastState 单例，支持纯字符串或带 i18n key/参数/链接的结构化消息，用 `key=id` 强制重播入场动画。
- `Tooltip.tsx`：悬停提示；100ms 延迟出现、移动端与滚动时直接收起，触发层 `display: contents` 不占布局。
- `anchored.ts`：**浮层定位核心（手写版 floating-ui）**；`useAnchoredPosition` 在 Portal 挂载后用 fixed 坐标测量摆放，含翻面策略、maxHeight 收口、ResizeObserver+滚动跟踪；`useDismiss` 处理点外部/Esc 关闭；`useDelayedOpen` 管理悬停延迟切换。**初学者值得逐行精读。**
- `icons.tsx`：内联 SVG 图标库（约 58KB）；全部为无 props 箭头函数组件，`fill="currentColor"` 随文字色变化。
- `toastState.ts`：Toast 全局状态（微型 Zustand store）；showToast 替换旧消息并重置计时，普通 3s、带链接 5s 自动消失。

### 7.3 components/chatInput/ — 输入区

- `ChatInput.tsx`：输入框总成；草稿文字附件持久在外部 store，自适应高度靠隐藏 mirror 元素撑高，处理 Enter/IME、粘贴文件、拖拽、深度思考与搜索开关，语音录音时频谱接管首行，发送前用 store 最新快照校验并停朗读。
- `DraftAttachments.tsx`：附件条；每张卡片单独 `useUploads` 订阅自己的上传任务，图片显缩略图+失败蒙层（可重试显刷新图标），非图片回落 FileCard。
- `DragMask.tsx`：全页拖拽上传；`useFileDrop` 用 depth 计数解决 dragenter/leave 在子元素间反复触发的问题，DragMask 经 Portal 覆盖全屏展示上传提示。
- `Spectrum.tsx`：录音实时频谱；按容器宽度定柱数，100–4000Hz 人声频段对数分段映射 FFT bin、相邻 bin 插值平滑，起快落慢处理后**直接改 CSS transform，全程不触发 React 渲染**。
- `VoiceButton.tsx`：语音按钮；idle/录音/转写三态外观，呼吸动画与音量无关（固定节奏），非安全上下文（HTTP）置灰提示。
- `useVoiceInput.ts`：语音输入 hook；用 operationRef 代数让取消/迟到回调全部失效，处理权限/无设备错误，录满 120s 自动结束，转写可取消。

### 7.4 components/files/ 与 components/markdown/

- `FileCard.tsx`：文档文件卡片；按 status（上传中/完成/失败）切换图标与文案，失败可重试显刷新图标，点击行为随状态分流。
- `FileIcon.tsx`：按扩展名分发文件图标；共享折角 FileIconFrame，按类型换底色与图形，全部纯 SVG。
- `CodeBlock.tsx`：代码块；Prism 语言包按需异步加载、完成后用 useReducer 计数触发重渲染，标题栏含语言名/复制按钮，mermaid 代码块提供"图/代码"切换。
- `Link.tsx`：链接与图片渲染；`sanitizeUrl` 无协议补 https、只放行 http/https/mailto **防注入**，图片一律降级为链接。
- `Markdown.tsx`：Markdown 渲染入口；链路为节流（流式时）→ `toMdAst` 解析 → `MarkdownAstRenderer`，sources 存在时把 `[citation:N]` 转角标。
- `MarkdownAstRenderer.tsx`：递归 mdast AST 渲染器；块级/行内节点各一套 memo 组件分发，支持 GFM 任务列表、表格补齐空单元格，行内 HTML 仅放行 `<br>`。
- `MathFormula.tsx`：KaTeX 公式（行内/块级）；异步等待就绪后 renderToString，**出错时保留上次成功结果以度过流式中未写完的公式**。
- `MermaidDiagram.tsx`：mermaid 图渲染；流式期间每 100ms 合并绘制一次，失败时截到最后完整行重试，仅回答结束后才显示渲染错误，随主题切换重绘。
- `useThrottleMarkdownRender.ts`：流式渲染节流；按上次渲染耗时 ×2（上限 2s）动态计算下次允许更新时间，连续输入合并为最新内容。
- CSS：`markdown.css` 定义 `.ds-markdown` 全套排版；`highlight.css` 为 Prism 提供 亮暗两套配色。

### 7.5 components/messages/ — 消息列表与消息项

- `MessageList.tsx`：消息列表核心；stickyRef+阈值 40px 实现"贴底跟随、上滑即停"，ResizeObserver 监听内容增高续滚，**切换分支版本时按 `data-message-version` 锚点恢复滚动位置**，跨移动端断点切换滚动层保持阅读位置。
- `AssistantMessage.tsx`：助手消息（memo 化）；渲染思考/正文 Markdown，处理安全拒绝（丢弃已下发内容只显模板答复）、错误/欠费提示、联网搜索摘要入口，底部复制/朗读/重新生成/继续生成操作行。
- `UserMessage.tsx`：用户消息；含附件区与气泡，编辑态为自增高 textarea（上限 202px，处理 IME/Enter/Esc），保存前 ensureApiKey 校验。
- `BranchSwitcher.tsx`：消息多版本切换器（"◀ 计数 ▶"形态）；单版本不渲染；点击时记录当前 rect.top 供列表切换后把切换器钉回原位。
- `CopyButton.tsx`：复制按钮；成功后图标换对勾 1 秒且期间不重复复制，失败弹 toast，兼容 execCommand 兜底。
- `MessageFile.tsx`：消息附件渲染；`useFileUrl` 从 IndexedDB 读 Blob 生成 objectURL **并在卸载时 revoke**；图片区分上传中/失败可重试/普通态。
- `ReplyArea.tsx`：思考/工具步骤时间线；每秒驱动"已思考（用时 N 秒）"标题与搜索词 shimmer 轮播，StepItem 按 thinking/text/search/open/find/read_link 分派，搜索行可点击打开结果面板。
- `ReadAloudButton.tsx`：TTS 朗读按钮；从 readAloud store 选择当前消息状态，loading/playing 时换 ReadingBars 动画。
- `ReadingBars.tsx`：朗读动画竖条；playing 用 SMIL `<animate>` 驱动（begin 负值错峰出波），loading 用逐条扫光 keyframes；尊重 `prefers-reduced-motion`。
- `scrollContext.ts`：定义 ScrollToBottomContext 与 `useScrollToBottom`；子组件渲染更新后请求滚底，由列表依据贴底状态决定是否执行。
- `useReplyView.ts`：把 AssistantMessage 转为视图数据（步骤/正文/来源）的 hook；关键点是 **sources 数组的稳定性——逐字段比较仅在实际变化时 setState**，避免流式增量绕过 Markdown 节流。
- CSS：ReplyArea 的 shimmer 流光文字（`background-clip:text` + 位移动画）、AssistantMessage 的等待三点呼吸动画与 hover 显隐操作行；`thinking.markdown.css` 是思考区小字号排版，大量使用 `:has()` 选择器。

### 7.6 components/search/ — 联网搜索 UI

- `CitationLink.tsx`：正文 `[citation:N]` 数字角标；嵌进 HoverCard，点击新标签打开来源。
- `FallbackImage.tsx`：通用三态图片（loading 骨架/loaded/error 兜底）；只管行为不管视觉，类名由调用方注入，供站点图标复用。
- `HoverCard.tsx`：悬停卡片；复用 anchored 的 `useDelayedOpen`（enter 300ms/leave 100ms 防抖）与 `useAnchoredPosition`，卡片可被鼠标移入而不关闭。
- `SearchPanel.tsx`：右侧搜索结果面板；从 searchPanelState 读当前 entry/sources，渲染可跳转的 SourceCard 列表。
- `SiteLogo.tsx`：站点图标（基于 FallbackImage）；另导出 SiteThumbnails——前 4 个站点图标的重叠缩略图组。
- `SourceCard.tsx`：搜索结果卡片（站点名/日期/标题/摘要），整卡为外链；元数据缺失逐项省略。
- `searchPanelState.ts`：微型 store 管面板开合；同一入口再点收起、换入口替换内容，切换会话/消息时由列表关闭。

### 7.7 components/session/、settings/、sider/

- `SessionView.tsx`：会话页骨架；按 isMobile 选移动顶栏或桌面标题栏，MessageList 的 footer 传入吸底的 ChatInput 与免责声明。
- `MobileHeader.tsx`：移动端顶栏；左侧菜单按钮开抽屉，传入 title 时显示居中标题与"新对话"。
- `Welcome.tsx`：新对话欢迎页；问候语 + ChatInput（sessionId 为 null，**首次发送才建会话**）。
- `SettingsDialog.tsx`：设置弹窗主体；"通用/数据管理"两组（切换分组不丢输入状态），内含主题/语言/音色/位置/导入导出/清空历史，导入导出用 ref 防并发，导出用 objectURL+延时 revoke 触发下载。
- `ApiKeyField.tsx`：API Key 字段；已保存只显掩码星号+编辑按钮，空值失焦回退已保存态。
- `useApiKeyForm.ts`：Key 表单逻辑；提交先用 `GET /models` 验证通过才保存，错误经 classifyApiError 分类为"无效/网络/其他"。
- `settingsDialogState.ts`：微型 store 存弹窗 open/section；`ensureApiKey` 无 Key 时自动打开设置弹窗并返回 false。
- `Sider.tsx`：侧边栏；应用名+新对话按钮、按时间分组的会话列表（定时器午夜刷新分组）、底部设置入口；另导出移动端抽屉 MobileSider。
- `SessionItem.tsx`：单个会话条目（memo）；菜单提供重命名（非受控 input，`isComposing` 防中文输入法误触）与删除（父级弹确认）。
- `CollapsedSider.tsx`：侧栏收起后左上角的展开按钮组。
- `mobileSiderState.ts`：微型 store 管移动端抽屉开关状态。

### 7.8 i18n/ 与 utils/

- `i18n/index.ts`：极简 i18n 实现；zh 为键类型来源（`TranslationKey`），createTranslate 用正则替换 `{name}` 占位符；`useTranslate` 订阅 settings 的 locale，translate 供非组件上下文调用。
- `i18n/zh.ts` / `en.ts`：中英文文案表；en 的类型约束为 `Record<keyof typeof zh, string>`，**两语言键缺漏直接编译报错**。
- `utils/clipboard.ts`：异步复制；优先 `navigator.clipboard.writeText`，受限环境回退隐藏 textarea + `execCommand('copy')`。
- `utils/format.ts`：日期格式化；formatDate 输出 YYYY-MM-DD，monthKey 生成"年-月"键用于会话分组。
- `utils/id.ts`：createId；优先 `crypto.randomUUID`，降级为时间戳+随机数拼接。

### 7.9 services/api/ — API 通信层

- `sse.ts`：**手写 WHATWG 规范的 SSE 解码器**。字节流 → UTF-8（`stream: true` 防多字节跨分片截断）→ 按行累积字段、空行派发事件、忽略保活注释，边界处理细致（结尾 CR 延迟判断）；异步生成器在中止时主动 `reader.cancel()` 释放连接。**全项目最值得精读的文件。**
- `client.ts`：API 客户端。统一 `request`（网络错误归一化、非 2xx 解析错误体）、Files API 上传/删除附件、`streamMessage` 异步生成器流式请求；校验响应必须为 `text/event-stream`，收到 error 帧或流提前结束均抛错。
- `errors.ts`：错误分类器。按状态码归类（401/402/413/429/5xx）并区分可重试情况；按文案正则识别附件 file_id 失效（提取失效 id 供精准重置）与配额类错误。**分类与文案分离的设计值得学。**
- `parseEvent.ts`：流式事件手动校验层。对协议的每个事件做逐字段运行时类型校验，不认识或非法的事件静默丢弃——**防止服务端变更打崩 UI**。
- `speech.ts`：语音接口。ASR 用 multipart 上传 WAV/Ogg 一次返回全文；TTS 返回 24kHz 裸 PCM 字节流，导出 `TTS_SAMPLE_RATE` 供播放端对齐。
- `types.ts`：协议类型全集（内容块、请求参数、流式事件、Files 对象），注释详细记录协议约定。

### 7.10 services/chat/ — 聊天数据层（核心）

- `store.ts`：**核心 Zustand 状态层（约 900 行）**。会话列表与 LRU 详情缓存（上限 5，保护活跃/流式/未保存会话）、流式生成（按动画帧刷新增量、页面隐藏时直接落盘）、发送/重新生成/编辑/续写/版本切换、自动补标题、附件补传与失效重试、历史原子替换；大量并发安全处理（historyEpoch、waitForIdle）值得精读。
- `reply.ts`：回复的纯函数模型层。`applyEvent` 用临时索引映射把流式事件合并进回复块；`toRequestMessages` 组装上下文（**只取"提问+有回答"成对轮次、历史思考不回传、只保留 read_link 工具块并原样回传 signature**）。
- `branches.ts`：消息树分支管理。`getVisibleMessages` 沿选中路径取可见消息，`buildMessageVersions` 一次遍历收集兄弟版本并**复用旧对象避免流式重绘**，`appendBranch`/`selectVersion` 实现编辑重答与版本切换。
- `db.ts`：IndexedDB 持久层。保存/重命名/删除在同一事务内原子完成；删除会话时用 multiEntry `fileIds` 索引判断附件原件是否仍被引用；`commitStaging` 游标逐条复制实现导入原子替换，失败显式 abort 事务。
- `types.ts`：本地数据模型。消息状态机 pending→streaming→done/stopped/error、分支字段、Session；`FileAttachment` 的 id 为本地主键、serverId 为远端 file_id，是理解附件流转的关键。
- `backup.ts`：导入导出入口。Promise 链串行化避免事务交错；导出用惰性 `import()` 动态加载 history 模块；导入先预检再原子替换。
- `describe.ts`：回复提示推导。把错误（配 i18n 文案与跳转操作）、安全拒绝、服务端模板统一转为 Notice。
- `drafts.ts`：输入草稿管理（内存态，null 为新对话）；附件添加先整批校验再建预览 URL 并触发上传，**全部上传成功才能发送**。
- `latestReminder.ts`：生成"最新提醒"快照（日期、星期、位置、平台、语言）；日期用浏览器本地时区避免 UTC 跨日。
- `sessionGroups.ts`：会话列表按 updatedAt 分组：今天/昨天/7 天/30 天，超一个月按月份聚合。
- `view.ts`：回复视图纯计算层。把内容块推导为时间线步骤（相邻同类合并）、带角标的正文与去重编号的来源列表（同 URL 只编号一次）；**只读不改内容块以便原样回传**。

### 7.11 services/chat/history/ — 导入导出

> `README.md` 概述了导出格式 v3 的 ZIP 结构与导入替换流程，是导读本目录的最佳入口。

- `zipReader.ts`：**零依赖手写 ZIP 解析器**。从中央目录定位条目、逐条目流式解压并边算 CRC（yield 暂停等调用方异步写入，不整包缓存）；拒绝 ZIP64/加密/分卷/非 UTF-8 文件名。安全健壮性是亮点。
- `zipWriter.ts`：ZIP32 STORE 流式写出；写入前预检总大小/条目数/文件名编码是否触及 ZIP64 哨兵值，最终校验输出字节数与预期一致。
- `schema.ts`：手写校验器组合子（string/nullable/list/oneOf/time 等）；**"严格容器、宽容内容"策略**——容器与元数据严格校验，单个内容片段解析失败降级为丢弃而不影响整条消息。
- `format.ts`：导出产物类型定义（`export_format_version: 3`），13 种内容片段与状态枚举，与官方导出格式一一对应。
- `exportArchive.ts`：导出组装器；按 manifest → 会话 JSON → index → 图片顺序生成 ZIP 条目，惰性读取本地有原件的图片。
- `importArchive.ts`：流式导入器；manifest 必须首个、支持会话与图片交错，图片按字节头嗅探 MIME，最后校验 index 与已导入会话一一对应，失败保留原数据。
- `toArchive.ts`：本地消息 → 归档片段；联网轮次剥除角标、按父节点优先重编号保证 parent_id < id。
- `toLocal.ts`：归档 → 本地消息；按 id 排序重建消息树（校验父 id 更小、角色交替、根为提问）。
- `paths.ts`：ZIP 条目路径常量与生成规则；`safeName` 把分隔符与 `.`/`..` 映射掉并 NFC 规范化，读写两端共用。
- `errors.ts`：`BackupError` 携带 i18n key，分类精确到"版本过新/版本无效/不完整"等界面文案。

### 7.12 services/ 根目录与 markdown/、speech/

- `settings.ts`：本机设置 store；API Key、语言、主题、位置等，初始值跟随浏览器语言/系统主题，localStorage 不可用时静默降级为内存态。
- `attachments.ts`：**附件上传中枢**。扩展名/大小/数量校验、按 purpose 分流上传、3 并发任务队列、按本地 id 去重的任务表支持中止与重试。
- `uploadExts.ts`：允许上传的扩展名白名单（近千项）。
- `location.ts` / `locationData.ts`：地区选择模型与静态数据（省市联动，`Intl.DisplayNames` 取地区名）。
- `markdown/parse.ts`：基于 micromark/mdast 的解析；GFM + 数学扩展，`[citation:N]` 换成角标节点，**递归 simplify 把删除线/脚注降级为原文，渲染器只需处理简化后的 AST**。
- `markdown/highlight.ts`：Prism 按需加载注册表；常用语言内置、其余动态 import 并声明依赖顺序（如 cpp 依赖 c），别名表映射语言名。
- `markdown/katex.ts`：KaTeX 懒加载单例；并行加载 katex + mhchem + 样式表，加载失败不缓存可重试。
- `markdown/mermaid.ts`：Mermaid 懒加载与串行渲染队列；转义 img 标签防注入、strict 安全级别、Promise 队列避免全局配置互踩。
- `speech/recorder.ts`：录音转 WAV。AudioContext 以 16kHz 采集 PCM（浏览器重采样），串联 AnalyserNode 供频谱；stop 时手写 WAV 头，资源释放逐节点容错保证麦克风必被释放。
- `speech/readAloud.ts`：朗读管理。边收 TTS 裸 PCM 边转 Float32 排程播放（首帧加 0.1s 缓冲抗抖动、网络慢于播放时从当前时间续上不留空洞）；必须在点击事件中同步创建 AudioContext 以满足浏览器手势要求。

---

## 八、一句话总结

> 这个项目示范了"不依赖任何重型框架，如何用最朴素的 Web 原生能力（fetch、IndexedDB、Web Audio、CSS Modules）+ 清晰的分层架构，做出一个功能完整、健壮、可读性极高的现代 AI 应用"。对初学者而言，读懂它比读十个模板项目收获更大。
