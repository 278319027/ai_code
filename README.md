# SSD 固件 AI 辅助编程

面向 SSD 固件团队的 AI 辅助编程体系，基于四工具架构：**Graphify**（知识图谱）+ **CodeGraph**（调用图）+ **OpenSpec CLI**（规格驱动）+ **Superpowers**（工程纪律），由 `OpenCode Agent` 统一编排 **KNOW → PLAN → BUILD → FEEDBACK** 闭环。

> ⚠️ 第一次来？先看 [项目导航](docs/navigation.md)（完整文件地图 + 配置索引 + 按角色找入口）

## 快速上手（5 分钟）

```bash
bash scripts/deploy_tools.sh /path/to/c-source

# 2. 验证环境
bash scripts/verify.sh    # 确认 17/17 通过

# 3. 选一条路径开始（在 OpenCode IDE 中）
#    路径 A（有设计文档）→ /opsx:propose <change-name>
#    路径 B（无设计文档）→ codegraph explore <区域>  先生成设计文档
```

> **工具安装 vs 项目分发**：`deploy_tools.sh` 安装的是有可执行文件的外部工具（Node.js、codegraph、graphify、openspec CLI）。Superpowers / openspec-workflow / sd-firmware-copilot 是项目级 Skill（`.opencode/skills/` 下的 Markdown 文件），随仓库分发，`git clone` 即可用，无需脚本安装。

## 两种使用路径

### 路径 A：设计文档驱动

已有设计文档（SAD/SDD/ICD），AI 直接理解设计并实现。

```
KNOW:     graphify query "<关键词>" && codegraph explore <区域>
PLAN:     /opsx:propose my-change "根据 SDD 第 X 章实现 Y 功能"
BUILD:    /opsx:apply my-change
FEEDBACK: /opsx:archive my-change && graphify update .
```

### 路径 B：代码驱动

无设计文档，AI 先分析代码自动生成设计文档，再按路径 A 执行。

```
KNOW:     codegraph explore <区域> && codegraph where <核心函数>
          graphify explain "<概念>"
          → AI 自动生成设计文档
PLAN → BUILD → FEEDBACK: 同路径 A
```

## 四工具架构

| 阶段 | 工具 | 部署方式 |
|------|------|---------|
| **KNOW** | Graphify + CodeGraph | `bash scripts/deploy_tools.sh` |
| **PLAN** | OpenSpec CLI v1.4.1 | `npm install -g @fission-ai/openspec` |
| **BUILD** | Superpowers + sd-firmware-copilot（测试验证） | `.opencode/skills/sd-firmware-copilot/` |
| **FEEDBACK** | OpenSpec CLI + Graphify | 同 PLAN |

## 配置（opencode.json）

`opencode.json` 声明 MCP 服务器、插件和加载的 Skill。其中 `codegraph` MCP 的 `--path` 形参使用 `opencode.json` 不支持注释（JSON 标准不支持），但可用 shell 变量表达式指定目标 SSD 固件源码根：

```json
"command": ["codegraph", "serve", "--mcp", "--path", "${FEMU_ROOT:-/home/zsf/AI_Proj/AI_SSD_SIM}"]
```

| 环境变量 | 作用 | 默认值 | 必需 |
|---------|------|-------|------|
| `FEMU_ROOT` | CodeGraph MCP 服务的目标 SSD 固件源码目录（直接指向，不拼接子路径） | `/home/zsf/AI_Proj/AI_SSD_SIM` | 否（未设则用默认） |

调整默认路径的方式：
- **临时覆盖**：`FEMU_ROOT=/path/to/your/ssd_femu_hw_femu_dir opencode`（当前 shell 启动 Agent 时生效）
- **永久设置**：`echo 'export FEMU_ROOT=/path/to/your/ssd_femu_hw_femu_dir' >> ~/.bashrc`

## 推荐阅读

