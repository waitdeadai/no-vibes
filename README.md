# no-vibes

[![tests](https://github.com/waitdeadai/no-vibes/actions/workflows/test.yml/badge.svg)](https://github.com/waitdeadai/no-vibes/actions/workflows/test.yml)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![GitHub release](https://img.shields.io/github/v/release/waitdeadai/no-vibes)](https://github.com/waitdeadai/no-vibes/releases)
[![GitHub stars](https://img.shields.io/github/stars/waitdeadai/no-vibes?style=social)](https://github.com/waitdeadai/no-vibes/stargazers)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-hook-orange)](https://code.claude.com/docs/en/hooks)

> A Claude Code Stop hook that blocks the model from closing a turn with positive language when it didn't actually verify anything.

`no-vibes` is one bash file (~370 lines, depends only on `jq`) wired into Claude Code's `Stop`, `SubagentStop`, `PreToolUse`, `PostToolUse`, `TaskCreated`, and `TaskCompleted` events. It pattern-matches the language Claude uses when it is about to claim success it didn't earn — and it returns the exact corrective shape the model should use instead.

This addresses the failure pattern documented in [anthropics/claude-code#46727 (April 2026)](https://github.com/anthropics/claude-code/issues/46727):

> "rules saying 'verify before claiming done' — Claude claims success without verification … not occasional — it's the default behavior."

## Receipts

The hook fired four separate times on Claude Opus 4.7 during a single research session while writing this very repo. Each fire was an Opus turn that *would otherwise have closed positively without evidence*. Each fire returned a repair-guidance template, and Opus repaired on the next turn without further intervention.

See [RECEIPTS.md](RECEIPTS.md) for the redacted transcripts.

## Install (30 seconds)

```bash
# 1. drop the hook into your project
mkdir -p .claude/hooks
curl -fsSL https://raw.githubusercontent.com/waitdeadai/no-vibes/main/no-vibes.sh \
  -o .claude/hooks/no-vibes.sh
chmod +x .claude/hooks/no-vibes.sh

# 2. wire it into your Claude Code settings
#    (merge the snippet from settings.example.json into .claude/settings.json)
```

Requires `jq` (most systems have it; `brew install jq` / `apt install jq` if not).

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

## Acknowledgments

`no-vibes` was extracted from a larger personal harness (the `minmaxing` workbench). The hook itself is licensed Apache-2.0; take it, fork it, improve it, send the PR back.

The pattern was crystallized after watching Claude Opus 4.7 close turns dishonestly enough times that pattern-matching the language became cheaper than hoping for better prompts.

## License

Apache-2.0. See [LICENSE](LICENSE).
