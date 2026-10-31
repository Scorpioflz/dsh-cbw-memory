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
- **Source**: [dsh-mnemon](https://github.com/nicepkg/dsh-mnemon) `src/host/lifecycle.ts` — `EXPLICIT_MEMORY_CANDIDATE` and `NO_MEMORY_MAINTENANCE` regex arrays
- **Change**: Before injecting memories, check the user's recent message against Chinese/English regex patterns. If "不要记住/don't remember" is matched, skip auto-recording. If "记住/remember" is matched, boost recording priority.
- **Files**: `lib/index.js` — added `EXPLICIT_MEMORY_CANDIDATE` and `NO_MEMORY_MAINTENANCE` regex arrays, integrated into injection logic.

#### Added: Memory deduplication (bigram similarity)
- **Source**: [@tr1v3r/dsh-ltm](https://www.npmjs.com/package/@tr1v3r/dsh-ltm) — dedup concept
- **Change**: Before writing a new memory, compare its content against the most recent N entries using bigram similarity. If similarity > 0.8, merge (update activation/importance) instead of creating a new entry.
- **Files**: `lib/index.js` — added `bigramSimilarity()` function, integrated into `remember` tool handler.

#### Added: Dual budget + truncated marker
- **Source**: [dsh-mnemon](https://github.com/nicepkg/dsh-mnemon) `src/core/composition.ts` — `normalizeEvidence()` dual budget pattern
- **Change**: Memory injection now uses both `maxItems` and `maxChars` as dual limits. When either is exceeded, a `[还有 N 条记忆未显示，可用 recall 工具查看]` marker is appended instead of silently truncating.
- **Files**: `lib/index.js` — modified `recall()` function and injection logic.

### P1: Medium-cost upgrades

#### Added: `recall` tool (on-demand recall)
- **Source**: [dsh-mnemon](https://github.com/nicepkg/dsh-mnemon) — routed View mode (`mnemon_view_route`)
- **Change**: Added a new `recall` tool that models can call with `query` and `limit` parameters to actively retrieve more memories. This complements the existing passive injection (reduce passive `maxItems` from 6 to 4, let the model decide when to recall more).
- **Files**: `lib/index.js` — added `recall` tool definition and handler.

#### Added: Capacity management + auto-archive
- **Source**: [dsh-mnemon](https://github.com/nicepkg/dsh-mnemon) `src/host/runtime-capacity.ts`
- **Change**: When memory count exceeds `maxMemories` (default 200), sort by score and archive the lowest-scoring entries to `archive-<sessionId>-<date>.json`. The main memory file only retains high-score entries. Archived memories are not deleted and can be manually restored.
- **Files**: `lib/index.js` — added `archiveOldMemories()` function, integrated into `remember` tool handler. Added `maxMemories` to `DEFAULT_POLICY` and `POLICY_SPEC`.

#### Added: Memory source tagging
- **Source**: [dsh-mnemon](https://github.com/nicepkg/dsh-mnemon) — ReadGrant security model (simplified)
- **Change**: `remember` tool now accepts a `source` parameter: `user_explicit` (user explicitly asked to remember) or `model_observed` (model decided to remember). Explicit memories get a 1.2× score boost during recall.
- **Files**: `lib/index.js` — modified `remember` tool definition and recall scoring.

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