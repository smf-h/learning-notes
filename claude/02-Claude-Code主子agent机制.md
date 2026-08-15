# Claude Code 主子 Agent：用 JSON 看创建、等待和汇报

> 源码基线：`C:\Users\yejunbo\Desktop\claude-code-source`。本树无 git、无 package.json，版本号由构建期 `MACRO.VERSION` 注入。本文只读分析该仓库，没有修改仓库源码。
>
> 同系列：[01 请求结构](./01-Claude-Code发给大模型的完整请求结构.md) · [03 工具与沙箱](./03-Agent工具系统-FunctionCalling-MCP-权限沙箱.md) · [04 记忆](./04-Claude-Code记忆系统-压缩与检索.md)
>
> 对照 Codex：[02 主子 Agent](../codex/02-Codex主子agent机制.md) · [对照表](../对照.md)

Claude Code **没有** Codex 那套 `spawn_agent` + `wait_agent`。委派就是一个工具，名字叫 `Agent`（权限规则里旧名还叫 `Task`）。默认父 turn **同步卡住**，子代理最后一段文本写进**同一张** `tool_result`。异步只是同一工具上的开关，立刻回执里没有正文。

**阅读约定：** `tool_use.input` 写成对象。线格式已在第 1 篇对照过。

## 面试先记住

```text
默认：Agent 的 tool_result 就是子代理最后一段话
异步：立刻回执 ≠ 任务结果
真正异步正文在稍后一条 user 消息的 <task-notification>
没有 wait_agent
```

---

## 1. 主代理发出去的申请单

用户说：“检查登录并修复。”主代理可以在同一条 assistant 消息里并排多个 `Agent`（工具层并发，`isConcurrencySafe() === true`）：

```json
{
  "type": "tool_use",
  "id": "toolu_01AgentExplore",
  "name": "Agent",
  "input": {
    "description": "Explore auth bugs",
    "prompt": "只分析 src/auth 的认证问题，不改代码。报告必须带文件、行号。",
    "subagent_type": "Explore",
    "model": "haiku"
  }
}
```

字段就这些（fork / teammate 实验还会多 `name` / `team_name` / `isolation` / `cwd`）：

| 字段 | 不传会怎样 |
| --- | --- |
| `description` | 必填，3–5 词，给 UI / 通知用 |
| `prompt` | 必填，子代理收到的第一句话 |
| `subagent_type` | 省略：fork 实验开 → 走 fork；关 → `general-purpose` |
| `model` | `sonnet` / `opus` / `haiku`。不传就用 agent 定义，再没有就继承父 |
| `run_in_background` | 不传则同步（除非 coordinator / fork / agent.background 强制异步） |

```json
{
  "时机": {
    "name": "Agent",
    "input.subagent_type": "Explore",
    "run_in_background": false
  },
  "变化前": { "父 query 循环": "正在这一轮" },
  "变化后": {
    "父 turn": "卡住，直到子 query() 结束",
    "子请求": { "messages": [{ "role": "user", "content": "只分析 src/auth…" }] }
  },
  "传输": "申请单走父这次 Messages POST；宿主在同进程 runAgent()；回执同 tool_use_id 写回父 messages"
}
```

主代理并没有「调用一个函数拿到返回值」——它只是发出一张申请单。和 Codex 不同的是：默认情况下宿主**不会立刻松手**，父的这一轮采样已经结束，但父的 query 循环还占着，等子跑完才写 `tool_result`。

---

## 2. 子 Agent 的配置 JSON 怎么叠出来

父当前 turn 可以看成：

```json
{
  "session": "repl_main_thread",
  "model": "claude-sonnet-4-6",
  "cwd": "C:\\Users\\yejunbo\\Desktop\\claude-code-source",
  "permissionMode": "default",
  "tools": ["Agent", "Bash", "Read", "Edit", "Write", "Grep", "Glob", "…"],
  "thinking": { "type": "adaptive" }
}
```

指定了 `subagent_type: "Explore"` 之后，子侧是：

```json
{
  "agentId": "aa3f2c1b4d5e6f7a8",
  "agentType": "Explore",
  "model": "haiku",
  "cwd": "C:\\Users\\yejunbo\\Desktop\\claude-code-source",
  "permissionMode": "acceptEdits",
  "thinking": { "type": "disabled" },
  "system": ["You are an agent for Claude Code…", "<env cwd/platform/date>"],
  "messages": [{ "role": "user", "content": "只分析 src/auth…" }],
  "tools": ["Read", "Grep", "Glob", "Bash", "WebSearch", "…"],
  "omitClaudeMd": true,
  "commands": []
}
```

注意：

