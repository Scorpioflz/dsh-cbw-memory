# Changelog

All notable changes to this project will be documented in this file.
Each entry includes source references for borrowed ideas/code.

## [0.4.1] - 2026-10-31

### Route prefix unification: `/dsh-pet-memory/` → `/dsh-cbw-memory/`

#### Changed: HTTP route prefix renamed to match plugin package name
- **Reason**: Plugin extracted from `dsh-pet` to independent `@dsh-cbw/memory` package, but all HTTP route prefixes still used the old `dsh-pet-memory` name. Unified to `dsh-cbw-memory` for consistency.
- **Change**: All `mroute('/dsh-pet-memory/...')` → `mroute('/dsh-cbw-memory/...')` (~35 routes), data directory `~/.dsh/dsh-pet-memory` → `~/.dsh/dsh-cbw-memory`, backup dir, config path, diag log path, client-side base prefix `B`, and `plugin` source identifier all renamed.
- **Files**: `lib/index.js` (42 occurrences), `lib/client.js` (1 occurrence: `const B`), `pet/chat/dsh_provider.py` in dsh-pet (1 occurrence: toggle URL)
- **Migration**: Development stage — no data migration needed. Existing users would need to rename `~/.dsh/dsh-pet-memory` → `~/.dsh/dsh-cbw-memory` manually.

## [0.4.0] - 2026-10-31

### P2+P3: Idle review + archive index + security model + LRU increase + archive dedup

#### Added: 4.7 Incremental timed sediment (idle review)
- **Source**: dsh-mnemon `skill-evolution.ts` (original), revised to incremental timed approach
- **Change**: Timer-based incremental extraction (default every 30 minutes, off by default). Checks for active SSE streams (skips if active), extracts incremental conversation since last review, skips if < 500 chars (0 token), detects idle chat via heuristics (no tool_call + no code blocks + no file paths + < 1000 chars → skip, 0 token), calls model to extract memories from engineering conversations. Extracted memories saved with `source: 'idle_review'`.
- **Config**: `idleReviewEnabled` (default false), `idleReviewInterval` (30 min), `idleReviewMinChars` (500), `idleReviewIdleChatThreshold` (1000)
- **Files**: `lib/index.js` — added config to `DEFAULT_POLICY`/`POLICY_SPEC`, `idleReview()` function in `apply()`, timer via `ctx.interval(60000)`

#### Added: 4.11 Security model enhancement for `remember` tool
- **Source**: dsh-mnemon ReadGrant (simplified), enhancement of P1 source marking
- **Change**: `source` parameter now validates against three allowed values: `user_explicit`, `model_observed`, `idle_review`. Invalid values default to `model_observed`. `idle_review` source has lowest priority.
- **Files**: `lib/index.js` — modified `source` parameter validation in `makeRememberTool`

#### Added: 4.12 Archive index (Redis/SQL model)
- **Source**: User-proposed, inspired by Redis/SQL layered architecture
- **Change**: When memories are archived, an `archive_index` record is left in main memory pointing to the archive file + content summary. `recall()` checks `archive_index` records and loads actual memories from archive files on query match (bigram similarity > threshold). Archive is no longer dead storage — it remains queryable.
- **Files**: `lib/index.js` — modified `archiveOldMemories()` to write index records, modified `recall()` to load from archive on match

#### Changed: 4.13 LRU threshold increase
- **Change**: `maxMemories` default increased from 200 to 500. `POLICY_SPEC` max increased from 1000 to 2000. With archive index (4.12), larger main memory doesn't affect query efficiency.
- **Files**: `lib/index.js` — `DEFAULT_POLICY.maxMemories` 200→500, `POLICY_SPEC` max 1000→2000

#### Added: 4.14 Archive deduplication (anti-cycling)
- **Source**: User-proposed, extension of P0 dedup logic to archive layer
- **Change**: Before adding memories to archive file, checks existing archive content for bigram similarity > 0.8. Similar memories are skipped (only removed from main memory, not re-archived). Prevents record→archive→re-record→re-archive cycles.
- **Files**: `lib/index.js` — modified `archiveOldMemories()` to check existing archive before appending

