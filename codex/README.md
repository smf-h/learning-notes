# Codex（四篇）

> 源码基线：`C:\Users\yejunbo\Desktop\codex`，提交 `385c0a9351e2199929e01f7864ec78a8f7d5e580`（2026-07-11）。只读分析，没有改仓库源码。
>
> 对照 Claude Code：[系列索引](../claude) · [对照表](../对照.md)

读完应能讲清：Responses 请求怎么组、子 Agent 怎样独立跑、工具从申请单走到沙箱执行、上下文压缩和跨任务记忆为什么是两条管线。

## 阅读顺序

| 顺序 | 文件 | 面试里它回答什么 |
| --- | --- | --- |
| 1 | [01 请求结构](./01-Codex发给大模型的完整请求结构.md) | 发给模型的是 `instructions + input + tools` |
| 2 | [02 主子 Agent](./02-Codex主子agent机制.md) | 子 Agent 不是函数；`wait_agent` 不带正文 |
| 3 | [03 工具与沙箱](./03-Agent工具系统-FunctionCalling-MCP-权限沙箱.md) | 暴露、审批、沙箱是三层 |
| 4 | [04 记忆](./04-Codex记忆系统-压缩与检索.md) | 压缩保当前会话，长期记忆跨会话；不是向量库 |

## 怎么读

不靠函数名。看 JSON 怎么变。

第 1 篇开篇是线上原样，`arguments` 会带 `\"`。第 2–4 篇写成解析后的对象。

第 3 篇 Windows 沙箱把 Token、`cap_sid`、各路径 ACL、一次 `Access is denied` 的判定过程写成了 JSON。建议和第 1 篇的完整请求对着看。
