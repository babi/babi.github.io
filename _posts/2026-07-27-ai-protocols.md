---
layout: post
title: AI 协议全景图：MCP、A2A、ACP、A2UI、JSON-Render、UCP——Agent 生态的"网络协议栈"
subtitle: 当 AI Agent 需要像互联网一样互联，我们需要怎样的协议？
date: 2026-07-27
tags: [AI, 协议, MCP, A2A, ACP, A2UI, JSON-Render, UCP, Agent, 架构]
author: babi
---

## 引言

2024 年底到 2026 年中，AI 领域发生了一场静默的基础设施革命：**Agent 通信协议的标准化**。

两年前，每个 AI Agent 还是一个孤岛——你为 Claude 写的工具集成不能在 ChatGPT 上复用，Google 的 Agent 无法和 Microsoft 的 Agent 对话，AI 生成的 UI 只能在特定的前端框架中渲染。每次集成都是 N×M 的适配工作。

今天，一个由 MCP、A2A、ACP、A2UI、JSON-Render、ANP、UCP 等协议组成的 **Agent 协议栈** 正在成型。它们的分工类似互联网协议栈：

| 层级 | 类比 TCP/IP | AI 协议 | 职责 |
|------|------------|---------|------|
| **工具接入** | HTTP ↔ 服务器 | **MCP** | Agent 如何调用工具和数据源 |
| **Agent 协作** | TCP ↔ 连接 | **A2A / ANP** | Agent 之间如何发现和协作 |
| **用户界面** | HTML/CSS ↔ 浏览器 | **A2UI / JSON-Render** | Agent 如何生成交互式 UI |
| **商业层** | HTTPS + 支付网关 | **UCP** | Agent 如何完成商业交易 |
| **编辑器集成** | LSP ↔ IDE | **ACP (Client)** | Agent 如何与编辑器通信 |

本文将逐一深入这些协议，梳理它们的定位、设计哲学和生态现状。

---

## 一、MCP（Model Context Protocol）—— Agent 的"USB-C 接口"

### 一句话定位

**Agent 连接工具和数据的标准化协议**。MCP 解决的是"每个 AI 模型对接每个数据源都需要定制集成"的问题。

### 背景

MCP 由 Anthropic 于 2024 年 11 月发布，是这场协议革命的开端。到 2026 年 3 月，**月 SDK 下载量达到 9700 万**（比发布时增长 970 倍，比 React 的 npm 增速还快），**17,000+ 公开 MCP 服务器**，**78% 的企业 AI 团队** 至少有一个基于 MCP 的生产 Agent。

### 核心概念

MCP 定义了三个原语（Primitive）：

- **Tools（工具）**：可调用的函数，有副作用（写数据库、发邮件、调用 API）
- **Resources（资源）**：只读数据，通过 URI 访问（文件、数据库记录、API 响应）
- **Prompts（提示模板）**：可复用的 prompt 模板

通信基于 **JSON-RPC 2.0**，传输层支持 stdio（本地进程）和 Streamable HTTP（远程）。

### 2026-07-28 重大版本更新（昨天发布！）

这是 MCP 发布以来最大的一次修订，核心变化包括：

**1. 无状态核心**
  - 移除了 `initialize` 握手和 `Mcp-Session-Id` 头
  - 协议版本、客户端信息、能力声明现在通过 `_meta` 字段随每个请求发送
  - **意义**：不再需要 sticky session，简单的 round-robin 负载均衡即可处理 MCP 请求

**2. 多轮交互（MRTR - Multi-Round-Trip Requests）**
  - 当服务器需要用户输入时，不再依赖长连接 SSE
  - 服务器返回 `InputRequiredResult`，客户端重新发起请求时带回用户输入

**3. 可路由头部**
  - `Mcp-Method` 头成为必选，网关无需解析请求体即可路由
  - `Mcp-Name` 对特定请求类型（`tools/call`、`resources/read`）成为必选

