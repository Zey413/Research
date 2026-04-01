# OpenClaw vs Claude Code: 深度代码级对比研究报告 v2

> **Author:** Zey413 (AARON ZHANG)  
> **Date:** 2026-04-01  
> **版本:** v2.0 — 基于源代码深度阅读的全面更新  
> **前置版本:** v1.0 (2026-03-31) 

---

## 摘要

本报告 v2 基于对 OpenClaw 和 Claude Code (CC) **源代码逐文件阅读**，在 v1 五维度对比基础上，新增了 10+ 个子系统的代码级分析。重点发现：OpenClaw 近期进入**安全加固阶段**（3天29个安全commit），而 CC 在多 Agent 团队协作、AI 权限分类、上下文压缩三个方面具有显著代码级领先。

---

## 第一部分：项目最新动态

### OpenClaw (3天内 29 commits — 安全审计扫荡)

| 类别 | 数量 | 代表性修复 |
|------|------|-----------|
| **安全加固** | ~12 | SSH沙箱符号链接逃逸拦截、可信代理HTTP源校验、危险环境变量覆盖阻断 |
| **媒体/通道** | ~8 | 跨域重定向时剥离认证头、CDP截图跨域图片修复、飞书群线程过滤 |
| **语音/配置** | ~4 | Plivo回调源锁定、超大媒体帧拒绝、Nostr私钥脱敏 |
| **UI/XSS** | ~2 | 删除确认弹窗改用DOM节点构建（消除HTML字符串XSS） |

**启示**: 安全相关PR在当前窗口**更容易被接受**。

### CC (近期状态)

CC 代码库保持稳定迭代，工具系统和多Agent体系已趋成熟。

---

## 第二部分：核心Agent运行时深度对比

### 2.1 OpenClaw Agent 循环 (`pi-embedded-runner/run.ts` — 1,446行)

```
runEmbeddedPiAgent()
├── 双重队列序列化: enqueueSession → enqueueGlobal
├── while(true) 主循环
│   ├── Auth Profile 轮转 (failover on auth failure)
│   ├── Context Engine 压缩 (pluggable)
│   ├── 模型热切换 (hasDifferentLiveSessionModelSelection)
│   ├── Overflow 压缩 (MAX_OVERFLOW_COMPACTION_ATTEMPTS=3)
│   ├── Timeout 压缩 (MAX_TIMEOUT_COMPACTION_ATTEMPTS=2)
│   └── 指数退避重试 (overload backoff with jitter)
├── Hook 生命周期: runBeforeCompaction → runAfterCompaction
└── 错误分类: 字符串匹配 (isLikelyContextOverflowError 等)
```

**设计亮点**:
- 🔹 **Auth Profile 轮转**: 多身份自动切换，单身份失败不阻塞
- 🔹 **双重队列**: session 级 + 全局级序列化，防止并发冲突
- 🔹 **模型热切换**: 运行中检测配置变化并切换模型

**问题点** (PR 机会):
- ⚠️ 1,446 行超出 700 LOC 指南，重试/故障转移逻辑深度嵌套
- ⚠️ 错误分类使用字符串匹配，脆弱且不可维护
- ⚠️ `while(true)` + 多分支 break/continue，应改为状态机模式

### 2.2 CC Agent 循环 (QueryEngine + Tool Loop)

```
QueryEngine.ts
├── LLM API 编排 + Tool Loop
├── Tool 分发: 45+ 工具
├── 权限检查: checkPermissions() per tool
├── 结果处理: maxResultSizeChars 预算
└── Ink 渲染: React 组件化输出
```

**CC 独有**:
- **AgentTool 双执行路径**: 同步等待 vs 后台任务 + race 背景化
- **AsyncLocalStorage 上下文继承**: 嵌套 Agent 生成时传递状态
- **条件 Schema 发射**: `lazySchema + .omit()` 基于 feature flag 控制字段可见性

### 2.3 关键差异

| 方面 | OpenClaw | CC |
|------|----------|-----|
| 执行模型 | 双重队列序列化 | 同步/异步双路径 + race |
| 重试策略 | Auth轮转 + 指数退避 | 结构化错误分类 + 预算控制 |
| 上下文继承 | Session-based | AsyncLocalStorage |
| 模型切换 | ✅ 运行中热切换 | ❌ 启动时固定 |
| Schema 动态性 | 静态 | ✅ feature flag 条件发射 |
| 文件大小 | ⚠️ 1,446行 | 分散在多文件 |

