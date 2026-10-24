# 统一 Token 计费中枢 · 立项报告书

> 版本：v2.1（2026-10-24）｜ 状态：待立项评审
> v2 修订：补扫 WSL 虚拟机、WorkBuddy、积分工具可行性实测。
> v2.1 修订：Windsurf 确认已卸载并移出范围；新增"灵动岛/悬浮窗轻量呈现层"竞品调研（10+ 项目）。
> **配套设计书**：《统一Token计费中枢-设计书.md》（v1.0 评审稿）——cc-switch 蓝本逐条借鉴、悬浮窗（非灵动岛）完整规格、DDL 级数据模型、开工 MCP/Skill/插件全清单。

---

## 0. 摘要

- **全量清点完成**：Windows + WSL(Ubuntu-24.04) 双环境共检出 **18 个在用/在装的 AI agent 数据源**，其中 12 个可精确取 token，4 个积分制，2 个仅账号信息。
- **WorkBuddy 惊喜发现**：本地 `workbuddy.db` 同时存有每会话 token 用量（`used`）、每请求积分消耗（`credit_json`）和模型名（`model`）——**积分↔token 换算不需要瞎猜，可以做回归校准**。
- **Trae 结论修正**：Trae / Trae CN 本地 16MB/6MB 数据库是**加密库**（非明文 SQLite），token 明细拿不到，只能走官方用量 API（cookie/token 认证），维持"额度派"定位。
- **技术路线定案**：**Fork [Javis603/token-monitor](https://github.com/Javis603/token-monitor)**（Electron + TS，46+ 工具，已支持 ZCode、WorkBuddy、WSL headless agent），底层继续用其 vendored 的 **tokscale** Rust 内核做解析，自己新增积分估算层 + 校准层 + 中文报表。不从零写。
- **依赖全自动**：运行时依赖全部走 `npx`/npm postinstall 自动拉取，开发期 skills 随仓库 `.zcode/skills/` 分发。
- **v2.1**：Windsurf 已卸载（仅剩残留目录），移出采集范围；轻量"灵动岛/悬浮窗"呈现层赛道已相当拥挤（AgentBro 等 10+ 项目），但它们全部只做 token/额度显示——积分明细、WSL 汇聚、换算校准仍是空白，差异化结论不变。

---

## 1. 扫描方法与覆盖范围

| 环境 | 扫描方式 | 覆盖 |
|---|---|---|
| Windows | home 目录枚举、`AppData/Roaming+Local` 枚举、`npm ls -g`、PATH CLI 探测、SQLite schema 实测 | 全部用户级数据 |
| WSL | `wsl.exe -e bash -lc` 进入 Ubuntu-24.04，同法枚举 + `du -sh` 计量 | 默认发行版（docker-desktop 已停、无 agent） |

> 教训记录：Git Bash 的 `/tmp` 路径传给 Windows Python 会让 sqlite3 静默新建空库，导致上一版误判 Trae 库为空。本版全部用 `cygpath -w` 转换后重测。

---

## 2. 全量数据源清单（全部实测）

### 2.1 明细派 · Windows（token 精确可得）

| # | 工具 | 数据位置（实测） | 格式 / 关键字段 | 采集难度 |
|---|---|---|---|---|
| 1 | ZCode | `~/.zcode/cli/rollout/model-io-session_*.jsonl`（现有 3 个会话） | JSONL，`usage.input_tokens / output_tokens / cache_read_input_tokens` 双格式（snake_case + camelCase 并存） | ★ 低 |
| 2 | Claude Code | `~/.claude/projects/**/*.jsonl` | JSONL，逐消息 `usage{input, cache_creation, cache_read, output, server_tool_use}` | ★ 低（ccusage 现成） |
| 3 | Codex | `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl` | JSONL，`total_token_usage{input, cached_input, cache_write, output, reasoning}` + `model_context_window` | ★ 低（ccusage 现成，beta） |
| 4 | OpenCode | `~/.local/share/opencode/opencode.db`（**5.7GB**）+ `auth.json` | SQLite + WAL；额度走官方 `/zen/go/v1/usage`（[opencode-go-monitor](https://github.com/AndersOnLin4/opencode-go-monitor) 已验证，支持多 key 自动发现） | ★★ 中（体量大，必须增量） |
| 5 | Gemini CLI | `~/.gemini/`（history/、projects.json、oauth_creds.json） | JSONL 历史 | ★★ 中 |
| 6 | DeepSeek Harness (dsh) | `~/.dsh/`：`session-query.sqlite`（FTS 索引，110 docs）+ `sessions/<项目名>/session.v3.jsonl.**zstd**` | zstd 压缩 JSONL（纯 JS `fzstd` 可解） | ★★ 中（zstd + 私有 schema） |
| 7 | GitHub Copilot Chat | VS Code `globalStorage/github.copilot-chat/` | 本地仅缓存，额度走 GitHub API | ★★★ 高（厂商口径） |
| 8 | OpenClaw | `~/.openclaw/completions/` | 仅补全记录 | ★★ 中 |

### 2.2 明细派 · WSL（Ubuntu-24.04）——本版新增

| 工具 | WSL 内位置（实测） | 体量 | 说明 |
|---|---|---|---|
| Claude Code | `/home/lqy/.claude/projects/` | 22MB / 12 个会话 | 与 Windows 格式一致，同一 parser 复用 |
| Codex | `/home/lqy/.codex/sessions/` | **2.8GB** / 6 个会话（单会话很大） | 同上 |
| OpenCode | `/home/lqy/.local/share/opencode/opencode.db`（774KB + 2.3MB WAL） | 3.1MB（+ `~/.config/opencode` 63MB） | 同上 |
| DeepSeek Harness | `/home/lqy/.dsh/` | 2.1MB | 同上 |
| CLI 安装确认 | `claude`、`codex`、`opencode` 均在 WSL PATH（fnm/local 安装） | — | 说明 WSL 是真实使用环境，不是摆设 |

**采集通道二选一**（架构上两者都支持）：
- A. `\\wsl.localhost\Ubuntu-24.04\home\lqy\...` 路径直读（简单，但 5GB 级文件跨 9P 协议拷贝慢）；
- B. **WSL 内跑 headless agent**（token-monitor 自带 Node headless agent，官方支持 SSH/WSL 场景，推荐）。

### 2.3 积分/额度派（credits 制或厂商黑盒）

| 工具 | 本地数据（实测） | 积分→token 可行性 | 定级 |
|---|---|---|---|
| **WorkBuddy（腾讯）** | `~/.workbuddy/workbuddy.db`：`session_usage(session_id, used, size, updated_at, credit_json)` 45 行；`sessions(id, cwd, title, model, mode, status, created_at)` 48 行；另有 automations、buddy_snapshots 等 11 表（Drizzle 迁移，Node 应用） | ✅ **最优**。`used`=已用上下文 token（如 398,901/1,000,000），`credit_json`=每请求积分（哈希键→数值），`model`=实际模型（实测含 deepseek-v4.1-flash、hy4-preview、space-bunny、balanced-model）。同一会话既有 token 又有积分 → **直接回归出 per-model 转换系数** | A 级重点 |
| Trae（国际版） | `Roaming/Trae/ModularData/ai-agent/database.db` 6MB | ❌ 加密库（"file is not a database"），token 明细不可得 | 走官方会话用量 API（cookie/token，[trae-usage-monitor](https://github.com/alanqin888/trae-usage-monitor) 有现成认证同步方案可抄） |
| Trae CN | 同上，16MB | ❌ 同上 | 同上 |
| Cursor | `Roaming/Cursor/state.vscdb`（账号态）；`~/.cursor/ai_tracking/`（当前为空） | ❌ 本地无明细 | 官方 usage-events API（tokscale 已实现，厂商口径） |
| ~~Windsurf~~ | 主程序已卸载（注册表无卸载项、无 CLI）。残留：`~/.codeium/`（cascade/code_tracker/database）、`~/.windsurf/`、`Roaming/Windsurf/`、开始菜单快捷方式 | — | **移出采集范围**（用户确认弃用）；残留目录可择机清理，不影响其他工具 |

### 2.4 未安装、但生态已支持（装了就能收，无需开发）

Warp、Amp、Goose、Crush、Kiro、Zed、Google Antigravity、Qwen CLI、Kimi、MiniMax、Cherry Studio、LM Studio、Roo/Cline/Kilo（VS Code 扩展）——tokscale / token-monitor 均已内置 adapter。

### 2.5 数据源汇总（按采集方式归类）

```
本地文件解析（token 精确, confidence=exact）
  Win:  ZCode.jsonl | Claude.jsonl | Codex.jsonl | OpenCode.db | Gemini | dsh.zstd | OpenClaw
  WSL:  Claude.jsonl | Codex.jsonl | OpenCode.db | dsh
本地文件解析（积分+token 双轨, confidence=exact/vendor）
  Win:  WorkBuddy.db（credit_json + used + model）
厂商 API（额度/厂商口径, confidence=vendor）
  Trae / Trae CN（cookie）| Cursor（usage-events）| Windsurf | Copilot（GitHub）| OpenCode Go（/zen/go/v1/usage）
```

---

## 3. 参考项目全景（fork 候选与可抄设计）

| 项目 | 形态 | 对本项目的价值 | 许可 |
|---|---|---|---|
| [token-monitor](https://github.com/Javis603/token-monitor) | Electron 桌面面板 | **主 fork 候选**：46+ 工具、已支持 **ZCode/GLM、WorkBuddy、Trae CN、DeepSeek**、WSL/SSH headless agent、多端同步（自建 hub / Cloudflare Worker / iCloud）、托盘+悬浮球+热力图 | MIT |
| [tokscale](https://github.com/junhoyeo/tokscale) | Rust 内核 + TS CLI + Web | **解析内核**（token-monitor 已 vendor）：Rust 并行扫描 + SIMD JSON，40+ agent，Trae/Cursor 厂商 API、各家配额端点、LiteLLM 定价 | MIT |
| [ccusage](https://github.com/ccusage/ccusage) | 轻量 CLI | Claude/Codex/OpenCode 三家最成熟 parser，交叉验证基准 | MIT |
| [opencode-go-monitor](https://github.com/AndersOnLin4/opencode-go-monitor) | Electron 单用途 | **OpenCode 适配教科书**：`/zen/go/v1/usage` 额度端点、多 API key 自动发现、`opencode.db` 本地历史读取、30s±jitter 轮询节流 | MIT |
| [68HUB](https://github.com/evanfu0110/68hub) / [GoGauge](https://github.com/yphyphyph/opencode-go-gauge) | Electron / 面板 | OpenCode Go 多账户、滚动窗口（5h/周/月）配额展示设计可抄 | — |
| [cc-bar](https://github.com/nanvon/cc-bar) | macOS 菜单栏 | 多家配额+重置倒计时的轻量呈现形态参考 | — |
| [trae-usage-monitor](https://github.com/alanqin888/trae-usage-monitor) | VS Code 扩展 | **Trae 认证方案**：浏览器"魔法书签"一键同步 token，避免手动拷 cookie | — |
| [cc-switch](https://github.com/farion1231/cc-switch) | 供应商切换器 | OpenCode Go 套餐/供应商配置管理思路 | — |
| 腾讯云社区《WorkBuddy 离线看板》文章 | Skill（SkillHub） | 佐证 WorkBuddy 本地数据可挖（每日积分、模型 token 占比）；与本报告实测互印证 | — |

**结论**：不存在"直接拿来就用"的完整方案——没有任何一个项目同时做到「WSL + WorkBuddy 积分明细 + credits↔token 校准 + 统一置信度」。但 **token-monitor 覆盖了 85% 的采集面**，缺的恰是我们的差异化层。

### 3.1 轻量化呈现层生态（灵动岛 / 悬浮球 / 托盘 —— 即短视频平台上推广的那类）

| 项目 | 形态 | 平台 | 数据面 | 备注 |
|---|---|---|---|---|
| [AgentBro](https://github.com/shirenchuang/agentbro) | **灵动岛浮窗 + 桌面宠物**（活力随 token/上下文压力变化） | macOS 完整版 / Windows x64 MVP | Claude Code、Codex、Gemini、Cursor、Copilot、Cline、Qwen、Kimi、DeepSeek、OpenCode 等，hook 本地采集（本地 socket，无云端） | Tauri（Rust+React/TS）；Apache-2.0 但 **fork 必须改名**；只显示 token，无费用/积分 |
| [shxtmaker/Token-Consumption-Monitoring](https://github.com/shxtmaker/Token-Consumption-Monitoring) | Win11 桌面小组件 + 托盘 + 配置面板 | Windows | 多服务账户的额度窗口、token 用量、费用、余额 | 2026-10 仍在活跃更新 |
| Agenton（[自荐帖](https://github.com/ruanyf/weekly/issues/9602)） | 桌面端用量/费用监控 | Windows | Session/Weekly 用量百分比 + 重置倒计时、当日及 30 天费用（USD）、token 级 | 仓库地址见帖子 |
| "AI Token 用量監測"（微软商店） | **悬浮球**，点击展开 | Windows | GLM、Kimi 等编码方案额度 | 商店分发，疑似推广量最大的那批 |
| [ClaudeWatch](https://blog.csdn.net/weixin_35612842/article/details/160151481) | 桌面悬浮窗 | Windows | Claude Code 会话状态与花销 | 单工具向 |
| [CodeBurn](https://github.com/getagentseal/codeburn) | CLI 面板 + macOS 菜单栏 | 跨平台 CLI | 37 种工具，按模型/项目统计，本地直读 session，免 API key | 无 Windows 桌面版 |
| [soulduse/ai-token-monitor](https://github.com/soulduse/ai-token-monitor) | 系统托盘轻应用 | macOS/Windows | 额度/用量 | 轻量向 |
| [cc-bar](https://github.com/nanvon/cc-bar) | 菜单栏 | macOS | Codex/Claude/Cursor/Antigravity 配额+重置 | 前文已列 |
| [Poietic-TokenForest](https://github.com/Ericcccccc777/Poietic-TokenForest) | token 长成树的可视化 | — | Claude Code/Codex | 趣味向，设计可借鉴 |
| Claude HUD | 终端状态栏插件 | 全终端 | Claude Code 用量 | 状态栏形态 |

**判读**：纯"悬浮窗看花销"已是红海，无需自证先发；但这一层**全部停留在 token/额度显示**，没有一个做积分明细入库、跨 Windows+WSL 汇聚、credits↔token 校准。呈现层抄 AgentBro 的交互，数据层做别人没有的——正是本项目的定位。

---

## 4. 技术方案三轮讨论记录

### Round 1 —— 路线之争：从零写 / Fork / 抄设计自建

| 方案 | 论点 | 裁决 |
|---|---|---|
| ① 从零写 | 完全掌控；但要手写 20+ 本地格式 parser、追 10+ 厂商接口变动、独自做三端 UI。以一人维护量，三个月内必烂尾 | ❌ 否决 |
| ② Fork tokscale | 内核最强；但核心是 Rust（改动有门槛），且带社交平台/排行榜包袱，Web 前端与桌面面板割裂 | ❌ 否决（作为内核保留使用） |
| ③ **Fork token-monitor** | TS/Electron 全家桶（匹配你 Node.js 栈）；46+ adapter 白拿；**已内置 vendored tokscale**（不碰 Rust 也能升级解析）；自带 WSL headless agent 和多端同步——正是"WSL 也统计进来"的现成答案；缺的 WorkBuddy 积分明细、Trae CN、校准层由我们补 | ✅ **采纳** |
| ④ 抄设计自建外壳 | 轻量；但等于把 ① 的 parser 工作量买回来，不划算 | ❌ 否决 |

**Round 1 结论**：Fork token-monitor 为主体，tokscale 作解析内核保留，逐个抄 opencode-go-monitor（OpenCode 额度）、trae-usage-monitor（Trae 认证）、GoGauge（滚动窗口展示）的具体设计。

**v2.1 补充讨论（呈现形态）**：用户明确偏好"灵动岛式"轻量呈现。最成熟的 [AgentBro](https://github.com/shirenchuang/agentbro) 是 Tauri 项目（Rust 后端，与我们的 TS 栈不匹配），且 Apache-2.0 附加"fork 必须改名"条款、Windows 端仅 MVP、无费用维度——**不作为 fork 对象，作为交互设计抄袭对象**。token-monitor 自带悬浮球/托盘/Edge Dock 三种轻量形态，改造其皮肤即可获得灵动岛体验，采集与呈现解耦不受影响。

### Round 2 —— 技术栈选型

| 维度 | 决策 | 理由 |
|---|---|---|
| 语言/运行时 | TypeScript + Node 22 | 与 fork 基座一致；你现有技术栈零学习成本 |
| 桌面壳 | Electron（fork 保留） | 基座已有托盘/悬浮球/widget 全套 |
| 解析内核 | vendored tokscale 二进制（运行时自动下载更新） | Rust 性能白拿，不参与其编译链 |
| 存储 | SQLite（每机一份）+ 水位（high-watermark）增量同步 | 你的 Windows opencode.db 已 5.7GB、WSL Codex 2.8GB，**严禁全量重扫** |
| 多机汇聚 | fork 自带 hub（自建 Node 或 Cloudflare Worker），主从同步 | Windows + WSL + 未来其他机器统一入湖 |
| Web 报表 | fork 内置 React 面板改造 + 中文优先 | 不另起服务 |
| 网关兜底（v3） | LiteLLM（`uvx litellm` 拉起） | 能配 baseURL 的工具全部精确计量，兼做校准探针 |

### Round 3 —— 采集与计费架构（差异化核心）

1. **Adapter Registry**：每工具一个 adapter（file-parser / vendor-api / hybrid 三类），独立失效、独立配置、健康度可见。
2. **统一记录模型**：
   `UsageRecord{ts, machine, tool, project, model, in/out/cache_read/cache_write/reasoning, cost_usd?, credits?, confidence}`
3. **三级置信度**：
   - `exact`：本地解析（ZCode/Claude/Codex/OpenCode/dsh/WSL 侧）
   - `vendor`：厂商 API 口径（Trae 额度、Cursor、Copilot、OpenCode Go 配额）
   - `estimated`：按 conversion profile 估算的 token 等值（仅展示列，不污染真实积分流水）
4. **积分换算与校准**（回答上一版"没头绪"的问题）：
   - WorkBuddy：本地 `used`(token) × `credit_json`(积分) 同会话并存 → **按 model 分组线性回归**，得出该模型实际 credits/token 斜率，替代官方换算表；
   - Trae 等纯 API 工具：credit 事件流 + 官方定价页建 profile，标 `estimated`；
   - 终极校准：LiteLLM 网关侧计 token、厂商侧扣积分，双轨对齐收敛系数。

---

## 5. 定案技术栈清单

| 层 | 选型 |
|---|---|
| 桌面应用 | Electron + TypeScript（fork token-monitor） |
| 解析内核 | tokscale（vendored，自动更新） |
| 存储 | SQLite（WAL，水位增量） |
| 同步 | 内置 hub（Node / Cloudflare Worker 可选）+ WSL headless agent |
| 前端 | React 面板（fork 内置）中文化改造 |
| CLI 交叉验证 | ccusage（npx） |
| 网关 | LiteLLM（v3 阶段引入） |
| 打包分发 | electron-builder（fork 现成配置） |

---

## 6. 依赖清单：插件 / Skill / 运行时（全部自动下载）

### 6.1 运行时依赖（用户机器零预装，脚本自动拉取）

| 依赖 | 用途 | 拉取方式 |
|---|---|---|
| tokscale 二进制 | 解析内核 | fork 内置下载器（首发）+ 升级检查 |
| ccusage | 独立交叉验证 | `npx ccusage@latest` |
| better-sqlite3 | 聚合库读写 | npm 依赖（prebuilt 二进制） |
| fzstd（纯 JS） | 解压 dsh 的 `session.v3.jsonl.zstd` | npm 依赖 |
| LiteLLM | 校准网关（v3） | `uvx litellm`（需 Python） |

### 6.2 开发期 ZCode 插件 / Skill（随仓库分发）

| 项 | 用途 |
|---|---|
| `frontend-design` skill | 面板视觉定制（中文报表页） |
| `visualize` skill | 会话内数据图表原型验证 |
| `browser-use:web-gui-tester` | 自动化测 Trae/Cursor 登录同步流（GUI 黑盒） |
| `dynamic-workflows` | 并行 fan-out 调研 / 多 adapter 回归测试 |
| `spreadsheets:xlsx` / `pdf:pdf` | 月度账单导出 |
| `humanizer-zh` | 报告与文案润色 |
| MCP：`csv-mcp` | 导出 CSV 后的 SQL 复核 |
| MCP：`token-optimizer` | 大扫描日志的省 token 读取 |

Skills 放项目 `.zcode/skills/`（ZCode 自动发现），MCP 配置写入项目 `.mcp.json`，克隆即得。

### 6.3 工程规范（依 AGENTS.md）

- 仓库落 `E:\1project\token-billing-hub\`，注册 GitHub remote 并完成首推；
- 可复用知识镜像到 `E:\1obsidian\Obsidian Vault`，链回 canonical 仓库。

---

## 7. 落地路线图

| 阶段 | 周期 | 内容 | 验收标准 |
|---|---|---|---|
| M0 验证 | 半天 | fork token-monitor 本机编译运行，确认 ZCode/Claude/Codex/OpenCode/WorkBuddy 五个 adapter 在你机器上出数 | 面板可见今日 Windows 侧用量 |
| M1 MVP | 2–3 天 | ① WSL headless agent 部署（收 WSL 四源）② dsh zstd parser ③ 水位增量（防 5.7GB 全量扫） | 双环境数据合并，日/月报表正确 |
| M2 积分层 | 1–2 周 | ① WorkBuddy `credit_json` 解析 + per-model 回归系数 ② Trae/CN、Cursor、Windsurf 额度 adapter（抄 trae-usage-monitor 认证）③ confidence 全链路标注 | 积分与 token 两列并存且可过滤 |
| M3 体验 | 2–3 周 | 中文面板改造、**灵动岛/悬浮球形态（交互参考 AgentBro）**、月度账单导出（xlsx/pdf）、多机同步（NAS/CF Worker） | 你日常只开这一个面板 |
| M4 校准 | 长期 | LiteLLM 网关 + 双轨校准收敛系数；upstream PR 回馈（ZCode adapter 增强、WorkBuddy 积分明细） | estimated 列误差收敛 |

---

## 8. 风险与合规

| 风险 | 缓解 |
|---|---|
| 厂商 API/加密库随版本变动 | adapter 隔离 + 健康度监控 + 格式探测降级（vendor→estimated） |
| Trae/Cursor cookie 认证属灰色 | 只读接口、不自动化登录、token 本地存储；官方开放导出后立即切换 |
| 5.7GB opencode.db 拖垮扫描 | 水位增量 + WAL 感知；首扫放夜间 |
| fork 上游漂移 | 我们的新增全部收敛在 `credits-layer/` 模块，rebase 成本最小化；通用增强回 PR 上游 |
| WorkBuddy `used/size` 字段语义待官方确认 | 按"上下文窗口用量"理解并留字段映射开关，防语义反转 |

---

## 9. 附录：实测证据

- **ZCode usage 样本**：`"usage":{"input_tokens":5494,"output_tokens":1193,"cache_read_input_tokens":35392,...}`（`~/.zcode/cli/rollout/`）
- **Codex 样本**：`"total_token_usage":{"input_tokens":2169408,"cached_input_tokens":1986688,"output_tokens":14634,"reasoning_output_tokens":7151,...}`
- **Claude 样本**：`"usage":{"input_tokens":135,"cache_creation_input_tokens":0,"cache_read_input_tokens":57344,"output_tokens":75,...}`
- **WorkBuddy 样本**：`session_usage: (session_id, used=398901, size=1000000, credit_json='{"<hash>":25.78,...}')`；`sessions.model ∈ {deepseek-v4.1-flash, hy4-preview, space-bunny, balanced-model}`（含 Hunyuan 系）
- **WSL 体量**：`.codex 2.8G`、`.claude 22M(12 会话)`、`opencode.db 774K`、`.dsh 2.1M`、`.config/opencode 63M`
- **dsh 会话**：`~/.dsh/sessions/--E-1project--/session-*/session.v3.jsonl.zstd`（zstd 压缩）
- **Trae 库**：`database.db`（6MB/16MB）非明文 SQLite（`file is not a database`）
- 扫描命令全集见本报告生成会话（wsl.exe 枚举、find/ls 全盘探测、SQLite PRAGMA table_info 抽样）。

---

*报告完 · 下一步：批准后执行 M0（fork + 本机跑通五源验证）*
