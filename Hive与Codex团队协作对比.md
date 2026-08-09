# Hive 与 Codex 团队协作对比

> 本文根据 [tt-a1i/hive](https://github.com/tt-a1i/hive) 的公开 README 和已知设计，对比 Hive 与 Codex 主代理/子代理机制在团队协作方面的差异。重点只讨论五个方面，不把“产品工作流更好”误解成“模型能力更强”。

## 目录

- [1. 先说结论](#1-先说结论)
- [2. 真实 CLI 团队](#2-真实-cli-团队)
  - [2.1 Hive 的运行方式](#21-hive-的运行方式)
  - [2.2 与 Codex Thread 的区别](#22-与-codex-thread-的区别)
- [3. 显式任务图和状态](#3-显式任务图和状态)
  - [3.1 `.hive/tasks.md` 的作用](#31-hivetasksmd-的作用)
  - [3.2 与 Codex 会话历史的区别](#32-与-codex-会话历史的区别)
- [4. 简单明确的团队通信](#4-简单明确的团队通信)
  - [4.1 `team send` 和 `team report`](#41-team-send-和-team-report)
  - [4.2 汇报机制的优点和边界](#42-汇报机制的优点和边界)
- [5. 多 CLI 和角色协作](#5-多-cli-和角色协作)
  - [5.1 混用不同 Agent](#51-混用不同-agent)
  - [5.2 角色分工](#52-角色分工)
- [6. Auto-staff 和 Workflows](#6-auto-staff-和-workflows)
  - [6.1 自动组建团队](#61-自动组建团队)
  - [6.2 多阶段流水线](#62-多阶段流水线)
- [7. Hive 并非全面优于 Codex](#7-hive-并非全面优于-codex)
- [8. 最终选择](#8-最终选择)

## 1. 先说结论

Hive 的优势主要在“团队协作产品形态”，Codex 的优势主要在“统一的多代理运行时”。

```mermaid
flowchart LR
    U[用户] --> H[Hive 工作台]
    H --> O[Orchestrator]
    O --> A[多个真实 CLI Worker]
    A --> T[任务图、终端、报告]
    T --> O
    O --> U
```

可以先用两句话理解：

```text
Hive：把多个真实 CLI Agent 组织成一个看得见的本地开发团队。
Codex：在统一运行时中创建和管理多个 Thread 子代理。
```

Hive 相对 Codex 主要好在：

1. 真实 CLI 进程和终端更容易观察。
2. `.hive/tasks.md` 让项目任务状态显式持久化。
3. `team send` / `team report` 让通信协议更容易理解。
4. 可以混用 Codex、Claude、Gemini、OpenCode 等 CLI。
5. Auto-staff 和 Workflows 更适合固定团队流程。

## 2. 真实 CLI 团队

```mermaid
flowchart TD
    H[Hive Runtime] --> O[Orchestrator PTY]
    H --> A[Codex Worker PTY]
    H --> B[Claude Worker PTY]
    H --> C[Gemini Worker PTY]
    O --> T[team 命令]
    A --> T
    B --> T
    C --> T
```

Hive README 将 Orchestrator 和 Worker 描述为真实 CLI 进程，并通过 PTY（伪终端）运行。Hive 本身主要负责启动、连接、展示和协调这些进程，而不是替换它们。

### 2.1 Hive 的运行方式

Hive 会在浏览器工作台中展示：

- Orchestrator 终端
- 每个 Worker 的终端
- Worker 的工作状态
- 任务和报告
- 重新启动或恢复入口

```mermaid
sequenceDiagram
    participant U as 用户
    participant H as Hive
    participant O as Orchestrator CLI
    participant W as Worker CLI
    U->>H: 创建 workspace
    H->>O: 启动真实 PTY
    H->>W: 启动真实 PTY
    O->>W: 通过 team 派发任务
    W-->>H: 终端输出和报告
    H-->>U: 展示团队状态
```

这种方式的关键优势是“过程可见”。用户可以直接看到 Worker 是否执行了命令、是否卡住、是否退出，而不只看到主代理整理后的最终结论。

### 2.2 与 Codex Thread 的区别

Codex 子代理主要以独立 Thread 存在，由 `AgentControl`、Thread Manager、代理注册表和代理间通信管理。

```mermaid
flowchart LR
    C[Codex Root Thread] --> A[Child Thread A]
    C --> B[Child Thread B]
    A --> M[AgentControl/通信]
    B --> M
    M --> C
```

Hive 的 Worker 是外部 CLI 进程；Codex 的子代理是 Codex 运行时内的 Thread。前者更适合观察真实终端和混用工具，后者更适合统一管理模型配置、上下文 fork、权限和生命周期。

## 3. 显式任务图和状态

```mermaid
flowchart TD
    R[项目需求] --> G[.hive/tasks.md]
    G --> A[实现任务]
    G --> B[审查任务]
    G --> C[测试任务]
    A --> G
    B --> G
    C --> G
```

### 3.1 `.hive/tasks.md` 的作用

Hive 会在工作区中维护：

```text
<workspace>/.hive/tasks.md
```

它把任务状态从聊天上下文中拿出来，成为项目目录中的共享 Markdown 文件。人和代理都可以查看任务列表、完成状态和任务依赖。

```mermaid
flowchart LR
    O[Orchestrator] --> F[.hive/tasks.md]
    W[Worker] --> F
    H[人类开发者] --> F
    F --> S[统一任务状态]
```

这对长期项目很重要，因为项目约束和任务进度不会只存在某个代理的临时上下文里。重启 Hive、换一个 Worker 或交接任务时，仍然可以从文件恢复整体进度。

### 3.2 与 Codex 会话历史的区别

Codex 也会持久化 Thread、rollout 和父子代理关系，但重点是保存代理会话和执行历史。

```mermaid
flowchart LR
    C[Codex] --> H[Thread/rollout 历史]
    H --> R[恢复代理上下文]
    V[Hive] --> T[.hive/tasks.md]
    T --> P[恢复团队任务状态]
```

二者的关注点不同：

| 机制 | 更像什么 | 主要回答的问题 |
| --- | --- | --- |
| Codex Thread/rollout | 代理会话记录 | 这个代理之前做了什么？ |
| Hive `.hive/tasks.md` | 团队任务看板 | 项目整体还有什么没做？ |

因此 Hive 在人机共同管理任务、任务交接和项目级进度方面更直观；Codex 在恢复模型上下文和代理运行状态方面更原生。

## 4. 简单明确的团队通信

```mermaid
flowchart LR
    O[Orchestrator] -->|team send| W[Worker]
    W -->|team report| H[Hive Runtime]
    H --> S[更新状态]
    S --> O
```

### 4.1 `team send` 和 `team report`

Hive 注入了内部 `team` 命令，典型协作流程是：

```text
Orchestrator：team send worker-a "实现设置搜索"
Worker：执行任务
Worker：team report
Hive：保存报告并更新 Worker 状态
```

对应关系可以简化为：

| Hive 命令 | 作用 | Codex 中的近似能力 |
| --- | --- | --- |
| `team send` | 派发任务 | `send_message` 或 `followup_task` |
| `team report` | 显式报告完成和结果 | 代理间通信 |
| `team list` | 查看团队成员和状态 | `list_agents` |
| `team spawn` | 创建 Worker | `spawn_agent` |

```mermaid
sequenceDiagram
    participant O as Orchestrator
    participant W as Worker
    participant H as Hive
    O->>W: team send
    W->>W: 执行任务
    W->>H: team report
    H->>H: 记录报告和状态
    H-->>O: 可继续汇总或追问
```

### 4.2 汇报机制的优点和边界

Hive 的优点是把“完成”变成了显式动作。README 说明，Worker 调用 `team report` 后才会回到 `idle`，Hive 不只是根据进程有没有输出来猜测任务是否结束。

```mermaid
flowchart TD
    W[Worker 执行中] --> R{是否 team report}
    R -->|否| B[保持 working 或等待处理]
    R -->|是| I[记录报告]
    I --> D[变为 idle/可继续派活]
```

但 `team report` 仍然是协作协议，不是事实证明。Worker 可能报告得不完整，也可能错误地声称任务完成。因此 Orchestrator 仍然需要检查：

- 修改了哪些文件
- 是否运行测试
- 测试是否真的通过
- 报告是否有具体证据
- 是否违反了任务范围

## 5. 多 CLI 和角色协作

```mermaid
flowchart TD
    O[Orchestrator] --> C[Codex Worker]
    O --> A[Claude Worker]
    O --> G[Gemini Worker]
    O --> Q[OpenCode/Qwen Worker]
    C --> R[实现/报告]
    A --> R
    G --> R
    Q --> R
```

### 5.1 混用不同 Agent

Hive 不绑定单一 Agent 模型。一个团队可以安排：

```text
Orchestrator：Codex
实现 Worker：Claude Code
审查 Worker：Gemini
测试 Worker：OpenCode
```

这样可以让不同 Agent 交叉验证同一个任务。

```mermaid
flowchart LR
    A[模型 A 实现] --> R[统一报告]
    B[模型 B 审查] --> R
    C[模型 C 测试] --> R
    R --> O[Orchestrator 比较和汇总]
```

这比单一模型树的优势在于模型多样性，而不是某个单独模型一定更强。不同 Agent 可能拥有不同的上下文能力、工具体验和错误倾向，交叉结果有助于发现遗漏。

### 5.2 角色分工

Hive 的角色更接近真实团队岗位：

```mermaid
flowchart LR
    T[同一项目任务] --> C[Coder 实现]
    T --> R[Reviewer 审查]
    T --> Q[Tester 测试]
    C --> O[Orchestrator 汇总]
    R --> O
    Q --> O
```

常见分工是：

| 角色 | 关注点 | 产出 |
| --- | --- | --- |
| Coder | 修改代码和实现功能 | 修改文件、实现说明 |
| Reviewer | 查找缺陷和边界问题 | 审查意见、风险清单 |
| Tester | 运行测试和补充场景 | 测试结果、回归风险 |
| Researcher | 收集事实和背景 | 证据、调研报告 |

Codex 也有 `agent_type` 和角色配置，但 Hive 的角色在 UI 团队面板中更直接地体现为“岗位”。两者都需要注意：角色提示词是软约束，不等于权限隔离。

## 6. Auto-staff 和 Workflows

```mermaid
flowchart TD
    U[复杂任务] --> O[Orchestrator]
    O --> S[Auto-staff]
    S --> C[临时 Coder]
    S --> R[临时 Reviewer]
    S --> T[临时 Tester]
    C --> W[Workflow/任务结果]
    R --> W
    T --> W
```

### 6.1 自动组建团队

Hive 的 Auto-staff 允许 Orchestrator 根据任务临时创建一组 Worker，例如：

```text
实现功能 -> 创建 coder
检查风险 -> 创建 reviewer
运行测试 -> 创建 tester
```

```mermaid
flowchart LR
    R[任务需求] --> O[Orchestrator 判断岗位]
    O --> C[创建 coder]
    O --> V[创建 reviewer]
    O --> T[创建 tester]
    C --> X[完成后清理临时 Worker]
    V --> X
    T --> X
```

相比手动调用多个 `spawn_agent`，Hive 把“团队人员配置”产品化了。优势是用户不必手动管理每个 Worker，但代价是临时进程、并发和成本可能增加。

### 6.2 多阶段流水线

Hive 的 Workflows 更适合把协作固定成阶段：

```mermaid
flowchart LR
    A[实现] --> B[审查]
    B --> C[测试]
    C --> D[汇总]
    B -->|发现问题| A
    C -->|测试失败| A
```

Codex 的主代理也可以动态完成同样的流程，但通常依赖模型在运行中临时决策。Hive Workflows 则更像一个可观察的开发流水线，适合记录每个阶段的日志、结果、停止状态和失败原因。

## 7. Hive 并非全面优于 Codex

```mermaid
flowchart LR
    H[Hive 协作优势] --> V[可见性、任务图、多 CLI]
    C[Codex 运行时优势] --> S[统一上下文、权限、线程管理]
    V --> D{根据场景选择}
    S --> D
```

Hive 的主要代价是：

- 需要管理多个真实进程和 PTY。
- 不自带 Agent 沙箱，也不提供完整的多用户权限边界。
- Worker 通常继承运行 Hive 的 shell 权限。
- 不同 CLI 的登录、恢复、参数和平台行为可能不一致。
- Worker 如果不调用 `team report`，任务状态可能不能正确收敛。

Codex 的优势则是统一性更强：模型、Thread、上下文 fork、AgentControl、权限、审批和沙箱都在同一套运行时里管理。

因此 Hive 的优势主要是工作流和可视化，不是安全性或底层代理控制能力。

## 8. 最终选择

```mermaid
flowchart TD
    Q{你最看重什么？} -->|可视化、多 CLI、任务看板| H[选择 Hive]
    Q -->|统一运行时、上下文、权限和沙箱| C[选择 Codex]
    Q -->|两者都需要| X[Hive 协调多个 CLI，其中包含 Codex]
```

选择 Hive，如果你更看重：

- 看见每个 Agent 的真实终端。
- 在同一项目中混用多个 CLI Agent。
- 用 `.hive/tasks.md` 管理长期任务。
- 通过 `team report` 形成明确的工作交接。
- 使用 Auto-staff 和 Workflows 组织固定流程。

选择 Codex，如果你更看重：

- 统一的 AgentControl 和 Thread 生命周期。
- 原生的上下文继承和历史 fork。
- 更统一的模型、权限、审批和沙箱配置。
- 少管理一层外部 CLI 进程。

最准确的结论是：

> Hive 更像“本地多代理团队协作平台”；Codex 更像“带原生子代理能力的智能编码运行时”。Hive 解决的是多个 Agent 如何像团队一样工作，Codex 解决的是多代理如何在一个受控运行时中执行。
