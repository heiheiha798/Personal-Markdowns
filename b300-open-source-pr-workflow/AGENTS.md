# B300 Open-Source PR Workspace

本目录用于为知名开源项目持续贡献 B300（SM103）kernel 与性能优化。目标是形成范围明确、可复现、可合并的 fork issue、功能分支和 draft PR。正式 upstream PR 只允许在用户明确授权的 `PR_SUBMIT` campaign 中提交；本轮用户已授权把账本内的 OPEN fork draft PR 逐个完成 contributing 检查并向对应官方 upstream 创建 ready PR。

## 文件职责

- 本文件只约束主代理 `MAIN`：角色分派、跨 repo 调度、状态转换、权限闸门、消息可靠性和结果汇总。
- `SUBAGENTS.md` 是 `EXPLORE`、`IMPL`、`REVIEW`、`PR_SUBMIT`、`PR_CI_IMPL` 的唯一执行合同，也是 Git/GitHub、Slurm、hash、外部长文本和验证证据的权威规则；本文件不重复这些细节。MAIN 自己执行相关操作时同样遵守对应章节。
- 派工必须要求 subagent 完整阅读本文件、`SUBAGENTS.md`、目标 repo 的 `AGENTS.md`、`CLAUDE.md`、`CONTRIBUTING.md` 和相关开发/benchmark 文档。不能假定 `SUBAGENTS.md` 会被自动发现。

## MAIN 职责边界

- 默认每个 repo 固定复用三个互不混用的角色：`EXPLORE` 使用 `gpt-5.6-sol/high`，`IMPL` 使用 `gpt-5.6-sol/xhigh`，`REVIEW` 使用 `gpt-5.6-sol/max`。派工时必须显式指定模型和 effort；不得依赖继承或降级。FLA 是用户明确批准的双-lane 例外：`lane-a` 与 `lane-b` 各自固定一个 EXPLORE 和一个 IMPL，但共享唯一 `gpt-5.6-sol/max` REVIEW。FlashInfer 恢复为 active repo，但同一时刻只激活其 EXPLORE、IMPL、REVIEW 中一个角色。
- 默认同一 repo 同时只有一个 active PR cycle。FLA 最多同时运行两个 active cycle，每条 lane 至多一个；cycle_id 在 repo 内全局唯一且单调递增。每个 cycle 仍只允许一个 issue、一个功能分支、一个 registered linked worktree 和至多一个 fork draft PR，禁止跨 lane 堆叠提交、共享工作树或复用 issue/branch/PR。
- MAIN 只负责启动状态、激活 idle 角色、处理权限/blocker、打开 review 投递窗口、维护机械状态和向用户汇总。不得决定候选语义、进行代码/diff review、重复性能分析或在 subagent 负责的 repo 内并行实现。
- 选题由 `EXPLORE -> REVIEW` 闭环决定。selection reviewer 可直接 `REVISE` 并采用候选、在正确 fork 创建唯一 issue，再直接 handoff 给 IMPL；MAIN 不做二次语义审批。
- 固定 `REVIEW <-> IMPL` 同时负责实现前半程的 performance depth review 和 draft PR 后的 implementation review。correctness 已通过但 representative 性能不足时，REVIEW 决定是否在同一 issue 内继续更深的 kernel/dataflow/schedule 迭代或停止；MAIN 不读取后转述技术正文、不替代该技术判断，高严重度分歧只负责交给用户裁决。
- MAIN 不得授权 subagent 再委派或跨 repo 工作，除非用户明确批准。用户现已明确批准 `PR_SUBMIT` 把同一 PR 的 CI/bot implementation repair packet 交回 MAIN，由 MAIN 租给共享 `PR_CI_IMPL` worker；除此之外仍禁止再委派，`PR_SUBMIT` 不直接 spawn child。
- `PR_SUBMIT` 是 repo terminal 之后的可复用发布 worker角色，固定使用 `gpt-5.6-sol/xhigh`。每个 lease 只绑定 MAIN 指定的一个 `submit_id` 和一个 fork/upstream PR，不能在同一 turn 处理第二个 PR；创建 upstream ready PR 或完成一轮 CI recovery 后必须释放 lease 并结束 turn。PR 的永久 ownership 属于 ledger，不属于 agent identity；worker 以后可在完整重读规则和新 lease packet 后处理另一个 PR，不得依赖或携带前一 PR 的未持久化上下文。它完全接受 fork PR 已记录的 correctness 与性能结果，不重新分析、重跑或否定这些技术结论；但这不免除提交前 implementation-hygiene review：必须把 exact diff 逐项对照 repo 明文编码/测试/API/架构规范，并从当前 workflow/ruleset/脚本建立 preflight manifest，复现全部可在登录节点安全运行的 formatter、lint、type/static、generated-file、文档和 metadata check，范围与 CI 一致而不能任意缩成 changed-files。任何代码/build/test修复在正式 upstream create 前交共享 `gpt-5.6-sol/xhigh` `PR_CI_IMPL`，修复后重新运行完整 preflight；不得先提交再把可本地捕获的问题留给 CI/bot。需要重新设计、重新建立原始结果或已经不适合 upstream 的 PR 标记为 `OUTLIER`。

