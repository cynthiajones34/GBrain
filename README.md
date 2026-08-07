# GBrain for BOS

A cost-optimized personal-knowledge brain for running The Builders' Ops Studio.

This is a fork of [garrytan/gbrain](https://github.com/garrytan/gbrain) (MIT, 27.9k★) tuned for a single-operator boutique consultancy at minimum monthly spend.

## What's different from upstream

- **Default cost posture: ~$1–5/mo** for active use. Upstream defaults assume enterprise scale.
- **Conservative search mode + Google Gemini free tier + Haiku synthesis** set as the install defaults. Zero new-account cost (Gemini uses your existing Google login). Zero surprise bills.
- **`skills/bos-cost-optimization/`** — operating manual for keeping spend at floor.
- **Brain Cost Tracker schema** documented in this README so the spend is auditable, not implicit.

The engine, hybrid search, MCP server, OAuth admin, and 43 upstream skills are unchanged. They're excellent. Don't fork what works.

## Quick start (cost-optimized)

```bash
# 1. Install
curl -fsSL https://bun.sh/install | bash
export PATH="$HOME/.bun/bin:$PATH"
bun install -g github.com/cynthiajones34/GBrain

# 2. Set the API keys BEFORE init so the picker auto-selects the cheapest recipe
export GOOGLE_GENERATIVE_AI_API_KEY=<from https://aistudio.google.com/apikey>
export ANTHROPIC_API_KEY=<from https://console.anthropic.com>

# 3. Init with PGLite (zero infra cost) and Gemini free-tier embeddings
gbrain init --pglite --model google:gemini-embedding-001 --embedding-dimensions 768

# 4. Force conservative search mode (smallest token budget)
gbrain config set search.mode conservative
gbrain config set synthesis.expansion false
gbrain config set synthesis.model claude-haiku-4-5

# 5. Wire as MCP memory to your existing Hermes Agent
gbrain serve --http &
gbrain connect https://your-hermes-host/mcp --install

# 6. Import your existing Notion content (free tier covers ~500 pages)
gbrain import ~/notion-export/

# 7. Verify
gbrain doctor
gbrain think "what did I work on with [client name]?"
```

## Cost target

| Component | Choice | Monthly cost |
|---|---|---|
| Database | PGLite embedded | $0 |
| Embeddings | Google Gemini `gemini-embedding-001` (768d, free tier: 1,500 req/day) | $0 |
| Reranker | (none — hybrid retrieval uses vector + BM25 + RRF) | $0 |
| Search mode | conservative (4K token budget) | $0 |
| Synthesis | Haiku 4.5 | $0.80–2.50 |
| LLM expansion | OFF | $0 |
| **Total at 1K queries/mo** | | **~$1–2.50** |

Anything above $5/mo means a knob has drifted. See `skills/bos-cost-optimization/SKILL.md`.

## Tracking spend

All actual spend is logged to the **Brain Cost Tracker** database in your Notion HQ. Each cost event records service, use case, cost, query count, tokens, model, and an optimization flag. Daily audit cron surfaces drift within 24 hours.

## When to upgrade (only after measuring)

- `conservative` → `balanced` search mode: only if recall tests fail
- Haiku → Sonnet synthesis: only if `gbrain think` answers hallucinate facts you know are in the brain
- PGLite → Supabase: only at 50K+ pages or multi-machine access

**Never upgrade two knobs at once.** You won't know which change caused what.

## What's upstream, unchanged

All credit for the engine, search stack, MCP server, OAuth admin, dream cycle, and eval framework goes to Garry Tan and the gbrain contributors. This fork adds cost discipline; it does not replace engineering.

Read `skills/RESOLVER.md` for skill routing. Read `CLAUDE.md` if you're Claude Code working on this fork.

## License + credit

MIT. Engine and architecture © Garry Tan and gbrain contributors. Cost-optimization layer © Cynthia Jones / The Builders' Ops Studio.

Upstream: [github.com/garrytan/gbrain](https://github.com/garrytan/gbrain)
Fork: [github.com/cynthiajones34/GBrain](https://github.com/cynthiajones34/GBrain)