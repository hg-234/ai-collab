# 多智能体小队设计 v2.0：角色分工 · 通信协调 · 任务调度

> v2.0（2026-10-02 修订）｜v1.0 建立于 2026-10-02，经 `DRYRUN-20261002.md` 完整虚拟推演后修订。
> 与 `PROTOCOL.md` v1.1（派单/验收流程）配套：本文件管「谁做什么、怎么通信、怎么调度」。
> **v2 核心变化**：把 v1 的「书面约定」升级为「机器强制」（`ticket.py`），并补齐超时、锁、审计、验收分级、端点分级。
> **决策治理**：重要决定走《多 AI 协同决策框架》`FRAMEWORK.md` —— 独立提案 → 交叉评审 → 迭代≤3 → Decision Record 留痕 → Subtask Card 派发。本文件管分工/通信/调度，FRAMEWORK.md 管"怎么立项决策"。

## 0. 设计原则（来自检索证据，非拍脑袋）

| 原则 | 证据支撑 | 置信度 |
|---|---|---|
| 默认 **Supervisor/Worker**，不要一上来就 Swarm/P2P | 2026 生产默认模式是 supervisor（digitalapplied、atlan 一致） | HIGH |
| **交接必须有显式契约**（输入/输出/错误码/超时/重试） | Alice Labs：每次交接 schema 校验可消除多数级联失败 | HIGH |
| **共享状态单写者** | devsatva："every piece of shared state has exactly one writer" | HIGH |
| **迭代上限 10 + 断路器 3 连败 + 人工升级** | Alice Labs 2026 四层防御 | HIGH |
| **顺序任务禁止并行** | Google/MIT 180 配置实验：顺序任务强行并行退化 39%–70% | MEDIUM-HIGH |
| **显式所有权**（对冲旁观者效应） | 滑铁卢大学 2026.05：责任分散致整体性能下降 | MEDIUM |
| **协商要结构化，不能自由群聊** | 哲学家就餐实验：自由通信反使死锁率 25%→65%（趋同推理） | MEDIUM |
| **默认少 Agent** | Anthropic：多 Agent 耗 ~15x token；生产失败率 41%–87% | HIGH |

### 0.1 v2 新增原则（由干跑暴露的缺陷反推）

| 原则 | 为什么（干跑证据） |
|---|---|
| **约定必须可执行，否则等于没有** | D1/D2：单写者原则被自身设计违反、`review.py` 无测试 exit 0 造成假阳性 |
| **异步协作必须有超时，否则必然悬挂** | D3/N3：派单与回传全靠人肉，无 TTL → 任务静默死亡 |
| **无账本即无问责** | D4：共享根目录非 git（实测），改了什么/谁改的/怎么回滚全无据 |
| **不可用的端点不派单** | A2：向未就绪端点派单必阻塞 → 派单前须**实时探测端口**（Trae:39240 / 豆包:9333 / DeepSeek:9334），不通则报错不假装成功 |

---

## 1. 角色分工（Role Matrix）

| Agent | 角色 | 负责任务 | 明确不负责 |
|---|---|---|---|
| **WorkBuddy** | **编排者 + 验收者**（Orchestrator / Planner / Critic） | 拆任务、派单、维护任务板、逐条核查验收、裁决冲突、最终判定 | 不替 Worker 写产出（避免自审自） |
| **Trae CN** | **代码执行者（主力）** | 读写共享目录/代码仓、写代码、跑测试 | 不做全局编排、不做最终验收 |
| **CodeBuddy CN** | **代码执行者（腾讯 AI IDE）** | 读写共享目录/代码仓、写代码、跑命令、MCP 生态（Figma/GitHub/CloudBase/Tencent Cloud）、CLI 自动化 | 不做全局编排、不做最终验收 |
| **Claude Code** | 代码执行者（复杂/长链重构） | 跨文件重构、长依赖链、深度 agentic 编码 | 同上 |
| **Codex** | 代码执行者（快速/沙箱） | 独立小任务、批量生成 | 同上 |
| **Cursor** | 代码执行者（IDE 快速编辑） | 单文件/局部改动 | 同上 |
| **豆包 Doubao** | **文档/数据/办公执行者 + 研究综合** | 文档、表格、PPT、数据整理、检索综合 | 不做精确定制代码（GUI 自动化弱于编程式） |
| **DeepSeek** | **深度分析 / 推理复核者**（第二独立大脑） | 数理推导、逻辑链复核、第二意见交叉验证、方案交叉评审 | 不做写文件类执行（网页版无本地文件通道） |

