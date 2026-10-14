# open-tag vs Raft（raft-source）代码级深度对比

> 基于对同级 `raft-source` 仓库（Raft 平台，Botiverse, Inc. 的 source-available 发布镜像）的源码逐模块分析，2026-10-14。
> Raft 即 open-tag README 对比表中的 "Slock / Raft" 竞品。行号均引自 raft-source 当前树。

## 0. 结论先行

同一三层骨架（server ↔ WebSocket ↔ daemon ↔ CLI ↔ agent 子进程），但复杂度押注完全不同：

- **open-tag** 把复杂度押在**回复授权协议**（服务器硬约束协作秩序：Turn 聚合、grant、两阶段准入、FIFO）。
- **Raft** 押在**投递可靠性 + 凭据代理 + 工程基建**（协作秩序交给 prompt 自律 + task claim 锁 + freshness hold）。
- open-tag 全面领先：资源压力管控（Job Object / cgroup）、协议化回复协调、单文件纪律。
- Raft 全面领先：凭据安全模型（agent 零持有）、多副本同步、记忆注入/迁移、产品工程基建。

## 1. 许可 / 开放性（最本质）

| | open-tag | Raft |
|---|---|---|
| 许可 | **Apache-2.0 真开源** | **FSL-1.1-ALv2 source-available**：可读/可跑/可改，但**禁止用于与 Raft 竞争**；发布 2 年后转 Apache |
| 协作 | 接受 PR，社区驱动 | 私有仓库的发布镜像，外部 PR 自动关闭；商标归 Botiverse, Inc. |

## 2. 代码形态与规模

| | open-tag | raft-source |
|---|---|---|
| 结构 | 单仓：`src/`（server+daemon+cli+db）+ `web/` + `packages/daemon` 发布包 | pnpm monorepo **15 个包**（server / daemon / cli / web / computer / raft-sdk / sync-core / trace-client / trace-upload-worker / raft-event-buffer / desktop-contract / visual-testing / ui-inventory …） |
| 生产 TS | ~2.2 万行 | **~54 万行**（含测试逾 100 万行） |
| 单文件纪律 | **≤1000 行硬规则**（core-beliefs #8） | 无此约束：`agentOrchestrator.ts` **16,479 行**、`messageService.ts` 9,139 行、`agentProcessManager.ts` 10,968 行 |
| DB schema | `src/db/schema.ts` **384 行** | `packages/server/src/db/schema.ts` **7,612 行 / 229 张表** |
| 运行时要求 | Node 20+，npm scripts | Node 24 + pnpm 12（corepack 切换有坑，README 有排错表） |

## 3. 鉴权与凭据（架构级差异）

### open-tag（`src/server/auth.ts`）
- 三平面：人类 JWT 30 天 + `x-server-id`；agent `sk_agent_*`（daemon 注入 env，DB 存 **SHA-256 hash** + `x-agent-id`）；daemon WS `?key=` machine key。
- **agent 进程能直接看到自己的 token**（靠 standing prompt 要求脱敏兜底）。
- 角色 capability（8 项）+ agent scopes（14 项）两套权限体系。

### Raft（更强一层：agent 永不持有服务器凭据）
- **`agentCredentialProxy.ts`（2,206 行）**：daemon 持有 `sk_agent_*`（**argon2id** 哈希，撤销行不参与校验），agent 只拿 loopback `127.0.0.1` 的 per-launch 代理 token（`sap_*`）。代理做 **origin 钉死**（防绝对 URL/反斜杠换源当 SSRF 跳板 → 403 `agent_proxy_origin_mismatch`）+ 凭据换头；token 走 0600 文件不进 spawn env（`cliTransport.ts:858-893`）。
- 人类：15min access + 30d refresh + `session_families`；**撤销 session 即踢实时连接**（`socket/accessRevocation.ts`，跨副本 `fanoutWithAck`，握手中途竞态也补检）。
- 可见性收敛**单一函数** `canUserAccessChannel`（`channelService.ts:4748`），HTTP / realtime join / sync replay 三处共用。
- 凭据前缀是严格主体命名空间（`sk_machine_` / `sk_computer_` / `sk_agent_` / `sap_` / JWT），错主体 → 401 `invalid_principal`；未注册路径 fail-closed（`authFromRegistry.ts`）。
- agent scopes 19 项，服务端权威校验。

