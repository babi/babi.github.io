---
layout: post
title: Vercel AI SDK HarnessAgent：统一封装 Claude Code、Codex、Pi 等编程代理
subtitle: 一个 API 驱动所有 Agent——Vercel AI SDK 7 的 HarnessAgent 模块深度解析
date: 2026-07-27
tags: [AI, Vercel, AI SDK, HarnessAgent, Claude Code, Codex, Pi, Agent, TypeScript]
author: babi
---

## 引言

2026 年的 AI 开发生态中，**编程代理（Coding Agent）** 已经成为了开发者工作流中不可或缺的一部分。Claude Code、OpenAI Codex、Pi……每个代理都有自己的运行方式、CLI 接口和配置体系。当你想在应用中集成多个代理，或者在它们之间切换时，你面临的是一个碎片化的集成地狱。

Vercel AI SDK 7 引入的 **HarnessAgent** 模块正是为了解决这个问题——它提供了一个**统一的抽象层**，让你用一套 API 驱动不同的编程代理，无需关心底层的实现差异。

本文将深入介绍 HarnessAgent 的设计理念、核心 API、实战用法，以及它如何改变我们与编程代理的交互方式。

---

## 一、什么是 HarnessAgent？

**HarnessAgent** 是 `@ai-sdk/harness` 包的核心导出，位于 `@ai-sdk/harness/agent`。它是一个 AI SDK Agent 实现，提供标准的 `generate()` 和 `stream()` 方法，但背后由真正的编程代理（Harness）驱动执行。

直白地说：**HarnessAgent 是对现有编程代理的"适配器包装"**。它把 Claude Code、Codex 这些各自为政的 Agent，封装成了 AI SDK 统一接口下的组件。

```
┌─────────────────────────────────────────────┐
│          你的应用代码                        │
│  HarnessAgent.generate() / stream()         │
├─────────────────────────────────────────────┤
│          统一 API 层                        │
├──────────┬──────────┬──────────┬────────────┤
│Claude    │ Codex    │ Pi       │ OpenCode   │
│Code      │          │          │            │
│Harness   │ Harness  │ Harness  │ Harness    │
└──────────┴──────────┴──────────┴────────────┘
```

### 当前支持的 Harness 适配器

| 适配器 | 包名 | 说明 |
|--------|------|------|
| **Claude Code** | `@ai-sdk/harness-claude-code` | Anthropic 的终端编程助手 |
| **Codex** | `@ai-sdk/harness-codex` | OpenAI 的 Codex SDK |
| **Pi** | `@ai-sdk/harness-pi` | 在宿主进程运行的轻量代理 |
| **Deep Agents** | `@ai-sdk/harness-deepagents` | LangChain 的 deepagents 运行时 |
| **OpenCode** | `@ai-sdk/harness-opencode` | 开源的 OpenCode 服务器 |
| **ACP 通用适配器** | `acpx-ai-harness`（社区） | 支持任何 ACP 协议 Agent |

所有适配器都遵循统一的 `HarnessV1` 接口规范，这意味着**切换底层代理只需改动一行 import**。

---

## 二、核心 API 详解

### 安装

```bash
pnpm add @ai-sdk/harness @ai-sdk/harness-claude-code @ai-sdk/sandbox-vercel
```

### 创建 Agent 实例

```typescript
import { HarnessAgent } from '@ai-sdk/harness/agent';
import { claudeCode } from '@ai-sdk/harness-claude-code';
import { createVercelSandbox } from '@ai-sdk/sandbox-vercel';

const agent = new HarnessAgent({
  harness: claudeCode,    // ← 要封装的底层代理
  sandbox: createVercelSandbox({
    runtime: 'node24',
    ports: [4000],
  }),
  instructions: '你是一个谨慎的编程助手。在修改文件前先阅读并理解代码。',
  // 可选：注入自定义工具
  tools: {
    sendSlackMessage: tool({
      description: '发送 Slack 消息',
      parameters: z.object({ 
        channel: z.string(), 
        message: z.string() 
      }),
      execute: async ({ channel, message }) => {
        // 在宿主环境执行，而非代理内部
        return await postToSlack(channel, message);
      },
    }),
  },
  // 可选：注入自定义技能（可复用的指令包）
  skills: [
    {
      name: 'code-review',
      instructions: '请按以下维度审查代码：正确性、性能、安全性、可维护性...',
    },
  ],
});
```

**关键设计理念**：`HarnessAgent` 实例是配置对象，不是运行状态。它可以在模块作用域中全局创建，实际的运行状态归属于 `HarnessAgentSession`。

### 创建会话（Session）

所有交互都从 Session 开始。Session 持有代理的原生对话历史、运行时状态和沙箱连接。