### 1.1 端点分级（Tier）—— v2 新增，解决"向不可用端点派单"

| Tier | 端点 | 可用性 | 派单规则 |
|---|---|---|---|
| **Tier 0（自动）** | WorkBuddy（编排/验收） | 恒在线 | 恒为 Dispatcher/Reviewer |
| **Tier 1（自动加载规则）** | Trae / Claude Code / Codex / Cursor / **CodeBuddy CN** | 打开即用，规则自动生效 | 主力 Worker，可直接派单 |
| **Tier 2（半自动）** | 豆包（桌面 Agent，需本地授权 + 手动触发读规则） | 需用户开一次 | 派单时**必须提醒用户去触发**；GUI 执行期间用户勿动键鼠 |
| **Tier 2.5（半自动直连）** | DeepSeek 网页版（Edge CDP @9334，通道已建） | ⚠️ **2026-10-04 实测不可用**：端口在线但页面主内容区未渲染 | **暂停派单**；恢复条件 = `healthcheck.mjs` 报 READY。端口不通或页面 UNUSABLE 均报错（无建票降级） |
| **Tier 3（手动/未就绪）** | （当前无） | — | — |

> Tier 3 端点在完成安装前，路由规则一律跳过；若任务确需其能力，由 WorkBuddy 以自身能力替代或升级人工。

---

## 2. 通信协调（Communication）

### 2.1 双层黑板（v2 修正 D1）

| 层 | 载体 | 性质 | 写者 |
|---|---|---|---|
| **机器态（真源）** | `collab/tasks/<T-ID>/ticket.json` | 每票独立文件，含状态/锁/轮次/产出 | **仅 `ticket.py` 写**（Worker 通过 CLI 改，不手编） |
| **人类视图（只读）** | `collab/TASKS.md` | 由 `ticket.py regen` 生成 | **仅脚本写，任何人不得手改**（手改会被覆盖） |
| **审计账本** | `collab/log.jsonl` | 每次变更一条 JSON（含被拒绝的操作） | 脚本追加，不清空 |

> 修正要点：v1 中所有人直接改同一个 `TASKS.md` → 违反单写者原则、必然丢更新。
> v2 把状态迁到每票独立的 `ticket.json` + 脚本重建视图，**单写者原则真正成立**。

### 2.2 单写者原则（字段级）

| 字段 | 唯一写者 |
|---|---|
| 任务目标/输入/约束/验收标准/分派给 | WorkBuddy（`ticket.py new`） |
| `in_progress`/`done`/产出路径 | 该 Worker（`claim` / `set`，且须为锁 owner） |
| 验收报告、`passed`/`rejected` | WorkBuddy（`set` + `review.py`） |

要修正他人字段 → **追加评论**（`set --note`）或协商文件，不覆写。

### 2.3 交接契约（每票必须齐全，缺一则 `new` 拒绝建票）

1. **输入**：任务目标 + 输入数据（路径或内联）
2. **输出**：产出文件路径 + 格式
3. **约束**：技术栈/禁区/不变量
4. **验收**：可核查（命令/断言/产物/rubric 关键词），禁止"要优雅"
5. **错误与超时**：Worker 遇阻须 `set blocked --note` 回报，**不得静默失败**

### 2.4 协商机制（结构化、有载体、有轮次）

- **触发**：仅当两个 Agent 结论**证据冲突**时。
- **载体（v2 新增）**：`collab/tasks/<T-ID>/negotiation.md` —— 各方写「结论 + 证据 + 置信度」，WorkBuddy 写裁决。**禁止只口头协商不落盘**。
- **形式**：列点式，不做开放式群聊；轮次上限 **2**，超限升级人工。
- 裁决结论同步进 `review_report.md`。

---

## 3. 任务调度（Scheduling）

### 3.1 路由策略
- **能力路由为默认**（代码→Trae 系；文档数据→豆包；深度推理/交叉评审→DeepSeek，走 Edge CDP @9334）。
- **Tier 门禁**：端点未就绪（端口不通）则报错不派单（见 §1.1）。
- **负载感知 tie-break**：同能力多 Agent 空闲时派给在办票最少者。
- **置信度竞价**：仅对不可逆/对外/高成本任务启用。

