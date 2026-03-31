# OpenClaw PR 机会笔记 — 从 Claude Code 借鉴的特性

> **Author:** Zey413 (AARON ZHANG)
> **Date:** 2026-03-31
> **目标仓库:** zey413/openclaw (fork)
> **上游仓库:** openclaw/openclaw

---

## PR 可行性评估总表

| # | PR 标题 | 难度 | 影响 | 优先级 | 涉及文件 | 状态 |
|---|---------|------|------|--------|----------|------|
| 1 | 实现 LLM 可见任务 CRUD 工具 | ⭐⭐ | 🔥🔥🔥 | **P0** | agents/tools/ | 📋 待开始 |
| 2 | 添加 buildTool() 统一工厂 + 安全默认值 | ⭐⭐ | 🔥🔥🔥 | **P0** | agents/tools/common.ts | 📋 待开始 |
| 3 | 实现 ToolSearch + 延迟加载 | ⭐⭐⭐ | 🔥🔥🔥 | **P1** | agents/openclaw-tools.ts, tool-catalog.ts | 📋 待开始 |
| 4 | 添加工具结果大小预算 | ⭐ | 🔥🔥 | **P1** | agents/tools/ | 📋 待开始 |
| 5 | 添加 Team 概念 (sessions 之上) | ⭐⭐⭐ | 🔥🔥🔥 | **P1** | agents/tools/, gateway/ | 📋 待开始 |
| 6 | 实现结构化 Agent 间协议消息 | ⭐⭐ | 🔥🔥 | **P2** | agents/tools/sessions-*.ts | 📋 待开始 |
| 7 | 添加 Plan 审批工作流 | ⭐⭐ | 🔥🔥 | **P2** | agents/ | 📋 待开始 |
| 8 | 多层配置优先级 | ⭐⭐⭐ | 🔥🔥 | **P2** | config/ | 📋 待开始 |

---

## PR #1: 实现 LLM 可见任务 CRUD 工具 (P0 — 最推荐)

### 为什么这是最佳 PR

OpenClaw **已经有** `task-registry.ts` (SQLite 持久化的任务后端)，但这些任务对 LLM **不可见** — LLM 无法创建、查看或更新自己的工作计划。CC 通过 `TaskCreate/Get/Update/List` 工具让 LLM 可以：

- 将复杂任务分解为可追踪的子步骤
- 标记进度 (pending → in_progress → completed)
- 设置任务间依赖 (blocks/blockedBy)
- 在 UI 中展示实时进度

### 实现方案

```typescript
// 新文件: src/agents/tools/task-tools.ts

import { createTool } from './common.js';
import { taskRegistry } from '../../tasks/task-registry.js';

export function createTaskCreateTool(ctx) {
  return createTool({
    name: 'task_create',
    description: 'Create a task to track progress on a complex operation',
    inputSchema: z.object({
      subject: z.string(),
      description: z.string(),
    }),
    async call({ subject, description }) {
      const task = await taskRegistry.create({
        subject, description,
        runtime: 'acp',
        sourceId: ctx.agentId,
        status: 'queued',
      });
      return { taskId: task.taskId, status: 'created' };
    }
  });
}

export function createTaskListTool(ctx) { /* ... */ }
export function createTaskUpdateTool(ctx) { /* ... */ }
export function createTaskGetTool(ctx) { /* ... */ }
```

### 需要修改的文件

1. **新增** `src/agents/tools/task-tools.ts` — 4个工具实现
2. **修改** `src/agents/openclaw-tools.ts` — 注册新工具
3. **修改** `src/agents/tool-catalog.ts` — 添加到 profile 配置
4. **可选** `src/tasks/task-registry.ts` — 添加 blocks/blockedBy 字段

### PR 消息模板

```
feat(agents): add LLM-facing task CRUD tools

Expose the existing task registry to LLM agents via four new tools:
- task_create: Create tasks to track multi-step work
- task_list: View all tasks and their status
- task_get: Get detailed task info
- task_update: Update status and add dependencies

The task registry backend (SQLite) already exists; this PR bridges
the gap by making tasks visible and manageable by the LLM itself.

Inspired by Claude Code's task management pattern where the LLM
can decompose complex work into tracked sub-steps.
```

