---
name: bos-cost-optimization
description: "Run gbrain at minimum cost for a single-operator personal brain. Use when installing, tuning, or auditing gbrain spend."
version: 1.0.0
author: Cynthia Jones (BOS)
license: MIT
metadata:
  hermes:
    tags: [gbrain, cost, optimization, ops]
---

# BOS Cost Optimization Playbook

This skill is the operating manual for keeping the BOS gbrain deployment at floor cost. Every optimization in here has been tested on a single-operator scale (~500–5K pages, 1K–5K queries/mo). Read it before any change that touches search mode, embedding provider, or synthesis model.

## The target: $0–5/mo for a single operator

The default gbrain install assumes enterprise scale (100K+ pages, 100K+ queries/mo, multi-user). For a single-operator personal brain, every default is over-provisioned. The minimum-cost posture is:

| Component | Minimum-cost choice | Monthly cost at 1K queries/mo |
|---|---|---|
| Database | PGLite embedded | $0 |
| Embeddings | ZeroEntropy free tier | $0 |
| Reranker | ZeroEntropy free tier | $0 |
| Search mode | `conservative` | $0 (smaller chunks → fewer synthesis tokens) |
| Synthesis model | Haiku 4.5 | $0.80–1.50 |
| LLM expansion | OFF | $0 |
| Cron enrichment | Once weekly, not nightly | $0 |
| Backup storage | Local on the Railway volume | $0 |

**Floor:** ~$1/mo with active use. **Ceiling:** ~$5/mo at moderate use. Anything above $10/mo means a knob has drifted.

## The 7 levers, ranked by $/quality impact

1. **Default to ZeroEntropy free tier.** Free tier covers 10K embeddings/mo + 1K reranks/mo. That's a 500–1,000-page brain with normal use. Don't open an OpenAI key unless you've exhausted this.

2. **`gbrain search` mode = `conservative`.** 4K token budget. Cuts synthesis input by ~3x vs `balanced` with <5% quality loss for typical queries. Set via `gbrain config set search.mode conservative`.

3. **Disable LLM expansion.** `expansion: false`. Saves ~$1.50 per 1K queries. Re-enable only if recall tests fail on real queries.

4. **Haiku 4.5 for synthesis.** $1/M input. Move to Sonnet only when `gbrain think` answers are demonstrably bad on real BOS queries. Run a 20-query eval first; don't switch on a hunch.

5. **Cache aggressively.** `query_cache` defaults are fine (0.92 similarity threshold, 1hr TTL). Don't raise the threshold; lower it only if you see cache pollution.

6. **Prompt caching on the agent side.** If Hermes uses Anthropic prompt caching (`cache_control: ephemeral` on the system prompt), repeat-prefix cost drops ~80% on the synthesis side. Verify with `gbrain think --explain` to see the cache key.

7. **Local Ollama fallback for non-critical queries.** `gbrain search` (raw retrieval) can use local embeddings forever — $0. Use local for browsing, Haiku for synthesis.

## Daily audit script

Run once daily for the first 30 days. Surfaces drift from projection.

```bash
gbrain doctor --json | jq '.costs'
gbrain stats --since 24h
```

Expected output during steady state:

```json
{
  "queries_24h": <50>,
  "cache_hit_rate": ">=0.90",
  "synthesis_calls_24h": <50>,
  "embedding_calls_24h": <5>,
  "reranker_calls_24h": <50>
}
```

**Red flags:**
- `cache_hit_rate < 0.80` → you're running unique queries that should be cached. Check `gbrain search stats` for distribution.
- `embedding_calls_24h > queries_24h / 10` → reindex is running or content changed. Confirm it's intentional.
- `synthesis_calls_24h > queries_24h` → some queries are calling synthesis twice. Check the agent's loop.

## Mid-month optimization sweep (Day 15)

