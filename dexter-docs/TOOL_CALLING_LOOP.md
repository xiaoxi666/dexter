# Dexter 干净的 Tool-Calling Loop 详解

> 主文档：[PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md)
>
> 这份文档聚焦"tool-calling loop 内核机制"——generator/yield 基础与进阶、loop 主干逐层拆解、并发合流器 `all()` 的实现、streaming chunk 合并。这是主文档第 3 节"Agent 实现思路"的技术展开。

## 目录

- [一、Generator 与 yield](#一generator-与-yield)
- [二、干净的 tool-calling loop（详解）](#二干净的-tool-calling-loop详解)
- [三、`all(generators, maxConcurrency)` 合流实现](#三allgenerators-maxconcurrency-合流实现)
- [四、Streaming chunk 合并的细节](#四streaming-chunk-合并的细节)

---

## 一、Generator 与 yield

### 1.1 yield 是什么

`yield` 是 JavaScript/TypeScript **generator 函数**里的关键字，意思是"**暂停这个函数，把一个值抛出去，等外面处理完再回来继续跑**"。

普通函数：

```typescript
function normal() {
  return 1;
  return 2;  // ← 永远执行不到
}
```

Generator 函数（注意 `function*`）：

```typescript
function* gen() {
  yield 1;   // 暂停，抛出 1
  yield 2;   // 从上次暂停处继续，暂停，抛出 2
  yield 3;
}

const g = gen();
console.log(g.next()); // { value: 1, done: false }
console.log(g.next()); // { value: 2, done: false }
console.log(g.next()); // { value: 3, done: false }
console.log(g.next()); // { value: undefined, done: true }
```

关键点：generator 函数被调用**不会立即执行函数体**，而是返回一个"迭代器"。每次调 `.next()` 才跑到下一个 `yield`，然后又停住。

### 1.2 生活化的比喻

- 普通函数 = 电影，一次性播完
- Generator = 电视剧，播一集停一下，你说"下一集"（`.next()`）它才继续

`yield 1` 就是"播完这一集，屏幕上打出 1，等你继续按遥控器"。

### 1.3 配合 for...of 和 async 版本

```typescript
function* countTo(n: number) {
  for (let i = 1; i <= n; i++) yield i;
}

for (const num of countTo(3)) {
  console.log(num);  // 依次打出 1、2、3
}
```

Dexter 用的是 **async generator**（异步版本）：

```typescript
async function* fetchPages() {
  yield await fetch('page1');
  yield await fetch('page2');
}

for await (const page of fetchPages()) {  // 注意是 for await
  console.log(page);
}
```

### 1.4 `yield` 在 Dexter 的具体作用

```typescript
async *run(query: string): AsyncGenerator<AgentEvent> {
  // ...
  yield { type: 'tool_start', tool: 'get_financials', args: {...} };
  const result = await tool.invoke(args);
  yield { type: 'tool_end', tool: 'get_financials', result };
}
```

翻译成大白话：

1. Agent 跑到 `yield { type: 'tool_start', ... }` 时**暂停**，把事件对象抛给外面
2. 外面（UI 层）拿到事件，显示"正在调用 get_financials..."
3. 外面消费完，**回来继续跑** Agent 的下一行
4. `await tool.invoke(args)` 执行完
5. 再次遇到 `yield { type: 'tool_end', ... }`，把结果事件抛出去

UI 侧的消费代码：

```typescript
for await (const event of agent.run(query)) {
  switch (event.type) {
    case 'tool_start':  showLoadingRow(event.tool); break;
    case 'tool_end':    updateRowWithResult(event.tool, event.result); break;
    case 'done':        showFinalAnswer(event.answer); return;
  }
}
```

Agent 跑一步 → UI 更新一次 → Agent 再跑一步 → UI 再更新，完全交替进行。这就是"实时流式 UI"的实现方式。

### 1.5 `yield*`（带星号）：透传另一个 generator

`yield*` 是"把另一个 generator 抛出的所有值原样透传"。等价于：

```typescript
for await (const event of this.callModelWithStreaming(messages)) {
  yield event;
}
```

写成 `yield* this.callModelWithStreaming(messages)` 更简洁。

一个具象例子：

```typescript
async function* inner() {
  yield 'a';
  yield 'b';
}

async function* outer() {
  yield 'start';
  yield* inner();     // ← 把 inner 抛出的 'a' 'b' 全部透传
  yield 'end';
}

// 消费 outer 会依次拿到：'start', 'a', 'b', 'end'
```

在 Dexter 里，`run()` 里的 `yield*` 让内层的流式进度事件、工具事件、compaction 事件都能通过同一个出口流出去，UI 侧不用关心"这个事件是 run() 直接产生的还是嵌套函数产生的"。

### 1.6 为什么 Dexter 用 generator 而不是 callback / EventEmitter

三种写法对比同一个功能——"每跑一步通知 UI"：

**方式 A：callback**
```typescript
agent.run(query, {
  onToolStart: (tool) => { ... },
  onToolEnd: (tool, result) => { ... },
});
```
问题：多个 callback 就是"回调地狱"，类型也散。

**方式 B：EventEmitter**
```typescript
agent.on('tool_start', (e) => { ... });
agent.run(query);
```
问题：事件类型不容易保证类型安全，生命周期不清晰。

**方式 C：Async Generator（Dexter 的选择）**
```typescript
for await (const event of agent.run(query)) {
  // event 是 AgentEvent union，TypeScript 帮你穷举所有分支
}
```
优点：
- **类型完全穷举**（TS 会检查 switch case 有没有漏）
- **消费者拿控制权**——想中断就 `break`
- **AbortSignal 天然配合**（generator 被丢弃时清理逻辑会跑）
- **嵌套用 `yield*` 一行搞定**

### 1.7 进阶：yield 是双向管道（`.next(value)`）

前面讲的都是"yield 送出去、外面消费"的单向用法。其实 `yield` 表达式**本身有返回值**——外面调 `.next(value)` 传进来的 `value`，会变成 generator 里那次 `yield` 表达式的结果。

最小例子：

```typescript
function* dialog() {
  const name = yield '你叫什么名字？';
  const age  = yield `你好 ${name}，你多大了？`;
  return `${name} 今年 ${age} 岁`;
}

const g = dialog();
console.log(g.next().value);        // "你叫什么名字？"（第一次 next 参数被忽略）
console.log(g.next('小明').value);   // "你好 小明，你多大了？"（'小明' 赋给了 name）
console.log(g.next(18).value);      // "小明 今年 18 岁"
```

一次 `yield` 往返 = **一送一收**。方向图：

```
     外面                           Generator 内部
                          ▲
                          │  yield <出参>
     next(<入参>) ──────► │  ⇦ .next 的 <入参> 变成 yield 表达式的结果
                          ▼
                     暂停在这一行
```

**这个特性是 `async/await` 的底层原理**：早在 async/await 出现之前，社区用 generator + Promise 就能"模拟"出来——generator yield 一个 Promise，外面等 Promise resolve 后通过 `.next(result)` 把结果送回去，generator 里 `const x = yield promise` 就拿到结果。`async function` 本质上就是引擎自动帮你跑的 generator + Promise runner。

除了 `.next(value)`，还有 `.throw(err)`（在 yield 位置抛异常）和 `.return(value)`（强制结束 generator）——三种"入口"让外面完全控制 generator 的执行、异常、终止路径。

典型用法是 **effects 模式**（redux-saga 的核心思想）：generator **描述**流程，外面**执行** side effect：

```typescript
function* pipeline() {
  const data  = yield { action: 'fetch',   url: '/api/users' };
  const users = yield { action: 'parse',   body: data };
  const stats = yield { action: 'analyze', users };
  return stats;
}
```

外面拿 `pipeline` 抛出的 command 去做实际的 fetch/parse/analyze，再把结果 `.next(result)` 回来。业务逻辑（generator）和 side effect（外面）完全分离，非常好测试。

**但 Dexter 完全没用双向通信**。原因：

1. **Agent → UI 是单向数据流**：yield 出事件即可，不需要往回送值
2. **类型系统弱**：`Generator<TYield, TReturn, TNext>` 里 `TNext` 只有一个类型，Dexter 的异构事件不适合
3. **`for await...of` 不支持传值**：一旦用双向就必须手动 `while + next()`，代码变冗长
4. **回调更清晰**：需要"往回送值"的场景（审批、问答）用命名回调（`requestToolApproval: (req) => Promise<Decision>`）比匿名的 `.next(value)` 表意强得多

**记住这个能力就好，事件流场景一般不需要用**——但理解它能帮你看懂 async/await 底层机制、redux-saga 类库，以及为什么 generator 是"最通用的可暂停函数"。

---

## 二、干净的 tool-calling loop（详解）

先给结论：Dexter 的 loop 让人觉得"干净"，不是因为写得花哨，而是因为它**克制**——只用了 4 个数据结构、2 个对外函数、1 个 while 循环，就把 tool-calling agent 该有的能力串起来了。

### 2.1 大局观

```
                       ┌───────────────────────────────────┐
   query ──►  Agent.run() ─── async generator ── AgentEvent ─┼──►  UI (controller)
                       │                                   │
                       │     ┌─────────────────────────┐   │
                       │     │  while (iteration<max)  │   │
                       │     │                         │   │
                       │     │  1. call LLM (stream)   │◄──┼── model/llm.ts
                       │     │  2. tool_calls?         │   │
                       │     │       no → done         │   │
                       │     │       yes ↓             │   │
                       │     │  3. execute tools ──────┼───┼── AgentToolExecutor
                       │     │  4. push ToolMessages   │   │
                       │     │  5. manage context      │   │
                       │     └─────────────────────────┘   │
                       └───────────────────────────────────┘
```

**只有一个循环、一个消息数组、一个执行器**。没有 LangGraph 的节点图，没有 chain of chains，没有 StateGraph。

### 2.2 循环骨架（`src/agent/agent.ts` 精简版）

把 `run()` 剥到骨头：

```typescript
async *run(query: string): AsyncGenerator<AgentEvent> {
  const ctx = createRunContext(query);
  let messages: BaseMessage[] = [
    new SystemMessage(this.systemPrompt),
    new HumanMessage(query),
  ];

  while (ctx.iteration < this.maxIterations) {
    ctx.iteration++;

    // 1. 调 LLM（流式，可 yield 进度事件）
    const { response, usage } = yield* this.callModelWithStreaming(messages);

    // 2. 没有 tool_calls → 就是最终答案
    if (!hasToolCalls(response)) {
      yield { type: 'done', answer: extractTextContent(response) ?? '', ... };
      return;
    }

    // 3. 把 AIMessage 塞回消息数组
    messages.push(response);

    // 4. 执行工具 → 收 ToolMessage[]
    const { toolMessages, denied } = yield* this.executeToolsAndCollectMessages(response, ctx);
    messages.push(...toolMessages);

    if (denied) { yield { type: 'done', ... }; return; }

    // 5. 上下文管理（可能改写 messages）
    yield* this.manageContextThreshold(ctx, query, ..., { messages });
  }

  yield { type: 'done', answer: 'Reached maximum iterations…', ... };
}
```

**教科书 ReAct 循环**：LLM → 是否要工具 → 工具 → 结果回喂 → 再问 LLM。所有花活（compaction / microcompact / permission / concurrent / streaming）都是**挂在这个骨架上的旁支**，从来不改变骨架本身。

### 2.3 把 event stream 当"骨架"

`Agent.run()` 不是一个返回 `Promise<Result>` 的普通函数，而是一个 **async generator**——它一边推进主循环，一边通过 `yield` 把过程中发生的每一件事作为事件抛给消费者。这个选择是 Dexter 能"一个 agent 内核接多种前端"的根本原因。

```typescript
// src/agent/agent.ts:128
async *run(query: string, inMemoryHistory?: InMemoryChatHistory): AsyncGenerator<AgentEvent> {
  yield { type: 'thinking', message: 'Analyzing…' };
  ...
  const result = yield* this.callModelWithStreaming(messages);   // ← 嵌套 yield
  ...
  yield { type: 'done', answer, toolCalls, iterations, ... };
}
```

消费者用 `for await` 一条一条拿事件出来处理：

```typescript
for await (const event of agent.run(query)) {
  switch (event.type) { ... }
}
```

事件类型在 `src/agent/types.ts` 里定义为 **discriminated union**，一共 16 种：

```typescript
export type AgentEvent =
  | ThinkingEvent | ToolStartEvent | ToolProgressEvent
  | ToolEndEvent  | ToolErrorEvent | ToolApprovalEvent
  | ToolDeniedEvent | ToolLimitEvent
  | ContextClearedEvent | MicrocompactEvent | CompactionEvent
  | QueueDrainEvent | MemoryRecalledEvent | MemoryFlushEvent
  | StreamProgressEvent | DoneEvent;
```

每个事件都带一个 `type` 字段做判别标签。TS 会根据它自动收窄类型——在 `event.type === 'tool_end'` 分支里 `event.result` / `event.duration` 就是有类型且可访问的字段，不需要 `as` 断言。

为什么选 generator 而不是 EventEmitter 或者 callback？三个具体好处，都很关键：

#### 好处 1：UI 完全解耦，同一内核接不同前端

`agent.ts` 只管 yield 事件，**不知道**消费者是谁。目前同一份 `Agent.run()` 已经有三种截然不同的消费方式：

| 消费者 | 位置 | 关心的事件 | 用途 |
|---|---|---|---|
| **CLI TUI** | `src/cli.ts` | 全部 16 种 | Ink 实时渲染工具行、思考、上下文压缩横幅…… |
| **WhatsApp 网关** | `src/gateway/agent-runner.ts` | 主要 `done` | 拿到最终答案后回帖到聊天窗 |
| **父 Agent（Subagent 工具）** | `src/tools/subagent/spawn-subagent.ts` | 只挑 `tool_start` / `tool_error` / `stream_progress` / `done` | 折叠成一行 `Researching (5 searches, 3 reads)` 再向上 yield |

对比两个消费者的代码就能看出解耦的力度：

```typescript
// CLI：细粒度渲染，几乎每种事件都要显示
if (event.type === 'thinking')        { updateSpinner(event.message); }
if (event.type === 'tool_start')      { addToolRow(event); }
if (event.type === 'tool_end')        { finalizeToolRow(event); }
if (event.type === 'context_cleared') { showBanner(`Cleared ${event.clearedCount} results`); }
if (event.type === 'compaction')      { showBanner('Compacting…'); }
// ...

// WhatsApp：只等最终结果
for await (const event of agent.run(req.query, session?.history)) {
  await req.onEvent?.(event);
  if (event.type === 'done') finalAnswer = event.answer;
}
```

WhatsApp 端**不需要知道** `microcompact` / `stream_progress` / `memory_flush` 这些事件的存在——`switch` 里没写它们，就自动被忽略。**agent 内核以后新增事件类型也无需修改任何 UI 端代码**——不感兴趣的消费者天然透明。

换成 EventEmitter，每个消费者要注册十几个 `on('xxx', ...)` handler；换成 constructor 里的 callback，agent 构造函数得吞下十几个 `onXxx` 回调；换成"回调塞进 config 对象"就更不用说了。**Generator 的类型窄化 + 消费者自选订阅**让这层"接口"零成本。

#### 好处 2：嵌套 yield（`yield*`）把子流程的事件透传上来

Agent 内部有很多阶段本身也是 generator：LLM 流式响应、工具批量执行、上下文压缩、子 agent 派生。它们通过 `yield*` 把自己的事件直接接到主循环的输出流上：

```typescript
// agent.ts:168 — 主循环
const result = yield* this.callModelWithStreaming(messages);

// agent.ts:298 — 子 generator，返回类型带 return 值
private async *callModelWithStreaming(messages): AsyncGenerator<
  StreamProgressEvent,                                     // ← yield 的事件类型
  { response: AIMessage; usage?: TokenUsage }              // ← return 的结构化值
> {
  return yield* this.streamAndAccumulate(messages);        // 再嵌一层
}

// agent.ts:311 — 最内层：真正 yield 出流式进度
for await (const chunk of streamLlmWithMessages(messages, { model, tools, signal })) {
  const { charDelta, mode } = inspectChunkContent(chunk);
  yield { type: 'stream_progress', charDelta, mode };
}
```

`yield*` 的语义有两层：**把这个子 generator 剩下所有 yield 的事件都当外层自己 yield 的一样往上抛**，同时**把子 generator return 的值作为本表达式的求值结果**。所以外层 `for await (const event of agent.run())` 收到的 `stream_progress` 事件，跟主循环里直接 `yield { type: 'thinking' }` 抛出的事件走的是同一条通道——中间**没有任何 handler / 转发函数 / 事件桥**。

同样的模式在 agent 里用了好几处：

- `executeToolsAndCollectMessages()`（`agent.ts:388`）—— yield 工具执行的 `tool_start` / `tool_end` / `tool_error`，同时 return `{ toolMessages, denied }` 给主循环拼消息
- `manageContextThreshold()`（`agent.ts:457`）—— yield `context_cleared` / `compaction` 事件，同时 return `void`

**`AsyncGenerator<YieldType, ReturnType>` 这个双参数签名让子流程一边流式吐事件、一边给调用者返回结构化结果**——这是 EventEmitter 模型给不了的组合，也是回调地狱的根本解药。

#### 好处 3：中断能力天然

`AgentConfig.signal: AbortSignal` 从入口一路穿透到最底层 IO，不经过任何"取消状态位"：

```
外层（用户 Ctrl+C）
     │  controller.abort()
     ▼
AgentConfig.signal
     │
     ▼
this.signal          （agent.ts:47）
     │  显式带进每次 LLM / 工具调用
     ▼
streamLlmWithMessages(messages, { signal })    ← LangChain 传给底层 fetch
tool.invoke(args, { signal })                   ← 工具内部照样往下传
     │
     ▼
底层 IO（都 `if (signal.aborted) throw`）
  shell-runner.ts:162
  tools/fetch/utils.ts:352
  gateway/whatsapp/runtime.ts:31
```

一旦 `abort()` 触发：

1. 正在跑的 `fetch` 立刻 reject（`AbortError`），LLM 流被撬断
2. 工具里的 `await ...` 全部 reject 向外抛
3. 消费者 `for await (const event of agent.run(...))` 循环里下一次 `.next()` 直接抛异常
4. 上层 catch 到 `AbortError`（`controllers/agent-runner.ts:217`）走取消分支

关键在于——**generator 的 `for await` 循环本身就是可取消的**：外层不用主动 poll "还要不要继续跑"，只要底层 IO 被 signal 撬断，异常会顺着 `yield` 链条向上冒泡直到消费者。相比 EventEmitter（事件发出去就收不回来了，没法取消未来的事件），generator 的暂停–恢复语义让"停下来"变成一件不需要额外协议的事情。

---

**小结**：一条 `AsyncGenerator<AgentEvent>` 同时承担了三件事——**流式输出**（yield 逐个事件）、**结构化返回**（子 generator 的 return 值）、**协作式中断**（signal 撬断底层 IO 时异常沿 yield 链上抛）。这也是为什么 [多通道复用](./MULTI_CHANNEL.md) 一章能这么薄：所有通道差异都被压进了"事件消费者"这层，agent 内核一行都不用改。

### 2.4 消息数组是唯一的可变状态

```typescript
let messages: BaseMessage[] = [...];   // ← 唯一的 mutable 主状态
const ctx = createRunContext(query);   // ← scratchpad + tokenCounter + iteration
let overflowRetries = 0;                // ← 局部小计数器
```

**没有 state machine，没有 store，没有中间态字典**。消息数组本身就是完整的对话历史 + 工具轨迹（LangChain 的 `BaseMessage[]` 已经涵盖了 `SystemMessage` / `HumanMessage` / `AIMessage`(带 tool_calls) / `ToolMessage`）。

对比 LangGraph：LangGraph 让你显式定义 State schema 和节点转换。功能上更强，但对"就是个 ReAct loop"的场景是**过度抽象**——你要写 30 行 boilerplate 才能开始 tool-calling。Dexter 写了 30 行就跑起来了。

### 2.5 LLM 抽象只暴露两个函数

`src/model/llm.ts` 里 380 行代码，对外只暴露：

```typescript
export async function callLlm(prompt, opts): Promise<{ response; usage? }>;
export async function* streamLlmWithMessages(messages, opts): AsyncGenerator<AIMessageChunk>;
export async function callLlmWithMessages(messages, opts): Promise<{ response; usage? }>;
```

内部用 factory map 按 prefix 路由：

```typescript
const MODEL_FACTORIES: Record<string, ModelFactory> = {
  openai:    (name, opts) => new ChatOpenAI({ ... }),
  anthropic: (name, opts) => new ChatAnthropic({ ... }),
  google:    (name, opts) => new ChatGoogleGenerativeAI({ ... }),
  ollama:    (name, opts) => new ChatOllama({ ... }),
};
```

Agent 完全不知道自己在调哪个 provider。**没有 `LLMInterface`、`ProviderAdapter`、`ModelRegistry` 这种一层套一层的接口**——一个 factory map + 三个函数就够。

### 2.6 流式调用里的精妙细节

```typescript
private async *callModelWithStreaming(messages) {
  try {
    return yield* this.streamAndAccumulate(messages);
  } catch {
    return await this.callModelWithMessages(messages);  // fallback to blocking
  }
}
```

内层 `streamAndAccumulate`：

```typescript
let accumulated: AIMessageChunk | null = null;
for await (const chunk of streamLlmWithMessages(messages, ...)) {
  accumulated = accumulated ? accumulated.concat(chunk) : chunk;
  const { charDelta, mode } = inspectChunkContent(chunk);
  yield { type: 'stream_progress', charDelta, mode };
}

return {
  response: new AIMessage({
    content: accumulated.content,
    tool_calls: accumulated.tool_calls,
    ...
  }),
  usage: ...,
};
```

**关键点**：

1. LangChain 的 `AIMessageChunk.concat()` 会自动合并流式片段（包括 tool_calls JSON 的增量拼接），你不用手写解析器
2. 流式失败自动回退到 blocking——覆盖那些流式 API 不稳的 provider
3. `inspectChunkContent` 把不同 provider 的 chunk 结构（OpenAI 是 string，Anthropic 是 typed array 带 thinking/tool_use/text 分片）**统一映射成一个 `mode` 字段**，UI 只关心 mode 不关心 provider

### 2.7 Tool executor：分批的智慧

`src/agent/tool-executor.ts` 里 `partitionToolCalls`：

```typescript
private partitionToolCalls(toolCalls: ToolCall[], ctx: RunContext): ToolCallBatch[] {
  const batches: ToolCallBatch[] = [];
  for (const call of toolCalls) {
    const isSafe = this.concurrencyMap.get(call.name) ?? false;
    const lastBatch = batches[batches.length - 1];
    if (isSafe && lastBatch?.concurrent) {
      lastBatch.calls.push(call);   // 追加到当前并发批
    } else {
      batches.push({ concurrent: isSafe, calls: [call] });  // 开新批
    }
  }
  return batches;
}
```

举例：LLM 一次性返回 5 个 tool_calls：

```
[web_search, get_financials, read_filings, write_file, get_market_data]
     ✓            ✓                ✓             ✗           ✓
```

分成三批：

```
Batch 1: [web_search, get_financials, read_filings]  → 并发
Batch 2: [write_file]                                  → 串行（需审批）
Batch 3: [get_market_data]                             → 单独
```

**分批而不是全并发**的原因：写文件、bash、ask_user_question 这些要**审批**或**串行 side-effect** 的工具混在里面时，你不能一起跑。分批是最简单的解法。

### 2.8 单个工具执行：一个函数收编所有关切

`executeSingleWithId` 大约 60 行，囊括了工具执行的所有生命周期：

```typescript
private async *executeSingleWithId(call, ctx) {
  // ① 权限门
  const permission = evaluatePermission({ tool: call.name, args: call.args });
  if (permission.mode === 'deny') { yield { type: 'tool_denied', ... }; return; }
  if (permission.mode === 'ask') {
    const decision = await this.requestToolApproval?.({ ... }) ?? 'deny';
    yield { type: 'tool_approval', ..., approved: decision };
    if (decision === 'deny') { yield { type: 'tool_denied', ... }; return; }
    if (decision === 'allow-session') this.sessionApprovedTools.add(sessionKey);
    if (decision === 'allow-always') addRule('allow', permission.proposedRule);
  }

  // ② 软限流警告（不阻断）
  const limitCheck = ctx.scratchpad.canCallTool(call.name, ...);
  if (limitCheck.warning) yield { type: 'tool_limit', warning, blocked: false };

  yield { type: 'tool_start', tool, args, toolCallId };

  try {
    // ③ 建 progress channel
    const channel = createProgressChannel();
    const config = { metadata: { onProgress: channel.emit }, signal: this.signal };

    const toolPromise = tool.invoke(args, config).then(
      raw => { channel.close(); return raw; },
      err => { channel.close(); throw err; },
    );

    // ④ 并发消费 progress 消息
    for await (const message of channel) {
      yield { type: 'tool_progress', tool, message, toolCallId };
    }

    const rawResult = await toolPromise;
    yield { type: 'tool_end', tool, args, result, duration, toolCallId };

    ctx.scratchpad.recordToolCall(...);
    ctx.scratchpad.addToolResult(...);
  } catch (error) {
    yield { type: 'tool_error', tool, error: msg, toolCallId };
    ctx.scratchpad.addToolResult(tool, args, `Error: ${msg}`);
  }
}
```

一个函数覆盖：权限、限流、生命周期事件、progress channel、signal 传递、结果落盘。**没有 middleware chain，没有拦截器栈，没有 hook 系统**。你想加一个新关切（比如 rate limit），直接在函数里加一段。

### 2.9 RunContext：收拢零碎状态

```typescript
// src/agent/run-context.ts
export interface RunContext {
  readonly query: string;
  readonly scratchpad: Scratchpad;
  readonly tokenCounter: TokenCounter;
  readonly startTime: number;
  iteration: number;
  lastApiInputTokens: number;
}

export function createRunContext(query: string): RunContext {
  return {
    query,
    scratchpad: new Scratchpad(query),
    tokenCounter: new TokenCounter(),
    startTime: Date.now(),
    iteration: 0,
    lastApiInputTokens: 0,
  };
}
```

一个 20 行的接口 + factory function，把整轮跑的运行时状态归到一起。传参签名从 `run(query, scratchpad, tokenCounter, iteration, startTime, ...)` 变成 `run(query, ctx)`。**注意它是 interface 而不是 class**——工厂返回 plain object，可以 mutate，不需要 getter/setter。

#### 2.9.1 为什么要有这个容器

`run(query)` 里的 helper（`executeToolsAndCollectMessages`、`handleDirectResponse`、`manageContextThreshold`、`toolExecutor.executeAll` 等）**每个都要读/写这堆东西的一部分**。三种传法都有毛病：

- **挂 `this`**：并发跑多个 query 时状态互相污染（Agent 实例是可复用的，见 `Agent.create`）
- **一个个当参数传**：签名爆炸，`ctx: RunContext` 简洁得多
- **全局单例**：更糟，不用讨论

`RunContext` 就是 "**per-run 的可变状态容器**"——一次 `run(query)` 生一个，跟着这次调用从头传到尾，用完即弃：

```typescript
// agent.ts:run()
const ctx = createRunContext(query);
// ...
yield* this.executeToolsAndCollectMessages(response, ctx);
yield* this.manageContextThreshold(ctx, query, memoryFlushState, messageState);
```

#### 2.9.2 字段各干什么

| 字段 | 可变性 | 谁在写 | 谁在读 |
|---|---|---|---|
| `query` | `readonly` | `createRunContext` 一次性设 | Compaction / memory flush prompt、scratchpad 命名、日志 |
| `scratchpad` | `readonly` 引用（内部可变） | 每次 tool 执行完 `addToolResult` | Compaction 时 `getToolResults()` 取全量、context 决策时 `getActiveToolResultCount()` 计数 |
| `tokenCounter` | `readonly` 引用（内部可变） | 每次 LLM 调用后 `add(usage)`，包括主 loop + compaction 副调用 | 最后 `done` 事件里报 `tokenUsage` 和 `tokensPerSecond` |
| `startTime` | `readonly` | `createRunContext` 里 `Date.now()` | `handleDirectResponse` 算 `totalTime` |
| `iteration` | 可变 | 主 loop 每轮 `ctx.iteration++` | `done` 事件、`maxIterations` 判断 |
| `lastApiInputTokens` | 可变 | 每次 LLM 响应后从 `usage.inputTokens` 回填 | `manageContextThreshold` 判断是否要 compaction |

`readonly` 关键字在 TS interface 里只管**字段本身不能被重新赋值**，不管里面的对象内容能不能变。所以 `ctx.scratchpad = new Scratchpad(...)` 会编译报错，但 `ctx.scratchpad.addToolResult(...)` 完全 OK——这是故意的：**引用固定，内容可变**，防止有人不小心中途把 scratchpad 换掉丢历史。

反过来 `iteration` 和 `lastApiInputTokens` 没 `readonly`，因为它俩本来就是简单数字，语义上就是要每轮更新。

#### 2.9.3 `lastApiInputTokens`：把 API 真实反馈存下来

它跟 `tokenCounter` 看起来重复，其实作用完全不同：

- `tokenCounter` 是**累计的**（`inputTokens` 一直加），给最终账单用
- `lastApiInputTokens` 是**最近一次的**（每次覆盖），代表 "上一轮送进 API 时 context 的实际大小"

用途在 `agent.ts:manageContextThreshold`：

```typescript
const estimatedContextTokens = ctx.lastApiInputTokens > 0
  ? ctx.lastApiInputTokens
  : estimateTokens(messageState.messages.map(...).join('\n'));
```

翻译这段逻辑：**"如果我上一轮已经发过一次请求，API 告诉我当时输入是 N tokens，那我就用 N 作为当前 context 大小的锚；如果还没有 API 反馈（第一轮），就退化成字符数估算。"**

这是本项目里反复强调的一个原则（对应 [CONTEXT_MANAGEMENT.md 十一节](./CONTEXT_MANAGEMENT.md#十一四个可以拿走的原则) 那句 "API 报的 tokens 比字符估算准"）——`text.length / 3.5` 那种估算能差一个数量级，尤其是有 tool_call JSON 结构、Anthropic cache_control 元数据、系统 prompt 时。**有真实反馈就不要用估算**。

所以 `RunContext` 里塞这一个字段，本质是**把 API 的真实反馈存下来供下一轮决策用**——没有它，`manageContextThreshold` 只能盲猜。

#### 2.9.4 `iteration` 为什么放在 ctx 里

主 loop 长这样：

```typescript
while (ctx.iteration < this.maxIterations) {
  ctx.iteration++;
  // ...
}
```

看起来完全可以写成 `let iteration = 0; while (iteration < max)`。但 `iteration` 在 loop 外面也要读到：

- `handleDirectResponse` 里报 `iterations: ctx.iteration`（在 `done` 事件里）
- 未来 tool executor 里如果要按迭代号打 tag，也能拿到

放 ctx 里免得把 `iteration` 也当参数一路传，属于 "顺手放的"。

#### 2.9.5 生命周期

```
run(query) 开始
  │
  ├─ createRunContext(query)          ← 生一次
  │     ├─ query 冻结
  │     ├─ new Scratchpad(query)      ← 内部还会创个 JSONL 文件
  │     └─ new TokenCounter()
  │
  ├─ while (iteration < max):
  │     ├─ ctx.iteration++
  │     ├─ LLM call → ctx.tokenCounter.add(usage)
  │     │           → ctx.lastApiInputTokens = usage.inputTokens
  │     ├─ tool exec → ctx.scratchpad.addToolResult(...)
  │     └─ manageContextThreshold(ctx, ...)   ← 读 lastApiInputTokens 决策
  │
  └─ handleDirectResponse(ctx)
        └─ 拿 ctx.startTime / iteration / tokenCounter 组装 `done` 事件
```

一次 `run()` 结束，`ctx` 就随着 generator 关闭被 GC。**下一次 `run()` 从零再造一个**——这就是并发安全的来源：不同 query 的 ctx 是完全隔离的对象，只共享 `Agent` 实例本身的 `readonly` 配置（`tools`、`systemPrompt`、`model` 等）。

#### 2.9.6 一句话总结

`RunContext` 不是什么高级设计，它就是**把 "per-run 状态" 从散落的 loop 局部变量 + this 属性收成一个包**，让 helper 方法签名干净、并发运行互不打架。真正精妙的一点是 `lastApiInputTokens`——**把 API 的真实 tokens 反馈存下来当决策锚**，这是 context management 那一层能做对判断的基础。

### 2.10 抄作业：最小骨架

```typescript
type AgentEvent =
  | { type: 'tool_start'; tool: string; args: unknown }
  | { type: 'tool_end';   tool: string; result: string }
  | { type: 'done'; answer: string };

async function* runAgent(
  query: string,
  llm: BaseChatModel,
  tools: StructuredToolInterface[],
  systemPrompt: string,
  maxIterations = 10,
): AsyncGenerator<AgentEvent> {
  const toolMap = new Map(tools.map(t => [t.name, t]));
  let messages: BaseMessage[] = [
    new SystemMessage(systemPrompt),
    new HumanMessage(query),
  ];

  for (let i = 0; i < maxIterations; i++) {
    const response = await llm.bindTools(tools).invoke(messages) as AIMessage;

    if (!response.tool_calls?.length) {
      yield { type: 'done', answer: extractText(response) };
      return;
    }

    messages.push(response);

    for (const call of response.tool_calls) {
      yield { type: 'tool_start', tool: call.name, args: call.args };
      try {
        const result = await toolMap.get(call.name)!.invoke(call.args);
        const resultStr = typeof result === 'string' ? result : JSON.stringify(result);
        messages.push(new ToolMessage({
          content: resultStr,
          tool_call_id: call.id!,
          name: call.name,
        }));
        yield { type: 'tool_end', tool: call.name, result: resultStr };
      } catch (e) {
        messages.push(new ToolMessage({
          content: `Error: ${e}`,
          tool_call_id: call.id!,
          name: call.name,
        }));
      }
    }
  }

  yield { type: 'done', answer: `Reached max iterations` };
}
```

Dexter 就是在这个骨架上**逐个加了 5 件事**：

| 加什么 | 落在哪里 | 复杂度 |
|---|---|---|
| Streaming | `callModelWithStreaming` + `stream_progress` 事件 | +40 行 |
| 并发工具 | `partitionToolCalls` + `all(generators)` | +50 行 |
| 权限 | `evaluatePermission` + `permissions/*` | +独立模块 |
| 上下文治理 | `manageContextThreshold` + microcompact + compact | +100 行 + 独立模块 |
| Subagent | `spawn_subagent` 工具本身递归调 `Agent.create` | +独立工具 |

**每一件事都是一个可以移除的旁支**，去掉之后骨架还能跑。

### 2.11 可以拿走的三个原则

1. **AsyncGenerator + 事件 union 优于 EventEmitter / callback**。类型安全 + 天然支持嵌套 + 天然支持取消。
2. **不要给"就是个 ReAct loop"的场景引入状态图/DAG 抽象**。消息数组本身就是状态，一个 while 就是转换函数。等你真需要显式状态图（多 agent 编排、复杂 checkpointing）再上 LangGraph。
3. **抽象要伸缩，不要固化**。Dexter 的分层是"能删就删"的分层：
   - 删掉 streaming → 变 blocking，能跑
   - 删掉 concurrent → 变全串行，能跑
   - 删掉 permission → 变自动执行，能跑
   - 删掉 compaction → 上下文会爆，但业务能跑

   "每一层都独立可移除"的架构，比"每一层都绑死上下层"的架构更抗腐化。

---

## 三、`all(generators, maxConcurrency)` 合流实现

源码（`src/utils/concurrency.ts`，30 行）：

```typescript
export async function* all<A>(
  generators: AsyncGenerator<A, void>[],
  concurrencyCap = Infinity,
): AsyncGenerator<A, void> {
  const next = (generator: AsyncGenerator<A, void>) => {
    const promise: Promise<QueuedGenerator<A>> = generator
      .next()
      .then(({ done, value }) => ({ done, value: value as A, generator, promise }));
    return promise;
  };

  const waiting = [...generators];
  const promises = new Set<Promise<QueuedGenerator<A>>>();

  // 1. 启动初始批次
  while (promises.size < concurrencyCap && waiting.length > 0) {
    const gen = waiting.shift()!;
    promises.add(next(gen));
  }

  // 2. 主循环
  while (promises.size > 0) {
    const { done, value, generator, promise } = await Promise.race(promises);
    promises.delete(promise);

    if (!done) {
      promises.add(next(generator));   // 3. yield 完了，续订这个 generator
      if (value !== undefined) yield value;
    } else if (waiting.length > 0) {
      promises.add(next(waiting.shift()!));  // 4. 结束了，让候补上
    }
  }
}
```

要解决的问题："**我有 N 个 async generator，最多同时跑 K 个，把它们 yield 出来的值按到达顺序合并成一个流**"。

### 3.1 场景

Dexter 的 tool executor 里，一轮 LLM 返回可能一次要跑 5 个并发工具：

```
web_search(nvidia)     ─┐
web_search(amd)         │
get_financials(NVDA)    ├──►  合并成一个事件流
read_filings(NVDA)      │
get_market_data(NVDA)  ─┘
```

每个工具本身是一个 async generator（yield `tool_start` → 若干 `tool_progress` → `tool_end`）。UI 需要**实时**看到"任何一个工具的任何一个事件"，而不是等所有工具都跑完再一次性给结果。

要求：
1. **并发跑**：5 个工具同时跑
2. **合流**：谁先 yield 谁先送到 UI
3. **限流**：并发数封顶（默认 10）
4. **背压**：慢的 generator 不能拖住快的 generator

### 3.2 核心机制：`Promise.race` + 自续订

整个函数最精髓的一行：

```typescript
const { done, value, generator, promise } = await Promise.race(promises);
```

**`Promise.race([p1, p2, p3])` = 谁先 resolve 就返回谁的值**。

每个 promise 的形状：

```typescript
Promise<{ done, value, generator, promise }>
                     ↑           ↑
                 是哪个 gen   自己的 promise 引用
```

**每个 promise 里都带着"我是谁"的信息**。所以 `race` 出来的赢家，我马上就知道要给它续订下一次 `.next()`，也知道要从 `promises` 集合里把它删掉。

### 3.3 走一遍时间线（3 个 generator，并发 2）

假设 `concurrencyCap=2`、`waiting=[G1, G2, G3]`：

```
初始化：
  从 waiting 拿两个：G1, G2
  promises = { P1a: G1.next(), P2a: G2.next() }
  waiting = [G3]

Loop iter 1:
  Promise.race → P1a 先 resolve，value='hello1'
  promises.delete(P1a)
  非 done → promises.add(next(G1))    → promises = { P2a, P1b }
  yield 'hello1'                       → UI 拿到 'hello1'

Loop iter 2:
  Promise.race → P1b 先 resolve，value='hello2'
  promises = { P2a, P1c }（G1 又续订了一次）
  yield 'hello2'

Loop iter 3:
  Promise.race → P2a 先 resolve，value='world1'
  yield 'world1'
  promises = { P1c, P2b }

Loop iter 4:
  Promise.race → P1c resolve 并且 done=true
  promises.delete(P1c)
  waiting=[G3] → promises.add(next(G3))  → promises = { P2b, P3a }
  （G3 直到这时候才被启动！）

... 继续直到所有 generator 都 done 且 promises 为空
```

关键几个"精妙"：

1. **G1 一直"自我续订"**：只要它没 done，每次 yield 完立刻 `promises.add(next(generator))` 塞回赛道
2. **G3 直到有 slot 才启动**：如果 G1 和 G2 一直快速 yield，G3 会一直在 `waiting` 里等。这就是**背压**
3. **收工条件很简单**：`while (promises.size > 0)`。所有 generator 都 done 且没人在等，size 归零

### 3.4 每个 promise 为什么要"绑架"自己

看这行：

```typescript
const promise: Promise<QueuedGenerator<A>> = generator
  .next()
  .then(({ done, value }) => ({ done, value, generator, promise }));  // ← 引用自己！
return promise;
```

`.then(...)` 里返回的 object 有一个 `promise` 字段——**指向包含它的那个 promise 本身**。

**为什么要带上 `promise` 自引用？** 因为主循环需要 `promises.delete(promise)`——从 `Set` 里删掉赢家。如果没这个自引用，你就得**遍历整个 Set 找哪个 promise 对应这个 generator**，O(n)。带自引用后 O(1)。

把"我是哪个 slot"的信息**塞进 payload 本身**，回来的时候直接自证身份。

### 3.5 `Set` 而不是 `Array` 的原因

用 `Set` 是为了 O(1) 的 `delete(promise)`。如果用 `Array`，`.delete` 就得 `filter` 或者 `splice`，一次 O(n)。`Promise.race` 参数是 iterable，`Set` 也是 iterable。

### 3.6 和"最朴素合流"的对比

**朴素版：等所有 generator 全跑完再返回**
```typescript
async function* naive(generators) {
  const results = await Promise.all(
    generators.map(async g => {
      const arr = [];
      for await (const v of g) arr.push(v);
      return arr;
    })
  );
  for (const arr of results) yield* arr;
}
```

问题：
- **不是实时**：得等所有 generator 全结束
- **没限流**：一次全启动
- **失去了并发的意义**

**半天真版：Promise.race 但没有续订**
```typescript
async function* halfNaive(generators) {
  const iters = generators.map(g => g[Symbol.asyncIterator]());
  const pending = iters.map((it, i) => it.next().then(r => [r, i]));
  while (pending.length) {
    const [res, i] = await Promise.race(pending);
    // 忘了续订 pending[i] = iters[i].next() —— 无限循环！
  }
}
```

**"续订"** 是最容易出错的地方——忘续订就死循环，续订错位就漏事件。Dexter 用 `Set + promise 自引用` 让"续订"变成一句 `promises.add(next(generator))`，零心智负担。

### 3.7 语义细节：`value !== undefined`

```typescript
if (!done) {
  promises.add(next(generator));
  if (value !== undefined) yield value;   // ← 为什么加这个判断
}
```

Async generator 是可以 `yield undefined` 的。这里过滤掉 `undefined` 是**业务约定**——Dexter 的工具事件永远不会是 undefined。如果你把 `all` 用在其它场景，`undefined` 是合法值，这里就要小心。

### 3.8 异常处理：责任分离

`all` 里**完全没有 try/catch**。它把异常权限交给了消费者。

如果某个 generator 内部 throw，它的 `.next()` 返回的 promise 就 reject。`Promise.race` 遇到 reject 会**立即 reject**，`await Promise.race(promises)` 抛异常，`all` 这个 async generator 自身也 throw 出去。

Dexter 里，`AgentToolExecutor.executeSingleWithId` **自己就把每个工具的异常 catch 掉转成 `tool_error` 事件**：

```typescript
try {
  // ... tool.invoke ...
  yield { type: 'tool_end', ... };
} catch (error) {
  yield { type: 'tool_error', ... };  // ← 单个 generator 内部 catch，永远不 throw
}
```

所以到 `all` 层面已经没有异常了，`all` 只需要合流。**责任分离得很干净**。

### 3.9 抄作业版本

```typescript
async function* mergeConcurrent<T>(
  gens: AsyncGenerator<T>[],
  cap = Infinity,
): AsyncGenerator<T> {
  type Slot = { done: boolean; value: T; gen: AsyncGenerator<T>; p: Promise<Slot> };

  const request = (gen: AsyncGenerator<T>): Promise<Slot> => {
    const p: Promise<Slot> = gen.next().then(r => ({
      done: !!r.done, value: r.value, gen, p,
    }));
    return p;
  };

  const waiting = [...gens];
  const inflight = new Set<Promise<Slot>>();

  while (inflight.size < cap && waiting.length) {
    inflight.add(request(waiting.shift()!));
  }

  while (inflight.size) {
    const slot = await Promise.race(inflight);
    inflight.delete(slot.p);

    if (!slot.done) {
      inflight.add(request(slot.gen));
      yield slot.value;
    } else if (waiting.length) {
      inflight.add(request(waiting.shift()!));
    }
  }
}
```

30 行搞定，覆盖 90% 的合流场景。

### 3.10 值得记住的三个技巧

1. **`Promise.race` + payload 自引用**：让"谁先到"和"接下来给谁续订"用同一个 payload 承载，无需额外索引结构
2. **动态启动新任务**：`waiting` 队列 + slot 满就启动，天然实现并发上限
3. **异常责任分离**：合流器不吞不 catch，把异常权交给 generator 内部（就地 try/catch）或最外层消费者

30 行代码里没有一行浪费——这种"用语言原语解决问题、不引入抽象"的写法，就是"干净的 tool-calling loop"这种设计品味的具体体现。

---

## 四、Streaming chunk 合并的细节

第 2.6 节讲了流式调用的整体流程，这里把最容易踩坑的一步——**如何把 N 个 chunk 合成一个完整的 AIMessage**——展开讲清楚。

### 4.1 流式为什么难：不同 provider 给的 chunk 形状不一样

同样是 "调用 web_search 工具，参数 query='NVDA earnings'"，三个 provider 流出来的 chunk 长这样：

**OpenAI**（字符串增量 + tool_call_deltas）：
```
chunk 1: { content: "",  tool_call_chunks: [{ index: 0, id: "call_abc", name: "web_search",         args: '{"que' }] }
chunk 2: { content: "",  tool_call_chunks: [{ index: 0,                                              args: 'ry":"NVDA ' }] }
chunk 3: { content: "",  tool_call_chunks: [{ index: 0,                                              args: 'earnings"}' }] }
chunk 4: { content: "",  usage_metadata: { input_tokens: 1200, output_tokens: 42 } }
```

**Anthropic**（typed content parts，thinking 独立分片）：
```
chunk 1: { content: [{ type: 'thinking', thinking: '让我搜索' }] }
chunk 2: { content: [{ type: 'thinking', thinking: 'NVDA 财报' }] }
chunk 3: { content: [{ type: 'tool_use', id: 'toolu_xyz', name: 'web_search', input: {} }] }
chunk 4: { content: [{ type: 'input_json_delta', partial_json: '{"query":"NVDA' }] }
chunk 5: { content: [{ type: 'input_json_delta', partial_json: ' earnings"}' }] }
chunk 6: { usage_metadata: { input_tokens: 1200, output_tokens: 42 } }
```

**Google Gemini**（把整个 tool call 一次给你）：
```
chunk 1: { content: "" }
chunk 2: { tool_calls: [{ id: 'g_1', name: 'web_search', args: { query: 'NVDA earnings' } }] }
```

**如果每个 provider 都写一份解析代码，重复且极容易出错**——tool_call JSON 的增量拼接、Anthropic 的 thinking part 处理、Gemini 的一次性 dump，全部得单独考虑。

### 4.2 LangChain 的解决方案：`AIMessageChunk.concat()`

`@langchain/core` 帮你把三个 provider 的差异**统一封装到 `AIMessageChunk`** 里。每个 provider 的 SDK 都会把自己的原始格式转成 `AIMessageChunk`，然后 chunk 之间用 `.concat()` 合并。

看 Dexter 里的用法（`agent.ts`）：

```typescript
let accumulated: AIMessageChunk | null = null;

for await (const chunk of streamLlmWithMessages(messages, { model, tools, signal })) {
  accumulated = accumulated ? accumulated.concat(chunk) : chunk;
  // ...
}

// 最后转成完整的 AIMessage
const response = new AIMessage({
  content: accumulated.content,
  tool_calls: accumulated.tool_calls,
  invalid_tool_calls: accumulated.invalid_tool_calls,
  usage_metadata: accumulated.usage_metadata,
  response_metadata: accumulated.response_metadata,
});
```

**一行 `.concat()` 就把所有 provider 的合并逻辑搞定了**。内部它会：

- **字符串 content**：拼接（`"NV" + "DA earnings"` → `"NVDA earnings"`）
- **数组 content**（Anthropic）：按 `index` / `id` 匹配同一个 part 追加
- **tool_call_chunks**：按 `index` 找到对应 tool call，把 `args` 字符串增量拼起来，最后解析成完整 JSON 塞进 `tool_calls`
- **usage_metadata**：以最后一条为准（一般只有末 chunk 才带）

### 4.3 但 UI 要"实时反馈"，不能等最后

问题来了：`.concat()` 把 chunk 合并到 `accumulated`——但 UI 需要**在每次 chunk 到达时**就更新"working indicator"（比如 "Thinking... 234 chars"）。如果只在最后合并完再通知 UI，用户看到的就是"卡半天，突然刷屏"。

Dexter 的解法是**在合并的同时**从 chunk 里抽出"UI 想知道的信息"，实时 yield 出去：

```typescript
for await (const chunk of streamLlmWithMessages(messages, ...)) {
  accumulated = accumulated ? accumulated.concat(chunk) : chunk;
  const { charDelta, mode } = inspectChunkContent(chunk);
  if (charDelta > 0 || mode !== 'responding') {
    yield { type: 'stream_progress', charDelta, mode };
  }
}
```

UI 侧只关心两件事：
- **charDelta**：这个 chunk 带来了多少新字符（用来累加显示"Thinking... 234 chars"）
- **mode**：当前是在 `requesting` / `thinking` / `responding` / `tool-input` / `tool-use` 哪个阶段

### 4.4 `inspectChunkContent`：跨 provider 归一化

关键函数（`agent.ts` 底部）：

```typescript
function inspectChunkContent(chunk: AIMessageChunk): { charDelta: number; mode: StreamMode } {
  const content = chunk.content;

  // 情况 A: 字符串（OpenAI / xAI / Ollama / DeepSeek）
  if (typeof content === 'string') {
    return { charDelta: content.length, mode: 'responding' };
  }
  if (!Array.isArray(content)) {
    return { charDelta: 0, mode: 'responding' };
  }

  // 情况 B: 数组（Anthropic 的 typed parts）
  let charDelta = 0;
  let mode: StreamMode = 'responding';
  for (const part of content) {
    if (!part || typeof part !== 'object') continue;
    const partType = (part as { type?: string }).type;

    if (partType === 'text') {
      const text = (part as { text?: string }).text;
      if (typeof text === 'string') charDelta += text.length;
      if (MODE_PRIORITY.responding > MODE_PRIORITY[mode]) mode = 'responding';

    } else if (partType === 'thinking' || partType === 'redacted_thinking') {
      const thinkingText = (part as { thinking?: string }).thinking;
      if (typeof thinkingText === 'string') charDelta += thinkingText.length;
      if (MODE_PRIORITY.thinking > MODE_PRIORITY[mode]) mode = 'thinking';

    } else if (partType === 'tool_use' || partType === 'input_json_delta') {
      const partialJson = (part as { input?: unknown; partial_json?: string }).partial_json;
      if (typeof partialJson === 'string') charDelta += partialJson.length;
      if (MODE_PRIORITY['tool-input'] > MODE_PRIORITY[mode]) mode = 'tool-input';
    }
  }
  return { charDelta, mode };
}
```

三个 provider 的差异被这个 60 行的函数完全吸收：

| Chunk 形状 | Provider | 归一化后的 mode |
|---|---|---|
| `content: "hello"` | OpenAI/xAI/Ollama | `responding` |
| `content: [{ type: 'text', text: 'hi' }]` | Anthropic | `responding` |
| `content: [{ type: 'thinking', thinking: '...' }]` | Anthropic | `thinking` |
| `content: [{ type: 'tool_use', ... }]` | Anthropic | `tool-input` |
| `content: [{ type: 'input_json_delta', ... }]` | Anthropic | `tool-input` |

### 4.5 `MODE_PRIORITY`：一个 chunk 有多种 part 怎么办

Anthropic 的一个 chunk 里可能同时有 `text` + `tool_use`——这时候 UI 应该显示什么状态？Dexter 定义了一个优先级：

```typescript
const MODE_PRIORITY: Record<StreamMode, number> = {
  requesting: 0,
  responding: 1,
  thinking: 2,
  'tool-input': 3,
  'tool-use': 4,
};
```

**取所有 part 中最高优先级的 mode**。语义上：一旦 LLM 开始输出 tool_input（在准备调用工具了），UI 就不该再显示"responding"——即使这个 chunk 里也有普通文本。

优先级映射到 UI：
- **requesting**（0）：请求已发出，等第一个字节 → 显示"Requesting..."
- **responding**（1）：正常输出文字 → 显示"Responding... 234 chars"
- **thinking**（2）：模型在做 reasoning → 显示"Thinking... 512 chars"
- **tool-input**（3）：正在拼工具参数 JSON → 显示"Preparing tool call..."
- **tool-use**（4）：stream 结束，即将执行工具 → 显示"Calling web_search..."

### 4.6 特殊情形：流式失败回退

前面提到 `callModelWithStreaming` 有个 fallback：

```typescript
private async *callModelWithStreaming(messages) {
  try {
    return yield* this.streamAndAccumulate(messages);
  } catch {
    return await this.callModelWithMessages(messages);  // ← 走非流式
  }
}
```

为什么必要？因为：
- 有些 Ollama 版本流式接口不稳（bufferoverflow 报错）
- 某些 provider 的 tool-calling 流式和普通流式的支持度不一致
- xAI Grok 的 streaming + tool_calls 曾有 breaking change

**优雅回退**的关键：非流式路径 `callLlmWithMessages` 返回的也是 `AIMessage`（有 `tool_calls`），跟流式路径的返回值形状**完全兼容**。上层 `run()` 完全不感知走了哪条路。

### 4.7 usage metadata 的坑

大部分 provider 只在**最后一个 chunk**里带 `usage_metadata`（因为要等 output 结束才能算总 token）。但 Dexter 用了个偷懒的写法：

```typescript
const usage = accumulated.usage_metadata
  ? {
      inputTokens: accumulated.usage_metadata.input_tokens ?? 0,
      outputTokens: accumulated.usage_metadata.output_tokens ?? 0,
      totalTokens: accumulated.usage_metadata.total_tokens ?? 0,
    }
  : undefined;
```

从**合并后的 accumulated** 里读 usage——`AIMessageChunk.concat()` 的规则保证 `usage_metadata` 最后必然收敛到"最后一个非空 usage"。这么写不用关心 usage 到底出现在哪一 chunk。

### 4.8 抄作业：最小流式合并

如果自己实现，最简版：

```typescript
async function* streamAndAccumulate(
  messages: BaseMessage[],
  llm: BaseChatModel,
): AsyncGenerator<{ type: 'delta'; charDelta: number }, AIMessage> {
  let accumulated: AIMessageChunk | null = null;

  for await (const chunk of await llm.stream(messages)) {
    accumulated = accumulated ? accumulated.concat(chunk) : chunk;
    const text = typeof chunk.content === 'string' ? chunk.content : '';
    if (text.length > 0) yield { type: 'delta', charDelta: text.length };
  }

  if (!accumulated) throw new Error('empty stream');

  return new AIMessage({
    content: accumulated.content,
    tool_calls: accumulated.tool_calls,
    usage_metadata: accumulated.usage_metadata,
  });
}
```

约 15 行搞定基础版。想加 Anthropic thinking / tool-input 归一化就照着 `inspectChunkContent` 抄。

### 4.9 三个可以拿走的技巧

1. **依赖 `AIMessageChunk.concat()`，别自己拼 JSON**——LangChain 已经把 tool_call args 的增量合并、Anthropic typed content 的按 index 追加等复杂性都处理好了
2. **"合并"和"UI 反馈"分开**：合并到 accumulated 用于最终结果；`inspectChunkContent` 从 chunk 里抽 UI 关切的字段，实时 yield
3. **归一化的边界要清晰**：`StreamMode` 是所有 provider 差异的**唯一出口**——UI 层只 switch case 5 种 mode，永远不接触 provider-specific 的 chunk 形状

---

## 延伸阅读

- 主文档：[PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md)
- 权限引擎：[PERMISSION_ENGINE.md](./PERMISSION_ENGINE.md)
- 多通道复用：[MULTI_CHANNEL.md](./MULTI_CHANNEL.md)
- 上下文治理：[CONTEXT_MANAGEMENT.md](./CONTEXT_MANAGEMENT.md)
- Subagent 委派：[SUBAGENT_DELEGATION.md](./SUBAGENT_DELEGATION.md)
- Skills 系统：[SKILLS_SYSTEM.md](./SKILLS_SYSTEM.md)
- 源码入口：`src/agent/agent.ts`, `src/agent/tool-executor.ts`, `src/utils/concurrency.ts`, `src/model/llm.ts`
