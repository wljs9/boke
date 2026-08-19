---
title: "Hermes Agent v0.16 Kanban Swarm: 多智能体协作的深度解析"
date: "2026-08-19"
author: "施志亮"
tags: [hermes-agent, kanban, swarm, multi-agent, automation, AI]
description: "深度解析 Hermes Agent v0.16 的 Kanban Swarm 功能——从架构设计、并行工作流到结构化交接、熔断恢复和实际部署场景。"
---

# Hermes Agent v0.16 Kanban Swarm: 多智能体协作的深度解析

## 引言

在 AI Agent 领域，单智能体完成复杂任务的能力已经有了长足进步。但当任务需要**多个专业角色协同**——研究员调研、开发者实现、审查者把关、发布者部署——传统单智能体串联模式的瓶颈就暴露了：上下文膨胀、角色混淆、失败重试全栈阻塞。

Hermes Agent v0.16 的 **Kanban Swarm** 正是为解决这个问题而生。它不是另一个任务队列，而是一个**多智能体协作协议**——让不同 Profile 的 AI 工作者像一支高效的工程团队一样并行工作，通过结构化交接传递上下文，并自动处理失败恢复。

本文将从架构原理、核心机制、实战场景、运维监控四个维度，为 AI 开发者系统拆解 Kanban Swarm。

---

## 一、架构概览：从任务板到智能体Swarm

### 1.1 核心组件

Kanban Swarm 的架构围绕一个**共享的 SQLite 任务板**构建：

```
┌─────────────────────────────────────────┐
│            Hermes Gateway               │
│  ┌─────────────┐  ┌──────────────────┐  │
│  │  Dispatcher  │  │  Notifier        │  │
│  │  (调度引擎)   │  │  (事件通知)       │  │
│  └──────┬──────┘  └──────────────────┘  │
└─────────┼────────────────────────────────┘
          │ 调度 & 监控
          ▼
┌─────────────────────────────────────────┐
│         Kanban Board (SQLite)           │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐   │
│  │Triage│ │Todo  │ │Ready │ │Running│   │
│  ├──────┤ ├──────┤ ├──────┤ ├──────┤   │
│  │Block │ │Review│ │Done  │ │Archv │   │
│  └──────┘ └──────┘ └──────┘ └──────┘   │
└──────────────┬──────────────────────────┘
               │ Worker 接入
     ┌─────────┼─────────┐
     ▼         ▼         ▼
┌────────┐ ┌────────┐ ┌────────┐
│Worker A│ │Worker B│ │Worker C│
│研究员   │ │开发者   │ │审查者   │
└────────┘ └────────┘ └────────┘
```

每个 Worker 是一个独立的 Hermes Agent 进程，拥有自己的 Profile（环境变量、模型配置、技能集），通过 `kanban_*` 工具集与共享板交互。

### 1.2 Swarm 模式

`hermes kanban swarm` 命令是 v0.16 的核心创新。它创建一个有向无环图（DAG）的工作流：

```
hermes kanban swarm "撰写的技术博客并发布到 GitHub" \
  --worker researcher:调研 \
  --verifier writer \
  --synthesizer reviewer \
  --created-by orchestrator
```

这会产生一个 4 阶段管线：

```
        orchestrator
             │
     ┌───────┴───────┐
     │  researcher   │  ← 并行工作
     └───────┬───────┘
             ▼
     ┌───────┴───────┐
     │    writer     │  ← Verifier（验证输出）
     └───────┬───────┘
             ▼
     ┌───────┴───────┐
     │   reviewer    │  ← Synthesizer（综合交付）
     └───────┬───────┘
             ▼
     ┌───────┴───────┐
     │   publisher   │  ← 发布（非 Swarm 原生，可手动追加）
     └───────────────┘
```

Swarm 的设计哲学是：**Orchestrator 拆解目标 → 多个 Researcher 并行调研 → Verifier 把关质量 → Synthesizer 整合输出**。每一阶段的 Worker 只看到自己需要的上下文，不会因为全栈信息而过载。

---

## 二、核心机制深度解析

### 2.1 依赖引擎与自动推进

Kanban 的依赖引擎是整个系统的"隐形调度员"。当创建一个任务并指定 `--parent` 时：

```bash
SCHEMA=$(hermes kanban create "设计数据库Schema" --assignee backend-dev --json | jq -r .id)
API=$(hermes kanban create "实现API端点" --assignee backend-dev --parent $SCHEMA --json | jq -r .id)
TESTS=$(hermes kanban create "编写集成测试" --assignee qa-dev --parent $API)
```

**只有 SCHEMA 进入 `ready` 状态**。API 和 TESTS 停留在 `todo`，直到它们的父任务完成。Dispatcher 每 60 秒（可配置）执行一次调度 tick：

1. 检查所有 `ready` 任务，按优先级排序
2. 原子性地 claim 一个任务（写入 lock + TTL）
3. 为 assignee profile 创建一个 worker 进程
4. Worker 通过 `kanban_complete()` 完成任务并传递 handoff
5. 依赖引擎自动将所有子任务从 `todo` 提升到 `ready`