```bash
# 1. Check actual vs projected
cat > /tmp/audit.md <<'EOF'
## Month 1 mid-audit
- [ ] Cache hit rate >90%
- [ ] Synthesis model still Haiku (not Sonnet/Opus)
- [ ] Search mode still conservative
- [ ] Embeddings still on ZE free tier
- [ ] No nightly cron running expensive enrichments
- [ ] No reindex storm from gitignore drift
EOF

# 2. Re-run quality eval to confirm conservative mode is still good enough
gbrain eval longmemeval --sample 20
gbrain think "what did I work on with [client name]?" --explain

# 3. If quality regressed, ONE upgrade at a time:
gbrain config set search.mode balanced   # only if recall is bad
gbrain config set synthesis.model sonnet   # only if reasoning is bad
```

**Never upgrade two knobs at once.** If quality regresses, you can't tell which change caused it.

## Anti-patterns (what makes the bill spike)

- **Reimport storms.** If your `~/.gbrain/inbox/` or imported Notion folder has churn, gbrain re-embeds everything. Check `gbrain sync stats`. If `reindexed_pages > 100/day` from churn, switch that source to `db_only` (gitignored, DB-only) instead of `git_tracked`.
- **Nightly dream cycle on a busy brain.** Default is `every 24h`. If your brain is large and you don't need it, drop to `weekly` or disable. Check `gbrain doctor | grep dream`.
- **Opus for synthesis "just in case".** Opus is 5x Sonnet, 5x Haiku. The quality difference on retrieval-augmented answers is rarely worth it for personal use.
- **`tokenmax` search mode.** Unbounded token budget. Cost can spike 3x without warning. Only use for debug sessions.
- **Forgetting to set `expansion: false`.** This is the most expensive silent default. $15/mo at 10K queries with no benefit unless your queries are genuinely ambiguous.

## When to scale up (the right reasons)

Move from conservative → balanced when:
- You have >5K pages AND
- Recall tests show <70% P@5 on your real queries AND
- You've tried `expansion: true` alone (cheaper upgrade) and it wasn't enough

Move from Haiku → Sonnet when:
- `gbrain think` answers hallucinate facts you know are in the brain AND
- You've confirmed the retrieval is finding the right pages (`gbrain search --explain`) AND
- The hallucination isn't a frontmatter/extraction problem (run `gbrain doctor suspected-contradictions` first)

Move from PGLite → Supabase when:
- Brain size >50K pages AND
- You need multi-machine access (laptop + server) AND
- You're ready to pay Supabase's $25/mo Pro plan minimum

## Cost tracker integration

All actual spend is logged to the **Brain Cost Tracker** database in Notion HQ. Each `gbrain think` call should record:

- Service (ZeroEntropy, Anthropic, OpenAI, etc.)
- Use (Embedding, Reranker, Synthesis)
- Date
- Cost (USD)
- Query count
- Tokens in / out
- Model
- Optimization flag

If the field is empty, the operator forgot to log. Make logging part of the daily cron.

## Emergency: how to stop the bleeding

If `gbrain doctor` reports daily cost >$5 for 3 consecutive days:

1. `gbrain serve --http` ← find the live process
2. `kill <pid>` ← stop synthesis immediately
3. `gbrain config set synthesis.model haiku` ← force the cheap model
4. `gbrain config set search.mode conservative` ← force minimum search
5. Restart, audit what changed

The brain keeps working in read-only mode (`gbrain search`) without synthesis. You'll still get raw retrieval; you just won't get the synthesized answer. Cost drops to ~$0 immediately.

## Recovery: how to find the leak

```bash
# 1. What queries ran in the last 24h?
gbrain query_log --since 24h

# 2. Which used the most tokens?
gbrain query_log --since 24h --sort tokens

# 3. Were they unique or duplicates?
gbrain query_log --since 24h --group query_hash | head

# 4. Was expansion on?
grep "expansion" ~/.gbrain/config.json

# 5. Was the synthesis model what you expected?
grep "synthesis" ~/.gbrain/config.json
```

## What this skill does NOT cover

- Multi-user / company-brain cost (different scale, different levers)
- Self-hosted embedding server costs (different model)
- Cron enrichment economics (covered in `gbrain dream` skill)
- Backup storage costs (covered in `gbrain backup` skill)