**评价**：open-tag 的 SHA-256（无 salt）弱于 argon2id；更关键的是凭据暴露面——Raft 的"代理换凭据"模式使被注入/被攻破的 agent 进程拿不到任何可外传的服务器凭据，这是真正的零信任边界。

## 4. 消息管线与回复协调（设计哲学分叉点）

### open-tag：协议化授权
- `conversationTurnPolicy`：突发消息按 `(server, channel, sender)` 聚合成 Turn，边界即刻派发，capability 预检后并发 fan-out。
- `replyCoordinationPolicy`：**观察 ≠ 回复权**（服务器强制）——首个显式 @ 为 primary、后续 @ 各得 directed grant、无提及者竞争至多一个 supplemental；部分唯一索引强制一 primary + 一 supplemental；`(replyToMessageId, senderId)` 一次触发一发；发前有 bounded settle 窗口、可 transfer/handoff。
- `agentDeliveryAdmission`：两阶段准入（`deliver:pending → ready → admitted → ACK/NACK`），严格 FIFO 防并行，断线 catch-up 不重复选主。

### Raft：广播 + 门禁过滤 + 自律
- `deliverMessagesToAgents`（`messageService.ts:3834`）把消息投给**频道内所有 agent**（不只被 @），过滤链：静音（@ 穿透、task assignee 穿透、thread 永不静音）→ 已读资格（`inboxUnreadEligibility.ts` 单一未读规则，personal mention 无条件穿透）→ `inbox:receive` scope → 目标可达性。
- **没有 grant 协议**（grep `replyGrant|supplemental|replyCoordinat` 全仓零命中）。防"过时回复"有 **send-freshness hold**（`attestedSendService`：seenUpToSeq 落后于注意力锚点就 hold 成 draft，CLI `--send-draft` / `--anyway`，3 次后建议越闸）；防"抢活"有 **task claim CAS 锁**（失败的 claim 是锁不是裁决，`taskService.ts:190-310`）；**防"多人抢答"只靠 prompt 礼仪**（`raftCliGuide.ts:500-507`："Only the person doing the work should report on it" / "Respect ongoing conversations"）。
- Raft 独有：**5 层 inbox**（`inbox_notification_facts` 持久事实 / server replayable inbox（内存 1000 条上限）/ daemon 本地 inbox / `agentInboxProjection` 渲染 / `agentInboxStateMachine` freshness 决策）、mention 投递可持久化可重放（`redriveMentionDelivery` + `raft mention pending|notify|add`）、外部 agent 的签名 wake-hint 拉取模式、跨副本机器路由。

**评价**：open-tag 把"谁该回话"做成了数据库约束；Raft 只保证"消息不丢、不旧"，"谁回"交给模型。前者可推理、可审计；后者灵活但依赖模型素质。

## 5. 进程模型 / Runtime

| | open-tag | Raft |
|---|---|---|
| 运行时 | 9 个适配器（claude/codex/copilot/opencode/kimi/pi/cursor/hermes/reasonix） | 12 个 driver（含 grok、gemini、kimi-sdk、builtin=内嵌 Pi；copilot/antigravity 已 deprecated） |
| 生命周期 | 混用：claude/codex 持续会话；其余 one-shot-per-turn + `--resume` | **显式契约枚举** `RuntimeLifecycleContract = persistent \| per_turn`（`drivers/types.ts:13-24`）+ contract test 钉死矩阵 |
| 会话形态 | `--resume` 续会话，空闲休眠，`MEMORY.md` 压缩自救援 | persistent（stdin 直投 / busy steering）+ **in-process SDK 会话**（kimi-sdk、builtin）；**wakeSessionRecycle**（RFC 070：冷置 ≥1h 且上下文 ≥128k 弃 resume 换新会话+重建简报，省 provider 缓存重写费）；冷闲置 10min 回收 |
| 启动 | start guard 纯函数门禁 | `AgentStartCoordinator`：per-agent 单槽 + 全局 FIFO + `maxConcurrentStarts=5` + 500ms 最小间隔 |
| 工具审批 | 全自动放行 | 同样 yolo（claude `--dangerously-skip-permissions`、codex `approvalPolicy:"never"`、grok 自动 allow_once），但禁 runtime 自带 cron/wakeup 工具以保住 reminder 归属 |
| **资源管控** | **`resourceLimit.ts`：Windows Job Object（koffi FFI）/ Linux cgroup `memory.max`，`OPEN_TAG_PRESSURE_MEM_MB` 压力阈值** | **无任何 OS 沙箱 / 内存 CPU 限制**（仅启动节流） |

