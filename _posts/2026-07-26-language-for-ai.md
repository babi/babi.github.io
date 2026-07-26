---
layout: post
title: NL Is Not Enough：我们需要一种介于自然语言和代码之间的语言来规范 AI
subtitle: 当系统把缺少边界、约束和验证的自然语言直接当成可执行定义，混乱就开始了
date: 2026-07-26
tags: [AI, 编程语言, 架构, 系统设计, 思考]
author: babi
---

## 一个正在蔓延的趋势

打开任何一个 AI 平台、框架或工具，你都会看到同样的画面：

- **Skills** —— 用自然语言描述一个能力："当用户问天气时，调用天气 API 返回结果"
- **Sub-agents** —— 用自然语言定义角色："你是一个资深代码审查员，专注于安全性和性能"
- **System prompts** —— 用自然语言写几十页规则："回答要简洁，不要编造信息，如果不知道就说不知道"
- **Agent orchestration** —— 用自然语言描述工作流："先搜索资料，然后写提纲，再逐段撰写，最后润色"

自然语言正以前所未有的速度，变成**系统定义的默认语言**。

这个趋势本身并不可怕。让非技术背景的人用自然语言描述行为，这正是 AI 降低门槛的承诺。**真正可怕的是：这些自然语言描述，正在被系统直接当作可执行定义来对待。**

## 问题：自然语言不是代码

先看一个简单的例子：

```
技能：处理用户退款请求
当用户要求退款时，核实订单状态，如果符合退款条件，执行退款操作。
```

读起来很清晰，对吗？但这段文字在不同的人（或同一个 AI 在不同上下文中）执行时，可能产生完全不同的行为：

| 模糊点 | 可能的理解 A | 可能的理解 B |
|---|---|---|
| "核实订单状态" | 调用订单 API 检查状态字段 | 查询本地数据库缓存 |
| "符合退款条件" | 订单状态为"已支付"且未超过 30 天 | 订单状态为"已支付"且金额小于 1000 元 |
| "执行退款操作" | 调用支付网关退款接口 | 在系统里标记为已退款 |
| 边界条件 | 如果订单状态查询超时怎么办？ | 如果用户已经部分退款了怎么办？ |
| 错误处理 | 退款失败，重试 3 次 | 退款失败，报错让管理员处理 |
| 副作用 | 发送退款通知邮件 | 不通知，等用户自己查 |

**同一段自然语言描述，不同实现者会推导出完全不同的执行语义。** 这不是 bug，这是自然语言的天性——它依赖上下文、共识和常识来消歧，而这些恰恰是计算机系统没有的东西。

当你用自然语言定义一个 AI 行为，再让另一个 AI 去执行它，你实际上在做一场**语义传递的接力赛**：

```
写作者的意图 → 自然语言文本（信息已损失）→ 执行者的理解（再次损失）→ 实际行为
```

每一次传递都是信息熵增的过程。

## 问题不止于此

### 1. 没有边界和约束

自然语言描述天然缺少类型系统、值域约束和边界条件。一个简单的"处理用户请求"，展开来可能有 20 种不同的路径，但自然语言描述往往只覆盖最顺利的那一条 sunny path。

当异常发生时——参数非法、依赖不可用、结果超出预期——自然语言描述没有给出任何指引，AI 只能自己"猜"该怎么做。而不同 AI 猜的结果不同。

### 2. 没有可验证性

代码可以写单元测试、集成测试、属性测试。自然语言能怎么测？

你怎么为"回答要简洁"写一个测试？定义"简洁"的字数阈值？那么 AI 会不会在阈值边缘做文章："这段回答正好 100 字，满足要求"——而内容空洞无物？

### 3. 版本化和 diff 困难

Git 为代码提供了精细的版本追踪：每一行变更都能 review，每个 diff 都有意义。

而自然语言描述的 diff 常常是灾难性的——改一个词可能完全改变语义，但没有工具能帮你发现。把"必须"改成"可以"，整段逻辑就从强制变成了可选，而 review 时可能完全看不出来。

### 4. 组合爆炸

当你有 10 个 skill，每个都用自然语言定义，然后让它们协同工作——交互行为完全不可预测。Skill A 的"快速响应"和 Skill B 的"全面思考"同时触发时，谁优先？

在代码里，这是明确的优先级调度问题。在自然语言里，你只能靠"提示"和"希望"。

## 解是什么：一种中间语言

我们需要的东西，介于自然语言和通用编程语言之间。它不是用来替代自然语言（那是给人类读的），也不是用来替代代码（那是给机器读的），而是给 **AI Agent 读取和执行**的规范语言。

我给它的暂定名字叫做 **Constraint Definition Language（CDL）**，核心特征应该是：

### 1. 结构化而非自由文本

不再是写一段话描述"做什么"，而是用结构化字段定义：

