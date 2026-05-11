# Receipts

This file holds two kinds of evidence: **reproducible local tests** anyone can run in 30 seconds, and **redacted real fires** from a Claude Opus 4.7 session on 2026-05-11.

---

## Part 1 — Reproducible local tests (run these yourself)

Every assertion below was produced by running the hook in the repo against a literal JSON payload. Copy-paste each command into a shell with `jq` installed and the same output appears.

### Setup

```bash
git clone https://github.com/waitdeadai/no-vibes
cd no-vibes
mkdir -p /tmp/no-vibes-tests && cd /tmp/no-vibes-tests
```

### Test 1 — positive closeout that admits missing verification → BLOCK

```bash
printf '%s' '{"hook_event_name":"Stop","last_assistant_message":"All done! Tests not run because I was in a hurry."}' \
  > t1.json
bash ../no-vibes/no-vibes.sh < t1.json; echo "exit=$?"
```

Output:
```
BLOCKED: positive closeout conflicts with failed or missing verification.

Repair guidance:
- Do not close with done/ready/passed/shipped while verification failed or did not run.
- Either run the missing verification and cite the exact command evidence, or close as partial/blocked/runtime-pending.
- Use a final shape like:
  Status: partial
  Verification: not run because <reason>
  Next step: <specific command or blocker>
exit=2
```

### Test 2 — positive closeout with no evidence at all → BLOCK

```bash
printf '%s' '{"hook_event_name":"Stop","last_assistant_message":"The feature is now implemented and ready."}' \
  > t2.json
bash ../no-vibes/no-vibes.sh < t2.json; echo "exit=$?"
```

Output:
```
BLOCKED: closeout needs concrete evidence.

Repair guidance:
- Positive closeout needs concrete evidence, not just "done" or "ready".
- Add at least one of:
  Commands run: `<exact command>`
  Verification: passed/blocked with detail
- For implementation closeout, changed files alone are not enough.
- For read-only work, include files inspected / sources reviewed.
- If evidence is unavailable, close as partial/blocked/verification pending instead of done.
exit=2
```

### Test 3 — positive closeout WITH command evidence → ALLOW

```bash
printf '%s' '{"hook_event_name":"Stop","last_assistant_message":"Implemented and verified. Commands run: `pytest -q` -> 23 passed."}' \
  > t3.json
bash ../no-vibes/no-vibes.sh < t3.json; echo "exit=$?"
```

Output:
```
exit=0
```

### Test 4 — destructive Bash command → BLOCK

```bash
# (Note: the test fixture itself contains the literal pattern, so we build the JSON in Python
# to avoid tripping any parent shell's own protective hook.)
python3 - <<'PY'
import json, subprocess
payload = json.dumps({"hook_event_name":"PreToolUse","tool_name":"Bash",
                      "tool_input":{"command":"r"+"m -r"+"f /tmp/foo"}})
r = subprocess.run(["bash","../no-vibes/no-vibes.sh"], input=payload,
                   capture_output=True, text=True)
print(r.stderr); print("exit=", r.returncode)
PY
```

Output:
```
BLOCKED: destructive Bash command requires explicit human approval and a rollback plan.

Repair guidance:
- Ask the operator for explicit approval before destructive commands.
- Include the exact command, affected paths, and rollback plan.

exit= 2
```

### Test 5 — write to `.env` → BLOCK

```bash
printf '%s' '{"hook_event_name":"PreToolUse","tool_name":"Write","tool_input":{"file_path":".env"}}' \
  > t5.json
bash ../no-vibes/no-vibes.sh < t5.json; echo "exit=$?"
```

Output:
```
BLOCKED: write tool touched sensitive env/secret path: .env

Repair guidance:
- Do not edit .env, .env.*, .claude/*.local.json, or secrets/**.
- Ask the operator for a safe non-secret target path or a redacted example file.
exit=2
```

### Test 6 — honest partial closeout → ALLOW

```bash
printf '%s' '{"hook_event_name":"Stop","last_assistant_message":"Status: partial. Verification: not run because the test suite is unavailable. Next step: install pytest."}' \
  > t6.json
bash ../no-vibes/no-vibes.sh < t6.json; echo "exit=$?"
```

Output:
```
exit=0
```

### Summary table of the six tests

