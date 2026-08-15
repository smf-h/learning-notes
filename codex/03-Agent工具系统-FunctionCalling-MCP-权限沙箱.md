# Agent 工具系统：用 JSON 看申请单、审批和沙箱

> 同系列：[01 请求结构](./01-Codex发给大模型的完整请求结构.md) · [02 主子 Agent](./02-Codex主子agent机制.md) · [04 记忆](./04-Codex记忆系统-压缩与检索.md)
>
> 对照 Claude Code：[03 工具与沙箱](../claude/03-Agent工具系统-FunctionCalling-MCP-权限沙箱.md) · [对照表](../对照.md)

这篇不靠函数名讲。一次工具调用就是几份 JSON 在变：模型先交申请单，宿主改权限表，Windows 再拿 Token 去对路径上的 ACL。看懂这几张表，就知道沙箱是怎么挡住的。

**阅读约定：** 线上 `function_call.arguments` 是字符串。下面讲解一律写成对象。线格式只在第 1 节对照一次。

## 面试先记住三张表

```text
1. 工具暴露表：模型现在能看见哪些工具
2. 审批表：这一次放不放行
3. Token + ACL：放行后，这个进程打开文件时 Windows 怎么判
```

---

## 1. 模型交上来的是申请单，不是已经执行完的结果

模型想跑命令，发出去的申请单是：

```json
{
  "type": "function_call",
  "call_id": "call_exec_001",
  "name": "exec_command",
  "arguments": {
    "cmd": "echo hello > src\\a.txt",
    "workdir": "C:\\Users\\yejunbo\\Desktop\\codex"
  }
}
```

线上 `arguments` 其实是一段字符串。宿主先 parse 成上面这个对象，再校验、再执行。

```json
{
  "时机": { "模型输出.type": "function_call", "name": "exec_command" },
  "变化前": { "arguments": "{\"cmd\":\"echo hello > src\\\\a.txt\"}" },
  "变化后": { "arguments": { "cmd": "echo hello > src\\a.txt" } },
  "传输": "模型通道先把申请单写入 input；宿主 parse 后走系统通道起进程；结果同 call_id 再 POST 回去"
}
```

申请单进历史，不等于文件已经被写完。模型这时已经结束这一轮采样；真正 `echo` 发生在宿主里。下一轮模型看见的是同 `call_id` 的 output，不是操作系统异常。

工具定义里的 `parameters` 才是对象形态的 schema，用来校验字段，不管安全：

```json
{
  "type": "function",
  "name": "exec_command",
  "parameters": {
    "type": "object",
    "required": ["cmd"],
    "properties": {
      "cmd": { "type": "string" },
      "workdir": { "type": "string" },
      "sandbox_permissions": {
        "type": "string",
        "enum": ["use_default", "require_escalated"]
      }
    }
  }
}
```

`cmd` 是 string，并不等于它只能写工作区。形状对了，只说明申请单填得合法。安全在后面的审批和 Token/ACL，不在这张 schema。

---

## 2. 工具列表 JSON 怎么变：Direct 和 Deferred

本轮请求顶层 `tools` 只放 Direct 工具。延迟工具一开始不在这里。

**第 1 次请求顶层：**

```json
{
  "tools": [
    { "type": "tool_search", "execution": "client", "parameters": { "required": ["query"] } },
    { "type": "function", "name": "exec_command" },
    { "type": "function", "name": "spawn_agent" }
  ]
}
```

日历工具 `mcp__codex_apps__calendar/_create_event` 不在上面。模型要先搜：

```json
{
  "type": "tool_search_call",
  "call_id": "call_search_001",
  "arguments": { "query": "create calendar event", "limit": 1 }
}
```

这里的 `arguments` 已经是对象，和普通 `function_call` 不一样。

宿主回填的不是“创建成功”，而是**工具规格**：

```json
{
  "type": "tool_search_output",
  "call_id": "call_search_001",
  "tools": [
    {
      "type": "namespace",
      "name": "mcp__codex_apps__calendar",
      "tools": [
        {
          "type": "function",
          "name": "_create_event",
          "defer_loading": true,
          "parameters": {
            "required": ["title", "starts_at"]
          }
        }
      ]
    }
  ]
}
```

