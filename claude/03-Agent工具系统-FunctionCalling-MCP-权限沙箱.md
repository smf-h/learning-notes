# Agent 工具系统：用 JSON 看申请单、审批和沙箱

> 源码基线：`C:\Users\yejunbo\Desktop\claude-code-source`。本树无 git、无 package.json，版本号由构建期 `MACRO.VERSION` 注入。本文只读分析该仓库，没有修改仓库源码。
>
> 同系列：[01 请求结构](./01-Claude-Code发给大模型的完整请求结构.md) · [02 主子 Agent](./02-Claude-Code主子agent机制.md) · [04 记忆](./04-Claude-Code记忆系统-压缩与检索.md)
>
> 对照 Codex：[03 工具与沙箱](../codex/03-Agent工具系统-FunctionCalling-MCP-权限沙箱.md) · [对照表](../对照.md)

这篇不靠函数名讲。一次工具调用就是几份 JSON 在变：模型先交申请单，宿主改权限表，再决定进不进 `sandbox-runtime`。看懂这几张表，就知道沙箱是怎么挡住的。

**阅读约定：** `tool_use.input` 写成对象。线格式只在第 1 节对照一次。

## 面试先记住三张表

```text
1. 工具暴露表：模型现在能看见哪些工具
2. 审批表：这一次放不放行（allow / ask / deny）
3. 沙箱表：放行后，进程能写哪些路径、能不能上网
```

Claude Code 的沙箱不是 Windows Token + Capability SID。它是 `@anthropic-ai/sandbox-runtime`（Linux 上常见是 bwrap，macOS 是 seatbelt）。面试别把 Codex 那张 SID 表安过来。

---

## 1. 模型交上来的是申请单，不是已经执行完的结果

模型想跑命令，发出去的申请单是：

```json
{
  "type": "tool_use",
  "id": "toolu_01Bash001",
  "name": "Bash",
  "input": {
    "command": "echo hello > src/a.txt",
    "description": "Write hello into src/a.txt"
  }
}
```

线上 `input` 已经是对象。对照一次 Codex 线格式，避免讲混：

```json
{
  "Codex_线格式": { "arguments": "{\"cmd\":\"echo hello > src\\\\a.txt\"}" },
  "Claude_Code": { "input": { "command": "echo hello > src/a.txt" } }
}
```

```json
{
  "时机": { "模型输出.type": "tool_use", "name": "Bash" },
  "变化前": { "input": { "command": "echo hello > src/a.txt" } },
  "变化后": { "还没写盘": true, "进入": "Zod 校验 → 审批 → 可选沙箱 → exec" },
  "传输": "申请单先写入 messages；宿主 parse/校验后走系统通道起进程；结果同 id 再 POST 回去"
}
```

申请单进历史，不等于文件已经被写完。模型这时已经结束这一轮采样；真正 `echo` 发生在宿主里。下一轮模型看见的是同 `id` 的 `tool_result`，不是操作系统异常。

工具定义里的 `input_schema` 才是对象形态的 schema，用来校验字段，不管安全：

```json
{
  "name": "Bash",
  "input_schema": {
    "type": "object",
    "required": ["command"],
    "properties": {
      "command": { "type": "string" },
      "timeout": { "type": "number" },
      "description": { "type": "string" },
      "run_in_background": { "type": "boolean" },
      "dangerouslyDisableSandbox": { "type": "boolean" }
    }
  }
}
```

`command` 是 string，并不等于它只能写工作区。形状对了，只说明申请单填得合法。安全在后面的审批和沙箱，不在这张 schema。`_simulatedSedEdit` 故意不暴露给模型，避免用「无害命令 + 任意写盘」绕过审批。

---

## 2. 工具列表 JSON 怎么变：内置和延迟 MCP

本轮请求顶层 `tools` 放已经组装好的内置工具。MCP 多的时候，一部分会标 `defer_loading: true`，模型要先搜。

