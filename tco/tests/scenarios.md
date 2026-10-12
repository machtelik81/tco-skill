# tests/scenarios.md

Each scenario: input, expected decision, and the gate rule that yields it. Token numbers are rough illustrative estimates using prices.md (Claude column; the Codex column has the same shape). They test the rules, not the arithmetic precision.

| # | Task | Expected decision | Tiers |
|---|---|---|---|
| 1 | Single typo fix in one file | Do NOT decompose | one mid-or-low model, root |
| 2 | Bug, unclear root cause, one function | Do NOT decompose | one strong model (high) |
| 3 | Rename a term across 40 files | Decompose, parallel | low for all workers |
| 4 | New feature: analysis, implementation, tests | Decompose | mixed: low analysis, mid impl, mid tests |
| 5 | Two tightly coupled modules, same files | Do NOT decompose | one model, root |
| 6 | Bug in a reporting script: a NaN value passes an `is None` guard and yields a wrong status | Do NOT decompose | one mid model, root |
| 7 | Plan-mode prompt: analyze an error and plan the fix | Two-phase: analysis not decomposed; plan steps gated, allocation embedded in the plan | analysis: one model; steps: per gate |
| 8 | Choose a model for a recurring agent routine; past run records exist | Decide on measured cost per correct run | per measured data |
| 9 | Research split into 3 independent blocks, decomposed only ~20% cheaper but keeps bulky web content out of the root | Borderline: show both, ask the user | mid workers |

## 1. Typo fix
Gate: ~3k in / 0.2k out. Direct ≈ $0.01 even on mid. Decomposed adds context_dup + integration (> the work itself); no independent subtasks (rule 3 fails). Decision: no decompose. Recommend low or mid in root (rewording a typo needs no high tier). Match.

## 2. Unclear root cause, one function
Gate: single function, no independent streams (rule 3 fails); unclear root cause is a default-NO trigger; quality risk of splitting is higher (fragmented hypotheses). Recommend ONE high-tier model (hard debugging); Opus/Sol, not a cheap-first loop. Per the tier rules, high only after structured root-cause analysis: do that analysis first in root. Decision: no decompose, one strong model. Match.

## 3. Rename term in 40 files
Gate: ~80k in / 12k out. Direct on mid (Sonnet): 0.16 + 0.12 ≈ $0.28 (context grows, so likely higher). Decomposed: 4 low-tier workers × 10 files: workers ≈ $0.014, dup 4×5k×0.10/1M ≈ $0.002, root integration (mid) ≈ $0.07 → ≈ $0.09, about 68% lower. Files are disjoint (rule 3 holds); quality risk equal (mechanical, verifiable by grep). Task is mechanical and not irreversible → low tier OK. Separate worktrees for parallel writers. Total < confirm_threshold (2.00) → proceed without asking. Decision: decompose, lowest tier, parallel. Match.

## 4. New feature (analysis, implementation, tests)
Gate assumptions: direct on mid carries accumulating context ≈ 250k in / 25k out ≈ $0.75. Decomposed: analysis on low (60k/3k ≈ $0.008), implementation on mid (50k/15k ≈ $0.25), tests on mid with fresh context and independent of the implementer's reasoning (40k/8k ≈ $0.16), dup ≈ $0.02, return+integration ≈ $0.05 → ≈ $0.49, about 35% lower. Analysis is read-only and isolated (relieves root context); tests/impl have separable file scopes. Quality risk not higher. Tiers: analysis = low (inventory-type), implementation = mid, tests = mid; no high unless a security/architecture decision appears. Decision: decompose, mixed tiers. Match. (If the estimate came out <30% lower, the gate would flip to no-decompose; the rule is the test, not the label.)

## 5. Two tightly coupled modules, same files
Gate: overlapping files and shared state → default NO; parallel writing in one checkout is forbidden by the subagent rules; separate worktrees would create merge conflicts that raise integration and rework cost; rule 3 fails (no independent subtasks). Decision: no decompose; one mid model in root (high only if a design decision is actually architectural). Match.

## 6. NaN slips past a None guard (typical bug fix)
Input: in a reporting script, a guard checks only `is None`; a `nan` value falls into the `else` branch, the comparison evaluates `False`, and the report shows a wrong status instead of `n/a`. Proposed fix: `math.isnan` in the guard, or normalize NaN to `None` upstream, and make the `n/a` branch print its reason. Root cause is known.

Gate: one guard of ~20 lines in one file, plus one regression test; ~15k in / 3k out ≈ $0.04 on mid. Subtasks are sequential (decide approach → edit → test) and share the same function, so rule 3 (independent subtasks) fails. Decomposition would add context_dup + integration larger than the work itself. Quality risk: splitting adds handoff loss on a correctness fix. Decision: no decompose.
Model: one mid model (Sonnet / Terra-medium). Not high: this is a reporting script, not a security or architecture path, and the root cause is already known. Not low: it is a correctness change that needs the surrounding logic read. The open design decision (isnan vs. normalize) belongs to the user and is settled in root before editing; it is not delegable. Expected: no decompose, mid, root. Match.

## 7. Plan-mode prompt (two-phase gate)
Input: "Analyze error X in this project and create an implementation plan" (plan mode). Phase 1: analysis has an unclear root cause and is sequential, so no decomposition; one model (mid, high only after structured root-cause analysis and only if the cause is architectural/security). Phase 2: when the plan has steps, e.g. (a) inventory affected call sites, (b) fix in one module, (c) regression tests, (d) independent review, the gate runs on those steps: (a) low, read-only, no worktree; (b)+(c) share the same function, so root, no delegation; (d) mid, independent, read-only. The plan contains the Execution Allocation table, the TCO comparison and briefs for (a) and (d) before the implementation steps. A plan that lists steps without an allocation table, or a run that gates only the original prompt, fails this scenario. Expected: allocation embedded in the plan, mixed tiers, partial delegation. Walkthrough against Step 1b: match.

## 8. Model choice from measured data
Input: a recurring headless routine currently runs on the high tier; the project keeps session records. Gate: do not estimate from the price table. Read per-run records first: calls, cache-read tokens, output tokens per run on each model that has run the job. In one measured case the high tier needed about 3x the calls of the mid tier (179 vs 61) and cost about 2.9x per run. The cost is calls × context; a lower per-token price alone says nothing. Check quality with a structural checklist on the produced artifacts before switching, and switch as a time-boxed pilot with a one-line rollback. On a subscription, report tokens first and notional USD second. Decision: switch only if quality checks pass; otherwise report the numbers and let the user choose. Match.

## 9. Borderline decomposition
Input: three independent web-research blocks. Estimate: direct ≈ 3.1, decomposed ≈ 2.5 notional USD (about 20% lower); the subagents keep bulky page content out of the root context. Gate: rule 1 misses the 30% rule of thumb, rule 3 holds through context isolation, quality risk is not higher. Decision: borderline, so show both numbers and the isolation benefit and ask; do not decide silently. If estimated cost exceeds `confirm_threshold`, the same question applies. Match.