## 6. 记忆（同名 `MEMORY.md`，不同实现）

| | open-tag | Raft |
|---|---|---|
| 位置 | `~/.open-tag/agents/<id>/` + `personality.md` 覆盖 description | `$RAFT_HOME/agents/<id>/MEMORY.md` + `notes/` |
| 注入 | 指令要求启动重读；压缩后重读自救援 | **启动时把 MEMORY.md 头部注入首个输入**（4k token 预算；resume 不注入）——`startupMemoryBlock.ts` |
| 卫生 | prompt 规则 | prompt 规则 + **Cleaner app** 超 16KB 每日至多警告一次 |
| 迁移 | 无 | **agent 迁移导出**：MEMORY.md 永不排除、`.raftmigrateignore`、断点续传 tar 分块 + 收据、源目录归档 3 份/30 天、EXDEV 安全 copy-verify-swap |
| 自救 | MEMORY.md 重读 | + 压缩失败台账（3 连败拒 resume `session_needs_reset`）+ context-generation 信号（压缩后重教 CLI） |

## 7. 实时 / 同步 / 读状态

- **open-tag**：Redis INCR 分配 `seq`（服务端维度）；socket.io room（`channel:<id>` 内容隔离 + `server:<id>` 元数据），view-driven join；**单实例**（tech-debt I6：横向扩展要补 `@socket.io/redis-adapter`）。
- **Raft**：DB `bigserial` seq + `sync-core` 纯函数客户端同步状态机（RFC 043，无 IO/时钟/随机）+ redis adapter 多副本 + `sync:resume` lastSeq 回放（cap 500）+ 15s 心跳水位 + **读状态变更日志**（`read_mutations`：lease / authoritySeq / 幂等 / 墓碑）+ RisingWave 物化 inbox 链 + 公平的"从最旧未读开始连续回放" resume catch-up。

## 8. 工程方法论

- **open-tag 过程纪律见长**：doc-sync 硬规则（改代码必同 commit 改文档）、tech-debt 编号台账（80+ 条含证据/状态）、worktree 并行开发、chrome-devtools 浏览器截图验证、daemon npm 发布流程（merged ≠ shipped）。
- **Raft 产品工程深度见长**：RFC 编号设计文档（043/057/070/071/073）、契约测试钉行为（`runtimeContract.test.ts`）、trace/OTLP 全链路（trace-client + 独立上传 worker + 事件缓冲包）、visual-testing（react/android/ios/harmony 视觉回归）+ ui-inventory、SEA 单文件可执行 `raft-computer`（签名公证 + 自更新 + launchd/systemd 监督）、manual 体系（40+ 主题 agent 知识库 + recipes 卡片，`raft manual get --intent --reason` 检索）。

## 9. 一句话

Raft 是大体量 source-available 商业产品：**投递可靠、凭据零信任、基建豪华，但协作秩序靠 prompt 自律**；open-tag 是轻量真开源替代：**协作秩序做成数据库协议（Turn/grant/准入），但凭据暴露面和同步基建是待补的短板**。两者最互补的借鉴方向——open-tag 值得学 Raft 的 credential proxy 与 MEMORY.md 启动注入；Raft（若许可允许）值得学 open-tag 的显式回复授权协议。