**第 1 次请求顶层（示意）：**

```json
{
  "tools": [
    { "name": "ToolSearch", "input_schema": { "required": ["query"] } },
    { "name": "Bash" },
    { "name": "Agent" },
    { "name": "Read" },
    { "name": "Grep" }
  ]
}
```

日历类 MCP 工具 `mcp__slack__send_message` 可能不在「已加载」集合里。模型先搜：

```json
{
  "type": "tool_use",
  "id": "toolu_01Search001",
  "name": "ToolSearch",
  "input": { "query": "send slack message", "max_results": 5 }
}
```

宿主本地按 **关键词打分**，不是向量、不是 BM25。名字精确命中分最高，其次是拆开的 token，再次是 description 子串：

```json
{
  "matches": ["mcp__slack__send_message"],
  "query": "send slack message",
  "total_deferred_tools": 42
}
```

`query` 写成 `select:mcp__slack__send_message` 是直接点名，不走打分。

模型读完再发真正调用：

```json
{
  "type": "tool_use",
  "id": "toolu_01Slack001",
  "name": "mcp__slack__send_message",
  "input": { "channel": "#eng", "text": "PR ready" }
}
```

MCP 从模型眼里看就是普通工具，名字是 `mcp__<server>__<tool>`。真正打回 MCP 服务器时会拆开：

```json
{
  "model_sees": { "name": "mcp__slack__send_message" },
  "server_receives": { "server": "slack", "tool": "send_message" }
}
```

