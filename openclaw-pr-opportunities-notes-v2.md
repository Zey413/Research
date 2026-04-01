# OpenClaw PR 机会笔记 v2 — 深度代码级分析后更新

> **Author:** Zey413 (AARON ZHANG)  
> **Date:** 2026-04-01  
> **版本:** v2.0 — 新增 6 个 PR 机会，总计 14 个  
> **目标仓库:** zey413/openclaw (fork) → openclaw/openclaw (upstream)

---

## 全量 PR 可行性评估表

### 🔴 P0 — 立即可做 (本周)

| # | PR 标题 | 难度 | 影响 | 代码量 | 状态 |
|---|---------|------|------|--------|------|
| 1 | 实现 LLM 可见任务 CRUD 工具 | ⭐⭐ | 🔥🔥🔥 | ~300行新代码 | 📋 |
| 2 | 添加 buildOpenClawTool() 工厂 + 安全默认值 | ⭐⭐ | 🔥🔥🔥 | ~100行新 + 逐步迁移 | 📋 |
| 3 | 工具结果大小预算 (maxResultSizeChars) | ⭐ | 🔥🔥 | ~50行 | 📋 |
| 4 | **[新]** 拒绝追踪 (Denial Tracking) | ⭐ | 🔥🔥 | ~80行 | 📋 |

### 🟡 P1 — 短期 (下周)

| # | PR 标题 | 难度 | 影响 | 代码量 | 状态 |
|---|---------|------|------|--------|------|
| 5 | ToolSearch + 延迟加载 | ⭐⭐⭐ | 🔥🔥🔥 | ~200行 | 📋 |
| 6 | **[新]** 结构化 Agent 间协议消息 | ⭐⭐ | 🔥🔥🔥 | ~150行 | 📋 |
| 7 | **[新]** 确定性 Agent ID (从 team 名称派生) | ⭐ | 🔥🔥 | ~30行 | 📋 |
| 8 | **[新]** run.ts 状态机重构 (1446→700行) | ⭐⭐⭐ | 🔥🔥 | 重构 | 📋 |

### 🟢 P2 — 中期

| # | PR 标题 | 难度 | 影响 | 代码量 | 状态 |
|---|---------|------|------|--------|------|
| 9 | Team 概念 (Sessions 之上) | ⭐⭐⭐ | 🔥🔥🔥 | ~500行 | 📋 |
| 10 | Plan 审批工作流 | ⭐⭐ | 🔥🔥 | ~200行 | 📋 |
| 11 | 多层配置优先级 | ⭐⭐⭐ | 🔥🔥 | ~300行 | 📋 |
| 12 | **[新]** 上下文压缩后预算机制 | ⭐⭐ | 🔥🔥 | ~100行 | 📋 |
| 13 | **[新]** 媒体下载 SSRF 防护统一 | ⭐ | 🔥🔥 | ~20行 | 📋 |
| 14 | **[新]** tool-catalog includeInOpenClawGroup 反转 | ⭐ | 🔥 | ~15行 | 📋 |

---

## P0 详细方案

### PR #1: LLM 可见任务 CRUD 工具 (⭐⭐ | 🔥🔥🔥)

**为什么最佳**: OpenClaw 已有 `task-registry.ts` (SQLite持久化)，但对 LLM 不可见。

**实现文件清单**:
```
新增: src/agents/tools/task-tools.ts        (~250行)
修改: src/agents/openclaw-tools.ts          (+10行, 注册工具)
修改: src/agents/tool-catalog.ts            (+4行, 添加到 profile)
可选: src/tasks/task-registry.ts            (+20行, blocks/blockedBy)
```

