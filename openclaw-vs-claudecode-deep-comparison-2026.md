# OpenClaw vs Claude Code: 深度代码级对比研究报告

> **Author:** Zey413 (AARON ZHANG)
> **Date:** 2026-03-31
> **版本:** v1.0

## 摘要

本报告基于对 OpenClaw 和 Claude Code (CC) 两个项目源代码的深度分析，从工具系统、多Agent协作、权限安全、任务管理、配置系统五大维度进行代码级对比，识别核心架构差异，并提出可行的 PR 贡献方案。

---

## 1. 项目概览

| 维度 | OpenClaw | Claude Code (CC) |
|------|----------|-------------------|
| **定位** | 多通道 AI 网关 + 个人助手 | 终端 AI 编程助手 |
| **规模** | ~85 目录, 93 插件 | ~55 目录, 1900+ 文件, 512K+ LOC |
| **语言** | TypeScript/Node.js | TypeScript/Bun |
| **UI** | Web UI (Lit) + 原生 App | React + Ink (终端) |
| **运行时** | Node.js + Gateway (WebSocket :18789) | Bun (单进程) |
| **Agent 运行时** | Pi Agent (mariozechner/pi-agent-core) | 内置 QueryEngine + Tool Loop |
| **通道** | 20+ (WhatsApp/Telegram/Slack/Discord/Signal...) | 终端 + IDE Bridge |
| **模型支持** | 20+ 提供商 (Anthropic/OpenAI/DeepSeek/Ollama...) | Anthropic (Claude) 为主 |

---

## 2. 工具系统 (Tool System) 对比

### 2.1 CC 的工具架构

CC 使用 **强类型 `buildTool()` 工厂模式**，每个工具是一个 30+ 方法的丰富对象：

```typescript
// CC: src/Tool.ts — 核心 Tool 类型
export type Tool<Input, Output, P> = {
  name: string
  aliases?: string[]                     // 工具别名，向后兼容
  searchHint?: string                    // ToolSearch 关键词匹配
  inputSchema: Input                     // Zod 验证
  maxResultSizeChars: number             // 结果大小预算
  isReadOnly(input): boolean             // 只读标记
  isDestructive?(input): boolean         // 破坏性标记
  isConcurrencySafe(input): boolean      // 并发安全标记
  shouldDefer?: boolean                  // 延迟加载 (ToolSearch)
  interruptBehavior?(): 'cancel' | 'block'  // 中断策略
  checkPermissions(input, context)       // 每工具权限检查
  toAutoClassifierInput(input)           // 安全分类器输入
  renderGroupedToolUse?(...)             // 批量渲染
  // ...共 ~30 个方法
}

// buildTool() 填充安全默认值（fail-closed）:
export function buildTool<D>(def: D): BuiltTool<D> {
  return { ...TOOL_DEFAULTS, ...def }
  // TOOL_DEFAULTS: isReadOnly=false, isConcurrencySafe=false
}
```

工具注册通过 `getAllBaseTools()` 集中管理，支持 feature flag 门控：
```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool, BashTool, FileReadTool, ...,
    ...(isTodoV2Enabled() ? [TaskCreateTool, ...] : []),
    ...(isAgentSwarmsEnabled() ? [TeamCreateTool, ...] : []),
  ]
}
```

**关键特性：**
- **ToolSearch / 延迟加载**：`shouldDefer` 标记让工具按需加载，大幅减少 token 消耗
- **结果大小预算**：`maxResultSizeChars` 防止上下文窗口溢出
- **安全分类器集成**：`toAutoClassifierInput()` 支持 AI 自动分类
- **批量渲染**：`renderGroupedToolUse` 支持并行工具的统一展示

### 2.2 OpenClaw 的工具架构

OpenClaw 使用 **工厂函数模式**，基于 `@mariozechner/pi-agent-core` 的简单接口：

```typescript
// OpenClaw: src/agents/tools/common.ts
export type AnyAgentTool = AgentTool<any, unknown> & {
  ownerOnly?: boolean;
  displaySummary?: string;
};

// src/agents/openclaw-tools.ts
export function createOpenClawTools(options?): AnyAgentTool[] {
  return [
    createMessageTool(...),
    createSessionsSpawnTool(...),
    createWebSearchTool(...),
    createCronTool(...),
    // ...
  ];
}
```

