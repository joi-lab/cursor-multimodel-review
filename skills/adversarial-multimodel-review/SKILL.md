---
name: adversarial-multimodel-review
description: Runs a deep, full-context adversarial multi-model review of previous agent work. Always optimizes for correctness over speed, cost, and brevity. Use when the user wants independent verification before commit, merge, or deployment, when checking for bugs, regressions, or scope creep, or when getting Gemini, GPT, or Opus critic opinions.
license: MIT
---

# Adversarial Multimodel Review

Use this skill after an AI agent has planned or implemented work and the user wants an independent readiness check before commit, merge, or deployment.

The goal is not to generate more opinions. The goal is to reconstruct the real task, inspect the actual code and evidence, find disagreements, and decide what must happen next.

This skill is intentionally expensive by default. Independent critics matter most for high-stakes work, so this workflow always optimizes for correctness over speed, cost, and brevity.

## Architectural Constraint You Must Respect

Cursor subagents start with a **clean context**. They cannot see the parent chat history, prior tool outputs, system instructions, project rules, the user's earlier corrections, or the parent agent's TodoWrite. The Cursor docs are explicit about this:

> "Subagents start with a clean context. The parent agent includes relevant information in the prompt since subagents don't have access to prior conversation history."

This is the single biggest failure mode of multi-model review. A critic that did not receive the user's actual intent, accepted plan, and explicitly-rejected approaches will:

- Re-recommend approaches the user already turned down.
- Treat "the user wanted less code" as scope-creep removal.
- Find infinite edge cases in any code, because every diff has edges.
- Drift away from the user's real success criteria toward generic engineering critique.

This skill solves that constraint by **writing the parent context to files on disk** that critics MUST read before producing findings.

## Trigger Phrases

Apply this skill when the user asks things like:

- "Review the previous agent's work."
- "Can we commit this?"
- "Can we deploy this?"
- "Run a multimodel review."
- "Check for bugs, regressions, or scope creep."
- "Get Gemini/GPT/Opus to review this independently."

## Default Review Behavior

This skill always runs a deep, full-context adversarial review. There is no light mode. Always:

- Use the full available context budget. Do not optimize for token savings, speed, or brevity.
- Use the maximum available reasoning effort, thinking depth, output budget, and context window that the current Cursor model/runtime makes available.
- If Max Mode is enabled, assume the review should use the model's maximum supported context window. If you cannot verify Max Mode, model effort, or context limits from the environment, state that as a confidence limitation.
- Inspect the complete relevant diff, not only the implementation summary.
- Read complete relevant files instead of small snippets, especially prompts, rules, configs, schemas, migrations, and shared interfaces.
- Search for all readers, writers, and callers when a shared file, public API, data format, tool, prompt, or config changes.
- Review test output, runtime logs, deployment state, and known stale evidence with timestamps.
- Reconstruct the full task context: original request, follow-up corrections, plan, implementation summary, conversation history, and project rules.
- Do not stop at the first issue. Build the most complete picture possible before giving a verdict.
- Do not broaden into unrelated code or expose secrets. Deep review means complete relevant evidence, not random repository traversal.
- If evidence is too large or inaccessible, say exactly what could not be reviewed and how that limits confidence.
- In short: spend the review budget. The user opted into this workflow because missing a bug costs more than extra tokens.

## Workflow

### Step 1. Convergence Guard (Run First)

Before assembling anything, check `.adversarial-review/round.txt`. If it exists and shows a round count ≥ 3 on a substantively similar diff, **halt** and ask the user:

> "This is round N of adversarial review on a similar diff. Each round tends to find new edge cases in the previous round's fixes — that is the structure of any open-ended critic search, not new bugs. Confirm one of:
> (a) accept residual tradeoffs and commit now,
> (b) freeze scope now and commit; iterate on remaining items in a follow-up commit,
> (c) continue one more round on a specific named concern (state it)."

