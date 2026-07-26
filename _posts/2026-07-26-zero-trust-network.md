---
layout: post
title: 零信任网络：不再信任，始终验证
subtitle: 当边界消失之后，我们如何重新定义网络安全？
date: 2026-07-26
tags: [网络安全, 零信任, Cloudflare, OpenZiti, 架构]
author: babi
---

## 传统安全模型的困境

传统的网络安全模型是"城堡与护城河"模式：

- 你在城堡里（公司内网）——信任你
- 你在城堡外（互联网）——不信任你

这种模型假定所有**内部流量都是安全的**，只要通过了防火墙/VPN，就等于通过了安全检查。但这个假设在今天已经彻底崩塌了。

几个现实问题：

1. **边界已经消失了** — 员工用笔记本在咖啡店办公、应用跑在云端、SaaS 服务遍布各地。哪里是"内网"？哪里是"边界"？

2. **内部威胁真实存在** — 2023 年 MGM Resorts 被攻击，攻击者就是通过 SocEng 获取了一个内部员工的凭证，然后 VPN 进去横移，最终加密了整个数据中心。

3. **一旦被突破，内网就像不设防** — 传统模型下，一旦攻击者拿到内网入口，内网其他资源几乎裸奔。横向移动成本极低。

4. **VPN 本身就是攻击面** — VPN 网关暴露在公网上，本身就是攻击目标。而且 VPN 一旦建立连接，是所有端口都开放的。

这些问题的根源在于：**信任是隐式的**。你在内网，所以系统默认你没问题。

## 零信任的核心原则

零信任（Zero Trust）的概念最早由 Forrester 分析师 John Kindervag 在 2010 年提出。2014 年 Google 发布了 BeyondCorp 论文系列，证明了零信任在大规模生产环境中的可行性。2021 年，美国白宫发布了零信任安全战略行政令。

零信任的核心就一句话：**Never trust, always verify.**

展开来是几个基本原则：

### 1. 显式验证 —— 一切都要验证

每一次访问请求，不管来源是谁、位置在哪，都必须经过验证：

- **身份** — 你是谁（用户/服务）
- **设备** — 你用什么设备（设备健康度、合规性）
- **上下文** — 访问时间、地理位置、行为模式
- **授权** — 你有权限吗？

验证不是一次性的，而是**持续进行**的。一旦发现异常行为（比如从北京登录后 5 分钟又从纽约访问），立即重新验证或撤销访问。

### 2. 最小权限 —— 只给刚刚好的权限

每个用户、每个服务只获得完成其工作所需的最小权限：