**OpenClaw 独有特性：**
- **工具配置文件 (Tool Profile)**：`minimal`、`coding`、`messaging`、`full` 四级配置
- **每通道工具适配**：`channel-tools.ts` 根据消息通道动态调整工具集
- **Owner-only 门控**：`ownerOnly` 标记限制工具仅对认证用户开放
- **多层工具策略管线**：profile → provider → global → agent → group 五步过滤

### 2.3 关键差异总结

| 差异点 | CC | OpenClaw |
|--------|-----|----------|
| 工具构建模式 | `buildTool()` 工厂 + fail-closed 默认值 | 简单工厂函数，无统一默认值 |
| 延迟加载 | ✅ `shouldDefer` + ToolSearch | ❌ 全量加载 |
| 结果大小限制 | ✅ `maxResultSizeChars` | ❌ 无限制 |
| 中断策略 | ✅ per-tool `interruptBehavior` | ❌ 无 |
| 工具别名 | ✅ `aliases` | ❌ 无 |
| 安全分类器 | ✅ `toAutoClassifierInput` | ❌ 无 |
| 工具配置文件 | ❌ 无 | ✅ 4级 profile |
| 通道适配 | ❌ 终端单通道 | ✅ 20+ 通道 |
| Owner-only 门控 | ❌ | ✅ |

---

## 3. 多Agent协作系统对比

### 3.1 CC 的多Agent架构

CC 实现了 **三层多Agent体系**：

**a) AgentTool (子Agent)**
```
主Agent → AgentTool → 后台子Agent (独立 session)
                    → 可使用受限工具集 (ASYNC_AGENT_ALLOWED_TOOLS)
                    → 支持恢复 (resumeAgent) 和 fork (forkSubagent)
```

**b) Team/Swarm 系统**
```typescript
// TeamFile 结构
{
  name, description, createdAt,
  leadAgentId, leadSessionId,
  members: [{
    agentId, name, agentType, model,
    tmuxPaneId,                    // tmux 可视化管理
    cwd, subscriptions
  }]
}
```
- **邮箱消息机制**：`writeToMailbox()` 实现 Agent 间异步通信
- **结构化协议消息**：`shutdown_request`、`shutdown_response`、`plan_approval_response`
- **广播**：`to: "*"` 发送给所有队友
- **UDS 跨会话消息**：通过 Unix Domain Socket 实现点对点通信

**c) Coordinator 模式**
```typescript
// 主Agent 成为协调者，Workers 使用受限工具集
coordinatorMode.ts → 会话模式持久化
```

### 3.2 OpenClaw 的多Agent架构

OpenClaw 使用 **服务端子Agent系统**，基于 ACP (Agent Control Plane)：

```typescript
// ACP Spawn 参数
type SpawnAcpParams = {
  task: string;
  mode?: "run" | "session";     // 一次性 vs 持久会话
  sandbox?: "inherit" | "require";  // 沙箱继承
  streamTo?: "parent";           // 实时流输出到父级
  thread?: boolean;              // 线程绑定
}
```

**OpenClaw 独有特性：**
- **Session 作为多Agent基础**：spawn → send → list → yield → history
- **Yield 协作**：`sessions-yield-tool.ts` 让子Agent主动让出控制权
- **流式转发**：`acp-spawn-parent-stream.ts` 子到父实时输出
- **线程绑定**：自动将Agent绑定到消息线程
- **深度Agent配置**：每Agent独立配置 model、skills、workspace、identity

### 3.3 关键差异总结