---

## 第三部分：多Agent团队协作深度对比

### 3.1 CC Team 系统 — 完整生命周期

#### TeamCreateTool 关键设计

```typescript
// CC: 确定性 Agent ID — 从 team 名称派生 (非随机UUID)
const leadAgentId = `${TEAM_LEAD_NAME}@${teamName}`;

// 单例约束: 每 session 最多一个 team
if (appState.teamContext?.teamName) throw new Error("Already in team");

// 文件 + 状态双持久化
writeTeamFile(teamFile);  // 磁盘持久化
appState.teamContext = { teamName, teammates };  // 内存状态

// 清理注册: 防止 session 结束后遗留孤立文件
registerTeamForSessionCleanup();
```

#### SendMessageTool 关键设计

```typescript
// CC: 判别联合消息类型 — RPC 语义
type StructuredMessage = 
  | { type: 'text', content: string }
  | { type: 'shutdown_request', request_id: string }
  | { type: 'shutdown_response', request_id: string, acknowledged: boolean }
  | { type: 'plan_approval_response', request_id: string, approved: boolean };

// 多层路由策略
if (to === '*') handleBroadcast();           // 广播
else if (to.startsWith('uds:')) sendViaUDS(); // Unix Domain Socket
else if (to.startsWith('bridge:')) sendViaBridge(); // IDE Bridge
else resolveInProcessSubagent() || sendViaMailbox(); // 名称路由

// 自动恢复: 停止的 Agent 后台恢复
if (agentStopped) resumeAgentBackground(originalMessage);
```

#### Coordinator 模式

```typescript
// CC: 环境变量作为实时真值源 (非缓存)
const isCoordinator = process.env.CLAUDE_CODE_COORDINATOR_MODE === '1';

// 能力注入: Workers 看到受限工具集
const workerTools = ALL_TOOLS.filter(t => !INTERNAL_WORKER_TOOLS.has(t.name));

// 系统提示嵌入工作流指导 (369行!)
// 包含: 任务阶段(Research→Synthesis→Implementation→Verification)
//       并行策略, 反模式警告(懒惰委派), 综合指南
```

### 3.2 OpenClaw Session 系统

```typescript
// OpenClaw: ACP Spawn — 服务端子Agent
type SpawnAcpParams = {
  task: string;
  mode: "run" | "session";           // 一次性 vs 持久会话
  sandbox: "inherit" | "require";     // 沙箱继承
  streamTo: "parent";                 // 流式转发
  thread: boolean;                    // 线程绑定
}

// Yield 协作 — CC 没有的特性
sessions-yield-tool.ts  // 子Agent主动让出控制权

// 流式转发 — CC 没有的特性
acp-spawn-parent-stream.ts  // 子到父实时输出流
```

### 3.3 新发现的 PR 机会

| # | 特性 | 来源 | OpenClaw 差距 | PR 复杂度 |
|---|------|------|--------------|-----------|
| A | **确定性 Agent ID** | CC TeamCreateTool | OpenClaw 使用随机 ID，不利于路由预测 | ⭐ |
| B | **判别联合消息类型** | CC SendMessageTool | `sessions-send-tool.ts` 仅支持自由文本 | ⭐⭐ |
| C | **邮箱路由 + 自动恢复** | CC SendMessageTool | 停止的子 Agent 不能自动恢复 | ⭐⭐ |
| D | **Coordinator 系统提示** | CC coordinatorMode | OpenClaw 无嵌入式工作流指导 | ⭐ |
| E | **Team 清理注册** | CC TeamCreateTool | ACP session 无显式清理注册 | ⭐⭐ |

---

## 第四部分：权限与安全系统深度对比

### 4.1 CC 的 AI 权限分类器 (全新分析)

