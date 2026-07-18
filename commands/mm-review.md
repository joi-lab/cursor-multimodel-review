---
name: mm-review
description: Slash shortcut for adversarial multimodel review of the previous agent's work.
---

Use the adversarial-multimodel-review skill to review the previous agent's work.

Follow the skill's full workflow:

1. Check `.adversarial-review/round.txt` — if round count is already ≥ 2, halt and ask the user before launching more critics.
2. Write the evidence packet to `.adversarial-review/` (USER_INTENT.md with verbatim user quotes, FORBIDDEN_FINDINGS.md with rejected approaches, PLAN_ACCEPTED.md, DIFF.patch, FILES_TO_READ_WHOLE.txt, TESTS.txt, DECIDED_TRADEOFFS.md).
3. Resolve critic models at runtime, then launch `gpt-critic`, `gemini-critic`, `opus-critic`, `grok-critic` in parallel. Do not hardcode versions — model slugs rotate, and a stale or restricted frontmatter model silently falls back to a compatible model (typically the parent or Composer). Inspect the model identifiers your Task tool / Cursor model picker exposes right now and pick one strong model from each distinct provider: an OpenAI model → `gpt-critic`, a Google Gemini model → `gemini-critic`, an Anthropic Claude model → `opus-critic`, an xAI Grok model → `grok-critic`. Pass each as the explicit per-call `model` argument (the reliable path). The frontmatter slugs (`gpt-5.6-sol-max[context=1m]` / `gemini-3.5-flash[context=1m]` / `fable-5-max[context=1m]` / `grok-4.5-high`, no 1m bracket for Grok — Cursor caps it at 256k) are current-as-of-2026-07 fallback defaults only. When passing a per-call `model`, request the 1M context variant where the environment exposes it.
4. Do not treat critic names as proof of model diversity. Check available tool-call metadata, subagent transcript metadata, or UI/runtime evidence for the model each critic actually used. Target distinct providers (OpenAI / Google / Anthropic / xAI; three is the minimum for true multi-model). If this cannot be verified, classify the run as `model diversity unverified`.
5. Use `inherit-critic` (2-4 invocations with different perspectives) when at least three distinct providers cannot be obtained — picker limited, plan restricts subagents to `fast`/Composer, region/Max Mode blocks — or when the user explicitly accepts same-model fallback. If named critics are observed using the same parent model, do not call the result true multi-model review.
6. Each critic prompt is short: it points at `.adversarial-review/` and names the target decision. Do not duplicate evidence inline.
7. Synthesize: drop critic findings that re-propose anything in FORBIDDEN_FINDINGS.md or re-open DECIDED_TRADEOFFS.md, and include the final `Model Diversity Check`.
8. Return Hard Blockers vs Soft Suggestions separately. Verdict must be `SAFE TO COMMIT` / `SAFE TO DEPLOY AFTER RUNTIME CHECK` / `FIX FIRST` / `BLOCK` / `INSUFFICIENT EVIDENCE`.

Use the full available context. Do not optimize for token savings, speed, or a short answer. Verify from source, not from the previous agent's summary.
