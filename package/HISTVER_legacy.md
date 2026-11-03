# 私有包时期变更记录（legacy）

> 本文件只记录**正式公开之前**的私有迭代，仅供追溯参考：
>
> - 这些版本从未公开发布；本套文档所在的仓库不随附这些历史时期的源码与 `.tgz` 产物（大部分版本的实物归档在 `.del/DSH-Plugin-tgz/`，仅作参考），收版后的正式产物由本仓库 `deliver/releases/` 提供；
> - 记录可能不完整，**不作为对外契约**，行为以当前版本为准；
> - 正式公开后的变更记录见 [HISTVER.md](./HISTVER.md)。

---

## 版本历史（私有时期）

> **最后一版：1.5.0** —— `@local/dsh-user-attention-and-todo-inbox`（bundle），组件包 `@local/dsh-user-attention-core` 与 `@local/dsh-todo-inbox` 同为 `1.5.0`。
> 本文件按时间倒序记录插件形态与行为变化，面向维护者。上手与安装见 [README.md](./README.md)，架构、路由与状态机细节见 [TECHDOC.md](./TECHDOC.md)。
>
> **记录范围**：`1.4.2-hotfix.1` 之前的条目来自当时的三份 README（注意力组件 / 待办组件 / 合并版）与 `.del/` 下的历史 tgz，只保留可验证的关键变更。

## 目录