### 3.2 并行与在办上限
- **顺序依赖子任务禁止并行**（证据：退化 39%–70%）。
- **Fan-out ≤ 3**，且子任务必须真正独立（无共享文件、无依赖）。
- **WIP 上限 2**（v2 新增）：同时在办票不超过 2 张，避免人肉通道成为瓶颈时任务堆积悬挂。
- 默认 1 个 Agent 做完不派 2 个（对冲 15x token 与 41%–87% 失败率）。

### 3.3 锁 · 超时 · 容错（v2 新增，解决 D3/D5）

| 机制 | 规则 | 落地 |
|---|---|---|
| **认领锁** | 一票一 owner；他人认领被拒（锁未过期时） | `ticket.py claim` |
| **心跳 TTL** | 默认 **120 分钟**；Worker 长任务须 `beat` 续期 | `ticket.py beat` |
| **超时回收** | `in_progress` 且超 TTL 无心跳 → 自动回 `todo` 并释放锁 | `ticket.py scan` |
| **迭代上限** | 返工 > 10 轮 → 熔断，拒绝自动返工（退出码 3），升级人工 | `ticket.py set rejected` |
| **dead-letter** | 熔断票置 `cancelled` 并在 `review_report.md` 写明终止原因，不再悬挂 | WorkBuddy 执行 |
| **状态机强制** | 非法流转（如 `done→passed` 跳过 reviewing）直接拒绝 | `ticket.py set` |

```
状态机（v2，含 blocked 与 cancelled）：
todo ─▶ in_progress ─▶ done ─▶ reviewing ─▶ passed（终态）
  ▲          │  ▲          │        │
  │          ▼  │          │        └──▶ rejected ─▶ in_progress（重认领返工）
  │       blocked          │                 └──▶ cancelled（熔断归档，终态）
  └── 超时回收 ◀────────────┘（rejected 亦可直接 claim 重认领）
```

### 3.4 所有权
每票**一个 owner（执行）+ 一个 reviewer（验收）**，禁止"大家一起看"。

### 3.5 验收分级（v2 新增，解决 D2 假阳性）

| 类型 | 自动核查 | 退出码语义 |
|---|---|---|
| **code** | 跑 `test_*.py`（unittest） | 0 全绿 / 1 有失败 |
| **doc** | rubric 关键词核查（`acceptance_keywords`） | 0 达标 / 4 缺关键词 |
| **hybrid** | 两者都跑 | 取最差 |
| **无自动核查项** | —— | **3 = NO_AUTOMATED_CHECK（绝不冒充通过）**，必须人工逐条核并在 `review_report.md` 写证据 |

---

## 4. 协同工作流（v2）

```
用户目标
  ▼
WorkBuddy 判断：真的需要多 Agent 吗？（否 → 自己做，省 15x token）
  ▼ 是
Tier 门禁 → 能力路由 → 检查 WIP ≤ 2
  ▼
ticket.py new（四要素不全 → 拒绝建票，退出码 2）
  ▼
通知用户去对应端触发（Tier2/3 必提醒）
  ▼
Worker: ticket.py claim（抢锁）→ 执行 → beat 续期 → set done
  ▼
WorkBuddy: review.py <dir>（按类型分级核查）
  ├─ 全绿 → set passed，写 review_report.md
  ├─ 有 ❌ → set rejected（写明差距）→ Worker 重认领返工（≤10 轮）
  └─ 证据冲突 → negotiation.md 结构化协商（≤2 轮）→ WorkBuddy 裁决
  ▼
超时兜底：ticket.py scan 定期回收悬挂票；>10 轮 → cancelled 归档
```

---

## 5. 护栏命令速查（`collab/ticket.py`）

```bash
python collab/ticket.py new  --id T-002 --title "..." --assignee doubao \
  --goal "..." --input "..." --constraints "中文|≤3页" \
  --acceptance "含市场分析|含成本测算" --atype doc --keywords "市场,成本" --tier 2
python collab/ticket.py claim --id T-002 --by doubao      # 认领（抢锁）
python collab/ticket.py beat  --id T-002 --by doubao      # 长任务续期
python collab/ticket.py set   --id T-002 --status done --by doubao --output plan.md
python collab/ticket.py list | show --id T-002
python collab/ticket.py scan                               # 超时回收
python collab/ticket.py regen                              # 重建 TASKS.md 视图
python collab/review.py collab/tasks/T-002                 # 分级验收（0/1/3/4）
```

