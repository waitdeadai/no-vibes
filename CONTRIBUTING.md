# Contributing to no-vibes

Small project, low-ceremony.

## Filing an issue

If the hook fires when it shouldn't (false positive) or fails to fire when it should (false negative), please open an issue with:

1. The minimal JSON payload that reproduces the behavior. Like the fixtures in [`RECEIPTS.md`](RECEIPTS.md) — short, isolated, copy-pasteable.
2. The exit code and stderr you observed.
3. The exit code and stderr you expected.
4. Your `jq` version (`jq --version`).

A reproducer that someone else can run in 30 seconds is worth ten paragraphs of description.

## Filing a PR

PRs welcome for:

- new false-success patterns the regex misses
- new positive-closeout vocabulary that should trigger the hook
- new evidence patterns that should let a closeout pass
- bug fixes in the bash itself
- portability fixes (BSD/GNU `grep`, macOS `bash 3.2`, etc.)

Before opening:

- Add or update a fixture in the same shape as the existing tests in `RECEIPTS.md` Part 1.
- Run the new fixture against your branch and paste the exit code + stderr in the PR body.
- Keep the diff small. The hook is intentionally one file.

## Out of scope

- Running real test suites / builds / linters from the hook itself. That's a different mechanism (see [claudefa.st's Force Task Completion pattern](https://claudefa.st/blog/tools/hooks/stop-hook-task-enforcement)) and `no-vibes` is meant to *complement* it, not replace it.
- New hook events beyond what Claude Code natively supports.
- A whole governance framework. If you want one, the hook is Apache-2.0 — fork it.

## Releasing

Maintainer cuts a `vX.Y.Z` tag and writes release notes via `gh release create`. SemVer-ish: bump minor for new patterns, patch for fixes, major if the JSON payload contract changes in a non-backward-compatible way.
