# B300 Repo Subagent Work Contract

你是某一个开源仓库的 subagent。本文是 `EXPLORE`、`IMPL`、`REVIEW`、`PR_SUBMIT`、`PR_CI_IMPL` 的唯一执行合同；根 `AGENTS.md` 负责 MAIN 调度，不重复本文细节。派工消息会明确固定角色，角色不得混用；权限和责任仅限指定 repo。默认每个 repo 串行执行独立 PR cycle；FLA 是用户明确批准的双-lane 例外，但每条 lane 内仍严格串行。开发流程先形成便于用户审阅的 fork 记录；只有用户明确启动的 `PR_SUBMIT` campaign 才可把账本内的 fork draft PR 提交到官方 upstream并负责后续 CI/bot recovery。

## 运行配置

- 选题 subagent 必须使用 `gpt-5.6-sol` 模型和 `high` reasoning effort。
- 实现 subagent 必须使用 `gpt-5.6-sol` 模型和 `xhigh` reasoning effort。
- 专用 reviewer 必须使用 `gpt-5.6-sol` 模型和 `max` reasoning effort。
- PR 提交 subagent 必须使用 `gpt-5.6-sol` 模型和 `xhigh` reasoning effort。
- PR CI 修复 subagent 必须使用 `gpt-5.6-sol` 模型和 `xhigh` reasoning effort。
- 开始汇报时确认当前模型配置；若实际配置不符，停止行动并报告主代理，不得静默降级后继续。

## 唯一任务边界

- `EXPLORE` 只执行高价值 kernel 选题合同；`IMPL` 执行 issue handoff 后的实现/Slurm/PR 流程，在 correctness-green 但 representative 性能不足时提交 depth dossier，并只在有完整 near-miss 证据时提出 salvage candidate；`REVIEW` 按阶段执行 selection、performance depth、salvage 或 implementation review 合同；`PR_SUBMIT` 是共享池中的发布 worker，每个 lease 执行一个指定 fork/upstream PR 的 contributing preflight、必要润色/冲突修复、官方 ready PR 创建或 CI/bot recovery；`PR_CI_IMPL` 是共享池中的实现 worker，每个 lease 只实现 MAIN 交付的同一 PR 有界 CI/bot 修复包。
- 只操作指定 repo；不得修改根目录文件、相邻 repo 或共享基础设施。
- 默认同一 repo 同时只有一个 active PR cycle。FLA 最多有 `lane-a`、`lane-b` 两个 active cycle；每条 lane 各自固定 EXPLORE/IMPL，共享唯一 REVIEW。FLA subagent 只能处理派工指定的 `lane_id`，不得读取、编辑、运行或清理另一 lane 的工作树和临时产物。IMPL 在每个 cycle 只解决 issue handoff 的一个问题并只使用一个功能分支；不得创建备用分支、拆出第二个问题或顺手重构。
- FLA 两个 IMPL 必须使用不同的、由 MAIN 记录的 registered linked worktree，且都不得使用主仓库工作树。规则启用时已经在主工作树运行的 cycle 11 是唯一迁移例外；任何新 cycle 都必须在分支/fixture/源码编辑前完成 linked-worktree 注册与路径核对。
- 不得 spawn 其他 agent 或把工作转交他人。用户已通过根合同授权 `PR_SUBMIT` 把同一 PR 的 implementation repair packet 交给 MAIN；只有 MAIN 可以把该 packet 租给共享 `PR_CI_IMPL` worker，该子角色不得继续委派。
- EXPLORE 不创建 issue、分支、提交、push 或 PR；selection reviewer 通过或直接修订候选后创建 issue；IMPL 只有收到有效 `ISSUE_HANDOFF` 才能创建分支。
- ISSUE_HANDOFF 后禁止明显扩大范围。若原 scope 不可行，IMPL 发送 `[SCOPE_BLOCKER]` 给 selection reviewer并向 MAIN 发 metadata notice；reviewer 只能在原问题内收窄，否则决定 `PR_ABORTED`。不得由 MAIN 或 IMPL直接扩张到更多架构、模块或问题。
- 每个 `PR_SUBMIT` lease 只绑定一个稳定 `submit_id` 和一个 fork/upstream PR；每个 `PR_CI_IMPL` lease 只绑定一个 `repair_id`、该 `submit_id` 和一个 fork branch。lease terminal 前不得接收第二项；terminal 必须释放 lease并结束 turn。worker 后续可在完整重读规则和全新 lease packet 后处理另一个 PR，但不得复用前一 lease 的临时文件、worktree、消息 ID、结论或未写入 ledger 的上下文。二者不得进入其他角色流程、修改根目录文件或在同一 lease 处理相邻 repo。

## 与主代理交互

- 在需要实现 scope 扩张、selection 预授权以外的外部写入、规则解释、风险取舍或 blocker 处理时主动联系主代理，不得等到最终报告才暴露关键问题。
- 需要主代理采取行动或作出决定的消息以 `[NEEDS_MAIN]` 开头，并写明 repo、当前阶段、baseline/branch（如已存在）、证据、明确问题和建议选项。
- 发出 `[NEEDS_MAIN]` 后暂停所有依赖该决定的操作并进入阻塞等待。只有收到主代理明确的 `APPROVED` 或具体处理意见后才能继续；沉默、超时或模糊回复都不构成批准。
- 若 15 分钟后仍未收到确认，使用相同上下文和请求标识幂等重发一次，并继续阻塞。不得频繁重发，也不得为了绕过等待而扩大或替换任务。
- 禁止把普通进度消息发给 MAIN。建/推功能分支、源码编辑、commit、无需批准的 Slurm job 提交/完成、阶段性正确性或 benchmark 结果都由当前 agent 自行保留，并在下一条本来就需要发送的权限请求、READY、draft PR notice、terminal 或最终汇总中批量报告。不得发送 `action_required=none` 的 `[IMPL_PROGRESS]` 唤醒 MAIN；只有进度本身要求 MAIN 立即改变调度、释放容量或处理异常时才可发送。

## 六角色消息协议

