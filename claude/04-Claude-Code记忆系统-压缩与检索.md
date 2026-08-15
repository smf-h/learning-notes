# Claude Code 记忆：用 JSON 看压缩前后和跨会话检索

> 源码基线：`C:\Users\yejunbo\Desktop\claude-code-source`。本树无 git、无 package.json，版本号由构建期 `MACRO.VERSION` 注入。本文只读分析该仓库，没有修改仓库源码。
>
> 同系列：[01 请求结构](./01-Claude-Code发给大模型的完整请求结构.md) · [02 主子 Agent](./02-Claude-Code主子agent机制.md) · [03 工具与沙箱](./03-Agent工具系统-FunctionCalling-MCP-权限沙箱.md)
>
> 对照 Codex：[04 记忆](../codex/04-Codex记忆系统-压缩与检索.md) · [对照表](../对照.md)

两套管线，别串成一条。下面都用 `messages` 数组怎么变来讲。

**阅读约定：** `input` 写成对象，不写转义字符串。

## 面试先记住

```text
压缩：替换当前会话的 messages，让这轮还能继续说话
长期记忆：~/.claude/projects/<slug>/memory/ 里的 MEMORY.md + 主题文件，给下一个会话用
不是：压缩完写入 MEMORY.md，下次向量检索
也没有 BM25
```

中间还有一条容易讲混的：**SessionMemory** 是当前会话的 `summary.md` 笔记，路径挂在本 session 目录下，不是跨会话登记册。

---

## 1. 为什么要压：活动上下文这张账单

每次采样前可以看成：

```json
{
  "tokenUsage": 118000,
  "effectiveContextWindow": 108000,
  "AUTOCOMPACT_BUFFER_TOKENS": 13000,
  "autoCompactThreshold": 95000,
  "isAboveAutoCompactThreshold": true,
  "isAutoCompactEnabled": true
}
```

数字随模型窗口变，不要背死 95000。判断式是：

```text
effectiveContextWindow = contextWindow(model) - min(maxOutputTokens, 20000)
autoCompactThreshold   = effectiveContextWindow - 13000
触发                   = isAutoCompactEnabled && tokenUsage >= autoCompactThreshold
```

```json
{
  "时机": {
    "tokenUsage": 118000,
    "autoCompactThreshold": 95000,
    "因为": "118000 >= 95000"
  },
  "变化前": { "messages": "过长的活动历史" },
  "变化后": { "进入": "compactConversation", "isAutoCompact": true },
  "传输": "先在宿主算这张账单（query.ts / autoCompact.ts），再另采一轮摘要，然后 replace"
}
```

用户手动 `/compact`：同一套 `compactConversation`，`isAutoCompact=false`，`trigger: "manual"`。警告线再往前 20000 token，只提示不压。

连续失败 3 次（`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`）就停手，避免 prompt_too_long 时狂打 API。

---

## 2. 压缩：历史 JSON 怎么换

压之前（简化）：

```json
{
  "messages": [
    { "role": "user", "content": [{ "type": "text", "text": "先看请求怎么组装" }] },
    {
      "role": "assistant",
      "content": [
        { "type": "tool_use", "id": "toolu_01", "name": "Grep", "input": { "pattern": "paramsFromContext" } }
      ]
    },
    { "role": "user", "content": [{ "type": "tool_result", "tool_use_id": "toolu_01", "content": "claude.ts:1699" }] },
    { "role": "assistant", "content": [{ "type": "text", "text": "外壳在 paramsFromContext" }] },
    { "role": "user", "content": [{ "type": "text", "text": "继续讲压缩" }] }
  ]
}
```

宿主先另发一轮「请写交接摘要」的采样：`maxTurns: 1`，工具调用会被拒，提示词开头就是 `CRITICAL: Respond with TEXT ONLY`。拿到 assistant 文本后，**整段 messages 换成**：

```json
{
  "messages": [
    {
      "role": "user",
      "content": [{
        "type": "text",
        "text": "This session is being continued from a previous conversation that ran out of context. The summary below covers the earlier portion of the conversation.\n\n1. Primary Request and Intent: …\n2. Files and Code Sections: …\n\nIf you need specific details from before compaction (like exact code snippets, error messages, or content you generated), read the full transcript at: C:\\\\Users\\\\yejunbo\\\\.claude\\\\projects\\\\<slug>\\\\<sessionId>.jsonl"
      }]
    }
  ]
}
```

要点：

- 没有 `type=compaction`，没有 `compaction_trigger`。摘要是普通 `role=user` 消息，带固定英文前缀。
- 前面的 `tool_use` / `tool_result` 丢掉了，所以是有损的。
- 完整 transcript 还在磁盘上，摘要末尾会告诉模型路径，需要细节再 `Read`。
- 宿主内部还有 compact boundary 标记（`createCompactBoundaryMessage`），用来以后认出「从哪一段开始算新历史」。发给 API 的活动上下文以摘要 user 消息为主。