**第 2 次请求顶层 `tools` 仍然没有日历工具。** Schema 只活在历史里的 `tool_search_output`。模型读完再发真正调用：

```json
{
  "type": "function_call",
  "call_id": "call_cal_001",
  "namespace": "mcp__codex_apps__calendar",
  "name": "_create_event",
  "arguments": { "title": "Lunch" }
}
```

缺字段，宿主不崩，把失败写回同一 `call_id`：

```json
{
  "type": "function_call_output",
  "call_id": "call_cal_001",
  "output": "tool call error: missing field `starts_at`"
}
```

模型自己改参数，**换新 call_id** 再调。没有“原参数自动重试”。

MCP 工具从模型眼里看就是普通 function。真正打回服务器时名字会变回去：

```json
{
  "model_sees": {
    "namespace": "mcp__codex_apps__calendar",
    "name": "_create_event"
  },
  "server_receives": {
    "server": "codex_apps",
    "tool": "calendar_create_event"
  }
}
```

---

## 3. 审批表：这一次放不放行

配置里的审批策略可以看成：

```json
{
  "approval_policy": "on-request",
  "sandbox_mode": "workspace-write"
}
```

四种策略对应四种行为：

```json
[
  { "policy": "untrusted",    "auto_allow": "只有已知安全的只读命令", "else": "问人" },
  { "policy": "on-request",   "auto_allow": "策略允许的", "else": "模型要求升级时问人" },
  { "policy": "granular",     "auto_allow": "按子开关分别决定", "else": "对应开关关掉就直接拒" },
  { "policy": "never",        "auto_allow": "不问人", "else": "该拒就拒，错误回模型" }
]
```

`never` 不是“全部自动允许”。

```json
{
  "时机": {
    "approval_policy": "on-request",
    "sandbox_permissions": "require_escalated"
  },
  "变化前": { "filesystem": "workspace-write", "network": false },
  "变化后_点允许": { "filesystem": "workspace-write", "network": true },
  "变化后_policy=never": { "弹窗": false, "回模型": "rejected" },
  "传输": "审批在宿主 UI / Guardian，不进模型 POST。结果决定这次系统通道带什么 Token/网络"
}
```

`require_escalated` 是模型在申请单里开口要更多权限。人点允许，通常只打开这一次需要的网络或附加写根，不会把 Token 换成完整 yejunbo。`approval=never` 则连窗都不弹，该拒就拒，错误回模型。

模型如果觉得当前沙箱不够，申请单里会多两个字段：

```json
{
  "cmd": "npm install",
  "sandbox_permissions": "require_escalated",
  "justification": "需要访问 registry.npmjs.org 下载依赖"
}
```

批准前后，有效权限是加法，不是一把拨到全开：

```json
{
  "before": {
    "filesystem": "workspace-write",
    "network": false
  },
  "user_clicked": "allow this command with network",
  "after": {
    "filesystem": "workspace-write",
    "network": true,
    "still_not": "danger-full-access"
  }
}
```

执行策略是另一张小表，先给命令分类：

```json
{ "decision": "allow" }
{ "decision": "prompt" }
{ "decision": "forbidden" }
```

`allow` 不等于关掉沙箱。`prompt` 遇上 `approval_policy=never`，直接变成拒绝回模型。

---

## 4. Windows 沙箱：Token 和 ACL 到底长什么样

审批过了，命令还是要进子进程。Windows 上挡住越界读写的，不是提示词，是两张表：

```text
Token = 这个进程出门带的工作证（我是谁、我有哪几张能力卡）
ACL   = 每个文件夹门上贴的规则（哪张卡能进、哪张卡禁止）
Windows 开门时：拿工作证去对门上的表
```

下面用一次真实场景把两张表摊开。

### 4.1 配置先变成“允许写哪些根目录”

用户当前是 workspace-write，工作区是桌面上的 codex：