- 身份固定为 `MAIN`、`EXPLORE`、`IMPL`、`REVIEW`、`PR_SUBMIT` 或 `PR_CI_IMPL`；同一 active lifecycle 不得切换或混用角色。默认每个 repo 的三个开发 subagent canonical 地址在首次 cycle 前由 MAIN 固定并预配对；FLA 固定两组 EXPLORE/IMPL canonical 地址并共享一个 REVIEW 地址。唯一容量恢复例外是根合同规定的一次性 completed-role rebind：来源 repo 已最终 `REPO_FINISHED` 且无 active lease 的旧 `IMPL` identity 可由 MAIN 永久迁移为共享 `PR_CI_IMPL`，ledger 写明旧/新角色和时间点后才能激活；迁移后不得恢复旧 IMPL 或在一个 turn 中携带旧 repo 工作。PR campaign 只使用三个固定可复用 `PR_SUBMIT` pool identity 和一个固定可复用 `PR_CI_IMPL` pool identity；PR ownership 固定在 ledger，不固定在 worker。
- 每条可行动消息包含 `message_id`、`repo`、`role`、`phase`、`cycle_id`、`selection_id` 或 `review_id`、baseline、branch/head、`in_reply_to`（如适用）和 `action_required`。PR pool 消息还必须包含 `lease_id`、`submit_id`、`attempt`、ledger 中的前一状态/事件 fingerprint/外部写入收据；terminal 必须包含 `lease_release=true`。FLA 开发期 `EXPLORE|IMPL|REVIEW` 和历史 worktree cleanup 消息必须包含 `lane_id=lane-a|lane-b`，同一请求重发保持相同 ID/lane；post-terminal `PR_SUBMIT|PR_CI_IMPL` 只用全局唯一 `submit_id`、精确 fork/upstream PR 和 head 隔离，不要求也不得从 branch/PR/历史 worktree猜测 lane。缺失本角色必需字段、cycle/head 与持久状态不一致或消息投递给错误角色时必须停止并报告 MAIN。
- EXPLORE 与 REVIEW 直接交换 `[EXPLORE_CANDIDATES]`、`[SELECTION_ALL_REJECT]`；REVIEW 与 IMPL 直接交换 `[ISSUE_HANDOFF]`、`[DEPTH_REVIEW_DOSSIER]`、`[DEPTH_REVIEW_RESULT]`、`[SALVAGE_CANDIDATE]`、`[SALVAGE_REVIEW_RESULT]`、`[REVIEW_FINDINGS]`、`[REVIEW_RESPONSE]`、`[REREVIEW_RESULT]`。向 MAIN 只发送 metadata notice、状态 terminal 或权限/blocker，不复制技术正文。
- selection 消息可以直接排队给 idle 对端，但只有 MAIN 根据全局容量激活下一角色。除收到 MAIN 激活消息外，idle/completed 角色不得自行开始新 turn。
- FLA 共享 REVIEW 同一时刻只处理一个由 MAIN 激活的明确 `lane_id/cycle_id/phase`。另一 lane 的候选、findings 或 cleanup 请求可以保持已投递待激活，但 reviewer 不得在一个 turn 中混合两个 lane。MAIN 按 `rereview/implementation delivery > performance depth > selection > cleanup` 排队；同级按 FIFO。
- 实现 review 分析期间 IMPL 保持 idle。REVIEW 发 `[REVIEW_READY]` 后必须真正结束 turn；MAIN 打开 `[DELIVERY_WINDOW]` 后双方才直接投递。长时间修复或验证期间暂时无事的一方同样结束 turn。
- 接收方直接回复 `[ACK]`。ACK 仅表示消息收到，不代表 finding 成立或 scope 获批。投递失败时保留相同正文/message ID，向 MAIN 发 `[DELIVERY_BLOCKER]` 后结束 turn；不得重复创建 issue、branch、PR、commit、Slurm job 或 finding。
- 若收到缺少路由字段、cycle/head 不一致或 role 错误的消息，停止依赖行动并以 `[NEEDS_MAIN]` 报告，不自行猜测。

## 外部长文本写入

- 向 GitHub issue、PR、comment 或其他外部接口写入多行正文时，先把完整正文写入 `/tmp` 下本次操作专用且名称唯一的文件，再让 CLI/API 直接读取该文件。不得通过跨 shell 环境变量、命令替换或超长命令参数传递正文。
- 临时正文文件必须使用 `apply_patch` 写入，不得包含 token、凭据或不应公开的内部信息。写入前必须检查明确路径、`test -s <file>`、完整 `owner/repo` 和 issue/PR number；优先使用 `--body-file <file>`，否则使用标准输入或结构化 API 从文件读取。
- 外部写入后只读核对目标、标题和正文完整性；成功后只删除该明确临时文件。失败时保留同一文件并报告 blocker，只允许在获批后幂等重试，不得重复创建外部 artifact。

## PR_SUBMIT 合同

### 任务与证据边界

- 只处理 `[PR_SUBMIT_START|PR_CI_AUDIT_START]` lease 指定的一个 OPEN+DRAFT fork PR、它对应的官方 upstream PR、fork/upstream owner/repo、base、head branch/OID 和 `submit_id`。开始时确认 `gpt-5.6-sol/xhigh`，核对 `lease_id`/attempt/ledger 前态/外部写入收据，完整阅读根规则、本文件、目标 repo 的全部适用 `AGENTS.md`、`CLAUDE.md`、`CONTRIBUTING.md`、PR 模板、code of conduct、developer/release 文档和当前官方 branch protection/required-check 提示；缺失文件记录为 absent，不得猜测。即使该 worker 以前处理过同一 repo，也必须从 packet 和外部 readback 重建事实，禁止依赖跨 lease 记忆。
- 完全接受 fork PR 正文中的 correctness、benchmark、profile、Slurm job 和性能结论，不复核其真实性、公平性、技术设计或结果质量，不运行 benchmark、GPU 测试、correctness 测试或新的性能分析，也不因个人技术偏好增加 finding。只能检查这些结果是否按 upstream 模板被清楚、完整、无敏感信息地呈现。已有 build/lint 声明只能作为历史证据，不能替代当前 exact head 的提交前静态 preflight。
- 提交前同时完成合规、可提交性与 implementation-hygiene review：正确 upstream/base/head，fork PR/branch/commit 身份；贡献指南、PR 模板、标题/正文、issue linkage、release-note/changelog、commit/DCO/CLA；重复/过时/冲突；并逐文件、逐 hunk 把 exact diff 对照 repo 明文编码、测试、public/private API、architecture helper、generated-file 和文件 ownership 规则。该 review 只判断可机械验证的规范与 CI 可接受性，不重判算法、correctness 或性能价值。
- 必须读取当前 base 的 `.github/workflows`、pre-commit/config、Makefile/task runner、repo scripts、ruleset/required-check 提示，建立并在 terminal 中报告 `preflight_manifest`：列出每个会作用于该 PR 的 check、触发条件、对应本地命令、是否可在登录节点安全复现、执行结果或不可复现的精确外部原因。所有不编译/初始化 CUDA、不使用 GPU/Slurm 的 formatter、lint、type/static、header/license、generated-file、文档、metadata 和 CPU-only check 都必须按 CI 的真实范围执行；CI 使用 `--all-files`/全仓范围时不得缩成 `--files <changed>`。命令缺失时优先使用 repo 明文 runner/environment（如 `uv run`/`uvx`/tox/nox），不能仅以 PATH 缺工具为 PASS。
- 任一 preflight 命令修改文件、失败或逐 hunk review发现可复现的 repo 规范问题时，正式 upstream PR 创建闸门保持关闭。PR_SUBMIT 不自行实现代码修复；发送带 `phase=PREFLIGHT` 的 `[PR_CI_IMPL_REQUEST]`，允许 `upstream_pr=NOT_CREATED`，由共享 impl 完成 bounded 修复后必须在新 head 重新运行完整 manifest。只有 manifest 所有可本地项 PASS、不可本地项均有精确环境原因且无 actionable hygiene finding时才允许请求 upstream write lock。
- 不把 upstream CI pending、尚未由 maintainer triage、缺 label、无 reviewer 或普通 branch 落后本身判为 outlier。不得提交人类 review/comment、@maintainer、请求 review、添加标签或以任何方式催促维护者；仅当仓库文档明确要求且当前 CI recovery 必需时，允许一次精确机器 bot command/ack，并须写后只读核对。
- upstream PR create/ready/title/body/bot-command/rerun 等写入前必须持有 MAIN 明确授予的 account-level `UPSTREAM_WRITE_LOCK`；只读 fetch/search/readback、fork branch push和本地检查不需要该 lock。同一 lease最多持有一个 lock，完成写后 readback或失败 readback后立即释放。未获 lock 时保持已准备正文并等待，不得自行重试、换 API、空提交或与其他 lease 并发写 upstream。

