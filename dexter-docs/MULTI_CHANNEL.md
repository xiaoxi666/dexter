# Dexter 多通道架构：CLI + WhatsApp 共享 Agent 核心

> 主文档：[PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md)
>
> 这份文档专门讲清楚 Dexter 如何让 CLI 和 WhatsApp 两个前端复用同一个 `Agent` 类——一个 `channel: string` 配置参数怎么覆盖住"输出格式/工具集/交互能力/会话状态/事件消费"五个维度的差异。

Dexter 有两个"前端"：
- **CLI**（`src/cli.ts` + `src/controllers/` + `src/components/`）：pi-tui 渲染的终端界面
- **WhatsApp Gateway**（`src/gateway/`）：Baileys 收发消息，通过手机聊天使用

它们的**输入输出形态天差地别**（一个是全屏终端可以看流式动画，一个是分片文本消息不支持 markdown 表格），但**共用同一个 `Agent` 类**。这份文档讲清楚这个复用是怎么做到的。

## 目录

- [一、差异有多大](#一差异有多大)
- [二、抽象核心：`channel: string` 参数](#二抽象核心channel-string-参数)
- [三、差异点一：`ChannelProfile` 注入到 system prompt](#三差异点一channelprofile-注入到-system-prompt)
- [四、差异点二：CLI-only 工具过滤](#四差异点二cli-only-工具过滤)
- [五、差异点三：交互回调用"命名接口 + 可选"处理](#五差异点三交互回调用命名接口--可选处理)
- [六、差异点四：会话状态的架构](#六差异点四会话状态的架构)
- [七、差异点五：事件消费方式](#七差异点五事件消费方式)
- [八、差异点六：群聊上下文](#八差异点六群聊上下文)
- [九、复用度分析](#九复用度分析)
- [十、想加第三个 channel 要做什么](#十想加第三个-channel-要做什么)
- [十一、三个可以拿走的原则](#十一三个可以拿走的原则)

---

## 一、差异有多大

| 维度 | CLI | WhatsApp |
|---|---|---|
| 输出格式 | 允许 markdown headers/tables | 不能用 `##`（会被当字面量显示） |
| 表格 | pi-tui 渲染成 box table | 不支持，得转成 flowing text |
| 中断能力 | 用户按 ESC 立刻中断 | 只能靠 signal 传递 |
| 交互式反问 | 弹出多选浮层等用户点选 | 没法弹窗 → 工具不可用 |
| 审批 | 弹出批准/拒绝浮层 | 没法批准 → 只能预置规则 |
| 会话 | 单会话，进程内 InMemory | 每个手机号 1 个 session，跨断线保留 |
| 语气 | "professional, objective" | "concise, knowledgeable friend texting" |
| 中途接消息 | 靠 message queue drain | 靠 gateway 层加锁 + 队列 |

真正抽象成"channel"这个概念要覆盖上面所有维度。

## 二、抽象核心：`channel: string` 参数

`AgentConfig` 里就一个字段：

```typescript
export interface AgentConfig {
  channel?: string;      // 'cli' | 'whatsapp' | undefined（默认 cli）
  groupContext?: GroupContext;
  // ...
}
```

CLI 侧调用：
```typescript
Agent.create({ channel: 'cli', ... });
```

Gateway 侧调用（`src/gateway/agent-runner.ts`）：
```typescript
Agent.create({
  channel: 'whatsapp',
  groupContext,   // 群聊才有
  ...
});
```

**Agent 内部拿到 channel 之后，做三件事**：改 prompt、过滤工具、检查交互回调。下面逐个展开。

## 三、差异点一：`ChannelProfile` 注入到 system prompt

`src/agent/channels.ts` 里定义了每个 channel 的 profile：

```typescript
const CLI_PROFILE: ChannelProfile = {
  label: 'CLI',
  preamble: 'Your output is displayed on a command line interface. Keep responses short and concise.',
  behavior: [
    "Prioritize accuracy over validation - don't cheerfully agree with flawed assumptions",
    // ...
  ],
  responseFormat: [
    'Keep casual responses brief and direct',
    'Do not use markdown headers or *italics* - use **bold** sparingly for emphasis',
    // ...
  ],
  tables: `Use markdown tables. They will be rendered as formatted box tables.\n...`,
};

const WHATSAPP_PROFILE: ChannelProfile = {
  label: 'WhatsApp',
  preamble: 'Your output is delivered via WhatsApp. Write like a concise, knowledgeable friend texting.',
  behavior: [
    "You're chatting over WhatsApp — write like a knowledgeable friend texting, not a research terminal",
    'Keep messages short and scannable on a phone screen',
    // ...
  ],
  responseFormat: [
    'No markdown headers (# or ##) — they render as literal text on WhatsApp',
    'No tables — they break on mobile',
    'Short paragraphs (2-3 sentences each)',
    // ...
  ],
  tables: null,  // ← 完全省略表格章节
};

const CHANNEL_PROFILES: Record<string, ChannelProfile> = { cli, whatsapp };

export function getChannelProfile(channel?: string): ChannelProfile {
  return CHANNEL_PROFILES[channel ?? 'cli'] ?? CLI_PROFILE;
}
```

**`buildSystemPrompt`** 里根据 profile 拼装 prompt——preamble 插在日期后面，`behavior` / `responseFormat` 变成 markdown bullet，`tables` 是 null 就跳过整段。同一个 SOUL + rules + tool descriptions，因 channel 不同呈现不同的"外交礼仪"。

**这一步是纯 prompt engineering**——不改代码，只改 LLM 看到的说明书。

## 四、差异点二：CLI-only 工具过滤

有些工具**只在 CLI 有意义**——比如 `ask_user_question`（弹出多选让用户点）、`bash`（每次都要用户批准）。在 WhatsApp 里根本没法弹窗，绑上去 LLM 一调用就卡死。

`agent.ts` 里的处理：

```typescript
const CLI_ONLY_TOOLS = new Set<string>(['ask_user_question', 'bash']);

static async create(config: AgentConfig = {}): Promise<Agent> {
  const allTools = getTools(model);
  let tools = config.toolAllowlist ? allTools.filter(...) : allTools;

  const isCli = !config.channel || config.channel === 'cli';
  if (!isCli) {
    tools = tools.filter(t => !CLI_ONLY_TOOLS.has(t.name));
  }
  // ...
}
```

一行 filter 搞定。WhatsApp 里的 LLM 甚至看不到这两个工具的 schema——tool_calls 里永远不会出现它们，从根源上避免"LLM 想调却调不了"的错误状态。

## 五、差异点三：交互回调用"命名接口 + 可选"处理

审批和反问需要"agent 问用户、用户答"的双向交互，Dexter 没用 `.next(value)` 双向 generator，而是用**命名回调**：

```typescript
export interface AgentConfig {
  requestToolApproval?: (request: {...}) => Promise<ApprovalDecision>;
  requestUserInput?: (request: { questions: Question[] }) => Promise<UserAnswers>;
  // ...
}
```

**关键设计：这两个字段是 optional。**

- **CLI 提供**：`controllers/agent-runner.ts` 实现这两个函数——弹出浮层、等用户点选、resolve promise
- **Gateway 不提供**：`gateway/agent-runner.ts` 里两个字段都是 undefined

那 tool 需要审批时怎么办？看 tool-executor 的处理：

```typescript
const decision = (await this.requestToolApproval?.({ ... })) ?? 'deny';
```

**没提供回调 → 结果是 undefined → `?? 'deny'` 兜底拒绝 → yield tool_denied → agent 结束**。这个 fallback 是防御性的——即使不小心在 WhatsApp 里绑了个需要审批的工具，最坏结果也就是拒绝，绝不会卡死或异常。

**这种"能力用可选回调表达"的模式很值得学**——比"要求所有 channel 实现同一个接口"轻量得多，channel 完全可以选择"不提供某项能力"，agent 自动 gracefully 降级。

## 六、差异点四：会话状态的架构

两边的会话管理形态完全不同。

**CLI 侧**（`controllers/agent-runner.ts`）：
- 单个 `AgentRunnerController` 实例，跟着进程走
- `InMemoryChatHistory` 存对话历史
- `AbortController` 支持 ESC 中断
- 一个 session = 一个进程

**Gateway 侧**（`gateway/agent-runner.ts`）：
```typescript
type SessionState = {
  history: InMemoryChatHistory;
  tail: Promise<void>;    // ← 每个 session 的 tail promise，用来串行化
  queue: MessageQueue;
  isRunning: boolean;
};

const sessions = new Map<string, SessionState>();
```

多个手机号并发使用时，用 `sessions` Map 隔离：
- **不同 session 之间可以并发**：手机 A 和手机 B 同时问问题，跑两个独立 agent
- **同一 session 内串行**：手机 A 在 agent 处理中又发一条，通过 `session.tail = session.tail.then(run, run)` 排队执行

**中途新消息处理**是 gateway 的一个亮点：

```typescript
if (isSessionRunning(route.sessionKey)) {
  enqueueForSession(route.sessionKey, model, query);  // ← 塞到消息队列
  return;
}
```

如果 agent 正在跑，新消息塞进 `MessageQueue`。agent 的 loop 每轮工具执行完会 drain 一次这个队列，把新消息作为 HumanMessage 追加到 conversation。**用户不用等 agent 跑完就能继续对话，agent 会"边跑边听"。**

CLI 侧也支持类似机制（`defaultQueue`），但因为终端本来就是同步交互，用得少。

## 七、差异点五：事件消费方式

同一份 `AgentEvent` 流，两边消费方式完全不同：

**CLI 侧**：`AgentRunnerController` 逐个 dispatch 到不同 UI 组件
```typescript
for await (const event of agent.run(query, history)) {
  switch (event.type) {
    case 'tool_start':      appendHistoryItem({ ... }); break;
    case 'tool_progress':   updateHistoryItem({ ... }); break;
    case 'tool_end':        completeHistoryItem({ ... }); break;
    case 'stream_progress': updateWorkingIndicator({ ... }); break;
    case 'done':            appendAnswerBox(event.answer); break;
    // ...
  }
}
```

每种事件对应一个 UI 更新，用户看到实时进度。

**Gateway 侧**：几乎只关心 `done`
```typescript
for await (const event of agent.run(req.query, session.history)) {
  await req.onEvent?.(event);
  if (event.type === 'done') {
    finalAnswer = event.answer;
  }
}
// 只把 finalAnswer 发到 WhatsApp
```

中间的 `tool_start`、`stream_progress` 全部忽略——WhatsApp 一秒发一条中间态没意义。**唯一利用中间事件的地方是 typing indicator**：gateway 用一个独立的 `setInterval(sendComposing, 5000)` 保持"正在输入"动画，跟 agent 事件流并行跑。

**为什么这样也能复用？** 因为 `AgentEvent` 是一个 discriminated union，消费方**只需要挑关心的事件**——TS 帮你处理没关心的分支类型收窄。CLI 消费 15 种，Gateway 消费 1 种，都是合法的。

## 八、差异点六：群聊上下文

WhatsApp 独有的情况——bot 在群里被 @mention 触发。这在 Dexter 里表达为 `groupContext`：

```typescript
export type GroupContext = {
  groupName?: string;
  membersList?: string;
  activationMode: 'mention';
};
```

`buildSystemPrompt` 里如果 groupContext 存在，会追加一段 Group Chat section，提示 LLM：

- 参与的是哪个群
- 是被 @mention 触发的
- 群聊行为准则（叫出提问者名字、参考近期上下文、简洁、避免重复群里已说过的信息）
- 群成员列表

以及 gateway 层专门做的**群历史 buffering**：即使没被 mention，群里的消息也会被 `recordGroupMessage` 缓存下来。下次 mention 触发时，把之前几条群消息作为上下文一起塞进 query：

```typescript
const history = getAndClearGroupHistory(inbound.chatId);
query = formatGroupHistoryContext({ history, currentSenderName, currentBody });
```

这样 bot 就能"看到"群里最近发生了什么，不会完全断片。

## 九、复用度分析

同一份 `Agent` 类被两个 channel 复用，改动量对比：

| 组件 | CLI 需要 | WhatsApp 需要 | 共享度 |
|---|---|---|---|
| `Agent.run()` 主循环 | ✓ | ✓ | 100% |
| Tool executor | ✓ | ✓ | 100% |
| 上下文治理（compact / microcompact） | ✓ | ✓ | 100% |
| Streaming | ✓ | ✓ | 100% |
| 15 种事件类型定义 | ✓ | ✓ | 100% |
| System prompt（除了 profile 部分） | ✓ | ✓ | 100% |
| Channel profile | CLI | WhatsApp | 差异化 |
| Tool 集合 | 全集 | 少 `ask_user_question` `bash` | 差异化 |
| 交互回调 | 提供 | 不提供 | 差异化 |
| Session 管理 | Controller | sessions Map + tail promise | 差异化 |
| 事件消费 | 全部 UI 更新 | 只看 done | 差异化 |

Agent 内核**零改动**，channel 差异全部通过 config 参数注入。这就是"多通道复用"的干净程度。

## 十、想加第三个 channel 要做什么

比如加个 Slack channel，只需要：

1. `agent/channels.ts` 里加 `SLACK_PROFILE`（3-5 分钟）
2. `getChannelProfile` 的 CHANNEL_PROFILES map 加一项（30 秒）
3. `agent/prompts.ts` 里如果 Slack 有独特语境（比如 thread 语义）加一段可选 section（可跳）
4. 写一个 Slack 版的 `runAgentForMessage`——通过 Slack Web API 收发消息（外围代码，不进 agent 核心）

**agent/、tools/、permissions/、memory/、skills/ 目录一行都不用改**。

## 十一、三个可以拿走的原则

1. **Channel 是配置项，不是接口**：一个 `channel: string` 加一个 profile map，比强类型 `interface Channel { renderMessage(); handleApproval(); ... }` 灵活得多。想加个 channel 就填个 profile
2. **能力用可选回调而不是必需接口表达**：`requestToolApproval` 是 optional，channel 不提供就 gracefully 降级到 deny。这让"能力子集"表达非常自然
3. **事件流让消费者按需订阅**：同一个 discriminated union，CLI 消费 15 种事件，Gateway 消费 1 种。两边都对，都类型安全

---

## 延伸阅读

- 主文档：[PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md)
- Tool-calling loop：[TOOL_CALLING_LOOP.md](./TOOL_CALLING_LOOP.md)
- 权限引擎：[PERMISSION_ENGINE.md](./PERMISSION_ENGINE.md)
- 源码入口：`src/agent/channels.ts`, `src/agent/prompts.ts`, `src/controllers/agent-runner.ts`, `src/gateway/agent-runner.ts`, `src/gateway/gateway.ts`