```json
{
  "sandbox_mode": "workspace-write",
  "cwd": "C:\\Users\\yejunbo\\Desktop\\codex",
  "network": false,
  "extra_writable_roots": [],
  "deny_read": []
}
```

宿主把它收成一张运行时权限单：

```json
{
  "token_mode": "writable_roots",
  "writable_roots": [
    "C:\\Users\\yejunbo\\Desktop\\codex"
  ],
  "protect_subdirs": [
    "C:\\Users\\yejunbo\\Desktop\\codex\\.git",
    "C:\\Users\\yejunbo\\Desktop\\codex\\.codex",
    "C:\\Users\\yejunbo\\Desktop\\codex\\.agents"
  ],
  "readable": "platform_defaults_plus_workspace",
  "network": false
}
```

如果是 `read-only`，`writable_roots` 就是 `[]`，后面会换另一张只读 Token。  
如果是 `danger-full-access`，根本不走这套 Token/ACL。

### 4.2 Capability SID：给每个可写根目录发一张能力卡

Codex 在 `~/.codex/cap_sid` 里存的就是这种 JSON（SID 数字是示意，真实机器上是随机生成后持久化复用）：

```json
{
  "workspace": "S-1-5-21-8844221100-1001-2002-3003",
  "readonly": "S-1-5-21-8844221100-4004-5005-6006",
  "workspace_by_cwd": {
    "C:\\Users\\yejunbo\\Desktop\\codex": "S-1-5-21-8844221100-1001-2002-3003"
  },
  "writable_root_by_path": {
    "C:\\Users\\yejunbo\\AppData\\Local\\Temp\\codex-extra": "S-1-5-21-8844221100-7007-8008-9009"
  }
}
```

读这张表的方式：

| 字段 | 含义 |
| --- | --- |
| `workspace` / `workspace_by_cwd` | “允许写这个工作区”的能力卡 |
| `readonly` | 只读沙箱用的能力卡，路径上不会给它写权限 |
| `writable_root_by_path` | 额外可写根各有各的卡，避免旧目录的 ACL 把新沙箱权限撑大 |

这些 SID **不是** Windows 用户。你在“计算机管理 → 用户”里找不到它们。它们只是贴在 Token 和 ACL 上的同一串号码，用来对上号。

### 4.3 Token 长什么样：从完整工作证剪成受限工作证

Codex 主进程还是你这个用户，Token 接近：

```json
{
  "process": "codex.exe",
  "user": {
    "name": "LAPTOP\\yejunbo",
    "sid": "S-1-5-21-2183995604-111-222-1001"
  },
  "groups": [
    "S-1-5-32-544",
    "S-1-5-32-545",
    "S-1-5-21-2183995604-111-222-1001"
  ],
  "privileges": ["SeChangeNotifyPrivilege", "SeImpersonatePrivilege", "SeIncreaseQuotaPrivilege"],
  "restricting_sids": null,
  "flags": []
}
```

拿着这张证，`C:\Windows`、`C:\Users\yejunbo\.ssh` 只要 ACL 允许 yejunbo，就能写。这就是为什么不能让模型命令继承主进程 Token。

沙箱子进程拿到的是剪过的证。workspace-write 时大致是：

```json
{
  "process": "powershell.exe",
  "created_from": "codex.exe 的 Token",
  "user": {
    "name": "LAPTOP\\yejunbo",
    "sid": "S-1-5-21-2183995604-111-222-1001"
  },
  "flags": [
    "DISABLE_MAX_PRIVILEGE",
    "LUA_TOKEN",
    "WRITE_RESTRICTED"
  ],
  "privileges": ["SeChangeNotifyPrivilege"],
  "restricting_sids": [
    {
      "sid": "S-1-5-21-8844221100-1001-2002-3003",
      "role": "workspace capability，写工作区靠它"
    },
    {
      "sid": "S-1-5-5-0-392817",
      "role": "本次登录 SID，用来建管道"
    },
    {
      "sid": "S-1-1-0",
      "role": "Everyone，读系统公共文件"
    }
  ]
}
```