### 润色与 upstream 更新

- 可直接修正同一 PR 的提交机械问题：PR title/body/template/release-note/changelog/拼写、repo formatter 产生的纯规范变化、生成文件同步，以及不改变已声明功能与结果的构建元数据。多行外部正文严格使用唯一 `/tmp` body file、`apply_patch`、目标/非空检查和写后只读复核。
- 必须先 `fetch` fork 与官方 upstream。若最新 upstream 无冲突，保持原 feature head，不为形式上的落后制造无意义 merge/rebase。若确有冲突，在该 PR 专属 isolated linked worktree 中解决，只允许保留原功能并适配当前 upstream API/文件布局；使用普通、可追踪的新提交和 non-force push。不得在历史 worktree、主工作树或另一 PR 的 worktree 中编辑。
- 专属 worktree 必须位于 `/home/lcpu/12921178/code/B300-pr/pr-submit-worktrees/<submit_id>` 并由目标 repo 注册，不得复用现有路径。terminal 前若该 worktree clean、所有新增提交已从 fork readback、且没有唯一未持久化证据，可执行一次不带 `--force` 的 `git worktree remove <exact-path>` 并只读确认注册项消失；普通移除失败或存在任何 dirty/untracked 唯一内容时原样保留并在 terminal 中报告，不得 force、clean、reset 或逐文件删除。此权限只适用于该 agent 自己创建的 PR submission 临时 worktree，不适用于历史 cycle worktree。
- 禁止 amend、force-push、改写原提交、直接 push 官方 upstream 或默认分支。若 repo 强制线性/DCO 历史而原分支不能无 force 修复，可从最新 upstream 创建唯一 `pr-submit/<submit_id>` 分支，按原顺序 cherry-pick 原功能提交并补充机械修复，普通 push 到用户 fork；必须保留原作者身份，不得伪造他人的 `Signed-off-by`、CLA、copyright 或 attestations。
- `PR_SUBMIT` worker 自身运行上述完整 CPU/static preflight manifest和 `git diff --check`，并只读审查 exact diff 的 repo 规范符合性。任何会编译 CUDA、初始化 GPU、执行 correctness/performance workload 或需要 Slurm 的命令仍禁止；原 fork PR 结果仍是技术证据。若 preflight/conflict/CI/bot recovery需要具体代码、build或test实现，向 MAIN 交付 repair packet，由共享 `PR_CI_IMPL` 完成；若修复必须重新设计或重建原始 benchmark/correctness 结论，标记 `OUTLIER`。

### Upstream ready PR

- 创建前只读检查官方 repo 默认/base branch、现有所有状态的 upstream PR 与 commit，避免重复。若同一 fork head 已有 OPEN upstream PR，幂等复用：确认 title/body/base/head；若它是 draft 且其余检查通过，可将这个既有 upstream PR 标为 ready。若同一功能已由其他 head 合并或被活跃 upstream PR 实质覆盖，本 fork draft 标记 `OUTLIER`，不得创建竞争 PR。
- 需要新建时，从用户 fork 的精确 head（原分支或上述唯一 submission branch）向官方 repo/base 创建 exactly one OPEN non-draft PR。标题和正文必须符合 repo 模板，保留 fork PR 已声明的事实、限制和测试/benchmark 结果，并链接原 fork issue/PR；不得夸大、重新解释或隐去原有限制。不得创建 draft upstream PR。
- 创建或 ready 转换后，只读复核官方 owner/repo、PR number/URL、OPEN、`isDraft=false`、base、fork owner/head branch/head OID、title/body 和原 fork PR 仍 OPEN+DRAFT。成功后发送 `[UPSTREAM_PR_READY]` 给 MAIN，附外部写入收据和 `lease_release=true`，然后结束当前 turn。后续 CI audit 可由任一 idle `PR_SUBMIT` worker 获取新 lease。不得 merge/close、删除分支、关闭 issue、提交人类 comment/review、请求 reviewer、改 labels/milestone/project。

### CI 与 bot recovery

- `[PR_RETRO_PREFLIGHT_START]` 是用户明确要求的强制提交后复核，适用于已经存在的 upstream OPEN PR，不受旧 `EXTERNAL_WAIT` fingerprint 不变规则限制。worker 必须在 fresh registered worktree 对 exact upstream/fork head 重跑与提交前完全相同的 manifest和逐 hunk repo 规范审查，同时读取当前 CI/bot 事实。无 finding 时发送 `[PR_RETRO_PREFLIGHT_PASS]`，包含 `retro_pass_head`、完整 manifest、不可本地项、当前 CI fingerprint、恢复的原 CI 状态和 `lease_release=true`；发现问题则发送 `[PR_CI_IMPL_REQUEST]` 并在 repair 后对新 head重复 retro-preflight。普通 CI green、bot success或旧 pre-commit 声明都不能替代这一步。