- 子 **不继承父 transcript**。第一条 user 就是 `prompt` 字符串。
- system 是 agent 自己的 `getSystemPrompt()`，再补环境细节，不是父那一整段 CLI 提示。
- 工具是 `assembleToolPool` 按子的 `permissionMode` 重装的。父把某个工具藏起来，子不一定跟着藏。
- 权限模式默认 `'acceptEdits'`，不是父的 `default`。父如果已经是 `bypassPermissions` / `acceptEdits` / `auto`，不会被压下去。
- Explore / Plan 会去掉 `claudeMd` 和 `gitStatus`，避免把仓库规矩整段灌进一次性调研。
- 外部构建里子通常不能再调 `Agent`（`ALL_AGENT_DISALLOWED_TOOLS`）。嵌套是 ant 内部能力。
- 角色提示是软的。Explore 提示词说「不要改文件」，但工具列表里若还有 `Edit`/`Write`，宿主不会因为 `subagent_type=Explore` 就拒写。要硬挡住得靠 agent 定义里的 `tools` / `disallowedTools`，或沙箱，见第 3 篇。

`agentId` 形状是 `a` + 16 位 hex，给 `SendMessage({to})` 用，不要念给用户听。

---

## 3. 默认同步：回执就是正文

创建并跑完，同一 `tool_use_id` 回来的内部对象是：

```json
{
  "status": "completed",
  "prompt": "只分析 src/auth 的认证问题…",
  "agentId": "aa3f2c1b4d5e6f7a8",
  "agentType": "Explore",
  "content": [
    { "type": "text", "text": "Found null deref in src/auth/session.ts:42 …" }
  ],
  "totalToolUseCount": 7,
  "totalDurationMs": 18420,
  "totalTokens": 12890
}
```

`finalizeAgentTool` 只抽 **最后一条 assistant 的 text**。中间 Read/Grep/Bash 的噪声留在 sidechain 文件里，不回灌父上下文。

压给父模型看的 `tool_result`：

**Explore / Plan（一次性内置，省 token）：**

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01AgentExplore",
  "content": [
    { "type": "text", "text": "Found null deref in src/auth/session.ts:42 …" }
  ]
}
```

**general-purpose（还可以 SendMessage 续跑）：**

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01AgentExplore",
  "content": [
    { "type": "text", "text": "Replaced session lookup and added a test." },
    {
      "type": "text",
      "text": "agentId: aa3f2c1b4d5e6f7a8 (use SendMessage with to: 'aa3f2c1b4d5e6f7a8' to continue this agent)\n<usage>total_tokens: 12890\ntool_uses: 7\nduration_ms: 18420</usage>"
    }
  ]
}
```

空输出时会塞一句 `(Subagent completed but returned no output.)`。有 worktree 时 trailer 再加 `worktreePath` / `worktreeBranch`。

这一条路径上：**同步回执 = 最终正文**。不要把 Codex 的「spawn 回执只有名字」安过来。

---

## 4. 异步：立刻回执只有收据

`run_in_background: true`，或 agent 定义写了 `background: true`，或 coordinator / fork 实验把全部 spawn 强制异步时：

```json
{
  "status": "async_launched",
  "agentId": "aa3f2c1b4d5e6f7a8",
  "description": "Explore auth bugs",
  "prompt": "只分析 src/auth…",
  "outputFile": "<task-output-dir>/aa3f2c1b4d5e6f7a8",
  "canReadOutputFile": true
}
```