Do not auto-launch critics if the round count would exceed 2 without explicit user confirmation. If `round.txt` does not exist, treat this as round 1. Increment and write the round number after every full review pass.

### Step 2. Write the Evidence Packet to Files

The evidence packet lives on disk in `.adversarial-review/`. Critics will read these files via Read/Grep. Do not rely on the initial Task prompt to carry context — keep that prompt short and point critics at files.

Create the directory and write the following files. Each file is mandatory unless explicitly marked optional. Use the exact filenames so critics know where to look.

#### `.adversarial-review/USER_INTENT.md`

Verbatim user quotes that define this task. **No paraphrasing.** Include:

- The original user request (full text or the relevant section).
- Later corrections, in chronological order, each prefixed with a timestamp or turn marker.
- Explicit success criteria the user stated.
- Explicit scope boundaries the user stated.

Template:

```markdown
## Original Request
> [verbatim user text]

## Corrections And Constraints (chronological)
- turn N: > [verbatim user text]
- turn M: > [verbatim user text]

## Stated Success Criteria
- [exact phrasing]

## Stated Scope Boundaries
- IN scope: [...]
- OUT of scope: [...]
```

If you cannot quote the user verbatim because the chat has been compacted, say so explicitly at the top of the file. Critics will treat that as a confidence limit.

#### `.adversarial-review/FORBIDDEN_FINDINGS.md`

Approaches the user has explicitly rejected, with quotes. Critics MUST drop any finding that re-proposes anything in this file.

Template:

```markdown
## Rejected Approaches
- Approach: [name]
  - Quote: > [verbatim user text rejecting it]
  - Do NOT recommend this in findings.

## Out-Of-Scope Items For This Review
- [item]: user said this is a separate follow-up.

## Anti-Patterns The User Called Out
- [pattern]: > [verbatim user text]
```

If empty, write `(none — user has not explicitly rejected any approach in this conversation)`. An empty file is OK; a missing file is not.

#### `.adversarial-review/PLAN_ACCEPTED.md`

Pointer to the plan the user accepted (if any). Critics use this to detect "implementation does less/more than plan".

Template:

```markdown
## Accepted Plan
- Source: [.cursor/plans/<plan>.plan.md OR inline quote]
- Acceptance evidence: > [user quote accepting the plan]

## Plan Items Status
- [done] item 1
- [done] item 2
- [deferred-to-next-commit] item 3
- [explicitly-dropped] item 4 (user quote: > [...])
```

If no plan was accepted, write `(no formal plan — see USER_INTENT.md for direct requirements)`.

#### `.adversarial-review/DIFF.patch`

Raw output of `git diff <base>..HEAD` (or `git diff --cached` if reviewing a staged-only change). Use the actual base ref the user is shipping against, not a guess.

#### `.adversarial-review/FILES_TO_READ_WHOLE.txt`

One file path per line. These are files critics MUST read end-to-end (not snippets) before reviewing. Include:

- Project governance docs: `BIBLE.md`, `docs/ARCHITECTURE.md`, `docs/DEVELOPMENT.md`, `docs/CHECKLISTS.md` if they exist.
- The accepted plan file if it lives in the repo.
- Shared interface files touched by the diff (`*/contracts/*`, `*/api/*`, schema files).
- Any prompt, rule, or config file the diff touches.

#### `.adversarial-review/TESTS.txt`

Exact test command and exact tail of output. Format:

```text
## Command
python -m pytest -q

## Result
[exit code, last ~40 lines]
```

If tests were not run, state why and which test command would be appropriate. Critics will treat "tests not run" as a confidence limit, not as automatic failure.

#### `.adversarial-review/DECIDED_TRADEOFFS.md`

Tradeoffs the user has already accepted in this scope. Critics drop findings that re-litigate these.

Template:

```markdown
## Accepted Tradeoffs
- Tradeoff: [name]
  - Cost: [what we lose]
  - Justification: [why this is OK]
  - User confirmation: > [user quote or "implicit via plan acceptance"]
```

