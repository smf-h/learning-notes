# Claude Code（四篇）

> 源码基线：`C:\Users\yejunbo\Desktop\claude-code-source`。本树无 git、无 package.json，版本号由构建期 `MACRO.VERSION` 注入。只读分析，没有改仓库源码。
>
> 对照 Codex：[系列索引](../codex) · [对照表](../对照.md)

读完应能讲清：Messages 请求怎么组、子 Agent 默认同步（异步时收据不是正文）、工具从申请单走到审批和沙箱、压缩和跨会话记忆为什么是两条管线。

## 阅读顺序

| 顺序 | 文件 | 面试里它回答什么 |
| --- | --- | --- |
| 1 | [01 请求结构](./01-Claude-Code发给大模型的完整请求结构.md) | 发给模型的是 `system + messages + tools` |
| 2 | [02 主子 Agent](./02-Claude-Code主子agent机制.md) | 没有 `spawn_agent`；默认 `tool_result` 就是正文 |
| 3 | [03 工具与沙箱](./03-Agent工具系统-FunctionCalling-MCP-权限沙箱.md) | 暴露、审批、沙箱是三层 |
| 4 | [04 记忆](./04-Claude-Code记忆系统-压缩与检索.md) | 压缩保当前会话，长期记忆跨会话；不是向量库 |

## 怎么读

不靠函数名。看 JSON 怎么变。

线上 `tool_use.input` 本身就是对象。第 1 篇对照过一次和 Codex `arguments` 字符串的差别。第 2–4 篇写成解析后的对象。

第 3 篇沙箱是 `sandbox-runtime` 的路径/网络表，不是 Windows Capability SID。

Claude Code 没有 `spawn_agent` / `wait_agent` / `compaction_trigger` / `cap_sid`。对照时只说「Codex 有、这里没有」。
