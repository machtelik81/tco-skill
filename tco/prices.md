# prices.md — editable price table for the tco skill

last_updated: 2026-10-08
confirm_threshold: 2.00   # USD. Ask the user before executing a plan whose estimated total exceeds this. Edit freely.

All prices: USD per 1M tokens, standard tier. Cache-read prices and the batch discount are listed separately below; the Input/Output columns are the uncached standard rates.

## Claude (source: https://platform.claude.com/docs/en/about-claude/pricing, fetched 2026-10-08)

| Tier | Model | Input | Output | Status |
|---|---|---|---|---|
| low | Claude Haiku 5.5 (prompts ≤100k) | 0.10 | 0.50 | verified (over 100k tokens: 0.50 / 2.50) |
| mid | Claude Sonnet 5.5 | 2.00 | 10.00 | verified |
| high | Claude Opus 5.5 | 4.00 | 20.00 | verified |
| (unmapped) | Claude Fable 5.1 | 10.00 | 50.00 | verified; not used by this skill |

### Claude cache and batch (added 2026-10-11 from the caching and pricing pages; a partial read, re-verify before large jobs)

| Model | Cache read (per 1M) | Note |
|---|---|---|
| Haiku 5.5 (≤100k) | 0.01 | |
| Sonnet 5.5 | 0.10 | 0.05× base input |
| Opus 5.5 | 0.20 | 0.05× base input |
| Fable 5.1 | UNVERIFIED | assume 0.1× base input (1.00) and say so |

Cache write: 1.25× base input (5-minute TTL) or 2× (1-hour TTL). Message Batches API: 50% off, stacks with caching; only for the API, not for CLI sessions. Prompts below the model's minimum cacheable length are not cached.

## GPT-5.6 / Codex (source: https://developers.openai.com/api/docs/pricing, fetched 2026-10-08, standard tier, ≤272K input)

| Tier | Model | Input | Output | Status |
|---|---|---|---|---|
| low | gpt-5.6-luna | 0.20 | 1.20 | verified (via page fetch; re-check before large jobs) |
| mid | gpt-5.6-terra | 2.00 | 12.00 | verified (same) |
| high | gpt-5.6-sol | 4.00 | 20.00 | verified (same) |

Reasoning effort (low/medium/high) changes billed output tokens, not the per-token price. Effort-to-token multipliers: UNVERIFIED. Fallback relative output-volume factors (heuristic, not measured): low 1.0 / medium 1.5 / high 2.5.
Cached-input prices: UNVERIFIED for GPT-5.6; the skill then prices re-read context at full input price (overstates both paths).

## Fallback relative cost factors (use only if a price above is UNVERIFIED or stale)

Blended cost per task, low tier = 1: mid ≈ 12, high ≈ 25 (Claude, derived from the table above at a 4:1 input:output mix). Heuristic.

## Overhead constants (heuristics, UNVERIFIED — tune from experience)

- context_dup: tokens each subagent re-reads (instructions + shared files). Default 5,000.
- return_flow: ~300 tokens per subagent result, read by the root at root input price.
- integration: per subagent, root spends ~2,000 input + 1,000 output tokens at root-tier price.
- rework_risk: expected redo cost = P(fail) × cost of subtask. Default P: 5% when tier fits difficulty, 30% if one tier too low, 60% if two tiers too low.
- calls_per_agent_run: number of model calls an agentic job makes. No safe default: measure it from past runs of the same kind (session records, usage reports). Observed in one project: 24 to 180 calls per run, with 50k to 150k context re-read on each call; do not reuse these numbers elsewhere.