```typescript
// 简单创建
const session = await agent.createSession();

// 带 ID 创建（用于跨 HTTP 请求恢复）
const session = await agent.createSession({ sessionId: chatId });
```

### generate() — 完整输出

适用于不需要流式处理的场景——批量任务、后台脚本、CI/CD 流水线：

```typescript
const session = await agent.createSession();
try {
  const result = await agent.generate({
    session,
    prompt: '分析这个仓库的 package.json，列出所有过时的依赖。',
  });
  console.log(result.text);
  // result 还包含 toolCalls、steps 等 AI SDK 标准字段
} finally {
  await session.destroy();
}
```

### stream() — 流式输出

适用于实时展示代理思考过程的场景——聊天 UI、终端 TUI、WebSocket 推送：

```typescript
const session = await agent.createSession();
try {
  const result = await agent.stream({
    session,
    prompt: '创建一个 TODO.md 文件，列出这个项目的优化点。',
  });

  for await (const part of result.fullStream) {
    switch (part.type) {
      case 'text-delta':
        process.stdout.write(part.text);
        break;
      case 'tool-call':
        console.log(`\n[工具调用] ${part.toolName}(${JSON.stringify(part.args)})`);
        break;
      case 'tool-result':
        console.log(`\n[工具结果] ${part.toolName} 完成`);
        break;
      case 'error':
        console.error(`\n[错误] ${part.error}`);
        break;
    }
  }
} finally {
  await session.destroy();
}
```

`result.stream` 提供简化的文本增量流，`result.fullStream` 则包含完整的粒度事件——文本增量、工具调用、工具结果、错误等。

---

## 三、会话生命周期管理

Session 的生命周期管理是 HarnessAgent 最强大的特性之一。每个 Session 都对应着一个真实的代理运行时实例，必须显式结束。

### 三种结束方式

| 方法 | 行为 | 适用场景 |
|------|------|---------|
| `session.destroy()` | 停止运行时，丢弃恢复能力 | 一次性脚本、测试 |
| `session.detach()` | 暂停运行时+沙箱（保持热状态），返回 resumeState | HTTP 多轮对话，短时间内恢复 |
| `session.stop()` | 保存 resumeState，然后停止运行时和沙箱 | 需要持久化恢复，无需热保持 |

### 多轮对话实战

```typescript
import { HarnessAgent } from '@ai-sdk/harness/agent';
import { claudeCode } from '@ai-sdk/harness-claude-code';

const agent = new HarnessAgent({
  harness: claudeCode,
  sandbox: createVercelSandbox({ runtime: 'node24' }),
});

// 第一轮：发送消息并 detach（保持沙箱热状态）
async function handleFirstMessage(chatId: string, userMessage: string) {
  const session = await agent.createSession({ sessionId: chatId });
  try {
    const result = await agent.stream({ session, prompt: userMessage });
    for await (const part of result.stream) {
      // 推送给前端...
    }
    // 暂停会话，保持沙箱热状态待命
    const resumeState = await session.detach();
    await db.saveResumeState(chatId, resumeState);
  } catch (error) {
    await session.destroy();
    throw error;
  }
}

// 第二轮：从 detach 状态恢复
async function handleSecondMessage(chatId: string, userMessage: string) {
  const resumeState = await db.loadResumeState(chatId);
  const session = await agent.createSession({ sessionId: chatId, resumeFrom: resumeState });
  try {
    const result = await agent.stream({ session, prompt: userMessage });
    // 处理结果...
    const newResumeState = await session.detach();
    await db.saveResumeState(chatId, newResumeState);
  } catch (error) {
    await session.destroy();
    throw error;
  }
}
```

### suspendTurn() — 跨进程的主动轮延续

`suspendTurn()` 是一个高级 API，用于在主动回合（active turn）中间跨进程边界延续执行。代理可能在执行过程中需要等待外部事件（如用户审批工具调用、等待 API 回调），`suspendTurn()` 可以在不丢失上下文的情况下暂停并稍后恢复。

---

## 四、沙箱：安全隔离的关键

HarnessAgent 的架构中，编程代理并非在宿主进程运行，而是在**隔离的沙箱**中执行。这是 Vercel AI SDK Agent Stack 的安全基石。

### 沙箱提供商

| 沙箱 | 包名 | 说明 |
|------|------|------|
| **Vercel Sandbox** | `@ai-sdk/sandbox-vercel` | Vercel 托管的微 VM，支持端口暴露、快照 |
| **Just Bash** | `@ai-sdk/sandbox-just-bash` | 纯 bash 沙箱，适用于宿主运行（Pi 等） |
| **Modal Sandbox** | `ai-sdk-modal-sandbox` | 社区适配的 Modal 云端沙箱 |
| **Coder Sandbox** | `@coder/ai-sdk-sandbox` | Coder 工作区沙箱 |

