# tco

`tco` is an agent skill for Claude Code and Codex. It decides whether a job is worth splitting across subagents and assigns each subtask the cheapest model that solves it correctly in one pass. It optimizes total cost of ownership (TCO) of the whole job, not the price of a single task: one correct run beats several cheap retries, so there are no "try cheap, escalate on failure" loops.

## How the gate works

1. Estimate tokens for the whole job (from measured usage records when they exist) and compare TCO(direct, best single model) with TCO(decomposed), including calls × context, context duplication, return flow, integration and rework risk. On a subscription, tokens are the primary figure and USD is notional.
2. Decompose only if the decomposed run is clearly cheaper (rule of thumb: at least 30% lower), quality risk is not higher, and at least 2 subtasks are independent (or one needs isolated context or independent review). Borderline cases (15-30% lower with a real isolation benefit) are shown to the user instead of decided silently.
3. Otherwise stay in the root agent with one well-chosen model; if decomposing, output an Execution Allocation table and a brief per subagent.

## Installation

Claude Code: copy or symlink the `tco/` folder into `~/.claude/skills/`.

```bash
git clone https://github.com/machtelik81/tco-skill.git
ln -s "$PWD/tco-skill/tco" ~/.claude/skills/tco
```

Codex: symlink the same folder into `~/.codex/skills/`.

```bash
ln -s "$PWD/tco-skill/tco" ~/.codex/skills/tco
```

The repository name `tco-skill` is a suggestion; use your actual repository name. Restart open sessions so the new skill is listed.

## Usage

Explicit only; the skill never activates on its own.

- Claude Code: `/tco`
- Codex: `$tco`

Invoke it together with the task, or before a plan-mode request.

## Example outputs

Illustrative, not measured.

**Do not decompose**

```
Gate: single typo fix in one file, ~3k in / 0.2k out, direct ≈ $0.01.
Decomposition would add context duplication and integration costing more than the work; no independent subtasks.
Decision: do not decompose. One mid-tier model, root agent.
```

**Decompose**

```
Task: rename a term across 40 files (disjoint, verifiable by grep).
| Subtask         | Model / reasoning | Execution | Worktree | Result       |
|-----------------|-------------------|-----------|----------|--------------|
| files 1-10      | haiku             | parallel  | yes      | file list    |
| files 11-20     | haiku             | parallel  | yes      | file list    |
| files 21-30     | haiku             | parallel  | yes      | file list    |
| files 31-40     | haiku             | parallel  | yes      | file list    |
TCO direct (mid) ≈ $0.28, decomposed ≈ $0.09 (about 68% lower).
Assumptions: 80k in / 12k out; context_dup 5k per worker; integration at mid tier.
```

## Limitations

- The Codex `spawn_agent` model override (`model`, `reasoning_effort`, `fork_turns`) is derived from binary strings of Codex CLI 0.153.4 and was never run live [uncertain]. If the call rejects the parameters, the skill outputs the plan only.
- Overhead constants and GPT-5.6 reasoning-effort multipliers are UNVERIFIED heuristics.
- Which model version the Claude aliases `haiku`, `sonnet` and `opus` resolve to is [uncertain].
- Prices are a snapshot and go stale; the skill warns when `last_updated` is older than 90 days.
- The scenarios (9) were walked through against the rules, not exercised in live runs.
- Cache-read and batch prices were added from a partial read of the provider pages; re-verify before large jobs.

## Maintaining prices.md

Before each release, re-verify every price against the provider pages linked in `tco/prices.md` and bump `last_updated`. Keep prices that cannot be verified marked UNVERIFIED. `confirm_threshold` is an editable default. There is no automated price fetching by design.

## License

MIT, see `LICENSE`.