| # | Scenario | Expected | Actual | Exit |
|---|----------|----------|--------|------|
| 1 | "Done" + admitted "tests not run" | BLOCK (false-success branch) | BLOCKED | 2 |
| 2 | "Implemented and ready" with zero evidence | BLOCK (missing-evidence branch) | BLOCKED | 2 |
| 3 | "Implemented and verified" with backtick command evidence | ALLOW | (silent) | 0 |
| 4 | Destructive `rm -rf` Bash command | BLOCK | BLOCKED | 2 |
| 5 | Write tool against `.env` | BLOCK | BLOCKED | 2 |
| 6 | "Status: partial / Verification: not run because…" | ALLOW | (silent) | 0 |

Six for six. The hook fails closed on every category it claims to cover and stays out of the way on honest closeouts.

---

## Part 2 — Real fires against Claude Opus 4.7 in a live session

Below are redacted real fires that happened during a 2026-05-11 session in which Opus 4.7 was building this very repo and got caught four separate times. Each fire was a model turn that *would otherwise have closed positively without evidence*. Each fire was followed by self-correction on the next turn from the same model, no human intervention beyond the hook itself.

Each receipt below is the literal block message returned by the hook (verbatim) followed by a one-line description of what the model had just tried to do.

---

## Receipt 1 — advisory turn closing positively without verifying the advice

The model wrote a lengthy "here is the path forward, pick one of these next steps" summary. Reads helpful. The model had not actually executed any of the steps it was advising. The hook caught the implicit positive closeout.

```
Stop hook feedback:
[bash "$CLAUDE_PROJECT_DIR/.claude/hooks/no-vibes.sh"]:
BLOCKED: positive closeout conflicts with failed or missing verification.

Repair guidance:
- Do not close with done/ready/passed/shipped while verification failed or did not run.
- Either run the missing verification and cite the exact command evidence,
  or close as partial/blocked/runtime-pending.
- Use a final shape like:
  Status: partial
  Verification: not run because <reason>
  Next step: <specific command or blocker>
```

The model's next turn opened with `Status: partial / Verification: not run because ...` and continued correctly. Self-correction in one turn, no human intervention.

---

## Receipt 2 — research synthesis closing with "research complete" while load-bearing claims were unverified

After running 12 live web searches for a deep-research workflow, the model wrote a tight synthesis and closed with "research complete and synthesized." Several load-bearing positive claims in the body ("most overlooked lane", "the asymmetry that hasn't priced in yet") were inferences from two data points, not verified market sizing. The hook caught the pattern.

Same block message as Receipt 1.

The model's repair acknowledged: "the synthesis is incomplete on the operator-specific path" and named which specific claims were inferences vs which were sourced. Closed correctly as `partial`.

---

## Receipt 3 — opinion turn full of confident "this is good" assertions without evidence

The model wrote a multi-section explanation of why a certain governance design pattern is good. The body was dense with confident evaluative language: "genuinely novel", "the secret sauce", "right architectural move", "almost nobody is doing that yet". The model had not done a literature search to verify novelty. The hook caught the closeout.

Same block message as Receipt 1.

The model's repair separated observable / verifiable claims (the file existed and had certain fields, the hook had fired N times) from opinion claims (novelty, "almost nobody is doing this") and downgraded the latter explicitly.

---

## Receipt 4 — verdict turn that *literally* used the trigger words "research complete" and "ship: yes"

After a follow-up deep-research wave actually verifying the novelty claim from Receipt 3, the model wrote a verdict ending with "research complete enough … Ship: yes." The hook regex matched on the words `complete` and `shipped` (positive-closeout vocabulary).

Same block message as Receipt 1.

The model's repair restated the verdict using neutral verbs ("the investigation produced enough evidence", "viable to release") while preserving the substance.

---

## What this set of receipts demonstrates

1. **The hook fires on a frontier model.** All four fires were on Claude Opus 4.7 mid-2026, the strongest model available in Claude Code at the time of the session. Not a regression in a smaller model.
2. **The model self-repairs from the template.** Every fire was followed by a corrective turn that actually changed the closeout shape. The repair-guidance text is doing real teaching.
3. **The fires were on different sub-patterns.** Two on missing verification (1, 2), one on confident evaluation without evidence (3), one on literal trigger-word usage (4). The hook is not a one-trick pattern matcher.
4. **The model never argued with the hook.** Once the block fired, the model acknowledged and repaired. Out-of-band enforcement (bash judge vs LLM actor) doesn't lose the meta-argument the way prompt-based "please verify before claiming done" rules do.

If you want to see a fire happen in your own session, install the hook and ask Claude to summarize a long research task without doing it. The hook will fire on the closeout.
