# Dexter Subagent 并行委派

> 主文档：[PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md)
>
> 这份文档专门讲清楚 `spawn_subagent` 工具的设计——一层深度的并行任务委派、隔离的 agent loop、以及为什么这个模式在 tool-calling agent 里特别有用。

`spawn_subagent` 是 Dexter 里一个"元工具"——LLM 可以在一轮里同时发多个 `spawn_subagent` 调用，每个都启动一个**独立的 Agent 实例**并行跑，最后主 agent 拿到 N 份答案再综合。这个模式在 Claude Code、Cursor 等 tool-calling agent 里都有类似实现，值得单独讲。

## 目录

- [一、要解决的问题](#一要解决的问题)
- [二、核心设计：One level deep](#二核心设计one-level-deep)
- [三、Subagent 类型与工具白名单](#三subagent-类型与工具白名单)
- [四、并行执行是如何"自然发生"的](#四并行执行是如何自然发生的)
- [五、Subagent 的隔离机制](#五subagent-的隔离机制)
- [六、进度回传：progress channel + JSON 编码](#六进度回传progress-channel--json-编码)
- [七、结果综合：一份 verbatim 答案](#七结果综合一份-verbatim-答案)
- [八、Subagent vs 主 agent 自己调工具](#八subagent-vs-主-agent-自己调工具)
- [九、局限与坑](#九局限与坑)
- [十、可以拿走的三个原则](#十可以拿走的三个原则)

---

## 一、要解决的问题

**场景**：用户问 "对比 NVDA、AMD、INTC 三家公司过去 5 年的护城河演变"。

**朴素方案**：主 agent 自己一步步跑——先 `get_financials(NVDA)`，然后 `read_filings(NVDA)`，再 `get_financials(AMD)`... 三家公司串行来，20+ 次工具调用把主 agent 的 context 塞满。等到最后要综合分析时，context 里全是原始数据，主 agent 已经"lost in the middle"了。

**Subagent 方案**：主 agent 一次发 3 个 `spawn_subagent`，一个 subagent 负责一家公司：

```
主 agent 说：spawn_subagent × 3（并行）
  ├─ subagent A: 深入研究 NVDA 护城河 → 返回 500 字结论
  ├─ subagent B: 深入研究 AMD 护城河  → 返回 500 字结论
  └─ subagent C: 深入研究 INTC 护城河 → 返回 500 字结论
主 agent 拿到 3 份结论 → 综合对比 → 最终答案
```

**收益**：
1. **主 agent context 保持干净**——只看到 3 份浓缩结论，看不到 20+ 次工具调用的原始 JSON
2. **并行加速**——3 家公司同时研究，wall time 从 3 倍降到 1 倍多
3. **失败隔离**——subagent A 挂了，B 和 C 照跑

## 二、核心设计：One level deep

**只允许一层委派**——subagent 不能再派 subagent。看 `src/tools/subagent/types.ts`：

```typescript
export const SUBAGENT_DISALLOWED_TOOLS = new Set<string>([
  'spawn_subagent',      // ← 关键：subagent 不能再派 subagent
  'ask_user_question',    // subagent 见不到用户
  'bash',                 // subagent 不做副作用
]);

export function resolveSubagentTools(typeKey: string): string[] {
  const cfg = SUBAGENT_TYPES[typeKey] ?? SUBAGENT_TYPES[DEFAULT_SUBAGENT_TYPE];
  return cfg.tools.filter(t => !SUBAGENT_DISALLOWED_TOOLS.has(t));
}
```

**为什么只一层深？**

1. **可预测性**：多层树状 agent 展开后，谁在什么状态、谁在等谁、谁失败了、总消耗多少 token——全部失控。一层深的"扇出"模型可控多了
2. **成本可控**：如果 subagent 也能派 subagent，理论上呈指数扩张（虽然 maxIterations 会兜底，但边界不清晰）
3. **调试友好**：主 agent → subagent → 结果，链路短、trace 简单。深度递归调试是噩梦
4. **Claude Code、Cursor 都这么做**：业界经验，这个决策不是拍脑袋的

**权衡**：某些真正需要多层分解的任务做不了。但 Dexter 的定位是"金融研究"，绝大多数任务扇出一层就够。

## 三、Subagent 类型与工具白名单

`SUBAGENT_TYPES` 定义三种预设：

```typescript
const READ_ONLY_TOOLS = [
  'get_financials', 'get_market_data', 'read_filings', 'stock_screener',
  'web_search', 'x_search', 'web_fetch',
  'read_file', 'memory_search', 'memory_get',
];

export const SUBAGENT_TYPES: Record<string, SubagentTypeConfig> = {
  'general-purpose': {
    whenToUse: 'Multi-step research or analysis on one focused sub-task.',
    systemPrompt: `${WORKER_PREAMBLE}\n\nYou are a general-purpose research worker...`,
    tools: READ_ONLY_TOOLS,       // ← 全套只读工具
    maxIterations: 8,
  },
  research: {
    whenToUse: 'Gather and synthesize external information on a single topic.',
    systemPrompt: `${WORKER_PREAMBLE}\n\nYou are a research worker...`,
    tools: ['web_search', 'x_search', 'web_fetch', 'read_filings', 'get_market_data'],
    maxIterations: 8,
  },
  analysis: {
    whenToUse: 'Quantitative financial analysis on specific companies.',
    systemPrompt: `${WORKER_PREAMBLE}\n\nYou are a financial analysis worker...`,
    tools: ['get_financials', 'get_market_data', 'stock_screener', 'read_filings'],
    maxIterations: 8,
  },
};
```

**Worker preamble** 是 subagent system prompt 的第一段：

```
You are a subagent working on a single sub-task assigned by an orchestrator.
You run in isolation: you cannot see the main conversation and you cannot delegate to
other subagents. Complete only the assigned task. Your final message is returned
verbatim to the orchestrator, so make it a complete, self-contained answer — state
your findings and conclusions directly, not a description of what you did.
```

**关键点**：

- **`READ_ONLY_TOOLS` 白名单**：只读工具才能给 subagent。写文件、编辑 memory、跑 bash 都不给——subagent 并行跑，如果都能写文件会 race condition，且写操作需要审批而 subagent 没法弹窗
- **不同类型不同工具集**：research 类型只要 web + filings，不给财报工具（约束更紧的 subagent 更容易保持聚焦）
- **maxIterations = 8**：比主 agent 的 10 稍紧。subagent 定位是"聚焦的一个子任务"，不该跑太长
- **强制 disallow list 兜底**：即使 config 里错写了 `spawn_subagent`，`resolveSubagentTools` 会 filter 掉

## 四、并行执行是如何"自然发生"的

**这里有个特别巧妙的设计**：subagent 的并行**不是新写的并行代码**，而是**复用了 tool executor 的并发能力**。

回到 `src/tools/registry.ts`：

```typescript
{
  name: 'spawn_subagent',
  tool: createSpawnSubagent(model),
  description: SPAWN_SUBAGENT_DESCRIPTION,
  compactDescription: 'Delegate a focused sub-task to an isolated subagent. Emit multiple calls in one turn to run independent sub-tasks in parallel.',
  concurrencySafe: true,   // ← 关键：标记为并发安全
},
```

`concurrencySafe: true`——`spawn_subagent` 被标记为**只读工具**（虽然它启动一个 agent，但从主 agent 视角看是只读的：不改主 agent 的任何状态）。

于是 tool executor 的 `partitionToolCalls` 会把连续的 `spawn_subagent` 调用**分到同一并发批**：

```
LLM 一次返回：
[spawn_subagent(NVDA), spawn_subagent(AMD), spawn_subagent(INTC)]
     ✓ concurrency-safe    ✓            ✓

→ 同一批 → 并发跑 → 通过 all(generators, maxConcurrency=10) 合流
```

**这是"复用已有基础设施"的典范**：想让 subagent 并行 → 不需要新的 promise pool、不需要新的调度器、不需要新的合流逻辑，**只要在 tool registry 里标一个 `concurrencySafe: true`**。剩下的全部走已有的 tool-executor 通道。

系统 prompt 里也告诉 LLM：

```
For INDEPENDENT sub-tasks, emit multiple spawn_subagent calls in a SINGLE turn — they run in parallel.
Chain across turns only when one sub-task depends on another's output.
```

**LLM 一次发 3 个 tool_call → 自动并发 3 个 agent**。

## 五、Subagent 的隔离机制

`spawn-subagent.ts` 里的核心创建代码：

```typescript
const { Agent } = await import('../../agent/agent.js');   // ← 懒加载解循环依赖

const subagent = await Agent.create({
  model,
  maxIterations: typeCfg.maxIterations,
  signal,
  memoryEnabled: false,                       // ← 关键
  toolAllowlist,                              // ← 白名单
  systemPromptOverride: typeCfg.systemPrompt, // ← 完全独立的 prompt
  agentLabel: typeKey,
});
```

**几个隔离维度**：

**1. `memoryEnabled: false`**——subagent 不访问 memory。原因：
- Memory 是**用户长期偏好**，subagent 做的是**当前 query 的子任务**，看用户偏好意义不大
- 避免 memory search 污染 subagent context
- 避免 memory flush 在 subagent 里意外触发写入

**2. `toolAllowlist`**——按 subagent type 只暴露白名单里的工具。主 agent 视角这就是 `Agent.create` 里那个 filter：

```typescript
let tools = config.toolAllowlist
  ? allTools.filter(t => config.toolAllowlist!.includes(t.name))
  : allTools;
```

Disallow list 里的（`spawn_subagent` / `ask_user_question` / `bash`）**根本没进 tool schema**，subagent 的 LLM 永远看不到它们。

**3. `systemPromptOverride`**——subagent 用自己的 self-contained system prompt，**跳过 SOUL、rules、memory context、channel profile**。原因：
- SOUL 是主 agent 的"人格"，subagent 是执行工，不需要
- Rules 已经反映在主 agent 传下来的 task 里
- Memory context 上面禁用了
- Channel profile 只影响最终输出格式，subagent 的输出是给主 agent 看的，不是给终端用户

这一步节省大量 system prompt token——subagent 的 prompt 可能只有 500 tokens，主 agent 的是 5000+。

**4. Task 是"自包含"的**——主 agent 通过 `task` 参数把所有 subagent 需要的信息传进去：

```typescript
const query = input.context
  ? `${input.context}\n\n---\n\nTask: ${input.task}`
  : input.task;
```

Subagent 见不到主 agent 的对话历史，所以主 agent 必须**把所有上下文塞进 `task` + `context`**。System prompt 里也告诉 LLM 这一点：

```
The subagent runs in isolation — it cannot see this conversation and cannot delegate further.
Put everything it needs into `task` (and optional `context`).
It returns one complete answer that you then synthesize.
```

## 六、进度回传：progress channel + JSON 编码

Subagent 跑起来后，主 agent 需要**知道它跑到哪一步了**——不然 UI 上就是一片死寂。

用的是 tool 的 `onProgress` channel（第一节 tool-executor 里详解过）：

```typescript
const onProgress = config?.metadata?.onProgress as ((msg: string) => void) | undefined;

let toolUseCount = 0;
let streamedChars = 0;
let searchCount = 0;
let readCount = 0;
let lastTool = '';

const emit = (done = false) => {
  const tokens = usage?.totalTokens ?? (streamedChars > 0 ? Math.round(streamedChars / 4) : null);
  onProgress?.(
    encodeSubagentProgress({
      toolUseCount,
      tokens,
      activity: done ? 'Done' : activityText(searchCount, readCount, lastTool),
      done,
    }),
  );
};

emit();
for await (const ev of subagent.run(query)) {
  switch (ev.type) {
    case 'tool_start':
      toolUseCount++;
      lastTool = ev.tool;
      if (SEARCH_TOOLS.has(ev.tool)) searchCount++;
      else if (READ_TOOLS.has(ev.tool)) readCount++;
      emit();
      break;
    case 'stream_progress':
      streamedChars += ev.charDelta;  // 累积，不每次 emit
      break;
    case 'done':
      answer = ev.answer;
      usage = ev.tokenUsage;
      break;
    // 其它事件全部 swallow
  }
}
emit(true);
```

**progress payload** 用 `subagent-progress` 前缀 + JSON 编码，跟普通 progress 消息区分开（`src/tools/subagent/progress.ts`）：

```typescript
const PREFIX = 'subagent-progress';

export function encodeSubagentProgress(progress: SubagentProgress): string {
  return PREFIX + JSON.stringify(progress);
}

export function decodeSubagentProgress(message: string): SubagentProgress | null {
  if (!message.startsWith(PREFIX)) return null;
  try {
    const parsed = JSON.parse(message.slice(PREFIX.length)) as SubagentProgress;
    if (typeof parsed.toolUseCount === 'number' && typeof parsed.activity === 'string') return parsed;
    return null;
  } catch { return null; }
}
```

UI 层（`components/subagent-group.ts`）解码后显示：

```
🔧 Analyze NVDA moat        Searched 3×, read 2 sources  [4 tools, ~2.4K tokens]
🔧 Analyze AMD moat         Ran get_financials           [2 tools, ~1.1K tokens]
🔧 Analyze INTC moat        Done                         [7 tools, 4.8K tokens]
```

**注意 stream_progress 不 emit**——它变化太频繁（每个字符一次），如果每次都推给主 agent 会淹掉主 agent 的进度流。只累加到本地计数器，`emit()` 时读取总数。

**activityText 的启发式滚动**：

```typescript
function activityText(searchCount: number, readCount: number, lastTool: string): string {
  if (searchCount + readCount >= 2) {
    const parts: string[] = [];
    if (searchCount > 0) parts.push(`Searched ${searchCount}×`);
    if (readCount > 0) parts.push(`read ${readCount} source${readCount === 1 ? '' : 's'}`);
    return parts.join(', ');
  }
  if (lastTool) return `Ran ${lastTool}`;
  return 'Initializing…';
}
```

**≥ 2 次工具就滚动汇总**（"Searched 3×, read 2 sources"），少于 2 次显示单个工具名。**UI 显示不刷屏也不静默**。

## 七、结果综合：一份 verbatim 答案

Subagent 跑完，主 agent 拿到什么？看 `spawn-subagent.ts` 结尾：

```typescript
for await (const ev of subagent.run(query)) {
  // ...
  case 'done':
    answer = ev.answer;
    usage = ev.tokenUsage;
    break;
}

if (!answer) {
  return `Subagent (${typeKey}) finished without producing an answer.`;
}
return usage
  ? `${answer}\n\n_[subagent ${typeKey}: ${usage.totalTokens} tokens]_`
  : answer;
```

**只返回 `answer`（final 一段 text）**——中间的 tool_start / tool_end / thinking 全部丢弃，主 agent 看不到。

Subagent worker prompt 里明确要求：

```
Your final message is returned verbatim to the orchestrator, so make it a complete,
self-contained answer — state your findings and conclusions directly, not a description
of what you did.
```

**返回值末尾附加 token 消耗**（`_[subagent research: 8432 tokens]_`）——主 agent 能感知每个 subagent 花了多少 token，方便自己做成本判断。

主 agent 视角：3 个 `spawn_subagent` 工具的 tool_end 分别 yield 3 个 answer 字符串，压回 messages 数组：

```
messages = [
  system prompt,
  user query,
  AI: "let me spawn 3 subagents",
  tool_calls: [spawn_subagent(NVDA), spawn_subagent(AMD), spawn_subagent(INTC)],
  ToolMessage(NVDA): "NVDA moat analysis: ... [subagent: 3200 tokens]",
  ToolMessage(AMD):  "AMD moat analysis: ...  [subagent: 2800 tokens]",
  ToolMessage(INTC): "INTC moat analysis: ... [subagent: 4100 tokens]",
]
```

下一轮 LLM 调用时看到这 3 份浓缩答案，做综合分析。**主 agent 只看结论，看不到过程**——这就是 context 保持干净的关键。

## 八、Subagent vs 主 agent 自己调工具

同一个任务两种做法对比：

**主 agent 自己跑**：
```
主 agent messages (10 轮后):
- get_financials(NVDA) → 45KB JSON
- read_filings(NVDA, "moat")  → 30KB text
- get_financials(AMD) → 42KB JSON
- read_filings(AMD, "moat")  → 28KB text
- get_financials(INTC) → 38KB JSON
- read_filings(INTC, "moat") → 25KB text
- ... reasoning between
主 agent 综合分析时 context ≈ 220KB
```

**Subagent 分派**：
```
主 agent messages (3 轮):
- spawn_subagent(NVDA) → 500 words 结论
- spawn_subagent(AMD)  → 500 words 结论
- spawn_subagent(INTC) → 500 words 结论
主 agent 综合分析时 context ≈ 8KB
Subagent 的 220KB context 已经"焚烧"完，只有结论进主 agent
```

**Token 账**：
- 自己跑：主 agent 每次 API 调用输入 220KB × N 次 = 大成本
- Subagent：主 agent 每次输入 8KB × N 次 + subagent 一次性输入各自的 context

**大多数情况 subagent 更省 token**，尤其是"多轮综合分析"占主导时。

**什么时候不该用 subagent**：

- 单次工具调用能搞定（"AAPL 现在市值多少" → 直接 get_market_data，spawn_subagent 是过度设计）
- Sub-task 之间**互相依赖**（一个的输出是另一个的输入）——subagent 见不到彼此，只能串行 chain，那还不如主 agent 自己跑
- 需要**交互**（比如中途要问用户）——subagent 见不到用户，`ask_user_question` 在 disallow list 里

## 九、局限与坑

**1. Task 描述不清 → subagent 空跑**

Subagent 完全靠 `task` 字符串知道要做什么，见不到主 agent 的历史。如果主 agent 传的 task 是 "分析这家公司"，subagent 根本不知道哪家公司——它会瞎猜或者反问（但没法反问用户，可能干脆报错）。

**缓解**：Worker preamble 里明确 "your final message is verbatim returned"，让 LLM 意识到 task 必须自包含。但仍然依赖主 agent 有意识地写清楚。

**2. 并行 = 并行付费**

3 个 subagent 同时跑 = 同时 3 份 API 调用。**wall time 快了，但 API 费用不省**。如果预算敏感，注意别让 LLM 一次 spawn 太多。

**3. 无法共享中间结果**

Subagent A 拉到的 NVDA 财报数据，subagent B 完全不知道。如果 B 也要 NVDA 财报，会重新拉一次。**没有跨 subagent 缓存**（现在没做）。

**4. 单点失败不停止其它**

Subagent A 出错，B 和 C 照跑到底。这一般是好事（失败隔离），但如果任务是"必须三个都成功"，主 agent 得自己判断收到的答案里有没有 error。

**5. Fast model 会算错综合结论**

主 agent 拿到 3 份 subagent 答案后要做综合，这时用的还是主 model（不是 fast model）。如果综合逻辑复杂，主 model 可能算错——subagent 只是数据收集层，综合质量仍取决于主 agent。

## 十、可以拿走的三个原则

1. **委派 = 隔离 + 自包含**：subagent 见不到主对话是特性不是 bug——强迫你把 task 写清楚，也让并发跑不 race。想要"隔离但共享"是自相矛盾的
2. **复用已有并发基础设施**：`concurrencySafe: true` 一个标记就把并行搞定，不新开 promise pool。**"新能力尽量落到已有抽象里"** 是避免代码膨胀的关键
3. **一层深就够**：多层树状 agent 展开听起来很酷，但可预测性、成本可控性、可调试性都会显著下降。业界（Claude Code、Cursor）都收敛到一层深是有原因的

---

## 延伸阅读

- 主文档：[PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md)
- Tool-calling loop：[TOOL_CALLING_LOOP.md](./TOOL_CALLING_LOOP.md)
- 上下文治理：[CONTEXT_MANAGEMENT.md](./CONTEXT_MANAGEMENT.md)
- 源码入口：`src/tools/subagent/spawn-subagent.ts`, `src/tools/subagent/types.ts`, `src/tools/subagent/progress.ts`, `src/tools/registry.ts`
