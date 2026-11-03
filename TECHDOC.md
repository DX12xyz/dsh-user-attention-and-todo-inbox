# 用户注意力与待办收件箱 · 技术文档（TECHDOC）

> 文档状态：适用版本 **`@local/dsh-user-attention-and-todo-inbox` 1.5.0**（bundle），组件包
> `@local/dsh-user-attention-core` 与 `@local/dsh-todo-inbox` 同为 `1.5.0`；最后校对时间：2026-11-03。
>
> 本文只写代码里实际存在的行为，以 `package.json`、`cordis.patch.yml`、
> `components/attention/index.js`（1396 行）、`components/attention/client.js`（1978 行）、
> `components/todo-inbox/index.js`（2495 行）、`components/todo-inbox/client.js`（2369 行）、
> 三个 `test/*.mjs` 为准；行数为当前工作区实测。
>
> 读者：开发者 / 运维。同目录配套文档：[README](./README.md)（定位、安装与最小上手）、[HISTVER](./HISTVER.md)（正式公开及之后的版本历史；私有时期迭代见 [HISTVER_legacy](./HISTVER_legacy.md)，仅供参考）。

## 目录

- [1. 架构](#1-架构)
  - [1.1 三层结构](#11-三层结构)
  - [1.2 为什么两个组件能打进一个 tgz](#12-为什么两个组件能打进一个-tgz)
  - [1.3 目录与文件清单](#13-目录与文件清单)
  - [1.4 注意力组件 Host 半身](#14-注意力组件-host-半身)
  - [1.5 待办组件 Host 半身](#15-待办组件-host-半身)
  - [1.6 两个 Client 半身](#16-两个-client-半身)
  - [1.7 组件边界与降级](#17-组件边界与降级)
- [2. 工具与 API 参考](#2-工具与-api-参考)
  - [2.1 三个 AI 工具](#21-三个-ai-工具)
  - [2.2 /api/user-attention/* 路由表](#22-apiuser-attention-路由表)
  - [2.3 /api/todo-inbox/* 路由表（含 purge）](#23-apitodo-inbox-路由表含-purge)
  - [2.4 服务接口](#24-服务接口)
- [3. 状态与持久化](#3-状态与持久化)
  - [3.1 state.json（注意力）](#31-statejson注意力)
  - [3.2 state.json（待办）](#32-statejson待办)
  - [3.3 history.json](#33-historyjson)
  - [3.4 写入与修复规则](#34-写入与修复规则)
  - [3.5 reloadExpected 对账](#35-reloadexpected-对账)
- [4. 安全模型](#4-安全模型)
  - [4.1 脱敏与哈希校验](#41-脱敏与哈希校验)
  - [4.2 一次性 grant 与 TTL](#42-一次性-grant-与-ttl)
  - [4.3 执行前置检查顺序](#43-执行前置检查顺序)
  - [4.4 approval 门控顺序](#44-approval-门控顺序)
  - [4.5 reviewOnly（仅复核）](#45-reviewonly仅复核)
  - [4.6 跨会话边界](#46-跨会话边界)
- [5. 配置字段](#5-配置字段)
  - [5.1 注意力行（id: user-attention）](#51-注意力行id-user-attention)
  - [5.2 待办行（id: todo-inbox）](#52-待办行id-todo-inbox)
  - [5.3 合并设置页写回哪一行](#53-合并设置页写回哪一行)
  - [5.4 409 / force 语义](#54-409--force-语义)
- [6. 测试与验证](#6-测试与验证)
  - [6.1 语法与单测](#61-语法与单测)
  - [6.2 隔离 profile 配置检查](#62-隔离-profile-配置检查)
  - [6.3 隔离 profile 启动 Web](#63-隔离-profile-启动-web)
  - [6.4 界面手工验收清单](#64-界面手工验收清单)
- [7. 附录](#7-附录)
  - [7.1 打包产物与校验](#71-打包产物与校验)
  - [7.2 错误码表](#72-错误码表)
  - [7.3 常量表](#73-常量表)
  - [7.4 术语表](#74-术语表)
  - [7.5 已知限制](#75-已知限制)

## 1. 架构

一句话结论：本包是「一张插件卡 + 两个可独立开关的组件行」的 bundle——注意力组件判定人是否在场并
门控人类交互，待办组件接住被最终拒绝的交互供人复核与执行；两者之间没有共享模块，只通过两个窄接口
协作。

数据流：`approval/request` / `user-questions/request` 进入注意力组件的门控 → 超时或离线时
`finalizeRejection()` 产生拒绝事件 → 待办组件 `userAttention.onRejection()` 入队 →
用户在右栏复核（批准/拒绝/回答/返回修改）→ `todo_execute`（AI 侧）或 `apply`（界面侧）执行一次。

### 1.1 三层结构

| 层 | 包 | 关键声明 | 职责 |
|---|---|---|---|
| 插件卡（bundle） | `@local/dsh-user-attention-and-todo-inbox@1.5.0` | `dsh.bundle.patch: ./cordis.patch.yml`、顶层 `icon: ./icon.svg`、`locale/{zh,en}.json` 的 `meta.title/description` | 卡片标题/描述/图标；两条 Loader 行的默认 config；用 `dependencies` 精确声明两个组件版本（`1.5.0`，不带 `^`/`~`） |
| 组件 1 | `@local/dsh-user-attention-core@1.5.0` | `dsh.client: { platform: web, immediately: true, inject: ['@deepseek-ai/dsh-client-ui-conversation'] }` | 在线/暂离/离线状态机、人类交互门控、`userAttention` 服务、超时策略、`/api/user-attention/*`、状态胶囊与提醒浮层 |
| 组件 2 | `@local/dsh-todo-inbox@1.5.0` | `dsh.client: { platform: web, immediately: true, inject: [session-controller, client-locale, conversation, sidebar-right, workspace] }` | 持久待办收件箱、`todoInbox` 服务、`todo_list/get/execute`、一次性 grant、右侧栏待办页、输入框高亮条、`/api/todo-inbox/purge` |

bundle 自身**没有** `dsh.client`：它不是客户端插件，只负责「一张卡 + 一个补丁 + 两个依赖」。
`package.json` 的 `files` 白名单只含卡片资产（`cordis.patch.yml`、`locale/*.json`、`icon.svg`
与三份文档），组件不进白名单——它们靠打包阶段塞进 `node_modules/`。

`cordis.patch.yml` 只有一个 `insert`，两行记录：

```yaml
- insert:
    - id: user-attention
      name: '@local/dsh-user-attention-core'
      config: { ... }
    - id: todo-inbox
      name: '@local/dsh-todo-inbox'
      config: { ... }
```

### 1.2 为什么两个组件能打进一个 tgz

打包方案出自「多模块插件打包为 tgz」这份方案文档，用的是其中的**方案 B：`bundledDependencies`**。

原理：npm 会把包内 `node_modules/<dep>` 原样留在 tarball 里；安装时 pnpm 不解包重装、也不回源
registry。DSH 启动时沿 **bundle 包的依赖闭包**（从 bundle 自己的 `package.json` 起做 Node 查找）
把组件登记进 profile 的模块解析表，`cordis.patch.yml` 的 `name:` 因此解析到嵌套副本。

打包时先生成一份 `stage/`：bundle 本体按 `files` 白名单取出，每个组件按包名放进
`stage/node_modules/<name>/`，再对 stage 的清单写入 `bundledDependencies`，最后只对这一棵树打包
（工具见 7.1）。因此**源码 `package.json` 里看不到 `bundledDependencies`**，它只存在于打包中间产物
和最终 tgz 的 `package/package.json` 里。

### 1.3 目录与文件清单

bundle 侧只有卡片资产（`package.json`、`cordis.patch.yml`、`icon.svg`、`locale/{zh,en}.json`）
与三份文档；每个组件目录是一份完整包：`package.json`、`index.js`、`client.js`、`icon.svg`、
`locale/{zh,en}.json`、`test/`（注意力 779 行 / 19 用例；待办 Host 2495 行 / 49 用例 +
Client 渲染 1466 行 / 29 用例）。两个组件都**不 import 任何 `@deepseek-ai/*` 包**，Host 能力全部
从 `ctx` 取；Client 只 `require('react')`。

### 1.4 注意力组件 Host 半身

入口 `components/attention/index.js` 导出 `apply(ctx, config)`；`normalizeConfig()` 先把 YAML
收敛成安全值（不信任类型），再进入状态机。

#### 1.4.1 三态与转换

状态：`online` / `away` / `offline`；`manual` 标记「人类手动选择」。

| 当前 | 触发 | 结果 | 代码位置 |
|---|---|---|---|
| `online` | `tick()` 发现 `now - lastActivityAt >= idleTimeoutMs`（`tickMs` 一次） | → `away` | `tick()` |
| `online` / `away` | 浏览器上报活动且 `!manual` | → `online` | `noteActivity()` |
| `away` | 门控重试耗尽且非手动 | → `offline` | `presentedGate()` |
| `away` | `awayRetryCount === 0` 且发生门控 | → `offline`（先拒绝本次） | `attentionGate()` |
| 任意 | 手动 `POST /api/user-attention/set` | 目标态，`manual = (next !== 'online')` | `set()` 路由 |
| 手动 `away`/`offline` | 鼠标/键盘活动 | 只刷新 `lastActivityAt`，**不**回在线 | `noteActivity()` |
| 手动 `away` | 交互超时耗尽 | 只拒绝本次，**不**降级离线（`timeout-manual`） | `presentedGate()` |

手动 `away` / `offline` 是粘性的：`manualState = true` 时自动状态机被冻结，只有手动切回 `online`
才解冻并删除状态文件。冷启动时先 `loadPersistedManualState()`，恢复成功则 `state` 直接是手动态，
`initialState` 只在没有可恢复状态时生效。

#### 1.4.2 门控接入：`prepend` 抢在浏览器应答者之前

```js
ctx.on('approval/request', (req, next) => handleApproval(req, next), { prepend: true })
ctx.on('user-questions/request', (req, next) => handleQuestion(req, next), { prepend: true })
```

`prepend: true` 让本插件成为瀑布流第一环；返回 `next()` 的结果表示交回下游（浏览器 Remote
应答者），返回 `'rejected'` / `'allowed-once'` 或抛错则表示已裁决。

`handleApproval()` 的判定顺序（顺序本身是安全边界）见 4.4。`handleQuestion()` 没有 grant 分支
（问题不重放）。`trackOnlineInteraction()` 的注释写明：当前实现不把在线投递纳入注意力监督，该语义
留给后续版本。

#### 1.4.3 呈现、等待与提醒重试

门控先处理两种「不呈现」的短路：

- `signal.aborted` → `abortResult()`（approval 返回 `'cancelled'`，提问抛 `ASK_ABORTED`）；
- `state === 'offline'` → 立即 `finalizeRejection(..., 'offline')`，不占用 `activeInteractions`。

其余进入 `presentedGate()`，`maxAttempts = awayRetryCount + 1`：

| 环节 | 实现 |
|---|---|
| 可撤回呈现 | `presentWithdrawable()` 把 `req.signal` 换成 `AbortSignal.any([original, controller])`，`withdraw()` 只 abort 自己的控制器 → gateway 给页面发 cancel 帧，卡片消失 |
| 等待竞速 | `raceGate()` 同时等 `next()` 结果 / 控制器唤醒 / `awayRetryIntervalMs` 超时 / `req.signal` abort |
| 每一轮超时 | 按**当前真实状态**决策：`online` 把 `attempt` 归零继续等；`offline` 撤回并拒绝；`away` 累计 `attempt`，到 `maxAttempts` 撤回并按手动/自动分别以 `timeout-manual` / `timeout` 拒绝 |
| 提醒 | 每轮调用 `publishRetryAlert(attempt, maxAttempts)`，文案「第 n/awayRetryCount 次提醒」，`ttlMs = awayRetryIntervalMs + 20000` |
| 唤醒 / 收尾 | `manual-offline` 立即拒绝（`manual`），`cancel` 视为中断；`finally` 里 `presented.restore()` 还原 `req.signal`，从 `activeGates` / `activeInteractions` 摘除 |

计数语义：`awayRetryCount = 5` → 6 个等待间隔（约 `6 × awayRetryIntervalMs`），期间发出 5 次
提醒，最后一个间隔到期即拒绝。`awayRetryCount = 0` → `maxAttempts = 1`，暂离下第一次门控就直接
拒绝并（非手动时）降级离线。

#### 1.4.4 最终拒绝的四种原因

`finalizeRejection(kind, req, reason)` 是唯一收尾口，顺序固定：推荐项自动回答 → 发布拒绝提醒 →
给模型注入说明 → 广播拒绝事件 → 返回/抛出。

| `reason` | 触发场景 | approval 结果 | question 结果 | 状态影响 |
|---|---|---|---|---|
| `manual` | 手动切到离线后唤醒门控 | `'rejected'` | 抛 `USER_ATTENTION_REJECTED` | 保持手动离线 |
| `offline` | 门控时已离线 / 等待中转为离线 | 同上 | 同上 | 无 |
| `timeout` | 自动暂离、重试耗尽 | 同上 | 同上 | 自动 → `offline` |
| `timeout-manual` | 手动暂离、重试耗尽 | 同上 | 同上 | 保持手动暂离 |

模型可见说明走 `agent.inject({ role: 'user', source: { kind: 'user-attention', form: 'notice' } })`，
用户界面能看到这条可回放的 user 消息。文案按原因分四种；当 `todoInboxKeepsRejections()` 为真时追加
一句「已进待办，用户批准/回答后调用 `todo_execute`，不要自己重试」。

`todoInboxKeepsRejections()` 的判定（这就是「待办是否可用」的 Host 判据）：

```js
const inbox = ctx.get('todoInbox')
if (inbox === null || typeof inbox !== 'object' || typeof inbox.takeApprovalGrant !== 'function') return false
return typeof inbox.autoEnqueueEnabled === 'function' ? inbox.autoEnqueueEnabled() === true : true
```

#### 1.4.5 超时策略与推荐项

| 能力 | 规则 |
|---|---|
| `approvalTimeoutAction` / `questionTimeoutAction` | `reject` \| `enqueue`，**只对超时类原因**生效（`timeout` / `timeout-manual`）；`offline` / `manual` 的拒绝事件始终带 `enqueue: true`（真正是否入队仍由待办组件自己判断） |
| 事件字段 | `emitRejection()` 里写 `event.enqueue = timeoutEnqueueDecision(kind, reason)`，待办侧 `event.enqueue === false` 直接丢弃 |
| `preferRecommended` | 仅 `question`、仅超时原因、且 `buildRecommendedAnswer()` 对**整批问题**都找到推荐项时，直接返回推荐答案（不进拒绝、不入队）；任一问题缺推荐项则退回正常拒绝 + 入队 |
| 推荐标记 | `RECOMMENDED_LABEL_SUFFIX`（见下方代码块），与 `dsh-client-ui-user-questions` 的 `parseRecommendedLabel` 同规则 |

```js
RECOMMENDED_LABEL_SUFFIX = /\s*(?:\((?:recommended|推荐)\)|（(?:recommended|推荐)）)\s*$/i
```

#### 1.4.6 一次性 grant 消费

门控第一环调用 `takeTodoGrant(callId)` → `ctx.get('todoInbox').takeApprovalGrant(String(callId))`；
服务不存在或方法缺失时返回 `false`，门控行为与没有待办插件时完全一致。grant 的登记、TTL、取出即删
与消费日志统一见 4.2。

#### 1.4.7 `userAttention` 服务

`ctx.provide('userAttention', service)`，这是本组件对其他 Host 插件的唯一稳定面。服务成员为
`version`（值为 `1`，即 `ATTENTION_REJECTION_VERSION`）、`snapshot()`、`subscribe(fn)`、
`onRejection(fn)`、`todoAutoEnqueue()`：其中 `snapshot()` 返回
`{ state, manual, since, lastActivityAt, idleTimeoutMs, awayRetryCount }`，`onRejection(fn)` 在每次
最终拒绝时回调一个拒绝事件（由 `buildAttentionRejection()` 构造）；完整签名、`subscribe()` 的载荷与
拒绝事件字段见 2.4。**调用参数不进事件**——待办按 `(originSessionId ?? sessionId) + callId` 回会话
日志读取并做哈希校验（`originSessionId` 只出现在分支复制的待办上，见 1.5.3）。

#### 1.4.8 上下文注入

- `systemPrompt.context({ name: 'user-attention', order: 118, text })`：三种状态各一段英文运行时
  说明——在线说明 `idleTimeoutMs` 秒无活动后转暂离；暂离说明重试次数与「无响应即拒绝并转离线」；
  离线说明立即拒绝且不要依赖人类输入。手动态额外强调「粘性，活动不会自动恢复」。
- `agent.inject(...)`：最终拒绝时的可回放说明（见 1.4.4）。

冷启动时这些服务可能还没就绪，所以注册统一走 `ctx.inject(['systemPrompt'], ...)` /
`ctx.inject(['timer'], ...)` / `ctx.inject(['connection'], ...)`，注册体再包 `scope.effect(...)`
以便随插件卸载回收。

### 1.5 待办组件 Host 半身

入口 `components/todo-inbox/index.js`；`inject = ['tools', 'agents', 'sessions']`，`userAttention`
与 `connection` 是可选面（`ctx.get` / `ctx.inject` 懒查）。

#### 1.5.1 记录结构与状态

`createTodoRecord()` 产出的记录（持久化字段）：

| 字段 | 说明 |
|---|---|
| `id` / `source` | `todo-<base36 时间>-<随机>`；`source` 固定 `'attention'` |
| `status` | `pending` →（批准/回答）`approved` →（执行中）`executing`（重启类为 `restarting`）→ 成功后删除；`pending` →（拒绝）`rejected`（不删除、可 `restore` 回 `pending`）；失败 → `failed` |
| `kind` | `approval` / `question` |
| `sessionId` / `callId` / `toolName` | 来源会话与原始调用身份（`toolName` 仅 approval） |
| `originSessionId` | 可选：分支复制来的待办指回原会话（≤200；本会话产生的为 `null`）。执行时优先用它查原始 `tool/call`，缺失时回退 `sessionId` |
| `rejectReason` | `manual` / `offline` / `timeout-manual` / `timeout` |
| `reviewOnly` | 元工具（`^todo_`、`^ask_user_question$`、`^workflow$`、`^subagent_fork$`）标记为仅复核，可拒绝不可执行；`restart_dsh_*` 自 1.4.3-hotfix.1 起已放开 |
| `title` / `detail` / `requestReason` | 人类可读文案（`describeRejection()` 生成，有界）；`requestReason` 为原始审批理由（≤400 字） |
| `argsHash` / `argsPreview` / `referencedFiles` | 原始参数的 sha256、脱敏预览（≤1000 字）、最多 8 条引用文件路径 |
| `questions` / `answers` | 提问快照（经第二道 `sanitizeTodoQuestions`）与用户答案；`rejected` / `pending` 里的 `answers` 是「草稿」（允许没答完），`approved` 才是严格校验后的完整答案 |
| `createdAt` / `decidedAt` / `executedAt` | 时间戳 |
| `resultSummary` / `failReason` | 成功输出预览 / 失败原因（≤500 字） |

安全边界：**只持久化脱敏预览与哈希**，完整参数永远只从来源会话 `tool/call` 事件里读
（`findToolCallArguments()`），执行前重算 `sha256Hex(call.arguments)` 比对。

**`rejected` 的删除时机（1.4.3-hotfix.10）**：`reject` 本身只置状态、不删除；`rejected` 条目只在
两处由 `purgeRejectedTodos()` 归档删除（`outcome='rejected'`）——`POST /api/todo-inbox/apply`
处理过程中，以及 `todo_list` / `todo_get` / `todo_execute` 进入实现后。`state` / `history` /
`decide` / `restore` 都不会清理它，避免用户还在「已操作」里查看时被后台轮询删掉。
`purgeRejectedTodos()` 同步执行（Node 单线程内不会与其它请求交错），循环里按 `id + status`
重新定位、`archiveTodo()` 按 `id` 去重，重复调用幂等。

#### 1.5.2 自动入队

订阅路径：`userAttention.onRejection(enqueueRejection)`，先 `ctx.get('userAttention')` 直取；
拿不到再 `ctx.inject(['userAttention'], ...)` 等待。成功后日志
`[todo-inbox] subscribed to userAttention rejection events`。

`enqueueRejection(event)` 的过滤链：

| 顺序 | 条件 | 结果 |
|---|---|---|
| 1 | `effectiveAutoEnqueue()` 为假 | 丢弃 |
| 2 | `event.enqueue === false`（注意力超时策略选了 `reject`） | 丢弃 |
| 3 | `kind` 不是 `approval`/`question`，或 `sessionId` 为空 | 丢弃并记日志 |
| 4 | 同 `kind + sessionId + callId` 且 `status !== 'failed'` 已存在 | 视为重复丢弃 |
| 5 | `todos.length >= maxTodos` | 丢弃并记日志 `todo inbox is full` |
| 6 | `createTodoRecord()` 返回 `{ skipped }` | 丢弃 |

`effectiveAutoEnqueue()`：优先读 `userAttention.todoAutoEnqueue()`（bundle 设置页写在**注意力行**），
注意力组件不在时回退本行 `config.autoEnqueue`。

#### 1.5.3 分支对话复制（`session/created`）

DSH 从既有会话派生分支时发 `session/created`，分支的 `session.header` 带 `parentSession`
（原会话 id，`string`）与 `isSeeded: true`；普通新会话没有 `parentSession`。Host 在 `apply()`
里用 `ctx.effect(() => ctx.on('session/created', handler), 'todo-inbox: fork copy todos')` 订阅
（`ctx.on` 返回 disposer，随插件卸载回收），handler 整体 try/catch：复制失败只记日志
`fork todo copy failed`，绝不影响会话创建。

| 规则 | 行为 |
|---|---|
| 触发条件 | `session.header.isSeeded === true` 且 `header.parentSession` 为非空字符串、不等于分支 id；无 `parentSession` 的普通会话直接跳过 |
| 复制范围 | 只复制 `status` 为 `pending` / `approved` 的父会话待办；`failed` / `executing` / `restarting` / `rejected` 不复制 |
| 副本字段 | 新 `id`（`newTodoId()`，确保不与现有 id 重复）、`sessionId = 分支会话 id`、`originSessionId = parentSession`；其余字段（`kind` / `toolName` / `callId` / `argsHash` / `argsPreview` / `title` / `detail` / `requestReason` / `referencedFiles` / `createdAt` / `decidedAt` / `answers` / `reviewOnly` 等）原样保留，`questions` / `answers` 深拷贝 |
| 原会话 | 只新增副本，不移动、不修改原待办；两边此后各自独立（分支执行/拒绝不影响原会话） |
| 幂等 1 | `forkedSessions`（有界 100 条、随 `state.json` 持久化）记录已处理的分支；同一分支重复 `session/created`（恢复分支）或宿主重启后都不再复制 |
| 幂等 2 | 分支里已经存在任何待办时视为已处理：只记录、不复制 |
| 容量 | 复制条数受 `maxTodos` 限制，超出部分不复制并记日志 |

`originSessionId` 是执行链路的关键：DSH 的 `ownEvents()` 只返回 fork 继承前缀之后的自有事件，
分支会话里没有父会话的 `tool/call`。`executeApprovalTodo()` 因此用
`callSourceSessionId(todo)`（`originSessionId ?? sessionId`）调 `readCallFromSession()`，工具名比对
与 `argsHash` 校验不变；原会话已不在活跃列表时仍返回原来的「无法从来源会话日志定位原始工具调用
（会话可能已归档或日志已清理）；待办保留，可检查后重试」。

#### 1.5.4 `todoInbox` 服务

待办组件把对外能力收敛成一个 `todoInbox` 服务（`version` 为 `1`）：它提供待办的读取入口
（`list` / `get` / `totals`）、一次性 grant 的消费入口（`takeApprovalGrant`，机制见 4.2）与
「待办是否可用」的判据来源（`autoEnqueueEnabled`）。成员表见 2.4；注意 `list` / `get` 走
`summarizeTodo` / `detailTodo` 投影，**不含答案**。

#### 1.5.5 失败保留

执行结果有三类「看起来成功、其实失败」，都必须保留待办为 `failed` 而不是删除：

| 检查 | 判据 | 处理 |
|---|---|---|
| 工具错误 | `result.isError === true` | `failReason = 内容预览`，`executedAt = null`，抛错 |
| 业务失败 | `detectToolValueFailure(result.value)`：`value.application === 'failed'` 或 `value.ok === false` | 同上，附 `error.code/message` |
| 进程失败 | `detectProcessFailure(value, text)`：`kind === 'foreground'` 的 `exitCode !== 0`、`signal`、`timedOut`、`aborted`、`stopped`，或文本里的 `[exit code: N]`（N≠0） | 同上 |

只有三类都通过才 `status = 'approved'`、写 `resultSummary`、`removeTodo()`。`finally` 里
`removeGrant(callId)`，保证授权不会残留。

#### 1.5.6 一次性 grant 与 TTL

一次性 grant 的登记（`registerGrant`）、TTL（`TODO_GRANT_TTL_MS`）、取出即删
（`takeApprovalGrant`）、过期清理（`sweepGrants`）与执行收尾时的 `removeGrant` 统一见 4.2。

#### 1.5.7 回注来源会话

| 动作 | 注入内容 | 目的 |
|---|---|---|
| `decide approve` | 「用户已操作待办 … 可调用 todo_execute 执行一次」 | 让模型知道可以继续 |
| `decide answer` | 「用户已回答待办 … 请调用 todo_execute 取回答案」 | 同上 |
| `decide reject`（仅 `failed`） | 「用户拒绝了待办 … 这次操作不会执行」 | 明确不要重试；`pending` 的拒绝**不注入**（可返回修改） |
| `apply` | `agent.followup(...)` 主动把「请调用 todo_execute」推进会话 | 用户在界面上点「应用操作」时立即唤醒模型 |

注入统一 `source: { kind: 'todo-inbox', form: 'notice', summary }`，摘要截断 120 字。注入与
`apply` 都针对 `todo.sessionId`（待办所属会话）：分支复制来的待办属于分支会话，通知只发给分支，
原会话不会被唤醒。

### 1.6 两个 Client 半身

#### 1.6.1 加载形态

两个 `client.js` 都是同一个外壳，没有 `import`，只用 `require('react')`：

```js
window.__ModuleLoader__.load({
  id: '@local/dsh-user-attention-core',   // 或 '@local/dsh-todo-inbox'
  factory(require) {
    const React = require('react')
    // ... 组件 ...
    return { inject: ['slots'], apply }   // 待办：['slots','locale','sessions','uiWorkspace']
  },
})
```

Loader 通过包清单的 `dsh.client` 发现入口（`exports["./client"]`），`immediately: true` 表示随页面
立即加载；`inject` 列表里是客户端 UI 包的预置依赖。

#### 1.6.2 注意力 Client

`inject: ['slots']`，`apply(ctx)` 里只做四件事：三个 slot 注册 + 一个运行期 effect。

| slot | 选项 | 组件 | 作用 |
|---|---|---|---|
| `conversation.composer.dock` | `id: 'user-attention'`, `order: 10` | `ChipEntry` → `Chip` | 输入框下方的状态胶囊：圆点颜色 + 「在线/暂离/离线/未连接」+ 手动标记「手动」；点击弹层选三态，写入 `POST api/user-attention/set` |
| `shell.overlay` | `id: 'user-attention-alert'`, `order: 900` | `AlertEntry` → `AlertBanner` | frame 级提醒浮层：重试提醒（20s 自动消失）、最终拒绝提醒（8s）、以及没有浮层时的「已自动拒绝 N 次」汇总条（需手动确认，走 `POST api/user-attention/ack`） |
| `plugins.bundle.config` | `key: '@local/dsh-user-attention-and-todo-inbox'` | `ConfigEntry` → `AttentionConfigPage` | bundle 合并设置页（见 5.3） |

运行期 `createRuntime()` 负责活动上报、轮询、提醒与标题闪烁/提示音：

| 常量 | 值 | 含义 |
|---|---|---|
| `ACTIVITY_THROTTLE_MS` | 1500 | 活动上报最小间隔；额外 250ms 合并延迟（`ACTIVITY_REFRESH_DELAY_MS`） |
| `DEFAULT_POLL_MS` / `MIN_POLL_MS` / `MAX_POLL_MS` | 1500 / 1000 / 5000 | 状态轮询间隔，以 Host 返回的 `pollIntervalMs` 为准并夹在此区间 |
| `HIDDEN_POLL_MS` | 5000 | 页面 `visibilityState === 'hidden'` 时的轮询下限 |
| `MAX_BACKOFF_MS` / `OFFLINE_AFTER_FAILURES` | 10000 / 3 | 连续失败 ≥3 后按 `currentPollMs × 2^(failures-2)` 退避，上限 10s；界面显示「未连接」 |
| `REQUEST_TIMEOUT_MS` / `WATCHDOG_GRACE_MS` | 5000 / 1000 | 每个 fetch 硬超时；轮询看门狗比它多 1s，防止 fetch 永不 settle 时停摆 |
| `RETRY_BANNER_TTL_MS` / `REJECTED_BANNER_TTL_MS` / `TITLE_FLASH_MS` | 20000 / 8000 / 12000 | 浮层与标题闪烁时长 |

活动采集：`pointermove`、`pointerdown`、`keydown`、`wheel`、`touchstart`（`passive: true`）与
`window` 的 `focus`、`document` 的 `visibilitychange`（可见时上报 `visible` 并立即刷新）。
非 `isTrusted` 的 `pointermove` 被忽略；`pointerdown` / `keydown` / `touchstart` 会顺便
`audio.unlock()` 解锁提示音。`alertSeq` 或 `instanceId` 变化时重置提醒纪元，避免 live reload 后
重复弹旧提醒。

配置页（`AttentionConfigPage`）字段：空闲判断时间（分钟，1–1440）、暂离重试次数（0–5）、
「启用待办收件箱（自动入队）」勾选、批准超时/提问超时各一组「拒绝 / 加入待办」单选、「优先推荐」
勾选；`todoAvailable !== true` 时整块隐藏并显示「未启用待办收件箱组件；超时行为固定为拒绝」。

**危险区（1.5.0）**：仅在 `todoAvailable === true && todoAutoEnqueue === true` 时渲染，位置在
批准/提问超时与「优先推荐」说明之后的待办区块末尾。结构：

| 元素 | 内容 |
|---|---|
| 容器 | `.ua-cfg-danger`：红色虚线框（`color-mix` 8% 底 + 45% 边框，颜色取 `--dsw-alias-state-error-primary`） |
| 标题 | 「释放并删除待办缓存」（`.ua-cfg-danger-title`，红色 600 字重） |
| 说明 | 「立即释放并删除未审批、未执行与历史的待办缓存；正在执行的待办会保留。DSH 本身不受影响。需要长按 5 秒确认。」 |
| 按钮 | `TodoPurgeHoldButton` → `button.ua-cfg-btn.ua-hold`：空闲「长按 5 秒释放并删除」、按住「继续按住… x.x 秒」、请求中「正在释放…」（`disabled`） |
| 结果行 | 成功「已释放并删除缓存（N 条待办、M 条历史；保留 K 条执行中）」；失败「释放并删除缓存失败：原因」；`role="status"` |

（`todoAutoEnqueue` 关闭或待办组件缺席时整块不出现。）

**长按交互（照搬逃生舱 `HoldButton`）**：常量 `TODO_PURGE_HOLD_MS = 5000`、
`TODO_PURGE_TICK_MS = 50`。`pointerdown`（`event.button === 0`）`preventDefault` +
`setPointerCapture` 后起表；`pointerup` / `pointercancel` / `pointerleave` /
`lostpointercapture`、`pointermove` 移出按钮矩形、`window` 的 `blur`、`document` 的
`visibilitychange`（`hidden`）都取消（`stop()` 清定时器、进度归零）且**不发请求**；只有累计按住
≥5000ms 才 `onConfirm()` → `rpc.purgeTodos()` → `POST api/todo-inbox/purge`。按住时按钮内底部
`::after` 进度条宽度取 CSS 变量 `--ua-hold-progress`（0–100%），`animationDuration` 随进度从
700ms 压到 120ms 的脉冲加快；`onClick` 只 `preventDefault`（单击不是确认）；键盘按住 Enter/Space
与指针同手势。请求进行中与配置保存中（`busy`）按钮禁用。成功/失败都不刷新整页；成功后注意力
Client 派发同源 window 事件 `dsh-todo-inbox:purged`（派发与监听语义见 2.3、1.6.3）。

样式新增 `ua-` 前缀（避免与逃生舱 `.dshpod-hold` 冲突）：`.ua-hold`
（`position/overflow/touch-action:none/user-select:none`、红色边框与文字、脉冲动画参数）、
`.ua-hold:hover:not(:disabled)`、`.ua-hold::after`（底部 2px 进度条）、
`.ua-hold-active { animation-name: ua-hold-pulse }`、`@keyframes ua-hold-pulse`
（`box-shadow` 0→4px），并在 `@media (prefers-reduced-motion: reduce)` 里关掉动画与进度条过渡。

i18n 现状：注意力 Client **没有**注册 locale 命名空间，界面文案是中文硬编码
（`STATE_LABELS = { online: '在线', away: '暂离', offline: '离线', unknown: '未连接' }`）。
`locale/zh.json|en.json` 只影响插件卡片标题/描述。

#### 1.6.3 待办 Client

`inject: ['slots', 'locale', 'sessions', 'uiWorkspace']`。

| 注册面 | 选项 / 方式 | 组件 | 作用 |
|---|---|---|---|
| `conversation.input.dock` | `id: 'todo-inbox-bar'`, `order: 25`, `inject: (sessionId) => ({ store, sessionId, openPanel })` | `DockBar` | 输入框上方全宽高亮条：待审核条数 + 打开右侧栏待办页；最新一条已操作时显示「应用操作」（见下方文案说明） |
| `sidebarRightTabs.register` | `{ id, kind: 'todo-inbox', priority: 'extension', title, guide[] }` | — | 右侧栏 page tab 类型；`guide` 提供「待办 / 复核被自动拒绝的批准与提问」入口 |
| `sidebar.right.pane.tab` | `key: '@local/dsh-todo-inbox'`, `inject: (sessionId) => ({ store, sessionId, openResource, sessions, openSession })` | `PaneTab` | 待办页正文：列表、批准/拒绝/回答、单次应用（发送回答请求 / 执行请求）、已拒绝条目的「返回修改」、打开引用文件、跳到来源会话 |
| `sidebar.right.pane.tab.title` | `key: '@local/dsh-todo-inbox'`, `inject: () => ({ store })` | `PaneTabTitle` | tab 标题与待审核计数 |
| `plugins.bundle.config` | 仅当 `attentionAvailable === false` 时注册，`key` 为 bundle 包名 | `BundleStandaloneNotice` | 橙色说明「用户注意力组件未启用…」（见 1.7） |

**应用按钮文案（1.4.3-hotfix.10）**：`DockBar` 的应用按钮显示「应用操作」（`EN_TEXT`:
`Apply operation`），但点击仍只对当前条范围里最新一条（`newest`）调用 `store.apply(newest.id)`；
`TodoCard` 的两个应用按钮显示「单次应用（发送回答请求）」/「单次应用（发送执行请求）」
（`EN_TEXT`: `Single apply (send answer request)` / `Single apply (send execution request)`），
仍只应用该卡片对应的待办。两者都不调用新的批量接口，Host 的 `api/todo-inbox/apply` 契约不变；
高亮条应用按钮的显示条件（非全局模式、`newest.status === 'approved'`、
`newest.reviewOnly !== true`）也不变。

**待办状态机与卡片（1.4.3-hotfix.10）**：`PaneTab` 的分段行显示「待审核 / 已操作 / 失败 / 历史」，
分段 key 仍是 `pending / approved / failed / history`；「已操作」（key `approved`）的过滤与计数合并
`status === 'approved' || status === 'rejected'`。卡片行为：

- `pending`：保持批准 / 回答 / 拒绝；问答类表单的草稿由 `TodoCard` 持有
  （`initialQuestionDraft(todo)` 在挂载时把 `todo.answers` 逐题还原成 `{ selected, custom }`），
  提交与拒绝都用当前表单值；
- `approved`：单次应用（发送回答请求 / 执行请求）；
- `rejected`：**不显示**「单次应用」与「拒绝」，只显示「返回修改」（`store.restore(id)` →
  `POST api/todo-inbox/decide { action: 'restore' }`）；问答类条目用历史卡片的只读问答块
  （`HistoryQa`）展示每题选中项与自定义文本，不渲染可编辑表单；
- `failed`：重新允许 / 拒绝。

`store.restore(id)` 成功后 `kick()` 立即刷新；返回修改带回来的 `todo.answers` 由 `state` 快照下发，
表单初始化时据此预填。拒绝问答类待办时，`TodoCard` 把 `toAnswerArray(questions, draftAnswers)`
随 reject 请求提交，Host 的 `validateDraftAnswers()` 宽松校验后存入 `todo.answers`。

**提问表单（`QuestionForm`，1.4.3-hotfix.9 起）**：`toggle()` 在单选题目（`multiSelect !== true`）
选中预设选项时同时清空该题 `custom`；`setCustom()` 在单选题目写入非空（`trim()` 后）自定义时清空
`selected`，清空输入不动 `selected`。渲染上，单选且存在选项时 `.ti-options` 末尾多一行「自定义」
`radio`（与预设选项同 `name`，`checked` 取自定义文本非空，`ti-option-on` 高亮随选中联动，点击/聚焦
把焦点交给保留下来的文本框）；多选不加这一行，仍可在选项之外补充自定义。`validateLocalAnswers()`
对单选增加防御：`selected.length > 0 && custom.trim() !== ''` 返回「单选题只能在预设选项和自定义
之间选一个」（多选不受影响）。提交的答案结构不变，仍是 `{ id, selected, custom }`。

轮询：`requestJson()` 打 `api/todo-inbox/state?scope=all`（无前导斜杠，与 DSH 应用约定一致）；
`PaneTab` 挂载/切换会话时调用共享 `store.setActiveSession(sessionId)`，请求变成
`api/todo-inbox/state?scope=all&session=<id>`，让 Host 能按当前会话回传 `sessionHasContent`；
会话切换时若请求在途，会在其结束后立刻补发一次，不等下一轮轮询。成功后按
`min(max(config.pollIntervalMs, 1000), 10000)` 排下一次；`visibilitychange`（可见）与 `window`
`focus` 会立即 `refresh()`；`decide` / `apply` / `restore` 成功后 `kick()` 立即刷新。失败保留旧
快照，界面显示「状态同步失败：…（每 N ms 重试）」。每个请求 5s 硬超时。轮询 effect 里同时注册
`dsh-todo-inbox:purged` 监听：收到后立刻 `refresh()`（`kick()`），页内待办列表下一帧就变空、不
刷新整页（事件由注意力 Client 在 purge 成功后派发，见 2.3）。

页面骨架：`PaneTab` 只有一条渲染路径，四个分段共用同一套头部、状态行与列表容器，只切换数据源。

| 区域 | class / 组件 | 收件箱三个分段 | 历史 |
|---|---|---|---|
| 标题行 | `ti-title-row` + `ti-count` + `ti-header-actions` | 「待审核 N · 已操作 N · 失败 N」 | 「共 N 条 · 只读」 |
| 刷新按钮 | `ti-header-actions` 里的 `ti-btn` | `store.refresh()` | `historyReload` 计数 +1 |
| 作用域行 | `ti-scope-row` + `ScopeSeg`（`ti-seg`） | 共享 `scope`（`scopeOverride ?? 按会话新旧决定的默认值`） | 同左：同一个共享 `scope` 与 `setScopeOverride`，没有独立的 `historyScope`（也不放徽章/总条数/按钮） |
| 分段行 | `SegmentBar`（`ti-seg`） | 待审核 / 已操作 / 失败 / 历史 | 同左，历史不带数字角标 |
| 状态行 | `ErrorLine`（`ti-error`）+ `ti-hint` | 「状态同步失败：…（每 N ms 重试）」 | 「历史加载失败：…」；加载中统一「正在加载…」 |
| 列表 | `ti-card` 列表 / `EmptyState`（`ti-empty`） | 空态标题 + 说明 | 同左，无内联「重试」按钮 |

历史数据源是 `PaneTab` 内调用的 `useHistoryFeed({ enabled, scope, sessionId, reloadToken })`，
不再单独渲染 `HistoryView`：`scope` 就是四个分段共用的那一个
（`scopeOverride ?? (sessionHasContent === false ? 'all' : 'session')`），`PaneTab` 里没有第二套
scope state，`ScopeSeg` 的 `value` / `onChange` 也不再按 `historyOpen` 分支；进入/离开历史只改
`enabled` 与分段主体，不改 scope、不触发重拉。作用域切换先清空列表与游标再拉第一页（旧 scope 的
条目不会串台）；分段来回切换不重挂载、不重复请求、保留已加载页；刷新保留当前列表重拉第一页；
每个请求带递增 `requestId`，作用域切换或新请求让在途响应失效。分页仍用 `before=seq`，跨页按
`key`（`seq:id`）去重。历史卡片只读：不渲染批准/拒绝/回答/应用按钮，只保留「跳到来源会话」
「打开引用文件」导航，结局徽章复用 `Badge`（`ti-badge-*`），失败/结果文本用 `ti-error` /
`ti-detail`。

默认作用域的判定集中在 `sessionHasContentSignal(snapshot, sessionsState, sessionId)`：

1. snapshot 回显的 `sessionId` 与面板会话一致时，采信 Host 的 `sessionHasContent`（`true`/`false`）；
2. 否则退回 DSH 会话列表投影的 `blank`（`blank === false` = 已有内容），老 Host 也能得到正确默认值；
3. 两者都拿不到时按“已有内容”处理（`null` → 本会话），不把老服务/冷会话的用户全当新会话。
   没有会话 id 时按「全部会话」。默认值只在 `scopeOverride === null` 时生效，用户手动点选永远优先；
   snapshot 轮询让 `sessionHasContent` 从 `false` 变 `true` 时，未被覆盖的默认值自然从「全部会话」
   切到「本会话」。

i18n：`LOCALE_NS = 'todo-inbox'`，`locale.register('todo-inbox', 'zh', {})`（中文即源码键，空字典
让 `L()` 回退到原文）、`locale.register('todo-inbox', 'en', EN_TEXT)`，再 `locale.bind('todo-inbox')`
取 `boundTranslate`；监听 `locale/change` 与 `<html lang>` 变化触发重渲染，`useLocaleTick()` 把语言
版本接进组件。

### 1.7 组件边界与降级

两个组件互相只通过 `ctx.get('userAttention')` / `ctx.get('todoInbox')` 松耦合，任何一侧缺失都不
影响另一侧加载。

| 场景 | 注意力侧 | 待办侧 |
|---|---|---|
| 都在 | 完整门控 + 超时策略 + 入队 | 自动入队、待办页、`todo_execute` 正常；bundle 设置页由注意力组件提供 |
| 只开注意力 | 正常 | 无消费者：`todoInboxKeepsRejections()` 返回 `false`，模型说明不再提待办；配置页隐藏待办块并提示「超时行为固定为拒绝」 |
| 只开待办 | `ctx.get('userAttention')` 为空 → `attentionAvailable: false` | `effectiveAutoEnqueue()` 回退本行 `autoEnqueue`；`plugins.bundle.config` 出现橙底说明（`.ti-config-warning`，`#f59e0b`）；已有待办仍可复核/执行 |

需要留意的实现细节：**「只开注意力」时配置里的 `enqueue` 并没有被改写成 `reject`**，而是没有
`todoInbox` 订阅者，`event.enqueue` 发出去也没人接收——行为上等价于拒绝。界面文案
（「超时行为固定为拒绝」）与 Host 的 `finalizeRejection → emitRejection` 路径共同保证这一点，但
`GET /api/user-attention/config` 仍会原样返回 `approvalTimeoutAction: 'enqueue'`。若以后有第二个
订阅者，语义会随之改变。

同理，「只开待办」时不会凭空产生新待办：入队依赖注意力的 `onRejection`，没有注意力组件就没有拒绝
事件源（其他插件若实现同名服务，也会被 `effectiveAutoEnqueue()` 认到）。

## 2. 工具与 API 参考

本节把 AI 可调用的三个工具、两个组件注册的 HTTP 路由、两个 Host 服务接口并列成契约表；门控与执行
的安全顺序集中在第 4 节。

### 2.1 三个 AI 工具

| 工具 | 参数 | 输出要点 | 限制 |
|---|---|---|---|
| `todo_list` | `limit?`（默认 30） | `sessionId, total, pending, approved, failed, items[], message`；`message` 每条含状态、工具名、目的（`requestReason`）与参数预览（各截断） | 只列**当前会话**；`limit` 夹在 1–30；完整参数需 `todo_get` |
| `todo_get` | `id`（必填） | `id, todo, message` | 跨会话、不存在一律抛错；不返回答案 |
| `todo_execute` | `id` 或 `ids`（二选一，批量 ≤10）、`mode?: serial\|parallel`、`continueOnError?` | 单条：`ok, id, kind, toolName, executed, answers?, outputPreview?, message`；批量：`items[], succeeded, failed, skipped, message` | 只接受 id；必须由来源会话的存活 AI 调用；并行允许包含重启类工具 |

三者都要求 `requireCallerAgent(exec)`：`exec.agent` 存在且 `ctx.agents.get(agent.id) === agent`
（当前存活），否则抛「需要当前存活的调用 AI」。

三者进入 `execute()` 后的第一件事都是 `purgeRejectedTodos()`：先把用户已拒绝（`rejected`）的条目
归档删除（`outcome='rejected'`），再执行原来的鉴权 / 查询 / 执行逻辑；因此被拒绝的条目不会出现在
`todo_list` / `todo_get` 的结果里。

分支复制来的条目（`originSessionId` 非空且不等于当前会话）在 `todo_list` 的条目里多一行
「来源：分支复制（原会话 xxxxxxxx；执行时回原会话日志取参数）」，`todo_get` 的 `message` 多一行
「来源：由分支对话复制自原会话 xxxxxxxx」；`summarizeTodo` / `detailTodo` 投影也带
`originSessionId`（`null` 表示本会话产生）。

`todo_execute` 的执行前置检查（顺序即拒绝顺序）、通过后的落盘与执行链路、重启类工具的状态落法
（`RESTART_TOOL_PATTERN`）、批量预检与 `serial` / `parallel` 语义统一见 4.3。

### 2.2 /api/user-attention/* 路由表

全部 `ctx.inject(['connection'])` 后经 `connection.fetch.register` 注册，`requestBody: 'buffered'`，
响应统一 `content-type: application/json; charset=utf-8`、`cache-control: no-store`。

| 方法 | 路径 | 请求 | 成功响应 | 错误 |
|---|---|---|---|---|
| GET | `/api/user-attention/state` | — | `ok, version: 2, now, instanceId, state, manual, since, lastActivityAt, idleTimeoutMs, awayRetryIntervalMs, awayRetryCount, pollIntervalMs, nextAwayAt, alertSeq, alert, rejections:{seq,count,items}` | 500 |
| POST | `/api/user-attention/ack` | `{ upto }` 非负安全整数 | `{ ok, rejections }` | 400 `upto must be a non-negative safe integer` |
| POST | `/api/user-attention/activity` | `{ kind }` ∈ `pointermove/pointerdown/keydown/wheel/touchstart/focus/visible` | `{ ok, state, manual, lastActivityAt, alertSeq }` | 400 列出允许值 |
| POST | `/api/user-attention/set` | `{ state }` ∈ `online/away/offline` | `{ ok, state, manual }` | 400 |
| GET | `/api/user-attention/config` | — | 全部生效字段 + `todoAvailable` / `todoEnabled` / `writable` | 500 |
| POST | `/api/user-attention/config` | 7 个可写字段任意组合 + 可选 `force` | 写回后的字段值 | 400 字段非法或无字段；409 `PENDING_INTERACTIONS`；409 配置行不可用；500 写入失败 |

`ack` 会把 `ackedThrough` 前推到 `min(upto, rejectionSeq)` 并丢弃已确认项；内部最多保留 20 条，
`state` 只回传最近 5 条。`set` 在切到非 `online` 时写盘，切到 `online` 时删文件；切到 `offline`
会 `controller.wake('manual-offline')` 立即收尾所有等待中的门控。

### 2.3 /api/todo-inbox/* 路由表（含 purge）

| 方法 | 路径 | 请求 | 成功响应 | 错误 |
|---|---|---|---|---|
| GET | `/api/todo-inbox/state` | query `session`、`scope=all` | `ok, version: 1, now, instanceId, attentionAvailable, attentionState, config:{autoEnqueue,pollIntervalMs}, totals, viewTotals, sessionId, sessionHasContent, todos[]`（`todos[]` 额外带 `questions` 与问答类 `answers`） | 500 |
| GET | `/api/todo-inbox/history` | query `scope=session\|all`、`session`、`limit`（1–100，默认 30）、`before`（seq 游标） | `ok, scope, sessionId, total, items[], nextBefore` | 400（`scope=session` 缺 `session`）；500 |
| POST | `/api/todo-inbox/decide` | `{ id, action: approve\|reject\|answer\|restore, answers? }` | `ok, id, action, todo?, answers?, archived?, noticeInjected` | 400 参数/草稿答案非法；404 `NOT_FOUND`；409 `EXECUTING`；409 `REVIEW_ONLY`（仅复核条目）；409 `NOT_REJECTED`（restore 非 rejected）；409 `NOT_REJECTABLE`（拒绝非 pending/failed）；409 `REJECTED`（rejected 上 approve/answer）；400 类型与 action 不匹配；500 |
| POST | `/api/todo-inbox/apply` | `{ id }` | `ok, id, sessionId, message`；进入 handler 先归档删除所有 `rejected` | 400；404 `NOT_FOUND`；409 `NOT_READY`；409 `SESSION_NOT_LIVE`；500 `FOLLOWUP_FAILED` |
| POST | `/api/todo-inbox/purge` | —（无请求体；仅本机 UI 在 5 秒长按后调用，不需要审批） | `ok, todos`（删除条数）, `history`（清空条数）, `kept`（保留的执行中条数） | 500 |
| GET | `/api/todo-inbox/config` | — | `ok, autoEnqueue, pollIntervalMs, maxTodos, writable` | 500 |
| POST | `/api/todo-inbox/config` | `{ autoEnqueue?, pollIntervalMs? }` | 写回后的值 | 400 无字段/非法；409 配置行不可用；500 |

`state` 里的 `todos` 比模型侧多带 `questions`（客户端卡片要渲染选项）与问答类 `answers`
（「已操作」只读展示与「返回修改」预填的草稿；模型侧 `summarizeTodo` / `detailTodo` 仍不含答案）；
两者都带 `originSessionId`（分支复制来源，`null` 表示本会话产生）。`attentionAvailable` 由
`typeof attention.onRejection === 'function'` 判定，`attentionState` 取
`attention.snapshot().state`。

`decide` 分支（1.4.3-hotfix.10）：

- `approve`：只置 `approved` + `decidedAt`，等 `todo_execute` 或 `apply` 触发执行；
- `answer`：必须过 `validateAnswers()`（完整答案）；
- `reject`：`pending` → 置 `rejected` + `decidedAt`，持久化，**不删除、不归档、不注入通知**；
  问答类可带 `answers` 草稿，经 `validateDraftAnswers()` 宽松校验（允许没答完）后存入
  `todo.answers`。`failed` 的拒绝保持旧行为：通知来源会话 + 归档删除（`outcome='failed'`）；
- `restore`：仅 `rejected` → 置回 `pending`、`decidedAt = null`，草稿答案原样保留；响应带
  `answers` 供客户端预填。

`sessionId` / `sessionHasContent` 是 hotfix.8 新增的默认作用域信号：请求带 `session=<id>` 时，
Host 回显该 id 并用 `detectSessionHasContent(liveSessionById(id))` 判断这个会话是否已经有对话内容，
返回 `true` / `false`；没有 `session` 参数或会话不在 `ctx.sessions.list()`（冷会话/已归档）时两者
分别为 `null` / `null`。判断优先读 `session.surface.nodes`（DSH 会话消息面，只收录会产生模型可见
消息的事件：全新会话为空，发出第一条消息后非空），拿不到 surface 时退回扫描 `turn/start` / 消息 /
工具事件——**不用“事件条数 > 0”**，否则标题、配置等与会话内容无关的日志事件会把空会话误判成已有
内容。老 Host 不返回这两个字段、老 Client 忽略新字段，双方都可单独升级。

**危险清理 `POST /api/todo-inbox/purge`（1.5.0）**：设置页红色长按框（见 1.6.2）确认后调用的唯一
入口，无请求体、与其它 `api/todo-inbox/*` 同源、不走审批。语义：

- 删除所有 `status` 为 `pending` / `approved` / `rejected` / `failed`（以及任何其它非执行中状态）
  的待办；
- **保留** `executing` / `restarting`：正在执行原工具的待办不能打断，保留项在执行结束后仍走原来的
  `removeTodo` / `archiveTodo` 路径，不会变成孤儿；保留条数在响应的 `kept` 里；
- 清空历史数组（`history.json` 的 `entries`）、`forkedSessions` 与一次性 approval grants
  （`grants.clear()`）；
- 立即 persist：`persist()` + `persistHistory()` 把空快照写回 `state.json` / `history.json`，
  **保留文件本身**（不 `rm` 文件）；
- 整段同步执行（Node 单线程内不会与其它请求/正在执行的待办交错），`historySeq` 计数器不回退
  （清空后新归档条目 seq 仍单调递增），接口幂等（空缓存上再调用返回
  `todos: 0, history: 0, kept: 0`）。

清空成功后**注意力 Client** 派发同源 window 事件 `dsh-todo-inbox:purged`，**待办 Client** 在轮询
effect 里监听并立刻 `store.refresh()`（`kick()`），因此页内待办列表下一帧就变空，不刷新整页；
事件不可用或派发失败只影响即时刷新。返回的错误体缺可读文本时，Client 按 `timeout` / `network` /
`bad-json` / `no-fetch` / `exception` 与 HTTP 状态给出兜底文案。

### 2.4 服务接口

`userAttention` 是注意力组件的唯一稳定面（`ctx.provide('userAttention', service)`）：

| 成员 | 入参 | 返回 / 行为 |
|---|---|---|
| `version` | — | `1`（= `ATTENTION_REJECTION_VERSION`） |
| `snapshot()` | — | `{ state, manual, since, lastActivityAt, idleTimeoutMs, awayRetryCount }` |
| `subscribe(fn)` | 函数 | 状态变化时收到 `{ previous, state, manual, since, lastActivityAt, reason }`；返回退订函数 |
| `onRejection(fn)` | 函数 | 每次最终拒绝收到拒绝事件；返回退订函数 |
| `todoAutoEnqueue()` | — | 本行 config 的 `todoAutoEnqueue === true` |

`onRejection(fn)` 收到的拒绝事件由 `buildAttentionRejection()` 构造，字段有上限（问题正文可能很长）：
`{ version, kind, reason, at, sessionId?, toolName?, callId?, requestReason?, questions? }`。
`questions` 经 `sanitizeQuestions()` 截断（最多 10 题、单题 2000 字、detail 4000 字、选项 20 个、
label 200 字、description 500 字，并保留 `intent.kind === 'plan-review'`）。**调用参数不进事件**
——待办按 `(originSessionId ?? sessionId) + callId` 回会话日志读取并做哈希校验（`originSessionId`
只出现在分支复制的待办上，见 1.5.3）。

`todoInbox` 是待办组件对其它 Host 插件的接口：

| 成员 | 入参 | 返回 |
|---|---|---|
| `version` | — | `1` |
| `autoEnqueueEnabled()` | — | `effectiveAutoEnqueue()` |
| `list(sessionId?, limit?)` | 会话 id 可选，`limit` 上限 100 | 摘要数组（`summarizeTodo`，不含答案） |
| `get(id)` | 待办 id | `detailTodo()` 或 `null`（含问题与参数预览，不含答案） |
| `takeApprovalGrant(callId)` | callId | `true` 表示消费掉一次授权 |
| `totals()` | — | `{ pending, approved, failed }` |

## 3. 状态与持久化

本包只落三份文件：注意力状态 `<home>/.user-attention/state.json`、待办状态
`<home>/.dsh-todo-inbox/state.json`、只读历史 `<home>/.dsh-todo-inbox/history.json`。三者都带
`version` 门槛、都用同一套写入机制，加载时都做校验与容量裁剪（见 3.4）。

### 3.1 state.json（注意力）

路径：`<home>/.user-attention/state.json`。`home` 解析顺序 = `config.home` →
`profileContext.home` → `DSH_HOME` → `~/.dsh`。

```json
{ "version": 1, "state": "away", "manual": true, "savedAt": 1730000000000 }
```

| 行为 | 规则 |
|---|---|
| 读取 | 仅当 `version === 1 && manual === true && state ∈ {away, offline}` 才恢复；损坏/缺失/`persistManualState: false` 一律忽略并记日志 |
| 写入 | 先写 `<file>.<pid>.<ts>.tmp` 再 `renameSync`；Windows 上目标已存在导致 rename 失败时回退为直接覆写 |
| 清理 / 失败 | 手动切回 `online` 时 `rmSync(stateFile, { force: true })`；写盘失败只记日志，不影响本次状态切换 |

### 3.2 state.json（待办）

路径 `<home>/.dsh-todo-inbox/state.json`（`home` 解析规则同 3.1），结构：

```json
{ "version": 1, "savedAt": 1730000000000,
  "todos": [ { "id": "todo-...", "status": "pending", "originSessionId": null, "...": "..." } ],
  "forkedSessions": [ "session-branch-id" ] }
```

| 规则 | 行为 |
|---|---|
| 版本 | `version !== 1`（`TODO_STATE_VERSION`）整份忽略并记日志 |
| 分支记录 | `forkedSessions` 保存已做过分支复制的分支会话 id（去重、最近 100 条，`TODO_LIMITS.forkedSessions`）；缺失/损坏按空数组处理，老版本 Host 读到该字段会忽略 |
| 答案校验 | 已存 `answers` 会用问题定义重新校验：`approved`（可执行）走严格 `validateAnswers()`；`pending` / `rejected` 的草稿走宽松 `validateDraftAnswers()`（允许没答完）；不合法则清空，避免坏数据被原样返回给模型 |
| 容量 | 加载后 `slice(-maxTodos)`，超出上限记日志 |

逐条修复（`sanitizeStoredTodo()`）与写入实现见 3.4，宿主重启/热重载后的对账见 3.5。

### 3.3 history.json

路径 `<home>/.dsh-todo-inbox/history.json`，与 `state.json` 分开持久化：

```json
{ "version": 1, "savedAt": 1730000000000, "seq": 42,
  "entries": [ { "seq": 42, "id": "todo-...", "kind": "question", "outcome": "answered",
                 "toolName": null, "sessionId": "...", "title": "...", "rejectReason": "offline",
                 "createdAt": 0, "decidedAt": 0, "executedAt": 0, "archivedAt": 0,
                 "argsPreview": null, "referencedFiles": [], "resultSummary": null,
                 "failReason": null, "questionCount": 1, "answersPreview": "...",
                 "questions": [ { "id": "q1", "question": "...", "options": [ { "label": "A" } ], "multiSelect": false } ],
                 "answers": [ { "id": "q1", "selected": [ "A" ], "custom": "..." } ] } ] }
```

| 规则 | 行为 |
|---|---|
| 版本 / 序号 | `version !== 1`（`TODO_HISTORY_VERSION`）整份忽略；`seq` 单调递增，重启后从文件里的 `seq` 继续 |
| 归档时机 | 执行成功 `executed`；**被拒绝的条目不立即归档**：`reject` 只置 `rejected`，直到点「应用操作」（`apply`）或 AI 调用 `todo_*` 工具时由 `purgeRejectedTodos()` 归档为 `rejected`；失败项被拒绝/删除 `failed`（立即）；重启对账 `restarted`；问题回答取回 `answered` |
| 结构化问答 | `questions` 用 `sanitizeTodoQuestions` 收敛；`answers` 里 `rejected` 用宽松 `validateDraftAnswers`、其余用 `validateAnswers` 重新校验；任一不可信则为 `null`；旧字段 `questionCount` / `answersPreview` 保留 |
| 去重 | 归档前按 `id` 去重：宿主热重载时新旧两个实例只保留第一次 |
| 容量 | 不做上限、全量保留；接口按 `before=seq` 分页，最新在前 |
| 只读 | 没有写路由，也不注册任何 AI 工具；`todo_list` / `todo_get` 看不到历史，模型无法恢复或重放 |
| 隔离 | `scope=session` 必须带会话 id 且只返回该会话；`scope=all` 每条自带 `sessionId`，客户端分开渲染与计数 |

### 3.4 写入与修复规则

- **写入**：临时文件 + rename（先写 `<file>.<pid>.<ts>.tmp` 再 `renameSync`）；Windows 上目标已
  存在导致 rename 失败时回退为直接覆写。注意力状态切换写盘失败只记日志，不影响本次状态切换。
- **加载修复（待办 `state.json`）**：逐条 `sanitizeStoredTodo()`：缺 `id/kind/sessionId` 丢弃；
  非法 `status` → `failed`；`executing` → `failed`（宿主中途重启，原工具副作用未知），但
  `executing + reloadExpected` 按 `executed` 归档。
- **答案校验与容量裁剪**：`state.json` 的规则见 3.2 表内的「答案校验」「容量」两行；`history.json`
  的问答重校验与归档去重见 3.3；本处不重复实现细节。

### 3.5 reloadExpected 对账

`plugin_manager` 执行前落 `reloadExpected: true`；安装/卸载本插件会热重载 Host，`load()` 据此把
`executing + reloadExpected` 归档为 `executed`（结果未知，需人工确认），不再误报 `failed`。

| 落盘状态 | 加载后的归档 / 修复 | 含义 |
|---|---|---|
| `executing`（非重启类工具，无 `reloadExpected`） | `failed` | 宿主中途重启，原工具副作用未知，需人工检查 |
| `executing + reloadExpected` | `executed` | 通过待办安装/卸载插件触发热重载，结果未知但不算失败 |
| `restarting` | `restarted` | 重启类工具执行中宿主消失，结果未知，需人工确认 |

## 4. 安全模型

本节汇总以下安全约束：调用参数**不落盘**（只持久化脱敏预览与哈希，执行时从会话日志读回并校验）；
授权**一次性**且有 TTL；执行与门控的**检查顺序本身就是安全边界**；待办与执行都受**会话归属**约束。

### 4.1 脱敏与哈希校验

- 待办只持久化脱敏参数预览（`argsPreview`）与参数哈希（`argsHash`）；完整参数永远只从来源会话
  `tool/call` 事件里读（`findToolCallArguments()`）。
- 执行前重算 `sha256Hex(call.arguments)` 与 `argsHash` 比对；日志里的工具名与记录不一致、参数不是
  合法 JSON、哈希不符，都在执行前置检查里拒绝（见 4.3）。
- 跨会话取参数时，`todo_execute` 按 `callSourceSessionId(todo)` = `originSessionId ?? sessionId`
  回对应会话日志读原始调用，哈希校验规则不变。

| 数据 | 上限 | 位置 |
|---|---|---|
| `argsPreview` | ≤1000 字 | 待办记录 / 模型可见摘要 |
| `referencedFiles` | 最多 8 条引用文件路径 | 待办记录 |
| `requestReason` | ≤400 字 | 待办记录（原始审批理由） |
| `failReason` | ≤500 字 | 待办记录（失败原因） |
| 回注摘要 `summary` | 截断 120 字 | `agent.inject` / `agent.followup` |
| 拒绝事件里的 `questions` | 最多 10 题、单题 2000 字、detail 4000 字、选项 20 个、label 200 字、description 500 字；保留 `intent.kind === 'plan-review'` | `sanitizeQuestions()` |

**调用参数不进拒绝事件**：注意力组件只负责告诉待办「哪个会话、哪个 `callId` 被拒绝了」，参数要靠
来源会话日志读回，避免审批参数被复制到另一份持久化文件里。

### 4.2 一次性 grant 与 TTL

- `TODO_GRANT_TTL_MS = 5 * 60 * 1000`：grant 只覆盖一次工具执行的审批窗口，不是长期授权；且只对
  本次执行生成的新 `callId` 有效。
- `registerGrant(callId, todoId, sessionId)` 写入 `{ todoId, sessionId, at, expiresAt }` 前先
  `sweepGrants()`（清过期，>100 条时保留最新 100）。
- `takeApprovalGrant(callId)` 取出即删；已过期返回 `false`；成功时日志
  `approval grant consumed for todo <id>`。
- 注意力侧 `takeTodoGrant(callId)` → `ctx.get('todoInbox').takeApprovalGrant(String(callId))`，
  命中即返回 `'allowed-once'`（gateway 视为已批准）并记日志
  `approval allowed once by a todo-inbox grant: <callId>`；服务缺失时按无待办插件处理。
- 执行侧 `finally` 里 `removeGrant(callId)`，保证授权不残留；`POST /api/todo-inbox/purge` 会清空
  全部一次性 grant。

### 4.3 执行前置检查顺序

`todo_execute` 的检查顺序即拒绝顺序：

| 检查 | 失败信息 |
|---|---|
| `todo.status === 'executing'` | 「正在执行中，请勿重复调用」 |
| `todo.status === 'rejected'` | 「已被用户拒绝；请让用户先点『返回修改』再执行」（正常入口已清理，属兜底） |
| `todo.status === 'pending'` | 「还没有被用户批准/回答」 |
| `kind === 'question'` | 不重放工具：取回 `answers` 后删除待办，`executed: false` |
| `reviewOnly` | 「标记为仅复核…只能拒绝或删除」 |
| `callId` / `toolName` 缺失 | 无法定位原始调用 / 无法执行 |
| 会话日志里找不到 `tool/call` | 「待办保留，可检查后重试」 |
| 日志工具名与记录不一致 | 「拒绝执行」 |
| `argsHash` 不符 | 「原始参数哈希校验失败」 |
| 参数不是合法 JSON | 「拒绝执行」 |

通过后：生成新 callId `todo-exec-<id>-<ts>` → `status = 'executing'` 落盘 →
`registerGrant(callId, todo.id, sessionId)` →
`ctx.tools.execute({ callId, name, arguments, agent, signal })`。日志查找的会话是
`callSourceSessionId(todo)` = `originSessionId ?? sessionId`：分支复制来的条目回原会话日志取参数，
执行仍发生在分支会话（`agent` 是调用方），`todo.sessionId` 的归属校验不变。执行结果按 1.5.5 的
三类「假成功」检查保留 `failed`，全部通过才删除待办。

重启类工具（`RESTART_TOOL_PATTERN = /^restart_dsh_(?!status$|force_safeguard$)/`）把状态落成
`restarting` 而不是 `executing`：宿主重启后 `load()` 会把它归档为 `restarted`（结果未知），不会
误报 `failed`；若工具返回了（例如 `force_dangerous` 未上保险），按 `failed` 保留。
`restart_dsh_status`（只读）与 `restart_dsh_force_safeguard`（只给下一次调用上保险）不按重启处理。

批量执行（`ids`）：先对全部条目做预检（存在、同会话、非 `executing`/`restarting`、已操作
（`approved`）/已回答、非仅复核、`callId`/`toolName` 齐备），任何一条不合格就整批拒绝，不执行任何
条目。`serial`（默认）按传入顺序执行，首个失败后停止并把后续标记 `skipped`
（`continueOnError: true` 可继续）；`parallel` 用 DSH 官方
`agentLoop.config.maxParallelToolCalls.get()` 作为并发上限（读不到回退 10，不叠加自定义上限），
并行可以包含重启类工具（宿主可能中途重启）。执行失败不再抛错，而是在 `items[]` 里给 `ok: false`
与 `error`。

### 4.4 approval 门控顺序

`handleApproval()` 的判定顺序（顺序本身是安全边界）：

| 顺序 | 条件 | 行为 |
|---|---|---|
| 1 | `takeTodoGrant(req.callId)` 为真 | 直接返回 `'allowed-once'`，**不呈现**第二张审批卡，也不受离线门控拦截 |
| 2 | `state === 'online'` | `trackOnlineInteraction()`：只记账 `activeInteractions`，不改变官方超时/撤回语义 |
| 3 | 其他 | `attentionGate()` |

`handleQuestion()` 没有 grant 分支（问题不重放），只有「在线」与「门控」两条路径。

### 4.5 reviewOnly（仅复核）

- 工具名匹配 `^todo_`、`^ask_user_question$`、`^workflow$`、`^subagent_fork$` 的调用被标记为
  仅复核：可以拒绝，不能执行。`restart_dsh_*` 自 1.4.3-hotfix.1 起已放开，可正常批准执行
  （状态落法见 4.3）。
- `/api/todo-inbox/decide` 对 `reviewOnly` 条目直接返回 `409 REVIEW_ONLY`（此前只是执行时拦）。
- `todo_execute` 的前置检查里 `reviewOnly` 失败信息是「标记为仅复核…只能拒绝或删除」；批量预检同样
  要求「非仅复核」。
- 待办 Client 的高亮条只在最新一条 `newest.reviewOnly !== true` 时显示应用按钮。

### 4.6 跨会话边界

- `todo_get` / `todo_execute` 强制 `todo.sessionId === 调用 AI 的会话`：跨会话、不存在一律抛错。
- `apply` 需要来源会话存活：来源会话没打开（或已归档）时返回 409 `SESSION_NOT_LIVE`。
- 回注（`decide` / `apply`）只针对 `todo.sessionId`；分支复制来的待办属于分支会话，原会话不会被唤醒。
- 分支复制来的条目 `sessionId` 是分支、`originSessionId` 是原会话：执行仍发生在分支会话（`agent`
  是调用方），原始参数从原会话日志读；原会话已归档/日志不可读时执行失败并保留待办（见 1.5.3）。
- `GET /api/todo-inbox/state` / `history` 用 `scope=session` / `scope=all` 严格隔离，
  `scope=session` 必须带会话 id；历史同样按 scope 返回，前端分开渲染与计数。
- 冷会话（Host 未加载）不在 `ctx.sessions.list()`，`sessionHasContent` 为 `null`，客户端按「已有
  内容」兜底（见 1.6.3、7.5）。
- `POST /api/todo-inbox/purge` 与其它 `api/todo-inbox/*` 同源、只由本机 UI 在 5 秒长按后调用、
  不走审批：这是「危险操作只能由人确认，模型无法通过工具触发」的实现方式。

## 5. 配置字段

两条 Loader 行各有自己的 config；`normalizeConfig()` 会把越界值夹紧、类型不对回落默认。下表「补丁值」
是 `cordis.patch.yml` 里实际写死的值，用户保存设置时写回 profile 覆盖层。

### 5.1 注意力行（`id: user-attention`）

| 字段 | 类型 | 代码默认 | 取值范围（`normalizeConfig`） | 补丁值 | 说明 |
|---|---|---|---|---|---|
| `idleTimeoutMs` | integer | 300000 | 1000 – 86400000 | 300000 | 在线判定为暂离的空闲阈值 |
| `awayRetryCount` | integer | 2 | 0 – 5 | 5 | 暂离提醒次数；`maxAttempts = 值 + 1` |
| `awayRetryIntervalMs` | integer | 30000 | 2000 – 600000 | 30000 | 每次等待/提醒间隔 |
| `tickMs` | integer | 5000 | 1000 – 60000 | 5000 | 空闲看门狗周期 |
| `pollIntervalMs` | integer | 1500 | 500 – 10000 | 1500 | 下发给 Client 的轮询间隔（Client 侧再夹到 1000–5000） |
| `initialState` | enum | `online` | `online` / `away` / `offline` | `online` | 无手动状态可恢复时的初始态 |
| `persistManualState` | boolean | true | true / false | true | 是否把手动 away/offline 写盘 |
| `approvalTimeoutAction` | enum | `reject` | `reject` / `enqueue` | `enqueue` | 批准超时后是拒绝还是拒绝并进待办 |
| `questionTimeoutAction` | enum | `reject` | `reject` / `enqueue` | `enqueue` | 同上，作用于提问 |
| `preferRecommended` | boolean | false | 仅 `=== true` 生效 | false | 有推荐项时按推荐项自动回答 |
| `todoAutoEnqueue` | boolean | true | true / false | 未写（即默认 true） | bundle 设置页的「自动入队」总开关 |
| `home` | string | undefined | 非空字符串 | 注释掉 | 覆盖 `~/.dsh` 等 home 解析 |

补丁值刻意对齐了旧 1.4.2 单包行用过的设置（`awayRetryCount: 5`、两个超时策略 `enqueue`），所以升级后
行为不变；代码默认值仍是保守的 `2` 与 `reject`。README 的「配置」节列的是**补丁值**（生效值），与本表
「补丁值」列一致，口径对照见 [README](./README.md)。

### 5.2 待办行（`id: todo-inbox`）

| 字段 | 类型 | 代码默认 | 取值范围 | 补丁值 | 说明 |
|---|---|---|---|---|---|
| `autoEnqueue` | boolean | true | true / false | true | 本行的入队开关；注意力组件存在时被其 `todoAutoEnqueue` 覆盖 |
| `pollIntervalMs` | integer | 2000 | 1000 – 10000 | 2000 | 下发给 Client 的轮询间隔 |
| `maxTodos` | integer | 200 | 10 – 1000 | 200 | 内存与磁盘最多保留条数，超出丢弃新拒绝 |
| `home` | string | undefined | 非空字符串 | 注释掉 | 覆盖 home 解析 |

### 5.3 合并设置页写回哪一行

| 控件 | 写回目标 | 路径 |
|---|---|---|
| 空闲时间、暂离重试、自动入队勾选、批准/提问超时、优先推荐 | **注意力行** | `POST /api/user-attention/config`，`editor.edit(entry, ...)`；行定位先按 `options.name === '@local/dsh-user-attention-core'`，回退 `options.id === 'user-attention'` |
| 危险区「释放并删除待办缓存」长按按钮 | **不改任何配置** | `POST /api/todo-inbox/purge`（无请求体）；只清缓存，不触发插件重载，也不写 profile 补丁 |
| 待办行的 `autoEnqueue` / `pollIntervalMs` | **待办行** | `POST /api/todo-inbox/config`（`options.name === '@local/dsh-todo-inbox'`，回退 `options.id === 'todo-inbox'`） |

关键事实：`todoAutoEnqueue` 是**注意力行**的字段，不是待办行的字段。合并页保存时并不会顺手改待办
行——`attention/index.js` 里的 `findTodoConfigTarget()` 定义了但从未被调用（早期「一次保存写两行」
方案的残留）。待办行目前也没有 UI 入口：`todo-inbox/client.js` 的 `ConfigEntry` 同样只定义未注册，
只有 `/api/todo-inbox/config` 这条 Host 路由可用（测试直接打路由验证）。

### 5.4 409 / force 语义

`409 / force` 语义（仅注意力行）：

| 条件 | 响应 |
|---|---|
| `activeInteractions.size > 0` 且 `body.force !== true` | `409 { ok: false, code: 'PENDING_INTERACTIONS', pending: N, error: '还有 N 个人工交互正在等待处理；保存配置会取消它们。确认后以 force:true 重试。' }` |
| 带 `force: true` | 正常写回（保存会重载插件，等待中的交互被取消） |
| `configEditor` 或目标行不可用 | `409 当前 profile 不支持写回注意力组件配置` |
| 字段非法（非整数、越界、枚举不符） | `400`，逐字段给原因 |
| 一个可写字段都没给 | `400` 列出允许字段 |

客户端收到 409 后把 `pending` 显示成「仍然保存（会取消 N 个等待中交互）」按钮，点击即以
`force: true` 重发。待办行的 `POST /config` 没有 `force`。

## 6. 测试与验证

### 6.1 语法与单测

| 检查 | 命令 | 结果 |
|---|---|---|
| 四个 JS 语法 | `node --check components/attention/index.js`、`components/attention/client.js`、`components/todo-inbox/index.js`、`components/todo-inbox/client.js` | 4/4 通过 |
| 注意力 Host | `node components/attention/test/host-state-machine.test.mjs` | `tests 19 / pass 19 / fail 0` |
| 待办 Host | `node components/todo-inbox/test/todo.test.mjs` | `tests 49 / pass 49 / fail 0` |
| 待办 Client 渲染 | `node components/todo-inbox/test/client.test.mjs` | `tests 29 / pass 29 / fail 0` |

前两条测试都用 `node:test` + 自带最小 Cordis 上下文（捕获路由、监听器、`ctx.effect` 清理与
`provide` 的服务），不启动真实 Host。

- **注意力套件**覆盖配置夹紧、离线即时拒绝、手动状态跨 `apply` 持久化、`DSH_HOME` 回退、损坏状态
  忽略、拒绝摘要分页与 ack、409 + force、`options.id` 回退、问题截断、拒绝事件构造、
  `userAttention` 服务、grant 一次性消费、推荐项与超时策略、配置写入同一行。
- **待办套件（基础）**覆盖配置夹紧、review-only 识别、脱敏与指纹、入队/批准/应用/`todo_execute`
  一次执行、问题答案取回、拒绝注入、跨会话与哈希校验、`pwsh` 非零退出保留 failed、批量串行/并行
  与首个失败即停、历史分页与 scope 隔离、`restarting` 对账、`todo_list` 目的/参数预览、历史结构化问答、
  拒绝时间、`reloadExpected` 热重载对账。
- **待办套件（1.4.3-hotfix.10 rejected 状态机）**：`reject` 后仍在列表且 `status='rejected'`、
  不归档、不注入通知、重新加载后保持 rejected；`restore` 回 `pending` 并保留问答草稿、对非 rejected
  返回 409；`apply` 与 `todo_list` / `todo_get` / `todo_execute` 三个入口归档删除 rejected；
  `failed` 的拒绝仍直接归档（`outcome='failed'`）；`validateDraftAnswers` 的宽松边界。
- **待办套件（会话信号与分支复制）**：`detectSessionHasContent`（无消息 / 只有无关日志事件 / 有消息
  / surface 缺失回退 / 冷会话未知）与 `state` 接口的 `sessionHasContent`（无消息 false、有消息无待办
  true、缺 session 参数 null、冷会话 null）；分支对话复制用例覆盖普通 `session/created`
  （无 `parentSession` / `isSeeded=false`）不复制、分支创建只复制 `pending` + `approved`
  （`failed` / `rejected` / `executing` 不复制）、副本新 id 且 `sessionId` / `originSessionId`
  正确、字段原样保留且原会话不受影响、重复 `session/created` 幂等、分支已有待办时不复制、
  `originSessionId` 与 `forkedSessions` 落盘后重新加载仍保留、`todo_execute` 优先按
  `originSessionId` 回原会话取原始调用（不带该字段时回退 `sessionId` 并给出原来的
  「无法从来源会话日志定位原始工具调用」错误）。
- **待办套件（1.5.0 purge）**：`pending` / `approved` / `rejected` / `failed` 全部删除（用例先把
  种下的 `pending` 在执行期间改成 `rejected`）、挂住原工具时 `executing` 条目被保留且 `kept=1`、
  历史条数与 `forkedSessions` 被清空、响应计数 `{ ok, todos, history, kept }` 精确匹配、
  `state.json` / `history.json` 立即写空但文件本身仍存在、重新加载后内存与磁盘都为空、空缓存上重复
  调用幂等（全 0）、执行期登记的一次性 approval grant 被清空（`takeApprovalGrant` 返回 `false`）、
  被保留的执行中条目恢复后仍能正常收尾并在历史里留下 `executed`。

Client 渲染测试用 `new Function` 把 `client.js` 载入 Node（注入假 `window.__ModuleLoader__`、假
`fetch` 与最小 hooks 运行时），递归渲染 `PaneTab` 后断言：历史与收件箱的头部顺序（标题行 →
作用域行 → 分段行）、刷新按钮只出现在 `ti-header-actions`、历史作用域行只有一个 `ScopeSeg`、
四个分段共用同一个 scope（收件箱选「全部会话」进历史后请求 `scope=all` 且高亮不变；历史里选
「本会话」按 session + sessionId 重拉，切回收件箱仍高亮；只切分段不产生请求也不作废在途响应；
共享 scope 切换时旧响应仍作废）、会话新旧默认值（新会话默认 all；已有内容但无待办默认 session 且
历史按 scope=session 拉取；Host 信号从 false 变 true 时默认值自然切换；手动覆盖后信号翻转仍保持；
旧 Host 缺字段时用会话列表 blank 兜底、再不行按已有内容兜底）、状态请求带 `session=` 的契约、
加载/空/错误态位置、刷新保留列表、`before=seq` 分页与跨页去重、历史卡片只读且带生成/审批/审批结果
/选项/自定义回答、分段来回切换不重复请求；1.4.3-hotfix.10 新增：分段文案「待审核 / 已操作 / 失败 /
历史」与 approved+rejected 合并计数、已操作空态改名、rejected 卡片没有单次应用/拒绝且「返回修改」
调用 `store.restore`、rejected 问答卡片只读展示选项与自定义文本、待审核表单用 `todo.answers` 预填
（选项与自定义）且拒绝提交当前草稿。另直接渲染 `DockBar` 断言按钮文案「应用操作」仍只对最新一条
调用 `store.apply`，以及最新一条仍在待审核 / 仅复核 / 无待办时不出应用按钮、卡片「单次应用（发送
执行请求 / 发送回答请求）」只应用自身；提问表单（`QuestionForm`）断言单选题把「自定义」并入
`.ti-options` 的 radio 行且与预设选项互斥（选选项清自定义、填自定义清选项、提交只带一种）、多选题
仍可同时保留选项与自定义、单选无选项时只靠自定义也能提交，并用源码提取的 `validateLocalAnswers`
锁住单选互斥防御（多选不受影响）。

上述 Host 套件与 Client 渲染两套测试在本文写作时重新执行，结果与记录一致。

### 6.2 隔离 profile 配置检查

在隔离 profile（临时 `DSH_HOME`）中验证过，两条行都组合成功，取值如下：`user-attention` →
`@local/dsh-user-attention-core`（`awayRetryCount: 5`、`approvalTimeoutAction: enqueue`、
`questionTimeoutAction: enqueue`）与 `todo-inbox` → `@local/dsh-todo-inbox`（`autoEnqueue: true`、
`pollIntervalMs: 2000`、`maxTodos: 200`）。

### 6.3 隔离 profile 启动 Web

启动日志按代码里的固定前缀逐条出现：

```text
[user-attention] host active — state=online, idleTimeoutMs=300000, retries=5
[todo-inbox] subscribed to userAttention rejection events
[todo-inbox] host active — autoEnqueue=true, pollIntervalMs=2000, todos=0, home=<tmp>
```

`retries=5` 说明补丁值生效（代码默认是 2），`autoEnqueue=true` 说明待办行配置生效。

同一次启动里，boot HTML 的 `window.__DSH_BOOT__` 有两条 client 条目：
`@local/dsh-user-attention-core` 与 `@local/dsh-todo-inbox`；两者的 combo 请求
`/plugins/??...&rev=...` 均返回 `200`，说明嵌套 `node_modules` 里的 `client.js` 被正确登记并参与
打包分发。

### 6.4 界面手工验收清单

6.1–6.3 已覆盖语法、配置与启动；视觉、焦点与交互层按下面这份清单验收：

1. 打开插件卡设置页，确认待办组件可用且「启用待办收件箱（自动入队）」勾选时，超时设置之后出现
   红色虚线框；取消勾选后整块消失，重新勾选后回来（不保存、刷新页面后仍按保存值决定）；
2. 鼠标按住按钮：文字变成「继续按住… x.x 秒」、底部进度条从左向右推进、按钮有脉冲；不到 5 秒
   松手 → 不发请求（Network 里没有 `api/todo-inbox/purge`）、进度归零、文字恢复；
3. 按住 5 秒：进度走满后自动发一次 `POST api/todo-inbox/purge`（无请求体），按钮变「正在释放…」
   并禁用；返回后显示「已释放并删除缓存（N 条待办、M 条历史；保留 K 条执行中）」；
4. 触摸（或 DevTools 设备模拟）：手指按住同样计时；按住后把手指拖出按钮再松开 → 取消且不发请求；
   按住时把页面切到别的标签再回来 → 取消；
5. 右栏待办页打开着执行一次：成功后列表在 1–2 秒内变空（`dsh-todo-inbox:purged` 事件触发
   `refresh()`），页面不整页刷新、右栏 tab 不关闭；若有执行中的待办，它仍在列表里且状态保持
   「执行中」；
6. 断网或让 Host 返回 500：显示「释放并删除缓存失败：…」，按钮恢复可再次长按；
7. 系统开启「减少动态效果」（或 DevTools 里模拟 `prefers-reduced-motion: reduce`）：按住时不再有
   脉冲缩放感，进度条瞬间跳变（无过渡），其余行为不变。

## 7. 附录

### 7.1 打包产物与校验

打包工具为 `tools/dsh-bundle-pack/`（开发中）；命令、参数与环境变量以其 README 为准，本文只记录
产物格式与校验要点，不复述操作步骤。

产物是**一个**自包含 tgz，格式上有三个固定事实（机制见 1.2）：

- `package/package.json` 是 bundle 清单：`dsh.bundle.patch` 指向 `cordis.patch.yml`，
  `dependencies` 精确钉住两个组件版本（不带 `^`/`~`）；
- 两个组件各自以完整包的形式躺在 `package/node_modules/@local/<name>/`（含各自的 `package.json`、
  `client.js`、`locale`、`icon.svg`）；
- `bundledDependencies` 只出现在打包中间产物与最终 tgz 的 `package/package.json` 里，源码
  `package.json` 保持干净。

产物文件名：`local-dsh-user-attention-and-todo-inbox-1.5.0.tgz`（随仓库 `releases/` 分发，文件集合与打包方式见本节上述流程）。

内容校验：

```bash
tar -tzf DSH-Plugin-tgz/local-dsh-user-attention-and-todo-inbox-1.5.0.tgz | wc -l   # 22
tar -tzf DSH-Plugin-tgz/local-dsh-user-attention-and-todo-inbox-1.5.0.tgz \
  | grep 'package/node_modules/@local/'
```

实测清单（22 项）：bundle 根 8 项（`package/package.json`、`package/cordis.patch.yml`、
`package/icon.svg`、`package/locale/{zh,en}.json`、`package/README.md`、`package/TECHDOC.md`、
`package/HISTVER.md`），加上 `package/node_modules/@local/` 下两个组件包各 7 项（`index.js`、
`client.js`、`package.json`、`icon.svg`、`locale/{zh,en}.json`、`README.md`）。注意 tgz 内**没有**
`package/components/`：组件是以包的形式躺在 `node_modules/@local/` 下的。

### 7.2 错误码表

下表汇总两个组件 HTTP 路由返回的状态码与业务 `code`（只从路由表的「错误」列抽取，不补推断文案；
`code` 列或说明列里的「—」表示源文档只给状态码 / `code`，未给说明文字）。

| 状态码 | code | 触发点 | 说明 |
|---|---|---|---|
| 400 | — | `POST /api/user-attention/ack` | `upto must be a non-negative safe integer` |
| 400 | — | `POST /api/user-attention/activity` | `kind` 不在允许值内，响应列出允许值 |
| 400 | — | `POST /api/user-attention/set` | `state` 不在 `online` / `away` / `offline` 内 |
| 400 | — | `POST /api/user-attention/config` | 字段非法（非整数、越界、枚举不符），逐字段给原因 |
| 400 | — | `POST /api/user-attention/config` | 一个可写字段都没给，响应列出允许字段 |
| 400 | — | `GET /api/todo-inbox/history` | `scope=session` 缺 `session` |
| 400 | — | `POST /api/todo-inbox/decide` | 参数 / 草稿答案非法 |
| 400 | — | `POST /api/todo-inbox/decide` | 类型与 `action` 不匹配 |
| 400 | — | `POST /api/todo-inbox/apply` | — |
| 400 | — | `POST /api/todo-inbox/config` | 无字段 / 字段非法 |
| 404 | `NOT_FOUND` | `POST /api/todo-inbox/decide` / `apply` | 待办不存在 |
| 409 | `PENDING_INTERACTIONS` | `POST /api/user-attention/config` | 还有人工交互在等待；需 `force: true` 重试 |
| 409 | — | `POST /api/user-attention/config` | 当前 profile 不支持写回注意力组件配置 |
| 409 | — | `POST /api/todo-inbox/config` | 配置行不可用 |
| 409 | `EXECUTING` | `POST /api/todo-inbox/decide` | 待办正在执行中 |
| 409 | `REVIEW_ONLY` | `POST /api/todo-inbox/decide` | 仅复核条目 |
| 409 | `NOT_REJECTED` | `POST /api/todo-inbox/decide`（`restore`） | `restore` 的目标不是 `rejected` |
| 409 | `NOT_REJECTABLE` | `POST /api/todo-inbox/decide`（`reject`） | 拒绝的目标不是 `pending` / `failed` |
| 409 | `REJECTED` | `POST /api/todo-inbox/decide`（`approve` / `answer`） | 条目已 `rejected` |
| 409 | `NOT_READY` | `POST /api/todo-inbox/apply` | — |
| 409 | `SESSION_NOT_LIVE` | `POST /api/todo-inbox/apply` | 来源会话不存活（未打开 / 已归档） |
| 500 | — | `GET /api/user-attention/state` | — |
| 500 | — | `GET /api/user-attention/config` | — |
| 500 | — | `POST /api/user-attention/config` | 写入失败 |
| 500 | — | `GET /api/todo-inbox/state` | — |
| 500 | — | `GET /api/todo-inbox/history` | — |
| 500 | — | `POST /api/todo-inbox/decide` | — |
| 500 | — | `POST /api/todo-inbox/purge` | — |
| 500 | — | `GET /api/todo-inbox/config` | — |
| 500 | — | `POST /api/todo-inbox/config` | — |
| 500 | `FOLLOWUP_FAILED` | `POST /api/todo-inbox/apply` | 向来源会话 `followup` 失败 |

不走 HTTP 的内部错误标识与失败标记：

| 标识 | 出现位置 | 含义 |
|---|---|---|
| `ASK_ABORTED` | 注意力提问门控 | `signal.aborted` 时抛出；approval 侧对应返回 `'cancelled'` |
| `USER_ATTENTION_REJECTED` | 注意力提问门控 | 最终拒绝时抛出；approval 侧对应返回 `'rejected'` |
| `[exit code: N]` | 待办执行结果文本 | `detectProcessFailure()` 的进程失败判据（N≠0 视为失败） |

### 7.3 常量表

| 常量 | 值 | 归属 | 含义 |
|---|---|---|---|
| `ACTIVITY_THROTTLE_MS` | 1500 | 注意力 Client | 活动上报最小间隔 |
| `ACTIVITY_REFRESH_DELAY_MS` | 250 | 注意力 Client | 活动上报的合并延迟 |
| `DEFAULT_POLL_MS` | 1500 | 注意力 Client | 状态轮询默认间隔 |
| `MIN_POLL_MS` | 1000 | 注意力 Client | 状态轮询下限 |
| `MAX_POLL_MS` | 5000 | 注意力 Client | 状态轮询上限 |
| `HIDDEN_POLL_MS` | 5000 | 注意力 Client | 页面隐藏时的轮询下限 |
| `MAX_BACKOFF_MS` | 10000 | 注意力 Client | 失败退避上限 |
| `OFFLINE_AFTER_FAILURES` | 3 | 注意力 Client | 连续失败阈值（达到后开始退避并显示「未连接」） |
| `REQUEST_TIMEOUT_MS` | 5000 | 注意力 Client | 每个 fetch 的硬超时 |
| `WATCHDOG_GRACE_MS` | 1000 | 注意力 Client | 轮询看门狗宽限（比硬超时多 1s） |
| `RETRY_BANNER_TTL_MS` | 20000 | 注意力 Client | 重试提醒浮层时长 |
| `REJECTED_BANNER_TTL_MS` | 8000 | 注意力 Client | 最终拒绝提醒浮层时长 |
| `TITLE_FLASH_MS` | 12000 | 注意力 Client | 标题闪烁时长 |
| `TODO_PURGE_HOLD_MS` | 5000 | 注意力 Client | 危险区长按确认时长 |
| `TODO_PURGE_TICK_MS` | 50 | 注意力 Client | 长按进度刷新间隔 |
| `ATTENTION_REJECTION_VERSION` | 1 | 注意力 Host | `userAttention.version` 的取值 |
| `TODO_GRANT_TTL_MS` | `5 * 60 * 1000` | 待办 Host | 一次性 grant 的有效期 |
| `TODO_STATE_VERSION` | 1 | 待办 Host | `.dsh-todo-inbox/state.json` 的版本 |
| `TODO_HISTORY_VERSION` | 1 | 待办 Host | `.dsh-todo-inbox/history.json` 的版本 |
| `TODO_LIMITS.forkedSessions` | 100 | 待办 Host | 分支复制记录的保留条数 |
| `RESTART_TOOL_PATTERN` | `/^restart_dsh_(?!status$\|force_safeguard$)/` | 待办 Host | 重启类工具判定 |
| `RECOMMENDED_LABEL_SUFFIX` | 见 1.4.5 | 注意力 Host | 推荐项标记识别 |

表中只收源文档里**具名**的常量；非具名限值（提醒 ttl、拒绝摘要窗口、问题截断、轮询夹取、批量与
条数上限、回注摘要截断）按源文留在正文对应小节，不重复收入本表。

### 7.4 术语表

| 术语 | 定义 |
|---|---|
| bundle / 插件卡 | 只带 `dsh.bundle.patch` 与 `dependencies` 的包装包，自身没有 `dsh.client`；通过补丁插入两条 Loader 行 |
| Loader 行 | `cordis.patch.yml` 里 `insert` 的一条记录（`id` + `name` + `config`），对应插件列表里可独立开关的一行 |
| 门控 | 注意力组件在 `approval/request` / `user-questions/request` 瀑布流最前端（`prepend: true`）的处理，决定呈现、撤回、提醒重试与最终拒绝 |
| 最终拒绝 | `finalizeRejection()` 的唯一收尾结果，按 `manual` / `offline` / `timeout` / `timeout-manual` 分类，伴随拒绝事件与模型可见说明 |
| 一次性授权（grant） | 待办执行前登记、只对本次新 `callId` 有效的一次性批准凭据；TTL 与消费规则见 4.2 |
| 分支对话复制 | DSH「分支对话」派生子会话时，把原会话 `pending` / `approved` 待办复制一份到分支（新 id、`sessionId` 改为分支、`originSessionId` 指回原会话） |
| 仅复核（reviewOnly） | 元工具待办的标记：可以拒绝，不能执行 |
| 待审核 / 已操作 | 界面分段名，key 仍为 `pending` / `approved`；「已操作」的过滤与计数合并 `approved \|\| rejected` |
| 返回修改（restore） | 把 `rejected` 置回 `pending`、保留草稿答案的决定动作 |
| 回注 | 把「请调用 `todo_execute`」等说明注入来源会话（`source: { kind: 'todo-inbox', form: 'notice' }`） |
| 冷会话 | Host 尚未加载、不在 `ctx.sessions.list()` 里的会话 |
| 隔离 profile | 用临时 `DSH_HOME` 启动的验证用 profile |
| 热重载对账 | 见 3.5：`reloadExpected` 让热重载期间的执行归档为 `executed` 而非 `failed` |
| 脱敏预览 / 参数哈希 | 见 4.1：只落盘脱敏参数与 sha256，执行时回来源会话读完整参数并校验 |

### 7.5 已知限制

| 限制 | 说明 |
|---|---|
| 冷会话默认值 | 冷会话（Host 未加载）不在 `ctx.sessions.list()`，`state` 的 `sessionHasContent` 为 `null`；客户端退回会话列表 `blank`，再不行按“已有内容”处理，因此冷会话可能先显示「本会话」，打开/恢复后随轮询校正 |
| 注意力 Client 未接 locale | 界面文案硬编码中文；`locale/*.json` 只服务插件卡片 |
| 待办行没有 UI 配置入口 | `todo-inbox/client.js` 的 `ConfigEntry` 定义了但未注册；`attention/index.js` 的 `findTodoConfigTarget()` 也从未被调用（合并页只写注意力行）。改待办行只能走 `POST /api/todo-inbox/config` 或直接改 profile 补丁 |
| tgz 含文档 | 最终产物 `DSH-Plugin-tgz/local-dsh-user-attention-and-todo-inbox-1.5.0.tgz` 共 22 个文件：bundle 根 8 项（含 `README.md` / `TECHDOC.md` / `HISTVER.md`）+ 两个组件包各 7 项（含各自的 `README.md`）；测试目录不进 tgz |
| 在线投递不受监督 | `trackOnlineInteraction()` 只记账；当前实现不接管官方在线超时语义 |
| 跨会话不可执行 | `todo_get` / `todo_execute` 强制 `todo.sessionId === 调用 AI 的会话`；来源会话没打开时 `apply` 返回 409 `SESSION_NOT_LIVE`。分支复制来的条目归分支会话所有（`sessionId` 是分支），原始参数从 `originSessionId` 的原会话日志读；原会话已归档/日志不可读时执行失败并保留待办 |
| 宿主重启中的执行不确定 | `executing` 状态在加载时一律降级 `failed`；重启类工具的 `restarting` 会归档为 `restarted`（结果未知，需人工确认） |
| 重启类工具拿不到返回值 | 通过待办执行 `restart_dsh_blocking_wait_safe` / `roll_call_caution` / `force_dangerous` 时宿主会消失，`todo_execute` 无法返回结果；只保证重启前状态已落盘 |
| 并行批量副作用 | `parallel` 会在同一会话并发执行多个工具，写同一文件等冲突由调用方保证；并行模式下重启类工具同样会被执行（源码 `runTodoBatch` 的 parallel 分支不排除重启类工具），串行模式遇到重启工具后停止后续条目 |
| 历史无上限 | `history.json` 全量保留、只做 `before` 游标懒加载，文件会随时间增长 |

## 许可

本文件以 MIT 许可发布，版权归 DX12_XYZ 所有。完整条款见仓库根 [LICENSE](./LICENSE)。

<!-- fused-by: deepseek-v4.1-flash -->

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