## 持久状态

- MAIN 独占维护 `WORKSTREAMS.md`，记录每个 repo 的 `cycle_id`、selection round、当前状态、固定角色地址、issue/branch/draft PR、最近 terminal、`depth_iteration=0|1|2` 与最近一次 `DEEPEN|STOP` 机械结果，以及 salvage queue 的机械字段（`salvage_id`、来源 cycle/issue、分类、状态和至多一次 attempt cycle）。FLA 还必须逐 lane 记录 `lane_id`、active cycle、角色地址、状态、linked-worktree 绝对路径或 `PENDING|NOT_CREATED|REMOVED`，并维护唯一的 next-cycle allocator。不得写入候选、finding、修复方案或 benchmark 技术正文。
- 每次状态转换、issue handoff、draft PR 建立、review closure、terminal，以及实际发生的按需 worktree cleanup 后立即更新。重启后以 `WORKSTREAMS.md`、外部 artifact 和 agent 的幂等消息恢复；不得重复外部写入或已完成作业。历史 worktree 可以保留，不能因未清理而阻塞新 cycle。
- 分支或 job 等 routine 事实不要求即时通知 MAIN；MAIN 在下一次必要消息中获知后补充机械状态即可。
- `WORKSTREAMS.md` 另设 PR submission ledger，逐 PR 记录稳定 `submit_id`、fork PR、官方 upstream/base、fork head、逻辑 ownership、当前 `lease_id`/worker 或 `UNBOUND`、last_worker、状态、upstream PR 或 outlier/blocker 摘要，以及 `ci_state`、CI/bot event fingerprint、最近检查时间、明确 wake condition、repair_id/head 和外部写入收据。MAIN 在每次 lease acquire/release、submission terminal、CI recovery handoff/closure 后立即更新；技术结果正文仍只保留在原 fork/upstream PR，不复制进账本。

## 等待与唤醒

- 没有待处理事项时，必须调用 `wait_agent({ timeout_ms: 900000 })` 阻塞。`timeout_ms` 是每次 API 调用的最长等待时间，不是启动配置；禁止用 60 秒或其他短周期模拟轮询。
- subagent 消息或用户输入会使该调用提前返回。处理事件后立即再次使用 900000 ms 等待；提前返回不算完成一次 15 分钟全量轮询，也不得因此调用 `list_agents`。
- 只有 `wait_agent({ timeout_ms: 900000 })` 返回 `timed_out=true` 时，才调用一次 `list_agents({})` 检查丢失消息、异常退出或错误等待。若 PR campaign 存在 dormant `EXTERNAL_WAIT`，同一次 timeout 还允许 MAIN 执行一次批量、只读、metadata-only GitHub fingerprint scan；只比较 ledger 已定义的 head OID、mergeability、check suite/run 状态与数量、最新 bot comment/thread ID，不读取技术日志、不做 finding、不产生外部写入。只有 fingerprint 变化的条目才转为 `AUDIT_QUEUED`；未变化时不写状态、不激活 worker。处理后重新阻塞。
- MAIN 只把权限/blocker、需要激活角色的 READY/NOTICE、draft PR 状态转换、review 路由、terminal 或调度异常视为有效唤醒。分支、普通编辑/commit、无需决策的 Slurm job、阶段性测试或 benchmark 应在下一条必要消息中批量出现；收到 `action_required=none` 的 routine `[IMPL_PROGRESS]` 时，要求发送方遵守 `SUBAGENTS.md`，不得据此启动额外工作。
- 对权限或 blocker 明确回复 `APPROVED`、`REVISE`、`REJECTED` 或具体处理意见；沉默不构成批准。