#### `.adversarial-review/RUNTIME.md` (optional but recommended)

Runtime / deployment state evidence with timestamps:

- Server restarted: yes/no, at HH:MM
- Latest logs reviewed: path + range
- Migrations applied: yes/no
- Credentials available: yes/no
- Known stale evidence: list

### Step 3. Optional Pre-Critic Mapping With `explore`

If the diff touches shared interfaces, use Cursor's built-in `explore` subagent to find every reader/writer/caller. Save its output as `.adversarial-review/RELATED_SURFACES.md`. Critics will use this to verify "cross-file consistency" instead of taking that claim on faith.

### Step 4. Launch Critic Subagents In Parallel

Launch in a single message with multiple Task tool calls. The named critic is not, by itself, proof that Cursor used a different model: Cursor's documented subagent default is `model: inherit`, and configured model requests can fall back when a model is unavailable, blocked by team policy, or requires Max Mode.

Required critic routes (one strong model per distinct provider):

- `gpt-critic` for rigorous implementation and regression review — an OpenAI model.
- `gemini-critic` for broad-context alternative reasoning — a Google Gemini model.
- `opus-critic` for deep architectural and intent review — an Anthropic Claude model.
- `inherit-critic` only when three distinct provider models cannot be obtained (see the fallback in the routing guard below).

#### Resolve critic models at runtime (primary path)

Model version strings rotate over time, and Cursor does **not** error on an unrecognized `model` slug — it silently clones the parent model, which collapses every critic onto one model and destroys the multi-model premise. So do not rely on memorized or hardcoded version numbers. Instead, at review time:

1. Inspect the model identifiers your current environment actually exposes **right now** — the `model` values your Task tool accepts, or the entries in Cursor's model picker. This live list is your source of truth, and the closest thing Cursor has to a durable `gpt-latest`-style alias.
2. Pick one strong model from each of three **distinct** providers:
   - one OpenAI model (a current GPT) → `gpt-critic`
   - one Google Gemini model (a current Gemini Pro) → `gemini-critic`
   - one Anthropic Claude model (a current Opus) → `opus-critic`
3. Pass each resolved model as the explicit per-call `model` argument when you launch that critic. This explicit per-call path is the reliable one; a critic's frontmatter slug is only a fallback default that can silently clone the parent once it goes stale.

The static frontmatter slugs in `agents/*.md` are kept current as fallback defaults, but treat them as examples, not as the authoritative version — re-resolve against the live model list each run.

Model-routing guard:

1. Primary: launch each critic with an explicit, runtime-resolved per-call `model` from a distinct provider (OpenAI / Google / Anthropic). If your tool does not expose a per-call `model` field, fall back to the critic's frontmatter slug, knowing it may be stale.
2. Before synthesizing, inspect the visible tool-call metadata, subagent transcript metadata, or any available Cursor UI/runtime evidence for the model each critic actually used.
3. Record both the intended critic route (resolved model + provider) and the observed model evidence in the final `Model Diversity Check` section.
4. If model evidence is unavailable, say so explicitly and classify the run as `model diversity unverified`.
5. If you cannot obtain three distinct providers — the picker is limited, the plan restricts subagents to `fast`, or region/Max Mode blocks a model — or if two or more named critics are observed using the same parent model, do NOT call the result true multi-model review. Either relaunch with verified distinct models, use 2-3 `inherit-critic` invocations with different perspectives as an explicit same-model fallback, or return `INSUFFICIENT EVIDENCE` for a request that specifically required multi-model review.

Each critic prompt should be **short**. Do not duplicate the evidence packet inline. The prompt should:

1. Point at `.adversarial-review/` and require the critic to read its mandatory pre-flight files.
2. Name the target decision (commit / merge / deploy / continue iteration).
3. Name any known limitations (stale logs, runtime not restarted, missing credentials, or model diversity unverified).

Example critic prompt:

```text
Review the diff at .adversarial-review/DIFF.patch.

MANDATORY pre-flight (return INSUFFICIENT EVIDENCE if missing):
- .adversarial-review/USER_INTENT.md
- .adversarial-review/FORBIDDEN_FINDINGS.md
- .adversarial-review/PLAN_ACCEPTED.md
- .adversarial-review/TESTS.txt

Read whole files listed in .adversarial-review/FILES_TO_READ_WHOLE.txt.

Target decision: commit.
Known limitation: runtime has not been restarted yet.

Apply your normal checklist. Drop any finding that re-proposes an approach in FORBIDDEN_FINDINGS.md or asks to re-litigate a tradeoff in DECIDED_TRADEOFFS.md.
```

### Step 5. Synthesize The Reviews

- Treat every critic finding as untrusted until checked against code or logs.
- Cross-check every "hard blocker" finding against `USER_INTENT.md` and `FORBIDDEN_FINDINGS.md`. If a finding re-proposes a rejected approach, drop it from blockers and note the disagreement.
- Merge duplicate findings.
- Highlight disagreements and decide which side has better evidence.
- Separate `Hard Blockers` (would block teammate's commit at code review) from `Soft Suggestions` (nice-to-have, not commit-blocking).
- Classify the review run as one of: `verified multi-model`, `model diversity unverified`, or `same-model fallback`; do not imply more model independence than the evidence supports.
- Increment `.adversarial-review/round.txt`.

### Step 6. Return The Final Readiness Decision

- `BLOCK`: serious correctness, data, security, or deploy risk.
- `FIX FIRST`: likely safe after targeted fixes.
- `SAFE TO COMMIT`: code is ready to commit, with any minor caveats.
- `SAFE TO DEPLOY AFTER RUNTIME CHECK`: code can be committed, but deployment needs restart, smoke test, or live verification.
- `INSUFFICIENT EVIDENCE`: the reviewer lacked enough task, diff, test, or runtime evidence to make a reliable call.

## Critic Checklist (orientation; per-critic file has authoritative version)

Each critic answers:

- Did the implementation satisfy the user's actual request in `USER_INTENT.md`, including follow-up corrections?
- Does the plan in `PLAN_ACCEPTED.md` still make sense after reading the final code?
- Does any candidate finding re-propose something in `FORBIDDEN_FINDINGS.md`? (If yes, drop it.)
- Does any candidate finding re-open a tradeoff in `DECIDED_TRADEOFFS.md`? (If yes, drop it.)
- Did the implementation preserve behavior outside the intended scope?
- Are there new bugs, regressions, race conditions, data-loss risks, security issues, or broken invariants?
- Are tests meaningful, passing, and aimed at the risky paths?
- Are docs, prompts, config, migrations, and shared data consumers still consistent?
- Did the previous agent ignore runtime evidence, stale logs, permissions, deploy state, or environment constraints?
- Did the implementation add unnecessary abstraction, scope creep, compatibility shims, or unrelated churn?
- **Asymmetric scope check:** if the user wanted the codebase to shrink, does the implementation actually shrink it? Did the previous agent re-add abstraction the user asked to remove?
- What must be fixed before commit, merge, or deploy? Does this need a fix, or is it nice-to-have?

## Output Format

Use this structure:

```markdown
## Verdict
`SAFE TO COMMIT` | `SAFE TO DEPLOY AFTER RUNTIME CHECK` | `FIX FIRST` | `BLOCK` | `INSUFFICIENT EVIDENCE`

A short verdict summary explaining the decision. The detail belongs below; this is only the headline. Brevity here does not imply a shallow review.

## Honest Self-Check
- Would I personally `git commit` this code as-is? [yes/no, one sentence]
- Would I block a teammate's commit at code review on this? [yes/no, one sentence]

## Hard Blockers
Findings ONLY if the answer to "would I block a teammate's commit" is yes.

### Blocker 1: short title
- Severity: `blocker` | `high`
- Status: `verified` | `plausible`
- Location: file, symbol, command, or runtime surface
- Evidence: what was observed
- Failure mode: how this could break
- Required fix: what must change
- Verification: how to prove the fix
- User-intent check: this is NOT a re-proposal of anything in FORBIDDEN_FINDINGS.md.

If none: write "No hard blockers. Commit-ready relative to USER_INTENT.md."

## Soft Suggestions
Nice-to-have improvements that DO NOT block this commit. Each item is one line. No severity field — these are explicitly not blockers.

## Scope And Intent Drift Check
- Does this commit do MORE than `USER_INTENT.md` asked? [list, or "no"]
- Does this commit do LESS than `USER_INTENT.md` asked? [list, or "no"]
- Did this commit re-introduce something `FORBIDDEN_FINDINGS.md` rejected? [list, or "no"]

## Model Diversity Check
- Intended critic routes: [critic name -> configured/requested model]
- Observed model evidence: [actual model evidence from tool metadata/transcripts/UI, or "unavailable"]
- Classification: `verified multi-model` | `model diversity unverified` | `same-model fallback`
- If not verified multi-model: explain whether the verdict still stands as same-model multi-perspective review, or return `INSUFFICIENT EVIDENCE` when true multi-model review was required.

## Model Disagreements
What the critics disagreed about and which interpretation is best supported by `USER_INTENT.md` + the actual code.

## Checks Performed
- Pre-flight files read: [list]
- Whole files read: [list]
- Diff inspected: yes/no
- Tests considered: yes/no
- Logs/runtime evidence considered: yes/no

## Follow-Up Prompt
Copy-paste prompt for the implementation agent to address the accepted findings.
```

## Follow-Up Prompt Template

Use this when handing review output back to the implementation agent:

```text
Here is an independent adversarial review of your previous work. Do not assume it is correct. Re-open the code, diff, tests, logs, and user requirements. Verify each finding against the actual project state. Fix only the Hard Blockers you can confirm. If a finding is wrong, explain why with evidence. If there are multiple reasonable fixes, ask before changing behavior. Do not act on Soft Suggestions unless the user explicitly asks for them.
```

## Rules Of Engagement

- Prefer evidence over confidence.
- `SAFE TO COMMIT` requires at minimum: `USER_INTENT.md`, `DIFF.patch`, and either `TESTS.txt` or a clear reason tests are unnecessary, all present and non-empty.
- Use `INSUFFICIENT EVIDENCE` when any mandatory pre-flight file is missing or empty, or when the user specifically requested multi-model review and model diversity cannot be verified or was observed to collapse to one parent model.
- Do not punish the previous agent for harmless style differences.
- Do not expand scope unless the current change created a real risk.
- Do not approve deployment based only on static code review when runtime restart, migrations, credentials, or live checks are required.
- If critics disagree, investigate the disputed point directly before making the final call.
- Adding code is not automatically better. If a finding asks the implementer to ADD a parser branch, abstraction, fallback, or sanitizer, justify it with a concrete reproducible failure mode. Otherwise drop the finding.
- A long Soft Suggestions list is fine. A long Hard Blockers list is not. Hard Blockers are claims about whether this can ship — be conservative about counting them.

## Why Files-On-Disk

This skill writes context to `.adversarial-review/` because Cursor subagents cannot read parent chat. Putting context in files:

- Survives chat compaction. Round-3 critics see the same `USER_INTENT.md` as round-1 critics.
- Closes the "critic invented the user's intent" failure mode. Either the file is there or `INSUFFICIENT EVIDENCE` fires.
- Lets the user audit what each critic was actually told. Open the directory, read the files.
- Lets the user version the review state. Add `.adversarial-review/` to git, or to `.gitignore` if it should stay local — the choice is the user's, not the skill's.

If you do not want `.adversarial-review/` checked into the repo, add it to `.gitignore` once.