**核心代码**:
```typescript
// src/agents/tools/task-tools.ts
import { z } from 'zod';
import type { AnyAgentTool } from './common.js';

export function createTaskCreateTool(ctx: ToolContext): AnyAgentTool {
  return {
    name: 'task_create',
    description: 'Create a task to track multi-step work with status tracking',
    inputSchema: z.object({
      subject: z.string().describe('Brief task title'),
      description: z.string().describe('What needs to be done'),
    }),
    async call({ subject, description }) {
      const task = await ctx.taskRegistry.create({
        subject, description,
        runtime: 'acp',
        sourceId: ctx.agentId,
        status: 'queued',
      });
      return jsonResult({ taskId: task.taskId, status: 'created' });
    }
  };
}

export function createTaskUpdateTool(ctx: ToolContext): AnyAgentTool {
  return {
    name: 'task_update',
    inputSchema: z.object({
      taskId: z.string(),
      status: z.enum(['queued', 'running', 'succeeded', 'failed']).optional(),
      progressSummary: z.string().optional(),
    }),
    async call({ taskId, status, progressSummary }) {
      await ctx.taskRegistry.update(taskId, { status, progressSummary });
      return jsonResult({ taskId, updated: true });
    }
  };
}

export function createTaskListTool(ctx: ToolContext): AnyAgentTool { /* ... */ }
export function createTaskGetTool(ctx: ToolContext): AnyAgentTool { /* ... */ }
```

**PR 消息**:
```
feat(agents): expose task registry to LLM via CRUD tools

The task registry (SQLite-backed) already exists as infrastructure
but is invisible to LLM agents. Add four tools that let agents:
- Create tasks to decompose complex work
- Track progress with status updates
- List all tasks to plan next steps
- Get detailed task info including dependencies

This enables LLM-driven project management within agent sessions.
```

---

### PR #2: buildOpenClawTool() 工厂 + 安全默认值 (⭐⭐ | 🔥🔥🔥)

**问题**: OpenClaw 工具使用简单工厂函数，无统一安全默认值。新工具容易遗漏安全标记。

**学习对象**: CC 的 `buildTool()` + `TOOL_DEFAULTS` (fail-closed 设计)

**核心代码**:
```typescript
// 修改: src/agents/tools/common.ts

export interface ToolSafetyFlags {
  isReadOnly: boolean;
  isConcurrencySafe: boolean;
  isDestructive: boolean;
  maxResultSizeChars: number;
  ownerOnly: boolean;
}

const SAFE_DEFAULTS: ToolSafetyFlags = {
  isReadOnly: false,           // fail-closed: assume writable
  isConcurrencySafe: false,    // fail-closed: assume unsafe
  isDestructive: false,
  maxResultSizeChars: 30_000,  // 30K chars — prevent context overflow
  ownerOnly: false,
};

export function buildOpenClawTool<TInput, TOutput>(
  def: ToolDefinition<TInput, TOutput> & Partial<ToolSafetyFlags>
): AnyAgentTool {
  const merged = { ...SAFE_DEFAULTS, ...def };
  
  // 结果截断 wrapper
  const originalCall = merged.call;
  merged.call = async (...args) => {
    const result = await originalCall(...args);
    if (typeof result === 'string' && result.length > merged.maxResultSizeChars) {
      return result.slice(0, merged.maxResultSizeChars) 
        + `\n[Truncated: ${result.length} chars → ${merged.maxResultSizeChars}]`;
    }
    return result;
  };
  
  return merged;
}
```

**PR 消息**:
```
feat(tools): add buildOpenClawTool factory with fail-closed safety defaults

Introduce a unified tool builder that enforces safe defaults:
- isReadOnly=false (assume tools can write until proven otherwise)
- isConcurrencySafe=false (assume not safe for parallel execution)  
- maxResultSizeChars=30000 (prevent context window overflow)

Includes result truncation wrapper that caps tool output size.
New tools created via buildOpenClawTool() are secure-by-default.
```

---

### PR #3: 工具结果大小预算 (⭐ | 🔥🔥)

**注意**: 如果 PR #2 先合并，此 PR 的核心逻辑已包含在 `buildOpenClawTool()` 中。独立提交时：

