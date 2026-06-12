# 巴别塔协议：让跨平台 AI 智能体说同一种话

> **Babel Tower —— 基于文件系统的跨平台 Agent 通讯协议**
>
> 一个下午就能跑通的零依赖方案，让 WorkBuddy、Claude Code、Codex CLI、Cursor 任意组合协同工作。

---

## 问题：Agent 的"巴别塔困境"

多智能体协作已成主流，但平台间的隔离比想象更严重：

- WorkBuddy 的 Team 无法直接唤醒 Claude Code 的子 Agent
- Codex CLI 的任务产出，Cursor 里的 Agent 看不到上下文
- 每个平台有自己的通讯机制，互不相通

**核心矛盾**：Agent 越来越聪明，但 Agent 之间的"语言壁垒"越来越高。

![跨平台AI智能体各自孤立，无法互通](images/agents-convergence.png)

巴别塔协议用最朴素的思路解决——文件系统是所有 Agent 的共同交集。

---

## 设计原则：最小公分母

| 方案 | 依赖 | 问题 |
|------|------|------|
| HTTP API | 网络、服务发现 | 需要 daemon，配置复杂 |
| WebSocket | 长连接、协议栈 | 跨平台支持参差不齐 |
| Message Queue | 中间件 | 引入外部系统，重 |
| **文件系统** | **无** | **任何 Agent 都能读写** |

巴别塔选择 JSONL 文件作为消息总线。原因只有一个：**如果 Agent 能执行代码，它就能 open() 和 write()**。

---

## 架构：两层 inbox

```
workspace/
├── babel-tower/                    # 协议根（模板+协议+脚本）
│   ├── scripts/                    # babel-init / send / check / summary
│   ├── protocols/                  # message-schema.json
│   └── templates/                  # manifest.yaml
│
└── tasks/active/TASK-xxx/
    └── babel-tower/                # 任务级实例（运行时生成）
        ├── manifest.yaml           # 团队成员注册表
        ├── inbox/{agent}.jsonl     # 各成员收件箱
        └── .state.json             # 读取状态追踪
```

**双模式设计**：
- **Task-scoped inbox**：每个任务独立，隔离上下文
- **Global inbox**：跨任务通知，用于会话级广播和调度

![文件系统作为消息总线连接各平台](images/message-bus.png)

---

## 消息协议

### 消息格式（JSONL）

```json
{
  "id": "msg_20260612_173200_a1b2c3",
  "from": "backend-dev",
  "to": "frontend-dev",
  "type": "handoff",
  "priority": "P1",
  "subject": "API module complete",
  "body": "POST /auth/login is ready with tests",
  "task_id": "TASK-20260612-001",
  "artifacts": ["deliverables/auth-api-spec.yaml"],
  "created_at": "2026-06-12T17:32:00+08:00"
}
```

### 12 种消息类型

| 类型 | 用途 |
|------|------|
| `task_assign` | 分配子任务 |
| `task_report` | 汇报进度/结果 |
| `handoff` | 工作交接（A 完成 → B 继续）|
| `query` | 查询/询问 |
| `block` | 阻塞（等待外部输入）|
| `unblock` | 解除阻塞 |
| `ack` | 确认收到 |
| `broadcast` | 广播通知 |
| `close` | 关闭任务 |
| `plan_req` | 请求制定计划 |
| `plan_resp` | 返回计划 |
| `summary` | 生成会话摘要 |

---

## 握手协议：异步持久化

巴别塔不是实时通讯，而是**异步持久化**——消息写入文件后一直存在，接收方按需读取。

**四步握手**：

1. **A 完成** → `babel-send --to B --type handoff`
2. **消息落入 B 的 inbox**（JSONL 追加）
3. **调度者轮询** → `babel-check --all --unread` 发现 B 有新消息
4. **唤醒 B** → B 读取 inbox 获取完整上下文，继续执行

关键洞察：**不需要 daemon、不需要 cron**。调度者每轮对话开始时检查 inbox，天然实现异步协作。

![异步握手协议四步流程](images/handshake-protocol.png)

---

## 快速开始

### 1. 初始化任务通讯

```bash
python babel-tower/scripts/babel-init.py \
  --task TASK-20260612-001 \
  --members "orchestrator,frontend,backend" \
  --task-name "Auth refactor"
```

### 2. 发送消息

```bash
# 任务内交接
python babel-tower/scripts/babel-send.py \
  --task TASK-20260612-001 \
  --to frontend --type handoff \
  --subject "API ready" --body "Start UI integration"

# 全局通知
python babel-tower/scripts/babel-send.py \
  --global --to orchestrator \
  --type query --subject "DB migration status"
```

### 3. 检查收件箱

```bash
python babel-tower/scripts/babel-check.py \
  --task TASK-20260612-001 --all --unread

python babel-tower/scripts/babel-check.py --global --all --unread
```

### 4. 生成会话摘要

```bash
python babel-tower/scripts/babel-summary.py --task TASK-20260612-001
```

---

## 跨平台集成

巴别塔不绑定任何框架。只要支持文件 I/O，就能接入：

| 平台 | 接入方式 |
|------|---------|
| **WorkBuddy** | 加载 `babel-tower` Skill |
| **Claude Code** | 直接读写 inbox 文件 |
| **Codex CLI** | 通过 Python 脚本调用 |
| **Cursor** | 子 Agent 读取 JSONL |
| **任意工具** | `open()` + `write()` |

![WorkBuddy作为中心枢纽连接各平台](images/workbuddy-hub.png)

---

## 与 GEM 架构集成

巴别塔作为 **Gene 16 (BabelComm)** 嵌入 GEM 架构：

```
Gene 13 (DevTeamAssemble) 触发团队组建
        ↓
自动初始化 babel-tower（任务级 inbox）
        ↓
Gene 16 (BabelComm) 激活通讯协议
        ↓
成员通过 inbox 异步协作
```

SOUL.md 中注入 Gene 16 后，Agent 在会话启动时自动检查 inbox，实现"上线即同步"。

---

## 为什么不用 HTTP/WebSocket？

| 维度 | HTTP API | 巴别塔（文件系统）|
|------|---------|------------------|
| 依赖 | 需要服务 | 零依赖 |
| 持久化 | 需数据库 | 文件即日志 |
| 人类可读 | 需工具 | cat 直接看 |
| 跨平台 | 需 SDK | 只要有文件系统 |
| 调试 | 抓包 | 直接打开 JSONL |

** trade-off**：延迟（轮询 vs 实时）。但 Agent 任务的粒度通常是分钟级，3-5 秒的轮询延迟完全可以接受。

---

## 源码与协议

- **协议规范**：`protocols/message-schema.json`
- **核心脚本**：4 个 Python 文件，共 ~400 行
- **模板**：`templates/manifest.yaml`
- **License**：MIT

```bash
git clone https://github.com/YOUR_USERNAME/babel-tower.git
```

---

## 总结

巴别塔协议的核心就一句话：**用文件系统做消息总线，让所有 Agent 都能读写**。

不追求实时，追求** universally accessible**；不引入中间件，追求**零依赖**；不做复杂协议，追求**人类可读**。

当一个 WorkBuddy Agent 完成前端代码，通过 JSONL 文件交接给 Codex CLI 做测试时——巴别塔的愿景就实现了。

---

*Built for cross-platform agent collaboration. Inspired by the original Tower of Babel — this time, everyone speaks the same language.*
