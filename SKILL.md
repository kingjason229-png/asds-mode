---
name: asds-mode
description: ASDS 全自动开发工作流。当用户提到「全自动开发」「ASDS」「autonomous devops」「无人值守」「持续集成」时激活。
---

# ASDS Mode - 全自动软件开发工作流

ASDS（Autonomous Software Development System）是一套 7×24 小时无人值守的全自动软件开发系统。

## 核心原则

1. **tasks.json 是唯一事实源** - 不从对话历史读进度
2. **禁止纯文本结束回合** - 每次必须调用工具
3. **回压门必须执行** - lint → typecheck → test → build 全部通过才能标记 DONE
4. **强制状态落盘** - 每次操作后必须更新 tasks.json 和 PROGRESS.md
5. **单文件规则** - 一次只改一个文件

---

## 系统架构

```
用户（提交需求）
    │
    ▼
main (Orchestrator) ← 调度中枢
    │
    ├─ PM 角色：main 读取 DEMAND.md → 拆解为 tasks.json
    │
    ├─ Developer (coder)：领 PENDING 任务 → 实施 → 回压门验证 → 标记 DONE
    │
    └─ QA (qa)：复核 DONE 任务 → 对照 acceptance_criteria → PASS 则最终确认 / FAIL 则退回
    │
    ▼
tasks.json（唯一事实源）
PROGRESS.md（进度摘要）
```

---

## 三角色分工

### main（Orchestrator）
职责：调度中枢，不写代码
- 读取 DEMAND.md，拆解为 tasks.json（PM 角色）
- 监控 tasks.json 状态，驱动循环
- 处理异常（FAILED → 重试，BLOCKED → 通知用户）
- 不写代码，只调度

### developer（coder）
职责：执行任务、跑验证命令
- 领 PENDING 任务，一次只做一件
- 实施后必须跑回压门：lint → typecheck → test → build
- 全部通过 → tasks.json 标记 DONE
- 失败 → 修复重试，最多 3 次
- 每次操作强制落盘 tasks.json + PROGRESS.md
- 模型固定 GPT-5.4

### qa
职责：业务逻辑复核，不重复跑测试
- QA 不跑测试（那是 developer 的职责）
- 读取 DONE 任务的源码，对照 acceptance_criteria 逐条复核
- 业务逻辑满足 → 最终 PASS
- 不满足 → 标记 FAIL，退回 developer 重做
- 只读不写（openclaw.json deny write/edit/exec）
- 模型固定 GPT-5.4（不改）

---

## 任务状态机

```
PENDING
    │
    │ developer 领取
    ▼
IN_PROGRESS
    │
    ├──► 回压门全过 → DONE（developer）
    │         │
    │         └──► QA 复核业务逻辑 → 最终确认 / 退回
    │
    ├──► 回压门失败（重试1次）
    │         │
    │         ├──► 重试成功 → DONE → QA 复核
    │         │
    │         └──► 仍失败 → FAILED → 通知用户
    │
    └──► 连续3次失败 → BLOCKED → 人工介入
```

---

## 自动化调度循环（核心）

main 的调度逻辑：

```
WHILE tasks.json 有 PENDING 任务：
    # 1. 取出最前面的 PENDING 任务
    task = pop(tasks.json, PENDING)
    tasks.json[task.id].status = IN_PROGRESS

    # 2. 启动 developer 执行
    spawn(coder, task)

    # 3. 等待 developer 完成（DONE 或 FAILED）
    wait_for_done(task.id)

    # 4. 如果 developer DONE，启动 QA 复核
    IF tasks.json[task.id].status == DONE:
        spawn(qa, task)
        wait_for_qa_result(task.id)

    # 5. 如果 QA FAIL：
    IF qa_result == FAIL:
        tasks.json[task.id].status = PENDING  # 退回重做
        task.attempts += 1
        IF task.attempts >= 3:
            tasks.json[task.id].status = BLOCKED
            notify_user("需要人工介入")

    # 6. 强制落盘
    persist(tasks.json)
    persist(PROGRESS.md)
END WHILE

# 全部 DONE → 通知用户验收
IF all_done:
    notify_user("项目完成，等待验收")
```