三个 flag 用人话讲：

| flag | 这张证上发生了什么 |
| --- | --- |
| `DISABLE_MAX_PRIVILEGE` | 高特权基本拔掉，只留遍历目录用的 `SeChangeNotifyPrivilege` |
| `LUA_TOKEN` | 按标准用户来，不走管理员那条完整权限 |
| `WRITE_RESTRICTED` | **写文件时，门上的“允许写”必须写给 restricting_sids 里的某一张卡。** 只写给 `yejunbo` 不够 |

最后一条是沙箱生效的关键。工作区目录本来就有：

```text
LAPTOP\yejunbo:(OI)(CI)(F)
```

你自己双击记事本，用完整 Token，这条就能写。  
沙箱 PowerShell 虽然 Token 里 `user` 还是 yejunbo，但 `WRITE_RESTRICTED` 规定：写操作不能靠 yejunbo 这条 ACE，必须靠 capability SID 那条。工作区上 Codex 补了 capability 的允许写，所以能写；`C:\Windows` 上没有这张卡的允许写，就被拒。

Token 自己**没有路径列表**。它只带着几张卡。路径在不在沙箱里，看门上贴没贴对应 ACE。

### 4.4 ACL 长什么样：每扇门一张表

Codex 会改这些路径的 DACL（Discretionary ACL）。用 JSON 写出来比 `icacls` 好读。

**工作区根目录，允许这张能力卡读写删：**

```json
{
  "path": "C:\\Users\\yejunbo\\Desktop\\codex",
  "dacl": [
    {
      "sid": "S-1-5-21-8844221100-1001-2002-3003",
      "who": "workspace capability",
      "type": "ALLOW",
      "rights": ["GENERIC_READ", "GENERIC_WRITE", "GENERIC_EXECUTE", "DELETE"],
      "inherit": ["OI", "CI"]
    },
    {
      "sid": "S-1-5-21-2183995604-111-222-1001",
      "who": "yejunbo（你本人，给资源管理器/编辑器用）",
      "type": "ALLOW",
      "rights": ["FULL"],
      "inherit": ["OI", "CI"]
    },
    {
      "sid": "S-1-5-18",
      "who": "SYSTEM",
      "type": "ALLOW",
      "rights": ["FULL"],
      "inherit": ["OI", "CI"]
    }
  ]
}
```

`(OI)(CI)`：这条规则会传给下面的文件和子目录。所以 `src\a.txt` 不用再单独贴一遍。

**敏感子目录，同一张卡改成拒绝写：**

```json
{
  "path": "C:\\Users\\yejunbo\\Desktop\\codex\\.git",
  "dacl": [
    {
      "sid": "S-1-5-21-8844221100-1001-2002-3003",
      "who": "workspace capability",
      "type": "DENY",
      "rights": ["WRITE", "DELETE"],
      "inherit": ["OI", "CI"]
    }
  ]
}
```

`.codex`、`.agents` 同样贴 DENY。Windows 判权限时 **DENY 先于 ALLOW**。父目录允许写，子目录仍可单独拒绝。

**工作区以外，比如用户主目录，Codex 不会给 capability 加 ALLOW：**

```json
{
  "path": "C:\\Users\\yejunbo\\.ssh",
  "dacl": [
    {
      "sid": "S-1-5-21-2183995604-111-222-1001",
      "who": "yejunbo",
      "type": "ALLOW",
      "rights": ["FULL"]
    }
  ]
}
```

沙箱进程来写这里：Token 里有 yejunbo，但 `WRITE_RESTRICTED` 不认 yejunbo 这条 ACE；又没有 capability 的 ALLOW → `Access is denied`。

**只读沙箱**换另一张卡，路径上不给它写权限：

```json
{
  "token.restricting_sids": ["S-1-5-21-8844221100-4004-5005-6006", "logon", "Everyone"],
  "workspace_acl_for_this_sid": []
}
```

读公共文件可以靠 Everyone；写任何用户目录都会失败。