| 差异点 | CC | OpenClaw |
|--------|-----|----------|
| Team 概念 | ✅ 完整 team 生命周期 | ❌ 仅有 session |
| 邮箱消息 | ✅ 文件基础邮箱 | ❌ 走 gateway |
| 结构化协议 | ✅ shutdown/plan_approval | ❌ 无类型化协议 |
| Plan 审批 | ✅ 执行前审查 | ❌ 无 |
| Coordinator 模式 | ✅ 专用协调模式 | ❌ 依赖工具过滤 |
| Agent 颜色管理 | ✅ 视觉区分 | ❌ 无 |
| Yield 协作 | ❌ | ✅ 协作式多任务 |
| 流式转发 | ❌ | ✅ 子到父实时流 |
| 线程绑定 | ❌ | ✅ 消息线程自动绑定 |
| Session 模式 | ❌ | ✅ run vs session |

---

## 4. 权限安全系统对比

### 4.1 CC 的权限模型

CC 实现了 **多层权限系统**：

```
Permission Check Flow:
1. PermissionMode 判定 (default/auto/plan/bypass)
2. ToolPermissionRules 匹配 (allow/deny/ask patterns)
3. AI Classifier 自动审批 (bashClassifier, yoloClassifier)
4. 交互式用户审批 (React 终端 UI)
5. Denial Tracking (追踪连续拒绝)
```

**独特模式：**
- **Auto 模式**：AI 分类器自动判断操作安全性
- **Plan 模式**：显示计划后才执行
- **Permission Rule Patterns**：`Bash(git *)` 语法支持细粒度权限
- **Shadowed Rule Detection**：检测冲突规则
- **PreToolUse/PostToolUse Hooks**：用户可配置的外部脚本钩子

### 4.2 OpenClaw 的安全模型

OpenClaw 实现了 **策略驱动安全系统**：

```
Security Pipeline:
1. 多层工具策略管线 (profile → provider → global → agent → group)
2. Owner-only 工具门控
3. 全面安全审计 (security/audit.ts, 57KB!)
4. 每通道安全检查
5. 外部内容清洗
6. SSRF 防护
7. 安全自动修复
```

**独特能力：**
- **57KB 安全审计系统**：严重级别 + 发现 + 修复建议
- **自动修复**：`security/fix.ts` 自动解决安全问题
- **ReDoS 防护**：`safe-regex.ts` 正则安全验证
- **Per-channel 安全**：每通道独立安全策略

### 4.3 关键差异总结

| 差异点 | CC | OpenClaw |
|--------|-----|----------|
| 交互式审批 | ✅ 终端 UI 审批 | ❌ 服务端无交互 |
| AI 自动审批 | ✅ bashClassifier/yoloClassifier | ❌ 无 |
| 权限规则语法 | ✅ `Bash(git *)` | ❌ 仅 allow/deny 列表 |
| 外部 Hooks | ✅ PreToolUse/PostToolUse | ❌ 仅内部 Hooks |
| 安全审计 | ❌ 无系统性审计 | ✅ 57KB 全面审计 |
| 安全自动修复 | ❌ | ✅ auto-fix |
| SSRF 防护 | ❌ | ✅ |
| ReDoS 防护 | ❌ | ✅ |
| 通道安全 | N/A | ✅ |

---

## 5. 任务管理系统对比

### 5.1 CC 的双重任务系统

**a) 后台任务 (Infrastructure)**
```typescript
TaskType = "local_bash" | "local_agent" | "remote_agent" 
         | "in_process_teammate" | "local_workflow" | "monitor_mcp" | "dream"
```

**b) LLM 可见任务列表 (Planning)**
```typescript
// TaskCreate/Get/Update/List — LLM 直接使用的任务管理
Task = {
  subject, description, activeForm,  // 显示
  status: "pending" → "in_progress" → "completed",  // 状态
  blocks: TaskId[], blockedBy: TaskId[],  // 依赖图
  metadata: Record<string, any>,  // 自定义元数据
  owner: string  // Agent 归属
}
```

**独特能力：**
- **依赖图**：`blocks/blockedBy` 实现任务间依赖
- **Dream Tasks**：后台投机性工作
- **Team 级作用域**：每 team 独立任务列表
- **进度旋转器**：`activeForm` 在 UI 显示实时进度

### 5.2 OpenClaw 的任务注册表

