# Wind 使用约定（agent 执行规则 + 用户参考）

最后更新：2026-09-17

> ⚠️ 本文件含个人持仓信息，**不要提交到公开仓库或分享**。

---

## 0. 核心事实：API Key 不会"自动调用"任何东西

`WIND_API_KEY` 只是一张**凭证**，它自己不会发请求。任何一次 Wind 调用都必须由**某次明确的任务**发起：

- ❌ 没有后台轮询、没有定时任务、没有守护进程；**闲置时零消耗**。
- ✅ 真实存在的"自动"只有一种：**skill 自动路由**——用户不需要记 skill 名，agent 遇到金融问题会自动加载对应 skill。这是 agent 的判断，不是 Key 的行为。
- ✅ 每次取数都是 agent 为了让回答有数据依据而主动发起的，**用户可以随时喊停**。

---

## 1. 三层结构：谁在"思考"

| 层 | 内容 | 谁在思考 | 成本 |
| --- | --- | --- | --- |
| ① `wind-mcp-skill` | **取数接口，不是模型** | 当前 agent 的模型 | 低，按次 |
| ② 16 个工作流 skill | **万得的方法论模板** | 当前 agent 的模型 | 中（=取数那几次） |
| ③ `wind-alice` | **万得自己的 Agent / 大模型** | **万得 Alice** | **高**，单次数分钟~十几分钟 |

当前 agent 运行在 DSH 中，模型为 DeepSeek。**没有单独的"万得大模型" skill——所谓"调万得大模型"就是调 Alice（③）。**

---

## 2. 消耗矩阵

| 动作 | 是否消耗 Wind 额度 |
| --- | --- |
| 安装 / 加载 skill、读 SKILL.md 与契约文档 | ❌ |
| 读本地文件、读截图、做算术 | ❌ |
| `wind-mcp-skill` 数据调用 | ✅ 按次 |
| `wind-alice` Agent 运行 | ✅ 最贵的一档 |
| `update-check.mjs`（每次用 skill 时静默跑一次） | ❌ 只查 GitHub/Gitee，不调 Wind |
| 闲置 | ❌ |

---

## 3. Agent 必须遵守的执行规则

1. **Alice（③）类运行，一律先征求用户同意**再发起；发起前必须告知"耗时数分钟到十几分钟、可能消耗较多积分、不要中途取消或重复发起"。
2. **数据类，尽量合并调用、优先复用已取到的数据**；不得为了省事重复取同一批数据。
3. **用户说"只用已有数据回答 / 别调接口"时，必须 0 调用作答**，并说明哪些结论因此受限。
4. 每次取数后，**在回复末尾交代本轮消耗次数**。
5. 传问句给 Alice 时**必须原样传入，禁止改写润色**。
6. 报告数据时**只基于 Wind 返回值**，不补常识、不臆测；未取到的指标要显式列为"未取数项"。
7. 涉及金融数据的问题，**不得用网页搜索 / 通用知识替代 Wind 取数**。

---

## 4. 点单手册

| 用户意图 | 实际触发 | 成本 |
| --- | --- | --- |
| 要数字（"茅台今天多少"） | ① | 低 |
| 要分析判断（"沪深300 跌了能买吗"） | ②+① | 中 |
| 要万得 Agent 出报告（"用 Alice 出一份公司一页纸"） | ③+指定子技能 | 高 |
| 让万得 Agent 自由发挥（"问一下 Alice…"，不点技能） | ③ auto 路由 | 高 |

**Alice 的 14 个子技能**（说中文名即可）：公司一页纸、上市公司调研问题清单、全球上市公司季报点评、事实核验、按主题选股、投资标的创意与筛选、基金对比分析、基金筛选与投资建议、宏观数据解读、信用分析、债券利率走势研判、通胀情景债券轮动策略、市场规模测算与战略建模、可比公司分析。

---

## 5. 如何直接调用（不经过 agent）

**方式 A：自然语言** —— 直接问，agent 自动路由（最省事）。

**方式 B：自己在终端跑 CLI**（Key 自动从配置读，无需传入）：

```powershell
cd C:\Users\Ruigu\.agents\skills\wind-mcp-skill
'{"windcode":"600519.SH"}' | Set-Content -Encoding utf8 D:\deep1_alice\_p.json
node scripts/cli.mjs call stock_data get_stock_price_indicators "@D:\deep1_alice\_p.json"
Remove-Item D:\deep1_alice\_p.json
```

PowerShell 下必须用参数文件 + `@路径` 传参。`server_type` 取值：`stock_data` / `fund_data` / `index_data` / `bond_data` / `financial_docs` / `economic_data` / `analytics_data`。

**方式 C：原始 HTTP**（给其他程序集成用；Key 作 Bearer token）

- 取数端点：`https://mcp.wind.com.cn/vserver_<域>/mcp/`
- Alice Agent 端点：`https://mcp.wind.com.cn/skills/alice`（SSE 流式）
- 请求头：`Authorization: Bearer <WIND_API_KEY>`
- 注意：这两条是 **MCP / JSON-RPC 协议**，手写 HTTP 需要处理握手与流式解析，实操价值低——用现成 CLI 更省事。

**Key 与额度管理入口**：https://aifinmarket.wind.com.cn/#/user/overview

---

## 6. 配置与停止消费