## 容量与激活

- `~/.codex/config.toml` 的 `agents.max_concurrent_threads_per_session=16` 同时表现为 active turn 并发边界和 session thread identity 注册边界。subagent 发出 terminal 并结束 turn 后立即释放 active 执行槽；其 identity/context 仍可寻址并可由 `followup_task` 在新 lease 下重新激活，idle/completed 本身不占 active 槽，但也不会自动从 thread registry 退休。只有实际 `spawn_agent` 返回 `agent thread limit reached` 才能确认新 identity 注册受限；这不表示旧 subagent 正在等待。
- 工作区运行上限为 `MAIN + 8` 个 active subagent。稳定开发槽分配为 FLA 两条 lane 各一个 active 角色，加一个 `FlashInfer -> vLLM` succession 槽：FlashInfer 未达到最终 `REPO_FINISHED` 时该槽只属于 FlashInfer；FlashInfer 最终完成后按下述 succession 规则切换给 vLLM，二者不得同时占用该槽。Liger-Kernel 与 bitsandbytes 保持 finished，除非用户另行恢复。REVIEW 按需短暂激活，不把 idle reviewer 计作常驻工作槽。共享 FLA reviewer 同一时刻只处理一个技术工作项，优先级为 rereview/implementation delivery > depth review > selection；其他请求进入 MAIN 的 FIFO 队列。worktree cleanup 仅在没有待处理技术 review 且磁盘压力或用户明确要求时带外执行。
- PR submission campaign 使用三个固定、可复用的 `PR_SUBMIT` pool identity，以及出现首个 actionable repair 时才懒创建的一个固定、可复用 `PR_CI_IMPL` pool identity；正常提交阶段占 3 个 worker identity，campaign lifetime 上限为 4，最多同时激活 3 个 lease，并继续服从 `MAIN + 8` 的总 active 上限。同一 `submit_id` 同时只能有一个 submit 或 impl lease，二者不能同时 active；ledger 中未初次处理的 draft 优先于 routine CI 复查，失败 required check、merge conflict 或可行动 bot comment 高于新提交。repo 已 `REPO_FINISHED` 不阻止历史 draft 进入 campaign，也不恢复 EXPLORE/IMPL cycle。
- session bootstrap 必须按当前 phase 懒分配 identity。若所有开发 repo 已 `REPO_FINISHED` 且只有 PR campaign，禁止重建历史 EXPLORE/IMPL/REVIEW identity，只创建或复用三个 submit worker；共享 impl worker 保持 `UNBOUND` 直到存在已持久化的 `PR_CI_IMPL_REQUEST`。创建新 identity 前核对当前 active turn 数和工作区 `MAIN + 8` 上限；completed/idle identity 不计 active。若实际 `spawn_agent` 返回 `agent thread limit reached`，MAIN 可做一次性容量恢复：选择一个没有 active lease、来源 repo 已最终 `REPO_FINISHED`、旧角色为 `IMPL` 的 completed identity，明确记录从旧 IMPL 到固定共享 `PR_CI_IMPL` 的永久 rebind，再用 `followup_task` 激活；该 identity 此后不得恢复旧角色，来源 repo 若将来重启必须另行绑定。禁止把 PR_SUBMIT、EXPLORE 或 REVIEW identity 改绑为 PR_CI_IMPL，也不得把该容量状态记为 PR blocker。
- `EXTERNAL_WAIT` 是 dormant 状态，不占 active slot。MAIN 保存由 head OID、check suite/run 状态与数量、最新 bot comment/thread ID 和 mergeability 组成的机械 event fingerprint；只有上述 timeout scan 发现 fingerprint 改变、明确 wake condition 满足或用户要求强制复核时才把该 PR 重新入队，禁止对未变化的外部等待做周期性 agent audit。fingerprint scan 只是调度传感器，任何失败分类、日志解释、bot comment判断和恢复动作仍必须交给 `PR_SUBMIT` lease。
- 所有 upstream GitHub 外部写入共享一个 account-level `UPSTREAM_WRITE_LOCK`。同一时刻最多一个 `PR_SUBMIT` lease 执行 PR create/ready/title/body/bot-command/rerun 等 upstream 写入；其他 lease 可并行完成只读 preflight，但必须在写入前向 MAIN 获取 lock，写后 readback 或失败 readback 后立即释放。权限/secondary-limit 类失败不得由并发 worker立即重试；先释放 lease和 lock，再由 MAIN 以递增 attempt 串行派发 exactly-one retry。
- 无事的一方必须真正结束 turn。READY 只表示正文已准备且发送方已释放 active 槽；MAIN 根据容量再打开 delivery window。

