# open-tag vs Raft（raft-source）对比分析

> 基于对同级 `raft-source` 仓库（Raft 平台，Botiverse, Inc. 的 source-available 发布镜像）的源码分析，2026-10-14。
> Raft 即 open-tag README 对比表中的 "Slock / Raft" 竞品。

## 1. 开放性 / 许可（最本质的区别）

| | open-tag | Raft |
|---|---|---|
| 许可 | **Apache-2.0 真开源** | **FSL-1.1-ALv2 伪源码开放**：可读 / 可跑 / 可改，但**禁止用于与 Raft 竞争**；发布 2 年后转 Apache-2.0 |
| 协作模式 | 接受 PR，社区驱动 | 私有仓库的**发布镜像**（每版本一个导出提交），外部 PR 自动关闭；"Raft" 商标归 Botiverse, Inc. |

## 2. 体量与形态

- **open-tag**：约 2.2 万行 TS（不含测试），单仓多包（server + web + docs-site + `packages/daemon` npm 发布包），Node 20+，npm 脚本驱动。
- **Raft**：约 54 万行 TS（含测试逾 100 万行），pnpm monorepo 15 个包（server / web / cli / daemon / computer / raft-sdk / sync-core / trace-client / desktop-contract / agent 迁移导出 / 视觉测试等），Node 24 + pnpm 12，功能面大得多（桌面端、SDK、trace、agent 迁移与对象存储打包）。

## 3. Agent 编排哲学（差异最大）

- **open-tag — 协议化授权**：
  - 观察 ≠ 回复权：`check → decide → send only when granted`；
  - primary / supplemental 回复授权、Conversation Turn 聚合调度、两阶段投递准入、严格 FIFO 防并行；
  - agent 会话可 sleep / resume（`--resume` 持续会话 + `MEMORY.md` 持久记忆）；
  - 9 种运行时适配器（claude / codex / copilot / opencode / kimi / pi / cursor / hermes / reasonix）。
- **Raft — 消息驱动 + LLM 自主判断**：
  - 频道里**所有 agent** 都收到消息（不只被 @ 的），靠静音 / 已读 / inbox 资格过滤，@提及穿透静音；
  - 每轮对话结束进程退出，下次 wake 重新拉起（按需 spawn，非常驻）；
  - 凭据安全：agent 从不持有服务器密钥，daemon 经本地 `agentCredentialProxy` 代理转发，每次启动生成 per-launch token；
  - 围绕 Claude / Codex + managed MCP runtime proxy 构建（`agentProcessManager.claude/codex`）；
  - Server 侧 `AgentOrchestrator` 全局调度（消息路由 / 生命周期 / mention 投递 / inbox 聚合 / 状态机）。

## 4. 相同点

三层架构几乎同构：Server（调度大脑）↔ WebSocket 机器协议 ↔ Daemon（本机进程管理）↔ `raft` / `open-tag` CLI ↔ agent 子进程；同为频道 / 线程 / DM / 任务板工作区，均有唤醒、inbox 聚合、状态与记忆持久化、离线补发（replayable inbox / reconnect catch-up）。

## 5. 结论

Raft 是大体量的 source-available 商业产品（license 上不允许你用它做竞品）；open-tag 是它的轻量真开源替代。编排上：open-tag 走**显式授权协议**（回复冲突可控、可观测），Raft 走**广播 + LLM 自主判断**（实现简单，但依赖模型自律）。
