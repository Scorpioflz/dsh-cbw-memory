# @dsh-cbw/memory

<p align="center">
  <a href="./README_ZH.md">中文</a> | <a href="./README.md">English</a>
</p>

A DeepSeek Harness (DSH) long-term memory plugin with half-life decay, emotion weighting, and per-workspace sharing.

> **Derivative work** of [Yuuyuko-uu/dsh-fish-memory](https://github.com/Yuuyuko-uu/dsh-fish-memory) — original author **Yuuyuko-uu**, used under the **Yuuyuko-uu Plugin License 1.2**. See [§ Attribution](#attribution) for the full list of changes relative to upstream.

## Features

- **Half-life forgetting** — `A = A₀ × 2^(-days / H)`, where H is modulated by importance (`H = base × (1 + 2×I)`). Important memories decay slower.
- **Emotion decay** — Independent emotion half-life (default 7 days); emotional memories fade faster than factual ones.
- **Rarity saturation** — `C = s / (s + K)` controls how much rare-word hits contribute vs. common words.
- **Query gate** — Low-information queries (e.g., "hello") skip memory recall entirely.
- **Long-term facts** — `factsMax` facts are injected every turn regardless of the current query, without occupying the `maxItems` slot.
- **Daily quota** — Limits how many memories the model can record per day (default 8).
- **Per-workspace sharing** — Memories are shared within a DSH workspace, not per-session.
- **Session-independent** — Each agent has its own memory file, keyed by session ID.
- **Timezone auto-adaptation** — Automatically detects the machine's timezone.
- **Weight auto-normalization** — The four scoring weights always sum to 100%.

### v0.2.0 Upgrades (P0 + P1)

- **Explicit/implicit memory request detection** — When the user says "don't remember" or "记住", the plugin adjusts behavior accordingly. [Borrowed from dsh-mnemon `lifecycle.ts`]
- **Memory deduplication** — New memories are compared against recent entries using bigram similarity; similar entries are merged instead of duplicated. [Borrowed from @tr1v3r/dsh-ltm dedup concept]
- **Dual budget + truncated marker** — `maxItems` + `maxChars` dual limit; when exceeded, a truncated marker is shown instead of silently cutting off. [Borrowed from dsh-mnemon `composition.ts:normalizeEvidence`]
- **`recall` tool** — Models can actively recall more memories on demand, rather than only receiving passive injection. [Borrowed from dsh-mnemon routed View mode]
- **Capacity management + auto-archive** — When memory count exceeds `maxMemories` (default 200), low-score entries are archived to a separate file. [Borrowed from dsh-mnemon `runtime-capacity.ts`]
- **Memory source tagging** — `remember` tool accepts a `source` parameter (`user_explicit` / `model_observed`); distinguishes user-requested memories from model-observed ones. [Borrowed from dsh-mnemon ReadGrant concept, simplified]

### v0.3.0 Upgrades (P2)

- **JSONL storage + category** — Memories stored as JSONL with a `category` field (`user_preference`/`fact`/`project_context`/`task_state`/`conversation`/`general`). Half-life varies by category: `project_context` ×3, `fact`/`user_preference`/`task_state` ×2, others ×1.
- **Incremental idle review** — Timer-based incremental extraction (30 min default, skips chitchat, prioritizes engineering dialogs). Off by default.

### v0.4.0 Upgrades (P3)

- **Security model** — `source` parameter validated against allowed values (`user_explicit`/`model_observed`/`idle_review`); no forged IDs.
- **Archive index** — `archive_index` records point to archive files; `recall` loads on demand (Redis/SQL pattern).
- **LRU threshold raised** — `maxMemories` 200→500.
- **Archive deduplication** — Bigram similarity check before archiving, prevents cyclic re-archiving.
- **CJK bigram search** — CJK-aware tokenized search (already present, formally documented).

### v0.4.1

- **Route prefix unified** — All HTTP routes `/dsh-pet-memory/` → `/dsh-cbw-memory/`, consistent with package name.
- **Plugin market index** — Published to DSH plugin market via GitHub Pages (`docs/index.json`).

## Scoring Formula

```
P = C × (baselineWeight + activationWeight × A + importanceWeight × I + emotionWeight × E)
```

Where:
- **C (evidence)**: `C = s / (s + K)`, s = sum of hit-word rarities, K = `rarityK` (0.8)
- **A (activation)**: `A = A₀ × 2^(-days / H)`, H = `baseHalfLifeDays` (30) × (1 + 2×I) × `categoryMult` (project_context: 3, fact/user_preference/task_state: 2, else: 1)
- **E (emotion)**: `E = emotion × 2^(-days / emotionHalfLifeDays)`, `emotionHalfLifeDays` (7)
- **I (importance)**: User-annotated importance level

Default weights: `baselineWeight=0.55, activationWeight=0.25, importanceWeight=0.15, emotionWeight=0.05`

Threshold: `threshold=0.3` (gates on C only, not activation), `maxItems=6, maxChars=800, dailyQuota=8`

## Installation

```bash
# In DSH settings → plugins, add:
dsh plugin --profile <profile> add github:Scorpioflz/dsh-cbw-memory

# Or from local path:
dsh plugin --profile <profile> add /path/to/dsh-cbw-memory
```

> **Note**: `@dsh-cbw/memory` and the original `dsh-fish-memory` both register the `remember` tool — **do not install both in the same DSH profile**. If fish-memory is already installed, remove it first: `dsh plugin --profile <profile> remove dsh-fish-memory`.

## Configuration

All settings are configurable via the DSH settings page (memory panel) or by editing `~/.dsh/dsh-cbw-memory-config.json`.

| Parameter | Default | Description |
|---|---|---|
| `baseHalfLifeDays` | 30 | Base half-life for activation decay |
| `emotionHalfLifeDays` | 7 | Half-life for emotion decay |
| `baselineWeight` | 0.55 | Baseline weight (always present) |
| `activationWeight` | 0.25 | Activation (recency) weight |
| `importanceWeight` | 0.15 | Importance weight |
| `emotionWeight` | 0.05 | Emotion weight |
| `threshold` | 0.3 | Evidence threshold (gates on C only) |
| `maxItems` | 6 | Max memories injected per turn |
| `maxChars` | 800 | Max characters injected per turn |
| `dailyQuota` | 8 | Max memories recorded per day |
| `rarityK` | 0.8 | Rarity saturation point |
| `queryGate` | 0.5 | Query information threshold |
| `factsMax` | 2 | Long-term facts injected every turn |
| `unknownActivation` | 0.5 | Activation for memories without timestamps |
| `maxMemories` | 500 | Max memories before auto-archive (v0.4.0+: raised from 200) |

## Tools

| Tool | Parameters | Description |
|---|---|---|
| `remember` | `text`, `importance`, `tags`, `取代`, `事实键`, `source` | Record a memory |
| `recall` | `query`, `limit` | Actively recall memories matching a query (v0.2.0+) |

## Attribution

This plugin is a derivative work of [Yuuyuko-uu/dsh-fish-memory](https://github.com/Yuuyuko-uu/dsh-fish-memory), used under the **Yuuyuko-uu Plugin License 1.2** (see [LICENSE](./LICENSE)).

Original author: Yuuyuko-uu. The derivative continues under the same license.

### Changes relative to upstream

- Per-workspace memory sharing (vs. per-session)
- `<system-reminder>` machine-source query filtering
- queryGate small-lib guard (0.8→0.5, skip when <10 memories)
- Settings UI session→workspace wording
- Packed-dir U+FFFD encoding fix
- Package renamed to `@dsh-cbw/memory` (register name stays `memory` for `remember` tool compatibility)
- v0.2.0: Explicit/implicit request detection, memory deduplication, dual budget + truncated marker, `recall` tool, capacity management + auto-archive, memory source tagging
- v0.3.0: JSONL storage + category field, incremental idle review
- v0.4.0: Security model (source validation), archive index, LRU threshold raised (200→500), archive deduplication, CJK bigram search
- v0.4.1: Route prefix unified (`/dsh-pet-memory/` → `/dsh-cbw-memory/`), plugin market index
- v0.5.0: Category-based half-life differentiation (`project_context` ×3, `fact`/`user_preference`/`task_state` ×2, others ×1)

### Ideas borrowed from

- [dsh-mnemon](https://github.com/omdsh-dev/dsh-mnemon) — Composable View mode, dual budget control, explicit/implicit request detection, idle review, ReadGrant security model, capacity auto-archive
- [@tr1v3r/dsh-ltm](https://www.npmjs.com/package/@tr1v3r/dsh-ltm) — CJK-aware tokenized search, deduplication concept

## License

See [LICENSE](./LICENSE) — Yuuyuko-uu Plugin License 1.2.