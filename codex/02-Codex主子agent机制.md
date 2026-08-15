# Codex 主子 Agent：用 JSON 看创建、等待和汇报

> 同系列：[01 请求结构](./01-Codex发给大模型的完整请求结构.md) · [03 工具与沙箱](./03-Agent工具系统-FunctionCalling-MCP-权限沙箱.md) · [04 记忆](./04-Codex记忆系统-压缩与检索.md)
>
> 对照 Claude Code：[02 主子 Agent](../claude/02-Claude-Code主子agent机制.md) · [对照表](../对照.md)

子 Agent 不是函数返回值。看三份 JSON 怎么接力：`spawn_agent` 的参数 → 同步回执 → 稍后单独进来的 `agent_message`。

**阅读约定：** 线上 `function_call.arguments` 和 `function_call_output.output` 是字符串。正文里一律写成已经解析好的对象，避免满屏 `\"`。只有第 1 篇开篇那份完整请求保持线格式。

## 面试先记住

```text
spawn 回执 ≠ 任务结果
wait 回执 ≠ 任务结果
真正正文在 type=agent_message
```

---

## 1. 主代理发出去的申请单

用户说：“检查登录并修复。”主代理拆任务后，同一轮可以并行发出两个调用（`parallel_tool_calls: true`）：

```json
{
  "type": "function_call",
  "call_id": "call_spawn_security_001",
  "name": "spawn_agent",
  "arguments": {
    "task_name": "security",
    "message": "只分析 src/auth 的认证问题，不改代码。报告必须带文件、行号、风险等级。",
    "fork_turns": "none",
    "agent_type": "researcher"
  }
}
```

字段就这些：

| 字段 | 不传会怎样 |
| --- | --- |
| `task_name` | 必填，树上的短名 |
| `message` | 必填，子 Agent 收到的第一句话 |
| `fork_turns` | 默认 `"all"`。只能是 `"none"` / `"all"` / `"2"` 这种正整数字符串 |
| `agent_type` | 不传就用默认角色 |
| `model` / `reasoning_effort` / `service_tier` | 不传就继承父 turn |

多传未知字段会直接失败，不会忽略。`fork_turns` 写成 `"0"` 或 `"maybe"` 也会失败。

```json
{
  "时机": {
    "parallel_tool_calls": true,
    "name": "spawn_agent",
    "arguments.fork_turns": "none",
    "arguments.task_name": "security"
  },
  "变化前": { "父线程": { "agent_path": "/root", "depth": 0 } },
  "变化后": {
    "子配置.depth": 1,
    "子配置.agent_path": "/root/security",
    "同步 output": { "task_name": "/root/security" }
  },
  "传输": "申请单走父线程这次 POST 的模型通道；宿主占槽后拉起独立 Thread；回执同 call_id 写回父 input"
}
```

主代理并没有「调用一个函数拿到返回值」。它只是在自己的模型通道里发出一张申请单。宿主占到槽才创建子线程；立刻写回父 input 的只有 `/root/security` 这个名字。子线程之后自己再 POST，和父这轮采样已经脱钩。

---

## 2. 子 Agent 的配置 JSON 怎么叠出来

父代理当前 turn 可以看成：

```json
{
  "thread_id": "019a1000-...-0002",
  "agent_path": "/root",
  "model": "gpt-5.4",
  "reasoning_effort": "medium",
  "cwd": "C:\\Users\\yejunbo\\Desktop\\codex",
  "approval_policy": "on-request",
  "sandbox_mode": "workspace-write",
  "developer_instructions": "...",
  "depth": 0
}
```

`fork_turns=none` 且带了 `agent_type` 时，叠完是：

```json
{
  "thread_id": "（新分配）",
  "agent_path": "/root/security",
  "parent_thread_id": "019a1000-...-0002",
  "depth": 1,
  "agent_role": "researcher",
  "model": "gpt-5.4",
  "reasoning_effort": "medium",
  "cwd": "C:\\Users\\yejunbo\\Desktop\\codex",
  "approval_policy": "on-request",
  "sandbox_mode": "workspace-write",
  "history": []
}
```

注意：cwd、审批、沙箱默认跟父代理一样。子 Agent 不是另一个 Windows 用户，也不是另一套沙箱。多个子 Agent 同时改同一个文件会互相踩，是因为权限表本来就是同一张。角色 `researcher` 只叠提示词和行为默认值，不会把 `apply_patch` 从工具列表里拿掉。要硬挡住写文件，得把 `sandbox_mode` 改成 `read-only`，那是第 3 篇的系统通道。

