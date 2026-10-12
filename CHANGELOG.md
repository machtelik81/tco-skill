# Changelog

## v0.2.0 - 2026-10-11

- Billing mode: ask API vs subscription once; on a subscription report tokens first, USD as notional.
- Gate uses measured usage records before estimating, and prices agent runs by calls × context (cache reads) instead of the first prompt only.
- Borderline rule: decomposition 15-30% cheaper with a real isolation benefit is shown to the user, not decided silently.
- Step 3 compares actual usage (when reported by the tool) with the estimate and names the cause of deviations above 2x.
- `prices.md`: Claude cache-read prices, cache-write factors and the 50% batch discount (partial read, re-verify); new constant `calls_per_agent_run`.
- Two new scenarios (8: model choice from measured data, 9: borderline decomposition).

## v0.1.0 - 2026-10-09

First public release.

- TCO gate: compares total cost of a direct run against a decomposed run and decomposes only when it is clearly cheaper (rule of thumb: at least 30% lower), quality risk is not higher, and at least two subtasks are independent.
- Two-phase gate for plan-only requests (Step 1b).
- Tier table (low / mid / high) with subagent rules; root is the sole integrator.
- Platform mapping for Claude Code (haiku / sonnet / opus) and Codex CLI 0.153.4.
- `prices.md` with a price snapshot (last_updated: 2026-10-08), overhead constants and a staleness warning after 90 days.
- 7 test scenarios in `tco/tests/scenarios.md`.
- Known gaps: the Codex `spawn_agent` model override is derived from binary strings and not verified in a live run [uncertain]; overhead constants and effort multipliers are UNVERIFIED heuristics.