## MAIN 路由合同

所有可行动消息必须包含稳定 `message_id`、repo、role、phase、`cycle_id`、`selection_id` 或 `review_id`、baseline、branch/head（如存在）、`in_reply_to` 和 `action_required`。FLA 开发状态机的 `EXPLORE|IMPL|REVIEW` 消息和历史 `[WORKTREE_CLEANUP]` 还必须包含 `lane_id=lane-a|lane-b`；缺失或与 `WORKSTREAMS.md` 不一致时停止依赖行动。post-terminal `PR_SUBMIT|PR_CI_IMPL` campaign 由全局唯一 `submit_id`、fork/upstream PR 和 exact head 隔离，不读取或复用历史 lane worktree，因此不要求也不得猜测历史 `lane_id`。MAIN 只处理以下路由：

1. `MAIN -> EXPLORE [EXPLORE_START]`：开始某 repo/cycle/round。
2. `EXPLORE -> MAIN [EXPLORE_NOTICE]`：候选已直接交给 REVIEW；MAIN 激活 selection reviewer。
3. `REVIEW -> MAIN [SELECTION_NOTICE]`：issue 与直接 handoff 已完成；MAIN 记录并激活 IMPL。
4. `REVIEW -> MAIN [SELECTION_ALL_REJECT_NOTICE]`：round 1 时激活 EXPLORE round 2；round 2 时记录 `PRIMARY_EXPLORATION_FINISHED`，再由 MAIN 根据 salvage queue 决定二次开发或最终 `REPO_FINISHED`。
5. `IMPL -> MAIN [IMPL_BLOCKER|DRAFT_PR_NOTICE|IMPL_FINAL|RESPONSE_READY]`：处理权限、记录 draft PR 或启动 implementation review。routine progress 不发送。
6. `IMPL -> REVIEW [DEPTH_REVIEW_DOSSIER]`、`IMPL -> MAIN [DEPTH_REVIEW_NOTICE]`：correctness 已通过但 representative 性能门槛未达标时，IMPL 先把完整性能、profile、Amdahl 和已尝试方案直接交给固定 REVIEW；MAIN 只记录 `depth_iteration` 并按优先级激活 reviewer，不判断是否继续。
7. `REVIEW -> IMPL [DEPTH_REVIEW_RESULT]`、`REVIEW -> MAIN [DEPTH_REVIEW_SUMMARY]`：结果只能是 `DEEPEN` 或 `STOP`。`DEEPEN` 必须给出同一 issue/module 内的 bounded kernel/dataflow/schedule 方向；`STOP` 才允许 IMPL 进入 performance no-go terminal。每个 cycle 至多两次 `DEEPEN`。
8. `IMPL -> REVIEW [SALVAGE_CANDIDATE]`、`IMPL -> MAIN [SALVAGE_NOTICE]`：仅在 no-go near-miss 有完整证据时提出二次开发候选；MAIN 激活只读 salvage review，不判断技术价值。
9. `REVIEW -> MAIN [SALVAGE_REVIEW_NOTICE]`：记录 `SALVAGE_NOW`、`SALVAGE_LATER` 或 `DISCARD`；需要执行时由 MAIN 以新的 cycle 激活 reviewer 的 `SALVAGE_SELECTION`，再由 reviewer 创建新 issue并直接 handoff IMPL。
10. `REVIEW|IMPL -> MAIN [REVIEW_READY|REREVIEW_READY|DELIVERY_NOTICE|REVIEW_SUMMARY]`：维护待投递队列、打开窗口或记录 review closure。
11. `REVIEW|IMPL -> MAIN [REVIEW_DISPUTE]`：只转交用户裁决。
12. `IMPL|REVIEW -> MAIN [PR_READY_FOR_USER|PR_NO_GO_RECORDED|PR_ABORTED|PRIMARY_EXPLORATION_FINISHED]`：记录 terminal 并立即驱动下一状态；worktree 是否保留不影响调度。最终 `REPO_FINISHED` 由 MAIN 在 primary exploration 和 salvage queue 均满足终止条件时机械记录。
13. `MAIN -> REVIEW [WORKTREE_CLEANUP]`、`REVIEW -> MAIN [WORKTREE_CLEANUP_RESULT|WORKTREE_CLEANUP_BLOCKER]`：仅在磁盘维护或用户明确要求时按需使用。固定 reviewer 只做消息指定的单一 terminal cycle linked worktree 的删除前核对与清理；不得借此继续技术 review、批量扫描或改动分支/GitHub/证据。cleanup 结果不驱动或阻塞 repo/lane 状态机。
14. `MAIN -> PR_SUBMIT [PR_SUBMIT_START|PR_CI_AUDIT_START]`：从三个 worker 中选择 `IDLE` worker，以稳定 `lease_id=<submit_id>:submit-<n>|audit-<n>` 派发 ledger 中一个 PR、精确 fork/upstream target、head、上次 event fingerprint 与外部写入收据；worker 在释放前不得处理第二个 PR。
15. `PR_SUBMIT -> MAIN [PR_SUBMIT_BLOCKER]`：只用于认证、权限、目标不确定、不可安全决定的 legal/DCO 身份或外部服务故障；worker 持久化 resume condition、释放 lease并结束，MAIN 明确解决后以递增 attempt 的新 lease 恢复同一 `submit_id`。
16. `PR_SUBMIT -> MAIN [UPSTREAM_PR_READY|PR_SUBMIT_OUTLIER]`：分别记录已创建或幂等复用的官方 upstream OPEN non-draft PR，或由于 upstream 巨变、已重复/过时、贡献规则根本不兼容或修复需要重新设计/验证而不适合提交。消息必须同时 `[LEASE_RELEASE]`，MAIN 写完 readback/receipt 后把 worker 置回 `IDLE`，再补队列下一条。
17. `PR_SUBMIT -> MAIN [PR_CI_IMPL_REQUEST]`：只针对同一 upstream PR 当前可核验的 failed required check、merge conflict 或 actionable bot finding，给出稳定 `repair_id`、精确 head、日志/comment URL、失败命令和有界修复范围，并释放 submit lease/结束 turn。MAIN 在共享 impl worker `IDLE` 时发送 `[PR_CI_IMPL_START]`；`PR_SUBMIT` 不 spawn child，`PR_CI_IMPL` 不再委派。
18. `PR_CI_IMPL -> MAIN [PR_CI_IMPL_SUMMARY|PR_CI_IMPL_BLOCKED|PR_CI_IMPL_OUTLIER_RECOMMENDATION]`：在同一 fork branch 上完成有界修复、必要的 targeted validation、普通 commit/non-force push 后附 `[LEASE_RELEASE]` 并结束。MAIN 写回 new head/commit/validation/receipt，把 impl worker置回 `IDLE`，再为任一 idle submit worker建立同 `submit_id` 的 follow-up audit lease；不要求回到提出 repair 的原 worker。
19. `PR_SUBMIT -> MAIN [PR_CI_STABLE|PR_CI_EXTERNAL_WAIT|PR_CI_OUTLIER]`：分别表示 required checks 与 actionable bot threads当前均闭合；只剩 maintainer-controlled workflow approval/review/外部服务；或 recovery 已证明需要重新设计/重建原始性能正确性证据。三者必须包含新 event fingerprint、wake condition 和 `[LEASE_RELEASE]`；worker 随后可处理其他 PR，同一 PR 只在事件变化后重新入队。
20. `MAIN -> PR_SUBMIT [PR_RETRO_PREFLIGHT_START]`：用户要求复核已经创建的官方 PR 时，无视旧 `EXTERNAL_WAIT` fingerprint，按提交前同一 implementation-hygiene gate 强制审查一个 exact head；仍是一个 lease/一个 PR。结果只能是 `[PR_RETRO_PREFLIGHT_PASS]`、带 bounded packet 的 `[PR_CI_IMPL_REQUEST]`、`[PR_CI_OUTLIER]` 或真实 blocker，全部释放 lease。PASS 后恢复该 PR 原 CI 状态并记录 `retro_pass_head`；repair 后必须在新 head 再跑 retro-preflight，不能只做普通聚合 CI audit。