- **Just-in-Time (JIT）** — 权限仅在被需要时授予，用完即收回
- **Just-Enough-Access (JEA）** — 给刚刚够的权限，不多一分

传统模型是"入职时给你一个角色的所有权限"，零信任是"每次请求时检查你是否需要这个权限"。

### 3. 假设被攻破 —— 以最坏情况做设计

这是一个心态上的根本转变：不再假设"我是安全的"，而是假设"我已经被攻破了"。

在这种假设下：

- 网络默认分段，每个 micro-segment 之间需要显式授权
- 所有流量都加密，包括内部的
- 所有访问都记录审计日志
- 权限不用时是撤销状态

### 4. 微隔离 —— 把你的网络切成最小的碎片

传统的内网"扁平"架构中，一个 Web 服务器和数据库服务器可以在同一网段自由通信。零信任要求：

- 应用层之间要验证（不是网络层）
- 每个 workload 有独立的身份
- 通信默认拒绝，除非显式允许

## 零信任的核心组件

一个完整的零信任架构通常包含这些组件：

| 组件 | 作用 | 代表实现 |
|---|---|---|
| **身份提供者 (IdP）** | 统一身份认证 | Okta、Azure AD、Keycloak |
| **设备管理 (MDM)** | 检查设备健康度 | Jamf、Workspace ONE |
| **策略决策点 (PDP)** | 计算是否允许访问 | OPA、Zanzibar 系统 |
| **策略执行点 (PEP)** | 执行访问决策 | Gateway、Agent |
| **安全网关** | 代理和验证流量 | Cloudflare Gateway、Zscaler |
| **审计与 SIEM** | 日志记录与告警 | Splunk、Elastic |

在实际部署中，这些组件通常集成在**零信任网络接入（ZTNA）** 平台中。

## 商业实现：Cloudflare Zero Trust

[Cloudflare Zero Trust](https://www.cloudflare.com/zero-trust/)（原名 Cloudflare for Teams）是目前最成熟的零信任商业产品之一。

### 架构概览

Cloudflare 的零信任方案核心组件：

1. **Cloudflare Gateway** — 安全 Web 网关，过滤所有对外流量
2. **Cloudflare Access** — 零信任应用网关，取代 VPN
3. **Cloudflare Tunnel** — 安全的出站连接，让服务不暴露公网 IP
4. **Cloudflare WARP** — 设备端客户端，把设备流量路由到 Cloudflare 网络

### 工作方式

```
用户 → WARP 客户端 → Cloudflare 边缘网络
                          ├── Gateway（DNS/HTTP 过滤）
                          ├── Access（身份验证 + 策略）
                          ├── Tunnel（连接到内部服务）
                          └── 服务（从未暴露公网 IP）
```

关键在 **Cloudflare Tunnel** — 内部运行一个 `cloudflared` 守护进程，主动建立**出站连接**到 Cloudflare 边缘网络。这样：

- 你的服务器不用开任何入站端口
- 没有公网 IP 暴露
- 攻击者扫描不到你的服务器
- 所有流量经过身份验证后才被允许

### 优点

- **全球网络** — Cloudflare 有 330+ 个边缘节点，延迟极低
- **天然 DDoS 防护** — 流量先经过 Cloudflare 清洗
- **丰富的策略引擎** — 支持设备 posture、地理位置、身份等多维度策略
- **部署简单** — 不需要额外硬件，DNS 接入即可开始
- **零信任成熟** — 产品经过大规模验证

### 缺点

- **价格** — 按用户收费，团队规模大时成本不低
- **厂商锁定** — 深度依赖 Cloudflare 生态
- **合规限制** — 数据需要经过 Cloudflare 网络，某些行业可能有合规问题
- **自建控制力有限** — 无法完全定制底层行为

### 适合谁？

中小企业快速落地零信任的最佳选择。15 分钟内可以完成配置，不需要云原生团队，不需要采购硬件。成本透明，按人头付费。

## 开源实现：OpenZiti

[OpenZiti](https://openziti.io/) 是目前最完整的开源零信任网络平台，由 NetFoundry 公司于 2020 年开源。它不是一个单一组件，而是一个完整的零信任网络架构。

### 架构概览

OpenZiti 的核心架构建立在一个**覆盖网络**（Overlay Network）之上：

```
                        控制器平面
                            │
                    ┌───────┴───────┐
                    │  Ziti Controller │
                    │  ∙ 身份与证书管理 │
                    │  ∙ 策略决策引擎  │
                    │  ∙ 端点编排     │
                    └───────────────┘
                            │
    ┌──────────────┐       │       ┌──────────────┐
    │  Ziti Edge    │◄──────┼──────►│  Ziti Edge    │
    │  Router       │       │       │  Router       │
    └──────┬───────┘              └──────┬───────┘
           │                              │
    ┌──────┴───────┐              ┌──────┴───────┐
    │ Ziti Tunnel   │              │ Ziti SDK     │
    │ (Client)      │              │ (App Embedded)│
    └───────────────┘              └───────────────┘
```

核心组件：

1. **Ziti Controller** — 控制平面，管理身份、证书、策略
2. **Ziti Edge Router** — 数据平面，转发加密流量
3. **Ziti Tunnel** — 客户端代理，拦截和处理流量
4. **Ziti SDK** — 可嵌入应用的 SDK，实现零信任网络直接嵌入应用
5. **Ziti Enrollee** — 端点注册和身份验证

### 工作原理

以场景为例：让我安全地 SSH 到一台内部服务器，但不暴露它的 IP。

传统方式：
```
你 ──SSH──► VPN 网关 ──► 内网 ──► 服务器（IP 暴露） ──► SSH 服务
```

OpenZiti 方式：
```
你 ──► Ziti Tunnel ──► Edge Router ──► 服务器 Ziti Tunnel ──► SSH 服务
      ↑              ↑               ↑
      身份验证        加密隧道        身份验证 + 授权
      （双向mTLS）   （端到端加密）     （双向mTLS）
```

关键差异：

- **没有入站端口** — 服务器只主动建立出站连接到 Edge Router
- **没有 IP 暴露** — 连接不上 Zigzag，只有经过认证的身份才能在 Overlay 上通信
- **双向认证** — 客户端和服务端都通过 X.509 证书互相验证
- **应用级身份** — SSH 连接对客户端来说是本地 localhost，实际流量通过 tunnel 加密传输到对端

从用户角度，他只需要 `ziti tunnel start`，SSH 的连接就像连接 localhost:

```bash
# 在服务器上
ziti edge login
ziti create config my-service
systemctl enable ziti-tunnel

# 在客户端上  
ziti tunnel start
ssh user@localhost -p 2222  # 实际连接到内网服务器
```

### OpenZiti 的核心特性

**1. 双向 mTLS 身份**

所有节点都有 X.509 证书，每次通信都做双向 mTLS 验证。没有人能伪造身份。

**2. 应用级零信任**

通过 Ziti SDK，零信任可以直接嵌入到你的应用中。比如你的 Web 应用可以直接集成 Ziti SDK，只接受经过零信任验证的请求，不需要额外的代理或网关。

```go
import "github.com/openziti/sdk-golang/ziti"

func main() {
    ctx := ziti.NewContext()
    listener, _ := ctx.Listen("my-service")
    // 现在只接受通过 Ziti 网络来的请求
    http.Serve(listener, myHandler)
}
```

**3. 私有 DNS 和服务发现**

OpenZiti 内置了分布式的服务发现和 DNS 系统。服务通过名称注册，客户端通过名称发现，不依赖公网 DNS。

**4. 策略引擎**

通过 Controller 可以定义详细的访问策略：

```
身份：工程师张 → 只能 SSH 到 prod-app-* 组
身份：CI/CD 流水线 → 只能访问 build-server:8080
设备：未打补丁的 → 不能访问任何生产资源
时间：凌晨 2-5 点 → 所有生产访问需要二次审批
```

**5. 端到端加密**

即使流量经过 Edge Router 转发，Router 也无法解密内容（只有两端持有私钥）。这就是零信任的"assume breached"原则——中间的转发节点被攻破了也不影响数据安全。

### 部署方案对比

| 部署方式 | 复杂度 | 控制力 | 推荐场景 |
|---|---|---|---|
| **Docker Compose** | 低 | 中 | 开发测试/小团队 |
| **Kubernetes Helm** | 中 | 高 | 生产环境 |
| **云托管 (NetFoundry)** | 低 | 低 | 不想自运维 |
| **裸机/VM** | 高 | 最高 | 需要完全自定义 |

最简单的快速开始（仅使用 Docker）：

```bash
git clone https://github.com/openziti/ziti.git
cd ziti/quickstart/docker

# 启动完整的零信任网络（含 Controller + Router)
docker compose up -d

# 创建一个服务
ziti edge create identity "my-device" -a "devices"
ziti edge create service "my-api" 
ziti edge create service-policy "allow-api" --service-roles "@my-api" --identity-roles "#devices"
```

### 优点

- **完全开源** — Apache 2.0 许可证，代码在 [GitHub](https://github.com/openziti) 上
- **架构完整** — 不是零信任的一个组件，是一整套零信任网络
- **SDK 嵌入** — 真正的应用级零信任，不是简单的代理
- **私有覆盖网络** — 端点之间建立自己的网络，保留网络控制权
- **不暴露任何端口** — 所有通信都是出站的（outbound-only）

### 缺点

- **学习曲线陡** — 概念多（Controller、Router、Tunnel、Edge...），新手容易迷失
- **运维成本高** — 需要维护 Controller 和 Edge Router 集群
- **文档生态不如商业产品** — 社区还在成长中
- **性能开销** — Overlay 网络增加了加密和封装层，相比裸 network 有 10-20% 的性能损失

### 适合谁？

有运维能力、需要完全控制零信任网络的技术团队。特别是对合规要求高、数据不能经过第三方平台的企业，以及想深入理解零信任底层原理的爱好者。

## 总结

零信任不是一个产品，而是一种**安全思维方式的转变**：

| 传统模型 | 零信任模型 |
|---|---|
| 信任基于位置（内网 = 信任） | 信任基于身份 + 上下文 |
| 一次验证，长期有效 | 持续验证 |
| 网络扁平，横向移动容易 | 微隔离，默认拒绝 |
| 边界防御 | 每个点都是安全边界 |
| VPN 暴露入口 | 出站-only 连接 |

回到具体的落地选择：

- **想快速落地、不操心运维、预算充足** → Cloudflare Zero Trust（或者 Zscaler、Okta 等商业方案）
- **需要完全控制、有运维能力、对合规敏感** → OpenZiti（自建或 NetFoundry 的托管版）
- **初创公司/个人项目** → 可以先从 Cloudflare Zero Trust 的免费版开始（一支团队内免费），或者自建 OpenZiti

----

*PSS：零信任落地一定要从实际痛点出发，别为了零信任而零信任。如果你的场景只是远程办公偶尔 SSH 到家里服务器，一个 WireGuard 隧道就够了。零信任是为需要精细控制、合规要求高的企业场景设计的。*