如果这次还写了 `"model":"某个不存在的模型"`，不会偷偷换备用模型，而是：

```json
{
  "type": "function_call_output",
  "call_id": "call_spawn_security_001",
  "output": "model `某个不存在的模型` is not available"
}
```

`fork_turns=all` 时不能同时改角色和模型。传了会被拒：

```json
{
  "fork_turns": "all",
  "agent_type": "researcher",
  "model": "gpt-5.4"
}
```

```json
{
  "output": "Full-history forked agents inherit the parent agent type, model, and reasoning effort; omit agent_type, model, and reasoning_effort, or spawn without a full-history fork."
}
```

并发/深度超了也是同一类回执，不会排队偷偷创建：

```json
{
  "config": {
    "max_concurrent_threads_per_session": 4,
    "max_threads": 8,
    "max_depth": 3
  },
  "this_request": { "depth": 4 },
  "output": "spawn rejected: max depth exceeded"
}
```

---

## 3. `fork_turns`：历史 JSON 哪些留下

父历史里通常有一串异构 item。`fork_turns=all` 不是整份拷贝。过滤后只剩：

```json
[
  { "type": "message", "role": "developer", "keep": true },
  { "type": "message", "role": "user", "keep": true },
  { "type": "message", "role": "assistant", "phase": "final_answer", "keep": true },

  { "type": "function_call", "keep": false },
  { "type": "function_call_output", "keep": false },
  { "type": "reasoning", "keep": false },
  { "type": "agent_message", "keep": false },
  { "type": "compaction", "keep": false }
]
```

```json
{
  "时机": { "fork_turns": "none" },
  "变化后.history": [],
  "时机2": { "fork_turns": "all" },
  "变化后2": "只留 developer/user/assistant.final_answer，丢掉 function_call 和 agent_message",
  "时机3": { "fork_turns": "3" },
  "变化后3": "先截最近 3 个 turn，再按同一张 keep 表滤",
  "传输": "fork 发生在宿主创建子线程时，拷的是父 rollout 快照，不是再向模型要一份"
}
```

`none` 适合「独立审查、不要被父分析带偏」。`all` 适合「核对整条链，需要和父看到同一批用户原话」，但会丢掉工具中间态，也不能同时改模型/角色。`"3"` 是折中：只要最近三次决策背景。长期记忆不在这张 keep 表里。

长期记忆（`MEMORY.md`）不在这张表里，那是另一套管线。

---

## 4. spawn 的同步回执只有名字

创建成功立刻回：

```json
{
  "type": "function_call_output",
  "call_id": "call_spawn_security_001",
  "output": {
    "task_name": "/root/security",
    "nickname": "security-review"
  }
}
```

没有报告正文。正文还没开始写。

---

## 5. `wait_agent` 的三种回执

```json
{
  "type": "function_call",
  "call_id": "call_wait_001",
  "name": "wait_agent",
  "arguments": { "timeout_ms": 30000 }
}
```

```json
{
  "时机": { "timeout_ms": 30000, "合法区间": [10000, 3600000], "不传则": 30000 },
  "变化前": { "父邮箱": "空或未读" },
  "变化后_有活动": { "message": "Wait completed.", "timed_out": false },
  "变化后_用户插话": { "message": "Wait interrupted by new input.", "timed_out": false },
  "变化后_到期": { "message": "Wait timed out.", "timed_out": true },
  "传输": "wait 只订阅父线程 input_queue；30 秒内邮箱有变就醒。正文不在这条 output 里"
}
```

等的是父线程邮箱有没有动静，不是“所有子 Agent 都结束”。回执只有三种文案。`30000` 是毫秒；小于 `10000` 或大于 `3600000` 直接报错回模型，不会悄悄改成默认值。`timed_out=true` 只说明这 30 秒没响，子线程可能还在跑。

```json
{ "message": "Wait completed.", "timed_out": false }
```

```json
{ "message": "Wait interrupted by new input.", "timed_out": false }
```

```json
{ "message": "Wait timed out.", "timed_out": true }
```

没有 `payload`，没有子 Agent 的分析结论。

主代理还可以发：

```json
{ "name": "list_agents", "arguments": {} }
```

