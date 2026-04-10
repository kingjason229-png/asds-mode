---
name: asds-mode
description: ASDS 全自动开发工作流。当用户提到「全自动开发」「ASDS」「autonomous devops」「无人值守」「持续集成」时激活。
---

# ASDS Mode - 全自动软件开发工作流

ASDS（Autonomous Software Development System）是一套 7×24 小时无人值守的全自动软件开发系统。人类只做两件事：提出需求、验收结果。

## 核心原则（不可违反）

1. **tasks.json 是唯一事实源** - 所有状态来自文件，不依赖 session 记忆
2. **禁止纯文本结束回合** - 每次必须调用工具
3. **回压门强制执行** - lint → typecheck → test → build 全部通过才能标记 DONE
4. **强制状态落盘** - 每次操作后必须更新 tasks.json + PROGRESS.md（python3 写入）
5. **单文件规则** - 一次只改一个文件
6. **自驱动闭环** - Orchestrator 自动驱动 coder → qa → coder → qa 的循环，不需要人触发下一步

---

## 系统架构

```
用户（提交需求）
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│  Orchestrator Agent（自驱动循环）                               │
│  · 读取 tasks.json                                          │
│  · PENDING → spawn coder → coder DONE → spawn qa           │
│  · qa DONE → 更新 tasks.json → loop                         │
│  · 遇到 BLOCKED → 通知用户 → 退出                          │
│  · 全部 DONE → 通知用户 → 退出                              │
└───────────────────────────┬────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │  coder   │ │    qa    │ │ tasks.json│
        │GPT-5.4   │ │GPT-5.4   │ │唯一事实源 │
        │写代码+验证│ │业务复核  │ │PROGRESS.md│
        └──────────┘ └──────────┘ └──────────┘
```

---

## 三角色

### Orchestrator
- **职责**：自驱动闭环调度，不写代码
- **模型**：MiniMax-M2.7
- **运行方式**：persistent session 或 cron 定时触发
- **调度方式**：spawn coder → 阻塞等完成 → spawn qa → 阻塞等完成 → 更新状态 → loop
- **异常处理**：BLOCKED → 飞书通知用户 → 退出

### Coder
- **职责**：执行任务 + 跑回压门验证
- **模型**：GPT-5.4（不改）
- **关键约束**：
  - tasks.json 必须用 python3 写入（禁止 echo/字符串拼接）
  - 回压门必须依次执行：lint → typecheck → test → build
  - PROGRESS.md 必须同步更新

### QA
- **职责**：业务逻辑复核（静态，不重复跑测试）
- **模型**：GPT-5.4（不改）
- **关键约束**：
  - 只读不写（openclaw.json deny write/edit/exec）
  - 复核结果用 python3 写入 tasks.json
  - PROGRESS.md 必须同步更新

---

## 任务状态机

```
PENDING
    │
    │ orchestrator 领取
    ▼
IN_PROGRESS（coder 执行中）
    │
    ├──► 回压门全过 → DONE（coder 标记）
    │         │
    │         └──► QA 业务复核
    │                   │
    │                   ├──► PASS → 最终确认 DONE
    │                   │
    │                   └──► FAIL → FAIL_RETRY → PENDING（退回 coder 重做）
    │
    ├──► 回压门失败（最多重试1次）
    │         │
    │         ├──► 重试成功 → DONE → QA 复核
    │         │
    │         └──► 仍失败 → FAILED → 通知用户 → 下一个任务
    │
    └──► 连续3次 FAIL_RETRY → BLOCKED → 人工介入
```

---

## tasks.json 安全写入（强制）

所有角色写入 tasks.json 必须用 python3：

```python
import json
from datetime import datetime

with open('tasks.json') as f:
    d = json.load(f)

# 修改
for t in d['tasks']:
    if t['id'] == 'TASK-001':
        t['status'] = 'DONE'
        t['updatedAt'] = datetime.utcnow().strftime('%Y-%m-%dT%H:%M:%SZ')

# 写入（python3 自动校验 JSON 合法性）
with open('tasks.json', 'w') as f:
    json.dump(d, f, indent=2, ensure_ascii=False)
```