- [版本总表](#版本总表)
- [逐版本详情](#逐版本详情)（共 17 个小节，倒序）：
  1.5.0 · 1.4.3-hotfix.11 · 1.4.3-hotfix.10 · 1.4.3-hotfix.9 · 1.4.3-hotfix.8 · 1.4.3-hotfix.7 ·
  1.4.3-hotfix.6 · 1.4.3-hotfix.5 · 1.4.3-hotfix.4 · 1.4.3-hotfix.3 · 1.4.3-hotfix.2 · 1.4.3-hotfix.1 ·
  1.4.3 · 1.4.2-hotfix.2 · 1.4.2-hotfix.1 · 1.4.2 · 1.4.0-hotfix.1
- [合并前的两条独立线](#合并前的两条独立线)：[用户注意力（合并前）](#用户注意力合并前) · [待办收件箱（合并前）](#待办收件箱合并前)

---

## 版本总表

| 版本 | 主题 | 一句话 |
|---|---|---|
| **1.5.0**（正式版） | 收版 | 把 `1.4.3-hotfix.9` / `hotfix.10` / `hotfix.11` 收进正式版，行为零变更，含按钮文案、单选自定义互斥、待审核/已操作与拒绝回流、分支复制、缓存释放危险区；只改版本号与文档（`.del` 内未打包，发布产物见 `deliver/releases/local-dsh-user-attention-and-todo-inbox-1.5.0.tgz`） |
| **1.4.3-hotfix.11** | 危险区 + 缓存清理 | 合并设置页新增红色 5 秒长按框「释放并删除待办缓存」，Host 新增 `POST /api/todo-inbox/purge`：删除 pending/approved/rejected/failed、保留 executing/restarting、清空历史与分支记录，立即 persist |
| **1.4.3-hotfix.10** | 状态机 + 文案 + 分支复制 | 「拒绝」改为 `rejected`（不删除、可返回修改）；「待复核/已批准」改名「待审核/已操作」、「应用全部」改名「应用操作」；`rejected` 只在 apply 与 `todo_*` 工具入口归档删除；分支对话派生时自动复制原会话未执行、未拒绝的待办 |
| **1.4.3-hotfix.9** | 按钮文案 + 提问单选 | 高亮条「应用」改名「应用全部」（仍只应用最新一条）；卡片改为「单次应用（发送回答/执行请求）」；提问单选：自定义与选项互斥，多选保持可并存 |
| **1.4.3-hotfix.8** | 默认作用域 | 按会话新旧决定：新会话默认「全部会话」，已有内容的会话（哪怕没待办）默认「本会话」 |
| **1.4.3-hotfix.7** | 作用域统一 | 「本会话 / 全部会话」改为四个分段共用的唯一选择，进/出历史不再跳变 |
| **1.4.3-hotfix.6** | 历史分段重做 | 历史页接入收件箱同一套头部 / 作用域行 / 状态行 / 卡片，排布与逻辑逐项对齐 |
| **1.4.3-hotfix.5** | 历史刷新 | 历史作用域行加回「刷新」按钮，经 `reloadToken` 重拉第一页 |
| **1.4.3-hotfix.4** | 历史作用域行对齐 | 历史的「本会话 / 全部会话」移到状态分段上方，与收件箱同位置；显示只读与总条数 |
| **1.4.3-hotfix.3** | 历史详情 + 热重载对账 | 历史条目带生成/审批/执行/归档时间与审批结果，问题型展示全部选项与自定义回答；自我安装不再误报 failed；拒绝补审批时间 |
| **1.4.3-hotfix.2** | 界面 | 右栏「历史」并入状态分段，紧挨「失败」；移除顶部「收件箱 / 历史」切换行 |
| **1.4.3-hotfix.1** | 放开重启 + 批量 + 历史 | `restart_dsh_*` 不再仅复核；`todo_execute` 支持多 id 串行/并行；新增只读历史与懒加载分页 |
| **1.4.3** | 正式 | 「1 卡 + 2 组件 + 单 tgz」转正式：版本号去掉预发布后缀，内容同 `1.4.2-hotfix.2`，只清掉源码里指向 hotfix.2 的过期注释 |
| **1.4.2-hotfix.2** | 文档 | TECHDOC 收窄为纯技术细节（注册/安装/升级移出），README 补升级与卸载清理；打包改用自研零依赖 CLI，产物仍是 22 文件 |
| **1.4.2-hotfix.1** | 拆分 | 恢复「1 张插件卡 + 2 个组件包」，仍是一个自包含 tgz（bundledDependencies）；两个组件各自一行、各自 Client 半身，可独立开关 |
| 1.4.2 | 合并 | 三份 README 合并；改为**单包单行**（`attention.js`/`todo.js`/合并 `client.js`），一个 tgz 装齐，代价是两个组件不再有独立行与独立 Client |
| 1.4.0-hotfix.1 | 预发布 | 先合入注意力，再按验收改为 **bundle + 两个组件**（`user-attention-core` + `todo-inbox`）、合并配置页、超时策略与推荐项自动回答 |
| 1.3.2-hotfix.2 … 1.0.0 | 注意力 | `userAttention` 服务与最终拒绝事件、一次性 grant、超时策略、配置面板、手动状态粘性等 |
| 0.1.0-hotfix.5 … 0.1.0-hotfix.1 | 待办 | 自动入队、`todo_list/get/execute`、右栏页与高亮条、i18n、失败保留等 |

---

## 逐版本详情

### 1.5.0（2026-11-02）

**结论：内容即 `1.4.3-hotfix.11`，行为零变更**——本轮只改版本号与文档，没有触碰任何行为代码。

#### 变更

- 版本号统一为正式版：bundle 与两个组件统一到 `1.5.0`（不带 hotfix 后缀），bundle 的 `dependencies` 精确钉住同版本；
  `components/todo-inbox/index.js` 的 `PLUGIN_VERSION` 同步为 `1.5.0`。
- 收进正式版的补丁：`1.4.3-hotfix.9`（高亮条「应用全部」与卡片「单次应用」文案、提问单选的自定义与选项互斥）、
  `1.4.3-hotfix.10`（「待审核 / 已操作」改名、拒绝改为 `rejected` 且可「返回修改」、`rejected` 只在 apply 与
  `todo_*` 工具入口归档、分支对话复制待办）、`1.4.3-hotfix.11`（设置页红色危险区「释放并删除待办缓存」的
  5 秒长按与 `POST /api/todo-inbox/purge`）。
- **只改版本号与文档**：路由、状态机、界面与 `1.4.3-hotfix.11` 完全一致；历史章节里 `1.4.3-hotfix.11`
  及更早的版本号保持原样，不改历史。
- 文档里的 tgz 文件名改为 `local-dsh-user-attention-and-todo-inbox-1.5.0.tgz`；本轮未在 `.del/DSH-Plugin-tgz/` 打包，发布产物由本仓库 `deliver/releases/local-dsh-user-attention-and-todo-inbox-1.5.0.tgz`（2026-11-03 13:50，169 618 B）提供。

#### 动机

正式化三个 hotfix 的成果，让使用方可以按稳定版本号引用。

#### 兼容性与迁移


#### 验证

`node --check` 4/4；待办 Host 49/49；Client 渲染 29/29；注意力 19/19（三套测试都没有写死版本号的断言，
只有 `PLUGIN_VERSION` 常量随源码改到 `1.5.0`，无需改测试）。本轮未在 `.del/DSH-Plugin-tgz/` 重新打包
（发布产物见 `deliver/releases/`），也未执行
`plugin_manager` 或重启 DSH。

### 1.4.3-hotfix.11（2026-11-02）

**结论：危险操作「释放并删除待办缓存」**——合并设置页新增红色 5 秒长按框，Host 配套新增 `POST /api/todo-inbox/purge`。

#### 动机

合并设置页此前没有清空待办缓存的入口；本轮新增一个不可误触的红色危险框，满 5000ms 才发请求。显示条件沿用
超时设置的 `todoAvailable === true && todoAutoEnqueue === true`，长按交互沿用逃生舱设置页的 `HoldButton`。

#### 变更 — Host 新接口 `POST /api/todo-inbox/purge`

无请求体；与其它 `api/todo-inbox/*` 路由同源，只由本机 UI 调用，不需要审批。语义：

- 删除所有 `status` 为 `pending` / `approved` / `rejected` / `failed` 的待办；
- **保留** `executing` / `restarting`——正在执行原工具的待办不能打断；它们执行结束后仍走原来的
  `removeTodo` / `archiveTodo` 路径收尾，不会变成孤儿；
- 清空历史数组（`history.json` 里的条目）、`forkedSessions` 与一次性 approval grants；
- 立即 persist：把空快照写回 `state.json` / `history.json`，**保留文件本身**，不删除文件；
- 响应 `{ ok: true, todos: <删除条数>, history: <删除条数>, kept: <保留条数> }`；整段同步执行
  （Node 单线程内不会与请求/执行交错），`historySeq` 不回退（清空后新归档条目 seq 仍单调递增）；
  接口幂等，空缓存上再调用返回全 0。

#### 变更 — Client 危险区与 5 秒长按

- 合并设置页的待办区块里，`todoAvailable === true && todoAutoEnqueue === true` 时在批准/提问超时设置之后
  显示红色虚线框：标题「释放并删除待办缓存」+ 说明 + 长按按钮；`todoAutoEnqueue` 关闭或待办组件缺席时
  整块不渲染（与超时设置同一显示条件）。
- 5 秒长按照搬逃生舱设置页的 `HoldButton`：`pointerdown`（主键）开始计时并 `setPointerCapture`；
  `pointerup` / `pointercancel` / `pointerleave` / `lostpointercapture`、指针移出按钮矩形、窗口失焦、
  标签页切到后台都取消且**不发请求**；满 5000ms 才发 `POST api/todo-inbox/purge`（50ms 一拍）。
- 按住时按钮文案「继续按住… x.x 秒」，按钮内底部 `::after` 进度条按 `--ua-hold-progress` 推进、
  脉冲动画随进度加快；`prefers-reduced-motion: reduce` 下关闭动画与过渡。
- 鼠标、触摸（`touch-action: none`）与键盘（按住 Enter/Space）都可用；单击永远不是确认。
- 按钮/结果文案：空闲「长按 5 秒释放并删除」、请求中「正在释放…」、成功「已释放并删除缓存
  （N 条待办、M 条历史；保留 K 条执行中）」、失败「释放并删除缓存失败：原因」（错误体优先，
  取不到时按超时/网络/HTTP 状态给出兜底文案）。请求进行中按钮禁用。
- **不刷新整页**：成功后由注意力 Client 派发同源 window 事件 `dsh-todo-inbox:purged`，待办 Client
  在轮询 effect 里监听该事件并立刻 `refresh()`，列表在下一帧就跟着变空（不依赖 `location.reload`）。
  事件不可用或派发失败只影响即时刷新，不影响结果文案。
- 样式：新增 `ua-hold` 系列（`.ua-hold`、`.ua-hold::after`、`.ua-hold-active`、`@keyframes ua-hold-pulse`
  与 `prefers-reduced-motion` 覆盖），从逃生舱的 `.dshpod-hold` 三条规则复制而来，换成 `ua-` 前缀避免
  类名冲突；危险框复用新的 `.ua-cfg-danger*`。

#### 兼容性与迁移

bundle 与两个组件统一到 `1.4.3-hotfix.11`（bundle 的 `dependencies` 精确钉住同版本，不带 `^`/`~`）。

#### 验证

`node --check` 4/4；待办 Host 49/49（新增 2 条 purge 用例：执行中保留 + 计数 + 文件落盘/重载为空 +
grant 清空）；Client 渲染 29/29；注意力 19/19。UI 长按未加自动化（注意力 Client 没有渲染测试台），
手工验证点见 TECHDOC 8.4。已打包：`.del/DSH-Plugin-tgz/local-dsh-user-attention-and-todo-inbox-1.4.3-hotfix.11.tgz`（2026-11-02 23:38，有 tgz 实物，来源见 `.del/DSH-Plugin-tgz/`）；未执行 `plugin_manager` 或重启 DSH。

### 1.4.3-hotfix.10（2026-11-02）

**结论：状态机改造 + 文案改名 + 分支复制**——拒绝不再删除，改为 `rejected` → 可「返回修改」。

#### 动机

原实现里 `reject` 直接把待办归档删除并通知来源会话；本轮把「拒绝」变成一次可撤销的人工决定。

#### 变更 — `rejected` 状态机

- `TODO_STATUSES` 新增 `rejected`。`decide reject` 只对 `status === 'pending'` 生效：置 `status = 'rejected'`、
  `decidedAt = Date.now()` 并持久化，**不删除、不归档、不向来源会话注入「用户拒绝了待办…」**
  （用户可以返回修改，注入会让 AI 误以为这项已被最终否决）；
- 新增 `decide action: 'restore'`：仅对 `status === 'rejected'` 生效，置回 `status = 'pending'`、
  `decidedAt = null` 并持久化；问答类待办保留其草稿答案（`todo.answers`），响应里带回 `answers`；
- `state` 快照对问答类待办新增 `answers` 字段（模型侧 `summarizeTodo` / `detailTodo` 仍不含答案），
  这是客户端「已操作」只读展示与「待审核」表单预填的数据来源；
- 拒绝问答类待办时，客户端把当前草稿（每题 `selected` / `custom`）随 reject 请求提交；Host 用新增的
  `validateDraftAnswers()` 做宽松校验（允许某题没答、允许空，但仍校验问题 id、选项 label、单选约束）
  后存入 `todo.answers`；`sanitizeStoredTodo` / `sanitizeHistoryEntry` 对 `pending` / `rejected` 的草稿
  使用宽松校验（只有可执行的 `approved` 仍要求完整答案）；
- 失败（`failed`）待办的拒绝保持原有行为：通知来源会话 + 归档删除（`outcome='failed'`）。

**删除时机（`rejected` 只在两处归档删除，`outcome='rejected'`）**：

1. `POST /api/todo-inbox/apply` 处理过程中（进入 handler 即清理，与该次 apply 成功与否无关）；
2. AI 调用 `todo_list` / `todo_get` / `todo_execute` 任一工具时，进入实现后先清理再执行原逻辑；
   `state` / `history` / `decide` / `restore` 一律不清理。

清理由 `purgeRejectedTodos()` 同步完成：按 id 重新定位并 `splice`、`archiveTodo()` 按 id 去重，
因此重复调用幂等，也不会与同一 tick 的读取交错。

#### 变更 — 文案改名

界面、`EN_TEXT`、Host 注入给 AI 的通知与文档统一把「待复核」改为「待审核」、「已批准」改为「已操作」；
输入框高亮条的应用按钮「应用全部」改为「应用操作」（仍只应用最新一条，不是批量）。`dsh-power` 的
「应用」与本插件无关，未改动。

#### 变更 — 客户端

分段仍用 key `approved`，但「已操作」的过滤与计数合并 `approved || rejected`；`rejected` 卡片不显示
「单次应用」与「拒绝」，只显示「返回修改」（调用 `store.restore(id)`）；问答类 rejected 卡片只读展示
每题选中项与自定义文本（复用历史卡片的只读问答块），不再可编辑；待审核的问答表单由 `TodoCard` 持有
草稿（初始化时读 `todo.answers`），提交与拒绝都用当前表单值。

#### 变更 — 分支对话复制待办（`session/created`）

用 DSH 的「分支对话」从原会话派生子会话时，原会话的待办原来不会跟过去。本轮让新分支自动获得一份副本：

- **触发点**：DSH 在会话创建时发 `session/created`（回调参数是 session 对象）。分支会话的
  `session.header.parentSession` 是原会话 id（string）、`session.header.isSeeded === true`；普通新会话
  没有 `parentSession`。Host 用 `ctx.effect(() => ctx.on('session/created', handler), 'todo-inbox: fork copy todos')`
  订阅（`ctx.on` 返回 disposer，随插件卸载回收），handler 内部整体 try/catch——复制失败只写日志，绝不影响会话创建；
- **复制规则**：只复制 `status === 'pending'` 与 `status === 'approved'` 的待办；`failed` / `executing` /
  `restarting` / `rejected` 一律不复制。每条副本：新 `id`、`sessionId` 改为分支会话 id、新增
  `originSessionId = parentSession`，其余字段（`kind` / `toolName` / `callId` / `argsHash` / `argsPreview` /
  `title` / `detail` / `requestReason` / `questions` / `referencedFiles` / `createdAt` / `decidedAt` /
  `answers` / `reviewOnly` 等）原样保留；原会话的待办不受影响，两边此后各自独立；
- **幂等**：同一分支只处理一次——`forkedSessions`（有界 100 条，随 `state.json` 持久化）记录已复制的分支 id，
  因此同一分支多次触发、分支恢复时重复的 `session/created`、甚至宿主重启后都不会重复复制；另外分支里只要
  已经存在任何待办，就视为「已经处理过」，只记录、不复制；复制条数受 `maxTodos` 限制；
- **`originSessionId` 链路**：DSH 的 `ownEvents()` 只返回 fork 继承前缀之后的**自有事件**，分支会话里找不到
  父会话的 `tool/call`。待办模型新增可选字段 `originSessionId`（string ≤200，`sanitizeStoredTodo` 保留、
  随 `state.json` 持久化）；`executeApprovalTodo()` 改用 `todo.originSessionId ?? todo.sessionId` 去
  `readCallFromSession()` 查原始调用，工具名比对与 `argsHash` 校验不变；原会话已不在活跃列表时仍返回原来的
  「无法从来源会话日志定位原始工具调用……」，不抛别的异常；
- **模型可见文本**：`summarizeTodo` / `detailTodo` 带 `originSessionId`；`todo_list` / `todo_get` 对来自分支的
  条目注明「来源：分支复制（原会话 xxxxxxxx）」。

#### 兼容性与迁移

bundle 与两个组件统一到 `1.4.3-hotfix.10`（bundle 的 `dependencies` 精确钉住同版本，不带 `^`/`~`）。

#### 验证

`node --check` 4/4；待办 Host 47/47（新增 7 条分支复制用例）；Client 渲染 29/29；注意力 19/19。
已打包：`.del/DSH-Plugin-tgz/local-dsh-user-attention-and-todo-inbox-1.4.3-hotfix.10.tgz`（2026-11-02 22:36；包内组件 `package.json` 由 hotfix.6 时的 1445/1220 B 增至 1498/1221 B，与本轮分支复制改动的字段增加吻合，说明确为该版本源码所打）；未执行 `plugin_manager` 或重启 DSH。

### 1.4.3-hotfix.9（2026-11-02）

**结论：按钮文案表达粒度 + 提问单选自定义与预设选项互斥**，两者行为面刻意最小化。

#### 动机

两处按钮原来都只叫「应用」，读者分不清粒度；单选题又能同时给出预设选项和自定义两个答案。
本轮只改文案与交互约束，行为面刻意最小化。

#### 变更 — 按钮文案（粒度）

待办页卡片和输入框高亮条上的按钮原来都叫「应用」，分不清是「只发这一条」还是「发当前范围的全部」。
本轮把两处文案改成能表达粒度的写法，**行为完全不变**：

- 输入框高亮条（`DockBar`）的「应用」→「应用全部」；点击仍只对最新一条已批准待办调用
  `store.apply(newest.id)`，没有改成批量逐条应用，也没有新增接口或新的后端路由；
- 待办卡片（`TodoCard`）的「应用（发送回答请求）」→「单次应用（发送回答请求）」、
  「应用（发送执行请求）」→「单次应用（发送执行请求）」；仍只应用该卡片对应的那一条；
- 英文词典（`EN_TEXT`）同步：`Apply all`、`Single apply (send answer request)`、
  `Single apply (send execution request)`；
- 「允许一次」「去回答」「打开」「刷新」「拒绝」等其它按钮与全部判定逻辑保持不变：高亮条仍然只在
  非全局模式、最新一条已批准且非仅复核时出现应用按钮；无待办时整条不渲染。

#### 变更 — 提问单选：自定义与选项互斥；多选保持可并存（`QuestionForm`）

单选题（`multiSelect !== true`）原来「选了预设选项还能再填自定义」，等于同时给了两个答案。
本轮把「自定义」并入选项组：

- `.ti-options` 末尾多一行「自定义」radio（与预设选项同一个 `name`，`checked` = 自定义文本非空）；
  点击/聚焦后把焦点交给下面保留的文本框，选中时该行与其它选项一样走 `.ti-option-on` 高亮；
- 选中任一预设选项会清空该题 `custom`；输入非空（`trim()` 后）自定义会清空该题 `selected`，
  清空输入不动已选选项——即单选只能在「预设选项」和「自定义」之间二选一；
- `validateLocalAnswers()` 增加防御：单选题目 `selected.length > 0 && custom.trim() !== ''` 时返回
  「单选题只能在预设选项和自定义之间选一个」，此时提交按钮保持禁用；
- **多选题目完全不变**：仍是 checkbox 多选，选项之外可以继续补充自定义，两者同时保留；
- 提交答案结构不变，仍是 `{ id, selected, custom }`（`custom` 为空时省略该字段）。

#### 兼容性与迁移

bundle 与两个组件统一到 `1.4.3-hotfix.9`（bundle 的 `dependencies` 精确钉住同版本，不带 `^`/`~`）。

#### 验证

`node --check` 4/4；待办 Host 33/33；Client 渲染 23/23（原 19 例全绿，新增 4 例：单选自定义/选项互斥且提交只带一种、多选选项+自定义并存、单选无选项时只能自定义、`validateLocalAnswers` 的互斥防御；另有 3 例来自本轮前半段的按钮文案改动）；注意力 19/19。
已打包：`.del/DSH-Plugin-tgz/local-dsh-user-attention-and-todo-inbox-1.4.3-hotfix.9.tgz`（2026-11-02 18:05，有 tgz 实物，来源见 `.del/DSH-Plugin-tgz/`）；未执行 `plugin_manager` 或重启 DSH。

### 1.4.3-hotfix.8（2026-11-02）

**结论：默认作用域改为按「会话新旧」判断**——不再用「本会话有没有待办」猜，只看会话是否已经有对话内容。

#### 动机

原来 `PaneTab` 用「本会话有没有待办」（`ownTodos.length > 0`）猜默认值，导致有对话但没有待办的会话
一打开面板就跳到「全部会话」。现在默认值只看会话是否已经有对话内容：

- 全新会话（还没有任何对话内容）→ 默认「全部会话」；
- 已有内容的会话（无论当前有没有待办）→ 默认「本会话」；
- 用户手动点选（`scopeOverride`）仍然最优先，并继续对四个分段同时生效。

#### 变更 — Host 侧信号（`components/todo-inbox/index.js`）

- 新增 `detectSessionHasContent(session)`：优先读 `session.surface.nodes`（DSH 会话消息面，只收录会产生
  模型可见消息的事件），空数组 = 全新会话，非空 = 已有对话；拿不到 surface 时退回扫描 `turn/start` /
  消息 / 工具事件，标题、配置等无关日志事件不算内容；冷会话（不在 `ctx.sessions.list()`）返回 `null`。
- `GET /api/todo-inbox/state` 扩展：请求可带 `session=<id>`；响应新增 `sessionId`（回显，没有则为 `null`）
  与 `sessionHasContent`（`true` / `false` / `null`）。没有 `session` 参数或会话不在活列表里时为 `null`，
  老客户端忽略新字段即可；老 Host 不返回字段时新 Client 有兜底。

#### 变更 — Client 侧（`components/todo-inbox/client.js`）

- 共享 `store` 新增 `setActiveSession(id)`；状态请求变成 `api/todo-inbox/state?scope=all&session=<id>`，
  切换会话时立即补发一次（在途请求结束后重发），不必等下一轮轮询。
- `normalizeSnapshot()` 归一化 `sessionId` / `sessionHasContent`（缺字段 → `null`）。
- 新增 `sessionHasContentSignal()`：只有 snapshot 回显的 `sessionId` 与面板会话一致时才采信
  `sessionHasContent`；否则退回 DSH 会话列表的 `blank` 元数据（`blank === false` 表示已有内容），
  仍未知则按「已有内容」兜底（不把老服务/冷会话的用户全当新会话）。
  默认 `scope = scopeOverride ?? (sessionHasContent === false ? 'all' : 'session')`。
- 新会话发出第一条消息后，Host 下一轮 snapshot 把 `sessionHasContent` 置 true；未被手动覆盖的默认值
  自然切到「本会话」，手动选过的保持不变；四个分段继续共用同一个 `scope`。

#### 验证

`node --check` 4/4；待办 Host 33/33（新增 `detectSessionHasContent` 与 state 接口各 1 例，含无消息/有消息无待办/缺 session 参数/冷会话）；Client 渲染 16/16（原 11 例全绿，新增 5 例：新会话默认 all、已有内容无待办默认 session、首条消息后自然切换、手动覆盖优先、旧服务/undefined 兜底，以及状态请求带 session 的契约）；注意力 19/19。
已打包：`.del/DSH-Plugin-tgz/local-dsh-user-attention-and-todo-inbox-1.4.3-hotfix.8.tgz`（2026-11-02 17:17，有 tgz 实物，来源见 `.del/DSH-Plugin-tgz/`）。

### 1.4.3-hotfix.7（2026-11-02）

**结论：只改界面（作用域统一）**。

#### 动机

「本会话 / 全部会话」原来历史自己有一份 `historyScope`、收件箱用 `scope`/`scopeOverride`，导致在收件箱选
「本会话」后点「历史」会跳成「全部会话」。现在四个分段共用唯一的 scope 选择，切换分段不再改变它。

#### 变更

- 删除 `historyScope` state 与 `selectHistoryScope`；`useHistoryFeed({ scope })` 直接使用收件箱的共享
  `scope`（默认规则不变：`scopeOverride ?? (ownTodos.length > 0 ? 'session' : 'all')`）。
- `ScopeSeg` 的 `value` / `onChange` 不再按 `historyOpen` 分支，始终是共享 `scope` 与同一个
  `setScopeOverride`；作用域行只剩一条渲染路径，没有历史专用的分支、徽章或按钮。
- 切换分段（含进入/离开历史）不改共享 scope、不触发重新拉取；改 scope 时四个分段看到同一个值，历史按新
  scope 重新拉第一页，旧 scope 的在途响应仍由递增 `requestId` 作废（`useHistoryFeed` 的 `scopeKey` 逻辑保留）。
- 收件箱提示（自动入队已关闭 / 注意力组件未启用 / 当前会话没有待办）条件不变，历史打开时也按同一条作用域行渲染。

#### 验证

`node --check` 4/4；待办 Host 31/31；Client 渲染 11/11（含四个共享作用域用例：收件箱选 all 进历史仍 all、历史选 session 切回收件箱仍 session、只切分段不重拉、共享 scope 切换旧响应作废）；注意力 19/19。
已打包：`.del/DSH-Plugin-tgz/local-dsh-user-attention-and-todo-inbox-1.4.3-hotfix.7.tgz`（2026-11-02 15:53，有 tgz 实物，来源见 `.del/DSH-Plugin-tgz/`）。

### 1.4.3-hotfix.6（2026-11-02）

**结论：只改界面（历史分段重做）**。

#### 动机

历史页不再自带一套头部，改为与「待复核 / 已批准 / 失败」共用同一个 `PaneTab` 外壳，排布与交互逐项对齐。

#### 变更 — 排布

- 分段行（tab）、作用域行（本会话 / 全部会话）、工具条（刷新）统一走同一套组件与 class：标题行仍是
  `ti-count`（历史显示「共 N 条 · 只读」）+ `ti-header-actions` 里的「刷新」按钮；作用域行只有一个
  `ScopeSeg`（`ti-seg`），历史不再往里塞「只读」徽章、总条数与刷新按钮；分段行统一为 `SegmentBar`（`ti-seg`）。
- 加载中（`正在加载…`）、空态（`EmptyState`：`没有历史记录` + 说明）、错误（头部 `ti-error` 行，
  `历史加载失败：…`）与收件箱同一位置、同一组件；删掉空态里的「重试」按钮——头部刷新按钮即可重试。
- 历史卡片复用收件箱的卡片结构（`ti-card` / `ti-card-head` / `ti-meta` / `ti-details` / `ti-files` /
  `ti-actions`）；结局徽章并入共用 `Badge`（`ti-badge-*`），删除平行样式 `.ti-outcome*`；失败 / 结果文本
  改用 `ti-error` / `ti-detail`，删除 `.ti-preview`。

#### 变更 — 逻辑

- `HistoryView` 组件改为 `useHistoryFeed` 数据源，状态由 `PaneTab` 持有：分段来回切换不重挂载、不重复请求、
  保留已加载页；作用域切换先清空列表与游标再拉第一页（严格隔离，在途响应由递增 requestId 作废）；刷新走
  `reloadToken` 重拉第一页并保留当前列表；「加载更多」继续用 `before=seq` 游标并在跨页按 seq/id 去重。
- 只读、不暴露任何 AI 工具、`scope=session` 必带 session id、全量保留 + 惰性分页、生成/审批/执行/归档时间与
  审批结果、问题选项与自定义回答等既有行为全部保留；Host 侧 `history.json` 与 `GET /api/todo-inbox/history` 未改。

#### 验证

`node --check` 4/4；待办 Host 31/31；新增 Client 渲染 8/8（`node test/client.test.mjs`）。已打包：`.del/DSH-Plugin-tgz/local-dsh-user-attention-and-todo-inbox-1.4.3-hotfix.6.tgz`（2026-11-02 13:43，有 tgz 实物，来源见 `.del/DSH-Plugin-tgz/`）。

### 1.4.3-hotfix.5（2026-11-02）

**结论：只改界面（历史刷新）**。历史作用域行在「只读 / 共 N 条」之后加回「刷新」按钮；`PaneTab` 用
`historyReload` 计数作为 `HistoryView` 的 `reloadToken`，点击后不重挂载、直接重拉第一页（cleanup 会作废
在途请求）。收件箱的刷新按钮与请求、分页、scope 隔离均未改。

**验证**：`node --check` 4/4；待办 31/31；产物 `local-dsh-user-attention-and-todo-inbox-1.4.3-hotfix.5.tgz`。

### 1.4.3-hotfix.4（2026-11-02）

**结论：只改界面（历史作用域行对齐）**。历史视图的「本会话 / 全部会话」作用域行从内容区移出，固定到状态
分段行上方，与收件箱同一位置；右侧显示「只读」与总条数。历史列表本体的请求、分页、scope 隔离与只读逻辑
不变（历史的「刷新」按钮随旧作用域行移除，切换 scope 或重进历史会重新拉取）。

**验证**：`node --check` 4/4；待办 31/31；产物 `local-dsh-user-attention-and-todo-inbox-1.4.3-hotfix.4.tgz`。

### 1.4.3-hotfix.3（2026-11-02）

**结论：历史详情补全 + 热重载对账不再误报 failed**。

#### 变更 — 历史详情

- 历史条目保存结构化 `questions` 与 `answers`（选项 label、选中集合、自定义回答），
  旧 `questionCount` / `answersPreview` 保留兼容；
- 历史卡片显示**生成时间、审批时间、执行时间、归档时间**与**审批结果**；
- 问题型条目只读展示每道题的全部选项（选中 `✓`、未选 `○`）、`custom` 自定义回答与「未回答」状态。

#### 变更 — 热重载对账

- `plugin_manager` 执行前把 `reloadExpected` 落盘；安装/卸载插件热重载 Host 后，`load()` 把
  `executing + reloadExpected` 归档为 `executed`（结果未知，需人工确认），不再误报 `failed`；
- `archiveTodo` 按 `id` 去重，避免重载前后两个实例各归档一次；
- `decide` 的 `reject` 分支补 `decidedAt`，历史里的「审批时间」不再为空。

#### 验证

`node --check` 4/4；待办 31/31；产物 `local-dsh-user-attention-and-todo-inbox-1.4.3-hotfix.3.tgz`。

### 1.4.3-hotfix.2（2026-11-02）

**结论：只改界面**。右侧栏待办页把「历史」并进状态分段，顺序为「待复核 / 已批准 / 失败 / 历史」，移除顶部的「收件箱 / 历史」切换行。历史视图自己的「本会话 / 全部会话」作用域、分页与只读逻辑不变。

**验证**：`node --check` 4/4；待办 28/28；产物 `local-dsh-user-attention-and-todo-inbox-1.4.3-hotfix.2.tgz`。

### 1.4.3-hotfix.1（2026-11-02）

**结论：放开重启类工具 + `todo_execute` 批量 + 只读历史**，一次带来三项行为变更。

#### 变更

- **放开 `restart_dsh_*`**：从 `REVIEW_ONLY_PATTERNS` 移除，审批通过后可执行。`restart_dsh_status` 只读、
  `restart_dsh_force_safeguard` 只是给下一次调用上保险，两者不按「执行即重启」处理；其余 `restart_dsh_*`
  （`blocking_wait_safe` / `roll_call_caution` / `force_dangerous`）在调用前把待办落成 `restarting`，
  宿主重启后由 `load()` 归档为 `restarted`（已执行，结果未知），不再误报 `failed`。
- **`todo_execute` 支持批量**：新增 `ids`、`mode`（`serial` 默认 / `parallel`）与 `continueOnError`。
  批量预检全有或全无；串行按传入顺序，默认首个失败即停、后续标 `skipped`；并行并发上限读取 DSH 官方 Agent 循环的
  `maxParallelToolCalls`（读不到回退 10，不再叠加自定义上限）；并行可以包含重启类工具（宿主可能中途重启）。
  返回 `items[]` 逐条结果与 `succeeded/failed/skipped` 汇总；单条执行失败不再抛错，改为 `ok:false`。
- **模型可见的复核信息**：`todo_list` 每条附带工具名、目的（`requestReason`）与参数预览（`argsPreview`，
  上限提升到 1000 字），`todo_get` 同样补充目的；模型据此自行判断批量顺序（例如逃生舱命令 → 上保险 → 强重启）。
- **只读历史**：新增 `.dsh-todo-inbox/history.json`（`version 1`，`seq` 单调递增、全量保留、按页懒加载），
  执行成功/失败删除/拒绝/重启对账都会归档。新增
  `GET /api/todo-inbox/history?scope=session|all&session=&limit=&before=`，`scope=session` 必须带会话且
  只回该会话；**不注册任何 AI 工具**，模型无法读取、恢复或重放历史。
- **审批更严**：`/api/todo-inbox/decide` 对 `reviewOnly` 条目直接返回 `409 REVIEW_ONLY`（此前只是执行时拦）。

#### 验证

注意力 19/19、待办 28/28；`node --check` 4/4；打包为 `local-dsh-user-attention-and-todo-inbox-1.4.3-hotfix.1.tgz`。浏览器界面（历史页、scope 切换、懒加载）需真机确认。

### 1.4.3（2026-11-02，正式版）

**结论：「1 张插件卡 + 2 个组件 + 单个自包含 tgz」正式输出**，版本号去掉 pre-release 后缀；行 `id` / `name`、配置默认值、组件行为与 `1.4.2-hotfix.2` 完全相同。

#### 变更

- 唯一代码改动：`attention/index.js` 里 `trackOnlineInteraction()` 的注释不再写「留给 hotfix.2」（该语义本轮未实现），改为「留给后续版本」；TECHDOC 的引用同步。
- 打包沿用 `tools/dsh-bundle-pack`；产物 `DSH-Plugin-tgz/local-dsh-user-attention-and-todo-inbox-1.4.3.tgz`，22 个文件。

#### 兼容性与迁移

- 旧产物（`1.4.2`、`1.4.2-hotfix.1`、`1.4.2-hotfix.2`）归档到 `.del/DSH-Plugin-tgz/`。

### 1.4.2-hotfix.2（2026-11-02）

**结论：形态与行为不变**——仍是「bundle + 两个组件包」的单 tgz；行 `id` / `name`、配置默认值、组件代码均未改动。本轮只动文档与打包工具。

#### 变更

- TECHDOC 收窄为纯技术文档：删掉「为什么 1 卡 + 2 组件」「安装」「升级注意」「profile 覆盖层里的旧行」「旧设置并入补丁默认值」；打包章节只留产物格式与校验，命令交给工具 README。
- README 补两节：升级到 `-hotfix.N` 的做法、卸载后清理 profile 覆盖层残留；并提示 `config` 是整行替换、覆盖要写全字段。
- 打包从 `tools/pack-bundle-modules.sh` 切到自研零依赖 CLI `tools/dsh-bundle-pack`
  （`pack` / `unpack` / `inspect` / `verify`）：同一输入两次打包 sha256 相同，`unpack` 能把 tgz 反拆成
  `components/` 多包布局，产物仍是 22 个文件。

#### 验证

`inspect` 报 2 行 / 2 组件 / 22 条目；`unpack → pack` 条目 22/22 一致；`node --check` ×4；注意力 19/19、待办 21/21。

### 1.4.2-hotfix.1（2026-11-01）

**结论：形态从「单包单行」拆回「bundle + 两个组件包」**，但保持**一个 tgz 自包含安装**。

#### 变更

- `@local/dsh-user-attention-and-todo-inbox` 变为纯 bundle：`cordis.patch.yml` 只做一次 `insert`，两行分别是 `user-attention`（`@local/dsh-user-attention-core`）与 `todo-inbox`（`@local/dsh-todo-inbox`）。
- 两个组件恢复为独立 npm 包，各自带 `dsh.client`、`client.js`、`locale`、`icon` 与单元测试；
  组件代码来自合并前的同一批文件（Host 侧只改回组件自己的包名/行名，Client 侧恢复了待办组件的
  「注意力未启用」橙色说明）。
- 打包用 `tools/pack-bundle-modules.sh`（npm `bundledDependencies`）：组件被塞进 bundle 的 `node_modules/@local/...`，安装时 pnpm 不再回源 registry，单 tgz 离线可装。
- 用户可见变化：插件卡下重新出现两个可独立启停的组件行；关掉注意力组件时待办照常工作，关掉待办组件时注意力的 `enqueue` 策略没有消费者（不产生入队效果），门控与最终拒绝照常。

#### 兼容性

包名不变，版本 `1.4.2-hotfix.1`；1.4.2 的常用设置写进新补丁默认值（`awayRetryCount: 5`、批准/提问超时 `enqueue`、`autoEnqueue: true`、`maxTodos: 200`）。

#### 验证

`node --check` ×4；注意力 19/19、待办 21/21；隔离 profile `--dump-config` 两行 compose；隔离 profile 启动 web 后两条 Client 模块均发布（combo URL 200）。

### 1.4.2（2026-10-27，单包单行）

**结论：三份 README 合并为一份，形态收敛为单包单行**；版本统一为 `1.4.2`，去掉 `-hotfix.N` 后缀。

#### 变更

- **改为单包单行**：`index.js` 同时 `apply` 注意力的 `attention.js` 与待办的 `todo.js`，Client 合并为一个 `client.js`（两个 IIFE 半身：`attentionHalf` / `todoHalf`）。
- 合并配置页：自动入队、批准超时、提问超时、优先推荐（优先推荐与提问超时同一行）。

#### 动机

- 原因：DSH 的 Client 模块按精确包名发现 `dsh.client`，同一包不能出现在两行；当时认为单个自包含 tgz 只能对应「一个包、一行」，两个组件只能共用一行配置。
- 代价：两个组件不再有独立行；关掉「自动入队」不等于关掉待办组件；组件级故障无法隔离。

### 1.4.0-hotfix.1（2026-10-27，预发布）

**结论：首次尝试合并两组件**——先把待办并入注意力单插件，再按验收反馈改为 **bundle + 两个组件** 架构：`@local/dsh-user-attention-core` + `@local/dsh-todo-inbox`，bundle 只做包装与两行 insert。

#### 变更

- 新增合并配置页（bundle 槽位）、批准/提问超时策略、推荐项自动回答。
- 期间包名经历 `@local/dsh-user-attention` → `@local/dsh-user-attention-and-todo-inbox`。

#### 兼容性与迁移

三个包必须分别安装（bundle 的 `dependencies` 当时指向工作区绝对路径 `file:Z:/Project/...`），换机器即失效——1.4.2 的单包化正是为了解决这个问题。

---

## 合并前的两条独立线

`1.4.0-hotfix.1` 之前，「用户注意力」与「待办收件箱」各自是独立插件包、各走各的版本号。以下条目来自当时的
两份组件 README（合并前两个独立包各自的 README；`1.4.2-hotfix.1` 之前全时段的来源口径见文首「记录范围」）
与 `.del/` 下的历史 tgz，只保留可验证的关键变更；每个版本号与其改动点逐条列全。

### 用户注意力（合并前）

版本范围：1.0.0 → 1.3.2-hotfix.2。

| 版本 | 关键变更 |
|---|---|
| **1.3.2-hotfix.2** | 新增 `userAttention` Host 服务（`snapshot` / `subscribe` / `onRejection`）；最终拒绝收敛为统一事件；审批门控消费 `todoInbox.takeApprovalGrant(callId)` 一次性授权；拒绝说明提示模型用 `todo_execute` 执行/取回答案 |
| **1.3.2-hotfix.1** | 修复客户端请求无超时导致活动上报/轮询永久停摆；修复 timed 模式下自动拒绝被官方 `askTimed` 改写为 `{pending:true}`；门控改为状态驱动；手动暂离/离线经 `persistManualState` 跨插件重载与 Host 重启保留；新增离开期间拒绝摘要与 `POST /api/user-attention/ack`；配置保存 409/`force`；配置行按包名定位；提醒过期使用服务端 `now` 计算时钟偏移 |
| **1.3.2** | 修复超时自动拒绝后审批/提问卡片残留：把 `request.signal` 包装为可撤回组合信号，非用户作答收尾时 abort，浏览器撤下卡片；用户作答成功路径不撤回 |
| **1.3.1** | 修复「已自动拒绝人工交互」提示归因：离线时不再套用「多次提醒后仍无响应」文案；`awayRetryCount: 0` 的暂离立即拒绝仍显示「未启用重试」 |
| **1.3.0** | 配置面板新增「暂离重试次数」（0–5）；配置 API 支持 `idleTimeoutMs` 与 `awayRetryCount` 任意组合（至少一个） |
| **1.2.0** | 新增插件配置面板（空闲判断 1–1440 分钟），保存经 `configEditor.edit` 写回 profile；新增 `GET/POST /api/user-attention/config`；修复窄宽度下状态胶囊按钮竖排换行 |
| **1.1.0** | 手动状态粘性 + 隐藏计时：手动选择暂离/离线后冻结自动状态机，直到手动选回在线；胶囊显示「手动」小标签，不显示自动暂离倒计时 |
| **1.0.1** | 修复冷启动竞态：Host 首次装配时 `systemPrompt` / `connection` / `timer` 未就绪导致提示上下文和 `/api/user-attention/*` 路由静默跳过；改用 `ctx.inject` 等待服务 |
| **1.0.0** | 首版 |

### 待办收件箱（合并前）

版本范围：0.1.0-hotfix.1 → 0.1.0-hotfix.5。

| 版本 | 关键变更 |
|---|---|
| **0.1.0-hotfix.5** | `pwsh`/`bash` 非零退出码、signal、超时、abort、stopped 及 `[exit code: N]` 不再被当成成功；失败保留 `failed`，新增 `detectProcessFailure()` |
| **0.1.0-hotfix.4** | 接入 DSH locale（zh/en），界面/按钮/空状态跟随语言；语言切换即时生效；plan-review 英文常量本地化（提交仍用原始 label）；来源会话显示名称、悬停完整 Session ID、点击跳转 |
| **0.1.0-hotfix.3** | 允许/拒绝与「应用」分离；问题卡片完整渲染并支持多问题翻页；输入框高亮条与原生对话框同宽；审批/提问卡片使用原生边框颜色，仅复核项虚线且不可允许 |
| **0.1.0-hotfix.2** | `todo_execute` 返回有界输出预览与问题答案；识别成功工具结果里的业务失败（如 `application:'failed'` / `error.code`），保留 `failed` |
| **0.1.0-hotfix.1** | 首版：自动入队、持久化、`todo_list`/`todo_get`/`todo_execute`、一次性 approval grant、右栏待办页 + 输入框高亮条 + guide 入口、自动入队开关、问题/计划评审回答、拒绝通知、失败保留重试 |

---

## 许可

本文件以 MIT 许可发布，版权归 DX12_XYZ 所有。完整条款见仓库根 [LICENSE](./LICENSE)。

<!-- fused-by: deepseek-v4.1-flash -->

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
