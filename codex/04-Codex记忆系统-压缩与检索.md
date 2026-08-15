# Codex 记忆：用 JSON 看压缩前后和跨任务检索

> 同系列：[01 请求结构](./01-Codex发给大模型的完整请求结构.md) · [02 主子 Agent](./02-Codex主子agent机制.md) · [03 工具与沙箱](./03-Agent工具系统-FunctionCalling-MCP-权限沙箱.md)
>
> 对照 Claude Code：[04 记忆](../claude/04-Claude-Code记忆系统-压缩与检索.md) · [对照表](../对照.md)

两套管线，别串成一条。下面都用 `input` 数组怎么变来讲。

**阅读约定：** `arguments` 写成对象，不写转义字符串。

## 面试先记住

```text
压缩：替换当前线程的 input，让这轮还能继续说话
长期记忆：根会话启动时后台写 ~/.codex/memories/，给下一个线程用
不是：压缩完写入 MEMORY.md，下次向量检索
```

---

## 1. 为什么要压：活动上下文这张账单

每次采样前可以看成：

```json
{
  "active_context_tokens": 118000,
  "auto_compact_scope_tokens": 118000,
  "auto_compact_scope_limit": 110000,
  "full_context_window_limit": 128000,
  "token_limit_reached": true,
  "reason": "ContextLimit",
  "phase": "MidTurn"
}
```

```json
{
  "时机": {
    "auto_compact_scope_tokens": 118000,
    "auto_compact_scope_limit": 110000,
    "token_limit_reached": true,
    "因为": "118000 >= 110000"
  },
  "也会触发": {
    "active_context_tokens": 128000,
    "full_context_window_limit": 128000
  },
  "变化前": { "input": "过长的活动历史" },
  "变化后": { "进入": "run_auto_compact", "reason": "ContextLimit", "phase": "MidTurn" },
  "传输": "先在宿主算这张账单，再决定走本地 / v1 / v2，然后才 POST"
}
```

用户手动 `/compact`：`reason=UserRequested`，`phase=StandaloneTurn`。

账单里的数字会随模型和配置变，不要背死 110000。判断式是：`auto_compact_scope_tokens >= auto_compact_scope_limit`，或者整窗 `active_context_tokens >= full_context_window_limit`。`MidTurn` 表示用户这句话已经在历史里、工具还在转；压完要接着循环，不必等用户再回车。

---

## 2. 选哪条压缩：发请求前三选一

```json
{
  "Feature.TokenBudget": false,
  "provider.supports_remote_compaction": true,
  "Feature.RemoteCompactionV2": true,
  "chosen": "remote_v2"
}
```

```json
{ "supports_remote_compaction": false, "chosen": "local_summary" }
```

```json
{
  "supports_remote_compaction": true,
  "RemoteCompactionV2": false,
  "chosen": "remote_v1"
}
```

```json
{
  "时机_本地": { "supports_remote_compaction": false },
  "传输_本地": "普通 POST /v1/responses 要一篇摘要，再 replace",
  "时机_v1": { "supports_remote_compaction": true, "RemoteCompactionV2": false },
  "传输_v1": "POST /v1/responses/compact",
  "时机_v2": { "supports_remote_compaction": true, "RemoteCompactionV2": true },
  "传输_v2": "普通 POST /v1/responses，input[-1]={type:compaction_trigger}"
}
```

一次只走一条。没有「v2 失败自动降 v1」。

---

## 3. 本地压缩：历史 JSON 怎么换

压之前（简化）：

```json
{
  "input": [
    { "type": "message", "role": "user", "content": [{ "type": "input_text", "text": "先看请求怎么组装" }] },
    { "type": "function_call", "call_id": "c1", "name": "exec_command", "arguments": { "cmd": "rg ResponsesApiRequest" } },
    { "type": "function_call_output", "call_id": "c1", "output": "core/src/client.rs:824" },
    { "type": "message", "role": "assistant", "phase": "final_answer", "content": [{ "type": "output_text", "text": "外壳在 build_responses_request" }] },
    { "type": "message", "role": "user", "content": [{ "type": "input_text", "text": "继续讲压缩" }] }
  ]
}
```

客户端先另发一轮“请写交接摘要”的采样，拿到 assistant 文本后，**整段 input 换成**：

```json
{
  "input": [
    {
      "type": "message",
      "role": "user",
      "content": [{ "type": "input_text", "text": "继续讲压缩" }]
    },
    {
      "type": "message",
      "role": "user",
      "content": [{
        "type": "input_text",
        "text": "Another language model started to solve this problem and produced a summary...\n\n已确认请求外壳是 instructions+input+tools；下一步讲压缩三条路径。"
      }]
    }
  ]
}
```

要点：

