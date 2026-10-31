# Changelog

All notable changes to this project will be documented in this file.
Each entry includes source references for borrowed ideas/code.

## [0.2.0] - 2026-10-31

### Project Independence
- **Extracted from** `dsh-pet-chubby_blue_whale/integrations/dsh-pet-memory/` to independent project `dsh-cbw-memory`.
- **Package renamed** from `@dsh-pet/memory` to `@dsh-cbw/memory` (register name stays `memory`).
- **GitHub repo**: https://github.com/Scorpioflz/dsh-cbw-memory

### P0: Low-cost high-impact upgrades

#### Added: Explicit/implicit memory request detection
- **Source**: [dsh-mnemon](https://github.com/nicepkg/dsh-mnemon) `src/host/lifecycle.ts:126-137` — `EXPLICIT_MEMORY_CANDIDATE` and `NO_MEMORY_MAINTENANCE` regex arrays
- **Change**: Before injecting memories, check the user's recent message against Chinese/English regex patterns. If "不要记住/don't remember" is matched, append a system prompt note telling the model not to call `remember`. If "记住/remember" is matched, append a note encouraging the model to use `remember`. Injection still happens (user may want to recall existing memories).
- **Files**: `lib/index.js` — added `EXPLICIT_MEMORY_CANDIDATE`, `NO_MEMORY_MAINTENANCE` regex arrays and `matchesAny()` helper (line 131-144), integrated into injection callback (line 4156-4163).

#### Added: Memory deduplication (bigram similarity)
- **Source**: [@tr1v3r/dsh-ltm](https://www.npmjs.com/package/@tr1v3r/dsh-ltm) — dedup concept
- **Change**: Added character-level bigram similarity as an additional dedup condition in the `remember` tool handler. When bigram similarity ≥ 0.8 against an existing memory, the existing memory is reinforced (activation boosted) instead of creating a duplicate. Complements the existing word-set similarity (`像度`) and inclusion checks.
- **Files**: `lib/index.js` — added `bigrams()` and `bigramSimilarity()` functions (line 2862-2871), integrated into `remember` tool dedup loop (line 3427).

#### Added: Dual budget + truncated marker
- **Source**: [dsh-mnemon](https://github.com/nicepkg/dsh-mnemon) `src/core/composition.ts:421-452` — `normalizeEvidence()` dual budget pattern
- **Change**: `recall()` function now returns `truncated` and `truncatedCount` flags. When memories are omitted due to budget/limit constraints, the injection appends "还有 N 条记忆未显示，可调用 recall 工具按需查看" instead of silently truncating.
- **Files**: `lib/index.js` — modified `recall()` return value (line 3234-3238), modified injection callback (line 4230-4236).

### P1: Medium-cost upgrades

#### Added: `recall` tool (on-demand recall)
- **Source**: [dsh-mnemon](https://github.com/nicepkg/dsh-mnemon) — routed View mode (`mnemon_view_route`)
- **Change**: Added a new `recall` tool that models can call with `query` and `limit` parameters to actively retrieve more memories. This complements the existing passive injection — the model can now search for specific memories when the injected set isn't enough. Reuses the existing `recall()` scoring function.
- **Files**: `lib/index.js` — added `makeRecallTool()` function (line 3568-3618), registered globally alongside `remember` in `同步工具()` (line 4341-4358).

#### Added: Capacity management + auto-archive
- **Source**: [dsh-mnemon](https://github.com/nicepkg/dsh-mnemon) `src/host/runtime-capacity.ts`
- **Change**: When memory count exceeds `maxMemories` (default 200), sort by score (importance + activation) and archive the lowest-scoring entries to `archive-<sessionId>-<date>.json`. The main memory file only retains high-score entries. Archived memories are not deleted and can be manually restored. Triggered after each `remember` call.
- **Files**: `lib/index.js` — added `archiveOldMemories()` function (line 1023-1045), integrated into `remember` tool handler (line 3535-3540). Added `maxMemories` to `DEFAULT_POLICY` (line 98) and `POLICY_SPEC` (line 118).

#### Added: Memory source tagging
- **Source**: [dsh-mnemon](https://github.com/nicepkg/dsh-mnemon) — ReadGrant security model (simplified)
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