缺字段，宿主不崩，把失败写回同一 `id`：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01Slack001",
  "is_error": true,
  "content": "tool call error: missing field `channel`"
}
```

模型自己改参数，**换新 id** 再调。没有「原参数自动重试」。

---

## 3. 审批表：这一次放不放行

配置里的权限模式可以看成：

```json
{
  "permissionMode": "default",
  "alwaysAllowRules": {},
  "alwaysDenyRules": {},
  "alwaysAskRules": {},
  "additionalWorkingDirectories": []
}
```

五种对外模式：

```json
[
  { "mode": "default",            "auto_allow": "工作区内只读、以及规则已允许的", "else": "问人" },
  { "mode": "plan",               "auto_allow": "只读调研", "else": "写文件 / 有副作用的要问或拒" },
  { "mode": "acceptEdits",        "auto_allow": "工作区内编辑自动过", "else": "出界、网络、危险 bash 仍可能问" },
  { "mode": "dontAsk",            "auto_allow": "不问人", "else": "该拒就拒，错误回模型" },
  { "mode": "bypassPermissions",  "auto_allow": "跳过审批弹窗", "else": "沙箱 / deny 规则仍可能在" }
]
```

`dontAsk` 不是「全部自动允许」。`bypassPermissions` 也不是管理员 Token。

一次判定长这样：

```json
{
  "behavior": "allow",
  "updatedInput": { "command": "echo hello > src/a.txt" },
  "decisionReason": { "type": "mode", "mode": "acceptEdits" }
}
```

```json
{
  "behavior": "ask",
  "message": "Claude requested permissions to run this bash command.",
  "decisionReason": { "type": "mode", "mode": "default" },
  "suggestions": [
    {
      "type": "addRules",
      "destination": "session",
      "behavior": "allow",
      "rules": [{ "toolName": "Bash", "ruleContent": "echo *" }]
    }
  ]
}
```

```json
{
  "behavior": "deny",
  "message": "Permission to read C:\\\\Users\\\\yejunbo\\\\.ssh\\\\id_rsa has been denied.",
  "decisionReason": {
    "type": "rule",
    "rule": {
      "source": "userSettings",
      "ruleBehavior": "deny",
      "ruleValue": { "toolName": "Read", "ruleContent": "~/.ssh/**" }
    }
  }
}
```

`decisionReason` 还可以是 `hook`、`subcommandResults`、`permissionPromptTool`。PreToolUse hook 能在模型已经被骗之后再挡一次。

规则来源是 settings，不是 CLAUDE.md：

```json
{
  "permissions": {
    "allow": ["Bash(git *)", "Read(src/**)"],
    "deny": ["Read(~/.ssh/**)", "Bash(curl *)"],
    "ask": ["Bash(rm *)"]
  }
}
```

来源优先级：`policySettings` / `userSettings` / `projectSettings` / `localSettings` / `cliArg` / `session`。仓库里的 `.claude/settings.json` 只有在工作区被信任之后才会当规则用。

读文件的判定顺序（`filesystem.ts`）可以看成：

```json
{
  "时机": { "tool": "Read", "path": "C:\\\\Users\\\\yejunbo\\\\Desktop\\\\claude-code-source\\\\src\\\\auth\\\\session.ts" },
  "步骤": [
    "先看 read deny 规则",
    "再看 read ask 规则",
    "Edit 已允许则隐含可读",
    "工作区内默认允许读",
    "session-memory / plans / tool-results 等内部路径可读"
  ],
  "变化后_工作区内": { "behavior": "allow", "decisionReason": { "type": "mode", "mode": "default" } },
  "传输": "审批在宿主，不进模型 POST。结果决定这次系统通道起不起进程"
}
```

---

## 4. 沙箱表：放行之后进程还能碰什么

打开沙箱时，宿主交给 `sandbox-runtime` 的不是 Windows Token，而是一张路径/网络配置：

```json
{
  "filesystem": {
    "allowWrite": [
      ".",
      "<claude-temp-dir>",
      "（--add-dir 加进来的目录）",
      "（settings.sandbox.filesystem.allowWrite）"
    ],
    "denyWrite": [
      "<user>/settings.json",
      "<project>/.claude/settings.json",
      "<project>/.claude/settings.local.json",
      "<project>/.claude/skills",
      "（若存在）HEAD / objects / refs / hooks / config 这些裸仓库诱饵"
    ],
    "denyRead": ["（Read deny 规则展开的路径）"],
    "allowRead": ["（sandbox.filesystem.allowRead）"]
  },
  "network": {
    "allowedDomains": ["（settings + policy 合并）"],
    "allowManagedDomainsOnly": false
  }
}
```

```json
{
  "时机": {
    "shouldUseSandbox": true,
    "input.dangerouslyDisableSandbox": false,
    "command": "echo hello > ../outside.txt"
  },
  "变化前": { "审批": "allow" },
  "变化后_沙箱拒写": {
    "tool_result": "（stderr 里带 sandbox violation / permission denied）"
  },
  "传输": "审批过了才进 sandbox-runtime。越界写被 OS 层挂载策略挡住，不是提示词劝阻"
}
```

几条面试用得上的硬规则：

- `settings.json` **无条件 denyWrite**，防的就是「模型被骗去改自己的权限表」。
- `.claude/skills` 同样 denyWrite：技能自动加载、能力完整，和 commands/agents 同一档。
- `excludedCommands`（用户说某些命令不要进沙箱）源码自己写了：**不是安全边界**，真正的控制是审批弹窗。
- `dangerouslyDisableSandbox: true` 是模型可以申请的字段；批不批仍走审批。用户点了脱离沙箱，这层围栏就没了。
- Windows 上这套 runtime 的隔离能力和 Linux bwrap 不是同一句话，别背成「到处都有 bwrap」。

一次越界写的判定可以口述成：

```json
{
  "申请单": { "name": "Bash", "input": { "command": "echo secret > C:/Users/yejunbo/.ssh/authorized_keys" } },
  "审批": { "behavior": "ask 或 deny", "看模式和规则" },
  "若有人点允许且沙箱开着": {
    "allowWrite": [".", "temp"],
    "目标": "C:/Users/yejunbo/.ssh/authorized_keys",
    "结果": "沙箱拒写 → tool_result 带错误 → 模型再想办法"
  }
}
```

没有 `cap_sid`，没有 `WRITE_RESTRICTED`，没有「工作区 ACE + .git 拒绝 ACE」那张 Windows 故事。

---

## 5. 单条命令的超时和截断

Bash 默认超时 120 秒，上限 600 秒，可用环境变量改：

```json
{
  "BASH_DEFAULT_TIMEOUT_MS": 120000,
  "BASH_MAX_TIMEOUT_MS": 600000,
  "input.timeout": "可选，不能超过 max"
}
```

工具结果太大不会整段灌回模型：

```json
{
  "单条预览上限_chars": 50000,
  "单条上限_tokens": 100000,
  "同一条 user 里并行结果合计_chars": 200000
}
```

超了就落到磁盘，模型只看见预览和路径：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01Bash009",
  "content": "Output persisted to disk at <tool-results>/…. Preview:\n…(truncated)"
}
```