- 没有 `type=compaction`。摘要是普通 `role=user` 消息。
- 前面固定英文前缀用来以后认出“这是摘要，别再当用户原话收集”。
```json
{
  "时机": { "chosen": "local_summary", "user_retain_budget_tokens": 20000 },
  "变化前": { "input": ["user", "function_call", "function_call_output", "assistant", "user"] },
  "变化后": {
    "input": [
      { "role": "user", "text": "近期原话，从最新往回挑，合计 ≤ 20000 token" },
      { "role": "user", "text": "Another language model started...\\n摘要" }
    ]
  },
  "传输": "宿主再采一轮摘要 → replace_compacted_history。不是 append"
}
```

装进去的摘要是 `role=user`，不是 `type=compaction`。固定前缀用来下次识别，避免把旧摘要再当成用户原话收进 20000 预算。这和远端那条不透明密文不是同一种 item。

---

## 4. 远端 v1：专用端点直接回新的 input

发出去的压缩请求（字段比普通采样少）：

```json
{
  "url": "/v1/responses/compact",
  "body": {
    "model": "gpt-5.4",
    "instructions": "You are a coding agent...",
    "input": ["（当前整段结构化历史）"],
    "tools": ["（当前可见工具）"],
    "parallel_tool_calls": true
  }
}
```

没有 `tool_choice` / `stream` / `include`。服务端回一组新 `ResponseItem`，客户端洗完装上，常见核心是：

```json
{
  "input": [
    {
      "type": "compaction",
      "encrypted_content": "v1:opaque:客户端解不开的密文"
    }
  ]
}
```

客户端不解密。旧 developer、假用户包装会被丢掉。

---

## 5. 远端 v2：在现有 input 尾巴钉一个空对象

普通 `/v1/responses`，历史末尾多一项：

```json
{
  "input": [
    { "type": "message", "role": "developer", "...": "..." },
    { "type": "message", "role": "user", "...": "..." },
    { "type": "compaction", "encrypted_content": "上一次 v2 留下的密文" },
    { "type": "compaction_trigger" }
  ]
}
```

`compaction_trigger` 就是 `{}`，没有 payload。它是控制信号，不是摘要。

服务端必须正好回一个 `compaction`。客户端从压缩前历史里按最新优先留下 user/developer/system，大约 64k token，再把这个新 compaction 接到末尾：

```json
{
  "input": [
    { "type": "message", "role": "user", "content": [{ "type": "input_text", "text": "继续讲压缩" }] },
    { "type": "compaction", "encrypted_content": "v2:opaque:新密文" }
  ]
}
```

```json
{
  "时机": { "implementation": "responses_compaction_v2", "必须回 compaction 个数": 1 },
  "变化前": { "input[-1]": { "type": "compaction_trigger" } },
  "变化后": {
    "保留角色": ["user", "developer", "system"],
    "保留预算_tokens": 64000,
    "末尾": { "type": "compaction", "encrypted_content": "opaque" }
  },
  "传输": "流式 /v1/responses 回来后宿主 replace。旧 trigger 不留"
}
```

v2 的「挑选」按角色和 64000 token 预算留消息，不是语义检索。必须正好回 1 个 compaction，否则整次压缩失败，不会自动改走 v1。

教学示例把本地摘要、v1、v2 排在同一条历史里，是为了对照。产品一次压缩只会留下其中一种替换结果。

---

## 6. 长期记忆：根会话启动时的两份产物

后台任务不改当前这次已经发出的 `input`。它写磁盘和数据库，给**下一个**根线程用。

Phase 1 对每一条合格 rollout，模型被要求吐这种结构：

```json
{
  "raw_memory": "用户在读 Codex 请求结构，要求中文、逐字段、示例必须完整展开。",
  "rollout_summary": "讲解 Responses 请求与三种压缩。",
  "rollout_slug": "codex-request-structure"
}
```

进 State DB 后可以看成：

```json
{
  "thread_id": "019c6e27-e55b-73d1-87d8-4e01f1f75043",
  "raw_memory": "...",
  "rollout_summary": "...",
  "usage_count": 0,
  "last_usage": null,
  "generated_at": 1770000000
}
```

Phase 2 按 `usage_count`、最近使用、是否超过 `max_unused_days` 挑一批，写成：

```json
{
  "dir": "C:\\Users\\yejunbo\\.codex\\memories",
  "files": {
    "memory_summary.md": "短索引，新线程自动注入，大约 2500 token 上限",
    "MEMORY.md": "可搜索登记册，指向 rollout_summaries 和 skills",
    "raw_memories.md": "按 thread_id 升序合并的原始提取",
    "rollout_summaries/2026-08-11T09-42-18-Q7mP-codex-request.md": "单次运行 recap",
    "skills/": "可复用技能",
    "phase2_workspace_diff.md": "相对上次成功 baseline 的 diff，给整合 Agent 看"
  }
}
```

整合 Agent 无网、无审批、只能写这个目录、不能再 spawn。diff 是宿主算好的，不让它自己跑 `git diff`。

---

## 7. 新线程看见的不是整个 MEMORY.md

功能打开时，新根线程的 `input` 里会多一段 developer 文本，形状类似第 1 篇的 `04.01` 第三段：

