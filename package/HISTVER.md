# 版本历史

> 本文件记录**正式公开**（1.5.0）及之后的版本变更。正式公开之前的私有迭代见
> [HISTVER_legacy.md](./HISTVER_legacy.md)：其中版本从未发布、不保留源码与产物，记录也可能不完整，
> 仅供追溯参考。

## 版本总表

| 版本 | 日期 | 主题 |
|---|---|---|
| 1.5.0 | 2026-11-02 | 首次公开发布（基线） |

## 1.5.0 — 首次公开发布

- **一张插件卡、两个组件包**：bundle `@local/dsh-user-attention-and-todo-inbox` 与组件包
  `@local/dsh-user-attention-core`、`@local/dsh-todo-inbox` 版本统一为 `1.5.0`，bundle 的
  `dependencies` 精确钉住同版本，单个自包含 tgz 离线可装。
- **人类交互门控**：用户注意力组件把「人在不在」做成在线/暂离/离线三态；暂离期间对批准与提问多轮提醒，
  提醒耗尽后按 `reject` / `enqueue` 策略最终拒绝；已离线时立即拒绝、不提醒。
- **待办收件箱**：`enqueue` 策略下被最终拒绝的批准/提问收进持久化待办，右栏待办页复核；AI 用
  `todo_list` / `todo_get` / `todo_execute` 查看与执行（批量一次最多 10 条，默认串行、首个失败即停）。
- **一次性授权**：`todo_execute` 执行前把 grant 登记在本次执行的新 `callId` 名下，该 `callId` 的审批请求
  被门控命中直接放行（`allowed-once`）、取出即删，TTL 5 分钟。
- **待办状态机与分支复制**：「待审核 / 已操作」分段；拒绝为 `rejected`、不删除、可「返回修改」，只在
  apply 与 `todo_*` 工具入口归档删除；分支对话派生时自动复制原会话未执行、未拒绝的待办（幂等）。
- **危险区**：合并设置页 5 秒长按「释放并删除待办缓存」，Host 配套 `POST /api/todo-inbox/purge`
  （保留 `executing` / `restarting`，文件保留不删除，接口幂等）。
- **验证**：`node --check` 4/4；待办 Host 49/49、Client 渲染 29/29、注意力 19/19。

## 后续版本

（尚未发布。正式公开后的每次版本变更在此追加，格式：版本号 / 日期 / 变更 / 兼容性。）

## 许可

本文件以 MIT 许可发布，版权归 DX12_XYZ 所有。完整条款见仓库根 [LICENSE](./LICENSE)。

<!-- fused-by: qwen3.8-max -->

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