- **Key 位置**（全局最高优先级）：`C:\Users\Ruigu\.wind-aifinmarket\config`，内容为 dotenv 格式 `WIND_API_KEY=<KEY>`。
- 读取优先级：全局 config > skill 目录 `config.json` > 环境变量 `WIND_API_KEY`。
- **想彻底停掉消费**：删除或改名该 config 文件 → 所有 wind skill 立刻报 `AUTH_ERROR`，**不会静默花钱**。
- 绝不在命令、输出、日志或长期文档中显示真实 Key（只允许 `Bearer <WIND_API_KEY>` 占位写法）。

---

## 7. 已安装的 19 个 Wind skill

| 分组 | skill |
| --- | --- |
| 入口 | `wind-find-finance-skill`（路由）、`wind-mcp-skill`（取数）、`wind-alice`（Agent） |
| 市场主线/情绪 | `a-share-primary-theme-identification`、`market-environment-analysis`、`market-breadth-health-skill`、`market-sentiment-temperature-skill`、`sector-rotation-radar-skill`、`market-regime-switch-skill` |
| 复盘 | `post-market-debrief` |
| 个股研究 | `stock-first-look-skill`、`equity-investment-thesis` |
| 估值 | `valuation-snapshot-skill`、`dcf-model` |
| 选股 | `canslim-growth-scan-skill`、`high-quality-compounder-finder-skill` |
| 财报/交易 | `earnings-preview-skill`、`trade-plan-builder-skill`、`position-sizing-decision-skill` |

### 安装位置（3 处，19 个 skill × 3 = 57 个目录）

| 位置 | 用途 | 目录命名 | 谁读 |
| --- | --- | --- | --- |
| `C:\Users\Ruigu\.agents\skills\` | 规范库（canonical） | **原样**（保留下划线，兼容路由器按路径检测） | DSH、Cline / Dexto / Warp / Zed 等 |
| `C:\Users\Ruigu\.cursor\skills\` | Cursor 全局 | **kebab-case**（`_` → `-`） | Cursor |
| `C:\Users\Ruigu\.codex\skills\` | Codex 全局 | **kebab-case**（`_` → `-`） | Codex |

Cursor / Codex 两处是**手工复制**的，原因（均实测确认）：
1. `skills` CLI v1.6.0 在全局安装时**只写 `~/.agents/skills`，并在自己的注册表里把 agent 标记为已装，不真正写 per-agent 目录**（`skills ls -g -a cursor` 显示 `Agents: Cursor`，但 `~/.cursor/skills` 里没有文件）；
2. Windows 下**软链接需要管理员权限**，实测报 `Administrator privilege required`，CLI 静默跳过（加 `--copy` 也一样）；
3. **Junction 不可用**：Node 把 junction 报成 `isDirectory=false, isSymbolicLink=true`，Cursor / Codex 这类"只认目录"的扫描器会跳过它。

→ 只能用**真实目录复制**。目录名按 CLI 自身约定（源码 `join(agent.globalSkillsDir, sanitizedName)`）取 kebab-case。

### ⚠️ 命名坑（重要，涉及 33 个文件）

DSH / Cursor / Codex 都要求 skill 名为 kebab-case（`/^[a-z0-9]+(?:-[a-z0-9]+)*$/`），**下划线不合法且会被静默跳过**（文件在、但不进目录、也不报错）。

万得这批 skill 的 frontmatter `name` 带 `_skill` 后缀，已在**三个位置共 33 个 `SKILL.md`** 的第 2 行手工改为连字符。受影响的 11 个：

`market_breadth_health_skill`、`market_sentiment_temperature_skill`、`sector_rotation_radar_skill`、`stock_first_look_skill`、`valuation_snapshot_skill`、`canslim_growth_scan_skill`、`high_quality_compounder_finder_skill`、`earnings_preview_skill`、`trade_plan_builder_skill`、`position_sizing_decision_skill`、`market_regime_switch_skill`

**已实测的坑**：`npx skills add` / `npx skills update -g -y` 会**从仓库重新拉取并覆盖 `~/.agents/skills`**，把上述修复全部冲掉——本次会话就因此踩了一次（那 11 个 skill 一度从会话目录消失，副本也变回下划线）。

**修复方式**：把三个位置里这 11 个 `SKILL.md` 第 2 行 `name:` 的下划线换成连字符（共 33 个文件），并确保 `~/.cursor`、`~/.codex` 两处的**目录名**同为 kebab-case。让 agent 执行即可。

---

## 8. 用户投资者画像（2026-09-17 截图，可能已过期）

- 总资产约 **13.1 万元**，**保守型**。
- **国债逆回购 80.8%**（10.6 万，锁到 10/8 可用、10/9 可取）；可用现金仅约 197 元。
- **权益类 19.2%**：银行股（工行/中行）11.6%、黄金ETF 3.4%、沪深300ETF 2.8%、芯片/半导体及电网设备ETF 1.4%。
- **穿透后金融占权益暴露 77.2%** —— 集中度是主要结构问题（而非估值过高：工行 PB 0.73 / 股息率 3.95%，中行 PB 0.78 / 股息率 3.60%）。
- 逆回购遇长假顺延到期，**实际占款天数 > 计息天数**，真实年化低于名义利率（14天期名义 1.450% → 折算约 1.28%）。
- 分析偏好：要**数据支撑 + 明确风险提示**；不要投资建议式口吻；关注**费用与消耗**。