```json
{
  "时机": { "chosen": "local_summary", "maxTurns": 1 },
  "变化前": { "messages": ["user", "assistant.tool_use", "user.tool_result", "assistant", "user"] },
  "变化后": {
    "messages": [
      { "role": "user", "text": "This session is being continued…\\n摘要" }
    ]
  },
  "传输": "宿主再 POST 一轮摘要采样 → buildPostCompactMessages replace。不是 append"
}
```

自动压缩发生在工具循环中间。压完 `tracking.compacted = true`，query 继续下一轮。所以：**压缩会让工具死循环有机会接着转**，它只把窗口压下去。

没有「远端 v1 / v2 三选一」。Claude Code 这条主路径就是本地再采一轮摘要。`context_management` beta 是服务端清 thinking 的另一件事，别和 compact 说成同一条。

教学示例若把压缩前工具往返和压缩后摘要排在同一条 `messages` 里，是为了对照。产品一次 compact 只会留下摘要这一侧。

---

## 3. 长期记忆：跨会话的文件目录

后台任务不改当前这次已经发出的 `messages`。它写磁盘，给**下一个**会话用。

默认根目录：

```json
{
  "dir": "C:\\\\Users\\\\yejunbo\\\\.claude\\\\projects\\\\<sanitized-cwd>\\\\memory\\\\",
  "files": {
    "MEMORY.md": "索引，最多 200 行 / 25KB 自动截断后注入",
    "user_role.md": "一条记忆一个文件，带 YAML frontmatter",
    "feedback_testing.md": "主题文件，不是按时间堆",
    "logs/YYYY/MM/YYYY-MM-DD.md": "KAIROS 助手模式下的只追加日志（实验）"
  }
}
```

`MEMORY.md` 是索引，不是正文仓库。提示词要求两步：先 Write 主题文件，再在 `MEMORY.md` 加一行 `- [Title](file.md) — hook`。

frontmatter 形状：

```json
{
  "name": "user_role",
  "description": "用户做国内 Agent 面试讲义，要求 JSON 对照",
  "type": "user"
}
```

`type` 是封闭四类：`user` / `feedback` / `project` / `reference`。能从当前仓库代码推出来的东西，提示词明确说不要存。

谁往里写：

1. **主代理自己**用 `Write` / `Edit`（目录保证已存在，不许先 `mkdir`）。
2. **`extractMemories`**：一轮 query 正常结束后（模型不再调工具），fork 一份对话，后台抽该写的记忆。主代理这轮已经写过，后台会跳过这段。
3. 实验门 `tengu_passport_quail` 关掉时，后台抽取不跑；主代理提示词里的「怎么存」仍在。

```json
{
  "时机": {
    "query 结束": true,
    "isAutoMemoryEnabled": true,
    "feature.EXTRACT_MEMORIES": true
  },
  "变化前": { "本次 messages": "已经发给模型，不再改" },
  "变化后": { "磁盘": "memory/ 下多了主题文件和/或 MEMORY.md 指针" },
  "传输": "runForkedAgent，不占主会话下一包。下次会话的 getUserContext / loadMemoryPrompt 才看见"
}
```

`CLAUDE_CODE_DISABLE_AUTO_MEMORY`、`--bare`、远程且没有 `CLAUDE_CODE_REMOTE_MEMORY_DIR`，会整条关掉。

---

## 4. 新会话看见的不是整个记忆目录

功能打开时，新会话会看到两处，都不是向量库：

**A. system 动态段 `memory`**（`loadMemoryPrompt`）：告诉模型目录在哪、怎么存、不要存什么。`MEMORY.md` 正文在这条路径上可能内嵌，也可能只给指引。

**B. 第一条 user 的 `<system-reminder>`**（`getUserContext` → `claudeMd`）：`MEMORY.md` 走 CLAUDE.md 同一套管线加载，包在 `# claudeMd` 下面。

```json
{
  "role": "user",
  "content": [{
    "type": "text",
    "text": "<system-reminder>\n…\n# claudeMd\n…\n# MEMORY.md\n- [User role](user_role.md) — 面试讲义要 JSON 对照\n…\n# currentDate\nToday's date is 2026-08-15.\n</system-reminder>\n"
  }]
}
```

```json
{
  "时机": { "新会话启动": true, "子 Agent Explore/Plan": false },
  "变化前": { "磁盘": "MEMORY.md" },
  "变化后": {
    "system 动态段": "怎么用记忆",
    "messages[0]": "截断后的 MEMORY.md 正文",
    "顶层静态 system": "不变"
  },
  "传输": "启动时注入。后台 extractMemories 改磁盘，一般赶不上本次已经注入的索引"
}
```