---

## PR #2: 添加 buildTool() 统一工厂 + 安全默认值 (P0)

### 问题

OpenClaw 的工具使用简单工厂函数，没有统一的安全默认值。每个工具需要手动设置 `isReadOnly`、`isConcurrencySafe` 等属性。遗漏可能导致安全风险。

### CC 的做法

```typescript
// CC 的 TOOL_DEFAULTS — fail-closed 设计
const TOOL_DEFAULTS = {
  isReadOnly: () => false,        // 假设可写
  isConcurrencySafe: () => false, // 假设不安全
  isDestructive: () => false,
  maxResultSizeChars: 30000,
  interruptBehavior: () => 'block',
}
```

### 实现方案

```typescript
// 修改: src/agents/tools/common.ts

export interface ToolDefaults {
  isReadOnly: boolean;
  isConcurrencySafe: boolean;
  isDestructive: boolean;
  maxResultSizeChars: number;
  ownerOnly: boolean;
}

const SAFE_DEFAULTS: ToolDefaults = {
  isReadOnly: false,         // fail-closed: assume writable
  isConcurrencySafe: false,  // fail-closed: assume unsafe
  isDestructive: false,
  maxResultSizeChars: 30000,
  ownerOnly: false,
};

export function buildOpenClawTool<T>(
  def: Partial<ToolDefaults> & Required<Pick<AnyAgentTool, 'name' | 'call' | 'description'>>
): AnyAgentTool {
  return { ...SAFE_DEFAULTS, ...def };
}
```

### 需要修改的文件

1. **修改** `src/agents/tools/common.ts` — 添加 `buildOpenClawTool()` 和 `SAFE_DEFAULTS`
2. **逐步迁移** 各工具文件使用新工厂

### PR 消息

```
feat(tools): add buildOpenClawTool factory with fail-closed defaults

Add a unified tool builder function that enforces safe defaults:
- isReadOnly=false (assume tools can write)
- isConcurrencySafe=false (assume tools are not concurrency-safe)
- maxResultSizeChars=30000 (prevent context overflow)

This ensures new tools are secure-by-default without requiring
developers to remember every safety flag.
```

---

## PR #3: ToolSearch + 延迟加载 (P1)

### 问题

OpenClaw 在 `createOpenClawTools()` 中全量加载所有工具，即使大部分不会被使用。这浪费系统提示 token。

### 解决方案

利用已有的 `tool-catalog.ts` 元数据，实现 ToolSearch 工具：

```typescript
// 新文件: src/agents/tools/tool-search-tool.ts

export function createToolSearchTool(allTools, activeTools) {
  return createTool({
    name: 'tool_search',
    description: 'Search for specialized tools by keyword. Use when you need a tool not in your active set.',
    inputSchema: z.object({
      query: z.string().describe('What capability you need'),
    }),
    async call({ query }) {
      const deferred = allTools.filter(t => t.shouldDefer && !activeTools.has(t.name));
      const matches = deferred.filter(t => 
        t.name.includes(query) || t.description?.includes(query) || t.searchHint?.includes(query)
      );
      // Activate matched tools
      matches.forEach(t => activeTools.add(t.name));
      return { activated: matches.map(t => t.name), count: matches.length };
    }
  });
}
```

### 需要修改的文件

1. **新增** `src/agents/tools/tool-search-tool.ts`
2. **修改** `src/agents/tool-catalog.ts` — 为每个工具添加 `shouldDefer` 和 `searchHint`
3. **修改** `src/agents/openclaw-tools.ts` — 分离 core tools 和 deferred tools

---

## PR #4: 工具结果大小预算 (P1)

### 问题

工具返回大量结果时可能溢出上下文窗口。CC 通过 `maxResultSizeChars` 限制每个工具的输出大小，超出部分持久化到磁盘。

### 实现方案

```typescript
// 在工具执行后添加截断逻辑
function truncateResult(result: string, maxChars: number): string {
  if (result.length <= maxChars) return result;
  const truncated = result.slice(0, maxChars);
  return `${truncated}\n\n[Result truncated: ${result.length} chars, showing first ${maxChars}. Full result saved to disk.]`;
}
```

### 需要修改的文件