- 每次 `[PR_CI_AUDIT_START]` 先核对 packet 中的前一 event fingerprint，再只读获取当前官方 PR head/base/mergeability、branch protection/rulesets 可见信息、check suites/runs/jobs/log annotations、status contexts、review threads，以及 GitHub App/bot comments。记录 check 名称、required 与否、状态、run/job URL、失败步骤/日志片段和 comment/thread URL，并生成由 head OID、check suite/run 状态、最新 bot comment/thread ID 和 mergeability 构成的新 fingerprint；不得只看聚合红绿状态，也不得把 human review 意见纳入本合同。
- 分类只能是：`ACTIONABLE_OWNER`（正文、模板、bot command、metadata 或无代码设置问题，当前 submit worker 可直接恢复）、`ACTIONABLE_IMPL`（代码、build、test、formatter 或 conflict 修复，必须交共享 `PR_CI_IMPL`）、`FLAKY_INFRA`（日志证明与 patch 无关且 repo 允许 rerun）、`EXTERNAL_WAIT`（需要 maintainer workflow approval/review/secret/硬件/外部服务）、`NON_REQUIRED_ADVISORY`（非 required 且不影响提交）、`OUTLIER`（有证据表明必须重新设计或重建原技术结论）。不得把 CodeRabbit/Claude 等 bot 的推测性建议自动当成事实；只有具体到文件/行、可复现规则或 required CI 失败才是 actionable。
- `ACTIONABLE_OWNER` 可按原外部写入合同更新 fork/upstream PR metadata，或发送仓库文档明确要求的精确 bot command/ack。`FLAKY_INFRA` 只能使用官方提供且当前身份有权限的 rerun API，一次一项；不得靠空 commit、重复 comment 或改代码触发。所有写入均先查重、exactly once、写后 readback。
- `ACTIONABLE_IMPL` 时 worker 形成 `[PR_CI_IMPL_REQUEST]`，至少包含稳定 `repair_id=<submit_id>:repair-<n>`、精确 fork/upstream PR、branch/head、失败 check/job/log/comment URL、复现命令、允许文件/修复范围、禁止项和验收条件；把 packet 交给 MAIN，附 `lease_release=true` 并结束 turn。MAIN 负责在共享 impl worker 空闲时派发；submit worker 与 impl 不得同时持有同一 `submit_id` 的 lease。
- impl 完成后，MAIN 会把 ledger 中的 old/new head、提交/文件范围和 targeted validation 交给任一 idle submit worker建立 follow-up audit或retro-preflight lease。该 worker核对普通 non-force fork push和精确事实；若来源是 preflight/retro-preflight finding，必须在 fresh worktree 重跑完整 manifest和逐 hunk规范审查，不能只读聚合 CI。必要时可在获得 upstream write lock 后更新 PR body中的提交事实或发送 repo 明文要求的一条精确 bot recovery ack；不要求是提出 repair 的原 worker，也不得替换原性能/correctness技术结论。
- 当前 required checks 全部 terminal-success/skipped-by-policy、mergeable 且无 actionable bot thread时发送 `[PR_CI_STABLE]`；只剩 maintainer-controlled approval/review/secret/硬件/外部服务时发送 `[PR_CI_EXTERNAL_WAIT]`，列出精确外部条件、event fingerprint和可机械判断的 wake condition；需要重新设计/重建原始结果时发送 `[PR_CI_OUTLIER]`。三者都附 `lease_release=true` 并结束当前 turn。`EXTERNAL_WAIT` 未发生 fingerprint 变化时不得由 worker自行重审。

## PR_CI_IMPL 合同

- 只接受 MAIN 租给共享 impl pool 的一个 `[PR_CI_IMPL_START]`，核对 `lease_id`、`repair_id`、`submit_id` 和精确 old head，确认 `gpt-5.6-sol/xhigh`，完整读取根/本文件/repo 规则和消息中引用的 preflight/失败日志/comment。packet 可来自 upstream 创建前 preflight、已提交 retro-preflight 或 CI audit，`upstream_pr` 因此可以是 `NOT_CREATED`。不得重新评判原 fork PR 的既有结果，不得处理 packet 外 finding、PR、repo 或自行扩大 scope，也不得 spawn agent。
- 只能使用 `/home/lcpu/12921178/code/B300-pr/pr-submit-worktrees/<submit_id>-ci` 的 fresh registered isolated worktree，从消息精确 fork head 开始。实现最小修复，遵守 repo 风格；commit 不 amend、不改写历史，普通 non-force push 到同一用户 fork branch。不得更新 upstream PR metadata、回复 bot/human、push 官方 repo、merge/close、force-push或改默认分支。
- 运行与当前失败直接相关的最小 targeted validation。CPU 检查可在登录节点；任何 CUDA/GPU compile/test/profile 必须遵守 Slurm 边界并保留命令/job/log。不得重跑原 benchmark 矩阵，除非当前 required CI 失败本身就是该命令且 submit worker 的 repair packet 明确要求；若修复会使原性能/correctness 声明需要重新建立，停止并返回 `CI_IMPL_OUTLIER_RECOMMENDATION`。
- 完成后只向 MAIN 发送 `[PR_CI_IMPL_SUMMARY]`，包含 `lease_id`、`repair_id`、old/new head、commit、changed files、targeted validation、push readback、worktree disposition和 `lease_release=true`，然后结束。失败只能是带证据的 `[PR_CI_IMPL_BLOCKED]` 或 `[PR_CI_IMPL_OUTLIER_RECOMMENDATION]`，也必须释放 lease；不得自行关闭或降级 upstream PR。worker 后续可以获取另一个 repair lease，但必须完整重读新 packet并清除上一 lease 的临时状态。

### OUTLIER 与 blocker

- 只有以下情况可发 `[PR_SUBMIT_OUTLIER]`：功能已被 upstream 合并或等价 PR 占用；upstream 巨变使原 patch 语义过时；贡献政策明确不接受该 scope；冲突/规范修复需要重新设计、扩大功能、重新跑技术验证或无法在不伪造 DCO/CLA/作者 attestation 的前提下完成；fork head/提交已经不可恢复。OUTLIER 必须提供可核验的 upstream commit/PR/policy/conflict证据和已尝试的有界修复，不得只说“太旧”或“冲突太多”。
- 认证、GitHub/网络临时失败、权限不足、官方 target 不确定或用户身份/legal attestation 必须选择但无法安全推断时发送 `[PR_SUBMIT_BLOCKER]`，不算 OUTLIER。消息持久化 blocker 和精确 resume condition，附 `lease_release=true` 后结束；MAIN 解决后可把同一 `submit_id` 作为新 attempt 租给任一 idle worker，不得在旧 lease 换 PR。
- 所有消息包含 `message_id`、`lease_id`、`submit_id`、attempt、repo、role、phase、fork/upstream PR、upstream repo/base、branch/head、前后 event fingerprint、外部写入收据、`in_reply_to` 和 `action_required`。`UPSTREAM_PR_READY`、`PR_CI_STABLE`、`PR_CI_EXTERNAL_WAIT`、`PR_SUBMIT_OUTLIER|PR_CI_OUTLIER|PR_SUBMIT_BLOCKER` 是阶段 terminal；发出时必须 `lease_release=true` 并立即结束当前 turn。worker 可以在后续 turn 获取另一个 PR 的新 lease，但在任意一个 turn 内严格只处理一个 PR。

## EXPLORE 合同