#### Confirmed already implemented: 4.9 Management panel + 4.10 CJK bigram
- **4.9**: Memory management panel (list/search/edit/delete/batch/restore) already exists in `lib/client.js` with backend API in `lib/index.js` (`/list`, `/edit`, `/delete` routes)
- **4.10**: CJK-aware tokenization already exists via `分词()` (word-table segmentation) with `老分词()` bigram fallback in `lib/index.js`

## [0.3.0] - 2026-10-31

### P2: JSONL storage + category field

#### Changed: JSONL storage format
- **Source**: [@restregular/dsh-memory](https://www.npmjs.com/package/@restregular/dsh-memory) — JSONL storage concept
- **Change**: Memory files changed from JSON (single object) to JSONL (one record per line). Each line is a JSON object with a `type` field: `meta` (version/purgeEpoch/policy), `memory`, `trace`, or `deleted`. `loadStore`/`saveStore` interfaces unchanged — only the file format changed. Automatic migration from old `.json` to new `.jsonl` on first load (old file deleted after successful migration).
- **Why**: JSONL enables future append-only writes (O(1) vs O(n) read-modify-write), and each memory is independently readable by external tools. Migration cost is near-zero since current memory is empty (development stage).
- **Files**: `lib/index.js` — added `jsonlSerialize()`/`jsonlParse()` functions, `storePathLegacy()` for migration. Modified `storePath()` (`.json`→`.jsonl`), `loadStore()` (migration + JSONL parse), `saveStore()` (JSONL serialize), `archiveOldMemories()` (`.jsonl` archive), `blank()` (version 2→3). Updated 4 `readdirSync(STORE_DIR)` loops to match both `.jsonl` and `.json`.

#### Added: `category` parameter in `remember` tool
- **Source**: General-assistant positioning requirement (desktop pet evolving toward universal DSH front-end)
- **Change**: `remember` tool now accepts an optional `category` parameter. Reserved values: `user_preference`, `fact`, `project_context`, `task_state`, `conversation`, `general`. Default behavior unchanged (`stable_profile` for importance≥1, `event` otherwise) — new categories only apply when explicitly specified. No classification logic implemented yet; this lays the groundwork for future per-category half-life and retrieval.
- **Files**: `lib/index.js` — added `category` parameter to `remember` tool definition, modified memory object creation.

#### Fixed: cordis.patch.yml and client.js name mismatch
- **Change**: `cordis.patch.yml` `name` and `client.js` `id` updated from `@dsh-pet/memory` to `@dsh-cbw/memory` to match `package.json`. Without this fix, DSH would silently skip the plugin (cordis.patch.yml) or fail to activate it (client.js).

## [0.2.0] - 2026-10-31

### Project Independence
- **Extracted from** `dsh-pet-chubby_blue_whale/integrations/dsh-pet-memory/` to independent project `dsh-cbw-memory`.
- **Package renamed** from `@dsh-pet/memory` to `@dsh-cbw/memory` (register name stays `memory`).
- **GitHub repo**: https://github.com/Scorpioflz/dsh-cbw-memory

### P0: Low-cost high-impact upgrades

#### Added: Explicit/implicit memory request detection
- **Source**: [dsh-mnemon](https://github.com/omdsh-dev/dsh-mnemon) `src/host/lifecycle.ts:126-137` — `EXPLICIT_MEMORY_CANDIDATE` and `NO_MEMORY_MAINTENANCE` regex arrays
- **Change**: Before injecting memories, check the user's recent message against Chinese/English regex patterns. If "不要记住/don't remember" is matched, append a system prompt note telling the model not to call `remember`. If "记住/remember" is matched, append a note encouraging the model to use `remember`. Injection still happens (user may want to recall existing memories).
- **Files**: `lib/index.js` — added `EXPLICIT_MEMORY_CANDIDATE`, `NO_MEMORY_MAINTENANCE` regex arrays and `matchesAny()` helper (line 131-144), integrated into injection callback (line 4156-4163).

#### Added: Memory deduplication (bigram similarity)
- **Source**: [@tr1v3r/dsh-ltm](https://www.npmjs.com/package/@tr1v3r/dsh-ltm) — dedup concept
- **Change**: Added character-level bigram similarity as an additional dedup condition in the `remember` tool handler. When bigram similarity ≥ 0.8 against an existing memory, the existing memory is reinforced (activation boosted) instead of creating a duplicate. Complements the existing word-set similarity (`像度`) and inclusion checks.
- **Files**: `lib/index.js` — added `bigrams()` and `bigramSimilarity()` functions (line 2862-2871), integrated into `remember` tool dedup loop (line 3427).

#### Added: Dual budget + truncated marker
- **Source**: [dsh-mnemon](https://github.com/omdsh-dev/dsh-mnemon) `src/core/composition.ts:421-452` — `normalizeEvidence()` dual budget pattern
- **Change**: `recall()` function now returns `truncated` and `truncatedCount` flags. When memories are omitted due to budget/limit constraints, the injection appends "还有 N 条记忆未显示，可调用 recall 工具按需查看" instead of silently truncating.
- **Files**: `lib/index.js` — modified `recall()` return value (line 3234-3238), modified injection callback (line 4230-4236).

### P1: Medium-cost upgrades

#### Added: `recall` tool (on-demand recall)
- **Source**: [dsh-mnemon](https://github.com/omdsh-dev/dsh-mnemon) — routed View mode (`mnemon_view_route`)
- **Change**: Added a new `recall` tool that models can call with `query` and `limit` parameters to actively retrieve more memories. This complements the existing passive injection — the model can now search for specific memories when the injected set isn't enough. Reuses the existing `recall()` scoring function.
- **Files**: `lib/index.js` — added `makeRecallTool()` function (line 3568-3618), registered globally alongside `remember` in `同步工具()` (line 4341-4358).

#### Added: Capacity management + auto-archive
- **Source**: [dsh-mnemon](https://github.com/omdsh-dev/dsh-mnemon) `src/host/runtime-capacity.ts`
- **Change**: When memory count exceeds `maxMemories` (default 200), sort by score (importance + activation) and archive the lowest-scoring entries to `archive-<sessionId>-<date>.json`. The main memory file only retains high-score entries. Archived memories are not deleted and can be manually restored. Triggered after each `remember` call.
- **Files**: `lib/index.js` — added `archiveOldMemories()` function (line 1023-1045), integrated into `remember` tool handler (line 3535-3540). Added `maxMemories` to `DEFAULT_POLICY` (line 98) and `POLICY_SPEC` (line 118).

#### Added: Memory source tagging
- **Source**: [dsh-mnemon](https://github.com/omdsh-dev/dsh-mnemon) — ReadGrant security model (simplified)
- **Change**: `remember` tool now accepts a `source` parameter: `user_explicit` (user explicitly asked to remember) or `model_observed` (model decided to remember, default). The source is stored in each memory entry, distinguishing user-requested memories from model-observed ones.
- **Files**: `lib/index.js` — added `source` parameter to `remember` tool definition (line 3326-3330), modified memory object creation (line 3520).

## [0.1.0] - 2026-10-20

### Inherited from @dsh-pet/memory (pre-extraction)

- Per-workspace memory sharing (vs. upstream per-session)
- `<system-reminder>` machine-source query filtering
- queryGate small-lib guard (0.8→0.5, skip when <10 memories)
- Settings UI session→workspace wording
- Packed-dir U+FFFD encoding fix
- Half-life decay with importance-modulated H
- Emotion decay with independent half-life
- Rarity saturation (rarityK)
- Query information gate (queryGate)
- Long-term facts (factsMax)
- Daily quota (dailyQuota)
- Session-independent memory files
- Timezone auto-adaptation
- Weight auto-normalization (fitWeights)

### Upstream

- **Source**: [Yuuyuko-uu/dsh-fish-memory](https://github.com/Yuuyuko-uu/dsh-fish-memory)
- **License**: Yuuyuko-uu Plugin License 1.2
- **Original author**: Yuuyuko-uu