```typescript
// SQLite 持久化任务存储
TaskRecord = {
  taskId, runtime: "subagent" | "acp" | "cli" | "cron",
  status: "queued" | "running" | "succeeded" | "failed" | "timed_out" | "cancelled" | "lost",
  deliveryStatus: "pending" | "delivered" | "session_queued" | "failed",
  notifyPolicy: "done_only" | "state_changes" | "silent",
  // + 流程关联、审计、清理等
}
```

**独特能力：**
- **SQLite 持久化**：CC 使用 JSON 文件
- **投递状态追踪**：确保结果投递到请求者
- **通知策略**：细粒度控制
- **流程注册表**：多任务流编排
- **任务对账**：崩溃后状态恢复
- **丢失任务检测**：`"lost"` 状态

### 5.3 关键差异总结

| 差异点 | CC | OpenClaw |
|--------|-----|----------|
| LLM 任务工具 | ✅ CRUD 工具暴露给 LLM | ❌ 仅内部基础设施 |
| 依赖图 | ✅ blocks/blockedBy | ❌ 无 |
| Dream Tasks | ✅ 后台投机 | ❌ |
| 持久化 | JSON 文件 | ✅ SQLite |
| 投递追踪 | ❌ | ✅ |
| 崩溃恢复 | ❌ | ✅ 对账机制 |

---

## 6. 配置系统对比

### 6.1 CC 的多层配置

```
优先级（高→低）:
1. Enterprise Policy (~/.claude/managed-settings.json) — 不可覆盖
2. CLI Flags (--settings-json)
3. User Global (~/.claude/settings.json)
4. Project (.claude/settings.json)
5. Local (.claude/settings.local.json) — git-ignored
```

### 6.2 OpenClaw 的 YAML 配置

```yaml
# ~/.openclaw/config.yaml — 单文件 + includes
gateway: { port: 18789, auth: { token: ... } }
agents: { default: { models: [...], tools: {...} } }
channels: { slack: { enabled: true }, discord: {...} }
```

**独特能力：**
- 配置 includes/组合
- 环境变量替换 `${VAR}`
- 配置备份轮转
- 运行时快照 (Secret 脱敏)
- 遗留配置自动迁移
- 610KB 生成式 Schema + 150KB 帮助文档

### 6.3 关键差异

| 差异点 | CC | OpenClaw |
|--------|-----|----------|
| 多层优先级 | ✅ 5层 | ❌ 单文件 |
| Enterprise 策略 | ✅ 不可覆盖 | ❌ 无 |
| Local 配置 | ✅ git-ignored | ❌ 无 |
| 配置 includes | ❌ | ✅ |
| 环境变量替换 | ❌ | ✅ |
| 配置备份 | ❌ | ✅ |
| CLAUDE.md | ✅ 项目上下文 | ❌ 无等价物 |

---

## 7. 整体架构差异矩阵

| 维度 | CC 优势 | OpenClaw 优势 |
|------|---------|---------------|
| **工具系统** | 统一 builder, 延迟加载, 结果预算 | 工具 Profile, 通道适配, Owner 门控 |
| **多Agent** | Team 生命周期, 邮箱通信, Plan 审批 | Yield 协作, 流式转发, ACP spawn |
| **权限安全** | AI 自动审批, 交互式审批, 外部 Hooks | 全面审计, 自动修复, SSRF/ReDoS |
| **任务管理** | LLM CRUD 工具, 依赖图, Dream | SQLite 持久化, 投递追踪, 崩溃恢复 |
| **配置** | 多层优先级, Enterprise 策略 | Includes, 环境变量, 备份轮转 |
| **通道** | IDE Bridge | 20+ 消息平台 |
| **模型** | Anthropic 深度集成 | 20+ 提供商 |

---

## 8. 结论

OpenClaw 和 CC 虽然服务于不同场景（多通道个人助手 vs 终端编程工具），但在工具系统、多Agent协作、任务管理等核心架构上存在大量可交叉借鉴的设计模式。OpenClaw 在安全审计、配置灵活性、多通道支持上有显著优势；CC 在工具类型安全、LLM 任务可见性、多Agent团队协作上更为成熟。两者的交叉学习是双向的、互补的。

---

*本报告基于 2026-03-31 时点的代码库分析*