---

## 单次任务执行流（developer）

```
1. 读取 tasks.json，找到 PENDING 任务
2. 读取项目 AGENTS.md，了解验证命令
3. 标记 tasks.json[task.id].status = IN_PROGRESS
4. 写代码（一次只改一个文件）
5. 依次执行回压门：
   npm run lint          # 失败则修复，重跑
   npx tsc --noEmit      # 失败则修复，重跑
   npm run test          # 失败则修复，重跑
   npm run build         # 失败则标记 FAILED，通知用户
6. 全部通过 → tasks.json[task.id].status = DONE
7. 更新 PROGRESS.md
```

---

## PM 拆解标准（main 执行）

每个任务必须包含：
```json
{
  "id": "PROJ-001",
  "title": "任务简短描述",
  "description": "详细描述",
  "acceptance_criteria": [
    "标准1（可验证）",
    "标准2（可验证）"
  ],
  "status": "PENDING",
  "priority": "high | medium | low",
  "dependencies": [],
  "estimatedHours": 1
}
```

**acceptance_criteria 必须可验证**：
- ❌ "代码要写得好" → 无法验证
- ✅ "convertCtoF(0) 返回 32" → 可验证
- ✅ "npm run build 成功" → 可验证

---

## 项目结构标准

每个 ASDS 项目必须有：
```
项目目录/
├── tasks.json       # 任务队列（唯一事实源）
├── PROGRESS.md      # 进度日志，tail 即可巡检
├── AGENTS.md        # 项目验证命令（lint/typecheck/test/build）
└── DEMAND.md        # 原始需求文档
```

---

## 触发方式

ASDS 可以通过两种方式触发：

**方式 1：用户直接指令**
```
"用 ASDS 开发一个温度转换工具"
→ main 读取 → 拆解 → 启动循环
```

**方式 2：Cron 定时驱动**
```
每天 03:00 Nightly Build
→ 检查 tasks.json
→ 对 PENDING 任务启动循环
```

---

## 验证命令（项目 AGENTS.md 标准格式）

```markdown
# AGENTS.md - 项目操作指南

## 验证命令（必须按顺序执行）
| 类型 | 命令 | 失败处理 |
|------|------|---------|
| Lint | `npm run lint` | 修复后重跑 |
| Typecheck | `npx tsc --noEmit` | 修复后重跑 |
| Test | `npm run test` | 修复后重跑 |
| Build | `npm run build` | 标记 FAILED，通知用户 |

## 注意事项
- 测试必须通过才能提交
- Typecheck 失败阻塞提交
```

---

## 异常处理

| 情况 | 处理 |
|------|------|
| build 失败 | 标记 FAILED，通知用户，跳过继续下一个 |
| 连续 3 次失败 | 标记 BLOCKED，通知人工介入 |
| 需求模糊 | 停止，请求澄清，不自行猜测 |
| QA 复核 FAIL | 退回 PENDING，attempts+1，重试 |

---

## 部署检查清单

运行 ASDS 前确认：
```bash
# 1. Agent 就绪
openclaw agents list  # 确认 coder / qa 存在

# 2. 看门狗运行
ps aux | grep watchdog  # 确认有进程

# 3. 项目结构完整
ls 项目目录/tasks.json   # 必须存在
ls 项目目录/PROGRESS.md  # 必须存在
ls 项目目录/AGENTS.md   # 必须存在

# 4. Cron 任务
openclaw cron list  # 确认触发器已注册
```

---

## 相关文件

- 完整设计：`state/autonomous-devops.md`
- 操作手册：`state/autonomous-devops-manual.md`
- 角色定义：
  - coder：`.openclaw/workspace-coder/AGENTS.md`
  - qa：`.openclaw/workspace-qa/AGENTS.md`
  - orchestrator：`.openclaw/workspace-orchestrator/AGENTS.md`
