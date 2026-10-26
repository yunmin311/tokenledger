# TokenLedger · 上游同步与配置指南

## 一、官方（upstream）更新怎么办

**原则：绝不用应用内"检查更新"安装官方版**——更新器指向 Javis603 的 release，装上会覆盖本 fork 的积分/校准功能（`automaticAppUpdates` 已确认为关）。

正确姿势（命令行同步源码，功能全部保留）：

```bash
cd E:\1project\tokenledger
git fetch upstream                       # 拉官方
git checkout main
git merge upstream/main                  # 日常用 merge；提 PR 用干净分支 cherry-pick
npm ci                                   # 依赖变化时
npm run update:hub-build                 # 改过 src/shared 必须（worker 闭包 + hub 构建标记）
npx eslint .                             # 门禁
node --test "tests/{shared,electron,hub}/**/*.test.js"   # 按目录验收（全量并发在本机会停摆）
npm start                                # 本机冒烟
git push
```

- **冲突面极小**：本 fork 改动 = 独立模块（`src/shared/credits/`、`providers/workbuddy/credits.js`）+ 少量挂钩（usageTransform 可选步骤、usage.js 白名单、agent/main/worker 注入、renderer 设备行徽章）。
- 我们的功能已向官方提 PR（Javis603/token-monitor#1207）；合入后同步即可去掉本地重复。
- 上游若实现同类功能，以冲突为信号取舍，保数据结构兼容优先。

## 二、本机推荐配置

| 设置 | 建议 | 原因 |
|---|---|---|
| 系统玻璃 systemGlass | **关**（已写好） | Windows acrylic 本机渲染不完整 → 四角白色直角；关闭走真透明圆角 |
| 应用内自动更新 | **保持关**（已是） | 防官方包覆盖 fork |
| Trae CN 额度 | 待配 token | traE.cn 登录 → F12 → Cookies → 复制 `cloud-ide-jwt` 粘贴到设置 Trae 处 |
| 托盘内容 | 可切 tokens/cost/bars | 默认显示当期用量最高 agent 的 logo |
| DeepSeek 额度 | 已通过 `.env` 配好 | 余额卡片已就绪 |
| ZCode 历史基数 | `manualUsageBases`（3 条已配） | codex +453,699,906 **按模型 gpt-5.6-sol**（对齐官方 32.8 亿，缺口部分用户申报全在 5.6-sol）；zcode +123,033,649 **按模型 glm-5.3-flash**（两卷 6 亿 − 本地 4.77 亿）；claude **按模型 4 条合计 18,890,164,300，含分量**（opus-4-8 out 84.6M/cacheR 10.7b/cacheW 416.3m 等，来自公司机 /stats Models 页；分量折进 outputTokens/cacheReadTokens/cacheWriteTokens 与 modelOutputs 等六处映射）。均为恒定 allTime 偏移，改数字后重启生效；**基数只进 allTime，不进逐日热力图**（热力图=本地逐日档案，自 2026-04-20 起，内部一致已审计：77 天 perClient 合计与日总量 0 误差） |

## 三、隐私与数据安全边界

- credits 线上内容 = 模型名 + 积分数值 + 请求哈希 + 会话 id（无路径/内容/凭据），hub 归一化全部截断封顶。
- `.env`（DEEPSEEK_API_KEY）已 gitignore，永不入库。
- 项目路径仅存于本机设备记录，不进 credits 块；提 PR 前的 diff 已做隐私扫描（0 命中）。

## 一·b、后续更新节奏（2026-10-27 定）

1. **官方 upstream**：大约每月一轮。流程：`git fetch upstream` → 看 changelog → `git merge upstream/main` → `npm ci`（依赖变化时）→ `npm run update:hub-build` → `npx eslint .` + 按目录测试 → push → **改动攒批、widget 只重启一次**。永远不用应用内更新器。
2. **远端 origin（GitHub 备份）**：每批改动完成即 push main；PR #1207 等人类 review（CodeRabbit 意见当天修）；issue #1208 等 Trae 上游回复。PR 合入后：删 pr 分支，把 `manualUsageBases`（用户申报历史基数，通用功能）整理成第二个 PR。
3. **本机**：widget 跑本地工作树，代码更新=一次重启；**settings.json 修改必须先停 widget**（运行中会回写覆盖，已踩坑两次）；基数数值文档化在本文件，settings 在 userData 不进仓库。
4. **数据补录**：任何官方页面能读到的总数/分量（像 2026-10-27 的 Claude /stats 截图），发来即可补进基数——支持 tokens/output/cacheRead/cacheWrite/cost 五个分量、按模型细分。费用（costUsd）目前无申报数字，Claude 公司机账单金额给我即可补上。