```yaml
skill: process_refund
description: "处理用户退款请求"
triggers:
  - intent: "request_refund"
    confidence: >= 0.8
preconditions:
  - order.exists == true
  - order.status in ["paid", "partial_shipped"]
  - order.created_at > now - 30d
actions:
  - call: payment_gateway.refund
    with:
      order_id: "$order.id"
      amount: "$order.payable_amount"
    on_success:
      - update_order_status("refunded")
      - notify_user("refund_success")
    on_failure:
      max_retries: 3
      fallback: escalate_to_admin
limits:
  max_amount: 10000
  rate_limit: "10/min"
```

这不是 YAML，这是**一种结构化约束语言**。每一段都有明确的语义，可以被系统解析、验证和强制执行。

### 2. 可验证的合约

每个定义都附带可以自动化验证的合约：

- **类型检查**：`confidence: >= 0.8` —— 系统知道 `confidence` 是数值，`0.8` 是浮点数，`>=` 是合法操作符
- **边界验证**：`max_amount: 10000` —— 超出这个值的退款请求在定义层就被拒绝
- **状态机约束**：只能从特定状态转换到另一些状态，非法转换在编译期就能检测
- **副作用声明**：明确声明这个行为会调用哪些外部系统、修改哪些状态

### 3. 可测试性

因为定义是结构化的，工具可以自动生成测试用例：

- 给定 preconditions，测试每个 action 能否正确执行
- 给定边界值，验证 limits 是否生效
- 给定异常场景，验证 fallback 路径是否覆盖

这就像类型系统和契约测试给自然语言带来了**可计算性**。

### 4. 可组合的语义

当两个定义有交互时，系统应该能在定义层面检测冲突：

- 两个 skill 声明了同一个 trigger，但 with 不同的参数 —— 冲突
- Skill A 需要 `order.locked == false`，Skill B 设置 `order.locked = true` —— 冲突
- 两个 sub-agent 对同一个动作声明了不同的优先级 —— 需要显式裁决

在自然语言里，这些冲突要等到运行时才会暴露。在 CDL 里，它们可以在定义时就被发现。

## 这不是新想法——但我们有新的理由

编程语言的发展史本身就是一部"如何更精确地表达意图"的历史：

- 机器码 → 汇编（引入符号）
- 汇编 → C（引入类型和结构）
- C → Java/Rust（引入更丰富的类型系统和内存安全）
- 静态类型 → 类型推断（TypeScript、Rust——在保持安全性的同时减少冗余）

每一步都是**在表达力和安全性之间寻找更好的平衡点**。

现在，我们站在同一个分岔口：**自然语言是表达力最强但安全性最差的"编程语言"**。当 AI 开始把自然语言当作代码执行时，我们迫切需要给 AI 的那一侧加上类型系统、约束和验证——不是限制它的能力，而是确保它的行为可预测、可测试、可信任。

## 一个类比：HTML/CSS

想想如果没有 HTML 和 CSS，我们用自然语言描述网页是什么样子：

> "在页面的中间放一个蓝色的按钮，上面写着'提交'，点击后跳转到确认页面。"

这在 AI 生成 UI 的背景下正在发生——一些工具就是用自然语言描述来生成界面。但专业的前端开发不会这样做，他们用 JSX、Tailwind、TypeScript——**结构化的、可验证的、有约束的描述语言**。

AI Agent 的行为定义，比 UI 更需要结构化。因为 UI 错了最多是界面不好看，**AI 行为错了可能产生真实的业务损失**。

## 我理想中的 CDL 是什么样的？

以上面退款处理的完整 CDL 为例：

