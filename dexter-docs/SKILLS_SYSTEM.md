# Dexter Skills：Markdown 工作流解耦机制

> 主文档：[PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md)
>
> 这份文档专门讲清楚 Dexter 的 Skills 系统——为什么把"复杂工作流"外置成 markdown 而不是代码，怎么发现/加载/调用一个 skill，以及这种设计的收益和局限。

Skills 是 Dexter 里非常有意思的一个抽象：**把复杂多步任务的"如何做"从代码里搬出来，写成给 LLM 看的 markdown 文档**。DCF 估值就是内置的一个 skill——`src/skills/dcf/SKILL.md` 是一份 8 步 checklist，LLM 拿到后照做。

## 目录

- [一、要解决的问题](#一要解决的问题)
- [二、SKILL.md 的形状](#二skillmd-的形状)
- [三、生命周期：发现 → 元数据 → 按需加载](#三生命周期发现--元数据--按需加载)
- [四、注入 system prompt：只暴露元数据](#四注入-system-prompt只暴露元数据)
- [五、`skill` 工具：动态返回指令](#五skill-工具动态返回指令)
- [六、去重与幂等](#六去重与幂等)
- [七、内建 skill 一览](#七内建-skill-一览)
- [八、Skills vs 硬编码 vs Tool 的取舍](#八skills-vs-硬编码-vs-tool-的取舍)
- [九、局限与坑](#九局限与坑)
- [十、可以拿走的三个原则](#十可以拿走的三个原则)

---

## 一、要解决的问题

**场景**：LLM 面对"帮我做一个 AAPL 的 DCF 估值"这种复杂请求。

**朴素方案**：直接让 LLM 自己发挥——它可能：
- 只拉最新一年的现金流，忘了看 5 年趋势
- WACC 直接拍脑袋 8%，不看行业基准
- 忘了做敏感性分析
- 数字算完就报答案，没算安全边际

**Function calling 方案**：写一个 `dcf_valuation(ticker)` 工具，让代码里 hard-code 完整流程。缺点：
- 一旦流程改（比如换个 WACC 计算法），要改代码、改测试、发版本
- LLM 只能拿最终结果，看不到"每一步在算什么"——出错难 debug
- 复杂参数塞在 tool schema 里，LLM 调用不灵活

**Skill 方案**：把 8 步流程写成 markdown，LLM 触发时**把这份 markdown 塞进 context**，让 LLM 按 checklist 自己一步步跑（每一步都用现有的 tool）。

Skills 本质上是**"给 LLM 的操作手册"**——不是代码，不是工具，是**指令**。

## 二、SKILL.md 的形状

看 `src/skills/dcf/SKILL.md` 开头：

```markdown
---
name: dcf-valuation
description: Performs discounted cash flow (DCF) valuation analysis to estimate 
  intrinsic value per share. Triggers when user asks for fair value, intrinsic 
  value, DCF, valuation, "what is X worth", price target, undervalued/overvalued 
  analysis, or wants to compare current price to fundamental value.
---

# DCF Valuation Skill

## Workflow Checklist

Copy and track progress:
```

两段结构：

**YAML frontmatter**（`---` 包起来的头）：
- `name`：唯一 slug（`dcf-valuation`）
- `description`：给 LLM 看的"何时触发"说明——这段会拼到 system prompt 里

**Markdown 正文**：
- 完整流程指令：checklist、每步用什么工具、参数怎么填、fallback 怎么处理、什么时候 sanity check

DCF skill 的正文有 8 个 Step，每步都是这种形状：

```markdown
### 1.1 Cash Flow History
**Query:** `"[TICKER] annual cash flow statements for the last 5 years"`

**Extract:** `free_cash_flow`, `net_cash_flow_from_operations`, `capital_expenditure`

**Fallback:** If `free_cash_flow` missing, calculate: 
  `net_cash_flow_from_operations - capital_expenditure`
```

**指令是给 LLM 读的，不是给代码 parse 的**。LLM 看懂"用 get_financials 拉 5 年现金流"就够了，Dexter 不需要写代码解析这段 markdown。

## 三、生命周期：发现 → 元数据 → 按需加载

`src/skills/registry.ts` 定义了三个阶段：

### 3.1 发现（启动时）

```typescript
const SKILL_DIRECTORIES: { path: string; source: SkillSource }[] = [
  { path: __dirname, source: 'builtin' },
  { path: join(process.cwd(), dexterPath('skills')), source: 'project' },
];

export function discoverSkills(): SkillMetadata[] {
  if (skillMetadataCache) return Array.from(skillMetadataCache.values());
  
  skillMetadataCache = new Map();
  for (const { path, source } of SKILL_DIRECTORIES) {
    const skills = scanSkillDirectory(path, source);
    for (const skill of skills) {
      skillMetadataCache.set(skill.name, skill);  // 项目级覆盖内建
    }
  }
  return Array.from(skillMetadataCache.values());
}
```

两个来源：
- **内建**：`src/skills/*/SKILL.md`（跟代码一起 ship）
- **项目**：`.dexter/skills/*/SKILL.md`（用户在项目里添加）

**同名项目级覆盖内建**——用户可以重写内建 skill，就像 shell 里 alias 覆盖内置命令一样。

### 3.2 提取元数据（只读 frontmatter）

`extractSkillMetadata` 用 `gray-matter` 库解析：

```typescript
export function extractSkillMetadata(path: string, source: SkillSource) {
  const content = readFileSync(path, 'utf-8');
  const { data } = matter(content);  // ← 只读 frontmatter

  if (!data.name || typeof data.name !== 'string') {
    throw new Error(`Skill at ${path} is missing required 'name' field...`);
  }
  return { name: data.name, description: data.description, path, source };
}
```

**只读 frontmatter，不读正文**——因为正文可能很长（DCF skill 有几百行），启动时全部读完既费内存又慢。

### 3.3 按需加载（触发时）

`loadSkillFromPath` 才读整份文件：

```typescript
export function parseSkillFile(content: string, path: string, source: SkillSource): Skill {
  const { data, content: instructions } = matter(content);
  // ... validate ...
  return {
    name: data.name,
    description: data.description,
    path,
    source,
    instructions: instructions.trim(),   // ← 完整正文
  };
}
```

只有 LLM 真的调 `skill(skill: 'dcf-valuation')` 那一刻才读全文。**懒加载**。

## 四、注入 system prompt：只暴露元数据

`src/agent/prompts.ts` 里的 `buildSkillsSection`：

```typescript
function buildSkillsSection(): string {
  const skills = discoverSkills();
  if (skills.length === 0) return '';

  const skillList = buildSkillMetadataSection();  // 只列 name + description

  return `## Available Skills

${skillList}

## Skill Usage Policy

- Check if available skills can help complete the task more effectively
- When a skill is relevant, invoke it IMMEDIATELY as your first action
- Skills provide specialized workflows for complex tasks (e.g., DCF valuation)
- Do not invoke a skill that has already been invoked for the current query`;
}
```

LLM 在 system prompt 里看到的是：

```
## Available Skills

- **dcf-valuation**: Performs discounted cash flow (DCF) valuation analysis to estimate 
  intrinsic value per share. Triggers when user asks for fair value, intrinsic value, DCF...
- **write-memo**: Produces a professional investment memo...
- **x-research**: Searches X/Twitter for market sentiment and breaking news...

## Skill Usage Policy
...
```

**关键设计：正文不在 system prompt 里**。原因：

1. **省 token**：DCF skill 全文可能 3000 tokens，占用系统 prompt 就变成常驻开销
2. **只在需要时才付出成本**：LLM 判断"这次要 DCF 估值"再触发，触发时才把正文 dump 到 context
3. **description 就是"广告"**：LLM 只根据 description 决定要不要触发，所以 description 写得越明确、trigger 场景列得越全越好

## 五、`skill` 工具：动态返回指令

Skill 触发通过一个特殊工具 `skill`（`src/tools/skill.ts`）：

```typescript
export const skillTool = new DynamicStructuredTool({
  name: 'skill',
  description: 'Execute a skill to get specialized instructions for a task. Returns instructions to follow.',
  schema: z.object({
    skill: z.string().describe('Name of the skill to invoke (e.g., "dcf")'),
    args: z.string().optional().describe('Optional arguments for the skill (e.g., ticker symbol)'),
  }),
  func: async ({ skill, args }) => {
    const skillDef = getSkill(skill);
    if (!skillDef) {
      const available = discoverSkills().map(s => s.name).join(', ');
      return `Error: Skill "${skill}" not found. Available skills: ${available || 'none'}`;
    }

    let result = `## Skill: ${skillDef.name}\n\n`;
    if (args) result += `**Arguments provided:** ${args}\n\n`;

    // 把 markdown 里的相对路径链接改成绝对路径
    const skillDir = dirname(skillDef.path);
    const resolved = skillDef.instructions.replace(
      /\[([^\]]+)\]\(([^)]+\.md)\)/g,
      (_match, label, relPath) => {
        if (relPath.startsWith('/') || relPath.startsWith('http')) return _match;
        return `[${label}](${resolve(skillDir, relPath)})`;
      },
    );

    return result + resolved;
  },
});
```

**这个工具的返回值就是 SKILL.md 的正文**——LLM 在下一轮 iteration 里就能读到完整 checklist，然后照做。

**相对路径改写**是个精巧细节：DCF skill 里引用了 `[sector-wacc.md](sector-wacc.md)`——如果不改成绝对路径，LLM 后续用 `read_file` 找不到这个文件。工具在返回前把所有 markdown 相对链接改成绝对路径，让 LLM 能顺利读附加参考文件。

## 六、去重与幂等

一个 skill 每 query 最多触发一次——`tool-executor.ts` 里的处理：

```typescript
private partitionToolCalls(toolCalls: ToolCall[], ctx: RunContext): ToolCallBatch[] {
  const batches: ToolCallBatch[] = [];
  for (const call of toolCalls) {
    // Skill dedup — skip already-executed skills
    if (call.name === 'skill') {
      const skillName = (call.args as Record<string, unknown>).skill as string;
      if (ctx.scratchpad.hasExecutedSkill(skillName)) continue;  // ← 跳过
    }
    // ...
  }
}
```

**为什么要去重？** 因为 skill 返回的是**指令**——同一次 query 里，LLM 已经收到 DCF checklist 了，再触发一次就等于在 context 里塞了两份一模一样的指令。浪费 token 而且可能让 LLM 混乱。

System prompt 里的 policy 也强调："Do not invoke a skill that has already been invoked for the current query"——**双保险**：prompt 告诉 LLM 别重复触发，代码兜底就算触发了也不执行。

## 七、内建 skill 一览

Dexter 目前有三个内建 skill（`src/skills/`）：

**`dcf/`**：DCF 估值 8 步流程。附加文件 `sector-wacc.md`（各行业 WACC 参考区间）。核心 checklist：
1. 拉数据（现金流历史、财务指标、资产负债表、当前股价、公司信息）
2. 算 FCF 增长率（5 年 CAGR，上限 15%）
3. 估 WACC（用 sector 查 sector-wacc.md）
4. 预测未来 5 年 FCF（增长率年 5% 递减）
5. 算终值和现值
6. 敏感性分析（5×5 表格）
7. 结果验证
8. 呈现结果 + 免责声明

**`write-memo/`**：生成投资备忘录格式的报告（Bull case / Bear case / Valuation / Risks）。

**`x-research/`**：调用 `x_search` 工具做 X/Twitter 情绪研究（需要 `X_BEARER_TOKEN`）。

**用户添加自定义 skill 只需**：在 `.dexter/skills/my-skill/SKILL.md` 写一份 markdown，重启 CLI 就能用。

## 八、Skills vs 硬编码 vs Tool 的取舍

三种实现"复杂工作流"的思路对比：

| 特性 | 硬编码代码 | 传统 Tool（function call）| **Skill（markdown）** |
|---|---|---|---|
| 谁维护 | 工程师 | 工程师 | 分析师 / 非工程师也能改 |
| 修改流程要发版吗 | 是 | 是 | 否（改 md 即可） |
| LLM 灵活性 | 无（走死代码路径）| 中（tool schema 约束参数）| 高（LLM 按 checklist 灵活裁剪）|
| 出错可 debug 吗 | 只看代码 log | 只看代码 log | LLM 每步都能看到，看 scratchpad 全过程 |
| 语言能力 | 只能是代码支持的 | 参数受 schema 限制 | 自然语言，可写 fallback、条件、启发式 |
| Token 成本 | 零 | 零（schema 常驻）| 大（触发时全文进 context）|
| 决定性 | 强（一样输入一样输出）| 中 | 弱（LLM 可能偏离 checklist）|

**Skill 的核心权衡**：牺牲**决定性**换取**灵活性 + 可维护性 + 非工程师参与度**。

**什么时候该用哪种**：

- **硬编码**：性能敏感、绝对不能偏离的核心业务逻辑（比如支付金额计算）
- **Tool**：单次数据查询、明确的输入输出契约（比如"给我 AAPL 的市值"）
- **Skill**：多步骤流程、有条件分支、可能因输入类型走不同路径的启发式工作流（比如 DCF）

DCF 用 skill 特别合适，因为：
- 步骤明确但**参数需要判断**（增长率取多少要看历史稳定性）
- **有 fallback**（FCF 缺失就用经营现金流 − CapEx 算）
- **每步都用现有 tool**（get_financials、get_market_data），skill 本身不需要新代码
- **投资分析师能自己改**（换个 sector-wacc 参考、加个"如果亏损公司 → 换个方法"分支）

## 九、局限与坑

**1. LLM 可能偏离 checklist**

Skills 是软约束——LLM 完全可以看完 checklist 后决定"我先跳过第 2 步"。DCF skill 里把每步都编号 + 写清楚"copy and track progress"就是在**尽量降低偏离概率**，但不能百分百保证。

**2. Skill 之间不能组合**

现在的设计里，一个 query 里同一个 skill 只触发一次，也没有"skill A 触发 skill B"的机制。想做 skill 组合（比如"先跑 DCF 再写 memo"），得靠 LLM 手动串起来——不总是可靠。

**3. 大 skill 挤压其它 context**

DCF skill 全文 ~3000 tokens。触发后 LLM 后续 iteration 都要带着它，context 占用不小。如果 skill 越写越多、越写越长，会挤压工具结果和用户对话的空间。**skill 里的每一句话都要值得那个 token**。

**4. 版本管理弱**

SKILL.md 没有版本号。skill 改了、LLM 行为变化，从 log 里看是 LLM 变笨了还是 skill 改坏了不容易区分。理想情况下每个 skill 有 semver + changelog + 回归测试，但目前 Dexter 没做。

**5. 描述质量直接决定触发准确率**

`description` 就是给 LLM 的"广告"。写得太窄，用户问 "AAPL 值多少" 可能触发不到 DCF skill；写得太宽，什么问题都触发（浪费 token）。这一步很难，需要真实场景反馈来调。

## 十、可以拿走的三个原则

1. **不是所有"复杂逻辑"都该写代码**：LLM 有理解力，把工作流写成给 LLM 看的 markdown，比写死代码更灵活、更易维护、更让领域专家参与
2. **元数据和正文分离**：description 常驻 system prompt 用来"触发决策"，正文按需加载用来"提供指令"——分层付费
3. **软约束 + 硬去重**：LLM policy prompt 告诉它别重复触发，代码 dedup 兜底就算它想触发也拒绝——两层防护

---

## 延伸阅读

- 主文档：[PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md)
- Tool-calling loop：[TOOL_CALLING_LOOP.md](./TOOL_CALLING_LOOP.md)
- 源码入口：`src/skills/registry.ts`, `src/skills/loader.ts`, `src/tools/skill.ts`, `src/skills/dcf/SKILL.md`