- 每轮收到 `EXPLORE_START` 后，完整读取根规则、repo 规则、贡献/benchmark 文档，刷新 upstream 并检查开放 issue/PR、相关代码、已有 fork PR 和前序 cycle，避免重复或依赖未合并分支。
- FLA EXPLORE 还必须检查另一 lane 当前 issue/候选 scope、预计 tracked files、benchmark 目标和 open branch/PR；发现重叠时排除候选或向 MAIN/REVIEW 报告，不能把冲突留给两个 IMPL 并行解决。
- 保持只读：不得创建/更新 GitHub 内容、分支、提交、push、Slurm 作业或源码。候选必须能由后续 IMPL 在 B300 Slurm 中独立验证。
- 每轮向配对 REVIEW 提交一个候选集，通常包含 1-2 个互相独立的深度候选。每个候选至少说明问题、现有代码/历史 benchmark/profile 对瓶颈的支持、B300/SM103 价值、非平凡 kernel/dataflow/schedule 方法、重复工作检查、预计改动文件、正确性 oracle、representative operator benchmark、至少一个 integration/model 影响面、profile-first 停止条件、风险和为什么容易形成可合并 PR。
- 候选排序必须优先考虑“可写进简历的独立 kernel 技术成果”：目标应有可信路径在代表性 operator 上达到约 10% 以上，或在真实 integration/model 路径上达到约 5% 以上，同时能清楚讲明瓶颈、算法/数据流/调度方法、B300 资源机制和 upstream 用户影响。阈值是选题质量筛选，不是允许编造的性能声明；实际验收仍由 issue 的测量合同决定。
- 默认降低或拒绝 wrapper/dispatch-only、CLI/docs/test-only、单一小 shape 特化、只删少量同步/屏障、仅融合微型 epilogue、没有 Amdahl 上界或没有代表性端到端消费者的候选。只有 profile/覆盖面证据表明它具有广泛且显著价值时才可例外采用。小 diff 不是问题，但“小收益且无独立 kernel 方法”不再作为优先选题。
- 如果确实找不到候选，可以提交空候选集，但必须附搜索过的 kernel/模块/issue/PR 范围和逐类排除原因；单纯声明“没有想法”不是有效 round。
- 消息使用当前 `cycle_id` 和 `selection_id=<repo>:<cycle_id>:round-1|round-2`。发送 `[EXPLORE_CANDIDATES]` 给 REVIEW、向 MAIN 发送 metadata-only `[EXPLORE_NOTICE]` 后必须真正结束 turn。
- 第一轮收到整组 `[SELECTION_ALL_REJECT]` 后，第二轮必须探索新的候选或实质不同的问题方向，不得只重命名或轻微改写第一轮候选。第二轮发送后再次结束 turn。

## SELECTION_REVIEW 合同

- selection reviewer 必须使用以下 prompt 意图执行快速审核：`只选择非重复、有 profile/Amdahl 支撑、能形成独立 kernel/dataflow/schedule 技术成果且具有代表性 operator 与 integration/model 价值的 B300 方向；优先可信达到约 10% operator 或约 5% 真实集成收益的候选。默认拒绝 wrapper/dispatch-only、CLI/docs/test-only、单小 shape、微型 epilogue 和缺少端到端消费者的工作。scope 仍须单问题、可复现、可正确验证并适合 upstream；不要做实现级 code review，不要求未来泛化，不扩大问题。每轮最多选一个。`
- 对每个候选给出 `ACCEPT`、`REVISE` 或 `REJECT`。`REVISE` 不退回 EXPLORE：reviewer 直接修正问题陈述、scope、正确性 oracle、benchmark、验收条件和 issue 文案，然后作为该轮选定候选继续。修订不得换成无关问题、扩大到更多架构/模块或编造性能证据；无法安全修订则 REJECT。
- 每轮最多选定一个候选。选定后，reviewer 在完整核对 `origin owner/repo` 和 upstream 重复状态后，只在用户 fork 创建一个 issue；issue 必须明确它是待验证假设，不把未测性能写成事实。
- FLA selection reviewer 在采用候选前还必须核对另一 lane 的 active issue、branch、PR、tracked-file scope 和 benchmark surface。存在实质重叠时不得创建第二个 issue；应 REJECT、在不改变问题的前提下安全收窄，或发送 lane conflict blocker 给 MAIN。
- 创建 issue 是 selection 阶段的唯一写权限。不得修改源码、创建 branch/PR、push、提交 Slurm、更新其他 issue/PR 或接触 upstream。fork Issues 未启用、认证失败或目标不确定时发送 `[SELECTION_BLOCKER]` 给 MAIN，不得计为 REJECT。
- issue 创建成功后，直接向 IMPL 发送 `[ISSUE_HANDOFF]`，其中包含 issue URL、最终规范化 scope、baseline/profile-first 要求、正确性门槛、representative operator 与 integration/model 性能门槛、最多两次 reviewer-approved depth iteration 和明确禁止项；向 MAIN 发送 metadata-only `[SELECTION_NOTICE]`，然后结束 turn。MAIN 负责激活 IMPL，不重新审批候选。
- 只有整组候选逐项审核后均 REJECT，且没有任何候选可安全 REVISE 时，才能发送 `SELECTION_ALL_REJECT`。空候选集只有在 reviewer 验证搜索覆盖和排除证据充分后才能得到相同结果。round-1 时直接通知 EXPLORE 并通知 MAIN 激活 round-2；round-2 再次 ALL_REJECT 时发送 `[PRIMARY_EXPLORATION_FINISHED]` 给 MAIN，不创建 issue、不唤醒 IMPL。MAIN 只有在 salvage queue 为空或已耗尽时才记录最终 `REPO_FINISHED`。
- 认证、权限、环境、并发、消息失败或 upstream 暂时不可达属于 BLOCKED，不属于 ALL_REJECT。任一候选被选定后 reject 计数立即清零；普通 fresh cycle 从 round-1 开始，salvage cycle 不使用 selection round。

## PERFORMANCE_DEPTH_REVIEW 合同

- 适用条件仅为：baseline 和主要 correctness oracle 已通过，候选实现可执行，但 representative operator 或 issue 指定的 integration/model 性能门槛未达到。IMPL 此时不得直接记录 performance no-go；应把完整 `[DEPTH_REVIEW_DOSSIER]` 直接发给固定 REVIEW，并向 MAIN 发送不含技术正文的 `[DEPTH_REVIEW_NOTICE]` 后结束 turn。
- dossier 必须同时列出全部目标与 fallback/control shape 的分布统计、Amdahl 上界、torch-profiler/NCU/Nsys 等已有机制证据、correctness 覆盖、资源和数值边界、已经尝试及明确未尝试的方法、反对继续的证据，以及同一 issue/module 内至多两个可执行的下一步。不得只展示最佳 shape、隐藏回归或用未测推测替代证据。
- REVIEW 独立只读复核后，结果只能是 `DEEPEN` 或 `STOP`。`DEEPEN` 必须指出一个 bounded 的 kernel/dataflow/schedule 方向、为什么现有证据仍支持达到原门槛，以及本轮明确的 profile/correctness/performance 停止条件；不得放宽门槛、换问题、增加架构、扩到无关模块或要求通用重构。`STOP` 必须说明机制上界、回归或剩余方案为何不足以支持继续。
- 每个 cycle 的 `depth_iteration` 初始为 0，最多接受两次 `DEEPEN`。REVIEW 不能批准第三次；两次均未达标时必须 `STOP`。同一轮 `DEEPEN` 可包含一个内聚的生产迭代和按风险分阶段的验证，但 IMPL 仍按原权限合同向 MAIN 申请源码、Slurm 和外部写入，不因 reviewer 技术批准而获得机械权限。
- REVIEW 把完整 `[DEPTH_REVIEW_RESULT]` 直接发给 IMPL，向 MAIN 只发 `[DEPTH_REVIEW_SUMMARY]`，包含 `depth_iteration`、`DEEPEN|STOP` 和 message ID。`DEEPEN` 后 IMPL 在同一 issue、分支和 worktree 内继续；`STOP` 或深度预算耗尽后，IMPL 才能按证据记录 performance `PR_NO_GO_RECORDED`。
- correctness red、baseline 不可复现、编译/OOR、环境/权限故障、upstream 重复或候选失效不进入本合同；它们按 correctness no-go、BLOCKED 或 PR_ABORTED 处理。performance depth review 优先于新的 selection，但低于已打开的 implementation findings delivery/rereview。