```yaml
version: "1.0"
namespace: "commerce.payment"

import:
  - "commerce.order"     # 订单模块的类型定义
  - "notification"        # 通知模块
  - "payment_gateway"     # 支付网关接口

types:
  RefundReason:
    - "customer_request"
    - "product_defect"
    - "shipping_issue"
    - "duplicate_charge"
  RefundStatus:
    - "pending"
    - "approved"
    - "rejected"
    - "completed"
    - "failed"

state_machine RefundStateMachine:
  initial: "pending"
  transitions:
    - from: "pending"     to: "approved"  guard: "approve"
    - from: "pending"     to: "rejected"  guard: "reject"
    - from: "approved"    to: "completed" guard: "execute_refund"
    - from: "approved"    to: "failed"    guard: "refund_error"
    - from: "failed"      to: "pending"   guard: "retry"

skill: process_refund
version: "1.2"
author: "babi"
description: "处理用户退款请求，从创建退款到完成退款的全流程"

triggers:
  - intent: "request_refund"
    confidence: >= 0.8
    context:
      user_type: ["customer", "admin"]
      channel: ["web", "app", "客服系统"]

preconditions:
  must:
    - order.exists == true
    - order.status in ["paid", "partial_shipped", "shipped"]
    - order.created_at > (now - duration "30d")
    - user.id == order.user_id
  should:
    - "检查用户是否已在 24 小时内发起过退款（同一订单）"

actions:
  - id: "validate_order"
    type: "query"
    target: "order_service.get_order"
    with:
      order_id: "$request.order_id"
    output: "order"
    on_error: return_error("ORDER_NOT_FOUND")

  - id: "process_payment"
    type: "command"
    target: "payment_gateway.refund"
    depends_on: ["validate_order"]
    with:
      order_id: "$order.id"
      amount: "$order.payable_amount"
      reason: "$request.reason"
    retry:
      max_attempts: 3
      backoff: "exponential"
      base_delay: "1s"
    timeout: "30s"
    on_success:
      - update_state("completed")
      - call: "notification.send"
        with:
          type: "refund_success"
          to: "$user.notification_channel"
          data:
            order_id: "$order.id"
            amount: "$order.payable_amount"
    on_partial_success:
      - update_state("partial_refunded")
      - call: "notification.send"
        with:
          type: "refund_partial"
          to: "$user.notification_channel"
    on_failure:
      - update_state("failed")
      - call: "notification.send"
        with:
          type: "refund_failed"
          to: "$user.notification_channel"
      - escalate:
          to: "admin_queue"
          priority: "high"
          reason: "refund_failed_after_retries"

limits:
  max_refund_amount: 100000  # 单笔上限（元）
  daily_volume: 100           # 每日处理上限
  rate_per_user: "5/hour"    # 单个用户频率
  working_hours_only: false  # 是否仅在营业时间执行

observability:
  metrics:
    - "refund.total_count"
    - "refund.success_rate"
    - "refund.avg_processing_time"
  logs:
    - level: "info"
      on: ["init", "success", "partial_success"]
    - level: "warn"
      on: ["retry", "timeout"]
    - level: "error"
      on: ["failure", "escalation"]

audit:
  required_fields: ["operator_id", "action", "reason", "amount", "result"]
  retention_days: 365
```

这个定义的核心价值不在于"更详细"，而在于**每个字段都有确定的语义**：

- `confidence: >= 0.8` —— 不是"差不多确信"，而是明确的数值比较
- `order.status in ["paid", "partial_shipped"]` —— 精确的集合检查
- `max_attempts: 3, backoff: "exponential"` —— 可测试的重试策略
- `depends_on: ["validate_order"]` —— 显式的依赖关系，可以画成 DAG

**这些是代码级别的精确性，但保持了自然语言级别的可读性**。

## 那自然语言扮演什么角色？

自然语言在 CDL 中退回到它最擅长的位置——**注释和说明**：

```
should:
  - "检查用户是否已在 24 小时内发起过退款（同一订单）"
```

这段描述不精确？没关系——它的作用不是被执行的，而是被人类阅读和理解**业务意图**的。精确的执行逻辑由结构化的 constraints 来保证。

这也是 CDL 的关键理念：
- **人类之间**用自然语言沟通
- **人类和 AI 之间**用 CDL 沟通
- **AI 和系统之间**用代码/API 沟通

每一层用最合适的语言。

## 这套语言需要什么基础设施？

光有语法不够，还需要配套的工具链：

1. **Linter / Validator** —— 检查定义是否合法、字段是否完整、类型是否匹配
2. **Conflict Detector** —— 检测多个 skill/sub-agent 定义之间的冲突
3. **Test Generator** —— 根据 preconditions 和 limits 自动生成测试用例
4. **运行时** —— 一个能解析 CDL 并严格执行的 runtime，不按自然语言"自由发挥"
5. **Visualizer** —— 把 CDL 渲染成状态机图、依赖图、数据流图

这些东西对自然语言是不可能的，但对结构化约束语言是可行的。

## 结语

自然语言不是不够好——它是太好了。好到让 AI 可以假装理解，然后做出和预期完全不同的事。

我们正在建造 AI 原生的系统，但在用最不 AI-friendly 的语言来定义它们。**自然语言是人类交流的终极工具，但不是系统定义的可靠基础。**

我们需要一种中间语言：
- **结构化**到可以被系统解析和验证
- **可读**到可以被人类理解和审查
- **精确**到不同 AI 执行同一段定义得到相同结果
- **可组合**到多个定义可以安全协同

这可能是 AI 工程的下一个基础设施——就像 SQL 之于数据、TypeScript 之于 JavaScript、Protocol Buffers 之于网络通信。

**当 AI 的"代码"是自然语言时，AI 的行为就是不可预测的。** 如果我们想让 AI 系统可靠、可测试、可信任，我们就需要在自然语言的灵活性和机器代码的精确性之间，找到那个 sweet spot。

---

*PS：文中 CDL 的语法是我的初步构想，希望能抛砖引玉。如果你有类似的想法，或者在实践中碰到过因自然语言模糊性导致的 AI 行为问题，欢迎留言讨论。*