```json
{
  "type": "message",
  "role": "developer",
  "content": [
    { "type": "input_text", "text": "<permissions instructions>..." },
    { "type": "input_text", "text": "<collaboration_mode>..." },
    {
      "type": "input_text",
      "text": "## Memory\n...目录布局...\n========= MEMORY_SUMMARY BEGINS =========\n项目：C:\\Users\\yejunbo\\Desktop\\codex。\n========="
    }
  ]
}
```

```json
{
  "时机": {
    "根会话启动": true,
    "子 Agent": false,
    "Feature.MemoryTool": true,
    "use_memories": true,
    "memory_summary 上限_tokens": 2500
  },
  "变化前": { "磁盘": "memory_summary.md" },
  "变化后": { "input 里多一段": "role=developer 的 Memory 说明", "顶层 instructions": "不变" },
  "传输": "启动时注入进 input。后台 Phase1/2 改磁盘，一般赶不上本次已经注入的 summary"
}
```

两个时间点：线程一开始，旧的 `memory_summary.md` 已经进 developer；同一时刻后台才去抽更早的 rollout。所以「这次启动生成的记忆」通常要等**下一个**根会话才自动看见。子 Agent 启动不跑这套管线。

---

## 8. 检索：默认没有 memories.search

默认配置：

```json
{ "memories": { "use_memories": true, "dedicated_tools": false } }
```

顶层 `tools` 没有 `memories/search`。模型按提示词用通用命令搜：

```json
{
  "type": "function_call",
  "name": "exec_command",
  "arguments": {
    "cmd": "rg -n -i 'ResponsesApiRequest|compaction' 'C:\\Users\\yejunbo\\.codex\\memories\\MEMORY.md'"
  }
}
```

打开专用工具后，申请单变成：

```json
{
  "namespace": "memories",
  "name": "search",
  "arguments": {
    "queries": ["ResponsesApiRequest", "compaction"],
    "match_mode": { "type": "any" },
    "max_results": 20
  }
}
```

回执是子串命中，不是向量近邻：

```json
{
  "matches": [
    {
      "path": "MEMORY.md",
      "match_line_number": 42,
      "content": "rollout_path: rollout_summaries/2026-08-11T09-42-18-Q7mP-codex-request.md",
      "matched_queries": ["ResponsesApiRequest"]
    }
  ],
  "next_cursor": null,
  "truncated": false
}
```

`match_mode` 只有 `any` / `all_on_same_line` / `all_within_lines`。不会把 “压缩” 理解成 “compaction”。

---

## 9. 用了记忆之后：隐藏 citation JSON

模型最终回答末尾带一块标签。界面会藏起来，宿主解析成：

```json
{
  "entries": [
    {
      "path": "MEMORY.md",
      "lineStart": 42,
      "lineEnd": 47,
      "note": "请求外壳与压缩路径"
    }
  ],
  "rolloutIds": ["019c6e27-e55b-73d1-87d8-4e01f1f75043"]
}
```

只有 `rolloutIds` 会去改数据库：

```json
{
  "update": "stage1_outputs",
  "where": { "thread_id": "019c6e27-e55b-73d1-87d8-4e01f1f75043" },
  "set": { "usage_count": 1, "last_usage": 1770100000 }
}
```

```json
{
  "时机": {
    "assistant.phase": "final_answer",
    "正文含": "<oai-mem-citation>",
    "rollout_ids[0]": "019c6e27-e55b-73d1-87d8-4e01f1f75043"
  },
  "变化前": { "stage1.usage_count": 0, "last_usage": null },
  "变化后": { "usage_count": 1, "last_usage": 1770100000 },
  "传输": "宿主从 assistant 文本里拆标签。只有合法 UUID 去 UPDATE。citation_entries 不进 SQL。失败忽略，不挡回答"
}
```

搜过文件但没写 `rollout_ids`，计数不加。找不到这行 UUID，不会新建记忆。这个数会反过来影响下次 Phase 2 先挑谁。

用户要改记忆时，提示词要求只追加小文件，不直接改 `MEMORY.md`：

```json
{
  "write": "C:\\Users\\yejunbo\\.codex\\memories\\extensions\\ad_hoc\\notes\\2026-08-15-correct-cwd.md",
  "content": "工作区是 Desktop\\codex，不要再写 Documents。"
}
```

真正合并进 `MEMORY.md` 要等下一轮 Phase 2。

---

## 10. 面试怎么说

> 压缩是把当前 `input` 整段换成更短的。本地压完是“近期 user + 一段带固定前缀的摘要 user 消息”；v1 是专用端点回 opaque `compaction`；v2 是在 input 末尾加空的 `compaction_trigger`，再换上一个新密文。长期记忆是根会话启动时后台抽 rollout、整合成 `memory_summary.md` 和 `MEMORY.md`。新线程只自动注入短 summary。默认用 shell 搜 `MEMORY.md`，不是向量库。citation 里的 `rollout_ids` 才给 `usage_count` 加一。

不要说：“压缩完写入 MEMORY.md，下次语义检索。”
