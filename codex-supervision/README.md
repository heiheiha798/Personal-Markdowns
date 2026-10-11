# dot 监督服务器 Codex 的工作流

状态核验：2026-10-11 10:04（UTC+8）。源码依据：Codex `rust-v0.160.1`。

## 1. 整体链路

```text
用户设定目标与权限
       ↓
ChatGPT 中的 dot：审查结果、判断方向、制定下一步
       ↓
已授权且在线的用户桌面任务
       ↓ 现有 SSH
服务器：tmux 中的 Codex TUI
       ↓
代码、测试、实验原始证据 → dot review → 下一轮
```

云端不直接 SSH 到服务器。桌面任务负责读取现场和执行已授权操作，dot 负责审查与决策。GitHub 用于保存代码和证据，不必用 issue comment 中转每条指令。

每轮先核实实际项目、Codex pane、运行状态及未提交输入。正在进行合理实验时继续观察；回合结束后检查 diff、测试与原始证据，再决定是否发送下一步。每条指导明确目标、范围、验收标准和停止条件。

投递前再次确认目标和空闲状态；投递后核实接收并记录，结果不明时先查，不重复发送。不覆盖用户草稿，不向 shell 误投提示词，不用 Ctrl-C 打断，不启动重复 GPU 队列。合并、部署或扩大权限仍需相应授权。

## 2. “Working” 背后的逻辑

官方源码中，`TurnStarted` 进入运行状态，`TurnCompleted` 进入回合结束清理。TUI 内部维护 `agent_turn_running` 等状态，界面根据这些状态显示忙碌指示；审查模式、启动过程也可能影响忙碌显示。

因此，“Working” 有真实状态依据，但不能只检测屏幕文字：流式输出可能暂时隐藏指示，审批界面也可能覆盖它。字样消失不等于成功，回合结束也不等于整个开发任务验收完成。

应区分：

- `active`：回合活跃；还需识别 `waitingOnApproval`、`waitingOnUserInput`
- `idle`：当前空闲，仍需审查结果
- `systemError` / `notLoaded`：检查异常或加载状态
- 断线：状态未知，不能当作完成

## 3. 事件驱动监督的原理

当前现场已确认：TUI 连接着既有的 v0.160.1 daemon，并已核实对应 Unix socket 属于同一工作进程。

最小只读观察流程：

1. 连接这个既有 daemon，完成 `initialize` / `initialized`
2. 用 `thread/loaded/list` 和 `thread/read(includeTurns: false)` 识别目标线程、取得初始状态
3. 接收 `thread/status/changed`，只处理匹配的线程
4. 出现空闲、错误或等待批准等变化时，让桌面任务回传，触发 dot review

该版本的状态变化会向已初始化客户端广播，不需要为观察调用 `thread/resume`、`thread/start` 或启动另一个 app-server。必须连接持有当前任务的那个 daemon；另开服务不能据此获知旧 TUI 的实时状态。

连接后及重连时重新读取状态，不能假设广播可补发。只记录必要的线程标识、状态与时间，去重并保证监督串行；不收集其他线程的提示词正文。

## 4. 当前验证边界

- 已运行：三路独立 worktree / tmux session / Codex thread；dot 审查后，通过桌面 SSH 直接粘贴 prompt 并按 Enter，核实首条执行响应，避免把“中转接单”当成“服务器接收”
- 已确认：官方状态逻辑、当前 TUI 与既有 daemon/socket 的对应关系，以及三线程状态读取
- **已实测完成事件**：2026-10-11 10:03:13（UTC+8），只读观察器收到真实开发回合的 `active → idle`，约 38 秒即结束等待并回传最终输出，未等待 600 秒上限
- 定时检查已从每 20 分钟降为每小时；活跃工作以事件回传接续审查
- 尚未证明：长期无人值守、各种断线场景及所有错误/审批状态的完整可靠性

观察器是有界只读连接，不是无限常驻服务。回传后需要重新核实各线程并继续订阅；若建立连接时目标已空闲，直接读取结果，不等下一次状态变化。超时仅说明窗口内未见终态，断线必须记为 unknown，重新连接后读取现态；不能假设事件会重放。

### 本次启动问题

两个新 TUI 曾在 `account/read` 阶段报 workspace routing discovery timeout；改用独立 tmux session 本身没有解决。官方 0.160.1 源码显示，`-c model_reasoning_effort="xhigh"` 会排除隐式 daemon 复用。

显式 `--remote unix://` 可连接当前用户已有的 control socket；本次使用已核实的既有 socket 后成功进入新会话。它仍执行正常路由/认证，不是跳过认证，也不保证修复所有超时。新线程可分别指定工作目录、模型、推理强度与权限；文件夹信任、Full Access 等必须由用户明确批准。

已有 daemon 不继承新 TUI 客户端的环境变化。若需给新任务设置工具环境，应使用支持的线程级配置并核实实际值；例如仅在客户端设置 `CUDA_VISIBLE_DEVICES` 不能证明远端工具也使用该值。环境变量也不是权限隔离的替代品。

`notify` 的 `agent-turn-complete` 现场链路仍未验证成功；当前已验证的是既有 daemon 的状态事件路径。不能把本方案描述成任意 webhook 接口，也不能将 `idle` 当作任务验收通过。

## 官方源码

以下链接固定版本；升级 Codex 后需要重新核对协议和实现。

- [TUI 协议事件处理](https://github.com/openai/codex/blob/rust-v0.160.1/codex-rs/tui/src/chatwidget/protocol.rs)
- [回合运行与清理](https://github.com/openai/codex/blob/rust-v0.160.1/codex-rs/tui/src/chatwidget/turn_runtime.rs)
- [线程状态管理](https://github.com/openai/codex/blob/rust-v0.160.1/codex-rs/app-server/src/thread_status.rs)
- [消息发送与广播](https://github.com/openai/codex/blob/rust-v0.160.1/codex-rs/app-server/src/outgoing_message.rs)

- [隐式 daemon 复用条件](https://github.com/openai/codex/blob/rust-v0.160.1/codex-rs/tui/src/daemon_startup.rs)
- [远程会话参数与初始化](https://github.com/openai/codex/blob/rust-v0.160.1/codex-rs/tui/src/app_server_session.rs)