**禁止读取**是再贴一条 DENY READ。宿主还会记一份状态，避免换配置后旧 DENY 留着：

```json
{
  "principals": {
    "S-1-5-21-8844221100-1001-2002-3003": [
      "C:\\Users\\yejunbo\\.ssh",
      "C:\\Users\\yejunbo\\.aws"
    ]
  }
}
```

这是 `deny_read_acl_state.json` 的形状。普通受限 Token **读检查不走 WRITE_RESTRICTED 那套**，deny-read 要靠 elevated 专用账户那条路径才硬。配置了 deny-read 却走不了 elevated，Codex 会直接拒绝“裸跑”，而不是假装沙箱住了。

### 4.5 一次写文件：Windows 拿 Token 去对 ACL

模型申请：

```json
{
  "type": "function_call",
  "name": "exec_command",
  "arguments": { "cmd": "Set-Content -Path '.\\src\\a.txt' -Value 'ok'" }
}
```

审批过了，宿主启动：

```json
{
  "CreateProcessAsUserW": {
    "token": "上面那张 WRITE_RESTRICTED Token",
    "application": "powershell.exe",
    "cwd": "C:\\Users\\yejunbo\\Desktop\\codex"
  }
}
```

PowerShell 再去开文件。Windows 内部相当于在做：

```json
{
  "request": {
    "path": "C:\\Users\\yejunbo\\Desktop\\codex\\src\\a.txt",
    "desired_access": "WRITE"
  },
  "token_flags": ["WRITE_RESTRICTED"],
  "token_cards": [
    "S-1-5-21-8844221100-1001-2002-3003",
    "S-1-5-5-0-392817",
    "S-1-1-0"
  ],
  "acl_hits": [
    {
      "ace": "ALLOW WRITE for S-1-5-21-8844221100-1001-2002-3003 (inherited from workspace root)",
      "matches_a_token_card": true
    }
  ],
  "result": "STATUS_SUCCESS"
}
```

再写 `.git\config`：

```json
{
  "request": {
    "path": "C:\\Users\\yejunbo\\Desktop\\codex\\.git\\config",
    "desired_access": "WRITE"
  },
  "acl_hits": [
    {
      "ace": "DENY WRITE for S-1-5-21-8844221100-1001-2002-3003",
      "matches_a_token_card": true,
      "wins_because": "DENY 先于 ALLOW"
    }
  ],
  "result": "ACCESS_DENIED"
}
```

再写 `C:\Users\yejunbo\.ssh\id_rsa`：

```json
{
  "request": {
    "path": "C:\\Users\\yejunbo\\.ssh\\id_rsa",
    "desired_access": "WRITE"
  },
  "acl_hits": [
    {
      "ace": "ALLOW FULL for yejunbo",
      "matches_a_token_card": false,
      "ignored_because": "WRITE_RESTRICTED 写操作不认 user SID，只认 restricting_sids"
    }
  ],
  "result": "ACCESS_DENIED"
}
```

```json
{
  "时机": {
    "sandbox_mode": "workspace-write",
    "token.flags 含": "WRITE_RESTRICTED",
    "desired_access": "WRITE",
    "path": "C:\\Users\\yejunbo\\Desktop\\codex\\src\\a.txt"
  },
  "变化前": { "申请单.cmd": "Set-Content .\\src\\a.txt" },
  "变化后_工作区": { "result": "STATUS_SUCCESS" },
  "时机2.path": "C:\\Users\\yejunbo\\Desktop\\codex\\.git\\config",
  "变化后_git": { "ace": "DENY WRITE for capability SID", "result": "ACCESS_DENIED" },
  "时机3.path": "C:\\Users\\yejunbo\\.ssh\\id_rsa",
  "变化后_ssh": { "yejunbo 的 ALLOW FULL": "WRITE_RESTRICTED 不认", "result": "ACCESS_DENIED" },
  "传输": "系统通道：CreateProcessAsUserW 带着剪过的 Token。失败变成 function_call_output 回模型通道"
}
```

