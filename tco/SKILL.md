---
name: tco
description: Decides whether a task is worth splitting across subagents and assigns each subtask the cheapest model that solves it correctly in one pass, by comparing total cost of ownership. Invoked on request only (Claude Code /tco, Codex $tco); never activate automatically or from context.
---

# TCO: Cost-Aware Delegation

Goal: minimize total cost of ownership (TCO) of the whole job, not per-task price. One correct run beats several cheap retries. Match model to difficulty up front. Never design "try cheap, escalate on failure" loops.

## Tier rules

| Tier | Use for | Rule |
|---|---|---|
| low | Mechanical search, inventory, classification | Only well-specified work that grep or tests can verify. |
| mid | Default for normal implementation, tests, review | Use when nothing else applies. |
| high | Architecture, critical migrations, security, hard debugging | Hard debugging only after a structured root-cause analysis; never preemptively. |

Subagent rules:
- Use subagents only for real parallelism, isolated context, or independent review.
- No parallel writing in one checkout.
- The root agent is the sole integrator.

This skill adds only the TCO gate on top of these rules.

## Step 0: price and billing check
Read `prices.md` (same directory). If `last_updated` is older than 90 days from today, warn the user first: "prices.md is stale (last_updated: X)". Prices marked UNVERIFIED: use the relative cost factors and say so.

Billing mode decides what "cost" means. If unknown, ask once: API (pay per token) or subscription (usage limits)? On a subscription, prices are only notional list-price equivalents: report tokens as the primary figure and notional USD as a comparison value; the real cost is limit consumption. Compare options by tokens weighted by price, never by dollars alone.

## Step 1: GATE
Use measured numbers before guesses: if the project or tool keeps usage records (session transcripts, usage/cost reports, a token-report script), read per-run tokens for comparable past jobs first and estimate from them. An estimate made without data was off by about 15x in one measured case.

Agent runs are priced by calls × context, not by the first prompt: every call re-reads the whole context (cache reads), so long loops with many calls dominate the cost. Estimate calls and average context size; a model that needs fewer calls can be cheaper despite a higher token price.

Estimate input/output tokens for the whole job, then compare:

TCO(direct) = Σ calls × (context × P_cache_read or P_in + new_in × P_in) + out × P_out for the single best model, plus rework risk. Include context growth over the run.

TCO(decomposed) = Σ subtasks (same formula at its tier)
 + context duplication (N subagents × context_dup × input price)
 + return flow (root re-reads every result)
 + coordination/integration (root, per subagent)
 + expected rework risk (see prices.md constants).

Use `prices.md` cache-read prices where listed; otherwise price re-read context at full input price and say so (this overstates both paths, the direct path more).

Decompose ONLY if all hold:
1. TCO(decomposed) is clearly lower (rule of thumb: ≥30% lower; estimates are rough).
2. Quality risk is not higher.
3. At least 2 subtasks are independent (disjoint files, no shared state), or one needs isolated context or independent review.

Borderline (decomposed 15-30% lower and a real isolation benefit, such as keeping bulky research out of the root context): show both numbers and let the user choose; do not decide silently.

Default to NOT decomposing for: small jobs, sequential steps, overlapping files, shared state, unclear root cause.

## Step 1b: Planning requests (two-phase gate)
Decide first what the invoked request produces:
- Executable job (implement, fix, rename...): run Steps 1-3 once, as written.
- Plan only (plan mode, "analyze and plan", "create an implementation plan"): two phases.
  1. Planning work itself: run the gate on the analysis. Usually do not decompose (unclear root cause, sequential); pick one model for it per the tier rules.
  2. Plan content: once the plan steps are known, run the gate on those steps, not on the original prompt. The plan MUST contain the Execution Allocation table, the TCO comparison and a brief (template below) per delegated step, placed before the implementation steps. Steps that fail the gate are marked "root, no delegation". Approved plans are then executed per the table without re-invoking the skill.
- If execution reveals new or changed steps (e.g. root cause differs from the plan), re-run the gate for those steps only.

## Step 2a: NOT decomposing
Give a 2-3 line justification. Recommend the best single model for the whole job (often the largest saving: do not run a big model on mechanical work, and do not run a small one on a hard single problem). Then proceed in the root agent.

## Step 2b: Decomposing
Output the Execution Allocation table (subtask | model/reasoning | execution | worktree | result) and a TCO comparison (direct vs decomposed, numbers from prices.md, with the assumptions listed).

Tier per subtask, by difficulty and risk:
- Low: mechanical, well-specified, verifiable by grep/tests (search, inventory, rename, classification).
- Mid: normal implementation, tests, review. Default.
- High: architecture, security, critical migrations, hard debugging after structured root-cause analysis.
- Irreversible or security-relevant subtasks never go to the low tier.
- If a subtask is ambiguous in difficulty, pick the higher tier; a failed cheap run costs more than the price gap.
- Parallel writers get separate worktrees. Read-only work needs none.
- Current models tend to spawn subagents too eagerly; keep simple or sequential work in the root.

Confirmation: ask the user ONLY if the estimated decomposed TCO exceeds `confirm_threshold` in prices.md (notional USD on a subscription), or the case is borderline (Step 1); otherwise proceed.

## Subagent brief template (every brief must contain all five)
```
Goal: <one sentence, observable outcome>
File scope: <paths the agent may read/touch>
Allowed changes: <what may be edited; everything else read-only>
Verification: <exact command or check that proves success>
Return (max ~150 words, exactly this format):
  Result: <done / partial / blocked + one line>
  Files changed: <list>
  Open issues: <list or none>
  Test result: <command + pass/fail>
```

## Step 3: Integration and report
Root agent reviews each return, integrates, resolves conflicts, runs verification, and reports the final result with estimated vs. chosen cost (estimated TCO direct, estimated TCO decomposed, chosen path). Add actual usage when the tool reports it (for example subagent token counts or a usage report) and state the deviation from the estimate; if it is more than 2x off, say so and name the likely cause. Never invent actual numbers.

## Platform mapping
Tiers: low / mid / high. Prices in `prices.md`.

| Tier | Claude Code (Agent tool `model`) | Codex CLI 0.153.4 (model, reasoning) |
|---|---|---|
| low | `haiku` | `gpt-5.6-luna`, `low` |
| mid | `sonnet` | `gpt-5.6-terra`, `medium` |
| high | `opus` | `gpt-5.6-sol`, `high` |

Claude Code: pass `model: "haiku" | "sonnet" | "opus"` in the Agent call. Which model version each alias resolves to: [uncertain]; check the Agent tool description or the `model` field in the session records.

Codex: finding on 0.153.4 — the `multi_agent` feature is stable and enabled, and the binary's `spawn_agent` tool text describes optional `model` and `reasoning_effort` overrides (honoured only when `fork_turns` is `"none"` or a positive integer string; full-history forks inherit the parent). Derived from binary strings, not from a live run [uncertain]. Therefore: in Codex, spawn subagents with `spawn_agent` setting `model`, `reasoning_effort` and `fork_turns: "none"` per the table above. If the call rejects these parameters, output the plan only and tell the user to start each step manually. Never invent flags or slugs.
