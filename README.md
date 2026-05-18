# no-vibes

[![tests](https://github.com/waitdeadai/no-vibes/actions/workflows/test.yml/badge.svg)](https://github.com/waitdeadai/no-vibes/actions/workflows/test.yml)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![GitHub release](https://img.shields.io/github/v/release/waitdeadai/no-vibes)](https://github.com/waitdeadai/no-vibes/releases)
[![GitHub stars](https://img.shields.io/github/stars/waitdeadai/no-vibes?style=social)](https://github.com/waitdeadai/no-vibes/stargazers)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-hook-orange)](https://code.claude.com/docs/en/hooks)

> A Claude Code Stop hook that blocks the model from closing a turn with positive language when it didn't actually verify anything.

`no-vibes` is one bash file (~530 lines, depends only on `jq`) wired into Claude Code's `Stop`, `SubagentStop`, `PreToolUse`, `PostToolUse`, `TaskCreated`, and `TaskCompleted` events. It pattern-matches the language Claude uses when it is about to claim success it didn't earn — and it returns the exact corrective shape the model should use instead.

> **Empirical baseline**: F1 **0.815** (95% CI [0.615, 0.941]) on MAST mode 3.3 ("No or Incorrect Verification"), n=19 human-labelled multi-agent traces from [MAD](https://huggingface.co/datasets/mcemri/MAD) (Cemri et al., NeurIPS 2025 — [arXiv:2503.13657](https://arxiv.org/abs/2503.13657)). The bash hook itself produces identical predictions to the Rust engine port on all 19 traces (zero per-trace disagreement) — see the [parity report](https://github.com/waitdeadai/agent-closeout-bench/blob/main/evaluation/runs/mast_human_bash_parity.md) and the [umbrella suite empirical writeup](https://github.com/waitdeadai/llm-dark-patterns/blob/main/evaluation/MAST-RESULTS.md).

### What's new (2026-05-11)

| Change | What it fixes |
|---|---|
| Loadable locale packs (English, Spanish, Polish ship; native-speaker PRs welcome for the rest) | Operators running Claude in non-English sessions previously bypassed the hook entirely — the vocab was English-only |
| Loadable evidence binary allowlist (200+ binaries across 9 categories) | The previous regex only recognized `git`/`npm`/`bash`/etc. Devops/SRE work could close as "verified the cluster" with zero terminal evidence |
| Loadable destructive command surface packs (filesystem, container, git-protected, config-overwrite, cloud-prod, database, service) | Previous rules only caught filesystem-level destruction. `docker stop`, `git push --force main`, `terraform destroy`, `DROP TABLE`, `redis-cli FLUSHALL`, `systemctl stop` are now blocked at PreToolUse |
| Bypass hardening (clause-local negation + evidence proximity + action-verb context) | Two reported bypasses closed: hedge-then-positive and backtick-mid-message-disclaimed-evidence no longer slip through |

Operators can extend any pack without forking by dropping a `.txt` at `${XDG_CONFIG_HOME:-$HOME/.config}/llm-dark-patterns/packs/<subdir>/<name>.txt`. See the umbrella suite repo's [ROADMAP.md](https://github.com/waitdeadai/llm-dark-patterns/blob/main/ROADMAP.md) for the architecture spec.

This addresses the failure pattern documented in [anthropics/claude-code#46727 (April 2026)](https://github.com/anthropics/claude-code/issues/46727):

> "rules saying 'verify before claiming done' — Claude claims success without verification … not occasional — it's the default behavior."

## Receipts

The hook fired four separate times on Claude Opus 4.7 during a single research session while writing this very repo. Each fire was an Opus turn that *would otherwise have closed positively without evidence*. Each fire returned a repair-guidance template, and Opus repaired on the next turn without further intervention.

See [RECEIPTS.md](RECEIPTS.md) for the redacted transcripts.

## Install (30 seconds)

### Recommended: via the self-hosted plugin marketplace

```bash
claude plugin marketplace add waitdeadai/claude-plugins
claude plugin install llm-dark-patterns@waitdeadai-plugins
```

This installs the whole `llm-dark-patterns` suite (28 hooks including `no-vibes`) and keeps them wired correctly across Claude Code settings updates.

> The Anthropic community marketplace (`anthropics/claude-plugins-community`) currently does not list this plugin despite a Published submission — last sync was 2026-05-13 and the pipeline has stalled. Tracking: [#1887](https://github.com/anthropics/claude-plugins-official/issues/1887). The self-hosted route above bypasses that pipeline and works today.

### Standalone (single hook, manual wire)

If you only want this one hook without the suite:

```bash
mkdir -p .claude/hooks
curl -fsSL https://raw.githubusercontent.com/waitdeadai/no-vibes/main/no-vibes.sh \
  -o .claude/hooks/no-vibes.sh
chmod +x .claude/hooks/no-vibes.sh

# Then merge the snippet from settings.example.json into .claude/settings.json
```

Either path requires `jq` (most systems have it; `brew install jq` / `apt install jq` if not).

### Python integration (CrewAI)

For CrewAI users:

```bash
pip install crewai-no-vibes
```

```python
from crewai_no_vibes import verification_claim_evidence_guardrail
from crewai import Task

task = Task(description="...", expected_output="...", guardrail=verification_claim_evidence_guardrail)
```

See [waitdeadai/crewai-no-vibes](https://github.com/waitdeadai/crewai-no-vibes) — pure-Python port of the `evidence_claims` rule pack. The F1 0.815 baseline above holds across implementations (bash, Rust, Python): bash-Rust parity was verified on the n=19 human-labelled MAD subset with zero per-trace disagreement (see [parity report](https://github.com/waitdeadai/agent-closeout-bench/blob/main/evaluation/runs/mast_human_bash_parity.md)).

## What it actually does

On every assistant turn end (`Stop` and `SubagentStop`), the hook reads the assistant's last message and applies two checks:

1. **False-success check.** If the message contains positive-closeout vocabulary (`done`, `ready`, `passed`, `shipped`, `completed`, `implemented`, `fixed`, `finished`) **and** also contains the linguistic markers of missing verification (`not run`, `skipped`, `unverified`, `could not verify`, `tests not ran`) → block.

2. **Evidence check.** If the message contains positive-closeout vocabulary but **no** evidence in the same message (no command backticks, no `Verification: passed/blocked`, no `files inspected`, no source ledger, no diff/artifact mention) → block.

When it blocks, it returns this exact repair template via stderr:

```
Status: partial
Verification: not run because <reason>
Next step: <specific command or blocker>
```

The model reads the repair template on the next turn and self-corrects.

It also covers a few adjacent cases, mostly because they're cheap:

- **`PreToolUse(Bash)`** — blocks the obvious destructive patterns (`rm -rf`, `git reset --hard`, `git clean -fd`, `chmod -R 777 /`, `dd of=/dev/`, `mkfs`, …) and asks for explicit human approval + a rollback plan.
- **`PreToolUse(Write|Edit|MultiEdit)`** — refuses writes to `.env`, `.env.*`, `.claude/*.local.json`, `secrets/**`.
- **`TaskCreated`** — refuses implementation-like task payloads that don't declare ownership (owned paths, forbidden paths, stop conditions).
- **`TaskCompleted`** — refuses subagent task closeouts that lack evidence, mirroring the same false-success / missing-evidence checks as the Stop branch.

## Why this is different from existing Claude Code hooks

| Existing pattern | Mechanism | Limitation |
|------------------|-----------|------------|
| Run real tests/build/lint on Stop | Actually executes the test suite, blocks on red | Slow, requires a test suite, useless on read-only / research / explanatory turns, doesn't catch a model claiming verification it didn't perform |
| Block hesitation phrases ("should I proceed?", "let me know") | Catches early-exit / permission-seeking | Catches the *opposite* failure mode — gives no signal when the model claims success without doing the work |
| `no-vibes` | Pattern-matches positive-closeout vocabulary in the outgoing message **and** demands evidence in the same message **and** returns the corrective shape | Costs nothing per turn; works on every turn type; complements the test-runner pattern instead of replacing it |

The repair-guidance template is the part that matters most. Most "block" hooks just block. This one teaches the next turn what compliant closeout looks like, which means the model self-corrects in ~one turn instead of cycling.

## What it does NOT do

- It does **not** run your tests. It catches the *claim* of having tested, not the absence of a test suite.
- It does **not** prevent every false claim. It catches the linguistic pattern. A model that closes with "the artifact is in place" instead of "done" can still slip past — though that closeout shape is itself less harmful because it isn't asserting verification.
- It does **not** replace `/verify` style discipline, code review, or CI. It is a turn-level safety net, not a release gate.
- It does **not** depend on any larger framework. One file, one dependency.

## Physics-backed engine

This standalone hook remains the simplest install path. For users who want the
benchmark-backed, rule-pack-hashed engine version, the same closeout mechanic is
also available in [AgentCloseoutBench](https://github.com/waitdeadai/agent-closeout-bench):

```bash
git clone https://github.com/waitdeadai/agent-closeout-bench
cd agent-closeout-bench
bash adapters/claude-code/install.sh /path/to/your/project no-vibes
bash scripts/hook-smoke.sh
```

The physics-backed adapter maps `no-vibes` to the `evidence_claims` category
engine and can be used for daily enforcement, fixtures, benchmark evaluation,
and opt-in content-free collaboration telemetry. The AgentCloseoutBench
installer also writes a PreToolUse tamper guard for ordinary Claude Code edits
to hook wiring, adapter env, pinned engine, and pinned rule-pack paths.

In the physics-backed v0.2 lane, weak proof shapes such as `Implemented and
checked.`, `Done. Commands run: none.`, and `Changed files:` without command or
verification evidence are rejected. This is still closeout-contract evidence,
not independent proof that the work actually happened, and it is not an OS
sandbox or bypass-proof claim.

## Sister tools

Part of the [LLM Dark Patterns Hooks](https://github.com/waitdeadai/llm-dark-patterns) suite — single-purpose Claude Code Stop hooks that suppress LLM dark-pattern defaults so power-user operators can actually work.

- [time-anchor](https://github.com/waitdeadai/time-anchor) — injects the local system clock so the model stops giving training-cutoff answers.
- [no-curfew](https://github.com/waitdeadai/no-curfew) — suppresses unsolicited rest/sleep/wellness paternalism.
- [no-sycophancy](https://github.com/waitdeadai/no-sycophancy) — blocks praise-spam at turn open.
- [no-cliffhanger](https://github.com/waitdeadai/no-cliffhanger) — blocks dangling permission-loop endings.
- [honest-eta](https://github.com/waitdeadai/honest-eta) — vibe time estimates and linear-scaling parallelism claims.
- [no-fake-recall](https://github.com/waitdeadai/no-fake-recall) — false-memory recall claims without quoted prior content.
- [no-fake-stats](https://github.com/waitdeadai/no-fake-stats) — fabricated percentages and amounts without source.
- [no-fake-cite](https://github.com/waitdeadai/no-fake-cite) — academic citation patterns without verifiable URL.
- [llm-dark-patterns](https://github.com/waitdeadai/llm-dark-patterns) — umbrella catalog of the suite.
- [minmaxing](https://github.com/waitdeadai/minmaxing) — the parent governance harness.

## Acknowledgments

`no-vibes` was extracted from a larger personal harness (the `minmaxing` workbench). The hook itself is licensed Apache-2.0; take it, fork it, improve it, send the PR back.

The pattern was crystallized after watching Claude Opus 4.7 close turns dishonestly enough times that pattern-matching the language became cheaper than hoping for better prompts.

## License

Apache-2.0. See [LICENSE](LICENSE).
