# MCP (Model Context Protocol) 弃用争议深度调研报告

> **调研日期**: 2026年3月31日  
> **调研方式**: 多Agent并行深度调研 (4个独立研究Agent)  
> **调研范围**: 官方声明、社区讨论、技术分析、行业趋势、安全审计报告  
> **生成工具**: Claude Code Multi-Agent Teams

---

## 目录

1. [执行摘要](#1-执行摘要)
2. [MCP 是什么 — 技术架构概览](#2-mcp-是什么--技术架构概览)
3. ["MCP 要被弃用" 的说法从何而来](#3-mcp-要被弃用-的说法从何而来)
4. [五大根本原因深度分析](#4-五大根本原因深度分析)
5. [协议战争全景图：MCP vs A2A vs ACP](#5-协议战争全景图mcp-vs-a2a-vs-acp)
6. [社区声音：支持与反对](#6-社区声音支持与反对)
7. [真实采用数据与现状](#7-真实采用数据与现状)
8. [结论与前瞻](#8-结论与前瞻)
9. [参考来源](#9-参考来源)

---

## 1. 执行摘要

**核心结论：MCP 并未被官方弃用，但 "MCP 只是过渡产物" 的说法有其合理性。**

2025年底至2026年初，技术社区中出现了大量关于 MCP 前景的讨论。经过对官方声明、社区讨论、技术架构分析和行业趋势的全面调研，我们得出以下关键发现：

| 论断 | 真实性 | 说明 |
|------|--------|------|
| "MCP 已被弃用" | **错误** | Anthropic 已将 MCP 捐赠给 Linux 基金会 Agentic AI Foundation，这是制度化而非弃用 |
| "MCP 只是过渡产物" | **部分正确** | MCP 正在快速演进（SSE→Streamable HTTP），其核心模式可能被更高层抽象吸收 |
| "MCP 存在严重问题" | **正确** | 已发现 30+ CVE，82% 的服务器存在路径遍历漏洞，仅 8.5% 实现了 OAuth |
| "MCP 将被取代" | **短期不太可能** | 没有单一替代方案覆盖 MCP 的全部能力范围，替代方案更多是互补关系 |
| "MCP 对许多场景来说过度设计" | **正确** | 对于有 CLI 访问权限的个人开发者，MCP 增加了不必要的复杂性 |

---

## 2. MCP 是什么 — 技术架构概览

### 2.1 协议基础

MCP 构建于 **JSON-RPC 2.0** 之上，与 LSP (Language Server Protocol) 使用相同的请求-响应规范。通信分为三种类型：

- **请求 (Request)** — 双向消息，包含唯一 `id`、`method` 和可选 `params`
- **响应 (Response)** — 携带匹配的 `id`，包含 `result` 或 `error`
- **通知 (Notification)** — 单向发送，无 `id`，不期望响应

### 2.2 架构层次

```
┌─────────────────────────────────────────────┐
│  Host (宿主应用)                              │
│  Claude Desktop / Cursor / VS Code / IDE     │
├─────────────────────────────────────────────┤
│  Client (客户端)                              │
│  与单个 Server 维护 1:1 有状态会话              │
├─────────────────────────────────────────────┤
│  Server (服务器)                              │
│  暴露工具(Tools)、资源(Resources)、提示(Prompts) │
└─────────────────────────────────────────────┘
```

### 2.3 传输机制

| 传输方式 | 机制 | 适用场景 |
|---------|------|---------|
| **stdio** | 本地子进程的 stdin/stdout 管道 | 本地集成（文件系统、Git、数据库），零延迟 |
| **Streamable HTTP** (2025-03-26 规范) | HTTP POST + 可选 SSE | 远程服务器、多客户端、云部署 |
| **SSE (已弃用)** | Server-Sent Events over HTTP | 旧版实现，正在迁移中 |

> **关键澄清**: 被弃用的是 MCP 内部的 SSE 传输层，而非 MCP 协议本身。这是协议成熟化的标志。

### 2.4 协议原语

MCP 服务器暴露三种核心原语：

1. **Tools (工具)** — 模型可调用的函数（如 `search_database`、`create_file`）
2. **Resources (资源)** — 客户端可读取的数据端点（文件、数据库记录、API 响应）
3. **Prompts (提示)** — 可参数化的可复用提示模板

---

## 3. "MCP 要被弃用" 的说法从何而来

### 3.1 SSE 弃用引发的误解

2025年底，MCP 官方宣布将 SSE (Server-Sent Events) 传输方式替换为 **Streamable HTTP**。Atlassian 等公司随即发布了 SSE 弃用通知。这一消息被部分人误读为 "MCP 要被弃用了"。

**实际情况**: 这只是传输层的升级优化，Auth0 指出这一变更 "显著简化了安全模型"。

### 3.2 社区讨论中的质疑声浪

多个技术社区出现了对 MCP 前景的质疑：

- **Reddit r/programming** — ["MCP is dead. Long live the CLI"](https://www.reddit.com/r/programming/comments/1ri3kdv/mcp_is_dead_long_live_the_cli/)
- **Reddit r/ClaudeCode** — ["Will MCP be dead soon?"](https://www.reddit.com/r/ClaudeCode/comments/1rrl56g/will_mcp_be_dead_soon/)
- **Reddit r/ClaudeAI** — ["Is MCP already dead?"](https://www.reddit.com/r/ClaudeAI/comments/1s4s8fc/is_mcp_already_dead/)
- **Hacker News** — ["MCP is dead; long live MCP"](https://news.ycombinator.com/item?id=47380270)
- **Medium** — ["Why the MCP Standard Might Quietly Fade Away"](https://medium.com/@denisuraev/why-the-mcp-standard-might-quietly-fade-away-012097caaa85)

### 3.3 安全漏洞的持续曝光

2025-2026年间，MCP 生态系统遭遇了一系列严重安全事件，极大地动摇了社区信心。

### 3.4 竞争协议的涌现

Google 的 A2A、IBM 的 ACP 等竞争协议的出现，让人质疑 MCP 是否只是特定历史阶段的产物。

---

## 4. 五大根本原因深度分析

### 根本原因一：安全架构先天不足

这是 MCP 被质疑的**最核心原因**。MCP 在设计时优先考虑了开发者体验和快速采用，而安全模型严重滞后。

#### 漏洞统计数据（来自对 2,600-7,000+ 实现的审计）

| 指标 | 数据 | 来源 |
|------|------|------|
| 存在路径遍历漏洞 | **82%** | Endor Labs (2,614 servers) |
| 暴露代码注入 API | **67%** | 同上 |
| 使用静态 API 密钥/PAT | **53%** | Astrix Security (5,200+ servers) |
| 实现 OAuth 认证 | **仅 8.5%** | 同上 |
| 存在 SSRF 暴露 | **36.7%** | BlueRock Security (7,000+ servers) |
| 2026年1-2月 CVE 数量 | **30+** | 公开 CVE 数据库 |

#### 已记录的安全事件时间线

| 时间 | 事件 | 影响 |
|------|------|------|
| 2025.04 | WhatsApp MCP 工具投毒 | 整个聊天记录被窃取 |
| 2025.05 | GitHub MCP 提示注入 | 私有仓库内容+薪资数据泄露 |
| 2025.06 | Asana 跨租户漏洞 | 组织项目暴露给其他客户 |
| 2025.06 | Anthropic MCP Inspector RCE (CVE-2025-49596) | 开发工作站文件系统、API 密钥、环境变量暴露 |
| 2025.07 | `mcp-remote` 命令注入 (CVE-2025-6514, CVSS 9.6) | 影响 437,000+ 下载量，波及 Cloudflare、HuggingFace、Auth0 |
| 2025.08 | Anthropic 文件系统 MCP Server (CVE-2025-53109/53110) | 沙箱逃逸+符号链接绕过→主机任意文件访问 |
| 2025.09 | 恶意 Postmark MCP 包 | 所有邮件密送转发给攻击者 |
| 2025.10 | Smithery 路径遍历 | `~/.docker/config.json` 泄露，Fly.io token 暴露 |
| 2025.10 | Figma/Framelink 注入 (CVE-2025-53967) | `child_process.exec` 未做输入净化 |
| 2025.12 | Anthropic `mcp-server-git` (CVE-2025-68143/44/45) | 任意文件创建、覆盖、路径验证绕过 — 发现到修复长达6个月 |

#### 协议层面的安全设计缺陷

1. **认证是"推荐"而非"强制"** — 规范表述为 "SHOULD" 而非 "MUST"
2. **无消息签名或完整性验证** — 消息可被中间人拦截修改且无法检测
3. **Session ID 嵌入 URL** — 泄露到日志、浏览器历史、代理缓存
4. **工具描述拥有系统级提示权限** — 恶意工具描述可以覆盖代理行为
5. **动态工具重定义** — 服务器可在握手后更改工具名称和描述（"地毯式攻击"）
6. **无风险分层机制** — 无法区分无害的 `list_files` 和破坏性的 `delete_database`

> **Simon Willison 的安全分析核心观点**: "将工具与不受信任的指令混合使用在本质上就是危险的" — 这是整个行业的问题，不仅限于 MCP。

---

### 根本原因二：协议设计与现代基础设施的根本矛盾

#### 有状态会话 vs 无状态架构

MCP 的有状态会话模型（每个客户端与每个服务器维护持久会话）与企业已有的无状态 REST API 基础设施存在根本冲突：

- **负载均衡困难**: 会话状态绑定特定服务器实例
- **水平扩展受限**: 无法简单地通过增加节点来扩容
- **故障转移复杂**: 会话状态丢失意味着需要重新建立连接

#### Token 经济学的陷阱

| 维度 | 数据 |
|------|------|
| C/P (完成/提示) 比率 | 比一般 LLM 使用低 **2x-30x** |
| 完成占提示 token 比例 | 仅 0.18%-4.46% — 巨大输入消耗换取微小输出 |
| 提示大小增长 | 随启用工具数量**线性增长** |
| 平台工具并发上限 | ~40 个（防止上下文窗口耗尽） |
| 任务完成时间 | 32秒 (GPT-4o-mini) ~ 244秒 (DeepSeek R1 70B) |
| AWS Lambda 冷启动 | ~5秒，使无服务器部署不切实际 |

**核心矛盾**: 连接越多 MCP 服务器 → 工具描述越多 → token 成本越高 → 模型推理质量越差。**你越让代理有能力，它表现越差。**

#### 非确定性问题

LLM 是概率性的。相同的用户输入可能在不同运行中产生不同的工具调用序列。这从根本上打破了确定性 API 工作流中"相同请求产生相同响应"的预期。

---

### 根本原因三：MCP 解决的问题正在被超越

这是最具哲学深度的批评。核心论点：

> "如果 AI 足够聪明，能理解任意工具描述并编排复杂的多步骤工作流，那为什么它还需要一个专门的协议？为什么它不能直接阅读 API 文档并生成 HTTP 请求？"

#### 支持这一论点的证据

1. **LLM 已经能生成可用的 API 客户端代码** — 给定 OpenAPI 规范或文档，模型已经能产生正确的 `curl` 命令和 SDK 调用
2. **CLI 替代论** — HN 用户 tptacek 的核心观点：*"我们之所以有 MCP，是因为早期代理设计无法运行任意 CLI。一旦你能运行命令，MCP 就变得多余了。"*
3. **USB 类比的局限** — MCP 常被比作 USB（"AI 的通用连接器"）。但 USB 标准化的是电信号和物理接口——真正的硬问题。MCP 标准化的是 JSON 消息格式——LLM 已经能动态协商
4. **标准化的边际递减效应** — 随着模型理解任意 API 的能力提升，预标准化工具接口的价值在递减

#### 反对论点（为什么 MCP 仍然重要）

1. **运行时发现是真正的创新** — 此前没有方法让终端用户在运行时动态为 AI 应用添加能力
2. **LSP 先例** — 尽管 IDE 越来越能直接解析代码，LSP 仍然成功了。价值在于生态系统和工作去重
3. **预训练优势** — MCP 工具使用模式越来越多地被嵌入模型训练数据
4. **80/20 法则** — 标准协议降低了常见场景的错误率，原始 API 处理边缘情况

---

### 根本原因四：企业落地的鸿沟

#### 从桌面到企业的断裂

MCP 在开发者工具中爆发式增长（Claude Desktop、Cursor、VS Code），因为 stdio 传输 + 本地执行确实是零摩擦的。但这些模式**无法直接迁移到企业环境**：

- **服务器蔓延 (Server Sprawl)** — 不受控的 MCP 服务器增殖，无集中管理
- **缺乏信任注册中心** — 无可信的服务器注册表、包签名、认证发布者机制
- **观测性缺口** — 早期实现缺乏微服务所需的日志/监控能力
- **成本失控** — 代理循环或获取过多数据导致 "意外计费"

#### OAuth 2.1 规范的落地困境

| 问题 | 描述 |
|------|------|
| 动态客户端注册 (DCR) | 大多数企业拒绝"匿名 DCR"，但 MCP 客户端期望它 |
| 资源指示器 (RFC 8707) | 对 token 降权至关重要，但"大多数 IdP 今天不实现它" |
| 上游权限委托 | 无标准化模式让 MCP 服务器代表用户安全访问上游 API |
| Token 透传风险 | 必须防止盲目转发用户访问 token，否则代理可能"严重行为失常" |

#### 生产环境所需但 MCP 未提供的基础设施

- 服务发现/注册
- MCP 网关（路由和工具注册管理）
- 集中式日志和追踪
- 多层认证/授权
- 提示版本控制
- AI 行为集成测试框架
- 中止/熔断机制

---

### 根本原因五：协议竞争与生态碎片化

#### AI 代理协议栈全景图 (2026)

```
┌─────────────────────────────────────────────┐
│         商务层 (Commerce Layer)               │
│    ACP (开放) │ UCP (Google 生态)             │
├─────────────────────────────────────────────┤
│      代理协调层 (Agent Coordination)          │
│     A2A (Google + IBM/ACP 已合并)             │
├─────────────────────────────────────────────┤
│      工具访问层 (Tool Access)                 │
│          MCP (Anthropic → Linux Foundation)  │
├─────────────────────────────────────────────┤
│      AI 模型/代理运行时                        │
│   Claude │ GPT │ Gemini │ Llama │ etc.       │
└─────────────────────────────────────────────┘
```

#### 三大协议对比

| 维度 | MCP | A2A | ACP (已并入 A2A) |
|------|-----|-----|-----------------|
| **定位** | 代理 → 工具/数据 | 代理 → 代理 | 代理 → 代理 (简化版) |
| **发起方** | Anthropic | Google | IBM |
| **传输** | JSON-RPC 2.0, STDIO/HTTP | HTTP, SSE, JSON-RPC | REST/HTTP |
| **数据格式** | JSON 结构化 I/O | 文本、音频、视频、流 | REST 原生 |
| **发现机制** | 服务器 Schema | Agent Cards | 框架无关设计 |
| **治理** | Linux Foundation AAIF | Linux Foundation | 已合并进 A2A |
| **关键事件** | 2024.11 发布 → 2025.06 捐赠 LF | 2025.04 发布，50+ 合作伙伴 | 2025.09 合并进 A2A |

#### 关键类比

> *"MCP 是机械师使用诊断设备的方式。A2A 是客户与车间经理沟通的方式，或者经理与零件供应商协调的方式。"*

MCP 和 A2A 是**互补关系而非竞争关系**，但这也意味着 MCP 只是完整协议栈中的一层，而非终极解决方案。

---

## 5. 协议战争全景图：MCP vs A2A vs ACP

### 5.1 OpenAI 的关键选择

2025年3月，Sam Altman 公开为 MCP 背书，随后 OpenAI 在以下产品中集成了 MCP 支持：

- **Agents SDK** — 原生 MCP 客户端支持
- **Responses API** — MCP 工具连接
- **ChatGPT Developer Mode** — 自定义 MCP 连接器（读写操作）

这被广泛类比为苹果 1998 年 iMac G3 强推 USB 标准。OpenAI 的背书实际上终结了 MCP 能否成为跨厂商标准的疑问。

### 5.2 A2A + ACP 的合并

2025年9月，IBM 宣布其 ACP 正式合并到 Google 的 A2A 协议中，归入 Linux 基金会旗下。合并原因：

- 两个协议解决同一层面的问题（代理间通信）
- 单一组合协议网络比两个独立协议更有价值
- 合并减少了开发者的困惑和重复工作

### 5.3 Linux Foundation Agentic AI Foundation (AAIF)

MCP 和 A2A 现在都在 AAIF 旗下（146 个成员组织），拥有独立的治理结构。这是经典的标准战后配置，标志着从竞争碎片化向合作发展的成熟化转变。

### 5.4 替代方案对比

| 替代方案 | 类型 | 最佳适用场景 |
|---------|------|------------|
| OpenAI Function Calling | 专有 API 功能 | 简单、快速的 GPT 原型 |
| LangChain / LangGraph | 框架 | 复杂多步骤推理和编排 |
| Microsoft Semantic Kernel | SDK | .NET/Python 微软生态团队 |
| CLI/Shell 工具调用 | 直接执行 | 以开发者为中心的工作流 |
| OpenAPI/Swagger | API 规范 | 已有成熟 API 的系统 |
| Google A2A | 协议 | 代理间通信（与 MCP 互补） |

---

## 6. 社区声音：支持与反对

### 6.1 反对方核心论点

**"MCP 已死，CLI 万岁"派 (Reddit r/programming)**

> 核心观点：一旦 AI 代理可以运行任意 shell 命令，MCP 就成了不必要的中间层。CLI 更简单、更直接。

**"协议过度工程"派 (Hacker News)**

> troupo: "MCP 本质上就是加了 OAuth 的 JSON-RPC — 没有任何根本性创新。"
> hannasanarion: "为什么我们要为此发明一个全新的传输协议，而它唯一声明的目的只是文档化？"

**"Vibe-coded 协议"派 (技术博客)**

> Tom Bedor: MCP 主要解决一个已被解决的问题（序列化函数调用 schema），引入了不连贯的工具箱问题、不必要的运行时开销和安全隐患，其采用主要由非技术因素驱动（市场营销、投资者信心）。

**"MCP 就是个时尚"派 (Medium)**

> Denis Urayev: 随着 LLM 变得足够强大，能直接与 API 交互，MCP 将失去存在意义。

### 6.2 支持方核心论点

**"组织级标准化"派**

> CharlieDigital (HN): MCP 的价值在于一个实现可以覆盖多个客户端应用（Claude Desktop、VS Code、GitHub Actions），无需为每个写定制集成。

**"安全隔离"派**

> 支持者强调 MCP 远程服务器可以持有下游密钥，防止代理直接访问秘密——这是 CLI 方式无法提供的。

**"用户体验"派**

> 对非技术用户来说，MCP 无需安装，"大约 10 秒内完成认证，一切就能正常工作"。

**"不是死了，是在进化"派**

> LinkedIn 和 The New Stack 的分析认为 MCP 正在经历其从桌面工具到企业标准的痛苦蜕变。

### 6.3 社区共识

Hacker News 的共识并不是 MCP 已死，而是：

- **对于组织级跨平台标准化** → MCP 有价值
- **对于个人开发者或已有良好 CLI/文档的团队** → MCP 并非必需
- 真正的问题不是哪个方案客观更优，而是 **"你到底要解决什么问题？"**

---

## 7. 真实采用数据与现状

### 7.1 MCP 采用数据

尽管争议不断，采用数据讲述了另一个故事：

| 指标 | 数据 | 来源 |
|------|------|------|
| 月度 SDK 下载量 | **9,700 万** | Anthropic / npm 统计 |
| 活跃公开服务器 | **18,000+** | Anthropic 官方声明 |
| GitHub Stars | **37,000+** | GitHub |
| PulseMCP 注册表服务器 | **5,500+** | PulseMCP |
| 远程服务器增长 | 自 2025.05 增长 **~4x** | MCP Manager Stats |
| 月搜索量 (Top 20 服务器) | **180,000+** | Google Trends |
| 头部服务器: Playwright MCP | **35,000** 月搜索量 | 同上 |
| 头部服务器: Figma MCP | **23,000** 月搜索量 | 同上 |

### 7.2 官方支持状态

MCP 已获得以下主要公司/产品的官方支持：

- **AI 公司**: Anthropic (Claude)、OpenAI (ChatGPT/Agents SDK)、Google (Gemini)
- **开发工具**: Cursor、VS Code (GitHub Copilot)、JetBrains
- **企业**: Atlassian、Figma、Asana、Stripe、Cloudflare、AWS
- **治理**: Linux Foundation Agentic AI Foundation (146 成员组织)

### 7.3 Anthropic 的官方立场

Anthropic 于 2025 年中将 MCP 捐赠给 Linux Foundation，联合创始方包括 OpenAI 和 Block：

> *"MCP 的治理模型将保持不变：项目的维护者将继续优先考虑社区意见和透明的决策过程。"* — Anthropic 官方公告

这是**制度化而非弃用**的明确信号。

---

## 8. 结论与前瞻

### 8.1 为什么有人说 MCP 是过渡产物 — 根本原因总结

```
                     ┌─────────────────────────┐
                     │    "MCP 是过渡产物"       │
                     │      的五大根本原因        │
                     └────────────┬────────────┘
          ┌──────────┬───────────┼───────────┬──────────┐
          ▼          ▼           ▼           ▼          ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
    │  安全架构  │ │ 架构矛盾  │ │ 模型进化  │ │ 企业鸿沟  │ │ 协议竞争  │
    │  先天不足  │ │ 有状态vs  │ │ 将超越   │ │ 桌面到   │ │ 生态碎片  │
    │          │ │  无状态   │ │ MCP的需要 │ │ 企业断裂  │ │  化      │
    │ 30+ CVE  │ │ Token浪费 │ │ CLI替代  │ │ 缺基础设施│ │ A2A/ACP  │
    │ 82%漏洞  │ │ 扩展受限  │ │ OpenAPI  │ │ OAuth困境 │ │ 竞争&合并 │
    └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘
```

1. **安全架构先天不足** — 协议设计时安全是 "推荐" 而非 "强制"，导致 82% 的实现存在漏洞
2. **与现代基础设施的根本矛盾** — 有状态会话与无状态架构冲突，token 经济学存在结构性陷阱
3. **模型能力正在超越 MCP 的存在意义** — 随着 LLM 能直接理解 API 文档和运行 CLI，中间协议层价值递减
4. **桌面到企业的落地鸿沟** — 零摩擦的本地体验无法迁移到需要治理、审计、多租户的企业环境
5. **协议竞争引发的生态碎片化** — MCP 只是完整协议栈中的一层，不是终极方案

### 8.2 最可能的未来走向

**MCP 不会被"替代"，而是被"吸收"。**

最可能的结果是 MCP 的核心模式（标准化工具 schema、凭证隔离、客户端-服务器分离）将作为一等公民被嵌入到 LLM API 和代理框架中，使独立的协议层随时间变薄。协议本身可能作为兼容层持续存在，而真正的创新将转向上层的 A2A 协调层和商务层。

类比：正如 HTTP/1.0 和 SSL 2.0 需要多年安全补丁和规范修订才能成熟，MCP 正在经历每个成功协议都会经历的"痛苦青春期"。问题是 AI 生态系统的演进速度是否快到让 MCP 在成熟之前就被更根本的方法（利用模型日益增长的直接使用原始 API 的能力）所淘汰。

### 8.3 给开发者的建议

| 场景 | 建议 |
|------|------|
| 个人开发/原型 | 直接使用 CLI + Function Calling，无需 MCP |
| 多客户端集成 | 使用 MCP，一次实现多处复用 |
| 企业部署 | 使用 MCP + MCP Gateway + 严格安全策略 |
| 多代理协调 | 使用 MCP (工具层) + A2A (代理层) |
| 安全敏感场景 | 等待 MCP 安全模型成熟，或使用直接 API 调用 |

---

## 9. 参考来源

### 官方来源
- [Anthropic 官方：捐赠 MCP 并建立 Agentic AI Foundation](https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation)
- [Linux Foundation：AAIF 成立公告](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation)
- [MCP 规范 (2025-03-26)](https://modelcontextprotocol.io/specification/2025-03-26/basic)
- [MCP 传输层未来 — 官方博客](https://blog.modelcontextprotocol.io/posts/2025-12-19-mcp-transport-future/)
- [OpenAI MCP 和连接器文档](https://developers.openai.com/api/docs/guides/tools-connectors-mcp/)

### 安全分析
- [MCP 安全事件时间线 — AuthZed](https://authzed.com/blog/timeline-mcp-breaches)
- [MCP 安全：没人在发布前做安全的协议 — CloudTweaks](https://cloudtweaks.com/2026/03/mcp-security-the-protocol-nobody-secured-before-shipping/)
- [MCP 的六大致命缺陷 — Scalifi AI](https://www.scalifiai.com/blog/model-context-protocol-flaws-2025)
- [Simon Willison: MCP 提示注入分析](https://simonwillison.net/2025/Apr/9/mcp-prompt-injection/)
- [CVE-2025-6515: MCP 提示劫持攻击 — JFrog](https://jfrog.com/blog/mcp-prompt-hijacking-vulnerability/)
- [Anthropic MCP Inspector RCE — Oligo Security](https://www.oligo.security/blog/critical-rce-vulnerability-in-anthropic-mcp-inspector-cve-2025-49596)

### 技术分析
- [MCP 的一切问题 — Shrivu Shankar](https://blog.sshh.io/p/everything-wrong-with-mcp)
- [为什么 MCP 基本是扯淡 — Lycee AI](https://www.lycee.ai/blog/why-mcp-is-mostly-bullshit)
- [MCP 是时尚 — Tom Bedor](https://tombedor.dev/mcp-is-a-fad/)
- [MCP vs 传统 API 调用 — ByteBridge](https://bytebridge.medium.com/mcp-vs-traditional-api-calls-in-production-promises-pitfalls-and-proper-use-e0550c4b8065)
- [MCP 网络和系统性能表征 — arXiv](https://arxiv.org/html/2511.07426v1)

### 行业趋势
- [MCP vs A2A: 2026 协议战争如何结束 — Philipp Dubach](https://philippdubach.com/posts/mcp-vs-a2a-in-2026-how-the-ai-protocol-war-ends/)
- [AI 代理协议生态系统图谱 2026 — Digital Applied](https://www.digitalapplied.com/blog/ai-agent-protocol-ecosystem-map-2026-mcp-a2a-acp-ucp)
- [ACP 与 A2A 合并 — LF AI & Data Foundation](https://lfaidata.foundation/communityblog/2025/08/29/acp-joins-forces-with-a2a-under-the-linux-foundations-lf-ai-data/)
- [6 大 MCP 替代方案 — Merge](https://www.merge.dev/blog/model-context-protocol-alternatives)
- [OpenAI 为 ChatGPT 添加完整 MCP 支持 — InfoQ](https://www.infoq.com/news/2025/10/chat-gpt-mcp/)
- [MCP 采用统计数据 2025 — MCP Manager](https://mcpmanager.ai/blog/mcp-adoption-statistics/)

### 社区讨论
- [MCP is dead; long live MCP — Hacker News](https://news.ycombinator.com/item?id=47380270)
- [MCP is dead. Long live the CLI — Reddit r/programming](https://www.reddit.com/r/programming/comments/1ri3kdv/mcp_is_dead_long_live_the_cli/)
- [Will MCP be dead soon? — Reddit r/ClaudeCode](https://www.reddit.com/r/ClaudeCode/comments/1rrl56g/will_mcp_be_dead_soon/)
- [Is MCP already dead? — Reddit r/ClaudeAI](https://www.reddit.com/r/ClaudeAI/comments/1s4s8fc/is_mcp_already_dead/)
- [Why the MCP Standard Might Quietly Fade Away — Medium](https://medium.com/@denisuraev/why-the-mcp-standard-might-quietly-fade-away-012097caaa85)

---

> **免责声明**: 本报告基于公开信息和截至 2026年3月31日 的研究编写。AI 领域发展迅速，本报告的部分结论可能随时间推移而需要更新。本报告不构成任何投资或技术决策建议。