```typescript
// 新增: src/agents/tools/result-budget.ts

export function truncateToolResult(
  result: string, 
  maxChars: number = 30_000,
  saveToDisk: boolean = true
): { text: string; truncated: boolean; diskPath?: string } {
  if (result.length <= maxChars) {
    return { text: result, truncated: false };
  }
  
  const diskPath = saveToDisk 
    ? saveLargeResultToDisk(result) 
    : undefined;
  
  return {
    text: result.slice(0, maxChars) + 
      `\n\n[Result truncated: ${result.length} → ${maxChars} chars` +
      (diskPath ? `. Full result: ${diskPath}` : '') + ']',
    truncated: true,
    diskPath,
  };
}
```

---

### PR #4 [新]: 拒绝追踪 (Denial Tracking) (⭐ | 🔥🔥)

**来源**: CC `denialTracking.ts` — 简洁但有效地防止 Agent 循环请求被禁止的操作。

**问题**: OpenClaw 的 `tool-policy.ts` 拒绝工具调用后，Agent 可能反复尝试同一工具，浪费 token。

**核心代码**:
```typescript
// 新增: src/agents/tools/denial-tracking.ts

export interface DenialState {
  readonly consecutiveDenials: number;
  readonly totalDenials: number;
}

const MAX_CONSECUTIVE_DENIALS = 3;
const MAX_TOTAL_DENIALS = 20;

const denialMap = new Map<string, DenialState>();

export function recordDenial(toolName: string): void {
  const current = denialMap.get(toolName) ?? { consecutiveDenials: 0, totalDenials: 0 };
  denialMap.set(toolName, {
    consecutiveDenials: current.consecutiveDenials + 1,
    totalDenials: current.totalDenials + 1,
  });
}

export function recordSuccess(toolName: string): void {
  const current = denialMap.get(toolName);
  if (current) {
    denialMap.set(toolName, { ...current, consecutiveDenials: 0 });
  }
}

export function shouldSuppressTool(toolName: string): boolean {
  const state = denialMap.get(toolName);
  if (!state) return false;
  return state.consecutiveDenials >= MAX_CONSECUTIVE_DENIALS 
      || state.totalDenials >= MAX_TOTAL_DENIALS;
}
```

**集成点**: `src/agents/tool-policy-pipeline.ts` — 在策略管线末端添加拒绝检查。

**PR 消息**:
```
feat(agents): add denial tracking to prevent tool call loops

When a tool is repeatedly denied by policy, the agent may waste
tokens retrying. Add simple denial tracking:
- After 3 consecutive denials, suppress the tool
- After 20 total denials across a session, suppress
- Successful execution resets consecutive counter

This reduces wasted LLM tokens on prohibited operations.
```

---

## P1 详细方案

### PR #6 [新]: 结构化 Agent 间协议消息 (⭐⭐ | 🔥🔥🔥)

**来源**: CC SendMessageTool 的判别联合消息类型

**当前状态**: `sessions-send-tool.ts` 仅支持自由文本消息，无法表达"关闭请求"、"计划审批"等 RPC 语义。

```typescript
// 修改: src/agents/tools/sessions-send-tool.ts

// 新增结构化消息类型
const StructuredMessageSchema = z.discriminatedUnion('type', [
  z.object({ type: z.literal('text'), content: z.string() }),
  z.object({ type: z.literal('shutdown_request'), reason: z.string() }),
  z.object({ type: z.literal('shutdown_response'), acknowledged: z.boolean() }),
  z.object({ type: z.literal('status_update'), status: z.string(), progress: z.number().optional() }),
  z.object({ type: z.literal('task_complete'), summary: z.string(), success: z.boolean() }),
]);

// 在 send 工具的 inputSchema 中:
message: z.union([
  z.string(),  // 向后兼容: 纯文本
  StructuredMessageSchema,  // 新: 结构化消息
])
```

---

### PR #7 [新]: 确定性 Agent ID (⭐ | 🔥🔥)

**来源**: CC `TeamCreateTool` 中 `TEAM_LEAD_NAME@teamName` 模式

**好处**: 可预测的路由，无需查找随机 UUID

```typescript
// 修改: src/agents/acp-spawn.ts

// 之前: const childAgentId = generateRandomId();
// 之后:
function deriveAgentId(label: string, parentSessionKey: string): string {
  return `${sanitize(label)}@${parentSessionKey.slice(0, 8)}`;
}
```

---

### PR #8 [新]: run.ts 状态机重构 (⭐⭐⭐ | 🔥🔥)

**问题**: `pi-embedded-runner/run.ts` 1,446行，`while(true)` + 多分支 break/continue

**方案**: 提取为状态机模式

```typescript
// 新增: src/agents/pi-embedded-runner/run-states.ts

type RunState = 
  | { phase: 'init' }
  | { phase: 'attempt', profileIndex: number }
  | { phase: 'auth_failover', failedProfile: string }
  | { phase: 'overflow_compact', attempt: number }
  | { phase: 'timeout_compact', attempt: number }
  | { phase: 'overload_backoff', waitMs: number }
  | { phase: 'model_switch', newModel: string }
  | { phase: 'complete', result: EmbeddedPiRunResult }
  | { phase: 'error', error: Error };

export function nextState(current: RunState, event: RunEvent): RunState {
  // 纯函数状态转换 — 可测试、可追踪
}
```

**注意**: 这是纯重构 PR — OpenClaw 规则要求"不能是纯重构PR"。需要附带功能改进:
- 添加状态转换日志 (可观测性提升)
- 添加状态转换单元测试 (测试覆盖提升)
- 优化: 用事件驱动替代 `waitForActiveEmbeddedRuns` 的轮询

---

## P2 详细方案 (概要)

### PR #12 [新]: 上下文压缩后预算机制

**来源**: CC 的 `POST_COMPACT_TOKEN_BUDGET = 50K` 设计

```typescript
// 修改: src/agents/pi-embedded-runner/compact.ts
const POST_COMPACT_BUDGET = {
  files: 5 * 5_000,     // 5 files × 5K tokens
  skills: 5 * 5_000,    // 5 skills × 5K tokens  
  total: 50_000,
};
// 压缩后注入内容不超过此预算
```

### PR #13 [新]: 媒体下载 SSRF 防护统一

**问题**: `media/store.ts` 使用原生 `http.request`，而 `media/fetch.ts` 使用 `fetchWithSsrfGuard`。不一致。

```diff
// media/store.ts
- import { request } from 'node:https';
+ import { fetchWithWebToolsNetworkGuard } from '../infra/net/ssrf.js';
```

**这是当前安全审计窗口的完美 PR** — 与最近的安全 commit 方向一致。

### PR #14 [新]: tool-catalog 反转 includeInOpenClawGroup

**问题**: 27 个工具中仅 6 个被排除。用包含列表 (21个 true) 不如排除列表 (6个 false) 简洁。

```diff
// src/agents/tool-catalog.ts
- includeInOpenClawGroup: true,   // 重复21次
+ excludeFromOpenClawGroup: true, // 仅需6次
```

---

## 🎯 推荐执行顺序

```
Week 1 (立即):
├── PR #4 (Denial Tracking) — 最小改动, 独立性强
├── PR #13 (SSRF统一) — 安全窗口, 20行代码
└── PR #3 (结果预算) — 简单, 有价值

Week 2:
├── PR #1 (Task CRUD) — 最大价值
├── PR #2 (buildTool工厂) — 基础设施
└── PR #7 (确定性ID) — 快速

Week 3:
├── PR #6 (结构化消息) — 多Agent基础
├── PR #5 (ToolSearch) — 依赖 PR#2
└── PR #14 (catalog反转) — 代码清理

Week 4+:
├── PR #8 (run.ts重构) — 需社区讨论
├── PR #9 (Team概念) — 大特性
├── PR #12 (压缩预算) — 依赖compact.ts理解
├── PR #10 (Plan审批) — 依赖消息类型
└── PR #11 (多层配置) — 架构级变更
```

---

## ⚠️ OpenClaw 贡献规则提醒

1. **不能是纯重构 PR** — 必须附带功能价值 (PR #8 需要注意)
2. **安全 PR 优先** — 当前处于安全审计阶段 (PR #4, #13 最佳时机)
3. **必须通过 AI review**
4. **`pnpm build` + `pnpm test` 必须通过**
5. **无 `[INEFFECTIVE_DYNAMIC_IMPORT]` 警告**
6. **插件只能从 `openclaw/plugin-sdk/*` 导入** — 不要触碰 `src/**` 内部

---

*本笔记基于 2026-04-01 时点的源代码深度阅读分析*
