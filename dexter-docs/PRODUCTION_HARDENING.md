# Dexter 生产 Hardening：成本、可靠性、可审计

> 主文档：[PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md)
>
> 这份文档从"生产上线视角"梳理 Dexter 里的**横切能力**——成本控制、可审计、多层 fallback、长尾防御，以及**当前明显没做的短板**。前面几份分支文档聚焦"怎么做 agent 内核"，这份聚焦"怎么让这个 agent 能在生产环境跑得住"。

面试里经常问的"prompt injection 防御 / PII 脱敏 / token 成本 / 可审计 / 灾备"这类横切题目，Dexter 有一部分做得很好、一部分刻意没做——这份文档把两边都摊开。

## 目录

- [一、成本控制：五个协同机制](#一成本控制五个协同机制)
- [二、多层 Fallback：三种失败场景各有应对](#二多层-fallback三种失败场景各有应对)
- [三、可审计：Scratchpad JSONL + 事件流](#三可审计scratchpad-jsonl--事件流)
- [四、长尾任务防御：五重保险](#四长尾任务防御五重保险)
- [五、错误分类：可重试 vs 立即失败](#五错误分类可重试-vs-立即失败)
- [六、API 结果本地缓存](#六api-结果本地缓存)
- [七、当前"没做"的短板](#七当前没做的短板)
- [八、五个可以拿走的原则](#八五个可以拿走的原则)

---

## 一、成本控制：五个协同机制

Dexter 在成本上做了 5 层设计，从 API 计费机制到应用层裁剪，层层递进：

### 1.1 Anthropic Prompt Caching（`cache_control`）

`src/model/llm.ts` 里的 `annotateSystemMessageForCaching`：

```typescript
function annotateSystemMessageForCaching(messages: BaseMessage[]): BaseMessage[] {
  if (messages.length === 0 || messages[0]._getType() !== 'system') return messages;

  const systemMsg = messages[0];
  const text = typeof systemMsg.content === 'string' ? systemMsg.content : JSON.stringify(systemMsg.content);

  const annotated = new SystemMessage({
    content: [{
      type: 'text' as const,
      text,
      cache_control: { type: 'ephemeral' },   // ← Anthropic prompt caching
    }],
  });
  return [annotated, ...messages.slice(1)];
}
```

Anthropic 的 prompt caching 机制：给 SystemMessage 打上 `cache_control: { type: 'ephemeral' }`，之后 5 分钟内**同样前缀的请求，输入 tokens 收 10% 的价钱**。

对 Dexter 特别有用，因为：
- System prompt 有 SOUL + rules + tool descriptions + memory context + skill 元数据 = 5000-8000 tokens
- 每次 iteration 都要重发系统 prompt
- 一次 query 可能 10 轮 iteration → 5min 内 90% 命中 → 大部分调用只付 10% 的 system prompt 费用

**只对 Anthropic 做**（`if (provider.id === 'anthropic')`）——OpenAI/Gemini 有自动 prompt caching 无需显式标记；Ollama/其它是本地或按次计费，没这个机制。

### 1.2 Fast model 分级路由

`providers.ts` 里每个 provider 都有 `fastModel`：

```typescript
export const PROVIDERS: ProviderDef[] = [
  {
    id: 'openai',
    fastModel: 'gpt-5.6-luna',   // ← 主模型 gpt-5.6-sol 的便宜快速版
    contextWindow: 1_047_576,
  },
  // Anthropic: fastModel = Haiku
  // Google: fastModel = Gemini Flash
  // ...
];
```

Dexter 里**主模型**（Claude Sonnet / GPT-5.6-sol）负责主决策；**fast model** 承担几类"辅助但高频"的任务：

- **Compaction**：`compactContext` 用 fast model 摘要 20+ 条消息，主模型继续做研究
- **`get_financials` 内部路由**：LLM 决定"这个自然语言 query 用哪个子工具"这一步用 fast model（`callLlm(prompt, { model: fastModel })`）
- **`web_fetch` 摘要**：拉到 HTML 后转 markdown 再用 fast model 摘要
- **Memory flush**：从研究上下文里抽用户偏好用 fast model

**这个模式的收益极大**：主 model 只花在"值得深度推理"的地方，其它便宜跑。类比：主 model 是资深医生，fast model 是助理护士。

### 1.3 上下文治理三层

见 [CONTEXT_MANAGEMENT.md](./CONTEXT_MANAGEMENT.md)。核心思路：

- **Microcompact**（零成本）：旧 tool 结果替换成 `[cleared]` 标记
- **Compaction**（fast model 成本）：用 LLM 把研究摘要成 summary，替换整个消息数组
- **Truncate**（零成本，兜底）：直接删旧轮次

一个长 query 从 200K tokens 压到 30K tokens 是常态，token 成本大幅降。

### 1.4 结果落盘 + 预览

`utils/tool-result-storage.ts`：单个 tool 结果超过 50K chars → 落盘到 `.dexter/tool-results/`，context 里只留 2K 预览 + 文件路径。LLM 需要完整数据用 `read_file` 拿。

**这不是缓存，是"分层加载"**——大结果不占对话历史的 token 预算，但 LLM 想再看还能读。

### 1.5 Tool 结果 budget cap

`utils/tool-result-budget.ts`：单轮所有 tool 结果合计超过 200K chars → 按大小从大到小落盘，直到降下来。防止并发 5 个 tool 每个 45K（都没超单结果护栏但合计爆表）。

**这五层协同**的效果：一个"研究 NVDA 过去 5 年财报 + 新闻 + filings"的复杂 query，如果不做任何优化，10 轮 iteration 的输入 tokens 累计能到 1.5M；启用完整机制后压到 200K 以内，成本降 85%+。

## 二、多层 Fallback：三种失败场景各有应对

### 2.1 LLM streaming → blocking

`agent.ts` 的 `callModelWithStreaming`：

```typescript
private async *callModelWithStreaming(messages) {
  try {
    return yield* this.streamAndAccumulate(messages);
  } catch {
    return await this.callModelWithMessages(messages);   // ← fallback to blocking
  }
}
```

某些 provider 的流式接口偶尔抽风（Ollama 特定版本、xAI Grok 的 streaming + tool_calls 曾有 breaking change）。**不抛错，静默回退到非流式**——用户只是看不到流式动画，但结果照样出。

### 2.2 Web search provider chain

`tools/registry.ts` 里的 web_search：

```typescript
const allWebSearchProviders: WebSearchProvider[] = [];
if (process.env.EXASEARCH_API_KEY) allWebSearchProviders.push({ id: 'exa', ..., tool: exaSearch });
if (process.env.PERPLEXITY_API_KEY) allWebSearchProviders.push({ id: 'perplexity', ..., tool: perplexitySearch });
if (process.env.TAVILY_API_KEY) allWebSearchProviders.push({ id: 'tavily', ..., tool: tavilySearch });
if (process.env.LANGSEARCH_API_KEY) allWebSearchProviders.push({ id: 'langsearch', ..., tool: langSearch });

if (allWebSearchProviders.length > 0) {
  const preferred = getSetting('webSearchPreferredProvider', undefined);
  const orderedProviders = preferred
    ? [...filter first..., ...filter rest...]  // 用户偏好在前，其它 fallback
    : allWebSearchProviders;

  tools.push({
    name: 'web_search',
    tool: createWebSearchTool(orderedProviders),
    // ...
  });
}
```

**支持 4 家 search provider，按用户 `/search` 命令设置的偏好顺序，前一个失败自动 fallback 到下一个**。LLM 只看到一个 `web_search` 工具，provider 差异透明。

### 2.3 Retry with exponential backoff + error classification

`model/llm.ts` 里的 `withRetry`：

```typescript
async function withRetry<T>(fn: () => Promise<T>, provider: string, maxAttempts = 3): Promise<T> {
  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (e) {
      const message = e instanceof Error ? e.message : String(e);
      const errorType = classifyError(message);
      logger.error(`[${provider} API] ${errorType} error (attempt ${attempt + 1}/${maxAttempts}): ${message}`);

      if (isNonRetryableError(message)) {
        throw new Error(`[${provider} API] ${message}`);   // ← 立即抛
      }
      if (attempt === maxAttempts - 1) {
        throw new Error(`[${provider} API] ${message}`);
      }
      await new Promise((r) => setTimeout(r, 500 * 2 ** attempt));   // 500ms → 1s → 2s
    }
  }
  throw new Error('Unreachable');
}
```

关键：**分类错误再决定重试还是立即抛**——见第五节。

## 三、可审计：Scratchpad JSONL + 事件流

### 3.1 每 query 一个 JSONL 文件

`agent/scratchpad.ts` 每次 `Agent.run(query)` 创建一个 append-only 文件：

```
.dexter/scratchpad/
├── 2026-01-30-111400_9a8f10723f79.jsonl
├── 2026-01-30-143022_a1b2c3d4e5f6.jsonl
└── ...
```

文件名格式：`YYYY-MM-DD-HHMMSS_<query_md5前12位>.jsonl`——时间戳 + query hash，方便按时间或按 query 查找。

### 3.2 三种记录类型

每行一条 JSON：

```json
{"type":"init","timestamp":"2026-01-30T11:14:00.100Z","content":"分析 AAPL 的 DCF 估值"}
{"type":"tool_result","timestamp":"2026-01-30T11:14:05.123Z","toolName":"get_financials","args":{"query":"AAPL cash flow"},"result":{...}}
{"type":"thinking","timestamp":"2026-01-30T11:14:06.789Z","content":"我拿到 5 年现金流了，接下来算 CAGR"}
{"type":"tool_result","timestamp":"2026-01-30T11:14:08.456Z","toolName":"read_filings",...}
```

三类事件对应研究过程的三种"痕迹"：
- **init**：原始 query（一次性）
- **tool_result**：完整的工具输入 + 输出（多次）
- **thinking**：LLM 在两次 tool call 之间的推理文本（多次）

**注意存的是完整 args + 完整 result，不是摘要**——这是"审计"的基本要求，摘要就没法验证 LLM 是否根据真实数据推理了。

### 3.3 Append-only + JSONL

**append-only** 意味着写入过程中即使 agent 崩溃、系统重启，已经写入的部分都在盘上；**JSONL 格式**意味着可以流式解析（每行独立解析），也可以用 `grep`/`jq`/`awk` 直接分析。

对比 JSON array 形式：写一半崩了整个文件就损坏了；JSONL 无所谓，最后一行不完整就丢那一行。

### 3.4 用作 Replay 素材

想复现某次研究的过程：读那个 JSONL 文件，就能看到：
- 用户问的是什么
- LLM 依次调了哪些工具、传了什么参数
- 每次工具返回了什么
- LLM 在两次调用之间是怎么想的

面试题 5.5"可解释、可审计、可回溯"的三个维度全覆盖：
- **可解释**：thinking 记录 LLM 的推理
- **可审计**：完整 args + result 落盘
- **可回溯**：文件名带时间戳 + hash，容易定位

对比业界典型做法（`{trace_id, user_id, action, args, result, ts}` 落 ES + Kafka），Dexter 是本地 file-based 版本——**单人 CLI 场景够用，不需要 ES/Kafka 那套重工具**。上升到多租户场景直接把 append 逻辑换成 Kafka producer 就行。

### 3.5 事件流也是审计一部分

除了 scratchpad，agent 还 yield 完整事件流（`AgentEvent` union，20+ 种）。CLI 侧只用来渲染 UI，但 **gateway 侧可以选择把事件流也持久化**——目前 gateway 只做了 debug log（`.dexter/gateway-debug.log`），做完整 audit 需要自己 hook `onEvent`。

## 四、长尾任务防御：五重保险

Agent 循环里最怕两件事：**一直调工具不停**、**一次调用永远不返回**。Dexter 五层设防。

### 4.1 `maxIterations` 硬限（默认 10）

`agent.ts`：

```typescript
const DEFAULT_MAX_ITERATIONS = 10;

while (ctx.iteration < this.maxIterations) {
  ctx.iteration++;
  // ...
}

yield { type: 'done', answer: `Reached maximum iterations (${this.maxIterations}). ...` };
```

**10 轮就是天花板**，subagent 是 8 轮。触及天花板 agent 会 gracefully 报告"没搞完"，而不是无声无息挂掉。

### 4.2 Tool call 软限流（LLM-facing warnings）

`agent/scratchpad.ts` 的 `canCallTool`：

```typescript
const DEFAULT_LIMIT_CONFIG = { maxCallsPerTool: 3, similarityThreshold: 0.7 };

canCallTool(toolName: string, query?: string): { allowed: boolean; warning?: string } {
  const currentCount = this.toolCallCounts.get(toolName) ?? 0;

  if (currentCount >= maxCalls) {
    return {
      allowed: true,   // ← 永远 allowed，只 warning
      warning: `Tool '${toolName}' has been called ${currentCount} times ... consider: 
        (1) trying a different tool, (2) using different search terms, or 
        (3) proceeding with what you have and noting any data gaps.`,
    };
  }

  // Query similarity check
  const similarQuery = this.findSimilarQuery(query, previousQueries);
  if (similarQuery) {
    return { allowed: true, warning: 'This query is very similar to a previous call...' };
  }
}
```

**警告注入到下一轮 prompt 里，LLM 自己看到警告后决定要不要停**：

```typescript
// agent.ts:
const toolUsageWarning = ctx.scratchpad.formatToolUsageForPrompt();
if (toolUsageWarning) {
  messages.push(new HumanMessage(toolUsageWarning));
}
```

**这个"软限流"设计比硬阻断好在**：LLM 有时确实需要多次调用（比如 4 家公司比对）。硬阻断会误伤真实需求；软警告只在"看起来陷入循环"时提醒 LLM 主动降级。

### 4.3 Sub-tool 超时

`tools/finance/utils.ts`：

```typescript
export const SUB_TOOL_TIMEOUT_MS = 15_000;   // 15 秒

export function withTimeout<T>(promise: Promise<T>, ms: number, label?: string): Promise<T> {
  let timer: ReturnType<typeof setTimeout>;
  return Promise.race([
    promise.finally(() => clearTimeout(timer)),
    new Promise<never>((_, reject) => {
      timer = setTimeout(() => reject(new Error(`${label} timed out after ${ms / 1000}s`)), ms);
    }),
  ]);
}
```

`get_financials` 内部并发跑多个子工具时，每个包一层 `withTimeout(promise, SUB_TOOL_TIMEOUT_MS)`。**15 秒不返回的直接 reject，不拖累整体**。返回部分结果也比"一直等"好。

### 4.4 AbortSignal 全链路传递

`AgentConfig.signal` 是 `AbortSignal`，在整个调用链里传：

```typescript
Agent.create({ signal: config.signal, ... })
  → toolExecutor.executeSingleWithId(call, ctx)
    → tool.invoke(args, { signal: this.signal })
      → 底层 fetch(url, { signal })
```

用户在 CLI 按 ESC → controller `abort()` → signal 触发 → **正在跑的工具（包括 web_fetch、LLM streaming）都会响应中断**。不是"标记要停等着看"，是**立即打断底层 HTTP 请求**。

### 4.5 Overflow retry 上限

如果所有 context 管理都失败，API 直接报 overflow，还有兜底：

```typescript
const MAX_OVERFLOW_RETRIES = 2;

while (true) {
  try { ... } catch (error) {
    if (isContextOverflowError(errorMessage) && overflowRetries < MAX_OVERFLOW_RETRIES) {
      overflowRetries++;
      this.truncateMessages(messages, OVERFLOW_KEEP_ROUNDS);
      continue;
    }
    // ...
  }
}
```

**最多重试 2 次**——三次都溢出就放弃，转 `done` 并返回错误信息。防止死循环重试。

### 4.6 Compaction 失败计数

`agent.ts`：

```typescript
private compactionFailures: number = 0;

if (this.compactionFailures < MAX_CONSECUTIVE_COMPACTION_FAILURES && ...) {
  try {
    // ... compact ...
    this.compactionFailures = 0;   // 成功清零
  } catch {
    this.compactionFailures++;   // 累计失败
  }
}
```

**连续 3 次 compaction 失败就放弃 compaction，转 truncate**。防止 fast model 挂了每轮都尝试 compact 每轮都失败的死循环。

这几层组合起来：**永远有边界，永远能退出**。哪怕全世界都在崩，agent 也会走到 `done` 事件而不是无限挂起。

## 五、错误分类：可重试 vs 立即失败

`utils/errors.ts` 里定义了 7 种错误类型：

```typescript
export type ErrorType =
  | 'context_overflow'
  | 'rate_limit'
  | 'billing'
  | 'auth'
  | 'timeout'
  | 'overloaded'
  | 'unknown';
```

对应的分类规则用**多正则 + 关键字**匹配：

```typescript
const ERROR_PATTERNS = {
  rateLimit: [
    /rate[_ ]limit|too many requests|429/i,
    'model_cooldown',
    'quota exceeded',
    'tokens per minute',
    // ...
  ],
  overloaded: [
    /overloaded_error|"type"\s*:\s*"overloaded_error"/i,
    'service unavailable',
    'high demand',
    // ...
  ],
  timeout: ['timeout', ...],
  // ...
};
```

**分类决定重试策略**（`isNonRetryableError`）：

- **可重试**（`rate_limit` / `overloaded` / `timeout`）：走 exponential backoff 重试 3 次
- **不可重试**（`billing` / `auth`）：立即抛错——没钱了、密钥错了，重试也没用
- **上下文溢出**（`context_overflow`）：走 overflow retry 分支（先 truncate 再重试）

**关键点**：**在错误分类的基础上决定策略，而不是"所有错误都重试 3 次"**。看似小事，但避免了：
- 密钥错误还傻乎乎重试 3 次浪费时间
- 账单欠费还继续尝试触发更多告警
- Rate limit 后不 backoff 直接重试触发更严重的 rate limit

## 六、API 结果本地缓存

`utils/cache.ts` 是一个 opt-in 的本地 JSON 缓存：

```typescript
const CACHE_DIR = dexterPath('cache');

// 用法（在 tools/finance/api.ts 里）：
if (cacheable) {
  const cached = readCache(endpoint, params, ttl);
  if (cached) return cached;
}
const result = await fetchFromAPI(url);
if (cacheable) writeCache(endpoint, params, result);
```

**特性**：

- **按 endpoint + params 生成缓存 key**（md5 hash，参数按 key 排序保证一致）
- **文件名带 ticker**：`prices/AAPL_a1b2c3d4e5f6.json`——人肉可读
- **TTL 分级**：15 min / 1 hour / 6 hours / 24 hours 四档
- **纯存储层**：调用方决定"这个 endpoint 值不值得缓存 + 缓多久"

**为什么本地文件而不是 Redis**：Dexter 是**单机 CLI 应用**，用户运行在自己的机器上。Redis 需要另开服务，本地文件零配置。上升到多用户场景直接把 `readFileSync/writeFileSync` 换成 Redis 客户端就行。

**缓存粒度**：财报数据一年更新一次，用 24H TTL 完全够；实时价格用 15 分钟；SEC filings 用 24H。**按数据变化频率给 TTL**，不搞一刀切。

## 七、当前"没做"的短板

对照面试题的 5.1-5.16，Dexter **明显没做**的部分——这些不是 bug，是 scope 之外的取舍：

### 7.1 Prompt Injection 防御（几乎没有）

- **没有** injection classifier
- **没有** 输入长度硬限
- **没有** chat template 关键字转义（`<|im_start|>`、`[INST]` 等）
- **没有** XML 包裹用户输入

原因：Dexter 是**单人 CLI 工具**，用户就是自己给自己写 prompt——注入自己没意义。上升到多租户场景（比如把 gateway 开放给团队用）就必须加。

### 7.2 PII 脱敏（没做）

- **没有** PII detector
- Memory 系统会存**用户主动说的**信息（投资目标、风险偏好），但没有 masking
- 日志（scratchpad）会**完整记录** query 和 tool result，包括用户提到的任何信息

原因同上：单人使用没脱敏需求。做多租户 SaaS 必须补上 Presidio 或类似方案。

### 7.3 输出审核 / 毒性检测（没做）

- **没有** 敏感词过滤
- **没有** 输出内容 classifier

原因：金融研究场景不太产生毒性内容；国内合规如果要过阿里绿网/腾讯天御是另一层适配。

### 7.4 主备灾备（部分做了）

- **做了**：LLM streaming → blocking fallback、search provider chain、错误分类重试
- **没做**：跨 provider 主备（比如 Anthropic 挂了自动切 OpenAI）——用户当前必须 `/model` 命令手动切
- **没做**：跨 Region 冷备（单机没这需求）

### 7.5 分布式限流（没做）

- 单机应用没有多实例限流问题
- 但**面向 LLM API** 也没有 client-side rate limit——完全依赖 provider 返回 429 后 backoff
- 用户如果并发发很多 query 可能触发 provider 限流

### 7.6 输出流式内容审核（没做）

- 内容边生成边审查、命中即 abort + retract——完全没做

**总结**：Dexter 的定位是"个人研究工具"，把工程能量花在**agent 核心机制**（loop、context、tools、subagent、skills、permissions），面向多租户/公开服务的横切能力（injection defense、PII、审核、灾备）刻意留白。**看它做了什么和没做什么，能反推出它的设计定位**。

如果要把 Dexter 改造成多租户 SaaS 版，最优先要补的是：
1. Prompt injection defense（输入侧）
2. PII masking（进 LLM 前 + 进日志前）
3. Provider 层灾备（主备切换）
4. 输出审核（合规红线）
5. 分布式 rate limit（防单用户刷量）

## 八、五个可以拿走的原则

1. **成本控制要在多个层面同时做**：Anthropic prompt caching（provider API 特性）+ fast model 分级路由（模型能力分级）+ 上下文治理（应用层裁剪）+ 结果落盘（存储换 token）——**四条独立技术栈叠加才有 80%+ 的降本**，单一手段效果都有限

2. **Fallback 要"分类而不是无脑重试"**：`classifyError` + `isNonRetryableError` 让 auth/billing 立刻抛错，rate_limit/overloaded 才 exponential backoff。**看似小事，实际决定用户拿到的错误消息有没有价值**

3. **审计的最小可行版本是 append-only JSONL**：单机场景不需要 Kafka + ES，一个 `appendFileSync` 就能满足 replay 需求。上升到多用户场景再换 pipeline，但**审计的数据模型（init/tool_result/thinking 三元组）不用变**

4. **长尾防御要多层，每层都可独立触发**：maxIterations、tool 软限流、sub-tool timeout、AbortSignal、overflow retry limit、compaction failure counter——**六个独立防线，任何一个触发都能导向 gracefully 结束**。不要指望单一机制兜住所有情况

5. **明知没做也要文档化**：Dexter 没做 PII、injection defense、输出审核——这不丢人。**明确写在 README 里定位为"个人工具"**，让用户知道用它做多租户会有什么风险，比"什么都说做了但都做得半吊子"强得多

---

## 延伸阅读

- 主文档：[PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md)
- Tool-calling loop：[TOOL_CALLING_LOOP.md](./TOOL_CALLING_LOOP.md)
- 上下文治理：[CONTEXT_MANAGEMENT.md](./CONTEXT_MANAGEMENT.md)
- 权限引擎：[PERMISSION_ENGINE.md](./PERMISSION_ENGINE.md)
- 源码入口：`src/model/llm.ts`, `src/utils/errors.ts`, `src/utils/cache.ts`, `src/agent/scratchpad.ts`, `src/tools/finance/utils.ts`, `src/providers.ts`