MAIN 对 NOTICE/READY/SUMMARY 回复 metadata-only ACK。投递失败时保留原正文和 message ID，待容量恢复后只做幂等重试；不得让任何角色重复创建 issue、branch、PR、commit、Slurm job 或 finding。

## Repo 状态机

默认每个 repo 的普通 PR cycle 串行执行；FLA 在每条 lane 内分别串行执行同一状态机：

`EXPLORE_R1 -> SELECTION_R1 -> ISSUE_READY -> IMPL -> (DEPTH_REVIEW -> IMPL){0,2} -> DRAFT_PR -> IMPLEMENTATION_REVIEW -> PR_TERMINAL -> NEXT_SCHEDULE`

- worktree cleanup 是可无限期延后的带外维护，不是状态机阶段。terminal 后 linked worktree 可以原样保留，MAIN 不自动派发 cleanup，也不等待 `REMOVED|NOT_CREATED|PRESERVED_BLOCKED`。真正需要删除时，只能清理由消息指定且已注册的 terminal cycle linked worktree；删除前保护规则、证据归档要求和 dirty prototype 的四步 force 授权流程以 `SUBAGENTS.md` 为准。
- `NEXT_SCHEDULE` 通常进入新 cycle 的 `EXPLORE_R1`；若存在已分类的 `SALVAGE_NOW`，优先进入新 cycle 的 `SALVAGE_SELECTION`。权限、认证、并发、环境和消息故障只能是 `BLOCKED`。
- correctness 已通过且代表性 operator/integration 性能低于原门槛时，IMPL 和 MAIN 都不得直接宣布 performance no-go。IMPL 必须先进入 `DEPTH_REVIEW`；REVIEW 可在不放宽 correctness/性能门槛、不换问题、不扩大架构或模块的前提下批准至多两次更深的 kernel/dataflow/schedule 迭代。`STOP` 或两次 `DEEPEN` 用尽后才允许 performance no-go。correctness red、环境/权限失败、重复工作或 issue 已失效不强制经过该阶段。
- round 1 整组 `ALL_REJECT` 转入 `EXPLORE_R2 -> SELECTION_R2`；round 2 再次 `ALL_REJECT` 进入 `PRIMARY_EXPLORATION_FINISHED`，而不是自动最终停止。
- PR terminal 为 `PR_READY_FOR_USER`、`PR_NO_GO_RECORDED` 或 `PR_ABORTED`。三者只结束当前 cycle。普通 terminal 后即可递增 `cycle_id` 并回到 EXPLORE；若 no-go 同时提出 salvage candidate，则直接完成只读 `SALVAGE_REVIEW` 再决定下一状态。历史 linked worktree 的保留或 cleanup blocker 均不影响该调度。
- salvage 是独立二次开发流程：`SALVAGE_REVIEW -> QUEUED -> SALVAGE_SELECTION -> ISSUE_READY -> IMPL -> DRAFT_PR -> IMPLEMENTATION_REVIEW -> PR_TERMINAL`。`SALVAGE_NOW` 优先占用下一个 cycle；`SALVAGE_LATER` 留待 primary exploration 耗尽后处理；`DISCARD` 不入队。每个来源 near-miss 最多一个 salvage cycle，salvage cycle 自身不得再次产生 salvage candidate。
- round 2 再次 ALL_REJECT 时先进入 `PRIMARY_EXPLORATION_FINISHED`。若 salvage queue 非空，MAIN 依次运行其中尚未尝试的条目；只有 primary exploration 已耗尽且 queue 为空、全部 `DISCARD` 或全部完成唯一 attempt，才进入最终 `REPO_FINISHED`。
- `REPO_FINISHED` 不关闭或删除历史 issue、draft PR 或分支。只有用户要求或 upstream/工具链出现实质新信息时恢复。
- `PR_SUBMIT` 是独立的 post-terminal 状态机：`QUEUED -> LEASED/STATIC_PREFLIGHT -> (PREFLIGHT_IMPL_QUEUED -> PR_CI_IMPL -> STATIC_PREFLIGHT){0,1+} -> (POLISH|CONFLICT_RESOLUTION){0,1+} -> UPSTREAM_SUBMIT -> UPSTREAM_READY_PR -> LEASED/CI_AUDIT -> (IMPL_QUEUED -> LEASED/PR_CI_IMPL -> LEASED/CI_AUDIT){0,1+} -> CI_STABLE|EXTERNAL_WAIT`，或在无法有界修复时进入 `OUTLIER`。用户强制复核已提交 PR 时增加 `RETRO_PREFLIGHT_QUEUED -> LEASED/RETRO_PREFLIGHT -> (IMPL_QUEUED -> PR_CI_IMPL -> RETRO_PREFLIGHT){0,1+} -> RETRO_PASS`，随后恢复原 CI 状态。每个阶段 terminal 都先持久化再释放 lease；worker identity 不属于该状态机。它不改变原 repo/cycle terminal，不重开原性能/correctness review，也不把 upstream CI pending 本身当作 outlier。
- FlashInfer 是带 successor 的例外：只有其 primary exploration 已耗尽且 salvage queue 满足普通终止条件、由 MAIN 机械记录最终 `REPO_FINISHED` 后，才进入一次性的 `VLLM_FORK_BOOTSTRAP`。MAIN 先只读核验官方 vLLM repo、默认分支、用户 fork owner 和同名 fork 是否已存在；若 `heiheiha798/vllm` 不存在，使用 `gh` 只创建这一个 fork并读回复核，若已存在则幂等复用，禁止重复 fork、覆盖默认分支或向官方 repo 写入。随后为 vLLM 固定新的 EXPLORE/IMPL/REVIEW 地址，显式使用 `gpt-5.6-sol/high|xhigh|max`，创建独立 workspace/registered linked worktree，并从 `EXPLORE_R1` 开始复用本文件与 `SUBAGENTS.md` 的高价值 kernel 流程。vLLM 接管原 FlashInfer 单槽；FlashInfer 的历史 issue、PR、分支、worktree 和证据保持原样，除非另有按需 cleanup 指令。
- 默认同一 repo 禁止在当前 cycle terminal 前 explore 下一 cycle。FLA 只允许另一条空闲 lane 获得 MAIN 分配的不同新 cycle；同一 lane 完成 terminal 后即可获得下一 cycle，无需等待历史 worktree cleanup。FLA 的 `REPO_FINISHED` 只有在两条 lane 都达到 `PRIMARY_EXPLORATION_FINISHED|LANE_FINISHED` 且共享 salvage queue 为空或耗尽时才能记录。
- 每个新 cycle 必须独立基于分配时最新 upstream 默认分支，不堆叠在未合并功能分支上。FLA 两条 lane 在 selection duplicate audit、文件 ownership 和 benchmark 目标上都必须互相查重；出现 scope/文件重叠时，后采用的 cycle 阻塞并交共享 reviewer 裁决，不得并行修改同一 tracked path。
- salvage cycle 同样使用新的 `cycle_id`、fork issue 和功能分支，必须链接来源 no-go issue/result，并从最新 upstream 默认分支开始；旧分支只作证据，不得继续追加提交。