---

## 6. v1.0 → v2.0 变更表

| 缺陷 | v1.0 | v2.0 |
|---|---|---|
| D1 单写者被违反 | 众人手改同一 `TASKS.md` | `ticket.json` 真源 + 视图由脚本重建 |
| D2 验收假阳性 | 无测试 → exit 0 视为通过 | 分级核查，无核查项 exit 3 明确不判定 |
| D3 人肉通道无超时 | 票可永久悬挂 | 认领锁 + 心跳 TTL 120min + `scan` 回收 |
| D4 无账本 | 依赖 git（实际未初始化） | `log.jsonl` 审计（含被拒操作）；git 仍建议但非必需 |
| D5 无并发控制 | 可重复认领 | 锁 + owner 校验 + 非法流转拒绝 |
| D6 协商无载体 | 无处落盘 | `negotiation.md` + 轮次上限 |
| 端点可用性 | 未区分 | Tier 0–3 分级，Tier 3 禁止派单 |
| 僵尸票 | 无处理 | dead-letter：`cancelled` + 终止原因 |

---

## 7. 与既有文件的关系

- `PROTOCOL.md` v1.2.1 — 派单四要素 + 状态机 + 验收规范（流程层）
- `SQUAD.md`（本文件）v2.0 — 角色 + 通信 + 调度（组织层）
- `TASKS.md` — 人类可读任务板（**只读视图**，由 `ticket.py regen` 生成）
- `log.jsonl` — 审计账本（运行层）
- `DRYRUN-20261002.md` — 本次修订的依据（推演记录）
- `AGENTS.md` 第九/十节 — 各端同步到的摘要（分发层）

## 8. 调度器落地状态（2026-10-03，v2.2 —— 真发送闭环已打通）

统一调度器已落地为可执行脚本，WorkBuddy 经它派单：

- `collab/dispatch.py` — 通道路由器（v2.3，2026-10-03）：**trae / doubao / deepseek 三方均 CDP 直连**。doubao 端口不通时降级建票；deepseek 端口不通时明确报错（网页版无本地文件通道）。
- `collab/ticket.py dispatch` — 子命令，封装 dispatch.py（读票取 goal 或直给 prompt，`--dry` 干跑）。
- `collab/trae_send.mjs` — **CDP 注入器（零依赖，Node ≥18 内置 WebSocket）**：`probe` / `dry` / `send` / `read` 四个子命令。`send` 用 **CDP 真实鼠标事件点击发送按钮**（`b.click()` 与纯 Enter 键均会被 React 合成事件忽略，必须派发受信任的 mousePressed/mouseReleased）。
- `collab/cdp.mjs` — 通用 CDP 探测/求值（`probe` 列 target、`eval` 跑 JS），用于排查 Trae 界面元素。
- `collab/shot.mjs` — CDP 截图（`Page.captureScreenshot` → `shot.png`），肉眼判断 Trae 当前界面状态的兜底手段。
- 桌面快捷方式「Trae CN (调试端口)」— 以此启动 Trae 才会带 `--remote-debugging-port=39240`；**注意 Electron 单实例锁：若 Trae 已在运行，快捷方式不会生效，必须先完全退出 Trae**。
- **⚠️ AI 侧无法代启 Trae（2026-10-03 实测）**：Trae 的 `Trae CN.exe` 是启动器，在非 shell 调用（PowerShell `Start-Process` / `cmd /c start`，本环境均受限或报错）下会自行解析参数并拒绝 `--remote-debugging-port`，报 `bad option`。**只有用户在桌面双击快捷方式或用 Win+R 粘贴命令（shell 调用）才能带上调试端口**。因此"启动 Trae"这步必须由用户手动完成，AI 只负责启动后的注入与验收。
- **工作区约定**：Trae 的工作区应切到 `D:\AI\Shared`（共享大脑），而非 `d:\tare训练`。启动命令加位置参数 `D:\AI\Shared` 即可：
  `"D:\Trae CN\Trae CN.exe" --remote-debugging-port=39240 D:\AI\Shared`

