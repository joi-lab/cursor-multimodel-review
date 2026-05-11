---
name: mm-review
description: Slash shortcut for adversarial multimodel review of the previous agent's work.
---

Use the adversarial-multimodel-review skill to review the previous agent's work.

Follow the skill's full workflow:

1. Check `.adversarial-review/round.txt` — if round count is already ≥ 2, halt and ask the user before launching more critics.
2. Write the evidence packet to `.adversarial-review/` (USER_INTENT.md with verbatim user quotes, FORBIDDEN_FINDINGS.md with rejected approaches, PLAN_ACCEPTED.md, DIFF.patch, FILES_TO_READ_WHOLE.txt, TESTS.txt, DECIDED_TRADEOFFS.md).
3. Launch `gemini-critic`, `gpt-critic`, `opus-critic` in parallel. Use `inherit-critic` if the named models are blocked by plan / Max Mode / region.
4. Each critic prompt is short: it points at `.adversarial-review/` and names the target decision. Do not duplicate evidence inline.
5. Synthesize: drop critic findings that re-propose anything in FORBIDDEN_FINDINGS.md or re-open DECIDED_TRADEOFFS.md.
6. Return Hard Blockers vs Soft Suggestions separately. Verdict must be `SAFE TO COMMIT` / `SAFE TO DEPLOY AFTER RUNTIME CHECK` / `FIX FIRST` / `BLOCK` / `INSUFFICIENT EVIDENCE`.

Use the full available context. Do not optimize for token savings, speed, or a short answer. Verify from source, not from the previous agent's summary.