### 沙箱配置选项

```typescript
const agent = new HarnessAgent({
  harness: claudeCode,
  sandbox: createVercelSandbox({
    runtime: 'node24',              // 运行时镜像
    ports: [4000, 3000],            // 暴露的端口
    onBootstrap: async (ctx) => {   // 一次性初始化（可缓存）
      await ctx.execute('git clone https://github.com/example/repo.git');
      await ctx.execute('npm install');
    },
    bootstrapHash: 'v1',            // 快照缓存键
    onSession: async (ctx) => {     // 每次会话执行（含恢复）
      await ctx.env('API_KEY', process.env.API_KEY);
    },
    workDir: '/workspace/repo',     // 稳定工作目录
  }),
});
```

- **`onBootstrap`**：昂贵的初始化操作（安装依赖、克隆仓库）。配合 `bootstrapHash` 可缓存快照，后续会话直接复用。
- **`onSession`**：轻量级会话级初始化（设置环境变量、认证 Token）。每次新建或恢复会话都会执行。
- **`workDir`**：设置代理的工作目录，使多个回合可以共享文件系统状态。

---

## 五、一行代码切换底层代理

HarnessAgent 最大的卖点：**切换代理只需修改一个属性**。

### Claude Code → Codex

```typescript
import { HarnessAgent } from '@ai-sdk/harness/agent';
import { claudeCode } from '@ai-sdk/harness-claude-code';
// 只需将上面这行改为：
import { codex } from '@ai-sdk/harness-codex';

const agent = new HarnessAgent({
  harness: claudeCode,  // → 改为 codex
  // 其余代码不变...
  sandbox: createVercelSandbox({ runtime: 'node24' }),
});
```

### Claude Code → Pi

```typescript
import { pi } from '@ai-sdk/harness-pi';

const agent = new HarnessAgent({
  harness: pi,  // Pi 在宿主 Node.js 进程运行
  sandbox: createVercelSandbox({ runtime: 'node24' }),
});
```

Pi 的特殊之处在于它在**宿主 Node.js 进程**中运行，使用沙箱作为远程文件系统和 shell——这使得它与其他桥接式代理（Claude Code、Codex）有不同的架构取舍。

### 插入：社区通用适配器

除了官方适配器，社区还提供了 **acpx-ai-harness** 包。它基于 **ACP（Agent Communication Protocol）**，可以接入任何实现了 ACP 协议的 Agent——包括 Gemini CLI、Copilot、Cursor 等。

```typescript
import { createAcpxHarness } from 'acpx-ai-harness';

const agent = new HarnessAgent({
  harness: createAcpxHarness({ agentType: 'gemini' }),
  sandbox: createVercelSandbox({ runtime: 'node24' }),
});
```

这使得 HarnessAgent 的生态远超 Vercel 官方支持的范围，理论上可以接入任何 ACP 兼容的 Agent。

---

## 六、实际应用场景

### 场景 1：可切换 Agent 的代码审查服务

构建一个代码审查 AI 服务，可以在不同代理间切换：

```typescript
import { HarnessAgent } from '@ai-sdk/harness/agent';
import { claudeCode } from '@ai-sdk/harness-claude-code';
import { codex } from '@ai-sdk/harness-codex';
import { createVercelSandbox } from '@ai-sdk/sandbox-vercel';

type AgentType = 'claude' | 'codex' | 'pi';

const agents: Record<AgentType, HarnessAgent> = {
  claude: new HarnessAgent({
    harness: claudeCode,
    sandbox: createVercelSandbox({ runtime: 'node24' }),
    instructions: '你是代码审查专家。重点关注：安全漏洞、性能问题、代码风格。',
  }),
  codex: new HarnessAgent({
    harness: codex,
    sandbox: createVercelSandbox({ runtime: 'node24' }),
    instructions: '你是代码审查专家。重点关注：安全漏洞、性能问题、代码风格。',
  }),
  pi: new HarnessAgent({
    harness: pi,
    sandbox: createVercelSandbox({ runtime: 'node24' }),
    instructions: '你是代码审查专家。重点关注：安全漏洞、性能问题、代码风格。',
  }),
};

async function reviewCode(agentType: AgentType, repoPath: string) {
  const agent = agents[agentType];
  const session = await agent.createSession();
  try {
    const result = await agent.generate({
      session,
      prompt: `审查 ${repoPath} 的代码变更，输出结构化报告。`,
    });
    return result.text;
  } finally {
    await session.destroy();
  }
}
```

### 场景 2：沙箱隔离的问题复现与分析