Token 上没有路径列表，只有几张 Capability 卡。工作区目录的 ACL 给这张卡 ALLOW，所以 `src\a.txt` 能写；`.git` 给同一张卡 DENY，DENY 先判；`.ssh` 只有 yejunbo 的 FULL，但 `WRITE_RESTRICTED` 写操作不认 user SID，所以也拒。npm/node/git 是子进程，继承同一张证。

命令失败后回给模型的是普通工具结果，不是异常炸掉整个 turn：

```json
{
  "type": "function_call_output",
  "call_id": "call_exec_001",
  "output": "Wall time: 0.21 seconds\nProcess exited with code 1\nOutput:\nAccess is denied"
}
```

`npm`、`node`、`git` 都是 PowerShell 的子进程，继承同一张 Token。不用再给每个 exe 单独做沙箱。

### 4.6 两条 Windows 路径，JSON 差在哪

**默认：从当前进程剪 Token**

```json
{
  "backend": "restricted_token",
  "who_logs_on": "仍然是 yejunbo，但 Token 被剪过",
  "create": "CreateRestrictedToken → CreateProcessAsUserW"
}
```

**elevated：先换成专用沙箱账户，再剪 Token**

`~/.codex` 下会有用户文件（密码是 DPAPI 密文）：

```json
{
  "version": 3,
  "offline": { "username": "CodexSbxOffline", "password": "AQAAANCMnd8BFdERjHoAwE/Cl+sBAAAA..." },
  "online":  { "username": "CodexSbxOnline",  "password": "AQAAANCMnd8BFdERjHoAwE/Cl+sBAAAA..." }
}
```

```json
{
  "backend": "elevated",
  "who_logs_on": "CodexSbxOffline 或 CodexSbxOnline",
  "create": "CreateProcessWithLogonW → runner 再 CreateRestrictedToken → 真正的 powershell",
  "why_exists": "防火墙断网、deny-read ACL 对读检查也生效"
}
```

离线账户配合防火墙规则，网络默认就是断的。在线账户才允许联网。这和文件 ACL 是两套开关。

### 4.7 升级权限时两张表怎么变

第一次：

```json
{ "cmd": "npm install", "sandbox_permissions": "use_default" }
```

沙箱里没网，失败回模型。模型再申请：

```json
{
  "cmd": "npm install",
  "sandbox_permissions": "require_escalated",
  "justification": "需要访问 registry.npmjs.org"
}
```

你点允许之后，**不是**换成完整 yejunbo Token。常见变化是：

```json
{
  "token": "还是 WRITE_RESTRICTED，writable_roots 不变",
  "network": true,
  "filesystem": "还是只能写工作区"
}
```

只有明确批准“脱离沙箱”时，才会去掉这套 Token/ACL。配置里如果还有 deny-read，宿主不会随手改成完全无沙箱，以免 `.ssh` 上的 DENY 被绕开。

---

## 5. 超时和输出也是 JSON 回填

```json
{
  "type": "function_call_output",
  "call_id": "call_exec_009",
  "output": "Wall time: 30.01 seconds\nProcess timed out\nOutput:\n...(truncated, original 180000 tokens)"
}
```

```json
{
  "时机": {
    "yield_time_ms": 10000,
    "yield_time_ms 有效区间": [250, 30000],
    "max_output_tokens": 10000,
    "Wall time_s": 30.01
  },
  "变化前": { "进程": "还在打日志" },
  "变化后": {
    "output": "Process timed out / truncated",
    "进程组": "已杀"
  },
  "传输": "宿主杀进程组后，把截断文本当 function_call_output 写入 input，再 POST。磁盘已写的不回滚"
}
```

---

## 6. 工具死循环：没有检测器，靠把伤害圈住

主循环可以看成：

```json
{
  "loop": "模型还要调工具 → 再采一次样",
  "stop_when": [
    "模型只回 assistant 文本，不再发 function_call",
    "用户 Esc / 取消 → TurnAborted",
    "Stop hook 要求停"
  ],
  "no_such_field": "max_tool_calls"
}
```