**4. 安全加固**
  - 与 OAuth 2.0 和 OpenID Connect 对齐
  - 客户端必须验证授权响应的 `iss` 参数（RFC 9207），防止 mix-up 攻击

**5. 扩展框架**
  - 扩展通过反向 DNS ID 标识，通过 `extensions` map 协商，独立版本管理
  - 两个官方扩展：**MCP Apps**（交互式 HTML 界面）和 **Tasks**（长时间运行任务）

### MCP Apps 扩展

2026 年 1 月发布的第一个官方扩展，让 MCP 工具可以返回交互式 HTML 界面（仪表盘、表单、可视化图表），在 Claude、ChatGPT、Goose、VS Code 中的沙箱 iframe 中渲染。由 Anthropic 和 OpenAI 联合开发。

### 安全挑战

MCP 的快速普及也带来了安全隐忧：
- **工具投毒（Tool Poisoning）**：恶意的工具描述可以隐藏指令，用户看不到但模型会执行
- **间接提示注入**：通过 `resources/read` 注入恶意文档到 Confluence、Notion 等知识库中，成为 2026 年 MCP 连接 Agent 的主要攻击向量

### 一句话总结

MCP 是当前最成熟、采用最广泛的 AI 协议。它完成了从 Anthropic 私有标准 → Linux Foundation 公共治理（2025 年 12 月捐赠给 AAIF）的转变，是 Agent 协议栈的基石。

---

## 二、A2A（Agent-to-Agent）—— 让 Agent 互相协作

### 一句话定位

**不同厂商、不同框架构建的 AI Agent 之间互相发现、委派任务、协调工作的标准协议**。

### 背景

Google 于 2025 年 4 月在 Google Cloud Next 上发布 A2A，首发即获得 50+ 合作伙伴支持。2025 年 6 月捐赠给 Linux Foundation。2025 年 8 月，IBM 的 ACP（Agent Communication Protocol）并入 A2A。2026 年 3 月发布 **v1.0 正式版**。截至 2026 年 4 月，超过 **150 家组织** 支持 A2A，GitHub 超过 22,000 Star。

### 核心概念

**Agent Card（Agent 名片）**
  - 每个 Agent 通过 `/.well-known/agent-card.json` 发布自己的"名片"
  - 包含身份、能力、约束、信任评分和接口说明
  - v1.0 支持 JWS（RFC 7515）加密签名验证

**任务生命周期**
  ```
  SUBMITTED → WORKING → INPUT_REQUIRED → COMPLETED
                                         → FAILED
                                         → CANCELED
                                         → REJECTED
  ```

**发现 + 执行二分法**
  - 先通过 Agent Card 发现对方的能力
  - 再通过 JSON-RPC 2.0 发送任务请求
  - 支持流式结果推送（SSE）、Webhook 通知

### 协议家族："A2Family"

A2A 所属的"协议家族"正在扩展：
- **A2A**（Agent-to-Agent）—— 核心的 Agent 间协作协议
- **A2UI**（Agent-to-User Interface）—— Agent 生成 UI 的协议
- **AP2**（Agent Payment Protocol）—— Agent 支付协议
- **UCP**（Universal Commerce Protocol）—— 通用商业协议

### MCP vs A2A 分工

| 协议 | 方向 | 类比 | 解决什么问题 |
|------|------|------|------------|
| **MCP** | Agent ↔ 工具/数据 | 垂直集成 | Agent 如何调用外部系统和数据 |
| **A2A** | Agent ↔ Agent | 水平集成 | Agent 之间如何协作完成任务 |

Google 的官方推荐是 **两者都用**：MCP 处理工具接入，A2A 处理 Agent 协作。

### 一句话总结

A2A 是 Agent 协作的事实标准。v1.0 的发布标志着它已具备企业级生产条件，150+ 家合作伙伴覆盖了从云平台到 SaaS 再到开源框架的整个生态。

---

## 三、ACP —— 同一个缩写，两个不同的协议