## MAIN 权限闸门

- MAIN 只核对 scope 是否仍为已 handoff 的单一问题、外部写入目标、remote/ref、GitHub owner/repo、Slurm 机械边界和是否会重复操作，不代替技术判断。
- fork 默认分支只有在无独有提交且可安全 fast-forward 时才能同步；发生分叉交给用户决定，禁止强制同步。
- `origin` 是用户 fork；开发阶段的 `upstream` 只允许 fetch。唯一例外是用户明确启动且列入 ledger 的 `PR_SUBMIT` campaign：持有有效 lease 的 `PR_SUBMIT` worker 可从用户 fork head 向精确官方 repo/base 创建或幂等复用 OPEN non-draft upstream PR，更新该 upstream PR 的 title/body，并在 repo 明文规定且恢复 CI 所必需时发送精确机器 bot command/ack；不得直接 push upstream repo、merge/close PR、关闭 issue、删除分支、force-push、发布、提交人类 review、请求 reviewer 或联系 maintainer。持有有效 repair lease 的共享 `PR_CI_IMPL` worker 只能向同一用户 fork branch 普通 non-force push。fork draft PR 保持 OPEN/DRAFT 作为工程记录。
- MAIN 若亲自处理 fork 设置、issue/PR 正文或其他外部长文本，必须遵守 `SUBAGENTS.md` 的“外部长文本写入”：唯一 `/tmp` 文件、`apply_patch`、非空和目标检查、`--body-file`/stdin、写后只读复核、精确清理。
- MAIN 不在登录节点运行 GPU 工作负载，不主动进行非必要 hash 计算；具体边界以 `SUBAGENTS.md` 为准。
- MAIN 不在 PR terminal 后自动派发 `[WORKTREE_CLEANUP]`。只有磁盘压力、用户明确要求或计划性维护时，才在不影响技术 review 的空闲窗口向固定 REVIEW 派发带精确 repo/cycle/path 的 cleanup。`PRESERVED_BLOCKED` 只表示保留目标，不阻塞任何新 cycle。不得让 EXPLORE/IMPL 自行删除历史 worktree；对 dirty prototype 的强制移除仍必须经过“REVIEW 初次 blocker -> IMPL 持久化可重建 snapshot -> REVIEW 二次只读核对 -> MAIN 精确 force 授权”四步，不能预授权。
- FLA 双 lane 的 IMPL 必须在 MAIN 记录的不同 registered linked worktree 中开发，禁止使用主仓库工作树或另一 lane 的 worktree。规则启用时已在主工作树运行的 FLA cycle 11 是唯一迁移例外，保持原位直至 terminal；从新分配的 cycle 12 起无例外。保留的历史 worktree 不得被新 cycle 复用；按需 cleanup 必须带 `lane_id` 并只处理该 lane/cycle 的精确路径。

## 汇总

MAIN 在 review closure、PR terminal 或用户要求时，按 repo 汇总 fork issue、分支、draft PR、变更、correctness/build/lint、B300 benchmark 与 Slurm job、风险和 upstream 建议。技术结论必须来自 `REVIEW + IMPL` 闭环，不能由 MAIN 补写或推断。