1. **修改** `src/agents/tools/common.ts` — 添加 `maxResultSizeChars` 到 ToolDefaults
2. **修改** Pi Agent 工具执行循环 — 添加截断逻辑

---

## PR #5: Team 概念 (P1 — 较大 PR)

### 问题

OpenClaw 有 Sessions 作为多 Agent 原语，但没有 "Team" 概念。Session 是 1:1 的，缺少：
- 一组 Agent 的生命周期管理
- 共享任务列表
- 广播消息
- 结构化协作协议

### CC 的做法

```typescript
// Team = 一组 Agent + 共享任务列表 + 邮箱通信
TeamFile = {
  name, description,
  leadAgentId,                    // 团队领导
  members: [{ agentId, name, model, subscriptions }],
  taskListDir: string,            // 共享任务目录
}
```

### OpenClaw 实现思路

利用现有 session 基础设施，在上层添加 Team 抽象：

```typescript
// 新文件: src/agents/tools/team-tools.ts
interface TeamDefinition {
  name: string;
  description: string;
  leadSessionId: string;
  memberSessionIds: string[];
  sharedTaskListId?: string;
}

export function createTeamCreateTool(ctx) {
  // 1. 创建多个 session (ACP spawn)
  // 2. 注册为 Team
  // 3. 创建共享任务列表
  // 4. 返回 team ID
}

export function createTeamMessageTool(ctx) {
  // 支持 to: "member_name" 或 to: "*" (广播)
}
```

---

## PR #6: 结构化 Agent 间协议消息 (P2)

### 问题

OpenClaw 的 `sessions-send-tool.ts` 发送自由文本消息。缺少类型化的协议消息（如关闭请求、计划审批等）。

### 实现方案

```typescript
// 扩展 sessions-send-tool.ts
type StructuredMessage = 
  | { type: 'text', content: string }
  | { type: 'shutdown_request', reason: string }
  | { type: 'shutdown_response', acknowledged: boolean }
  | { type: 'plan_approval_request', plan: string }
  | { type: 'plan_approval_response', approved: boolean, feedback?: string }
  | { type: 'status_update', status: string, progress?: number };
```

---

## PR #7: Plan 审批工作流 (P2)

### 问题

Agent 在执行复杂任务前没有 "先展示计划、获得批准后执行" 的机制。

### CC 的做法

- `EnterPlanMode` → Agent 进入计划模式，只读探索
- Agent 写计划文件
- `ExitPlanMode` → 用户审批
- 审批通过 → 开始实现

### OpenClaw 适配

由于 OpenClaw 是服务端（无交互终端），可实现：
- Agent 发送计划到父 session / 消息通道
- 等待用户通过消息回复审批
- 审批后继续执行

---

## PR #8: 多层配置优先级 (P2)

### 问题

OpenClaw 使用单一 YAML 文件 + includes，没有 enterprise → user → project → local 的优先级链。

### CC 的做法

5 层配置 merge，高优先级覆盖低优先级，enterprise 不可覆盖。

### OpenClaw 适配

```yaml
# Enterprise (不可覆盖): /etc/openclaw/policy.yaml
# User: ~/.openclaw/config.yaml
# Project: .openclaw/config.yaml  
# Local: .openclaw/config.local.yaml (git-ignored)
```

---

## 贡献策略建议

### 第一阶段 (本周)
1. **PR #1 (Task CRUD)** — 最小改动、最大价值
2. **PR #2 (buildTool)** — 基础设施改进，后续 PR 依赖

### 第二阶段 (下周)
3. **PR #4 (结果预算)** — 快速实现
4. **PR #3 (ToolSearch)** — 需要 PR #2 完成

### 第三阶段 (长期)
5. **PR #5-8** — 较大特性，需社区讨论

### 提交前检查清单

根据 OpenClaw 贡献规则：
- [ ] 不能是纯重构 PR（必须有功能价值）
- [ ] 必须通过 AI review
- [ ] 满足三个前提条件
- [ ] `pnpm build` 通过
- [ ] `pnpm test` 通过
- [ ] 无 `[INEFFECTIVE_DYNAMIC_IMPORT]` 警告

---

*本笔记基于 2026-03-31 时点的代码分析，具体实现需根据最新代码库调整*