```typescript
// yoloClassifier.ts (~1500行) — 两阶段XML分类器

// 阶段1: 快速分类 (64 token 预算)
Stage1Result = classifyAction(transcript, tools, {maxTokens: 64});

// 阶段2: 思考模式分类 (4096 token 预算, thinking model)
if (stage1.confidence < threshold) {
  Stage2Result = classifyAction(transcript, tools, {thinking: true, maxTokens: 4096});
}

// 关键数据结构
ClassifierResult = { 
  matches: ToolMatch[],
  confidence: number,  // 0-1
  reason: string
}
ClassifierBehavior = 'deny' | 'ask' | 'allow'

// 独特设计: 转录压缩
toCompactBlock(toolUse) → toAutoClassifierInput(input) // 每工具自定义投影

// 缓存控制策略
splitCacheControl(system, claudeMd, actionBlocks); // GrowthBook TTL 允许列表
```

**为什么重要**: 这让 CC 能自动判断 `git commit` 安全但 `rm -rf /` 危险，无需用户逐个审批。

### 4.2 OpenClaw 安全系统 (新深度分析)

```typescript
// security/audit.ts (57KB!) — 全面安全审计
// OpenClaw 的安全方法完全不同: 预防式审计而非运行时分类

// 审计管线
SecurityAudit = {
  findings: Finding[],      // 发现
  severity: 'critical' | 'high' | 'medium' | 'low',
  remediation: string,      // 修复建议
  autoFixable: boolean      // 是否可自动修复
}

// 安全特性矩阵 (CC 没有的)
- SSRF 防护 (infra/net/ssrf.ts)
- ReDoS 防护 (security/safe-regex.ts)
- 符号链接逃逸检测 (最新commit)
- 跨域认证头剥离 (最新commit)
- 环境变量覆盖阻断 (最新commit)
```

### 4.3 新发现的差异

| 方面 | CC 方法 | OpenClaw 方法 |
|------|---------|--------------|
| **分类方式** | 运行时 AI 分类器 (LLM判断安全性) | 预防式审计 + 白名单 |
| **分类精度** | ★★★★★ 上下文感知 | ★★★ 规则固定 |
| **计算成本** | 高 (每次工具调用消耗LLM token) | 低 (静态规则) |
| **拒绝追踪** | ✅ 3次连续/20次总计限制 | ❌ 无 |
| **SSRF防护** | ❌ | ✅ |
| **ReDoS防护** | ❌ | ✅ |
| **XSS防护** | ❌ (终端无需) | ✅ (Web UI) |

---

## 第五部分：上下文压缩系统对比

### 5.1 CC 的压缩系统 (`compact.ts` ~60KB)

```typescript
// 核心函数
compactConversation(messages[], context, cacheSafeParams, suppressFollowUp,
                    customInstructions?, isAutoCompact?, recompactionInfo?)

// API-Round 分组: 按LLM调用轮次分组消息
groupMessagesByApiRound() → preservedSegment{headUuid, anchorUuid, tailUuid}

// 预算分配 (压缩后)
POST_COMPACT_TOKEN_BUDGET = 50K
  = 5 files × 5K tokens/file (25K)
  + ~5 skills × 5K tokens/skill (25K)

// 独特设计
- 图片剥离: replaceImageBlocks → [image] 占位符
- 工具结果配对: ensureToolResultPairing() 防止孤立 tool_result
- 流式再压缩: 处理 "earlier conversation truncated" 标记
- PTL降级: truncateHeadForPTLRetry() 丢弃最老 API-Round
- Hook 重注入: 压缩后重新注入文件状态 + skill 发现 + MCP 指令
```

### 5.2 OpenClaw 的压缩系统 (`pi-embedded-runner/compact.ts` — 1,097行)

```typescript
// 核心函数
compactEmbeddedPiSession()

// 设计: 委托给 Context Engine (可插拔)
// Hook 生命周期: runBeforeCompaction → compress → runAfterCompaction
// 50+ imports — 极高耦合度

// 问题: 1,097行，严重超出指南
```

### 5.3 关键差异

| 方面 | CC | OpenClaw |
|------|-----|----------|
| **分组策略** | API-Round 分组 + 锚点保留 | Context Engine 委托 |
| **压缩后预算** | 50K token 明确预算 | 无明确预算 |
| **图片处理** | 替换为占位符 | 不明确 |
| **降级策略** | PTL 降级 + Head 截断 | Overflow + Timeout 压缩 |
| **Hook 集成** | 压缩后重注入 | Hook 生命周期 |

---

## 第六部分：记忆系统对比

### 6.1 CC Memory Directory (`memdir.ts` ~21KB)