**已实测（2026-10-02 23:4x，真实输出）**：
- 端口 39240 监听 ✅（PID 9544，命令行含 `--remote-debugging-port=39240`）。
- CDP 握手 ✅ `Chrome/142.0.7444.235`，`/json/list` 返回 workbench page target。
- 目标选择器 ✅ 输入框 `div.chat-input-v2-input-box-editable`（Lexical 编辑器）、发送按钮 `.chat-input-v2-send-button`、消息区 `.virtualized-message-list-view`。
- 注入 ✅ 干跑：写入 17 字 → 发送按钮由 disabled 变可用 → Ctrl+A/Del 清空复原，**全程未发送**。
- 端到端 ✅ `ticket.py dispatch --to trae --prompt "…" --dry` 全链路通。
- 读取 ✅ `trae_send.mjs read` 成功读出当前会话文本（3108 字符），"拿回结果"通道可用。
- Trae 未启动时 `dispatch --to trae` → 诚实报「端口未监听」，exit 1（不假成功）。
- 豆包降级建票 → exit 0，明确标注「依赖对方常驻任务」，不谎称已执行；deepseek 端口不通 → 明确报错 exit 1。

**DeepSeek 网页版 CDP 直连（2026-10-03 建；2026-10-04 标"暂停"是因运行时未启动+页面未渲染，非代码缺口；2026-10-05 完善后解禁）**：
- 通道：Edge `--remote-debugging-port=9334`（独立 user-data-dir `D:\AI\Shared\chrome-deepseek`）打开 `chat.deepseek.com`。
- 启动器（已验证存在于真实桌面）：`D:\My dog\Desktop\DeepSeek-CDP-9334.lnk`，ARGS = `--remote-debugging-port=9334 --user-data-dir=D:\AI\Shared\chrome-deepseek --no-first-run --no-default-browser-check https://chat.deepseek.com`。双击即带调试端口，不需 Win+R。
- 2026-10-04 的"暂停"根因：端口在线但**会话主内容区未渲染**（body 仅侧边栏、`.ds-markdown` 等命中 0）→ `read` 恒 0。属**页面级故障（未登录态 / 风控 / SPA 崩），不是选择器问题**；`healthcheck.mjs` 已升级为页面级三态判定（READY/UNUSABLE/DOWN），杜绝"端口通=可用"假阳性。
- 2026-10-05 完善项：① 桌面启动器已确认参数正确（无需重建）；② `web_llm_send.mjs` 的 `chat` 改为**轮询等回复**（每 2s 读一次、连续 3 次不增长即停，最长 90s），DeepSeek 长回答不再被固定 14s 截断读不全。
- 注入器：`collab/web_llm_send.mjs --agent deepseek`（统一工厂；`deepseek_send.mjs` 保留作参考）。
- **启用/恢复流程**（用户侧，约 3 分钟）：双击桌面 `DeepSeek-CDP-9334.lnk` → 登录 chat.deepseek.com（若 `chrome-deepseek` 档案登录态失效）→ 跑 `node collab/healthcheck.mjs` 对 DeepSeek 报 `READY` → 即可派单。详见 `collab/deepseek-collab-guide.md`。

**真发送闭环（2026-10-03 00:5x，✅ 已打通）**：
- 工作区确认 ✅ CDP 主页面标题 = `Shared - TraeCode CN`，进程命令行含 `--remote-debugging-port=39240 D:\AI\Shared`。
- 真发送 ✅ `trae_send.mjs send "请只回复两个字 OK，不要读写任何文件…"` → Trae **三次均回复 `OK`**（任务耗时 8s / 6s / 10s），未读写任何文件。
- 读回结果 ✅ `trae_send.mjs read` 自动回退逻辑读到完整对话（提问 + `OK` 回复）。
- **结论：派单（注入）→ 执行 → 读回 双向链路 100% 闭环**。