### 2.2 结构化交接（Structured Handoff）

这是 Kanban Swarm 区别于传统任务队列的关键设计。Worker 完成任务时不只是标记"done"，而是携带结构化数据：

```python
# Worker 内部的工具调用
kanban_complete(
    summary="实现了令牌桶限流器，按 user_id 键控，IP 回退；所有测试通过",
    metadata={
        "changed_files": ["limiter.py", "tests/test_limiter.py"],
        "tests_run": 14,
        "decisions": [
            "令牌桶速率: 100 req/min per user",
            "回退策略: IP-based 当 user_id 不可用时"
        ]
    },
    result="限流器已就绪"
)
```

下游 Worker 在 `kanban_show()` 时会自动看到父任务的 `summary` 和 `metadata`——**不需要翻阅代码评论或聊天记录**。这种设计让上下文传递变成结构化的、机器可读的契约。

### 2.3 熔断器与崩溃恢复

现实世界的 Worker 会失败——凭证过期、OOM 杀进程、网络超时。Dispatcher 有两层防御：

**熔断器（Circuit Breaker）:**

| 失败类型 | 行为 |
|---------|------|
| `spawn_failed`（启动失败） | 重试，计数递增 |
| `timed_out`（超时） | 重试，计数递增 |
| `crashed`（进程崩溃） | 重试，计数递增 |
| `protocol_violation`（协议违规） | 连续 3 次后自动阻塞 |
| `gave_up`（终止） | 任务进入 `blocked`，等待人工干预 |

默认连续 2 次失败即触发熔断（可通过 `--max-retries` 或 `kanban.failure_limit` 配置）。

**崩溃恢复：**

当 Worker PID 突然消失（SIGKILL、OOM）但 TTL 未到期，Dispatcher 通过 `kill(pid, 0)` 检测到进程死亡，自动释放 claim 并将任务重新放入 `ready`。重试的 Worker 能看到前一次尝试的完整记录：

```
hermes kanban runs t_abcd
# #  OUTCOME       PROFILE     ELAPSED  STARTED
# 1  crashed       backend-dev    2m     2026-04-27 14:02
#       → OOM at row 2.3M (process 99999 gone)
# 2  completed     backend-dev    8m     2026-04-27 15:18
#       → chunked with LIMIT + WHERE id > last_id
```

重试 Worker 看到 "OOM at row 2.3M"，自动选择了分块策略，第二次就成功了。

### 2.4 Review 生命周期

v0.16 引入了原生的 Review 生命周期，支持同一 Card 上的 实现↔审查 循环：

```
Worker A: kanban_request_review(summary="实现完成", reviewer="reviewer")
    → Card 进入 review 状态，实现 run 关闭为 outcome='review_requested'

Worker B (reviewer): kanban_show() → kanban_request_changes(reason="需要增加密码强度验证")
    → Review run 关闭为 outcome='changes_requested'，Card 回到 ready

Worker A: kanban_show()（看到审查反馈）→ 修改 → kanban_request_review(...)
Worker B: kanban_show() → kanban_complete(summary="审查通过")
    → Card 进入 done
```

每个迭代都记录在 `task_runs` 中，形成完整的审计轨迹。

---

## 三、实战场景

### 3.1 独立开发者：特性管线

一个典型的功能开发流程：设计 Schema → 实现 API → 编写测试。依赖引擎确保三者顺序执行，结构化交接让实现者知道 Schema 的决策，测试编写者知道 API 的签名。

### 3.2 内容团队：并行收割

10 个翻译任务 + 5 个转录任务 + 4 个产品描述任务——分配给不同的 Profile（translator / transcriber / copywriter）。Dispatcher 自动为每个 Profile 拉起 Worker 进程并行处理：

```
for lang in Spanish French German; do
    hermes kanban create "翻译首页为$lang" --assignee translator
done
for i in 1 2 3 4 5; do
    hermes kanban create "转录Q3客户通话#$i" --assignee transcriber
done
```

三个 Profile 的 Worker 同时在 Running 列中运行，互不干扰。Dashboard 按 Profile 分道显示，一眼看到每个 Worker 在做什么。

### 3.3 审查流水线

PM 写规格 → 工程师实现 → 审查者拒绝 → 工程师修改 → 审查者批准。完整的审计轨迹记录了谁在什么时候做了什么决定，每次审查迭代的反馈都结构化成 `task_runs` 数据。

### 3.4 CI 修复修复链

某个已完成的实现任务 2 小时后 CI 失败了。不需要 reopen 已完成的任务——而是创建一个以原任务为父的新修复任务：

```bash
hermes kanban create "修复 CI: test_backoff_jitter 在 3.11 上不稳定" \
  --assignee backend-dev --parent t_impl \
  --body "CI run #4812 失败... acceptance: tests green on 3.11/3.12"
```

新 Worker 的 `worker_context` 自动包含父任务的 summary + metadata——它知道改了哪些文件、做了什么决策。这是"已完成任务是历史，不是围栏"的设计哲学。

