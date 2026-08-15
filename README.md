# 学习笔记

给自己复习和面试用。现在全是编码助手的源码笔记：每个产品一个根目录文件夹，四篇对齐，用 JSON 看数据怎么变。

面试前先看 [对照](./对照.md)。

## 现在有什么

| 目录 | 产品 | 发给模型的协议 | 源码基线 |
| --- | --- | --- | --- |
| [`claude/`](./claude) | Claude Code | Anthropic Messages：`system + messages + tools` | `Desktop\claude-code-source`（无 git，版本号读不到） |
| [`codex/`](./codex) | Codex CLI | OpenAI Responses：`instructions + input + tools` | `Desktop\codex`，`385c0a9351e2199929e01f7864ec78a8f7d5e580`（2026-07-11） |

两套都是只读分析，没有改过对应源码。

## 四篇对齐

| # | 面试里它回答什么 | Claude | Codex |
| --- | --- | --- | --- |
| 1 | 发给模型的到底是什么 | [请求结构](./claude/01-Claude-Code发给大模型的完整请求结构.md) | [请求结构](./codex/01-Codex发给大模型的完整请求结构.md) |
| 2 | 子 Agent 的回执是不是正文 | [主子 Agent](./claude/02-Claude-Code主子agent机制.md) | [主子 Agent](./codex/02-Codex主子agent机制.md) |
| 3 | 申请单、审批、沙箱是不是一层 | [工具与沙箱](./claude/03-Agent工具系统-FunctionCalling-MCP-权限沙箱.md) | [工具与沙箱](./codex/03-Agent工具系统-FunctionCalling-MCP-权限沙箱.md) |
| 4 | 压缩和长期记忆是不是一件事 | [记忆](./claude/04-Claude-Code记忆系统-压缩与检索.md) | [记忆](./codex/04-Codex记忆系统-压缩与检索.md) |

对齐的是问题，不是协议。Claude Code 没有 `spawn_agent` / `wait_agent` / `compaction_trigger` / Capability SID。

## 怎么读

1. 先把一边四篇读完，再按编号对照另一边。
2. 不靠函数名。看申请单、回执、历史数组、权限表怎么变。
3. 第 1 篇开篇是完整请求（教学拼接处正文里标了）。第 2–4 篇写成解析后的对象，避免满屏转义。
4. 每篇末尾有「面试怎么说」和「不要说成什么」。对照页把两边差在哪收成一张表。

## 怎么加下一个助手

根目录再开一个小写文件夹，例如 `cursor/`、`grok/`。不要预先建空目录。

1. 尽量对齐上面四个问题，文件名 `NN-主题.md`。
2. 文件夹里放 `README.md`：阅读顺序 +「面试里它回答什么」。
3. 每篇文首两行：同系列链接、对照另一边的同号文章。
4. 更新本 README 的目录表、四篇对齐表，以及 [对照](./对照.md)。
5. 文末保留「面试怎么说」和「不要说成什么」。

产品特有的机制写成该系列自己的篇，不要硬塞进这四个编号里冒充同一套。

## 以后搬进 `agent/`

根目录助手到 3 个或以上时，整组搬进去：

```text
learning-notes/
  README.md
  agent/
    对照.md
    claude/
    codex/
    …
```

- 系列内部的 `./02-….md` 不用改。
- 系列之间的 `../codex/….md` 只要一起搬就不用改。
- 只改根 README 里的路径，并把 [对照](./对照.md) 跟着助手系列走。

现在先不搬，避免只有两个文件夹时多套一层。

## 笔记写法

- 字段名、角色、id 配对按源码；教学拼接必须标明。
- 不展示真实凭据，不假装抓包。
- 源码基线写在文首：路径、commit 或「读不到版本号」。
- 有内容再建目录。不要预留 `python/`、`frontend/` 这类空壳。