Vercel Knowledge Base 中有一个典型示例——在隔离沙箱中复现 GitHub Issue：

```typescript
const agent = new HarnessAgent({
  harness: claudeCode,
  sandbox: createVercelSandbox({
    runtime: 'node24',
    onBootstrap: async (ctx) => {
      await ctx.execute('git clone https://github.com/user/repo.git');
      await ctx.execute('npm ci');
    },
  }),
});

const session = await agent.createSession();
const result = await agent.generate({
  session,
  prompt: `
    复现并分析这个 Issue：#1234
    - 在沙箱中运行测试，确认 bug
    - 检查相关源代码，找出根本原因
    - 提出修复方案
    - 输出结构化的 Issue 分析报告
  `,
});
```

### 场景 3：集成到 Terminal UI

`@ai-sdk/tui` 包可以将 HarnessAgent 集成到终端 TUI 界面中：

```typescript
import { HarnessAgent } from '@ai-sdk/harness/agent';
import { codex } from '@ai-sdk/harness-codex';
import { createVercelSandbox } from '@ai-sdk/sandbox-vercel';
import { runAgentTUI, createTUIAgent } from '@ai-sdk/tui';

const agent = new HarnessAgent({
  harness: codex,
  sandbox: createVercelSandbox({ runtime: 'node24' }),
});

const session = await agent.createSession();
try {
  await runAgentTUI({
    title: 'Codex Terminal',
    agent: createTUIAgent({ agent, session }),
    tools: 'auto-collapsed',
    reasoning: 'collapsed',
  });
} finally {
  await session.destroy();
}
```

---

## 七、与 AI SDK Workflow 的集成

`@ai-sdk/workflow-harness` 包提供了在 AI SDK Workflow 系统中运行 HarnessAgent 的实用工具。它支持：

- **时间分片（Time-boxed slices）**：限制每一步的执行时间
- **恢复持久化（Resume persistence）**：Workflow 步骤的自动状态保存
- **消息流式**：与 AI SDK UI 层的无缝集成

这使得你可以在一个持久化 Workflow 中包含 HarnessAgent 步骤——即使用户关闭浏览器、或者部署重启，代理的执行状态也不会丢失。

---

## 八、注意事项

### 实验性状态

所有 `@ai-sdk/harness-*` 包目前都处于 **pre-1.0 实验性阶段**。API 可能会在后续版本中发生破坏性变更。在投入生产环境前，请关注 changelog。

### 成本考量

每个 HarnessAgent 会话都对应一个真实的编程代理运行时。Claude Code 和 Codex 的运行成本不低，加上 Vercel Sandbox 的计算资源消耗，在生产环境中使用需要考虑成本控制。

### 沙箱依赖

桥接式代理（Claude Code、Codex、Deep Agents、OpenCode）需要网络沙箱支持（端口暴露、WebSocket 桥接）。如果你只需要在宿主进程运行，Pi 可能是更轻量的选择。

---

## 九、HarnessAgent 的定位思考

HarnessAgent 的出现反映了 Vercel 对 AI 开发工具的一个判断：**未来的编程工具不会是"一个模型搞定一切"，而是多个专业化 Agent 的协作生态。**

Claude Code 擅长理解复杂代码库，Codex 在代码补全和轻量任务上表现优秀，Pi 提供了更轻量、更快速的交互体验——它们各有所长。HarnessAgent 让你可以：

1. **为不同任务选择最合适的代理**
2. **在同一个应用中编排多个代理**
3. **随时切换到更好的新代理**（无需重构集成代码）
4. **在沙箱中获得安全隔离**

这与 Vercel AI SDK 的"平台化"哲学一脉相承——正如 AI SDK 提供了统一的 `generateText` 接口来屏蔽底层模型的差异，HarnessAgent 提供了统一的 `generate`/`stream` 接口来屏蔽底层 Agent 的差异。

---

## 结语

Vercel AI SDK 7 的 HarnessAgent 模块为编程代理的集成带来了一个重要的抽象层。它解决了当今 AI 开发生态中的一个真实痛点——**Agent 碎片化**。

当然，它目前仍处于实验阶段，配套的沙箱服务也是 Vercel 商业产品的一部分。但方向是明确的：随着编程代理成为开发者工具链的标准组件，我们需要标准化的方式来管理、集成和编排它们。HarnessAgent 是 Vercel 对这个问题的回答。

如果你已经在使用 Vercel AI SDK 构建 AI 应用，HarnessAgent 值得一试。一行代码切换底层代理的能力，不仅仅是一个炫技功能——它可能是未来 AI 应用架构的关键拼图。

---

*你在项目中使用过哪些编程代理？HarnessAgent 的统一抽象能否解决你的实际问题？欢迎在评论区分享你的看法。*
