# Dexter Permission Engine 与 Shell 解析

> 主文档：[PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md)
>
> 这份文档是主文档第 3.5 节"权限引擎"的扩展分支，专门讲清楚 Dexter 的权限系统架构、shell 命令解析、以及"fail-closed"安全设计原则。

这是 Dexter 里最"硬核"的模块——`src/permissions/` 目录，一共约 800 行代码，专门解决一件事：**当 LLM 想跑一个 shell 命令时，能不能直接跑？如果不能，问用户几档，问什么**。

它不引 shell-quote 之类的库，**自己写了一个 fail-closed shell parser**——理由是要精确控制"哪些语法允许、哪些语法拒绝"，一个第三方库解析出的东西超出预期都可能出安全事故。

## 目录

- [一、权限决策的分层](#一权限决策的分层)
- [二、Shell parser：fail-closed 是最高原则](#二shell-parser-fail-closed-是最高原则)
- [三、段（segment）：多命令组合的处理](#三段segment多命令组合的处理)
- [四、内置安全底线：`builtinDeny`](#四内置安全底线builtindeny)
- [五、规则的表达形式：像 Claude Code](#五规则的表达形式像-claude-code)
- [六、`read-only` 分类：显示，但不放行](#六read-only-分类显示但不放行)
- [七、三档批准：allow-once / allow-session / allow-always](#七三档批准allow-once--allow-session--allow-always)
- [八、四个可以拿走的原则](#八四个可以拿走的原则)

---

## 一、权限决策的分层

`evaluatePermission({ tool, args })` 返回 `PermissionDecision`，`mode` 有三种：`allow` / `ask` / `deny`。

流程从简单到复杂：

```
tool 是啥？
├── write_file / edit_file → ask（历史遗留：文件操作永远问一下）
├── bash → 走 evaluateBash（下面详解）
└── 其它（web_search / get_financials / ...）→ allow（读操作自动放行）
```

真正复杂的只有 `evaluateBash`。看它的核心逻辑（`engine.ts`）：

```typescript
export function evaluateBash(command: string, rules: RuleSet = loadRuleSet()): PermissionDecision {
  const parsed = parseCommand(command);

  // 1. Parser 解析不了 → 立刻 ask（never cacheable）
  if (parsed.unknown) return { mode: 'ask', reason: `Command needs review (${parsed.reason})`, sessionCacheable: false, ... };

  const classification = classify(parsed);  // read-only / mutating / unknown

  // 2. 内置安全底线（不可用户覆盖）
  for (const seg of parsed.segments) {
    const floor = builtinDeny(seg);
    if (floor.denied) return { mode: 'deny', ..., matchedRule: '(built-in)' };
  }

  // 3. 用户 deny 规则 → 有任一段匹配就 deny
  for (const seg of parsed.segments) {
    const rule = matchRuleSet(seg, rules.deny);
    if (rule) return { mode: 'deny', ..., matchedRule: serializeRule(rule) };
  }

  // 4. Ask 规则 → 有任一段匹配就 ask
  for (const seg of parsed.segments) {
    const rule = matchRuleSet(seg, rules.ask);
    if (rule) return { mode: 'ask', ..., sessionCacheable: true };
  }

  // 5. Allow 规则 → 所有段都独立 allow 才 allow
  const allowMatches = parsed.segments.map(seg =>
    seg.hasWriteRedirect ? undefined : matchRuleSet(seg, rules.allow),
  );
  if (allowMatches.every(m => m !== undefined)) {
    return { mode: 'allow', ..., matchedRule: serializeRule(allowMatches[0]!) };
  }

  // 6. 默认：ask
  return { mode: rules.defaultBashDecision, ... };
}
```

**关键设计点**：

- **"任一段匹配 deny → 整条 deny"**：`rm file.txt && echo done` 里只要 `rm` 段被 deny，整条就 deny
- **"所有段都匹配 allow → 整条 allow"**：不像 deny 是 OR，allow 是 AND——防止用 `ls; rm -rf /` 这种组合绕过（ls 是允许的，但整条不允许）
- **写重定向永远不能被"词匹配"allow**：`echo foo > /etc/passwd` 里 echo 是无害的，但重定向目标不能被单词规则表达出来，所以强制降级到 ask
- **fail-closed**：parser 有任何 uncertainty 都 return `unknown`，engine 就走 ask 分支——**永远不会因为"我看不懂这段代码"而放行**

## 二、Shell parser：fail-closed 是最高原则

`command-parser.ts` 有 400 行代码，做的事其实是"**在我理解范围内的 shell 语法我解析；理解不了的我就报警**"。

**允许的语法**：
- 单双引号字符串
- 反斜杠转义
- 命令连接：`;`、`&&`、`||`、管道 `|`
- 后台运行 `&`
- 简单重定向：`>`、`>>`、`2>`、`&>`
- 环境变量赋值：`FOO=bar cmd`
- 常见 wrapper：`time`、`nohup`、`env`、`timeout` 等

**明确拒绝的语法**（都会返回 `unknown`）：
- 变量展开：`$VAR`、`${VAR}`
- 命令替换：`$(...)`、反引号
- 子 shell：`(cmd)`
- 花括号展开：`{a,b,c}`
- 注释 `#`
- 未闭合的引号

看代码里的关键片段：

```typescript
// 单引号：literal run 直到下一个单引号（sh 语义：里面完全不解释）
if (c === "'") {
  const end = s.indexOf("'", i + 1);
  if (end === -1) { fail('unterminated single quote'); break; }
  cur += s.slice(i + 1, end);
  i = end + 1;
  continue;
}

// 双引号：literal grouping，但 $ 和反引号还是会展开 → fail-closed
if (c === '"') {
  // ... 里面碰到 $ 或 ` 就 fail('variable or command substitution')
}

// 反引号：命令替换 → fail
// 子 shell / 分组：( ) → fail
// 花括号：{ } → fail  (可能是 brace expansion)
// 注释：# → fail  (谁知道 # 后面藏了啥)
```

**为什么这么保守？** 因为一个"变量展开"的例子：

```bash
X="rm -rf /"
$X important.txt   # 展开后：rm -rf / important.txt
```

如果 parser 没识别出 `$X` 是变量，它就会看到"`$X` 是命令，`important.txt` 是参数"——按词规则可能就放行了。fail-closed 意味着：一旦看到 `$`，我认定我看不懂这命令，让用户来判断。

## 三、段（segment）：多命令组合的处理

一个 shell 命令可能是多段串起来：

```bash
git status && npm test | grep FAIL > errors.log
```

parser 会分成多个 `ParsedSegment`：

```typescript
[
  { command: 'git',  base: 'git',  args: ['status'], env: [], hasWriteRedirect: false, raw: 'git status' },
  { command: 'npm',  base: 'npm',  args: ['test'],   env: [], hasWriteRedirect: false, raw: 'npm test' },
  { command: 'grep', base: 'grep', args: ['FAIL'],   env: [], hasWriteRedirect: true,  raw: 'grep FAIL > errors.log' },
]
```

每个段独立分类、独立匹配规则。这样 `git status && npm test` 才能被"给 git 和 npm 分别 allow"的两条规则组合允许。

**basename 处理**：命令词 `/usr/bin/rm` 的 `base` 是 `rm`——规则 `Bash(rm -rf:*)` 里的 `rm` 也是按 basename 对比。防止有人通过写全路径绕过规则。

## 四、内置安全底线：`builtinDeny`

这是**用户规则无法覆盖**的硬约束（`rules.ts`）：

```typescript
const DANGEROUS_ENV = /^(LD_PRELOAD|LD_LIBRARY_PATH|DYLD_[A-Z_]+|PATH|IFS|BASH_ENV|ENV|SHELLOPTS|PERL5LIB|PYTHONPATH|NODE_OPTIONS|GIT_SSH(_COMMAND)?)=/;

const SECRET_PATTERNS: RegExp[] = [
  /(^|\/)\.env(\.|$)/i,        // .env, .env.local, .env.production
  /(^|\/)\.ssh(\/|$)/i,        // ~/.ssh/
  /(^|\/)\.aws(\/|$)/i,        // ~/.aws/
  /(^|\/)\.gnupg(\/|$)/i,
  /(^|\/)id_(rsa|ed25519|ecdsa|dsa)\b/i,
  /\.pem$/i, /\.p12$/i, /\.key$/i,
  /(^|\/)\.netrc$/i,
  /(^|\/)credentials(\.[\w-]+)?($|\/)/i,
  /\.dexter\/credentials/i,
];
```

拦两类东西：
- **环境变量注入**：`LD_PRELOAD=evil.so ls` 这种利用动态库劫持
- **密钥文件读取**：`cat ~/.ssh/id_rsa`、`cat .env`

这些命令**即使用户手动"Allow always"也不会被放行**——engine 里 `builtinDeny` 检查在用户规则之前。

## 五、规则的表达形式：像 Claude Code

规则语法（`rules.ts` 里的 `parseRule`）：

```
Bash                       # 工具级：任何 bash 命令都匹配
Bash(git status)           # 精确：命令必须完全等于 git status
Bash(git:*)                # 前缀：git 开头的任何命令
Bash(npm install:*)        # 前缀+多词：npm install ... 都匹配
```

规则**持久化**到 `.dexter/settings.json`：

```json
{
  "permissions": {
    "allow": ["Bash(git status)", "Bash(ls:*)", "Bash(cat:*)"],
    "ask":   ["Bash(git push:*)", "Bash(rm:*)"],
    "deny":  ["Bash(sudo:*)", "Bash(curl:* | sh)"]
  }
}
```

**序列化用的是结构化对象**，反序列化 → 匹配 → 反过来 serialize 时**永远从结构化对象生成规则字符串**（不做字符串拼接）。这防止了"用户输入被当作规则一部分注入"的攻击面。

## 六、`read-only` 分类：显示，但不放行

看 `read-only.ts` 里的 `READ_ONLY` 允许列表：

```typescript
const READ_ONLY: Record<string, FlagCheck> = {
  ls: ALWAYS, pwd: ALWAYS, echo: ALWAYS, cat: ALWAYS,
  head: ALWAYS, tail: ALWAYS, wc: ALWAYS, stat: ALWAYS,
  git: gitReadOnly,   // ← 只有 git status/log/diff 等子命令算 read-only
  npm: npmReadOnly,   // ← 只有 npm ls/view 等
  find: findReadOnly, // ← 没有 -exec/-delete 才算
  // ...
};
```

以及"永远不算 read-only"的黑名单：

```typescript
const NEVER_READ_ONLY = new Set([
  'python', 'python3', 'node', 'bun', 'ruby', 'perl', 'php',   // 解释器：可以跑任意代码
  'sh', 'bash', 'zsh', 'ksh', 'fish',                          // shell：可以跑任意代码
  'eval', 'exec', 'source', '.',                               // 内置的"跑另一个"
  'xargs', 'awk', 'sed', 'tr', 'tee', 'dd',                    // 可以生成命令或写入文件
]);
```

有意思的一点：**分类为 read-only 的命令并没有被自动 allow**。engine 的第 6 步默认 ask：

```typescript
return {
  mode: rules.defaultBashDecision,
  reason:
    classification === 'read-only'
      ? 'Read-only command (still asks until the sandbox lands).'
      : 'No matching rule.',
  ...
};
```

**为什么？** 代码注释里说了："Phase 3 才引入 OS-level sandbox，在此之前 read-only 分类只是**给用户展示这条命令的性质**（帮 ta 决定要不要点 allow），并**不自动放行**"。

这是很成熟的安全设计——"分类正确"不等于"能自动放行"，需要有沙箱兜底才敢放。当前阶段每次都问用户，代价是烦，但绝对安全。

## 七、三档批准：allow-once / allow-session / allow-always

`ApprovalDecision` 有 4 种：`allow-once` / `allow-session` / `allow-always` / `deny`。

- **allow-once**：本次跑，下次同命令还问
- **allow-session**：本 session 缓存，下次同命令不问（缓存 key 是**完整命令字符串**，`git status` 允许了不代表 `git push` 也允许）
- **allow-always**：写入 `.dexter/settings.json` 的 allow 规则，永久生效——engine 生成 `proposedRule`（比如你允许了 `git status`，engine 提议规则 `Bash(git status)`）
- **deny**：拒绝并**结束整个 agent turn**

session cache 的 key 设计很讲究：

```typescript
export function sessionKey(tool: string, decision: PermissionDecision): string {
  if (LEGACY_APPROVAL_TOOLS.has(tool)) return FILE_WRITE_SESSION_KEY;  // 文件写共享一个 key
  if (tool === 'bash') return `bash:${(decision.command ?? '').trim()}`;  // ← bash 用完整命令做 key
  return tool;
}
```

**bash 用完整命令做 key**——你 approve 了 `git status`，只对 `git status` 生效，`git status -s` 是不同的 key，仍要问。这防止"我批准了一个宽泛类别，agent 悄悄升级参数"。

## 八、四个可以拿走的原则

1. **权限系统的正确顺序是"parse → classify → 匹配用户规则"**——先把 shell 语法拆干净，然后再谈"这条命令该不该跑"。跳过 parse 一步都不行
2. **fail-closed 是不可协商的**：`unknown` 分支必须走 ask，永远不能"看不懂就默认放行"
3. **内置安全底线独立于用户规则**：环境变量注入、密钥文件访问这类硬威胁，不能让用户用 allow 规则覆盖
4. **session 缓存的粒度要精确到命令级**——不然一次 allow 会滑坡到"允许一整类操作"

---

## 延伸阅读

- 主文档：[PROJECT_OVERVIEW.md](./PROJECT_OVERVIEW.md)
- Tool-calling loop：[TOOL_CALLING_LOOP.md](./TOOL_CALLING_LOOP.md)
- 多通道复用：[MULTI_CHANNEL.md](./MULTI_CHANNEL.md)
- 源码入口：`src/permissions/engine.ts`, `command-parser.ts`, `rules.ts`, `read-only.ts`