"ACP" 在 2026 年的 AI 协议版图中指代 **两个完全不同的协议**，分别服务于不同的场景。

### ACP v1：Agent Communication Protocol（已合并至 A2A）

IBM Research 于 2025 年 3 月为 BeeAI 平台推出的 Agent 间通信协议，采用四层架构（传输层、语义层、协商层、治理安全层），支持 Agent 发现、协商和递归委派。

**关键转折**：2025 年 8 月，IBM **正式将 ACP 合并到 Google 的 A2A 协议**，在 Linux Foundation 的 LF AI & Data 旗下统一治理。ACP 不再独立演进，其资产和经验全部贡献给 A2A v1.0。

### ACP v2：Agent Client Protocol（Zed Industries）

2025 年 8 月由 **Zed Industries** 创建，标准化 **AI 编程 Agent 与代码编辑器之间的通信**。类比 LSP（Language Server Protocol）之于代码补全，ACP 是 Agent 与编辑器的"协议桥梁"。

**核心特性**：
- 基于 JSON-RPC 2.0 over stdin/stdout
- 实时 Token 流式传输
- 内置会话管理（SQLite 持久化）
- 双向通信（Agent 可以向编辑器请求权限和文件）

**生态支持（截至 2026 年 3 月）**：
- **Agent 端**：Google Gemini CLI、GitHub Copilot CLI、Goose（Block）、Cline、OpenHands、Auggie 等 25+ 个 Agent
- **编辑器端**：Zed（原生支持）、JetBrains（集成中）、Neovim/Emacs（插件）、marimo
- Docker 通过 `docker agent serve acp` 直接暴露 Agent 的 ACP 接口

**不要混淆**：
- **MCP**：Agent 连接工具（Agent → Tool）
- **A2A**：Agent 连接 Agent（Agent → Agent）
- **ACP (Client)**：Agent 连接编辑器（Agent → IDE）

---

## 四、A2UI（Agent-to-User Interface）—— 让 Agent "说话" 带界面

### 一句话定位

**一种开放的、声明式的协议，让 AI Agent 能生成丰富的交互式 UI，并在 Web、移动端、桌面端原生渲染**。

### 背景

传统 AI 交互以文本为主：订餐要来回对话（"订 2 位"→"哪天？"→"明天"→"几点？"）。A2UI 的核心思想是：**与其让 Agent 用自然语言描述一个界面，不如让它直接生成这个界面**。

Google 于 2025 年 12 月发布 A2UI，它不是一个框架，而是一个 **协议**——定义了 LLM Agent 和客户端应用如何就 UI 进行通信。

### 核心架构

```
Agent（服务端）                    客户端（Web/Mobile/Desktop）
    │                                    │
    │  JSON Lines (JSONL) over SSE        │
    │─────────────────────────────────→   │
    │  surfaceUpdate                      │  组件注册表
    │  dataModelUpdate          ←──────── │  预批准的安全组件
    │  beginRendering                     │
    │                                    │
    │  ←─ userAction (click/submit) ────  │
```

**关键设计原则**：
- **声明式数据，非可执行代码**：Agent 只能从预批准的组件目录中请求组件，不能注入任意代码
- **数据绑定**：组件通过 JSON Pointer 引用数据模型，结构（组件树）与状态（数据）分离
- **渐进式渲染**：服务端流式发送 JSONL，客户端逐条处理，用户无需等待完整响应

### 安全模型

A2UI 最聪明的设计：**客户端维护一个受信任的预批准组件目录**。Agent 只能请求目录中的组件——就像 iOS 的沙箱机制，Agent 可以自由组合积木，但不能造新的积木。这从根本上杜绝了 UI 注入、XSS 和任意代码执行。

### 标准组件（18 个）

| 类别 | 组件 |
|------|------|
| 布局 | Column, Row, Card, List, Tabs, Modal, Divider |
| 内容 | Text, Image, Icon, Video, AudioPlayer |
| 交互 | Button, TextField, CheckBox, DateTimeInput, MultipleChoice, Slider |