## SALVAGE_REVIEW 与二次开发

- `PR_NO_GO_RECORDED` 不自动表示该技术方向永久无价值。只有同时具备以下证据的 near-miss，IMPL 才能把完整 dossier 直接发给固定 REVIEW 作为 `[SALVAGE_CANDIDATE]`，并向 MAIN 发送 metadata-only `[SALVAGE_NOTICE]`：核心机制在目标路径有稳定且达到原门槛的收益；主要 correctness oracle 已通过；失败回归已定位并可复现；存在一个不放宽 oracle/性能门槛、不扩大架构/模块/问题定义的有限替代方案；来源 cycle 不是 salvage cycle。
- correctness 失败需要改变数值语义、收益未达到原门槛、修复依赖新增大范围 fallback/抽象、已经耗尽 bounded fix、或必须扩大原问题时，不得包装成 salvage candidate。IMPL 在 dossier 中必须同时给出支持和反对继续的证据，不得只报告最佳数字。
- REVIEW 使用 `gpt-5.6-sol/max` 做独立、只读的快速 `SALVAGE_REVIEW`，只判断该 near-miss 是否值得获得一次新的独立 cycle，不做代码 review。结果只能是：`SALVAGE_NOW`（证据和有限替代方案均明确，优先下一 cycle）、`SALVAGE_LATER`（机制有价值但当前修复边界不足，排入 primary exploration 之后）或 `DISCARD`（不值得再次尝试）。完整结果直接发给 IMPL，向 MAIN 只发 `[SALVAGE_REVIEW_NOTICE]` 及机械字段。
- queue 条目使用稳定 `salvage_id=<repo>:source-cycle-<n>:<slug>`，记录来源 issue/result、分类、review message ID、状态和 attempt cycle；技术正文留在直接消息与来源 fork issue，不写入 `WORKSTREAMS.md`。
- 执行 queue 条目时，MAIN 分配新的 `cycle_id` 并激活同一 REVIEW 进入 `SALVAGE_SELECTION`。Reviewer 复核 upstream 是否仍未重复，把有限替代方案规范化为一个独立问题，只在 fork 创建一个新 issue，链接来源 no-go issue/result，然后直接 `[ISSUE_HANDOFF]` 给 IMPL。若此时已失效或无法形成安全 bounded issue，标记 `DISCARD`，不得创建 issue。
- salvage IMPL 必须从最新 upstream 默认分支创建新的唯一分支，不得 checkout、续写、rebase 或叠加来源 branch。后续实现、draft PR 和 max implementation review 与普通 cycle 相同，但每个 `salvage_id` 最多一次 implementation attempt；其 terminal 不得再产生 salvage candidate。
- `SALVAGE_NOW` 在当前 active cycle 结束后优先于新的 EXPLORE；`SALVAGE_LATER` 在两轮 fresh selection 均 ALL_REJECT、进入 `PRIMARY_EXPLORATION_FINISHED` 后按队列顺序处理。只有 primary exploration 已耗尽且 queue 为空、全部丢弃或完成唯一 attempt 时，MAIN 才机械记录最终 `REPO_FINISHED`。

## REVIEW 技术合同

- 固定 REVIEW 在 draft PR 前执行 performance depth review，在 draft PR 后执行 implementation review。两者都只审查派工指定的 repo、baseline、issue、功能分支和精确 HEAD；implementation review 使用独立 `review_id`。每次开始前重新读取根规则、repo 规则、贡献文档及相关测试/benchmark 规范。
- REVIEW 在 selection、performance depth、salvage、实现分析、findings 投递和 rereview 阶段严格只读。不得编辑文件、commit、push、创建/更新 GitHub 内容、向 upstream 交互或提交 Slurm 作业；selection 阶段创建 issue 的例外不得带入其他 review。唯一额外写权限是 MAIN 在按需维护窗口显式派发的 `[WORKTREE_CLEANUP]`，且只能按下述合同移除该 cycle 的 linked worktree。
- review 必须检查行为正确性、架构/shape 边界、回归风险、测试有效性、benchmark 公平性与可复现性、scope 和 PR 声明是否一致。不得只做风格总结。
- finding 按 `BLOCKER`、`MAJOR`、`MINOR` 排序，并包含文件/行、具体失败模式、触发条件、证据和最小修复方向。没有可行动问题时明确报告 `NO FINDINGS`，同时列出仍未覆盖的验证风险。
- 把个人偏好、未来泛化和可选增强明确标为 `OPTIONAL`，不得包装成 blocker。没有实际失败模式或测量证据时，不得要求新抽象、扩大架构支持、通用 fallback、额外防御分支或无关重构。
- 特别警惕过度设计和过度 fallback：B300 精确 guard、单问题 scope 和已经测量过的边界不应仅为“更通用”而扩张；fallback 只有在存在可复现的正确性、兼容性或性能风险时才可作为 finding。
- reviewer 只把完整报告直接发给配对的实现 subagent，不在 GitHub 留 review/comment；向主代理只发投递回执和最终状态摘要。收到实现 subagent 的 `ACCEPT/PARTIAL/REJECT` 判定与修复结果后，对同一 HEAD 做复核，并明确每项 finding 是 resolved、withdrawn 还是仍有证据支持。

## REVIEW 的按需 worktree 删除前检查合同

