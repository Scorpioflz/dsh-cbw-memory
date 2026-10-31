# @dsh-cbw/memory

<p align="center">
  <a href="./README.md">English</a> | <a href="./README_ZH.md">中文</a>
</p>

DSH 长期记忆插件：带半衰期遗忘、情绪权重、按工作区共享。

> **衍生作品**，派生自 [Yuuyuko-uu/dsh-fish-memory](https://github.com/Yuuyuko-uu/dsh-fish-memory) —— 原作者 **Yuuyuko-uu**，按 **Yuuyuko-uu 插件许可 1.2** 授权。相对上游的改动详见[§ 致谢](#致谢)。

## 功能

- **半衰期遗忘** — `A = A₀ × 2^(-days / H)`，H 受重要度调节（`H = base × (1 + 2×I)`）。重要记忆忘得慢。
- **情绪衰减** — 独立情绪半衰期（默认 7 天）；情绪记忆比事实记忆衰减快。
- **稀有度饱和** — `C = s / (s + K)` 控制命中词稀有度的饱和点。
- **问句信息量门槛** — 低信息量查询（如"你好"）直接跳过记忆召回。
- **长期事实常驻层** — `factsMax` 条事实每轮固定注入，不看当前查询，不占 `maxItems` 位置。
- **每日配额** — 限制模型每天最多记录几条记忆（默认 8）。
- **按工作区共享** — 记忆在 DSH 工作区内共享，不是按会话隔离。
- **会话独立** — 每个 agent 挂自己的记忆文件，按会话 ID 索引。
- **时区自动适配** — 自动认机器时区。
- **权重自动凑 100%** — 四个打分权重始终合计 100%。

### v0.2.0 升级（P0 + P1）

- **显式/隐式记忆请求检测** — 用户说"不要记住"或"记住"时，插件相应调整行为。[借鉴自 dsh-mnemon `lifecycle.ts`]
- **记忆去重** — 新记忆与最近条目做 bigram 相似度比较，相似条目合并而非重复记录。[借鉴自 @tr1v3r/dsh-ltm 去重思路]
- **双重预算 + truncated 标记** — `maxItems` + `maxChars` 双重限制，超限时显示 truncated 标记而非静默截断。[借鉴自 dsh-mnemon `composition.ts:normalizeEvidence`]
- **`recall` 工具** — 模型可主动召回更多记忆，而非只接收被动注入。[借鉴自 dsh-mnemon routed View 模式]
- **容量管理 + 自动归档** — 记忆条数超过 `maxMemories`（默认 200）时，低分条目归档到单独文件。[借鉴自 dsh-mnemon `runtime-capacity.ts`]
- **记忆来源标记** — `remember` 工具接受 `source` 参数（`user_explicit` / `model_observed`）；区分用户明确要求记住的和模型自己觉得该记的。[借鉴自 dsh-mnemon ReadGrant 思路，简化版]

### v0.3.0 升级（P2）

- **JSONL 存储 + category** — 记忆以 JSONL 存储，带 `category` 字段（`user_preference`/`fact`/`project_context`/`task_state`/`conversation`/`general`）。半衰期按分类差异化：`project_context` ×3、`fact`/`user_preference`/`task_state` ×2、其余 ×1。
- **增量定时沉淀** — 定时器+增量提取（默认 30 分钟，跳过闲聊，优先工程对话）。默认关。

### v0.4.0 升级（P3）

- **安全模型增强** — `source` 参数验证（`user_explicit`/`model_observed`/`idle_review`）；防伪造 ID。
- **归档索引** — `archive_index` 记录指向归档文件，recall 命中时按需加载（Redis/SQL 模型）。
- **LRU 阈值上调** — `maxMemories` 200→500。
- **归档去重** — 归档前 bigram 相似度检查，防循环归档。
- **CJK bigram 检索** — CJK 感知分词检索（已有，正式文档化）。

### v0.4.1

- **路由前缀统一** — 所有 HTTP 路由 `/dsh-pet-memory/` → `/dsh-cbw-memory/`，与包名一致。
- **插件市场索引** — 通过 GitHub Pages 发布到 DSH 插件市场（`docs/index.json`）。

## 打分公式

```
P = C × (基础权重 + 鲜活度权重 × A + 重要度权重 × I + 情绪权重 × E)
```

其中：
- **C（证据量）**：`C = s / (s + K)`，s = 命中词稀有度之和，K = `rarityK`（0.8）
- **A（鲜活度）**：`A = A₀ × 2^(-days / H)`，H = `baseHalfLifeDays`（30）× (1 + 2×I) × `categoryMult`（project_context: 3, fact/user_preference/task_state: 2, 其余: 1）
- **E（情绪）**：`E = emotion × 2^(-days / emotionHalfLifeDays)`，`emotionHalfLifeDays`（7）
- **I（重要度）**：用户标注的重要度

默认权重：`基础权重=0.55, 鲜活度权重=0.25, 重要度权重=0.15, 情绪权重=0.05`

门槛：`threshold=0.3`（卡 C，不掺鲜活度），`maxItems=6, maxChars=800, dailyQuota=8`

## 安装

```bash
# 在 DSH 设置 → 插件中添加：
dsh plugin --profile <profile> add github:Scorpioflz/dsh-cbw-memory

# 或从本地路径：
dsh plugin --profile <profile> add /path/to/dsh-cbw-memory
```

> **注意**：`@dsh-cbw/memory` 与原版 `dsh-fish-memory` 都注册 `remember` 工具，**不可装在同一个 DSH profile**。安装前如已有 fish-memory，先卸载：`dsh plugin --profile <profile> remove dsh-fish-memory`。

## 配置

所有设置可通过 DSH 设置页（记忆面板）或编辑 `~/.dsh/dsh-cbw-memory-config.json` 配置。

| 参数 | 默认值 | 说明 |
|---|---|---|
| `baseHalfLifeDays` | 30 | 鲜活度半衰期基数 |
| `emotionHalfLifeDays` | 7 | 情绪半衰期 |
| `baselineWeight` | 0.55 | 基础权重（只要被想起就有） |
| `activationWeight` | 0.25 | 鲜活度权重 |
| `importanceWeight` | 0.15 | 重要度权重 |
| `emotionWeight` | 0.05 | 情绪权重 |
| `threshold` | 0.3 | 证据门槛（卡 C） |
| `maxItems` | 6 | 每轮最多注入几条 |
| `maxChars` | 800 | 每轮注入字数上限 |
| `dailyQuota` | 8 | 每天最多记几条 |
| `rarityK` | 0.8 | 稀有度饱和点 |
| `queryGate` | 0.5 | 问句信息量门槛 |
| `factsMax` | 2 | 长期事实每轮带几条 |
| `unknownActivation` | 0.5 | 没写时间的记忆鲜活度 |
| `maxMemories` | 500 | 自动归档前最大记忆条数（v0.4.0+：从 200 上调） |

## 工具

| 工具 | 参数 | 说明 |
|---|---|---|
| `remember` | `text`, `importance`, `tags`, `取代`, `事实键`, `source` | 记录一条记忆 |
| `recall` | `query`, `limit` | 主动召回匹配查询的记忆（v0.2.0+） |

## 致谢

本插件是 [Yuuyuko-uu/dsh-fish-memory](https://github.com/Yuuyuko-uu/dsh-fish-memory) 的衍生作品，按其 **Yuuyuko-uu 插件许可 1.2**（见 [LICENSE](./LICENSE)）声明。

原作者：Yuuyuko-uu。衍生作品继续使用同一许可。

### 相对上游的改动

- 记忆按工作区共享（上游按会话）
- `<system-reminder>` 机器来源检索过滤
- queryGate 小库保护（0.8→0.5，<10 条跳过）
- 设置页会话→工作区归类
- 打包目录编码修复
- 包名改为 `@dsh-cbw/memory`（注册名仍为 `memory`，与上游 remember 工具兼容）
- v0.2.0：显式/隐式请求检测、记忆去重、双重预算+truncated 标记、recall 工具、容量管理+自动归档、记忆来源标记
- v0.3.0：JSONL 存储+category 字段、增量定时沉淀
- v0.4.0：安全模型（source 验证）、归档索引、LRU 阈值上调（200→500）、归档去重、CJK bigram 检索
- v0.4.1：路由前缀统一（`/dsh-pet-memory/` → `/dsh-cbw-memory/`）、插件市场索引
- v0.5.0：分类半衰期差异化（`project_context` ×3、`fact`/`user_preference`/`task_state` ×2、其余 ×1）

### 借鉴思路来源

- [dsh-mnemon](https://github.com/omdsh-dev/dsh-mnemon) — Composable View 模式、双重预算控制、显式/隐式请求检测、idle review、ReadGrant 安全模型、容量自动归档
- [@tr1v3r/dsh-ltm](https://www.npmjs.com/package/@tr1v3r/dsh-ltm) — CJK 分词、去重思路

## 许可证

见 [LICENSE](./LICENSE) — Yuuyuko-uu 插件许可 1.2。