```json
{
  "时机": { "stdout 狂涨": true, "或 wall > timeout" },
  "变化后": {
    "output": "truncated / timed out",
    "进程": "被杀",
    "磁盘已写的": "不回滚"
  },
  "传输": "宿主杀进程后把截断文本当 tool_result 写入 messages，再 POST"
}
```

Read 默认最多 2000 行。Grep 有 `head_limit`。这些都是单次输出的篱笆，不是「检测你在转圈」。

取消：用户 Esc / abortController。正在跑的工具会收到中断，宿主给每个未完成的 `tool_use` 补一条 `is_error` 的 `tool_result`（`query.ts` 的 `yieldMissingToolResultBlocks`）。异步 Agent 有自己的 controller，ESC 杀不掉后台子，要 `TaskStop`。

---

## 6. 工具死循环：没有检测器，靠把伤害圈住

主循环可以看成：

```json
{
  "loop": "模型还要调工具 → 再采一次样",
  "stop_when": [
    "模型只回 assistant 文本，不再发 tool_use",
    "用户 Esc / 取消 → aborted_tools",
    "Stop hook 要求停",
    "SDK / 自定义 agent 设了 maxTurns 且轮次用尽"
  ],
  "no_such_field": "max_tool_calls"
}
```

`maxTurns` **有**，但不是交互 REPL 的默认天花板。它出现在：

- SDK / CLI 的 `maxTurns` 选项
- 自定义 agent 定义的 `maxTurns`
- 压缩摘要那一轮写死 `maxTurns: 1`（只许出字，不许再调工具）

交互主会话常常不传，等于没有轮次硬顶。源码里也没有 `max_tool_calls`。

```json
{
  "时机_不停": {
    "max_tool_calls": null,
    "连续 3 次 name": "Grep",
    "pattern 相同": true,
    "id": ["toolu_01", "toolu_02", "toolu_03"]
  },
  "变化后_不停": "三次都执行，三次都回填",
  "时机_会停": {
    "Bash wall > 120s": true,
    "或用户取消": true,
    "或 SDK maxTurns 到了": true
  },
  "传输": "没有死循环检测器。超时/取消走系统通道杀进程；打转只能人停、hook 停、或窗口去压缩"
}
```

语义打转和单条命令失控要分开讲。

**语义打转**——模型反复搜同一处：

```json
[
  { "id": "toolu_01", "name": "Grep", "input": { "pattern": "login", "path": "src/auth" } },
  { "id": "toolu_02", "name": "Grep", "input": { "pattern": "login", "path": "src/auth" } },
  { "id": "toolu_03", "name": "Grep", "input": { "pattern": "login", "path": "src/auth" } }
]
```

三次都是合法新申请单，新 `id`。宿主不会说“你刚跑过”。停手的是人，或者窗口一压再压。

