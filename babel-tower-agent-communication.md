# 巴别塔协议：让跨平台 AI 智能体说同一种话

> **Babel Tower —— 基于文件系统的跨平台 Agent 通讯协议**
>
> 一个下午就能跑通的零依赖方案，让 WorkBuddy、Claude Code、Codex CLI、Cursor 任意组合协同工作。

---

## 引言：Agent 之间的墙

你在 WorkBuddy 里写完了后端 API。代码、测试、文档都在任务目录里，井井有条。

现在需要前端接入。你打开了 Cursor，新建了一个聊天，开始给里面的 AI 描述接口规范——字段类型、返回值结构、错误码含义。你一边翻着 WorkBuddy 里的输出文件，一边手动复制粘贴到 Cursor 的对话框。

十分钟后你意识到，刚才复制的那个字段名好像不太对。回到 WorkBuddy 核对，再复制一次。

这个场景你很熟悉。不是因为技术不够先进，而是因为**平台之间没有对话机制**。WorkBuddy 的 Agent 不知道 Cursor 里发生了什么，Cursor 的 Agent 也读不到 WorkBuddy 的文件上下文。每个平台都是一个孤岛。

巴别塔协议想解决的就是这个问题：**让不同平台上的 Agent 能够可靠地交接工作**。

![跨平台AI智能体各自孤立，无法互通](images/agents-convergence.png)

---

## 核心洞察：最小公分母

跨平台通讯的方案很多，但都有前提条件：

| 方案 | 前提 | 实际障碍 |
|------|------|---------|
| HTTP API | 需要服务发现和可用网络 | 不是每个环境都能起服务 |
| WebSocket | 需要长连接和协议支持 | 跨平台实现参差不齐 |
| Message Queue | 需要中间件 | Redis/RabbitMQ 不是随处都有 |
| **文件系统** | **能执行代码就能读写** | **没有障碍** |

文件系统是所有 Agent 的共同交集。无论你在什么平台，只要你能运行代码，你就能 `open()` 和 `write()`。不需要 SDK，不需要配置，不需要网络。

巴别塔选择 JSONL 文件作为消息总线，原因就这么简单。

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
        ├── manifest.yaml           # 成员注册表
        ├── inbox/{agent}.jsonl     # 各成员收件箱
        └── .state.json             # 读取状态追踪
```

### 为什么是两层

**Task-scoped inbox**：每个任务独立，上下文隔离。不同任务之间不会互相干扰，任务归档时通讯记录随之归档。

**Global inbox**：跨任务广播，用于需要全局知晓的通知，比如系统阻塞、紧急调度。

这和日常工作里的逻辑一样：项目群聊项目的事，全员大群发通知。

![文件系统作为消息总线连接各平台](images/message-bus.png)

---

## 消息协议

### 消息格式

```json
{
  "id": "msg_20260612_173200_a1b2c3",
  "from": "backend-dev",
  "to": "frontend-dev",
  "type": "handoff",
  "priority": "P1",
  "subject": "API module complete",
  "body": "POST /auth/login is ready with tests. See deliverables/auth-api-spec.yaml",
  "task_id": "TASK-20260612-001",
  "artifacts": ["deliverables/auth-api-spec.yaml"],
  "created_at": "2026-06-12T17:32:00+08:00"
}
```

JSONL 格式的好处是直接可读：`cat` 查看，`grep` 搜索，`tail -f` 追踪。不需要专用工具。

### 消息类型

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

`handoff` 是最常用的类型——它把一段工作的产出、上下文、注意事项打包成一条消息，从一个人交给另一个人。

---

## 握手协议

巴别塔不是实时通讯，而是**异步持久化**。消息写入文件后一直存在，接收方随时读取。

这个设计有一个隐含的合理性：Agent 的工作本来就是异步的。写 API 需要二十分钟，画 UI 需要半小时，不需要毫秒级的实时对话，只需要可靠的交接。

### 四步握手

以一个实际的交接为例：后端完成了认证接口，需要交给前端集成。

**第一步：发送消息**

```bash
python babel-tower/scripts/babel-send.py \
  --task TASK-20260612-001 \
  --to frontend --type handoff \
  --subject "Auth API ready" \
  --body "POST /auth/login implemented with JWT. Spec in deliverables/auth-api.yaml"