---

## 四、运维与监控

### 4.1 三重视角同一块板

| 视角 | 工具 | 适用场景 |
|------|------|---------|
| CLI | `hermes kanban ls/show/runs/tail` | 终端工作流、脚本集成 |
| Dashboard | Web UI (127.0.0.1:9119) | 可视化监控、拖拽操作 |
| 消息平台 | Telegram/Discord/Slack | 远程通知、移动端查看 |

三者都通过同一个 SQLite DB 操作，数据始终一致。

### 4.2 桌面通知 + 网关推送

- **Desktop 通知**：Kanban 插件在 Hermes Desktop 中显示原生 toast，Worker 完成/阻塞/崩溃时实时推送
- **Gateway 通知**：通过 Telegram/Discord/Slack 推送，支持 `notify`（仅消息）、`notify+wake`（消息+唤醒 Agent）、`wake`（仅唤醒）三种模式

### 4.3 多租户隔离

```bash
hermes kanban create "月报" --assignee researcher --tenant business-a
```

Worker 收到 `$HERMES_TENANT` 环境变量，在内存读写时自动命名空间隔离。同一块板、同一个 Dispatcher，但数据完全隔离。

---

## 五、技术架构亮点

### 5.1 多 Gateway 架构

大型部署中，Dispatch 和 Notification 可以分离：

- **一个 Dispatcher Gateway**：持有 `kanban.dispatch_in_gateway: true`，负责调度
- **多个 Profile Gateway**：不调度，但负责自己 Profile 的消息推送

一个由 `writer` profile 创建的 Task，即使由 `default` profile dispatcher 调度，它的完成通知也会由 `writer` 自己的 Gateway 投递到对应的聊天平台。不会出现"张三调度，李四通知"的混乱。

### 5.2 运行历史（Runs）

`task_runs` 表是 Kanban Swarm 的审计核心。每次尝试（成功或失败）都是一行：

- `outcome`: completed / blocked / crashed / timed_out / gave_up / review_requested / changes_requested
- `summary`: 人工可读的交接摘要
- `metadata`: 机器可读的 JSON 数据（changed_files, decisions, tests_run 等）
- `error`: 失败时的错误信息

这让"这次为什么失败"从猜测变成了可查询的事实。

### 5.3 协议违规保护（Protocol Violation Guard）

Worker 进程成功退出但未调用 `kanban_complete()` 或 `kanban_block()`——这通常意味着 Agent 回答了问题但忘了关闭任务。Dispatcher 检测到后自动重试，连续 3 次违规则触发熔断。这防止了"静默丢失"任务。

---

## 六、与其他工具的对比

| 维度 | Hermes Kanban Swarm | 传统消息队列 (RabbitMQ/Kafka) | 传统看板 (Jira/Linear) | LangGraph |
|------|---------------------|-------------------------------|----------------------|-----------|
| **工作单元** | AI Worker + 结构化交接 | 纯消息 | 人工卡片 | DAG 节点 |
| **失败处理** | 熔断器 + 自动重试 + 审计 | 死信队列 | 人工标记 | 异常边处理 |
| **上下文传递** | summary + metadata 结构化 | 消息体 (无标准) | 评论 (非结构化) | 状态传递 |
| **并行度** | 多 Profile 并行 Worker | 消费者组 | 多成员并行 | 分支并行 |
| **监控** | CLI + Dashboard + Gateway 通知 | 管理 UI | Web UI | 可视化 |

Kanban Swarm 的独特价值在于它是 **为 AI Worker 设计的协作协议**——关注的是 Agent 间的上下文传递、失败恢复、审计轨迹，而不是简单的消息投递。

---

## 七、快速上手

```bash
# 1. 初始化看板
hermes kanban init

# 2. 启动 Gateway (内嵌 Dispatcher)
hermes gateway start

# 3. 创建 Swarm
hermes kanban swarm "Hermes v0.16 功能深度分析博客" \
  --worker researcher:调研 \
  --verifier writer \
  --synthesizer reviewer

# 4. 监控进度
hermes kanban watch

# 5. 创建发布任务（手动追加非 Swarm 原生步骤）
hermes kanban create "发布到 GitHub" \
  --assignee publisher --parent <synthesizer_task_id>
```

---

## 结语

Hermes Agent v0.16 的 Kanban Swarm 不仅仅是一个"功能更新"。它代表了一种范式转变：**从单个 AI Agent 执行孤立任务，到多 Agent 像专业团队一样协作**。

结构化交接消除了"上下文丢失"这个多 Agent 系统最大的痛点；熔断器和崩溃恢复让系统在真实生产环境中可靠运行；而完整的审计轨迹让每一次失败都成为可学习的经验。

对于构建复杂 AI 工作流的开发者来说，Kanban Swarm 不是锦上添花——它是将实验性 Agent 转化为可维护的生产系统的关键基础设施。

---

*本文由 Hermes Agent Kanban Swarm 流水线自动编排发布。*