**禁止**：echo "..." > tasks.json、字符串拼接、JSON.stringify 手动拼接。

---

## PROGRESS.md 更新格式（强制）

每次状态变更后立即更新，格式如下：

```markdown
## [任务ID] - [ISO时间]

### 状态
- status: PENDING | IN_PROGRESS | DONE | FAIL_RETRY | BLOCKED

### 本次完成
- [具体做了什么]

### 验证结果
- Build: PASS | FAIL

### 下一步
- [下一个任务ID] 或 [全部完成]
```

---

## 自驱动调度伪代码

```
orchestrator_loop():
    WHILE True:
        tasks = read_tasks_json()

        if all_done(tasks):
            notify_user("全部任务完成")
            exit

        task = find_first_pending(tasks)
        update_status(task.id, 'IN_PROGRESS')

        # coder 执行（阻塞）
        spawn_and_wait(agentId='coder', task=task)

        # qa 复核（阻塞）
        tasks = read_tasks_json()
        if get_task(tasks, task.id).status == 'DONE':
            spawn_and_wait(agentId='qa', task=task)

        # 收尾
        tasks = read_tasks_json()
        update_progress(tasks)
        persist(tasks)

    END WHILE
```

---

## 启动方式

### 方式 1：用户指令触发（main）
```
用户 → main → sessions_spawn(orchestrator) → 自驱动循环开始
```

### 方式 2：Cron 定时调度（推荐）
```
cron（每10分钟） → 检查 tasks.json → 如有 PENDING → sessions_spawn(orchestrator)
```

---

## 项目结构标准

每个 ASDS 项目必须有：
```
项目目录/
├── tasks.json       # 任务队列（唯一事实源）
├── PROGRESS.md      # 进度日志（tail 即可巡检）
├── AGENTS.md        # 项目验证命令（lint/typecheck/test/build）
└── DEMAND.md        # 原始需求文档
```

---

## 异常处理

| 情况 | 处理 |
|------|------|
| build 失败 | coder 标记 FAILED，orchestrator 通知用户，继续下一个 |
| 连续 3 次 FAIL_RETRY | BLOCKED，orchestrator 通知用户，退出 |
| 需求模糊 | orchestrator 请求澄清，不自行猜测 |
| tasks.json 写入失败 | 不标记 DONE，记录 PROGRESS.md，等待人工 |
| Orchestrator 崩溃 | cron 定时检测，重新触发 orchestrator |

---

## 部署检查清单

```bash
# 1. Agent 就绪
openclaw agents list  # 确认 orchestrator/coder/qa

# 2. 看门狗运行
ps aux | grep watchdog  # 确认 PID

# 3. Cron 调度
openclaw cron list  # 确认 ASDS Orchestrator Scheduler

# 4. 项目结构
ls 项目/tasks.json   # 必须存在
ls 项目/PROGRESS.md  # 必须存在
ls 项目/AGENTS.md    # 必须存在

# 5. GitHub 更新
cd ~/.openclaw/workspace/skills/asds-mode && git push
```

---

## 相关文件路径

| 文件 | 路径 |
|------|------|
| Orchestrator AGENTS | `~/.openclaw/workspace-orchestrator/AGENTS.md` |
| Coder AGENTS | `~/.openclaw/workspace-coder/AGENTS.md` |
| QA AGENTS | `~/.openclaw/workspace-qa/AGENTS.md` |
| Skill | `~/.openclaw/workspace/skills/asds-mode/` |
| 设计文档 | `~/.openclaw/workspace/state/autonomous-devops.md` |
| 操作手册 | `~/.openclaw/workspace/state/autonomous-devops-manual.md` |
| GitHub | https://github.com/kingjason229-png/asds-mode |