```typescript
// 4类型闭合分类法
MemoryType = 'user' | 'feedback' | 'project' | 'reference'
// 明确排除: 代码模式, 架构, git 历史 (可从代码库推导)

// 索引 vs 内容分离
MEMORY.md = 始终加载的索引 (~150字符/条目)
topic_files/ = 按需加载的详细内容

// 双重截断
1. 行数限制 (200行)
2. 字节限制 (25KB)

// 保存协议: 写文件 → 添加索引指针 (两步)
```

### 6.2 OpenClaw Memory (`LanceDB 向量搜索`)

```
- LanceDB 向量数据库
- 语义搜索
- 无固定分类法
- 与 agents/memory/ 集成
```

### 6.3 差异

| 方面 | CC | OpenClaw |
|------|-----|----------|
| **存储** | Markdown 文件 + 索引 | LanceDB 向量DB |
| **搜索** | 关键词/索引 | ✅ 语义向量搜索 |
| **分类** | 4类型闭合分类 | 自由形式 |
| **截断** | 双重截断 (行+字节) | 无明确限制 |
| **加载** | 始终加载索引 | 按需搜索 |

---

## 第七部分：其他差异领域

### 7.1 Vim 模式 (CC 独有)

CC 实现了完整的 Vim 状态机:
- 10+ 命令状态 (idle, count, operator, find, replace, indent...)
- 持久状态 (lastChange, lastFind, register) — 支持 dot-repeat
- TaggedUnion 设计 — TS 编译器强制穷举所有分支
- 纯函数操作符执行

OpenClaw: 无等价物。这不是 PR 机会（场景不同）。

### 7.2 Denial Tracking (CC 独有)

```typescript
// 简洁但有效: 防止 AI 反复请求被拒绝的操作
DenialState = { consecutiveDenials: number, totalDenials: number }
MAX_CONSECUTIVE = 3  // 连续3次拒绝后停止请求
MAX_TOTAL = 20       // 总计20次后停止
// 成功执行后重置
```

OpenClaw PR 机会: 在 tool-policy 中添加拒绝追踪，防止 Agent 循环请求被禁止的工具。

### 7.3 工具配置差异 (新发现)

OpenClaw `tool-catalog.ts` 中的 `includeInOpenClawGroup` 模式:
- 27 个工具中只有 6 个被排除
- 应改为 **排除列表** 而非包含列表 (更简洁)

---

## 第八部分：整体架构差异矩阵 (更新版)

| 维度 | CC 优势 | OpenClaw 优势 |
|------|---------|---------------|
| **Agent 运行时** | 双路径执行, AsyncLocalStorage, 条件Schema | Auth轮转, 模型热切换, 双重队列 |
| **多Agent** | Team生命周期, 确定性ID, 邮箱路由, Coordinator | Yield协作, 流式转发, ACP spawn |
| **权限** | AI分类器(两阶段), 拒绝追踪, 外部Hook | 安全审计(57KB), SSRF/ReDoS, 自动修复 |
| **压缩** | API-Round分组, 50K预算, PTL降级, Hook重注入 | Context Engine委托(可插拔) |
| **记忆** | 4类型分类, 索引分离, 双重截断 | LanceDB向量搜索 |
| **工具系统** | buildTool工厂, ToolSearch, 结果预算, 别名 | Tool Profile, 通道适配, Owner门控 |
| **安全** | 运行时AI分类 | 预防式审计, XSS/SSRF/ReDoS全面防护 |
| **代码质量** | 文件分散合理 | ⚠️ 多个1000+行文件(run.ts, compact.ts, acp-spawn.ts) |

---

## 第九部分：结论

v2 分析揭示了两个项目在设计哲学上的根本差异:

1. **CC = 终端优先, 开发者交互**: AI分类器依赖实时LLM判断, Team系统设计精巧, 关注单用户多Agent协作体验
2. **OpenClaw = 服务端优先, 多通道网关**: 安全审计体系全面, Auth轮转灵活, 面向多用户多通道的消息分发

交叉借鉴的价值不在于简单移植，而在于**适配场景**的创新融合。例如:
- CC 的 Team 概念可以适配为 OpenClaw 的 "Agent Group"，但需要考虑多通道消息路由
- OpenClaw 的安全审计可以适配为 CC 的预提交检查，但需要考虑终端交互体验

---

*本报告基于 2026-04-01 时点的代码库深度阅读分析*