### 集成生态

**渲染器**：`@a2ui/angular`、`@a2ui/react`、Flutter（通过 GenUI SDK）、Lit

**框架集成**：
- **A2A 协议**：A2UI 消息以 `application/json+a2ui` 的 DataPart 形式嵌入 A2A 通信
- **CopilotKit / AG-UI**：以 A2UI 作为生成式 UI 响应的数据格式
- **CrewAI / AG2 (AutoGen)**：内置 A2UI 扩展模块
- **MCP Apps**：补充方案——MCP Apps 走沙箱 iframe 路线，A2UI 走原生渲染路线

### A2UI vs JSON-Render vs MCP Apps

| 方案 | 来源 | 渲染方式 | 协议层级 | 适用场景 |
|------|------|---------|---------|---------|
| **A2UI** | A2Family | 原生组件渲染 | Agent → 客户端协议 | 多渠道原生 UI（Web/移动/桌面） |
| **JSON-Render** | Vercel | 原生组件渲染 | 框架层（AI SDK） | React/Vue 应用的 LLM 生成 UI |
| **MCP Apps** | MCP 扩展 | 沙箱 iframe | MCP 工具层 | Chat 内的交互式内容 |

### 一句话总结

A2UI 是 AI 生成 UI 领域最有野心的协议。它的安全模型（预批准组件目录 + 声明式数据）和跨平台能力，让它有望成为 AI 界面的"HTML/CSS"。

---

## 五、JSON-Render —— Vercel 的生成式 UI 框架

### 一句话定位

**一个专注于安全、可预测的 AI 生成 UI 框架，把 LLM 输出约束到开发者定义的组件目录中，流式渲染原生 UI**。

### 背景

JSON-Render 是 Vercel 实验室于 2026 年初开源的项目。它不是严格的"协议"而是一个 **框架 + 协议模式**，但与 A2UI 形成了有趣的对照。

### 工作流程

1. **定义目录（Catalog）**：开发者用 Zod Schema 声明 AI 允许使用的组件及其属性
2. **AI 生成 Spec**：LLM 输出一个扁平的 JSON spec（`root` + `elements` + `children` 引用），通过 **JSONL 补丁流（RFC 6902）** 增量传输
3. **安全渲染**：`<Renderer>` 组件将 spec 翻译为你真实的组件——React、Vue、Svelte、Angular 都可以

### 关键差异 vs A2UI

| 维度 | JSON-Render | A2UI |
|------|------------|------|
| 定位 | 框架（AI SDK 生态） | 协议（A2Family 标准） |
| 技术栈绑定 | 深度绑定 Vercel AI SDK | 框架无关 |
| 输出格式 | JSONL 补丁（RFC 6902） | JSON Lines 消息 |
| 生态 | React/Vue/Svelte/Angular | Angular/React/Flutter/Lit |
| 是否支持 MCP | 是（`@json-render/mcp`） | 通过 MCP Apps 互补 |

### 一句话总结

如果你已经在用 Vercel AI SDK，JSON-Render 是最顺滑的生成式 UI 方案。它的 JSONL 补丁流式传输在体验上非常出色。

---

## 六、ANP（Agent Network Protocol）—— 去中心化的 Agent 互联网

### 一句话定位

**面向开放互联网的去中心化 Agent 发现和通信协议**。目标是成为"Agent 互联网时代的 HTTP"。

### 背景

ANP 是社区驱动的协议，比 MCP 和 A2A 更年轻、更激进。它不依赖任何中心化注册中心或网关，让 Agent 像网站一样通过 DNS 和标准 Web 技术互相发现。

### 三层架构

| 层级 | 功能 | 技术基础 |
|------|------|---------|
| Layer 1 | 身份与加密 | W3C DID（`did:wba` 方法）、端到端加密 |
| Layer 2 | 元协议协商 | 动态协议选择、能力匹配 |
| Layer 3 | 应用协议 | Agent 描述、发现（搜索引擎式索引）、P2P 交易 |