**⚠️ 关键坑 #2（2026-10-03 实测）—— Trae 默认是 SOLO 智能体模式**：
- Trae 默认运行在 **SOLO 模式**（左侧 `SOLO` 侧栏 + 任务列表），不是普通 IDE 聊天。提交的消息会作为「任务」在**左侧任务列表**里执行并展示（卡片显示任务名 / 耗时 / 结果）。
- 主聊天面板（`.virtualized-message-list-view`）在无焦点时可能只显示欢迎屏「一切就绪，只等你动手」，**故只读主面板会误判为`未发送`**。`read` 已加回退：主面板为空时改读 `[class*="showsoloaisidebar"]` 侧栏。
- **提交必须用 CDP 真实鼠标事件点击发送按钮**：`element.click()`（程序化 DOM 点击）与只发 Enter 键**都不生效**（输入框会清空但不创建任务），React 只认受信任的浏览器派发事件。这是能否真发成功的分水岭。
- SOLO 是 agentic 模式：派单消息**可能被 Trae 直接动手改文件**。派「只回复、不碰文件」这类无害指令时它确实没动文件。

**关键坑 #1（已踩）**：opencli 在 npm 官方源与国内镜像均 404（反向检验过），**Trae 目录只有 `Trae CN.exe`、无 CLI**。故直连方案 = 自研 CDP 注入，不依赖任何第三方包。

**豆包（Doubao）协同通道 —— CDP 直连已打通（2026-10-03，v2.3）**：

- **⭐ 通道模型（重大升级）**：豆包是 **CEF/Chromium 147 内核**，用 `--remote-debugging-port=9333` 启动后**同样可被 WB 用 CDP 直连**——与 Trae 完全同构（同步派单 + 真实点击 + 读回）。**上一版"WB 无法反向驱动豆包 GUI、只能异步轮询"的结论已被实测推翻。**
- **启动方式（必须用户 shell 调用，同 Trae 的 guard）**：Win+R 粘贴
  `"C:\Users\My dog\AppData\Local\Doubao\Application\Doubao.exe" --remote-debugging-port=9333`
  （非 shell 调用会被 launcher 静默拒掉 flag——已实测 `Start-Process` 失败）
- **CDP 实测（✅ 全绿）**：端口 9333 = `Chrome/147.0.7727.149`；主会话页 `doubao://doubao-chat/chat`。
- **关键选择器（与 Trae 不同，务必照用）**：
  - 输入框：`div.tiptap.ProseMirror`（**Tiptap/ProseMirror**，非 Trae 的 Lexical）
  - 发送按钮：**空输入时压根不在 DOM**（不是 disabled！），有输入才渲染 → 用类名 `.send-btn-wrapper` / `.text-g-send-msg-btn-text` 出现即代表可发送。**Trae 那套「disabled→enabled 跳变」判据在豆包完全无效。**
  - 清空：Ctrl+A → **Backspace**（Delete 对 ProseMirror 不灵）
- **注入器**：`collab/doubao_send.mjs`（`probe` / `dry` / `send` / `read`；**每条 CDP 命令独立建连**，避免持久 WS 挂住事件循环）。辅助：`doubao_shot.mjs`（CDP 截图，注意 CEF 后台窗口截图空白）、`doubao_read2.mjs`（读消息气泡）。
- **闭环实测证据** ✅：`doubao_send.mjs send "请只回复两个字 OK，不要读写任何文件…"` → 消息气泡实测 `请只回复两个字 OK，… / 今天 01:38 / OK`。**派单 → 豆包执行 → 读回，双向闭环 100% 坐实。**
- **备用通道（保留）**：共享文件派单（`ticket.py dispatch --to doubao` 真实建票落 `tasks/D-<日期>/ticket.json`）+ MCP 上下文桥（`mcp/http_bridge.py` → `http://127.0.0.1:9999/mcp`，握手测试全绿）。适合豆包未以调试端口启动时的降级场景。
- **⚠️ 设置页无 MCP 入口（仍成立）**：用户截图 + 本地 `_locales/zh_CN` 搜 `MCP|连接器` 零命中双重证实；豆包代码内建 MCP（`s1-mcp-app-node.js` 等 + `MCP=1/CLI=2/MINI_APP` 枚举）但入口由服务端灰度下发、当前未开放。**侧栏可见「定时任务 / 插件·技能·伙伴 / API 服务 / 操作电脑」入口**，以后若需可从这里探。
- **DeepSeek**:✅ 已接入（网页版 CDP @9334，2026-10-03 打通）。Cherry Studio 方案作废——DeepSeek 官方 API 无免费额度，改用网页版免费层直连，零下载零付费零 Key。`dispatch --to deepseek` 现走 CDP 直连。

