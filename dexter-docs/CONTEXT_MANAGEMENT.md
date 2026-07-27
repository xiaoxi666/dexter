# Dexter 上下文治理：三层机制

> 主文档：[PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md)
>
> 这份文档专门讲清楚 Dexter 如何"用同一份 message array 把上下文控制住"——microcompact、compaction、truncate 三层机制的触发条件、协作方式、以及配套的落盘和 budget 护栏。这是 Anthropic 官方 blog《Effective context engineering》推荐做法的一个完整开源实现。

## 目录

- [一、要解决的问题](#一要解决的问题)
- [二、总览：五层防线](#二总览五层防线)
- [三、Layer 1：Microcompact（轻量剪枝）](#三layer-1microcompact轻量剪枝)
- [四、Layer 2：Reasoning 剥离](#四layer-2reasoning-剥离)
- [五、Layer 3：Full compaction（LLM 摘要）](#五layer-3full-compactionllm-摘要)
- [六、Layer 3.5：Memory flush（compaction 前落盘）](#六layer-35memory-flushcompaction-前落盘)
- [七、Layer 4：Result 大小护栏](#七layer-4result-大小护栏)
- [八、Layer 5：Overflow 兜底](#八layer-5overflow-兜底)
- [九、协作时序](#九协作时序)
- [十、阈值和参数一览](#十阈值和参数一览)
- [十一、四个可以拿走的原则](#十一四个可以拿走的原则)

---

## 一、要解决的问题

Agent 跑长了，`BaseMessage[]` 会持续膨胀：

- 每轮 LLM 输出 → 一个 `AIMessage`
- 每个 tool call → 一个 `ToolMessage`（工具结果可能是几百 KB 的 JSON）
- 5 轮迭代下来，context 里塞 20+ 条消息，输入 tokens 从 10K 涨到 150K

问题：
1. **成本线性上涨**：每次 API 调用要付所有输入 tokens 的钱
2. **超过 context window 直接报错**：GPT-5 是 1M，Claude 是 200K，但也有边界
3. **超过阈值后模型性能反而变差**（"lost in the middle" 现象）

**Dexter 的答案**：**不删信息，压缩表达**。三层机制层层设防，用最轻的手段先扛，扛不住再上重的。

## 二、总览：五层防线

| 层级 | 触发条件 | 手段 | 信息损失 |
|---|---|---|---|
| Layer 1: microcompact | 每轮开始，ToolMessage > 8 或合计 > 80K tokens | 把旧只读工具结果替换成 `[cleared]` 标记 | 中（旧结果不见了）|
| Layer 2: reasoning 剥离 | 每轮 | 清空旧 AIMessage 的 text content，保留 tool_calls | 极小（只删推理过程）|
| Layer 3: full compaction | API 报 input tokens > effectiveWindow - 33K | fast model 摘要成结构化 summary，替换整个数组 | 小（信息被浓缩但保留）|
| Layer 3.5: memory flush | 与 Layer 3 同时触发，先跑 | 用户偏好/决策落盘到 `.dexter/memory/*.md` | 无（写盘持久化）|
| Layer 4: result 大小护栏 | 单结果 > 50K chars 或本轮合计 > 200K chars | 落盘到 `.dexter/tool-results/*.txt`，只留 preview | 无（落盘可 read_file 回来）|
| Layer 5: overflow 兜底 | LLM 直接报 context overflow | 删除最老 N 轮迭代 | 大（真删）|

**执行顺序**：Layer 4（结果护栏）在每轮工具执行后立刻做；Layer 1（microcompact）和 Layer 2（reasoning 剥离）每轮开始时跑；Layer 3+3.5 是"到阈值才启动"的重手段；Layer 5 是"其它都失败"时的最后手段。

## 三、Layer 1：Microcompact（轻量剪枝）

`src/agent/microcompact.ts`，每轮 LLM 调用前跑：

```typescript
const COUNT_TRIGGER_THRESHOLD = 8;   // 可压缩 ToolMessage > 8 时触发
const COUNT_KEEP_RECENT      = 4;   // 保留最新 4 条
const TOKEN_TRIGGER_THRESHOLD = 80_000;  // 或者合计 > 80K tokens 时触发

const COMPACTABLE_TOOLS = new Set([
  'get_financials', 'get_market_data', 'read_filings', 'stock_screener',
  'web_fetch', 'web_search', 'x_search', 'browser', 'read_file',
  'memory_search', 'memory_get', 'heartbeat', 'cron',
]);
```

核心逻辑：

```typescript
export function microcompactMessages(messages: BaseMessage[]): MicrocompactResult {
  // 找出可压缩的旧 ToolMessage
  const compactableIndices: number[] = [];
  for (let i = 0; i < messages.length; i++) {
    const msg = messages[i];
    if (msg instanceof ToolMessage &&
        COMPACTABLE_TOOLS.has(msg.name ?? '') &&
        typeof msg.content === 'string' &&
        msg.content !== MC_CLEARED_MESSAGE) {
      compactableIndices.push(i);
    }
  }

  // 两种触发方式
  const countTriggered = compactableIndices.length > COUNT_TRIGGER_THRESHOLD;
  const tokenTriggered = !countTriggered && totalTokens > TOKEN_TRIGGER_THRESHOLD;
  if (!countTriggered && !tokenTriggered) return { messages, cleared: 0, ... };

  // 保留最新 4 条，其它替换成标记
  const keepSet = new Set(compactableIndices.slice(-COUNT_KEEP_RECENT));
  const clearIndices = compactableIndices.filter(i => !keepSet.has(i));

  const newMessages = messages.map((msg, i) => {
    if (clearSet.has(i) && msg instanceof ToolMessage) {
      return new ToolMessage({
        content: '[Old tool result content cleared]',  // ← 只留标记
        tool_call_id: msg.tool_call_id,
        name: msg.name,
      });
    }
    return msg;
  });
  // ...
}
```

**关键设计**：

**1. 只处理只读工具**——`COMPACTABLE_TOOLS` 集合白名单里的都是"重复调用会得到同样结果"的只读工具。写入工具（`write_file`、`memory_update`）的结果**永远不清**——因为它们反映了系统状态变化，是 side effect 的凭证，删了 LLM 就没法推理"我做过什么"。

**2. 保留 `tool_call_id` 和 `name`**——这两个字段是 LLM 匹配 tool_call 和 ToolMessage 的凭证，删了会破坏 message 结构、API 会报错。所以只清 `content`，其它保留。

**3. 双触发条件**——单数量（超过 8 条）+ 单 token（超过 80K）：
- **数量触发**处理"很多小结果堆积"的场景（比如 15 次 web_search 每次几百字）
- **token 触发**处理"少数超大结果"的场景（比如一次 read_filings 返回整个 10-K）

**4. 幂等**：已经是 `[Old tool result content cleared]` 的不会被重复清（`msg.content !== MC_CLEARED_MESSAGE`）。

**成本**：几乎为零——就是遍历数组和字符串替换，不调用 LLM。

## 四、Layer 2：Reasoning 剥离

`agent.ts` 里的 `stripOldThinking`，每轮跑：

```typescript
private stripOldThinking(messages: BaseMessage[], keepLast: number): void {
  // 收集所有 AIMessage 的索引
  const aiIndices: number[] = [];
  for (let i = 0; i < messages.length; i++) {
    if (messages[i] instanceof AIMessage) aiIndices.push(i);
  }

  // 只留最新 keepLast 个的 text content
  const toStrip = aiIndices.slice(0, -keepLast);
  for (const idx of toStrip) {
    const msg = messages[idx] as AIMessage;
    // 只清有 tool_calls 的（那些一定有对应 ToolMessage 后续跟着）
    if (msg.tool_calls && msg.tool_calls.length > 0 && msg.content) {
      messages[idx] = new AIMessage({
        content: '',   // ← 清空 text
        tool_calls: msg.tool_calls,  // ← 保留 tool_calls
        // ...
      });
    }
  }
}
```

调用点：`this.stripOldThinking(messages, 2);` —— 保留最近 2 个 AIMessage 的文本，其它清空。

**为什么这么做**：

- LLM 的 text content 里往往是 reasoning（"我要先查一下 AAPL 的现金流，然后..."），对早期迭代来说，这些 reasoning **已经反映在后续动作里了**——没必要再占 context
- 但 `tool_calls` 结构**不能删**——ToolMessage 靠 `tool_call_id` 配对，删了 tool_calls 会破坏 API 契约
- 保留最新 2 个的 reasoning 是为了"上下文连续性"——LLM 需要看到自己最近怎么想的

**成本**：也是零 LLM 开销。

## 五、Layer 3：Full compaction（LLM 摘要）

`src/agent/compact.ts`，重手段——用 fast model 把整个研究过程摘要成结构化 summary。

### 5.1 触发条件

```typescript
const estimatedContextTokens = ctx.lastApiInputTokens > 0
  ? ctx.lastApiInputTokens
  : estimateTokens(messagesText);
const threshold = getAutoCompactThreshold(this.model);

if (estimatedContextTokens <= threshold) return;
```

其中 `getAutoCompactThreshold`（`utils/tokens.ts`）：

```typescript
const AUTOCOMPACT_BUFFER_TOKENS = 13_000;
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000;

export function getAutoCompactThreshold(model: string): number {
  const provider = resolveProvider(model);
  const contextWindow = provider.contextWindow ?? 128_000;
  const effective = contextWindow - MAX_OUTPUT_TOKENS_FOR_SUMMARY;
  return effective - AUTOCOMPACT_BUFFER_TOKENS;
}
```

举例：
- Claude Sonnet：`200K - 20K - 13K = 167K` → 到 167K 触发
- GPT-5.6-sol：`1M - 20K - 13K = 967K`

用**真实 API 报的 input_tokens**（`ctx.lastApiInputTokens`）作为估算，比字符估算准得多。字符估算是 fallback。

### 5.2 Compaction prompt

Prompt 强制 LLM 输出结构化 summary，包含 9 段：

```
1. Original Query and Intent
2. Key Concepts
3. Data Retrieved
4. Errors and Retries
5. Analysis Progress
6. Numerical Data
7. Pending Data Needs
8. Current Work State
9. Recommended Next Steps
```

同时用 `<analysis>` + `<summary>` 双 tag：

```
<analysis>
[draft your thoughts here - will be stripped]
</analysis>

<summary>
[the actual structured summary]
</summary>
```

`<analysis>` 是给 LLM 打草稿用（提升 summary 质量），最后会被 `formatCompactSummary` 剥掉——只保留 `<summary>` 段。

### 5.3 强制不调工具

Prompt 头尾都强调不要调用工具：

```typescript
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.
- Do NOT use any tool calls. You already have all the context you need below.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
`;

const NO_TOOLS_TRAILER = `\n\nREMINDER: Do NOT call any tools. Respond with plain text only —
an <analysis> block followed by a <summary> block.`;
```

而且 `callLlm` 调用时**不 bind tools**——从 API 层面就不给它工具，双保险。

### 5.4 用 fast model

```typescript
const provider = resolveProvider(model);
const fastModel = provider.fastModel ?? model;

const result = await callLlm(prompt, {
  model: fastModel,   // ← 用当前 provider 的 fast 变体
  systemPrompt,
  signal,
});
```

Anthropic 用 Haiku、OpenAI 用 gpt-5.6-luna——**便宜快速**。压缩不需要主模型那种深度推理能力。

### 5.5 摘要成功后替换整个数组

```typescript
private compactMessages(messages, summary, query) {
  return [
    messages[0],  // SystemMessage
    new HumanMessage(`${query}\n\n${summary}`),
  ];
}
```

数组从 20+ 条直接压缩到 **2 条**。Summary 里的引导语（`buildCompactSummaryMessage`）告诉 LLM：

```
This session is being continued from a previous research session that ran out of context.
The summary below covers the data retrieved and analysis performed so far.

[SUMMARY]

Continue working toward answering the query without asking the user any further questions.
Resume directly — do not acknowledge the summary, do not recap what was happening.
Pick up the research as if the break never happened.
```

### 5.6 失败容错

```typescript
export const MAX_CONSECUTIVE_COMPACTION_FAILURES = 3;
export const MIN_TOOL_RESULTS_FOR_COMPACTION = 3;
```

- 少于 3 个 tool results 不 compact（太少了没意义，直接 truncate 更好）
- 连续失败 3 次就放弃 compact，改走 truncate 兜底

#### 5.6.1 `MIN_TOOL_RESULTS_FOR_COMPACTION = 3` 背后的四个理由

这个阈值不是随手拍的，Compaction 的 cost/benefit 曲线在小样本上是**负的**，具体有四个层次：

**1. Compaction 有固定开销，1–2 条摊不平**

Compaction 不是"免费压缩"，它是**再发一次 LLM 请求**（fast model）：

- Prompt 本身有 ~1.5K tokens 的固定包袱（`BASE_COMPACT_PROMPT` 那 9 段 section 说明 + `<analysis>` 引导 + 示例结构）
- 输出还得再生成 `<analysis>` + `<summary>` 两块，output tokens 也不便宜
- 多一次网络 round-trip 的延迟

如果只有 1–2 条 tool result，被压缩的原始内容可能就几 K tokens，**压完的 summary 加上 prompt overhead 未必比原始还小**，纯亏。

**2. 压缩比在小样本上会崩**

Compaction prompt 强制要求 9 个 section（Original Query、Key Concepts、Data Retrieved、Errors and Retries、Analysis Progress、Numerical Data、Pending Data Needs、Current Work State、Recommended Next Steps）。

- 20 条 tool result → 摘成 9 段有意义，压缩比很高
- 2 条 tool result → 硬套 9 段模板，LLM 会**为了填格式生成废话**（"Errors and Retries: None"、"Analysis Progress: 尚未开始分析" 之类），summary 长度和原始差不多，甚至更长

**3. Truncate 在小样本上是无损或近似无损的**

Fallback truncate 的参数：

```typescript
KEEP_TOOL_USES = 5   // truncate 保留最新 5 轮
```

- 只有 1–2 条 tool result 时，truncate 是 **no-op**（全都在保留范围内）
- 就算真的要清，直接删掉最老那一条比 LLM 有损摘要更**可预测**——原始 JSON 保真度 > 自然语言总结（尤其是 Numerical Data，LLM 摘要偶尔会丢数字或改数字）

**4. 什么时候会同时"tool result 少 + token 爆"？答案是：单条巨大**

Autocompact 触发条件是 tokens 逼近 context window。如果只有 1–2 条 result 就把 context 撑爆了，说明**单条 result 巨大**。这种场景下：

- Layer 4 的 `MAX_TOOL_RESULT_CHARS = 50K` 落盘机制应该已经先兜住了
- 如果还没兜住，那这一条就是"污染源"——**扔掉这一条重新拉**比"让 LLM 把这一条摘要"更干净：
  - LLM 只有一个信息源可参考时，反而更容易漏关键数字（没有交叉验证的上下文）
  - agent 下轮可以带着更精确的 args（比如加 `limit`）重新调用

**结论**：`>= 3` 这个阈值本质上是在说 **"compaction 是给'多条 result 的会话状态'做的信息浓缩，不是给'单条巨大 result'做的裁剪工具"**——后者交给 truncate 或 result-storage 更合适。符合本文档十一节 "分层防御，按成本递增" 的原则：**下一层能扛住就不要花上一层的钱**。

## 六、Layer 3.5：Memory flush（compaction 前落盘）

在 Layer 3 触发时**先做一步 memory flush**——把用户长期偏好/决策落盘：

```typescript
if (this.memoryEnabled && shouldRunMemoryFlush({ ... })) {
  yield { type: 'memory_flush', phase: 'start' };
  const flushResult = await runMemoryFlush({ ... }).catch(...);
  yield { type: 'memory_flush', phase: 'end', ... };
}
```

`memory/flush.ts` 里的 prompt 明确要它抓什么：

```
Rules:
- Include durable facts, explicit user preferences, and stable decisions.
- Prioritize capturing personal financial information:
  - Financial goals (retirement targets, savings goals, income targets)
  - Risk tolerance and investment philosophy
  - Portfolio decisions and allocation changes
  - Trade history and the reasoning behind buy/sell decisions
- Do not include temporary tool output, market data, or stock prices.
- If nothing should be stored, reply exactly with NO_MEMORY_TO_FLUSH.
```

Fast model 返回 markdown bullets → 追加到 `.dexter/memory/YYYY-MM-DD.md`。

**为什么在 compaction 前跑**：compaction 会**摘掉所有细节**，如果用户说过一句"我风险偏好保守"埋在第三轮 tool 结果的 reasoning 里，compaction 有可能没抓到。先跑 memory flush 保底——**下次 session 至少能通过 memory_search 找回来**。

**每次 compaction 只 flush 一次**（`alreadyFlushed` 标志）——compaction 失败重跑不重复 flush。

## 七、Layer 4：Result 大小护栏

`utils/tool-result-storage.ts` + `utils/tool-result-budget.ts`：

### 7.1 单结果护栏

每次工具跑完，立即检查大小：

```typescript
export const MAX_TOOL_RESULT_CHARS = 50_000;   // 单结果上限
export const PREVIEW_CHARS = 2_000;

toolMessages = toolMessages.map(tm => {
  const content = ...;
  if (exceedsSizeCap(content)) {
    const { preview, filePath } = persistLargeResult(tm.name, tm.tool_call_id, content);
    return new ToolMessage({
      content: buildPersistedContent(filePath, preview, content.length),
      // ...
    });
  }
  return tm;
});
```

**落盘位置**：`.dexter/tool-results/<sanitized_tool_call_id>.txt`

**上下文里的替换内容**：

```
[Result persisted to /path/to/.dexter/tool-results/call_abc.txt (156 KB)]

Preview:
{"income_statements": [{"ticker":"AAPL","period":"annual","revenue":394328000000...

Use read_file to access the full result if needed.
```

LLM 一看就知道"哦这个结果太大被放盘上了"，需要的话用 `read_file` 拿完整内容。

### 7.2 每轮总量护栏

```typescript
export const MAX_TURN_RESULT_CHARS = 200_000;  // 单轮总量上限

export function enforceResultBudget(toolMessages: ToolMessage[]): ToolMessage[] {
  const totalChars = toolMessages.reduce(...);
  if (totalChars <= MAX_TURN_RESULT_CHARS) return toolMessages;

  // 按大小从大到小落盘，直到总量降到 budget 以下
  const bySize = [...indexed].sort((a, b) => b.content.length - a.content.length);
  let remaining = totalChars;
  const toPersist = new Set<number>();
  for (const entry of bySize) {
    if (remaining <= MAX_TURN_RESULT_CHARS) break;
    toPersist.add(entry.index);
    remaining -= entry.content.length - 2_500;  // 减去原始，加回 preview
  }
  // ... persist the marked ones ...
}
```

**为什么单结果护栏还不够**：并发跑 5 个工具，每个都返回 45K（都没触发单结果护栏），但总共 225K 一下就超了。这个函数按"从大到小"贪心地落盘，直到总量降下来。

### 7.3 三个数值的关系

- **单结果**：50K chars（约 15K tokens）
- **单轮总量**：200K chars（约 60K tokens）
- **microcompact token 触发**：80K tokens（累计的活跃 ToolMessages）

从工具单次调用 → 单轮所有工具 → 长期累积 tool results，三个不同的护栏分别把守。

## 八、Layer 5：Overflow 兜底

即便前面 4 层都尽力了，如果 API 直接报了 context overflow（比如某个 provider 的 window 比我们估的小），最后的兜底：

```typescript
const MAX_OVERFLOW_RETRIES = 2;
const OVERFLOW_KEEP_ROUNDS = 3;

while (true) {
  try {
    const result = yield* this.callModelWithStreaming(messages);
    // ...
    break;
  } catch (error) {
    const errorMessage = ...;
    if (isContextOverflowError(errorMessage) && overflowRetries < MAX_OVERFLOW_RETRIES) {
      overflowRetries++;
      const removed = this.truncateMessages(messages, OVERFLOW_KEEP_ROUNDS);
      if (removed > 0) {
        yield { type: 'context_cleared', clearedCount: removed, keptCount: OVERFLOW_KEEP_ROUNDS };
        continue;   // ← 重试
      }
    }
    // 最终失败
    yield { type: 'done', answer: `Error: ${...}`, ... };
    return;
  }
}
```

`truncateMessages` 只保留最新 3 轮 iteration（AI → Tools → AI → Tools → AI → Tools），前面的全删。最多重试 2 次，都失败就报错退出。

**这是真删信息**——所以是最后手段。前 4 层都是"想办法保留信息"，Layer 5 是"实在不行就删"。

## 九、协作时序

看一个完整循环里各层什么时候跑：

```
每轮 iteration 开始：
  ┌─────────────────────────────────────────────┐
  │  1. Layer 1: microcompact                    │  ← 每轮跑
  │  2. Layer 2: stripOldThinking                │  ← 每轮跑
  │  3. 尝试调 LLM                                │
  │     ├─ 成功 → 继续                            │
  │     └─ overflow error →                       │
  │        Layer 5: truncate + retry (最多 2 次)  │
  │  4. 如果没 tool_calls → done                  │
  │  5. push AIMessage                            │
  │  6. 执行工具（并发/串行）                     │
  │  7. push ToolMessages                         │
  │  8. Layer 4a: 单结果大 → 落盘                 │  ← 立刻
  │  9. Layer 4b: 单轮总量大 → 挑最大的落盘       │  ← 立刻
  │ 10. 检查 API 返回的 lastApiInputTokens        │
  │ 11. 若超过阈值：                              │
  │     Layer 3.5: memory flush (一次性)          │
  │     Layer 3: compaction                       │
  │     ├─ 成功 → messages 压缩成 2 条            │
  │     └─ 失败 → fallback truncate               │
  └─────────────────────────────────────────────┘
```

**优先级设计**：轻的先跑（免费），重的后跑（要钱），删信息的最后跑（不得已）。

## 十、阈值和参数一览

所有可调参数集中列一下（方便对照代码）：

| 参数 | 值 | 位置 | 含义 |
|---|---|---|---|
| `COUNT_TRIGGER_THRESHOLD` | 8 | microcompact.ts | 可压缩 ToolMessage 数量超过此值触发 |
| `COUNT_KEEP_RECENT` | 4 | microcompact.ts | microcompact 保留最新的多少条 |
| `TOKEN_TRIGGER_THRESHOLD` | 80,000 | microcompact.ts | 累计 tokens 超过此值也触发 |
| `keepLast` | 2 | agent.ts:stripOldThinking | 保留最近多少个 AIMessage 的 reasoning |
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | utils/tokens.ts | 距离 context window 多少 tokens 触发 compaction |
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20,000 | utils/tokens.ts | 给 output 预留多少 tokens |
| `MAX_CONSECUTIVE_COMPACTION_FAILURES` | 3 | compact.ts | 连续失败几次放弃 |
| `MIN_TOOL_RESULTS_FOR_COMPACTION` | 3 | compact.ts | 少于几个 tool result 不触发 |
| `MAX_TOOL_RESULT_CHARS` | 50,000 | tool-result-storage.ts | 单个 tool 结果超此大小落盘 |
| `PREVIEW_CHARS` | 2,000 | tool-result-storage.ts | 落盘后 context 里留多长预览 |
| `MAX_TURN_RESULT_CHARS` | 200,000 | tool-result-budget.ts | 单轮所有工具结果合计上限 |
| `MAX_OVERFLOW_RETRIES` | 2 | agent.ts | overflow 时最多重试几次 |
| `OVERFLOW_KEEP_ROUNDS` | 3 | agent.ts | overflow truncate 保留最新几轮 |
| `KEEP_TOOL_USES` | 5 | utils/tokens.ts | compaction fallback truncate 保留几轮 |

这些数值是根据实际经验调出来的，不同 provider、不同 model 可能需要重新调。

## 十一、四个可以拿走的原则

1. **分层防御，按成本递增**：轻的先扛（microcompact 零成本），扛不住再上重的（compaction 花 fast-model tokens），最后才删信息（truncate）。**不要一上来就用最重的手段**
2. **不删信息，压缩表达**：`[cleared]` 标记比"直接从数组里删"好——后者破坏 tool_call_id 配对，前者结构完好但内容归零；落盘 + preview 比"截断内容"好——后者信息丢失，前者可 recover
3. **API 报的 tokens 比字符估算准**：拿 `usage_metadata.input_tokens` 当作决策基准，比 `text.length / 3.5` 精确一个数量级——用真实反馈修正估计误差
4. **配套 fallback 也要清晰**：compaction 会失败（fast model 也会 rate limit）、memory flush 会失败——每一层都有下一层兜底，才不会出现"我以为压完了结果崩了"的失控局面

---

## 延伸阅读

- 主文档：[PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md)
- Tool-calling loop：[TOOL_CALLING_LOOP.md](./TOOL_CALLING_LOOP.md)
- 参考：[Anthropic - Effective context engineering](https://www.anthropic.com/news/prompt-caching)
- 源码入口：`src/agent/microcompact.ts`, `src/agent/compact.ts`, `src/agent/agent.ts:manageContextThreshold`, `src/utils/tool-result-storage.ts`, `src/utils/tool-result-budget.ts`, `src/memory/flush.ts`, `src/utils/tokens.ts`