- cleanup 是可延后的带外机械维护，不是 cycle 状态机或新的技术 review，也不得阻塞下一 cycle。MAIN 不会在每个 terminal 后自动触发；只有磁盘压力、用户明确要求或计划性维护且当前没有更高优先级技术 review 时才执行。只有收到 MAIN 的 `[WORKTREE_CLEANUP]`，且消息明确 repo、cycle、terminal、功能分支和候选 linked-worktree 绝对路径时才能行动；不得自行扫描后批量删除，也不得处理其他 repo/cycle。
- 先用 `git worktree list --porcelain` 确认目标是当前 repo 已注册的 linked worktree，并确认它不是主仓库工作树、不是当前任何 active agent 的工作目录、对应 cycle 已 terminal、IMPL 已结束、HEAD/branch 与 terminal 记录一致。
- 删除前必须只读检查 index、tracked worktree、untracked/ignored 文件和证据路径。只要存在未提交 tracked 改动、用户文件、未归档的唯一 benchmark/Slurm/profile 证据、路径不一致或无法证明可丢弃的产物，就发送 `[WORKTREE_CLEANUP_BLOCKER]` 并原样保留；不得 stash、reset、clean、checkout 覆盖或猜测所有权。
- 满足条件时先执行不带 `--force` 的 `git worktree remove <exact-absolute-path>`，随后仅用 `git worktree list --porcelain` 验证该注册项消失；不得删除功能分支、remote ref、issue、PR、提交、主工作树、共享 venv/cache 或证据目录，不得运行 `git worktree prune` 作为批量清理替代。
- 若普通 remove 仅因已确认可丢弃的 ignored/untracked 生成物而拒绝，停止并向 MAIN 报告精确文件清单；没有新的显式批准不得使用 `--force` 或文件级删除。
- 对包含未提交 tracked prototype 的 worktree，首次检查必须 `PRESERVED_BLOCKED`。MAIN 可另行激活 IMPL，把当前 base commit、完整 tracked diff 或逐文件 snapshot、所有相关 untracked source/benchmark 和复现说明持久化到 cleanup 目标之外；不得用 hash 代替内容归档，也不得由 IMPL reset/clean/remove worktree。归档后 REVIEW 再次只读核对：每个 dirty path 都必须属于该 cycle 的 issue scope，snapshot 可重建且路径/字节数有记录，不含用户或跨 cycle 文件，并且持久日志/结果仍存在。只有随后收到 MAIN 带 exact path、inventory 和 archive 的 `[WORKTREE_CLEANUP_FORCE]`，REVIEW 才能执行一次 `git worktree remove --force <exact-absolute-path>`；任一条件不成立继续 `PRESERVED_BLOCKED`。
- cleanup 完成后发送 `[WORKTREE_CLEANUP_RESULT]`，结果只能是 `REMOVED`、`NOT_CREATED` 或 `PRESERVED_BLOCKED`，并附目标路径、是否使用经授权的 force 和只读验证。三种结果都只是维护记录；`PRESERVED_BLOCKED` 不要求立即归档或重试，也不阻塞任何 repo/lane 调度。

## 实现 subagent 处理 review

- 收到专用 reviewer comment 后逐条标记 `ACCEPT`、`PARTIAL` 或 `REJECT`，结合原始 scope、代码事实、仓库既有模式、B300 数据和测试证据独立判断，不因 reviewer 使用 `max` 而默认其结论正确。
- `ACCEPT` 表示失败模式成立并按最小方案修复；`PARTIAL` 表示风险部分成立但建议方案过度，只实现必要部分；`REJECT` 表示 comment 无事实依据、超出 scope、要求过度设计/过度 fallback 或会损害已验证行为，并提供具体反证。
- 只修改合理部分。禁止为了让 review 数量清零而增加无测量支持的 fallback、抽象层、架构泛化、配置开关或无关测试矩阵。
- 修复留在同一功能分支，使用新的清晰提交和普通 push，不 amend/force-push。按风险通过 Slurm/CPU 重新验证，更新 fork draft PR，并把逐条判定、commit、测试与证据直接发给同一 reviewer；向主代理只发投递回执和阶段状态。
- 若 `BLOCKER`/`MAJOR` 仍有分歧，停止相关发布动作并等待 reviewer 复核或用户裁决；不得由主代理代替技术审查。

## 开始前必须完成

1. 完整阅读根目录 `AGENTS.md`、本文件以及 repo 内的 `AGENTS.md`、`CLAUDE.md`、`CONTRIBUTING.md` 和相关开发/benchmark 文档。
2. 检查 `git status --short --branch`、默认分支及 remote URL，确认工作树干净，`origin` 是用户 fork，`upstream` 是官方仓库。
   FLA IMPL 还必须确认当前绝对路径等于 MAIN 派发并在 `git worktree list --porcelain` 注册的本 lane linked worktree；除 cycle 11 迁移例外外，路径若是主仓库工作树或属于另一 lane，必须停止。
3. 只 fetch `upstream` 和 `origin`，确认基线 commit，并比较双方默认分支。若 fork 落后或发生分叉，报告主代理处理；subagent 不得自行同步默认分支。不得 pull 合并未知改动，不得改写现有分支。
4. 搜索现有 issue、PR、代码、测试和 benchmark，避免重复已有工作；不得在未授权情况下给 upstream 留言或创建任何内容。
5. EXPLORE 将候选直接提交给 REVIEW；selection reviewer 决定并创建 issue；IMPL 只执行 `ISSUE_HANDOFF` 的最终 scope。MAIN 不接收或转述候选/selection 技术正文。

发现脏工作树、remote 异常、凭据问题、已有同名分支/PR、无法访问 B300 或规则冲突时，停止外部写入并报告。不要使用 `git reset --hard`、`git clean -fd`、`git checkout -- <path>`、强推或其他可能覆盖用户数据的命令。

## IMPL 收到 ISSUE_HANDOFF 后的流程

### 1. 建立可追踪工作项

- 核验 selection reviewer 已在正确的用户 fork 创建唯一 issue，且 handoff 中的 issue URL、最终 scope、baseline 要求和禁止项一致；不一致时停止并报告，不得自行重写 selection。
- baseline 调查发现原 scope 不可行时，使用 `[SCOPE_BLOCKER]` 交回 selection reviewer。只允许在同一问题内收窄；无法收窄则记录 `PR_ABORTED`，不得把 issue 扩大成第二个问题。
- 从最新 `upstream` 默认分支创建唯一功能分支，推荐 `b300/<issue-slug>`。
- FLA IMPL 必须先由 MAIN 确认本 lane 的唯一 linked-worktree 绝对路径，再在该 worktree 内建立分支。两个 lane 的分支名、路径、ignored profile/evidence 子目录和 Slurm wrapper 名称都必须包含可区分的 cycle/lane 标识；不得依赖共享 checkout 的当前分支。
- 只把该分支推送到 `origin`。在执行 push 前再次核对目标 remote 和完整 refspec。
- 形成首个可审阅提交后，在用户 fork 创建 draft PR，base 为 fork 默认分支，head 为该功能分支，并链接 fork issue。

### 2. 实现

- 先建立能失败或能显示性能瓶颈的最小复现，再修改实现。
- 沿用仓库现有抽象、代码风格、架构 dispatch 和测试模式。
- 若引入 B300 专属路径，必须有明确的 SM103 guard；若改动通用路径，必须验证其他架构的编译与行为不被无意改变。
- 避免无依据的 magic number。调参范围、选择理由和不适用场景必须可解释。
- 每个提交保持单一意图并能够 review；提交消息遵守 repo 约定。

### 3. Slurm 验证

- 登录节点只做阅读、编辑、Git、CPU 静态检查和 `sbatch`/`srun` 提交。
- CUDA/Triton/CUTLASS 等 GPU 相关编译、所有 GPU 测试、profiling 和 benchmark 必须在 Slurm allocation/job 内执行。
- 优先复用仓库或工作区已有 Slurm 脚本。不得猜测或硬编码未经确认的 account、partition、QoS、节点名或环境模块。
- 首先记录基线，再记录修改后的结果。baseline 与候选版本使用相同节点、软件环境、输入、warmup、重复次数和统计方法；条件允许时在同一 allocation 内交错运行。
- 保存完整命令、job ID、已有 Git commit ID、GPU/driver/CUDA/PyTorch/Triton 等版本、关键环境变量、原始日志与结果文件路径。