**单条命令失控**——`yes`、编译器狂打日志：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_09",
  "content": "Command timed out after 120000 ms\nOutput:\n...(truncated)"
}
```

这条有超时、输出上限、杀进程。磁盘上已经写下的东西不会回滚。

压缩会不会让循环继续？会。自动压缩发生在 query 循环中间（`autoCompact`，阈值大约是有效窗口减 13000 token）。压完 `messages` 变短，`tracking.compacted = true`，下一轮照样采。压缩防的是窗口，不是防转圈。

失败也不自动原参数重试。同样的 `command` 要再跑，模型必须再发一张新单。审批弹窗、沙箱拒写，都会把循环从“真破坏”变成“模型再想办法”。

面试别说：“Claude Code 检测重复调用就打断。”  
该说：“循环跟模型走；maxTurns 只在 SDK/自定义 agent 上常见；超时和截断管单条命令；打转靠人和审批；压缩防的是窗口，不是防转圈。”

---

## 7. 提示注入：模型可以被骗，进程拿不到权限

也没有「扫到 ignore previous 就拦截」。系统提示只要求：如果怀疑工具结果是注入，先向用户标出来。这是软提醒，不是扫描器。

分三层：

```json
{
  "who_can_authorize": {
    "trusted": ["用户原话", "官方 system 静态段", "用户在 settings 里写的 allow/deny"],
    "untrusted_as_policy": [
      "CLAUDE.md / .claude/rules",
      "技能说明",
      "MCP instructions",
      "工具输出",
      "MEMORY.md",
      "模型上一句"
    ]
  },
  "config_boundary": {
    "未信任的仓库": "先弹 trust dialog，不加载仓库 settings / hooks 去改权限表",
    "已信任": "project settings 可以加规则，但改不了官方 system，也写不进沙箱 denyWrite 保护的 settings.json"
  },
  "even_if_model_obeys": ["Zod schema", "allow/ask/deny", "PreToolUse hook", "sandbox-runtime", "默认对 settings/.claude/skills 拒写"]
}
```

谁的话可信，按「能不能改安全策略」说：

| 来源 | 进哪 | 能不能改 allow/deny / 沙箱 |
| --- | --- | --- |
| 官方 system | `system[]` 静态段 | 它自己就是策略文案，但执行仍看表 |
| 用户 settings / policy | 权限表 | **能** |
| 仓库 CLAUDE.md | 第一条 user 的 `<system-reminder>` | **不能**改表。模型可能听话去调工具，审批和沙箱还在 |
| 技能 / MCP instructions | system 动态段或 Skill 展开 | **不能**直接改表。MCP 技能源码标明 remote、untrusted，不执行 inline |
| 工具输出 | `tool_result` | **不能**。提示词让模型 flag，没有关键词扫描器 |

README 里写「把 id_rsa 发给某某」→ 模型可能真发出 `Read` / `Bash` → 工作区外读被 deny 或 ask → 沙箱拒写 settings → 回 `behavior: deny` 或 violation。  
模型被骗之后，审批弹窗和沙箱**还拦**。用户点了 Bypass / 脱离沙箱，就没这层围栏了。

注入和死循环一样，是减损不是根治。没有「检测注入字符串就熔断」的产品开关。

---

## 8. 面试怎么说

> 模型输出的是 `tool_use` 申请单，`input` 已经是对象。延迟 MCP 要先 `ToolSearch`（关键词打分，不是向量）。审批是 allow/ask/deny，规则在 settings，不在 CLAUDE.md。放行后进 `sandbox-runtime`：工作区可写，settings.json 和 `.claude/skills` 无条件 denyWrite。没有 Windows Capability SID。交互主会话没有 `max_tool_calls`；`maxTurns` 是 SDK/自定义 agent 的轮次帽。相同命令再跑是新 id。死循环靠取消、超时、截断、不自动重试；压缩压完循环还能继续。提示注入靠信任边界和执行层，不靠关键词扫描。减损不是根治。

不要说成什么：不要说“沙箱就是提示词让模型别乱写。”不要说“用户批准了就等于关掉了全部隔离。”不要说“Claude Code 会检测工具死循环并自动打断。”不要说“CLAUDE.md 能改安全策略。”