两个时间点：会话一开始，旧的 `MEMORY.md` 已经进上下文；同一时刻后台才去抽刚刚结束的那一轮。所以「这次抽取的记忆」通常要等**下一个**会话才自动看见。Explore/Plan 子代理默认 `omitClaudeMd`，不灌这套。

---

## 5. 检索：不是向量，也不是 BM25

默认没有 `memories.search` 这种专用工具。模型按提示词用通用 `Grep`（底层 ripgrep）搜目录：

```json
{
  "type": "tool_use",
  "id": "toolu_01MemGrep",
  "name": "Grep",
  "input": {
    "pattern": "paramsFromContext|compaction",
    "path": "C:/Users/yejunbo/.claude/projects/<slug>/memory",
    "glob": "*.md",
    "output_mode": "content"
  }
}
```

回执是正则命中行，不是近邻：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_01MemGrep",
  "content": "user_role.md:3: 用户在读 Claude Code 请求结构，要求中文、逐字段。"
}
```

`Grep` 的 `pattern` 是正则。`"压缩"` 不会理解成 `"compaction"`。

另一条可选管线是 `findRelevantMemories`：扫每个记忆文件的 header（文件名 + description），再 **另采一轮 Sonnet**，让它最多挑 5 个文件名：

```json
{
  "sideQuery": {
    "model": "sonnet",
    "system": "You are selecting memories that will be useful… (up to 5)",
    "messages": [
      {
        "role": "user",
        "content": "Query: 检查登录\n\nAvailable memories:\n- user_role.md: 面试讲义要 JSON 对照\n- feedback_testing.md: …"
      }
    ],
    "max_tokens": 256,
    "output_format": {
      "type": "json_schema",
      "schema": {
        "type": "object",
        "required": ["selected_memories"],
        "properties": { "selected_memories": { "type": "array", "items": { "type": "string" } } }
      }
    }
  }
}
```

回来的是文件名列表，不是 embedding 距离：

```json
{ "selected_memories": ["user_role.md"] }
```

选中的文件再当附件 / Read 灌进主会话。`MEMORY.md` 自己已经在上下文里，选择器会排除它。不确定就返回空数组。

```json
{
  "时机": { "用户新问一句": true, "记忆文件数 > 0" },
  "变化前": { "磁盘": "一堆带 description 的 md" },
  "变化后": { "最多 5 个绝对路径": ["…/user_role.md"] },
  "传输": "sideQuery + json_schema。不是向量库，不是 BM25"
}
```

没有 embedding 索引，没有 `match_mode: any/all`。面试说「子串/正则 + 模型挑文件名」即可。

---

## 6. SessionMemory：当前会话的笔记，别当成长期记忆

路径：

```json
{
  "path": "C:\\\\Users\\\\yejunbo\\\\.claude\\\\projects\\\\<slug>\\\\<sessionId>\\\\session-memory\\\\summary.md"
}
```

后台按工具调用次数 / token 阈值，fork 一份对话去更新这个文件。它服务的是**这一次会话**的连贯（包括一种 session-memory 压缩路径），换 session 就换目录。不要把它和 `memory/MEMORY.md` 说成同一张表。

三种东西对照：

| 机制 | 改什么 | 给谁用 |
| --- | --- | --- |
| compact | 当前 `messages` 整段换成摘要 | 这一轮还能继续说话 |
| SessionMemory `summary.md` | 本 session 目录下的笔记文件 | 同一会话后续轮次 / 一种压缩 |
| memdir `MEMORY.md` + 主题文件 | `~/.claude/projects/<slug>/memory/` | **下一个**会话 |

---

## 7. 和 CLAUDE.md 的关系

`MEMORY.md` 走 `getMemoryFiles()`，和仓库 `CLAUDE.md` 一起进 `claudeMd`。所以长期记忆的「自动注入」看起来像又一份项目说明，角色仍是 user 侧 `<system-reminder>`。

它同样**不能**改 `permissions` 或沙箱 `denyWrite`。模型读到「以后默认 bypass」也只是一段不可信文本，见第 3 篇。

---

## 8. 面试怎么说

> 压缩是把当前 `messages` 整段换成更短的：宿主再采一轮、`maxTurns: 1`、只要纯文本，装进去是带固定前缀 `This session is being continued…` 的 user 消息，没有 compaction_trigger。自动压缩看的是有效窗口减 13000。压完工具循环还能继续。长期记忆是 `~/.claude/projects/<slug>/memory/` 里的 `MEMORY.md` 索引加主题文件；新会话只自动注入截断后的索引。检索默认是 `Grep`（ripgrep 正则），另有一条 Sonnet 按文件名/描述最多挑 5 个文件。不是向量库，也不是 BM25。SessionMemory 的 `summary.md` 只服务当前 session。

不要说成什么：不要说“压缩完写入 MEMORY.md，下次语义检索。”不要说“Claude Code 有 compaction_trigger 和远端 v2 密文。”
