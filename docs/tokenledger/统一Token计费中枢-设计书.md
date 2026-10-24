# 统一 Token 计费中枢 · 产品与技术设计书

> 版本：v1.0 评审稿（2026-10-24）｜ 前置文档：《统一Token计费中枢-立项报告书 v2.1》
> 一句话定位：**cc-switch 级的计费体验 × 全桌面（Windows + WSL + 积分制工具）计量覆盖**。
> 设计蓝本：[cc-switch](https://github.com/farion1231/cc-switch)（MIT）——借鉴其信息架构与交互，不复用其代码（技术栈不同）；不做其供应商切换/路由核心，只做计费。

---

## 0. 设计原则

| # | 原则 | 含义 |
|---|---|---|
| P1 | 三问闭环 | 任何界面必须能回答：**花了多少**（token/钱/积分）、**还剩多少**（额度/余额）、**何时重置** |
| P2 | 可信分级可见 | `exact / vendor / estimated` 三级置信度贯穿所有数字，badge 永不隐藏 |
| P3 | 采集零打扰 | 只读本地文件优先；厂商 API 只读轮询；**绝不做请求代理/注入**（与 cc-switch 路由核心划清界限） |
| P4 | 轻常驻 + 重按需 | 悬浮窗承担 80% 日常查看；主面板按需打开 |
| P5 | 中文优先 | zh 为第一语言，en 就绪，i18n 键从第一天提取 |
| P6 | 单机完整 | 断网、单机时功能完整；多机汇聚是增强不是依赖 |

---

## 1. 蓝本研究：cc-switch 做对了什么，我们各取什么

| cc-switch 特性 | 本项目处理 |
|---|---|
| 用量仪表盘：按供应商/模型统计请求数、token、缓存命中率、输出速度、花费；趋势图；全年热力图；单请求日志 | ✅ **全面借鉴**（§3.1/§3.2），数据面从"自有路由 + 部分扫描"扩到 46+ 工具 + WSL + 积分制 |
| 额度卡片："5h 94% left"、滚动窗口进度条、重置倒计时、余额；覆盖订阅/套餐/余额三类 | ✅ **借鉴为"额度中心"页**（§3.3），含 Coding Plan 5h/周/月窗口与 Codex Credits 式积分余额 |
| "Configure usage query"：卡片上配自定义查询脚本 | ✅ 借鉴：额度卡片支持自定义脚本数据源（逃生舱，厂商接口变了不断更应用也能救） |
| 自定义定价：每模型价格 + models.dev 导入 | ✅ 借鉴（§3.4/§6）：定价三源 = models.dev 导入 / LiteLLM / 手动，优先级可调 |
| 托盘：每工具一行 `name · mode · provider · quota` | ✅ 借鉴（§5），行内容改为 `工具 · 今日token · 花费/积分 · 额度%` |
| 会话管理（结构化阅读/搜索/导出 Markdown） | ⏸ **backlog**，v1 不做（避免 scope creep；立项报告已列） |
| 供应商切换、本地路由、聚合、故障转移 | ❌ **不做**——那是它的核心业务，与计费正交；做了反而引入代理风险，违反 P3 |
| MCP/Skills/Prompts 集中管理 | ❌ 不做（与 ZCode 自身能力重复） |
| 云同步（WebDAV/S3）、自动备份、原子写入、深浅色、多语言、自动更新 | ✅ 工程实践全盘借鉴 |
| Tauri 2 + Rust + React 18 + TS + SQLite | ⚠️ **栈不跟随**：我们 fork token-monitor（Electron+TS）保采集面；UI 层用 React 18 + Tailwind + Vite 平移 cc-switch 观感（同为 React，设计可直接映射）。附录 A 论证未来 Tauri 迁移路径 |

**结论**：cc-switch 证明了"卡片 + 仪表盘 + 托盘行"这套计费信息架构的成熟度；它的局限（只计自己接入的 API/路由流量，且无积分↔token 换算）正是本项目的全部差异化。

---

## 2. 产品形态与信息架构

### 2.1 三形态

```
┌─ 主面板（重型，按需）─────── Electron 主窗口，侧边导航
├─ 悬浮窗（轻量，常驻）────── 角落账单卡片（非灵动岛形态，见 §4）
└─ 托盘（系统级，常驻）────── 每工具一行 + 快捷菜单（见 §5）
```

### 2.2 主面板信息架构（v1.1 修订：cc-switch 导览制）

**导览完全仿 cc-switch：左侧一栏，工具在上、功能在底；不做"工作台式"功能导航。**

```
左侧边栏（232px）
├─ 品牌 TokenLedger（折叠按钮 «）
├─ 【工具】全部工具（默认选中）
│   ├─ Claude Code / Codex / ZCode / OpenCode / Gemini CLI
│   └─ WorkBuddy / Trae·CN / DeepSeek DSH …（右侧对齐显示该工具今日花费）
│   └─ 点击任一工具 = 用量/额度/积分全站按该工具过滤
├─ 【功能】（底部，cc-switch 同位）
│   ├─ 用量统计（选中态，挂"今日 $x"胶囊徽标）
│   ├─ 额度中心 / 积分与换算 / 报表 / 设置
└─ 侧栏底部常驻"今日账单"胶囊（同悬浮窗/托盘数据源）

主内容区 · 用量统计页（首页，cc-switch 用量页同构）
├─ 顶栏：标题 + 同步状态 + 立即同步 + 自动刷新30s + 数据来源
├─ 筛选行：时间段（今日/本周/本月/全部）+ 模型▾ + 机器（本机+WSL）▾ + 置信度三开关
├─ 四联统计卡：总成本 / 总请求数 / 真实消耗 Tokens / 缓存命中率（+「更多指标▾」）
├─ 用量热力图：26~53 周，Tokens / 请求 / 成本 三 Tab，少→多图例
└─ 请求日志表：请求日志/供应商/模型/定价 Tab；列＝时间·工具·模型·新增输入·输出·缓存命中·成本·速度·积分·置信度
其余功能页：额度中心 / 积分与换算 / 报表 / 设置（采集器、定价源、数据、同步、外观）
```

---

## 3. 主面板页面级设计

> 通用规范：Tailwind；数字 tabular-nums；所有表格列可显隐、可排序、可过滤；产品支持深浅色跟随系统（**评审稿钉浅色**）。
>
> **视觉令牌表（v1.2：cc-switch 截图实测提取，开工直接引用）**
>
> | 类别 | 值 |
> |---|---|
> | 底色 | 主背景 `#FBFBF9`；卡片与侧栏 `#FFFFFF`（侧栏不置灰，靠 `#E5E5E3` 分隔线区分）；表格行线 `#EEEEEC` |
> | 文字 | 主 `#1A1A1A` / 次级 `#3C3C3A` / 弱 `#5C5C5A`（表头）/ faint `#8A8A88`（meta、速度列） |
> | 交互态 | 列表选中块 `#E9E9E7` r8；hover `#F2F2F0`；分段控件容器 `#F2F2F0`、选中项白底+`0 1px 2px rgba(0,0,0,.07)` 微阴影；下拉白底 1px 边框高 32 |
> | 强调色 | teal `#1B6E8C`（唯一主强调）；"今日"徽标红 `#F45B5B`（全 UI 唯一暖色点缀）；语义：绿 `#15803D` / 琥珀 `#B45309` / 蓝 `#1D4ED8` / 负积分 `#B91C1C` |
> | 热力图 | 六档 teal 阶梯 `#E8F4F6 → #C3E3EA → #8FCBD8 → #5AAFC4 → #2E8FAE → #1B6E8C`；空格 `#F7F7F5`；格子 13px r3 间距 3；顶部月份标签、左侧一/三/五/日、右下图例"少→多" |
> | 字号 | 正文 12.5–13 / 页标题 17 semibold / KPI 数字 30 semibold / 表头 12.5 常规灰 |
> | 结构 | 半径：控件 8 / 卡片 12 / 徽标胶囊 999；卡片无阴影（仅悬浮窗/托盘浮层带投影）；KPI 卡两行式（13px 标签 + 30px 数字，无图标无子行）；表格左对齐、行高≈50、无斑马纹、成本列 600 字重、速度列灰字带 ⓘ |
> | 置信度徽章 | exact 绿 / vendor 蓝 / estimated 琥珀；白底 6% 同色tint、同色 32% 描边、r999、11px |

### 3.1 用量统计（首页，cc-switch 用量页同构）

```
┌───────────────────────────────────────────────────────────────┐
│ 📊 用量统计   会话日志·刚刚同步 [立即同步] [自动刷新30s▾][数据来源]│
├───────────────────────────────────────────────────────────────┤
│ [今日|本周|本月|全部] [模型▾] [本机+WSL▾]      exact vendor est  │
├─────────────┬─────────────┬─────────────┬─────────────┬───────┤
│ 总成本 $61.40│ 总请求数1284 │ 真实消耗18.6M│ 缓存命中68.2%│更多▾  │
├─────────────┴─────────────┴─────────────┴─────────────┴───────┤
│ 用量热力图  [Tokens|请求|成本]              最近26周 · 周/日    │
│ □□□▦▦▩▩…（teal 透明度阶梯，GitHub 风格）        少 □▦▩█ 多     │
├───────────────────────────────────────────────────────────────┤
│ [请求日志|供应商|模型|定价]                        [全部状态码▾] │
│ 时间|工具|模型|新增输入|输出|缓存命中|成本|速度|积分|置信度       │
└───────────────────────────────────────────────────────────────┘
```

- 页面骨架 = cc-switch 用量页逐项对应：四联统计卡（含"真实消耗 Tokens / 缓存命中率"这两个它特有的指标名，直接沿用其口径命名）、热力图三 Tab、请求日志表。
- 本项目加法（视觉保持安静，不破坏原语言）：**积分列**（credit 工具填值，其余 —）、**置信度徽章列**、筛选行的 **机器维度（本机 + WSL）** 与置信度三开关。
- 额度信息不在本页占版面（cc-switch 用量页同样不放）——聚合视图由悬浮窗预警行/托盘承担，明细在额度中心页。
- "更多指标▾"展开：模型分布、输出速度分布、按项目 Top、按会话 Top（即原"今日 Top 会话/项目表"降级为展开项，页面首屏保持 cc-switch 式克制）。
- 交互：时间/模型/机器/置信度为全局过滤器；点击任一行进入 §3.2 用量明细（全量抽屉）。

### 3.2 用量明细

| 列 | 说明 |
|---|---|
| 时间 / 机器 / 工具 / 会话 / 项目 / 模型 | 维度列；会话列可下钻（同 session 聚合展开） |
| in / cache R / cache W / out / reasoning | token 五列，默认隐藏 cache W、reasoning |
| 费用 $ | 按生效 pricing profile 计算 |
| 积分 | credit 工具专属（WorkBuddy/Trae），无则 — |
| 置信度 | badge |

- 虚拟滚动（10 万行不卡）；行点击 → 右侧抽屉显示该会话聚合 + 原始记录样本（脱敏）。
- 导出：CSV / xlsx（当前过滤结果）。
- 错误态：某 adapter 失效时表头出现琥珀横幅"ZCode 采集器自 10-24 14:00 起失败：格式变更"，点击跳设置页。

### 3.3 额度中心（cc-switch 供应商卡复刻：单列堆叠制）

```
⋮⋮ [C] Claude Pro  [订阅]   已使用 5h窗口 62%  剩余 3h12m    刚刚 ⟳ ⚙
     ────────────────▓▓▓▓▓▓░░░░░░░░ 4px 细条 ─────────────────
     5h 滚动窗口 · 数据源 API · 上次 09:41
⋮⋮ [W] WorkBuddy  [积分]    已使用 988  剩余 512 credits      刚刚 ⟳ ⚙
     ▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░（绿色）
     月度 1500 额度 · 今日 −38.2 · 数据源 本地（exact）
```

- 结构完全照搬 cc-switch 供应商卡：**单列堆叠 + 拖拽手柄（排序）+ 品牌色 logo 方块 + 名称 + 右侧"刚刚 / 刷新 / ⚙"**；余额沿用其**文字式**表达——"已使用（灰）剩余（绿加粗）"，不用大百分比大进度条。
- 本项目加法（克制）：类型徽章（订阅/套餐/积分/余额/额度）与卡底 4px 细进度条（滚动窗口可视化）；多窗口套餐（GLM 5h+周）并列两条细线。
- 每卡 `⚙` = Configure usage query（抄 cc-switch）：内置端点或自定义脚本（stdin JSON → stdout 标准结构），厂商接口变更时的逃生舱。
- 数据源标记 `API`（vendor）/`本地`（exact）；刷新节流 30s±30% jitter。

### 3.4 积分与换算（本项目差异化页）

```
┌─ 积分流水 ────────────────────┐ ┌─ Conversion Profile ─────────┐
│ 10-24 14:02 WorkBuddy          │ │ 工具/模型        系数   来源  │
│  deepseek-v4.1-flash  -3.7     │ │ WB/hy4-preview  1.02/M  回归  │
│ 10-24 13:47 WorkBuddy          │ │ WB/ds-v4.1-fl   0.87/M  回归  │
│  space-bunny         -25.8     │ │ Trae/任意       0.05/req 估*  │
│ …（credit_json 爆炸而来）      │ │ [+ 从定价页导入] [锁定官方值] │
└───────────────────────────────┘ └─────────────────────────────┘
┌─ 校准状态 ────────────────────────────────────────────────────┐
│ WorkBuddy 回归: 样本 45 会话 · 312 请求 · R²=0.94 · 更新 10-24 │
│ [重跑] [查看散点]   Trae: 无本地明细 → 依赖 profile(estimated) │
└──────────────────────────────────────────────────────────────┘
```

- 系数来源三级：`官方`（厂商公布，锁定）/ `回归`（本地双轨数据拟合，显示 R²）/ `估`（手工，永远 badge 为 estimated）。
- 回归仅对存在双轨数据的工具可用（当前仅 WorkBuddy；LiteLLM 网关接入后扩展到所有走网关的工具）。

### 3.5 报表

- 月度账单：按工具/项目/模型的 token·费用·积分汇总表 + 订阅 vs API 消耗对比（回答"我这几个订阅值不值"）。
- 导出：xlsx（先做）/ pdf（后做）；文件名 `bill-2026-10.xlsx`。

### 3.6 设置（要点）

- 采集器页：每 adapter 一行——开关 / 健康灯（绿/琥珀/红）/ 上次成功时间 / 水位值 / 路径覆盖输入框 / [立即同步] / [重扫全部]（危险操作，二次确认）。
- 数据页：目录、保留策略（明细 180 天 + 聚合永久，默认）、备份（每日 1 份留 10 份，抄 cc-switch）。
- 同步页：hub 地址 + 密钥；WSL agent 状态与安装引导；设备名。

---

## 4. 悬浮窗设计（非灵动岛形态，用户指定）

### 4.1 定位与形态

常驻桌面的**角落账单卡片**——固定矩形、安静、信息密度高；不做顶部居中弹出式灵动岛，不做表情/宠物/动画。

**展开态（默认，260×96px，圆角 10，右侧留 8px 呼吸）：**

```
╭──────────────────────────────────────╮
│ 今日账单            ⌃折叠  ⚙  ✕      │
│ 1.24M tok · $3.87 · 积分 -38.2  [绿] │
│ ▁▂▂▄▆█▅▃▂▁▁▂▄ (24h sparkline)        │
│ ⚠ Claude Pro 5h 窗口 62% · 3h12m 重置│
╰──────────────────────────────────────╯
```

**折叠态（260×30px 单行）：**

```
╭──────────────────────────────────────╮
│ $3.87 · 1.24M tok · 积分38  ⌄展开     │
╰──────────────────────────────────────╯
```

### 4.2 行为规格

| 项 | 规格 |
|---|---|
| 默认位置 | 右下角（距边 16px），记忆用户拖放位置 |
| 吸附 | 拖拽松手吸附最近屏幕角/边 |
| 折叠/展开 | 卡片右上 `⌃/⌄` 或双击切换；点击主行打开主面板总览 |
| 额度预警行 | 仅当某额度进入策略阈值（默认 <25% 或 <2h 重置）时出现第三行，平时两行 |
| 置顶/透明度 | 默认 always-on-top；失焦透明 65%，悬停 100%（可关） |
| 刷新 | 30s±30% jitter，本地读库，无网络请求（网络只在额度卡片刷新时） |
| 分工具明细 | 展开态点击任一数字 → 迷你列表（本工具今日 top3 + badge），不弹大窗 |
| Tab 切换 | 展开态数字行支持 Tab：Token / 费用 / 积分（记忆最后所用） |
| 主题 | 跟随主面板（深/浅），透明度独立调节 |
| 开机自启 | 默认开（托盘常驻 + 悬浮窗记住上次状态） |
| 多显示器 | 记忆所在显示器；不出现在任务栏（skipTaskbar） |

### 4.3 实现要点

- Electron `BrowserWindow`：`frame:false, alwaysOnTop, skipTaskbar, transparent`（Windows 下 transparent+resize 兼容性差 → 用背景色圆角代替真透明，写死规范避免坑）。
- 数据：只读聚合库的 `today_view` 物化视图，悬浮窗进程零解析（保证 P4 轻量）。

---

## 5. 托盘设计

```
🌀 Token 计费中枢        ← 总计行: $3.87 · 1.24M · 积分38
  Codex      412k · $1.90 · —
  ZCode      388k · $1.12 · —
  WorkBuddy   96k ·  —    · -38.2
────────────────────
打开主面板 / 悬浮窗开关 / 立即全部同步 / 设置 / 退出
```

每活跃工具一行（无数据不显示），点击行打开该工具的明细页（过滤态）。

---

## 6. 数据模型（DDL 级，SQLite）

```sql
-- 归一用量明细（append-only，唯一键防重）
CREATE TABLE usage_record (
  id INTEGER PRIMARY KEY,
  ts INTEGER NOT NULL, machine TEXT NOT NULL, tool TEXT NOT NULL,
  session_id TEXT, project TEXT, model TEXT,
  in_tok INTEGER, cache_read INTEGER, cache_write INTEGER,
  out_tok INTEGER, reasoning INTEGER,
  cost_usd REAL, credits REAL,
  confidence TEXT NOT NULL CHECK(confidence IN('exact','vendor','estimated')),
  dedup_hash TEXT UNIQUE NOT NULL  -- hash(machine,session,seq|ts,tool)
);
CREATE INDEX idx_usage_ts ON usage_record(ts);
CREATE INDEX idx_usage_tool_ts ON usage_record(tool, ts);

-- 积分流水（WorkBuddy credit_json 爆炸 / Trae 扣减事件）
CREATE TABLE credit_event (
  id INTEGER PRIMARY KEY, ts INTEGER NOT NULL, machine TEXT, tool TEXT NOT NULL,
  account TEXT, session_id TEXT, model TEXT,
  credits REAL NOT NULL,           -- 负数=消耗
  balance_after REAL,              -- 可得时填
  source TEXT NOT NULL,            -- 'local-db' | 'vendor-api' | 'manual'
  dedup_hash TEXT UNIQUE NOT NULL
);

-- 账户与额度快照
CREATE TABLE provider_account (
  id TEXT PRIMARY KEY, name TEXT NOT NULL,
  kind TEXT NOT NULL CHECK(kind IN('subscription','plan','balance','credits')),
  tool TEXT, query_config JSON,    -- endpoint/脚本配置（Configure usage query）
  enabled INTEGER DEFAULT 1
);
CREATE TABLE quota_snapshot (
  id INTEGER PRIMARY KEY, ts INTEGER NOT NULL, account_id TEXT NOT NULL,
  window_kind TEXT,                -- '5h'|'weekly'|'monthly'|'none'
  used_pct REAL, remaining REAL, reset_at INTEGER,
  confidence TEXT NOT NULL
);

-- 换算与校准
CREATE TABLE conversion_profile (
  tool TEXT NOT NULL, model TEXT NOT NULL,
  credits_per_mtok REAL, unit TEXT,                -- 或 credits_per_req
  source TEXT NOT NULL CHECK(source IN('official','regression','manual')),
  locked INTEGER DEFAULT 0,
  PRIMARY KEY(tool, model, unit)
);
CREATE TABLE calibration_run (
  id INTEGER PRIMARY KEY, ts INTEGER NOT NULL, tool TEXT, model TEXT,
  samples INTEGER, r2 REAL, slope REAL, intercept REAL, notes TEXT
);

-- 采集器与同步
CREATE TABLE adapter_state (
  adapter TEXT PRIMARY KEY, machine TEXT NOT NULL,
  enabled INTEGER DEFAULT 1, healthy INTEGER,
  last_ok_ts INTEGER, watermark JSON,      -- 各源独立水位
  path_overrides JSON, last_error TEXT
);
CREATE TABLE sync_peer (
  machine TEXT PRIMARY KEY, name TEXT, last_seen INTEGER,
  last_seq INTEGER, auth_hash TEXT
);
CREATE TABLE pricing_model (        -- models.dev/LiteLLM 导入 + 手动覆盖
  model TEXT PRIMARY KEY, in_usd REAL, out_usd REAL,
  cache_read_usd REAL, cache_write_usd REAL, source TEXT, updated_at INTEGER
);
```

物化视图：`today_view` / `month_view`（悬浮窗与托盘直读，避免实时聚合）。

---

## 7. 采集层规格（按立项报告 §2 的源清单落细）

| Adapter | 类型 | 触发 | 水位 | 降级路径 |
|---|---|---|---|---|
| zcode | file-parser | 文件 watch + 5min 兜底 | 文件 mtime+行数 | vendor→无（纯本地） |
| claude / codex | file-parser | 同上 | 同上 | 无 |
| opencode | file-parser(db) | 5min 定时 | db rowid/时间戳 | 额度页仍可用 API |
| opencode-quota | vendor-api | 30s±jitter | — | 标 stale |
| gemini / openclaw | file-parser | 5min | 同 zcode | 无 |
| dsh | file-parser(zstd) | 5min | 目录 mtime | 解压失败→跳过并告警 |
| copilot | vendor-api | 10min | — | 标 stale |
| **workbuddy** | hybrid(本地db) | 30s | `session_usage.updated_at` | 库结构变更→格式探测告警 |
| trae / trae-cn | vendor-api(cookie) | 10min | 事件游标 | cookie 失效→置灰+引导重同步 |
| cursor | vendor-api | 10min | usage-events 游标 | 同上 |
| wsl-agent | 汇聚通道 | 长连接推送 + 60s 心跳 | 各源水位随行携带 | 断连→缓存最后快照 |

**WorkBuddy adapter 详设**（积分↔token 校准的数据源）：
- `session_usage.credit_json` → 逐键爆炸为 `credit_event`（credits 取负）；键哈希与 `updated_at` 组成 dedup_hash，幂等重放。
- `session_usage.used` → 会话级 token 累计（口径按"上下文窗口用量"处理，字段语义开关可翻转）。
- `sessions.model` → 模型归一表：`hy4-preview*→hunyuan系`、`deepseek-v4.1-flash→ds系`、`space-bunny/balanced-model→别名映射（可编辑）`。
- 校准任务：按 (tool, model) 对 credits~tokens 做线性回归，样本 ≥50 且 R²≥0.9 才允许写入 `conversion_profile(source='regression')`。

**WSL agent 协议**：WSL 内 headless Node 进程（fork 自带），`POST /ingest`（批量 UsageRecord + 各 adapter 水位 + HMAC 签名），hub 侧验签入库；同时上报 WSL 侧额度 API 结果（如 WSL 内 OpenCode）。

---

## 8. 技术栈与工程结构

```
fork: Javis603/token-monitor (Electron+TS)
改造后 monorepo（pnpm workspace，对齐 cc-switch 工程风格）:
├─ apps/desktop        # Electron 主进程 + React 18 + Tailwind + Vite 前端（UI 重做）
│   └─ src/windows/    # main / floating / tray 三窗口
├─ packages/core       # 聚合、物化视图、校准回归、定价引擎（纯 TS，零 Electron 依赖）
├─ packages/adapters   # §7 全部 adapter（每目录一个 + fixtures 黄金样本）
├─ apps/hub            # 多机汇聚（fork 自带，自建 NAS / CF Worker）
└─ .zcode/             # 项目级 skills/commands（§9.2）
```

- 状态：zustand；图表：**recharts**（默认，与 fork 兼容；备选 ECharts，见 §11 决策点）。
- 测试：Vitest（core/adapters 单测 + 黄金样本回归）+ Playwright（面板/悬浮窗 E2E）。
- 构建：electron-builder（fork 现成）+ GitHub Actions（win 优先，mac 后补）。

---

## 9. 开工依赖清单（预置，不边做边找）

### 9.1 MCP servers（项目 `.mcp.json`，克隆即生效）

| MCP | 用途 | 触发场景 |
|---|---|---|
| `csv-mcp`（npx） | 导出账单 CSV 后 SQL 复核 | 报表页开发、数据对账 |
| `token-optimizer`（npx） | 大日志/大 JSONL 省 token 读取 | adapter 开发期读样本 |
| `obsidian`（HTTP 127.0.0.1:27200，本机已运行） | 知识镜像发布 | 结题/里程碑时镜像到 `E:\1obsidian` |

```json
// .mcp.json（放仓库根，随 clone 生效）
{
  "mcpServers": {
    "csv-mcp": { "command": "npx", "args": ["-y", "csv-mcp-server"] },
    "token-optimizer": { "command": "npx", "args": ["-y", "codex-token-optimizer"] }
  }
}
```
> obsidian 为用户级 HTTP 服务，不进项目配置，避免泄漏端口。

### 9.2 ZCode Skills / Plugins（开发期，项目 `.zcode/skills/` + 本机已装官方插件）

| 项 | 来源 | 用途 | 用在哪一步 |
|---|---|---|---|
| `frontend-design` | 本机 skill | 面板/悬浮窗视觉定制 | M3 UI 重做 |
| `visualize` | 官方插件 | 会话内图表原型（趋势/热力图草稿） | M1–M3 设计验证 |
| `browser-use:web-gui-tester` | 官方插件 | Trae/Cursor 登录同步流 + 面板 E2E 黑盒 | M2 额度 adapter |
| `dynamic-workflows` | 官方 skill | adapter 并行回归测试编排 | M1 起、每次改 parser |
| `spreadsheets:xlsx` | 官方插件 | 月度账单 xlsx 生成实现参考 + 导出验证 | M3 报表 |
| `pdf:pdf` | 官方插件 | pdf 账单（后置） | M3+ |
| `humanizer-zh` | 本机 skill | 面板文案/报表措辞 | M3 |
| `zcode-guide:diagnosing-mcp` / `diagnosing-plugins` / `diagnosing-skills` | 官方插件 | 调 `.mcp.json`/插件装载问题 | 全程 |
| `skill-creator` | 官方插件 | 制作项目自带 `/bill` `/quota` 命令 | M1 |

**项目自带 slash 命令（`.zcode/commands/`，M1 交付）**：
- `/bill [今天|本周|本月]` → 调 CLI 打印文本账单（开发期无 GUI 也能用）
- `/quota` → 全账户额度快照 + 异常标注
- `/rescan <adapter>` → 指定采集器重扫（带二次确认）

### 9.3 运行时依赖（用户机器零预装，postinstall 自动拉取）

| 依赖 | 拉取方式 | 说明 |
|---|---|---|
| tokscale 二进制 | fork 内置下载器 + 启动时版本检查 | 解析内核 |
| ccusage | `npx ccusage@latest`（仅交叉验证命令，不进主链路） | 对账 |
| better-sqlite3 | npm 依赖（prebuilt） | 聚合库 |
| fzstd | npm 依赖（纯 JS） | dsh zstd 解压 |
| LiteLLM | `uvx litellm`（v3，可选装） | 校准网关 |

```bash
# postinstall 伪代码
1. node scripts/check-env.js        # Node>=22, Win10+；缺则报中文指引
2. node scripts/fetch-tokscale.js   # 按平台下载内核 → vendor/，校验 sha256
3. electron-builder install-app-deps
4. echo "✔ 依赖就绪：npm run dev"
```

### 9.4 开发工具链（一次配齐）

Node 22 LTS + pnpm 9（workspace）+ Vitest + Playwright + electron-builder + GitHub Actions（win/mac 构建）+ conventional commits + Changesets（版本）。
**不需要**：Rust 工具链、Python（v3 LiteLLM 阶段除外，uvx 自管）。

### 9.5 `setup.ps1` 一键开工检查清单

```
□ Node 22 / pnpm 9         □ Git + gh 已登录
□ 仓库克隆到 E:\1project\token-billing-hub（命名见 §11-8）
□ GitHub remote 已建 + 首推（AGENTS.md 硬性规则）
□ .mcp.json 生效（ZCode 重启后 csv-mcp 可用）
□ vendor/tokscale 下载校验通过
□ fixtures 黄金样本就位（ZCode/Claude/Codex/OpenCode/WorkBuddy 各≥1 脱敏样本）
□ Vitest 冒烟全绿
```

### 9.6 工程规范（继承 AGENTS.md）

正式仓库 `E:\1project\`；GitHub remote + 首推验证；可复用知识镜像 `E:\1obsidian\Obsidian Vault`（链回 canonical）；对话中文、代码/commit 英文。

---

## 10. 里程碑与设计验收门（在报告 M0–M4 上加设计门）

| 阶段 | 新增设计验收门 |
|---|---|
| M0 fork 验证 | 五源出数 + dedup_hash 幂等重放测试通过 |
| M1 MVP | 明细页 10 万行虚拟滚动 <16ms 帧；水位增量：5.7GB opencode.db 二次扫描 <5s |
| M2 积分层 | WorkBuddy 校准样本 ≥50、R²≥0.9 才生效；confidence badge 全链路可见 |
| M3 体验 | 悬浮窗展开态首绘 <1s；常驻内存增量 <80MB；拖拽吸角落 E2E 通过；托盘行与悬浮窗数字一致性测试 |
| M4 校准 | 回归系数随网关双轨数据收敛（斜率漂移 <5%/周） |

---

## 11. 开放决策点（开工前请拍板，均已给默认值）

| # | 决策 | 默认值 | 备选 |
|---|---|---|---|
| 1 | 悬浮窗默认形态 | 右下角 260×96 展开态 | 左下/右上、默认折叠 |
| 2 | 图表库 | recharts | ECharts（更强但体积大） |
| 3 | 报表导出顺序 | xlsx 先、pdf 后 | 同时 |
| 4 | 数据保留 | 明细 180 天 + 聚合永久 | 全永久 |
| 5 | 会话阅读器 | backlog（v1 不做） | 提进 v1 |
| 6 | 主题默认 | 跟随系统 | 强制深色 |
| 7 | 多机同步通道 | 自建 hub（NAS） | Cloudflare Worker |
| 8 | 项目命名 | **TokenLedger** | BillDeck / OmniMeter（仓库与品牌需一致，GitHub 查重后定） |
| 9 | 额度预警阈值 | <25% 或 <2h 重置 | 自定义 |
| 10 | cc-switch 会话管理 | 不做 | 若强需求，M4 后评估 |

---

## 12. 设计相关风险补充

| 风险 | 缓解 |
|---|---|
| Electron 常驻内存偏重，与"轻量"预期冲突 | 悬浮窗进程只读物化视图零解析；若仍不达标，附录 A Tauri 迁移路径（前端 React 层平移） |
| Windows frameless 透明窗口兼容性坑 | 规范禁止真透明，用背景色圆角；提前做 Win10/11 双版本 E2E |
| cc-switch 设计借鉴边界 | MIT 允许；只对齐信息架构/交互模式，不拷贝视觉资产与文案；品牌独立命名 |
| 自定义 usage query 脚本 = 任意代码执行面 | 脚本仅本机手动添加、明示风险、设置页可审计列表 |

---

## 13. 代码适配性与移植性（面向后续移植）

### 13.1 分层耦合矩阵——每层"绑在哪、怎么拆"

| 层 | 当前实现 | 宿主依赖 | 移植改动点 | 移植成本 |
|---|---|---|---|---|
| `packages/adapters`（§7 全部采集器） | 纯 TS | 仅 Node `fs` / `child_process`（fzstd 纯 JS、tokscale 子进程） | **零改动**可跑在 Node / Bun / Deno / 任何 sidecar | 无 |
| `packages/core`（聚合/回归/定价/物化视图） | 纯 TS + better-sqlite3 | Node（better-sqlite3 为原生模块） | Tauri：换 `tauri-plugin-sql` 或 libsql；浏览器只读：sql.js/wasm。DB 访问已收敛到 repository 层，只改一处 | 低 |
| UI（React 18 + Tailwind + zustand） | DOM | 无（硬规约：组件内禁止 `import electron`，窗口能力一律走 `IShell` 桥接口） | 任何 webview（Tauri/Wails/纯浏览器）直接渲染 | 无 |
| 壳层（窗口/托盘/悬浮窗/自更新） | Electron `BrowserWindow`/`Tray`/`autoUpdater` | **强绑 Electron（唯一重绑层）** | 抽象 `IShell{createWindow/setAlwaysOnTop/setTray/notify...}`；Tauri 侧等价 API 实现约 300–500 行 | 中，集中在此 |
| 文件监听 | chokidar | Node | 抽象 `IFileWatcher`；Tauri 用 notify 插件，或干脆保留 Node sidecar | 低 |
| WSL agent | 独立 Node 进程（HTTP + HMAC） | 无桌面依赖 | 零改动 | 无 |
| hub（多机汇聚） | Node / Cloudflare Worker | 无 | 零改动 | 无 |
| tokscale 内核 | 独立二进制，stdin/stdout JSON 行协议 | **无**（不链任何 native API） | 零改动——最大适配性资产 | 无 |

### 13.2 三条移植路径

| 路径 | 触发条件 | 要做的事 | 保留的部分 |
|---|---|---|---|
| A. Electron → Tauri（减内存/减体积） | 常驻内存压不到 M3 验收门 | 实现 `IShell` 的 Tauri 版 + SQL 绑定替换；core/adapters/UI 原样 | 其余全部 |
| B. 桌面 → 无头服务器/CLI | 想在 NAS/服务器上跑采集 | core+adapters 本就可独立进程化（`tokenledger scan`），输出改 webhook/IM 推送 | 采集与聚合全量 |
| C. 只读 Web/移动端 | 手机看账单 | hub 暴露 REST + 复用同一套 React 面板组件读 `today_view` | UI 组件全量 |

### 13.3 保护可移植性的硬规约（写进 CI，不是口头约定）

1. `adapters/`、`core/` 禁止出现 `electron`/`node:net` 等宿主 API import——dependency-cruiser 规则在 CI 强制；
2. 壳层能力只经 `IShell` 接口暴露给 UI，UI 不直接碰 `window` 级原生 API；
3. 所有 DB 访问走 repository 层（换 SQLite 绑定只改一个目录）；
4. 子进程协议全部 JSON-line（tokscale、zstd 阶段工具），不链任何 native 库；
5. 定价/换算 profile 为纯 JSON schema，导入导出与宿主无关；
6. core/adapters 的单测在纯 Node 环境跑（无 Electron 依赖即通过 = 可移植性被持续证明）；每周 CI 附加一次"独立 CLI 构建"任务验证。

> 为什么现在不直接上 Tauri：fork 的采集面（主进程文件监听、托盘、headless agent、hub）全部长在 Electron/Node 生态上，直接换壳等于放弃 fork 优势从零写。用上述规约把未来迁移成本锁死在壳层一个目录内，是当前最优解。

## 附录 A：若未来迁 Tauri

UI 层（React 18 + Tailwind + zustand）可整体平移；需重写：窗口管理（Electron→Tauri API）、文件 watch、SQLite 绑定（better-sqlite3→tauri-plugin-sql）；适配器层纯 TS 无 Electron 依赖（§8 结构已保证），迁移成本集中在壳层。触发条件：常驻内存无法压到目标且社区有强诉求。

## 附录 B：悬浮窗与主面板数据一致性

单一数据源 = 聚合库 `today_view`；悬浮窗/托盘/面板 CLI `/bill` 全部读同一视图，数字不一致即 bug（一致性测试列入 M3 验收门）。
