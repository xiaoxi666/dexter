# Dexter 项目盘点

> 面向 `virattt/dexter` 仓库的架构梳理与实用价值评估。
> 生成时间：2026-07-24

## 目录

- [一、项目定位](#一项目定位)
- [二、源码结构总览](#二源码结构总览)
- [三、Agent 实现思路（重点）](#三agent-实现思路重点)
- [四、周边模块](#四周边模块)
- [五、实用价值评估：能不能用来指导投资？](#五实用价值评估能不能用来指导投资)
- [六、延伸阅读](#六延伸阅读)

---

---

## 一、项目定位

**Dexter** 是一个部署在终端里的自主金融研究 Agent（TypeScript + Bun + LangChain）。作者用一句话概括："Claude Code, but built specifically for financial research"——把 Coding Agent 那套（规划 → 工具调用 → 自我校验 → 循环）搬到了金融基本面研究上。

- 版本：`dexter-ts@1.0.3`，MIT
- 运行时：Bun（主）+ tsx（Gateway 用）
- 主入口：`src/index.tsx` → `runCli()`；WhatsApp 网关入口：`src/gateway/index.ts`
- 默认模型：`gpt-5.6-sol`；支持 OpenAI / Anthropic / Google / xAI / OpenRouter / Ollama / Moonshot / DeepSeek

一份很有意思的 `SOUL.md` 定义了它的"人格"：立足 Buffett/Munger 的投资哲学（价值 vs 价格、能力圈、安全边际、逆向思考），写在系统 prompt 里作为 identity 注入。

---

## 二、源码结构总览

```
src/
├── index.tsx / cli.ts           # CLI 入口（pi-tui + 自建组件系统）
├── agent/                       # ★ 核心 agent loop
│   ├── agent.ts                 # 主循环 + streaming + 上下文管理
│   ├── prompts.ts               # System prompt 装配（SOUL + rules + tools + skills + memory + channel profile）
│   ├── scratchpad.ts            # JSONL 记录本 + 工具调用限流预警
│   ├── tool-executor.ts         # 并发/串行分批 + permission gate
│   ├── microcompact.ts          # 轻量 per-turn 剪枝
│   ├── compact.ts               # LLM 摘要式 compaction
│   ├── channels.ts              # CLI/WhatsApp 输出画像 profile
│   ├── run-context.ts / token-counter.ts / types.ts
├── model/llm.ts                 # 多 provider 抽象（工厂 + retry + streaming）
├── providers.ts                 # provider 元数据（prefix、fastModel、context window）
├── tools/
│   ├── registry.ts              # 工具注册中心（按 env 条件启用）
│   ├── finance/                 # 财务数据（15 个子工具，走 financialdatasets.ai）
│   ├── search/                  # exa / perplexity / tavily / langsearch / x_search
│   ├── browser/                 # Playwright 无头浏览
│   ├── fetch/                   # URL → markdown → 快模型摘要
│   ├── filesystem/, bash/       # 本地文件与 shell（有 permission 门）
│   ├── memory/                  # memory_search / get / update
│   ├── heartbeat/, cron/        # 定时任务与心跳
│   ├── subagent/                # ★ spawn_subagent（并行委派）
│   ├── ask-user-question/       # CLI 交互式反问
│   └── skill.ts                 # 触发 SKILL.md 工作流
├── skills/                      # dcf / write-memo / x-research（SKILL.md + frontmatter）
├── memory/                      # SQLite + embeddings + MMR + 时间衰减，混合检索
├── permissions/                 # bash 命令解析 + 规则引擎（allow/ask/deny）
├── gateway/                     # ★ WhatsApp 网关（Baileys），路由 / 会话 / 群聊 mention
│   └── channels/whatsapp/       # 登录 / 收发 / 断线重连 / auth-store
├── cron/                        # croner 定时任务执行器 + heartbeat 迁移
├── controllers/                 # UI 侧控制器（agent runner、model 选择、input 历史）
├── components/                  # pi-tui 自定义组件（chat log、editor、prompts…）
├── evals/                       # LangSmith + LLM-as-judge 评测（50k CSV 数据集）
└── utils/                       # env / config / cache / tokens / errors …
```

依赖精选：`@langchain/core@1.1`（新版）、`@langchain/anthropic|openai|google|ollama`、`@whiskeysockets/baileys`（WhatsApp）、`playwright`、`better-sqlite3`（本地向量库）、`croner`、`langsmith`、`zod`、`@mariozechner/pi-tui`（React-like TUI，非 Ink）。

值得一提：`README.md` 里说的是 Ink，但仓库实际用的是 **pi-tui**——`AGENTS.md` 里的说明有一处过时。

---

## 三、Agent 实现思路（重点）

这是这个项目最值得看的部分——它没有依赖 LangGraph 或 LangChain 高阶封装，而是**自己撸了一个"够薄但要点齐全"的 tool-calling loop**，并叠加了完整的上下文治理机制。核心文件是 `src/agent/agent.ts`（~800 行）。

### 3.1 单一循环，事件流驱动

`Agent.run()` 是一个 async generator，每一轮 yield 出各种 `AgentEvent`（`thinking` / `tool_start` / `tool_progress` / `tool_end` / `tool_denied` / `stream_progress` / `context_cleared` / `microcompact` / `compaction` / `memory_flush` / `queue_drain` / `done`）。UI（controller）单方向消费事件，做到实时增量渲染 + 中断能力（`AbortSignal`）。

一轮迭代做的事：

```
① microcompact（可能）
② 剥离旧 AIMessage 的 reasoning（只留最近 2 条）
③ streaming 调 LLM，累积 chunk 成 AIMessage
④ 有 thinking 文本先 yield 出去
⑤ 若没有 tool_calls → 就是最终答案，done
⑥ 否则执行工具（并发或串行）→ 拿到 ToolMessage
⑦ 大结果落盘 + 预览注入（避免污染上下文）
⑧ 每轮 tool 结果做 budget cap（enforceResultBudget）
⑨ 若被拒（deny）→ 立即结束
⑩ 检查 token 阈值 → memory flush → compaction → 兜底 truncate
⑪ 注入使用软限制警告（若接近 tool 调用上限）
⑫ drain 消息队列（用户在 agent 工作时补发消息）
```

### 3.2 消息数组，而非状态图

它选择的是 "growing message array + full reasoning continuity"——一直往同一个 `BaseMessage[]` 里 push，直到触发 compaction 才重写数组。**没有** LangGraph 那种显式状态图。这个选择让代码非常直观，但需要自建大量"守护栏杆"，见 3.4-3.6。

### 3.3 流式 + 优雅回退

`callModelWithStreaming` 先试 streaming，异常时回退到 blocking `.invoke()`——为了兼容不同 provider 的流式差异。流式 chunk 里 `content` 可能是 string（多数 provider）或 typed array（Anthropic 的 thinking / tool_use / text 分片），`inspectChunkContent` 把它们统一映射为 `stream_progress` 事件的 `mode`（`requesting/thinking/responding/tool-input/tool-use`），驱动前端"working indicator"。

### 3.4 工具并发（Anthropic-style）

`AgentToolExecutor.executeAll`：

- 每个工具在 registry 里带 `concurrencySafe` 标记（只读类为 true）
- `partitionToolCalls` 把 tool_calls 分成**连续并发段** vs **单个串行**
- 并发段用 `all(generators, maxConcurrency=10)` 一起跑，事件按 `toolCallId` 归属正确的 UI 行
- 串行段（如 `write_file`、`bash`、`ask_user_question`）挨个跑，且在执行前走 permission engine

工具最终按 `response.tool_calls` **原始顺序**回填成 `ToolMessage[]`，避免并发导致的顺序错乱破坏 OpenAI/Anthropic 的 tool_call ↔ tool_message 配对约束。

### 3.5 权限引擎（很硬核）

`src/permissions/engine.ts` + `command-parser.ts`（400 行的 shell 解析）+ `rules.ts`：

- `write_file`/`edit_file` → 强制 ask
- `bash` → parser 分段 → classify（read-only / mutating / unknown）→ 匹配用户规则（`.dexter/settings.json`）→ built-in deny（禁访问密钥、env 注入）
- 拒绝走 rule，避免"橡皮图章疲劳"——用户不会一直看到明显违规的批准弹窗
- 支持四档批准：`allow-once` / `allow-session` / `allow-always`（持久化规则）/ `deny`

这套设计是从 Claude Code 那边借鉴的，属于 CLI Agent 里做得很扎实的一档。

### 3.6 上下文治理（三级）

这是我认为设计最有价值的部分——**用同一份 message array，靠三层机制把上下文控制住**：

**Layer 1: microcompact（`microcompact.ts`）**
- 每轮 LLM 调用前跑
- 计数触发：可压缩 ToolMessage > 8 时，保留最新 4，其余替换成 `[Old tool result content cleared]`
- Token 触发：可压缩 ToolMessage 合计 > 80k tokens 时同上
- **仅针对只读工具**（`get_financials/read_filings/web_search/browser/…`），mutating 工具的结果保留

**Layer 2: reasoning 剥离**
- 每轮把旧 AIMessage 的 text content 清空，只留最近 2 条 + 保留 `tool_calls` 结构（不能删，否则配对崩）

**Layer 3: full compaction（`compact.ts`）**
- 触发条件：`ctx.lastApiInputTokens > effectiveContextWindow - 33000`（真实 API 输入 tokens，比字符估算准）
- 用**当前 provider 的 fast model**（如 `gpt-5.6-luna`、Anthropic 的 Haiku）
- Prompt 强制返回 `<analysis>...</analysis><summary>...</summary>`，涵盖 9 个 section（原始 query、关键概念、数据、错误、进展、数值、待办、当前状态、下一步）
- 明确要求 fast model **不要**调用工具（`NO_TOOLS_PREAMBLE` + `NO_TOOLS_TRAILER`）
- 摘要成功后**替换整个 message 数组**为 `[SystemMessage, HumanMessage(query + summary)]`
- 连续失败 3 次后停止尝试，回退到直接 truncate 旧轮次

**Layer 3.5: memory flush（可选）**
- Compaction 之前先把 tool results 摘要写入 `.dexter/memory/*.md`（持久化到磁盘），防止长期记忆丢失

**Layer 4: 结果大小护栏**
- `exceedsSizeCap` 检查单个工具结果 → 太大就 `persistLargeResult` 落盘到 `.dexter/scratchpad/`，上下文里只放 preview + 文件路径
- `enforceResultBudget` 保证一轮工具结果的总 tokens 不超过预算

**Layer 5: overflow 兜底**
- 即便如此还溢出，`isContextOverflowError(err)` 触发时删除最老几个 round，最多重试 2 次

这一整套是 Anthropic 官方 blog《Effective context engineering》里推荐的做法，Dexter 实现得比很多开源 Agent 都完整。

### 3.7 Subagent 委派（"one level deep"）

`spawn_subagent` 是个特殊工具：

- LLM 可以在一轮里发多个 spawn_subagent 调用 → 因为它 `concurrencySafe=true` 且被 tool_executor 并发跑，**多个子 agent 并行**
- 三种类型：`general-purpose` / `research` / `analysis`，每种绑定一份只读工具白名单 + 独立 `maxIterations=8`
- 子 agent 用 `systemPromptOverride` 走一份 self-contained worker prompt——**跳过 SOUL、rules、memory**，避免污染 context
- 强制单层：`SUBAGENT_DISALLOWED_TOOLS = {spawn_subagent, ask_user_question, bash}`，子 agent 不能再派子 agent
- 子 agent 的最终答案 verbatim 返回给主 agent，主 agent 负责综合

### 3.8 Scratchpad = 单一事实源

`src/agent/scratchpad.ts` 是每次 query 的独立文件（`.dexter/scratchpad/YYYY-MM-DD-HHMMSS_<hash>.jsonl`），append-only JSONL：
- `init`（原始 query）
- `tool_result`（每次工具调用的完整 args + result）
- `thinking`（LLM 的中间推理）

它还内置**软限流**：同一工具超过 3 次会警告 LLM"考虑换工具/换 query/接受数据缺失并告诉用户"——是**警告而非阻断**，让 LLM 自己决定，避免代码硬阻断卡死研究。同时用 query 相似度（`findSimilarQuery`）识别重复调用循环。

### 3.9 Skills（SKILL.md 工作流）

不是代码模板，是**给 LLM 看的 markdown 指令**：

- `src/skills/*/SKILL.md`，YAML frontmatter（name + description）+ 正文（步骤 checklist）
- 启动时 `discoverSkills()` 扫描 builtin + `.dexter/skills/`
- LLM 通过 `skill` 工具调用 → 工具返回 markdown 内容注入到下一轮 context
- 每个 skill 每 query 最多触发一次（`hasExecutedSkill` 去重）
- 内建：`dcf`（DCF 估值 8 步流程 + `sector-wacc.md` 参数表）、`write-memo`、`x-research`

这是把"复杂 workflow"外置成 markdown，非常适合非工程师维护。

### 3.10 其它工程细节

- **模型抽象**：`model/llm.ts` 用 factory 模式按 prefix 路由（`claude-` → Anthropic，`gemini-` → Google 等），`callLlm` 和 `streamLlm` 是唯二的对外接口
- **多 provider 幂等 retry**：分类错误 → non-retryable 立即抛，其它指数退避 3 次
- **Anthropic prompt caching**：在 SystemMessage 上打 `cache_control`，长 system prompt 只收一次费
- **消息队列**：`utils/message-queue.ts` 让用户在 agent 工作时继续发言，下一轮 loop 之间 drain 进来（`queue_drain` 事件）
- **通道 profile**：`channels.ts` 里 CLI vs WhatsApp 有不同的 preamble / 输出格式 / 表格策略——WhatsApp 里不用 markdown 表格，CLI 用 pi-tui 渲染的 box table

---

## 四、周边模块

**Memory（`src/memory/`）**：SQLite + embeddings（OpenAI/Gemini/Ollama 三选一 auto）+ MMR 去冗余 + 时间衰减（半衰期 30 天）+ 向量 0.7 / BM25-ish 0.3 混合检索。Session 开始时会把最近 memory 塞进 system prompt，`memory_flush` 会在 compaction 前落盘。

**Gateway（WhatsApp）**：Baileys 扫码登录 → `inbound.ts` 收消息 → `resolve-route` 判定发给哪个 agent → `agent-runner.ts` 起 agent、`cleanMarkdownForWhatsApp` 净化输出 → `outbound.ts` 发回。群聊里靠 mention 触发，`group/history-buffer.ts` 缓存上下文。**同一 session 串行**（`isSessionRunning` + `enqueueForSession`）。

**Cron / Heartbeat**：`croner` 跑定时任务，可用 `cron` 工具让 agent 自己给自己排班；`.dexter/HEARTBEAT.md` 是个 checklist 文件，`heartbeat` 工具让 agent 阅读/更新。

**Evals**：`src/evals/run.ts` 拉 CSV 数据集 → 跑 agent → LLM-as-judge 打分（`evaluator.ts`）→ 上报 LangSmith，实时 UI 显示 accuracy。

---

## 五、实用价值评估：能不能用来指导投资？

直接说结论：**可作为研究辅助工具，不适合作为投资决策来源。** 分几层看：

### 5.1 它做得很好的地方（有实用价值）

- **数据获取**：接的是 `financialdatasets.ai`（付费机构级 API），能拉 5 年利润表/资产负债表/现金流量表、关键指标（P/E、ROE、ROIC、margin、growth）、SEC 10-K/10-Q/8-K 内容、内部人交易、机构持仓、盘中/历史价格。数据源本身可信。
- **数据整理**：`get_financials` 用 fast model 做 "路由"（自然语言 → 子工具组合），能一次处理"AAPL vs MSFT 过去 5 年营收对比"这种多公司多指标查询——比自己写代码调 API 快得多。
- **文档阅读**：`read_filings` 能 targeted 抽取 10-K 的特定章节（业务描述、风险因素、MD&A），配合 `browser` 打开 IR 页面，能顶掉不少人工翻页的时间。
- **DCF skill**：`src/skills/dcf/SKILL.md` 的 8 步流程是**教科书标准 DCF**——历史 FCF 增长 → WACC（按 sector 查表）→ 5 年现金流预测 + 终值 → PV → 敏感性分析。作为 sanity check 是够用的。
- **结构化研究**：SOUL.md + 循环里的"先取数再下结论"（约束 LLM 不先猜答案再找证据）设计得比较好，能在一定程度上减少 LLM 编造数字。
- **可审计性**：所有工具调用都在 `.dexter/scratchpad/*.jsonl` 里落盘，事后可以回看它到底看了什么数据，比某些黑盒 Agent 强得多。

### 5.2 硬性局限（不能直接指导投资）

- **README 顶部本身写了 disclaimer**："for educational, entertainment, and informational purposes only... Not financial advice"——作者自己就不建议拿它做真钱决策。
- **LLM 生成数字问题**：即使工具返回的是真实数据，final answer 是 LLM 综合出来的，数字仍可能被算错、比例弄反、单位混淆。DCF skill 里的很多 heuristic（增长率 15% 上限、5% 年 decay、WACC 加减）是**经验规则**，不同分析师会给出完全不同的结果。
- **数据缺一大块**：
  - 只支持美股财报（`financialdatasets.ai` 的覆盖）；港股、A 股、欧股不行
  - 没有宏观数据（利率、汇率、大宗商品）作为独立数据源
  - 没有真正的 quant 因子/回测能力
  - 没有实时 Level 2 行情或期权链
- **没接触定性研究的核心**：管理层通话（earnings call transcript）、行业访谈、竞品动态需要靠 `web_search` 挖，质量参差。
- **过拟合基本面价值投资视角**：SOUL.md 完全是 Buffett/Munger 学派——对成长股、周期股、事件驱动策略、量化策略基本不友好。
- **没有 portfolio 视角**：单只票分析是它的舒适区，组合层面的相关性/风险管理/仓位建议它做不了（memory 里能存偏好但没算力）。
- **agent 循环上限**：默认 10 轮迭代 + subagent 8 轮，遇到需要深挖多层的问题（比如"分析半导体全产业链"）会 hit 上限提前 done。

### 5.3 建议的用法

1. **当作 research copilot，不当作 decision maker**：让它帮你把 10-K 的风险因素 + 过去 5 年现金流 + 内部人交易一起拉出来做 pre-read，你自己看数字下判断。
2. **DCF skill 当 sanity check**：用它跑出一个"教科书 DCF"参考区间，再和你自己的模型/卖方模型对比差异——差异大在哪，就深挖那里。
3. **不要接实盘/自动交易**：项目本身也没有下单接口。
4. **本地化改造**：如果你研究 A 股/港股，需要自己写一套 finance 工具接 iFinD / Wind / AKShare，用同样的注册机制挂上去，SOUL 和 skill 都要重写。

### 5.4 对开发者的价值 > 对投资者的价值

坦白讲，这个仓库**作为"如何写一个生产级 CLI Agent"的教材，价值大于它作为"投资工具"的价值**。你能从里面学到：

- 干净的 tool-calling loop 怎么写（没有过度抽象）
- Anthropic-style 上下文治理三层（microcompact / compaction / truncate）
- Subagent 并行委派（one level deep）
- Skills = markdown 工作流的解耦机制
- 权限引擎（shell 解析 + 规则匹配）
- Streaming + 多 provider 抽象 + fallback
- 多通道（CLI / WhatsApp）复用同一个 agent 核心

如果你的目标是做投资，去买 Bloomberg 或订 alpha research 更划算；如果你想学**怎么造一个类似 Claude Code 但面向垂直领域**的 Agent，这个仓库是我最近读过最工整的开源实现之一。

---

## 六、延伸阅读

本主文档聚焦"是什么、怎么组织"。以下四份分支文档针对具体主题深挖：

| 文档 | 主题 | 何时看 |
|---|---|---|
| [TOOL_CALLING_LOOP.md](./TOOL_CALLING_LOOP.md) | Generator/yield 基础、tool-calling loop、并发合流器、streaming chunk 合并 | 想学"怎么造这种 agent" |
| [CONTEXT_MANAGEMENT.md](./CONTEXT_MANAGEMENT.md) | Microcompact / compaction / truncate 三层上下文治理 + result 落盘 + memory flush | 想学"怎么在长 context 下不爆炸" |
| [SUBAGENT_DELEGATION.md](./SUBAGENT_DELEGATION.md) | `spawn_subagent` 并行委派、one-level-deep 隔离设计 | 想学"怎么让 LLM 分派并行子任务" |
| [SKILLS_SYSTEM.md](./SKILLS_SYSTEM.md) | SKILL.md 工作流、markdown 解耦复杂流程 | 想学"如何让非工程师参与 agent 工作流" |
| [PERMISSION_ENGINE.md](./PERMISSION_ENGINE.md) | Shell parser、fail-closed 设计、规则匹配 | 想学"怎么给 agent 安全跑 bash" |
| [MULTI_CHANNEL.md](./MULTI_CHANNEL.md) | CLI + WhatsApp 复用同一个 Agent 类的架构 | 想学"怎么让一个 agent 跨前端复用" |
| [PRODUCTION_HARDENING.md](./PRODUCTION_HARDENING.md) | 成本控制、Fallback、可审计、长尾防御、当前的短板 | 想从"能不能上生产"视角评估 |

建议阅读顺序：先看完本主文档 → 想深入 agent 内核就看 TOOL_CALLING_LOOP.md → 关心具体子系统再挑对应分支文档。