父立刻看见的是收据，不是报告：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01AgentExplore",
  "content": [
    {
      "type": "text",
      "text": "Async agent launched successfully.\nagentId: aa3f2c1b4d5e6f7a8 (internal ID - do not mention to user. Use SendMessage with to: 'aa3f2c1b4d5e6f7a8' to continue this agent.)\nThe agent is working in the background. You will be notified automatically when it completes.\noutput_file: /.../aa3f2c1b4d5e6f7a8"
    }
  ]
}
```

```json
{
  "时机": { "shouldRunAsync": true },
  "变化前": { "父 query": "还在这一轮" },
  "变化后": {
    "tool_result.status": "async_launched",
    "正文": null,
    "子 abortController": "独立，ESC 杀不掉"
  },
  "传输": "registerAsyncAgent 后立刻 return；子在 void 里自己跑 query()"
}
```

同步跑着也可以中途变异步：用户点 background，或自动 120 秒（`CLAUDE_AUTO_BACKGROUND_TASKS` / GrowthBook）。那时当前 `tool_result` 立刻改成 `async_launched`，剩下的循环在后台继续。

**没有 `wait_agent`。** 异步完成后靠下一条 user 消息，不是再调一个 wait 工具。过时的 `TaskOutput({task_id, block, timeout})` 还能堵一会儿，prompt 已标明 deprecated。

---

## 5. 真正异步正文：稍后单独进来的 `<task-notification>`

子跑完后，宿主 `enqueueAgentNotification`，下一轮以 **user-role** 注入。前缀是 `A background agent completed a task:`，不是用户打的字：

```json
{
  "role": "user",
  "content": [
    {
      "type": "text",
      "text": "A background agent completed a task:\n<task-notification>\n<task-id>aa3f2c1b4d5e6f7a8</task-id>\n<tool-use-id>toolu_01AgentExplore</tool-use-id>\n<output-file>/.../aa3f2c1b4d5e6f7a8</output-file>\n<status>completed</status>\n<summary>Agent \"Explore auth bugs\" completed</summary>\n<result>Found null deref in src/auth/session.ts:42 …</result>\n<usage><total_tokens>12890</total_tokens><tool_uses>7</tool_uses><duration_ms>18420</duration_ms></usage>\n</task-notification>"
    }
  ]
}
```

三份 JSON 对照：

| 时间 | JSON | 里面有没有报告 |
| --- | --- | --- |
| T0 | `tool_use` Agent | 只有任务说明 |
| T0+ε | `tool_result` `async_launched` | 没有 |
| T1 | `role=user` `<task-notification>` | **有** |

```json
{
  "时机": { "子.status": "completed" },
  "变化前": { "04.06": { "status": "async_launched" }, "父 messages 无正文" },
  "变化后": {
    "role": "user",
    "标签": "task-notification",
    "不再用": "toolu_01AgentExplore 当回执 id"
  },
  "传输": "enqueuePendingNotification({mode:'task-notification'}) → 下一轮当 user 附件。不走原来的 tool_result"
}
```

`status` 还可以是 `failed` / `killed`。killed 时 `<result>` 是抽出的最后一段残文。提示词要求父代理 **不要去 Read `output_file`**，除非用户明确要看进度——读了就把子的工具噪声灌回父窗口，fork 就白做了。

---

## 6. 续跑、停、多孩子：协作层 JSON，不是另一套协议

还活着或已经停了，用 `SendMessage`，不是再 `Agent` 一次：

```json
{
  "type": "tool_use",
  "id": "toolu_01Resume",
  "name": "SendMessage",
  "input": {
    "to": "aa3f2c1b4d5e6f7a8",
    "summary": "fix the null pointer",
    "message": "Fix the null pointer in src/auth/session.ts:42 and add a test."
  }
}
```

立刻回执（还在跑）：

```json
{
  "success": true,
  "message": "Message queued for delivery to aa3f2c1b4d5e6f7a8 at its next tool round."
}
```

已经停了：从磁盘 transcript 续跑，再挂一条异步生命周期，你会再收到一条 `<task-notification>`。

停：

```json
{ "name": "TaskStop", "input": { "task_id": "aa3f2c1b4d5e6f7a8" } }
```

`TaskCreate` / `TaskGet` / `TaskList` / `TaskUpdate` 是 **todo 列表**，不是 spawn/wait。别把 `Task*` 听成 Codex 的子线程 API。

多个孩子：同一条 assistant 里放多个 `Agent` 块；或先异步发射再继续跟用户说话。没有「wait 所有孩子」的专用工具。

---

## 7. Fork 和 teammate：两条旁路，不要讲成默认

**Fork**（省略 `subagent_type` 且 fork 实验开）：

- 继承父已经渲染好的 system 字节，为了 prompt cache。
- 继承父完整 messages；父侧那些 `tool_use` 的 `tool_result` 先填占位 `"Fork started — processing in background"`。
- 再追加一条指令：`Your directive: {prompt}`。
- 强制异步。禁止再 fork。
- 不要改 `model`：换模型就复用不了父的 cache。

**Teammate**（传了 `name` + `team_name`）：长寿命进程，mailbox 通信，立刻回执类似「Spawned successfully, agent_id / tmux pane」。这不是一次性子代理，面试别当主路径讲。

---

## 8. 面试怎么说

> 工具名是 `Agent`，旧名 `Task`。参数是 description、prompt、可选 subagent_type。默认父 turn 卡住，同一张 `tool_result` 里就是子代理最后一段文本，Explore/Plan 连 agentId trailer 都省掉。异步时立刻回执只有 `async_launched` 和 agentId，正文是稍后一条伪装成 user 的 `<task-notification>`。没有 `wait_agent`。续跑用 `SendMessage({to})`。子默认不继承父历史，权限默认 acceptEdits，thinking 关掉。角色提示是软的。

不要说成什么：不要说“wait 把子 Agent 的答案当返回值拿回来。”也不要说“Claude Code 也有 spawn_agent，回执只有 /root/security 这个名字。”