| 序号 | 文档 | 内容 |
|------|------|------|
| 1 | [项目导航](docs/navigation.md) | 完整结构地图 + 按角色找文件 + 配置索引 + **AI_Copilot_UFS 与目标代码库关系** |
| 2 | [方法论概览](AGENTS.md#方法论概览) | 双路径、四阶段闭环、五级门禁、四条铁律 |
| 3 | [路线图](docs/roadmap.md) | 实施进度与规划 |
| 4 | [维护者指南](docs/maintainer.md) | 日常操作、FAQ、变更记录 |

## 核心原则

- **代码优先**：Source Code > Design Docs > Specs > Memory > Prompt
- **AI 辅助不替代人**：人负责架构决策和风险判断

> 完整原则（小任务原则、CodeGraph 检查、五级门禁、四条铁律）见 [docs/navigation.md §几条重要约定](docs/navigation.md#几条重要约定)。

AI Firmware Workflow Engine（MVP）总体设计文档 v1.0

1. 项目目标

设计一套适用于 SSD 固件及其他大型嵌入式 C/C++ 项目的 AI Workflow Engine。

设计目标不是开发一个新的大模型，而是在 OpenCode、GPT、Claude、Qwen 等 Agent 之上，构建一个可控、可验证、可恢复、可扩展的 AI 开发框架。

整个系统遵循一个核心原则：

«LLM 负责思考（Think），Workflow Engine 负责控制（Control）。»

Workflow Engine 永远拥有流程控制权。

Agent 永远不拥有流程控制权。

---

2. 设计原则

整个系统遵循以下原则：

2.1 Flow Controlled

所有流程由 Workflow Engine 控制。

Agent 不允许：

- 决定下一步
- 跳过步骤
- 修改 Workflow
- 自动结束 Workflow

Agent 只能完成当前 Step。

---

2.2 Skill Atomic

Skill 必须职责单一。

例如：

Requirement Analysis

Impact Analysis

Architecture Design

Coding

Review

每个 Skill 只完成一个目标。

禁止一个 Skill 同时完成：

需求分析 + 设计 + 编码。

---

2.3 Workflow First

Workflow 描述开发流程。

Skill 描述能力。

Knowledge 描述知识。

三者完全解耦。

---

2.4 Model Agnostic

Workflow 不关心底层模型。

可以切换：

- GPT
- Claude
- Qwen
- OpenCode
- Codex CLI

无需修改 Workflow。

---

2.5 Artifact Driven

Workflow 不依赖聊天记录。

Workflow 只依赖 Artifact。

例如：

Requirement.md

Impact.json

Design.md

Review.md

Patch.diff

Build.log

Assert.report

下一步永远读取 Artifact，而不是重新阅读整个对话。

---

3. 总体架构

                Human
                   │
          (审批 / 评论 / 决策)
                   │
                   ▼
         Workflow Engine
                   │
     ┌────────┬─────────┬────────┐
     │        │         │        │
 Scheduler  Validator  Session  State
     │
     ▼
  Skill Runner
     │
     ▼
   AI Agent
(OpenCode / GPT / Claude / Qwen)
     │
     ▼
CodeGraph / Git / Docs / Memory

Workflow Engine 是整个系统唯一的控制中心。

---

4. Workflow Engine 组成

Workflow Engine 包括：

Scheduler

负责：

- 当前 Step
- 下一 Step
- Dependency
- Retry
- Pause
- Resume

Scheduler 不执行 AI。

Scheduler 只调度。

---

State Manager

维护 Workflow 状态：

Current Step

History

Variables

Decision

Summary

Artifacts

整个 Workflow 的唯一真实状态保存在这里。

---

Validator

负责验证每个 Step 输出。

例如：

Impact Skill：

必须输出：

affected_modules

risk

dependency

Validator 检查：

字段是否存在

JSON 是否合法

是否满足业务规则

失败：

Reject

Retry

Pause

Workflow 永远不会相信 Agent 一定正确。

---

Runner

Runner 是唯一调用 Agent 的模块。

负责：

Prompt 构建

Context 注入

Tool 调用

Model Adapter

Output Parser

Workflow Engine 永远不直接调用 LLM。

---

Transition

负责：

状态迁移。

例如：

Review Failed

↓

Coding

User Reject

↓

Design

Build Failed

↓

Coding

整个 Workflow 本质上就是一个状态机。

---

Memory

保存：

Decision

Summary

Architecture

Workflow 不依赖历史聊天。

只依赖结构化 Memory。

---

5. Skill 设计

Skill 不是 Prompt。

Skill =

Prompt

+ 

Context Builder

+ 

Tool

+ 

Output Parser

+ 

Validator Contract

Skill 不拥有流程。

Skill 只完成一个目标。

---

6. Skill 分类

整个系统定义三种 Skill。

6.1 Atomic Skill

一次执行完成。

例如：

Impact Analysis

Review

Assert Analysis

FTL Analysis

特点：

同步

有输入

有输出

立即返回

---

6.2 Interactive Skill

需要人与 Agent 多轮交互。

例如：

Superpowers Brainstorm

Architecture Discussion

Requirement Clarification

Workflow 不等待一次返回。

Workflow 创建 Session。

Workflow Suspend。

用户与 Agent 自由交互。

直到：

Finish

Workflow Resume。

最终输出：

Artifact。

例如：

brainstorm.md

而不是聊天记录。

---

6.3 Background Skill

后台运行。

例如：

Build

Test

Index

Generate CodeGraph

这些任务可能持续数分钟。

Workflow 等待事件。

---

7. Session Manager

这是 Interactive Skill 的核心。

职责：

创建 Session

保存 Session

恢复 Session

结束 Session

接收 Artifact

发送 Workflow Event

Session 生命周期：

Created

↓

Running

↓

Paused

↓

Completed

↓

Archived

Workflow 不参与 Session 内部聊天。

Workflow 只等待 Session Event。

---

8. Workflow Context

Workflow Context 是整个 Workflow 唯一共享数据。

包含：

Workflow ID

Current Step

Variables

Summary

Artifacts

Memory

History

所有 Skill：

只能读取 Context。

不能访问其它 Skill Prompt。

---

9. Artifact

每一步都会生成 Artifact。

例如：

Requirement.md

Impact.json

Design.md

Review.md

Patch.diff

Build.log

Assert.report

Artifact 是 Workflow 唯一输入输出。

---

10. Execution Contract（执行契约）

每个 Skill 必须声明：

Allow

Forbid

Required Outputs

Exit Criteria

Failure Events

例如：

Impact Skill：

Allow：

读取 Requirement

读取 CodeGraph

读取 Docs

Forbid：

修改代码

Review

Commit

Required：

affected_modules

risk

dependency

Exit：

Validator Pass

Failure：

missing_context

invalid_output

tool_error

Workflow 根据 Contract 控制 Skill。

不是依赖 Agent 自觉遵守。

---

11. Human Checkpoint

Workflow 定义：

Human Step。

例如：

Architecture Approval

Review Approval

Commit Approval

Workflow：

Pause。

等待用户：

Approve

Reject

Comment

Comment 自动进入下一轮 Prompt。

Agent 永远不知道 Workflow 状态。

---

12. Workflow 状态机

Workflow 不采用固定流程。

采用状态机。

例如：

Requirement

↓

Impact

↓

Design

↓

Human Approval

↓

Coding

↓

Build

↓

Review

↓

Release

任何节点：

都可以：

Retry

Rollback

Pause

Resume

Goto

Workflow 支持事件驱动。

---

13. SSD 固件专用 Skill

Workflow Engine 不包含 SSD 知识。

SSD 能力全部来自 Skill。

例如：

Assert Dump Analysis

FTL Analysis

NAND Analysis

Wear Leveling Review

GC Review

DMA Review

Interrupt Review

CodeGraph Search

Architecture Review

Compile Verification

Static Analysis

未来也可以扩展到：

Linux Driver

RTOS

Bootloader

MCU

无需修改 Workflow Engine。

---

14. 推荐目录结构

workflow_engine/
│
├── engine/
│   ├── scheduler.py
│   ├── runner.py
│   ├── validator.py
│   ├── transition.py
│   └── state_machine.py
│
├── runtime/
│   ├── workflow_context.py
│   ├── artifact_store.py
│   ├── memory.py
│   └── session_manager.py
│
├── skills/
│
├── validators/
│
├── workflows/
│
├── tools/
│
├── adapters/
│
├── models/
│
└── config/

---

15. MVP 实施路线

Phase 1

实现：

State Manager

Scheduler

Workflow YAML

Runner

Validator

完成最小可运行 Workflow。

---

Phase 2

增加：

Artifact

Memory

Transition

Retry

Rollback

Human Checkpoint

形成完整 Workflow。

---

Phase 3

增加：

Session Manager

Interactive Skill

Background Skill

事件驱动 Workflow。

---

Phase 4

集成：

OpenCode

CodeGraph

Git

Build

Test

Static Analysis

形成完整 AI Firmware Workflow Platform。

---

16. 最终目标

最终形成一个领域无关（Domain-agnostic）、**模型无关（Model-agnostic）**的 AI Workflow Engine：

- Workflow Engine：负责状态、调度、恢复、审批、验证。
- Skill：负责具体能力，可插拔、可复用。
- Session Manager：负责人与 AI 的长生命周期协作。
- Knowledge：提供 CodeGraph、文档、Memory、Git 等上下文。
- AI Agent：专注于推理与执行，不拥有流程控制权。

整个系统最终实现：

«Workflow 控制 AI，而不是 AI 控制 Workflow。»

这是整个设计最核心的理念，也是该框架区别于当前多数 AI Agent 的关键所在。