**CodeBuddy CN（2026-10-06 接入，✅ CDP 直连打通）**：
- **本体**：`E:\CodeBuddy CN\CodeBuddy CN.exe`（腾讯 AI IDE，Electron/VS Code 系；中国版，微信登录、国内端点）。同盘另有国际版 `E:\CodeBuddy\CodeBuddy.exe`（GitHub 登录，校园网对 GitHub 域受限，未接入）。
- **通道**：CDP `@9336`，独立档案 `D:\AI\Shared\codebuddy-cn-profile`（避免与日常 CodeBuddy CN 单例冲突）。启动器：`D:\My dog\Desktop\CodeBuddyCN-CDP-9336.lnk`。
- **注入器**：`collab/codebuddy_send.mjs`（专用驱动，`probe`/`dry`/`send`/`chat`/`read`）。等价入口：`collab/web_llm_send.mjs --agent codebuddy`（工厂内 `import` 同一驱动，不 fork 子进程）。`dispatch --to codebuddy` 走 CDP 直连；端口 9336 不通则明确报错（无降级建票模式）。
- **选择器（2026-10-06 实测，非占位）**：输入框 `div.input-module_editable_gNSDL[contenteditable="true"]`（Slate 编辑器）；发送键 `div[class*="icon-button-module_icon_PJO5B"]`（composer 行最右，tooltip `.cb-tooltip-content` 文本＝「发送」）；回复容器 `[class*=assistantMessage]`；用户消息 `message-timeline-module_userMessage_THCjd`；分组 `group-messages`。
- **闭环实测证据** ✅：`codebuddy_send.mjs chat "终验：只回复【CDP闭环成功】五个字。"` → 读回 `【CDP闭环成功】`。另一次带工具调用的验证：提问要求「以【读回正常】开头 + 当前时间戳」→ CodeBuddy 实际调用 `Get-Date` 取时间后回复 `【读回正常】02:04`，**注入 → 发送 → 工具执行 → 生成 → 读回，全链路坐实**。
- **⚠️ 四个必知坑（改代码前必读 `codebuddy_send.mjs` 头部注释）**：
  1. 聊天 UI 在 **vscode-webview iframe** 内 → 必须连 **wrap 帧会话**（`17tck2vm` / `pjhapb7d8gekf`），page 会话探不到输入框；
  2. 输入框是 **Slate** → 必须经 **React fiber** 取 `editor` 实例调 `insertText()` 写「模型」；设 `textContent`/`InputEvent`/`Input.insertText` 只改 DOM，发送键恒 `disabled`、点了没用；
  3. 点击须用 **wrap 帧自身坐标**（跨源 iframe 会让 page 会话坐标偏移 → 点空）；
  4. 读回须读 **Slate 模型 + `[class*=assistantMessage]`**；读 `innerText` 会被两件事污染：输入框空态的 placeholder「提问或输入"/"快捷命令」(~15字)，以及宽选择器命中 `chat-input-module_*` / 底部 `chat-tips`「内容由 AI 生成，仅供参考」→ 只读到免责声明而非正文。
- **💰 成本**：实测每轮约 **0.64 积分**（Tokens ~15k），走 CodeBuddy 账号额度，**非免费通道**，批量派单前须告知用户。
- **能力定位**：与 Trae 同级代码执行者，额外带原生 MCP 生态（Figma 转码、GitHub、CloudBase、腾讯云）。CLI 版 `codebuddy-code`（`-p` headless / `--serve` / `daemon`）本机未装，仅接 IDE 形态。
- **复用同一账号额度**：CLI / IDE / 插件共享 CodeBuddy 账号资源配额（官方说明），多形态并存不额外计费但共用额度。

**待你手动**：
1. Trae：保持以「Trae CN (调试端口)」快捷方式启动（普通图标启动则 CDP 不可用）。
2. 豆包：保持以带 `--remote-debugging-port=9333` 的方式启动（Win+R 那条命令），WB 即可 CDP 直连派单。
3. DeepSeek：双击桌面 `DeepSeek-CDP-9334.lnk`（已带 `--remote-debugging-port=9334` + `chrome-deepseek` 档案）打开 `chat.deepseek.com` 并登录；WB 即可 CDP 直连派单。跑 `node collab/healthcheck.mjs` 看到 DeepSeek `READY` 即通。