```json
{
  "agents": [
    { "agent_name": "/root/security", "agent_status": "completed" },
    { "agent_name": "/root/tests", "agent_status": "running" }
  ]
}
```

```json
{
  "name": "send_message",
  "arguments": { "task_name": "security", "message": "补上复现命令" }
}
```

```json
{
  "name": "followup_task",
  "arguments": { "task_name": "security", "message": "按刚才格式重写报告" }
}
```

```json
{
  "name": "interrupt_agent",
  "arguments": { "task_name": "security" }
}
```

这些都是协作层 JSON，不是文件系统隔离。

---

## 6. 真正正文：稍后单独进来的 `agent_message`

子 Agent 自己跑完一轮模型循环后，父历史里多出一条**新类型**，不再用 spawn 那个 `call_id`：

```json
{
  "type": "agent_message",
  "author": "/root/security",
  "recipient": "/root",
  "content": [
    {
      "type": "input_text",
      "text": "Message Type: FINAL_ANSWER\nTask name: /root/security\nSender: /root/security\nPayload:\n任务：检查 src/auth\n结论：JWT 未校验 exp\n证据：src/auth/jwt.rs:88\n修改情况：未修改\n验证情况：未跑测试"
    }
  ]
}
```

三份 JSON 对照：

| 时间 | JSON | 里面有没有报告 |
| --- | --- | --- |
| T0 | `function_call` spawn | 只有任务说明 |
| T0+ε | `function_call_output` `{"task_name":"/root/security"}` | 没有 |
| T1 | `function_call` wait | 没有 |
| T1+ε | `function_call_output` `Wait completed.` | 没有 |
| T1+ε | `agent_message` | **有** |

```json
{
  "时机": { "子线程.status": "completed", "trigger_turn": false },
  "变化前": { "04.17": { "task_name": "/root/security" }, "父 input 无正文" },
  "变化后": {
    "type": "agent_message",
    "author": "/root/security",
    "recipient": "/root",
    "payload_token_预算": 1000
  },
  "传输": "宿主 completion watcher → 父邮箱 → 写入父 input。不走 spawn/wait 的 call_id。trigger_turn=false 所以不会单独再踢一轮，等父自己下次 POST 才看见"
}
```

正文有大约 1000 token 的包装预算，超了会裁。父代理 wait 醒来后，下一轮采样才在 `input` 里看见这条 `agent_message`。不要把 04.17 的 `task_name` 和 04.19 的 Payload 说成同一次返回。

如果走加密形态，content 会拆成两段：

```json
{
  "type": "agent_message",
  "author": "/root/security",
  "recipient": "/root",
  "content": [
    {
      "type": "input_text",
      "text": "Message Type: MESSAGE\nTask name: /root\nSender: /root/security\nPayload:\n"
    },
    {
      "type": "encrypted_content",
      "encrypted_content": "v1:opaque:..."
    }
  ]
}
```

教学示例用明文，所以只有一段 `input_text`。

---

## 7. 角色只改提示，不改权限表

```json
{
  "agent_type": "researcher",
  "role_prompt": "你是安全研究员，默认只分析不改代码",
  "task_message": "检查 src/auth，只分析",
  "runtime": {
    "sandbox_mode": "workspace-write",
    "tools_still_include": ["exec_command", "apply_patch"]
  }
}
```

角色提示词是软的。子 Agent 仍可能发出：

```json
{
  "name": "apply_patch",
  "arguments": {
    "command": ["apply_patch", "*** Begin Patch..."]
  }
}
```

宿主不会因为 `agent_type=researcher` 就拒掉这个调用。要硬挡住，得把运行时改成只读：

```json
{
  "sandbox_mode": "read-only",
  "token_mode": "readonly_capability",
  "writable_roots": []
}
```

这时 `apply_patch` 或写文件的 shell 会在 Windows 那一层 Access Denied，见 [03](./03-Agent工具系统-FunctionCalling-MCP-权限沙箱.md)。

---

## 8. 面试怎么说

> 主代理调 `spawn_agent`，参数是 task_name、message、fork_turns。同步返回只有 `/root/security` 这个名字。子配置从父 turn 继承 cwd 和沙箱，再叠角色。`wait_agent` 三种字符串，不含正文。正文是后来的 `agent_message`，靠 author/recipient 认人，不靠 spawn 的 call_id。角色是提示词；并发、深度、模型校验、沙箱才是硬的。

不要说：“wait 把子 Agent 的答案当返回值拿回来。”
