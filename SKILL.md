---
name: asds-mode
description: ASDS 全自动开发工作流模式。适用于需要 7×24 小时无人值守软件开发、持续迭代、长周期项目的场景。当用户提到「全自动开发」「ASDS」「autonomous devops」「无人值守开发」时激活。
---

# ASDS Mode - 全自动软件开发工作流

ASDS（Autonomous Software Development System）是一套 7×24 小时无人值守的全自动软件开发系统。

## 核心架构

```
用户（提需求）
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│                      Orchestrator（编排者）                   │
│  接收需求 → 调度 PM/Developer/QA → 监控状态 → 发通知        │
└───────────────────────┬─────────────────────────────────────┘
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
    ┌──────────┐  ┌──────────┐  ┌──────────┐
    │PM Agent  │  │Developer │  │  QA Agent │
    │ 需求拆解  │  │ 代码编写  │  │  测试验证  │
    └──────────┘  └──────────┘  └──────────┘
                      │
                      ▼
             tasks.json（唯一事实源）
             PROGRESS.md（进度日志）
```

## 三角色

| 角色 | 职责 | 模型 |
|------|------|------|
| orchestrator | 接收需求、调度任务、处理异常 | MiniMax-M2.7 |
| developer/coder | 写代码、跑验证命令、更新状态 | GPT-5.4 |
| qa | 验证输出是否符合标准（不改代码） | MiniMax-M2.7 |

## 任务状态流转

```
PENDING → IN_PROGRESS → DONE（通过）
                       → FAILED（重试1次，还失败则通知用户）
                       → BLOCKED（连续3次失败，人工介入）
```

## 强制规则

### Developer 强制规则
- 禁止纯文本回复结束回合，必须调用工具
- 每次状态变更必须落盘（更新 tasks.json）
- 构建/测试必须通过才能标记 DONE
- 单次任务代码修改不超过 200 行（超出必须拆分）
- **单文件规则**：一次只能改一个文件
- **PROGRESS.md 强制写入**：每个任务完成后必须更新

### 回压门（Backpressure Gates）
每个任务执行后必须依次验证：
```
Lint → Typecheck → Test → Build
```
全部通过才能标记 DONE，任一失败则停在原地修复。

## 启动流程

### 1. 确认 Agent 已就绪
```bash
openclaw agents list
# 确认 orchestrator / coder / qa 存在
```

### 2. 创建项目结构
每个项目必须有：
```
项目目录/
├── tasks.json        # 任务队列（唯一事实源）
├── PROGRESS.md       # 进度日志（tail 即可巡检）
├── AGENTS.md         # 项目验证命令定义
└── DEMAND.md         # 需求文档
```

### 3. 拆解需求
手动或让 orchestrator 把需求拆解为 tasks.json：
```json
{
  "id": "PROJ-001",
  "title": "任务标题",
  "acceptance_criteria": ["标准1", "标准2"],
  "status": "PENDING"
}
```

### 4. 启动 Developer
```bash
sessions_spawn(
  agentId="coder",
  model="tokenx24/gpt-5.4",
  task="执行 tasks.json 中 PENDING 任务..."
)
```

### 5. 监控进度
```bash
tail -f 项目目录/PROGRESS.md
cat 项目目录/tasks.json | jq '.tasks[].status'
```

## ASDS 部署检查清单

- [ ] `openclaw agents list` 确认 orchestrator/coder/qa 存在
- [ ] 看门狗运行中：`ps aux | grep watchdog`
- [ ] Cron 任务已注册：`openclaw cron list`
- [ ] 项目目录包含 tasks.json / PROGRESS.md / AGENTS.md
- [ ] DEMAND.md 包含清晰需求描述

## 相关文件

- 完整设计文档：`state/autonomous-devops.md`
- 操作手册：`state/autonomous-devops-manual.md`