```

消息被追加到前端的 `inbox/frontend.jsonl` 文件中。

**第二步：消息持久化**

JSONL 文件多了一行。即使接收方此刻不在线，消息也不会丢失。

**第三步：轮询检查**

主控 Agent 或用户在每轮对话开始时运行：

```bash
python babel-tower/scripts/babel-check.py \
  --task TASK-20260612-001 --all --unread
```

输出：`frontend 有 1 条未读消息，来自 backend，类型 handoff`。

**第四步：读取并继续**

前端 Agent 读取 inbox，获取完整的交接上下文——接口规范、文件位置、注意事项——然后继续工作。

不需要 daemon，不需要 cron，不需要长连接。每次对话开始时顺手检查一下 inbox 就行。

![异步握手协议四步流程](images/handshake-protocol.png)

---

## 快速开始

### 1. 初始化

```bash
python babel-tower/scripts/babel-init.py \
  --task TASK-20260612-001 \
  --members "orchestrator,frontend,backend" \
  --task-name "Auth refactor"
```

为每个成员创建 JSONL 收件箱。

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

### 4. 生成摘要

```bash
python babel-tower/scripts/babel-summary.py --task TASK-20260612-001
```

---

## 跨平台集成

接入方式：

| 平台 | 方式 |
|------|------|
| **WorkBuddy** | 加载 `babel-tower` Skill |
| **Claude Code** | 直接读写 inbox 文件 |
| **Codex CLI** | 调用 Python 脚本 |
| **Cursor** | 读取 JSONL |
| **任意工具** | `open()` + `write()` |

只要有文件 I/O，就能接入。不需要适配器，不需要协议转换。

![WorkBuddy作为中心枢纽连接各平台](images/workbuddy-hub.png)

---

## 与 Agent 系统集成

巴别塔可以嵌入到 Agent 的系统提示词（system prompt）中，让通讯成为 Agent 的默认行为。

具体做法是在提示词中加入一条指令："每次会话开始时，检查 babel-tower inbox 中是否有未读消息。如果有，先处理消息再继续原有任务。"

这样 Agent 一启动就自动同步状态，不需要人工提醒检查收件箱。通讯从外部工具变成了内置习惯。

---

## 为什么不是 HTTP/WebSocket？

| 维度 | HTTP API | WebSocket | 巴别塔（文件系统）|
|------|---------|-----------|------------------|
| 依赖 | 需要服务 | 需要服务+长连接 | 零依赖 |
| 持久化 | 需数据库 | 内存易失 | 文件即日志 |
| 可读性 | 需工具 | 需工具 | `cat` 直接看 |
| 跨平台 | 需 SDK | 需 SDK | 只要有文件系统 |
| 调试 | 抓包 | 抓帧 | 直接打开文件 |
| 延迟 | 毫秒级 | 毫秒级 | 秒级（轮询）|

Trade-off 是延迟。但 Agent 任务的粒度通常是分钟级，3-5 秒的轮询在分钟级工作流中完全可以接受。我们更需要的是**可靠的交接**，而不是实时的聊天。

---

## 源码

- **协议规范**：`protocols/message-schema.json`
- **核心脚本**：4 个 Python 文件，~400 行，零第三方依赖
- **模板**：`templates/manifest.yaml`
- **License**：MIT

```bash
git clone https://github.com/jafferchong/babel-tower-protocol.git
```

---

## 结语

巴别塔协议的名字取自那个关于语言分化的古老传说。但今天的情况其实更简单——不是语言不同，而是平台之间没有对话的通道。

用文件系统做消息总线，用 JSONL 做通用格式，用 inbox 做交接机制。没有复杂的协议栈，没有额外的依赖，只要 Agent 能读写文件，就能加入协作。

这就是巴别塔协议的全部。

---

*MIT License. Built for practical cross-platform agent collaboration.*