### 标准化进展

ANP 正在 **W3C Community Group** 和 **IETF** 中讨论，IETF Internet-Draft "Framework for AI Agent Networks"（2025 年 10 月）由 ANP 社区、中国移动、中国电信、中国联通、华为等联合撰写。

### 一句话总结

ANP 是四个主流协议中最不成熟但愿景最大的一个。它解决的是 A2A 不覆盖的问题——没有中心化信任的、开放互联网级别的 Agent 互联。

---

## 七、UCP（Universal Commerce Protocol）—— Agent 购物的通用语言

### 一句话定位

**让 AI Agent 可以跨商家完成商品发现、加购、结账和订单追踪的开放协议**。

### 背景

UCP 由 Google 联合 Shopify、Etsy、Wayfair、Target、Walmart 等 20+ 家企业发布，得到 Adyen、American Express、Mastercard、Stripe、Visa 等支付巨头背书。它是 A2Family 的一员，专为 **Agentic Commerce（代理式购物）** 设计。

### 核心流程

```
Agent                               商家（Shopify/WooCommerce/...）
  │                                           │
  │  GET /.well-known/ucp                     │
  │─────────────────────────────────────────→ │  发现商家能力
  │←─────────────────────────────────────────│  返回 UCP Profile
  │                                           │
  │  Create Checkout Session                  │
  │─────────────────────────────────────────→ │  创建购物车
  │←─────────────────────────────────────────│  Checkout ID + 选项
  │                                           │
  │  Submit Payment                           │
  │─────────────────────────────────────────→ │  完成交易
  │←─────────────────────────────────────────│  订单确认 + 追踪信息
```

### 技术设计

- **无状态**：会话 token 编码在 checkout ID 中
- **多传输**：支持 REST、MCP、A2A
- **安全优先**：Tokenized 支付、可验证凭证、OAuth 2.0 身份绑定、AP2（Agent Payment Protocol）的用户授权加密证明

### 生态

- Shopify 支持 480 万+ 可发现商家
- WordPress/WooCommerce 插件让中小商家零改造接入
- Google AI Mode（搜索）、Gemini App 已集成 UCP 直接购买
- 开发工具：`@agorio/sdk`、`ucp-cli`、`ucp-mcp-server`

### 一句话总结

UCP 是 AI 协议栈中的"商业层"。如果 AI Agent 想为你买东西，UCP 就是它和商家之间的通用语言。

---

## 八、全景视图：一个 2026 年的 Agent 通信示例

用一个真实场景串起所有协议：

> **你让 AI Agent 订一款最新的 iPhone 手机壳**

```
你的请求
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│                   你的个人 Agent（前端）                         │
│  使用 A2UI 渲染交互界面（商品列表、颜色选择、结账按钮）         │
│  使用 JSON-Render 流式渲染组件（如果前端用 Vercel AI SDK）      │
└────────────┬─────────────────────────────────────────┬──────────┘
             │ A2A                                      │ A2UI
             ▼                                          ▼
┌──────────────────────────┐  ┌──────────────────────────────┐
│ 购物 Agent（云端）       │  │ 用户界面                     │
│ 使用 MCP 调用：          │  │ 原生渲染 A2UI/JSON-Render    │
│   ├─ 商品搜索 API        │  │ 安全组件目录 + 数据绑定      │
│   ├─ 价格比较工具        │  └──────────────────────────────┘
│   └─ 库存查询工具        │
│                          │
│ 使用 A2A 联系：          │
│   ├─ Shopify UCP Agent   │  ← 通过 UCP 完成交易
│   └─ 物流追踪 Agent       │  ← 通过 A2A 获取配送信息
│                          │
│ 使用 ACP (Client) 连接：  │
│   └─ 代码编辑器整合       │  ← 让 Agent 帮你写代码时
└──────────────────────────┘
```

每一个协议在链条中扮演特定角色，相互补充而非替代。

---

## 九、总结与展望

### 协议成熟度矩阵