```json
{
  "时机_不停": {
    "max_tool_calls": null,
    "连续 3 次 name": "exec_command",
    "cmd 相同": true,
    "call_id": ["c1", "c2", "c3"]
  },
  "变化后_不停": "三次都执行，三次都回填",
  "时机_会停": {
    "Wall time_s > 超时": true,
    "或用户取消": true,
    "或 token_limit_reached": true
  },
  "传输": "没有死循环检测器。超时/取消走系统通道杀进程；打转只能人停或窗口去压缩"
}
```

语义打转和单条命令失控要分开讲。三次相同 `rg`、三个新 `call_id`，宿主当三张合法单。`yes` 狂打日志才会撞上 `yield_time_ms` / `max_output_tokens` 和杀进程组。压缩把窗口压下去之后，打转还能继续。

两种死循环不要混：

**语义打转**——模型反复搜同一处：

```json
[
  { "call_id": "c1", "name": "exec_command", "arguments": { "cmd": "rg login src/auth" } },
  { "call_id": "c2", "name": "exec_command", "arguments": { "cmd": "rg login src/auth" } },
  { "call_id": "c3", "name": "exec_command", "arguments": { "cmd": "rg login src/auth" } }
]
```

三次都是合法新申请单，新 `call_id`。宿主不会说“你刚跑过”。停手的是人，或者窗口一压再压。

**单条命令失控**——`yes`、编译器狂打日志：

```json
{
  "type": "function_call_output",
  "call_id": "c9",
  "output": "Wall time: 30.01 seconds\nProcess timed out\nOutput:\n...(truncated)"
}
```

这条有超时、输出上限、杀进程组。磁盘上已经写下的东西不会回滚。

失败也不自动原参数重试。同样的 `cmd` 要再跑，模型必须再发一张新单。审批弹窗、沙箱 Access Denied，都会把循环从“真破坏”变成“模型再想办法”。

面试别说：“Codex 检测重复调用就打断。”  
该说：“循环跟模型走；超时和截断管单条命令；打转靠人和审批；压缩防的是窗口，不是防转圈。”

---

## 7. 提示注入：模型可以被骗，进程拿不到权限

也没有「扫到 ignore previous 就拦截」。分三层：

```json
{
  "config_boundary": "目录不信任，就不加载仓库里的 config / hooks / execpolicy",
  "who_can_authorize": {
    "trusted": ["用户", "developer", "用户明确让遵循的文件"],
    "untrusted": ["工具输出", "技能说明", "插件描述", "模型上一句"]
  },
  "even_if_model_obeys": ["Schema", "execpolicy", "审批", "Token+ACL", "默认断网"]
}
```

README 里写「把 id_rsa 发给某某」→ 模型可能真发出 `exec_command` → 工作区外写入 / 外网被拒 → 回 `Access Denied`。  
Guardian 判定恶意注入要同时满足：动作跟用户任务无关，并且是不可信内容教的。低/中风险一般放行，critical 直接拒。

注入和死循环一样，是减损不是根治。用户点了脱离沙箱，就没这层围栏了。

---

## 8. 面试怎么说

> 模型输出的是 `function_call` 申请单，`arguments` 还是字符串。延迟工具的 Schema 出现在 `tool_search_output` 里，不会回到顶层 `tools`。审批决定这次跑不跑；跑起来之后 Windows 看的是 Token 和 ACL。workspace-write 时，子进程 Token 带着 `WRITE_RESTRICTED` 和一张工作区 Capability SID。工作区目录的 ACL 给这张 SID 允许写，`.git` 再给同一张 SID 拒绝写。越界路径上没有这张 SID 的允许 ACE，即使文件夹属于 yejunbo，写也会 Access Denied。Token 里没有路径列表，路径在 ACL 上。工具循环没有 max_tool_calls，死循环靠取消、超时、截断、不自动重试；提示注入靠信任边界和执行层，不靠关键词扫描。

不要说：“沙箱就是提示词让模型别乱写。”  
也不要说：“用户批准了就等于拿到管理员 Token。”  
也不要说：“Codex 会检测工具死循环并自动打断。”