正确性验证至少覆盖与改动相关的 dtype、典型与边界 shape、非对齐输入、数值容差，以及 repo 规定的回归测试。性能验证不得只选最有利 shape 或最佳一次运行；报告输入矩阵、warmup、重复次数、中位数和噪声/分位数，适用时补充吞吐、带宽与显存。

correctness 已通过但 representative 性能低于 issue 门槛时，IMPL 不得在第一次 schedule/实现尝试后直接放弃，也不得自行无限调参。先按 `PERFORMANCE_DEPTH_REVIEW` 交付完整 dossier；收到 `DEEPEN` 后仅执行 reviewer 指定的同问题 bounded 迭代，并保持原 oracle、benchmark 矩阵和门槛不变。收到 `STOP` 或两次 `DEEPEN` 均未达标后才结束性能方向。

### 4. 维护 fork draft PR

fork draft PR 是持续更新的工程记录，至少包含：

- 问题说明、scope 和 B300/SM103 适用条件；
- 设计选择、架构 guard 和未采用方案；
- correctness、lint、build 与 benchmark 的可复现命令；
- baseline/after 表格、原始结果位置和所有相关 Slurm job ID；
- 已知风险、其他架构影响、未完成检查和回滚方式。

失败结果也要如实记录。没有统计上可信的提升、正确性未通过或收益依赖不合理特化时，应把结论标为不建议 upstream，而不是扩大改动范围掩盖问题。

## Cycle 与 repo terminal

- 成功实现并完成 max implementation review 后，IMPL 向 MAIN 发送 `[PR_READY_FOR_USER]`；fork issue 与 draft PR 保持 OPEN/DRAFT，分支冻结等待用户处理。
- baseline 未复现或 correctness 门槛未通过时，可按证据记录 fork issue并发送 `[PR_NO_GO_RECORDED]`；performance 门槛未通过时必须先完成 `PERFORMANCE_DEPTH_REVIEW`，只有 reviewer 返回 `STOP` 或两次 `DEEPEN` 预算耗尽后才能发送 performance `[PR_NO_GO_RECORDED]`。不得为了形式完整创建空 draft PR。发现重复或候选失效时发送 `[PR_ABORTED]`。
- 三种 PR terminal 只结束当前 `cycle_id`。terminal 时应在 issue/PR 中记录原始 benchmark/Slurm/profile 证据路径，但允许唯一证据继续留在该 cycle 的历史 linked worktree；这会在未来真正删除前触发 `PRESERVED_BLOCKED`，届时再按删除前检查合同归档，无需为进入下一 cycle 提前搬运。MAIN 在 terminal 后可立即为同一 lane 分配使用全新 linked worktree 的下一 cycle；旧 worktree 必须保持只读且不得复用。若 IMPL 同时提出合格 `[SALVAGE_CANDIDATE]`，直接完成只读 salvage classification并由 MAIN 按 `NOW/LATER/DISCARD` 调度。IMPL 不得在 terminal 前开始本 lane 下一 cycle，也不得自行删除历史 worktree。
- selection round-1 的 ALL_REJECT 只触发 round-2；round-2 再次 ALL_REJECT 由 REVIEW 产生 `[PRIMARY_EXPLORATION_FINISHED]`，不是自动最终停止。只有 salvage queue 为空或全部完成/丢弃后，MAIN 才记录 `REPO_FINISHED`。两者都不得关闭或删除已有 issue、draft PR 或分支，EXPLORE/IMPL 不得自行宣布。
- salvage cycle 的 terminal 必须引用 `salvage_id` 和来源 issue；无论成功、no-go 或 aborted，都消耗该条目的唯一 attempt，并且禁止从该 terminal 再派生 salvage candidate。
- 权限、认证、Slurm、依赖、消息或并发故障只属于 BLOCKED；不能转换成 PR terminal、ALL_REJECT 或 REPO_FINISHED。

## 严格禁止

- 除用户明确启动且由 MAIN ledger 精确派发的 `PR_SUBMIT` 外，不得在 upstream 创建或更新 PR，也不得发任何 upstream 通知。`PR_SUBMIT` 只能从用户 fork head 创建/更新/ready 精确官方 PR，并仅在 repo 明文要求的 CI recovery 中发送一次精确机器 bot command/ack；仍不得直接 push 官方 repo、创建 issue、提交人类 review/comment、请求 reviewer或联系 maintainer。`PR_CI_IMPL` 只能普通 non-force push 同一用户 fork branch，不能写官方 repo/PR。
- 不得把 fork PR 的 base 指向 upstream；不得将 fork PR 标为 ready、merge、close，或删除远端分支。`PR_SUBMIT` 创建的是独立的官方 upstream non-draft PR，原 fork PR 必须保持 OPEN/DRAFT。
- 不得 force-push、改写共享历史、修改默认分支或直接 push `origin/main`。
- 不得在登录节点直接运行 GPU 编译、GPU 测试、profiling、benchmark 或任何会初始化 GPU 的脚本。
- 不得执行与任务无关或可以省略的 SHA、MD5、BLAKE、xxHash 等 hash/摘要计算或校验，不得运行 standalone checksum、hash benchmark、批量扫描、碰撞搜索、爆破、挖矿或自定义 CPU/GPU hash kernel。正常 Git 操作只读取已有 commit ID 作为版本引用。仓库原有流程或标准工具链在依赖安装、下载校验、编译、JIT/cache 和测试中不可避免的内部 hash 可以执行，但不得新增 hash 实现、扩大其计算规模、单独 benchmark 或重复运行非必要校验；无法判断必要性时以 `[NEEDS_MAIN]` 报告。
- 不得泄露 token、SSH key、内部路径中的敏感信息、集群凭据或私有日志。
- 不得提交与目标无关的格式化、依赖升级、生成物或 benchmark 数据清洗。

## 向主代理汇报

候选集交付、selection 结果、issue handoff、draft PR 建立、implementation review 和 cycle terminal 按各角色路由汇报。单独的 branch 建立、首次正确性、首次 benchmark、普通 job/commit 不通知 MAIN；它们在下一次必要消息或最终报告中合并。遇到需要 MAIN 处理的失败、权限或 blocker 时立即给出证据和下一步建议。

最终报告必须包含：

- repo、baseline commit ID、功能分支、fork issue 和 fork draft PR；
- 提交列表和变更文件；
- correctness/build/lint 的命令与结果；
- baseline/after 性能表、测量方法、Slurm job ID 和原始日志路径；
- 风险、限制、未执行项及原因；
- 明确结论：`建议整理后提交 upstream`、`需要继续验证` 或 `不建议提交 upstream`。

`PR_SUBMIT` 的阶段 terminal 报告改为只包含：`lease_id`、`submit_id`、attempt、fork/upstream PR、最终 fork head、已核对的 contributing 规则、实际润色/冲突/CI/bot recovery及新增提交、required checks 与 bot thread 的精确状态、前后 event fingerprint、外部写入收据、共享 `PR_CI_IMPL`/repair_id（如有）、外部等待项或带证据的 `OUTLIER` 原因，以及 `lease_release=true`。不得重新总结或裁决原技术结果。

在主代理和用户完成最终 review 前，保持 draft PR 未合并、issue 开启、分支可复现。