| 协议 | 发布时间 | 治理机构 | 成熟度 | 核心贡献者 |
|------|---------|---------|--------|-----------|
| **MCP** | 2024-11 | Linux Foundation (AAIF) | ⭐⭐⭐⭐⭐ | Anthropic, OpenAI, 全行业 |
| **A2A** | 2025-04 | Linux Foundation | ⭐⭐⭐⭐ | Google, 150+ 组织 |
| **ACP (Client)** | 2025-08 | Zed Industries | ⭐⭐⭐ | Zed, 25+ Agent |
| **A2UI** | 2025-12 | A2Family / 社区 | ⭐⭐⭐ | Google, Angular/React/Flutter |
| **JSON-Render** | 2026-01 | Vercel | ⭐⭐⭐ | Vercel |
| **UCP** | 2026-03 | A2Family / 社区 | ⭐⭐½ | Google, Shopify, Stripe |
| **ANP** | 2025-10 | W3C / IETF 提案中 | ⭐⭐ | 社区 + 中国运营商 |

### 未来的趋势

1. **收敛还是分裂？** —— MCP 和 A2A 已经完成向 Linux Foundation 的捐赠，协议丛林正在向 2-3 个核心标准收敛。但 ACP (Client)、A2UI、JSON-Render 在各自垂直领域仍有不可替代的价值。

2. **安全是第一优先级** —— MCP 的 30+ CVE、工具投毒、间接提示注入等安全问题正在推动所有协议层增加安全机制。A2UI 的预批准组件目录、CP 的可验证凭证、MCP 的 OAuth 2.0 对齐都在朝这个方向努力。

3. **从协议到基础设施** —— 就像 HTTP 需要浏览器、TCP/IP 需要路由器，AI 协议也需要配套基础设施：MCP 服务器注册表、A2A Agent Card 搜索引擎、UCP 商家验证器。这些基础设施的完善程度将决定协议能否真正普及。

4. **A2Family 的扩张** —— UCP、AP2、A2UI 组成了围绕 A2A 的"协议星系"，Google 正在构建一个比单一 Agent 协作协议更大的生态。

### 开发者该关注什么？

- **如果你在构建 AI 应用**：MCP 是必选项，A2UI 或 JSON-Render 取决于你的前端技术栈
- **如果你在构建多 Agent 系统**：A2A 是协作的基准标准（ACP 的 Agent 间通信部分已并入 A2A）
- **如果你在做 IDE 插件**：关注 ACP (Client Protocol) 作为 Agent-Editor 通信标准
- **如果你在做电商**：UCP 是未来的 Agentic Commerce 基础设施
- **如果你关注去中心化**：ANP 还在早期，但愿景最大

---

## 参考资料

- [Model Context Protocol 官方文档](https://modelcontextprotocol.io/)
- [A2A Protocol v1.0 发布公告](https://developers.googleblog.com/en/how-a2a-is-building-a-world-of-collaborative-agents/)
- [A2UI 官方项目](https://github.com/a2ui-project/a2ui)
- [JSON-Render 开源项目](https://github.com/vercel-labs/json-render)
- [Universal Commerce Protocol](https://github.com/Universal-Commerce-Protocol/ucp)
- [ANP 协议规范](https://github.com/anp2dev/anp2)
- [ACP (Client Protocol)](https://github.com/agentclientprotocol/agent-client-protocol)
- [MCP 2026-07-28 版本更新](https://4sysops.com/archives/2026-07-28-model-context-protocol-mcp-stateless-multi-round-trip-routable-headers-authorization-hardening/)
- [Agent Protocols: A Complete Guide to MCP, A2A, and ACP](https://tyk.io/learning-center/agent-protocols-a-complete-guide-to-mcp-a2a-and-acp/)

---

*这篇文章是根据 2026 年 7 月的公开资料整理的。AI 协议生态发展极快，信息可能在你读到这篇文章时已经过时。欢迎在评论区指出错误或补充新协议。*
