# Cursor Multimodel Review

Deep adversarial review for Cursor. One agent implements; several read-only critic agents review the work from scratch before you commit, merge, or deploy.

This plugin is intentionally expensive by default. It tells critics to use the full available context, read complete relevant files, inspect the full diff, check tests/logs/docs/rules/runtime state, and avoid saving tokens when review quality is at stake.

## Install: Paste This Into Cursor

Open a Cursor chat in any project and paste:

```text
Install this Cursor plugin locally and do nothing else:

https://github.com/joi-lab/cursor-multimodel-review

Steps:
1. Clone or pull the repo into ~/.cursor/plugins/local/adversarial-multimodel-review.
   If that folder already exists, run "git pull" inside it.
2. Do not modify my project files.
3. Do not touch ~/.claude/, ~/.codex/, Cursor settings.json, MCP config,
   installed_plugins.json, or marketplace settings. Cursor auto-discovers
   local plugins from ~/.cursor/plugins/local/, no manual registration.
4. After installing, tell me to run "Developer: Reload Window" in Cursor.
```

Then run **Developer: Reload Window** in Cursor.

Cursor's docs say local plugins are loaded from `~/.cursor/plugins/local` and require either **Developer: Reload Window** or a Cursor restart.

After reload, verify it loaded:

```text
list available skills
```

Confirm `adversarial-multimodel-review` appears. If it does not, the plugin did not load — check `~/.cursor/plugins/local/` and try **Developer: Reload Window** again.

## Run A Review

Primary option:

```text
Use the adversarial-multimodel-review skill to review the previous agent's work. Use the full available context.
```

Slash shortcuts:

- `/mm-review` — command shortcut included in this plugin.
- `/adversarial-multimodel-review` — skill slash form in Cursor 2.4+. If it does not appear, use `/mm-review` or the skill phrase above.

## Manual Install

If you prefer terminal commands:

```bash
mkdir -p ~/.cursor/plugins/local
git clone https://github.com/joi-lab/cursor-multimodel-review \
  ~/.cursor/plugins/local/adversarial-multimodel-review
```

Update later with:

```bash
cd ~/.cursor/plugins/local/adversarial-multimodel-review
git pull
```

Then run **Developer: Reload Window**.

## What It Adds

- Skill: `adversarial-multimodel-review`
- Command: `/mm-review`
- Read-only critics:
  - `gemini-critic` is configured to request `gemini-3.1-pro`
  - `gpt-critic` is configured to request `gpt-5.5-extra-high-fast`
  - `opus-critic` is configured to request `claude-opus-4-8-thinking-max-fast`
  - `inherit-critic` inherits the parent model

## How Context Reaches Critics

Cursor subagents start with a **clean context**. They cannot see the parent chat history, your earlier corrections, project rules, system instructions, or what tools the parent agent already ran. The Cursor docs are explicit:

> "Subagents start with a clean context. The parent agent includes relevant information in the prompt since subagents don't have access to prior conversation history."

This is the single biggest failure mode of multi-model review. A critic that did not receive your actual intent, the accepted plan, and the approaches you explicitly turned down will keep re-recommending those rejected approaches, drift toward generic engineering critique, and find infinite "edge cases" in every diff round.

This plugin solves the constraint by writing your context to disk in `.adversarial-review/` before launching critics:

| File | Purpose |
| --- | --- |
| `USER_INTENT.md` | Verbatim user quotes — original request, corrections, success criteria, scope boundaries. No paraphrasing. |
| `FORBIDDEN_FINDINGS.md` | Approaches you explicitly rejected, with quotes. Critics MUST drop findings that re-propose these. |
| `PLAN_ACCEPTED.md` | The plan you accepted (path or inline), with acceptance evidence and per-item status. |
| `DIFF.patch` | The actual `git diff` critics should review. |
| `FILES_TO_READ_WHOLE.txt` | Files critics MUST read end-to-end (governance docs, the plan, shared interfaces). |
| `TESTS.txt` | Exact test command + output, or an explicit "tests not needed because…". |
| `DECIDED_TRADEOFFS.md` | Tradeoffs already accepted in this scope. Critics drop findings that re-litigate. |
| `RUNTIME.md` (optional) | Server-restart status, log timestamps, migration state, credentials. |
| `round.txt` | Auto-incremented round counter. After round 2, the skill halts and asks for explicit user confirmation before launching more critics — closes the "infinite findings" attractor. |

Every critic has a **mandatory pre-flight check**: it reads these files first, and returns `INSUFFICIENT EVIDENCE` if any mandatory file is missing or empty. This is fail-closed: no critic can silently "guess what the user wanted" anymore.

If you do not want `.adversarial-review/` checked into git, add it to `.gitignore` once.

## Review Verdicts

The skill asks the parent agent to return one of these verdicts:

- `BLOCK`: serious correctness, data, security, or deploy risk.
- `FIX FIRST`: likely safe after targeted fixes.
- `SAFE TO COMMIT`: code is ready to commit, with minor caveats if any.
- `SAFE TO DEPLOY AFTER RUNTIME CHECK`: code can be committed, but deployment still needs restart, smoke test, migration check, or live verification.
- `INSUFFICIENT EVIDENCE`: the reviewer did not get enough task, diff, test, or runtime evidence to make a reliable call.

## Models And Max Mode

The critic agents are configured with these model requests:

```yaml
# agents/gemini-critic.md
model: gemini-3.1-pro

# agents/gpt-critic.md
model: gpt-5.5-extra-high-fast

# agents/opus-critic.md
model: claude-opus-4-8-thinking-max-fast
```

> **Keep these slugs in sync with your Cursor model picker.** The `model` value must match a model identifier Cursor currently exposes in its picker (dotted versions like `gemini-3.1-pro` / `gpt-5.5...`, and the thinking/effort variant for Claude). Model names rotate over time. If the slug is unrecognized, Cursor does **not** error — it silently clones the parent model, so every named critic runs the parent model and the multi-model premise collapses. The slugs above are current as of 2026-05; re-check them against your picker after Cursor or model updates, and confirm diversity via the Model Diversity Check.

**Primary mechanism: resolve models at runtime.** Because slugs rotate and a stale slug silently clones the parent, the skill's primary path is not the static frontmatter above. At review time it tells the parent agent to read the model identifiers your environment exposes *right now* — your Task tool's `model` options or Cursor's model picker — and pick one strong model from each **distinct** provider (a current OpenAI model → `gpt-critic`, a current Google Gemini model → `gemini-critic`, a current Anthropic Claude model → `opus-critic`), passing each as an explicit per-call `model`. This live resolution is the closest thing Cursor has to a durable `gpt-latest`-style alias: there are **no** provider-level "latest" aliases (`inherit`, `fast`, and `auto` are the only stable tokens, and all three collapse provider diversity). The frontmatter slugs are kept current only as fallback defaults for when an explicit per-call `model` is not available.

Cursor's subagent docs say the `model` field defaults to `inherit`, meaning the subagent uses the parent agent's model unless a specific model is configured. The docs also say Cursor can override or fall back from a configured model when the model is blocked by team policy, requires Max Mode, or is unavailable on the current plan. On legacy request-based plans the Task tool's `model` parameter may be restricted to `fast`; in that case use the `inherit-critic` fallback (2-3 perspectives) and classify the run as `model diversity unverified`.

Because of that, a critic name is not proof of model routing. A valid multi-model synthesis must include a **Model Diversity Check**:

1. Intended route: which critic was launched and which model it was configured or explicitly requested to use.
2. Observed evidence: tool-call metadata, subagent transcript metadata, Cursor UI/runtime evidence, or an explicit note that no model evidence was available.
3. Classification: `verified multi-model`, `model diversity unverified`, or `same-model fallback`.

If your transcript shows all named critics using the parent model, treat the result as same-model multi-perspective review, not true multi-model review. If the user specifically asked for multi-model review and model diversity cannot be verified, return `INSUFFICIENT EVIDENCE` or relaunch with verified distinct models.

Cursor's documented subagent frontmatter does **not** include separate `reasoning_effort`, `max_tokens`, or `context_length` fields. This plugin asks critics to use the maximum available effort/context in their prompts. For the biggest context window, turn on **Max Mode** in Cursor before running the review.

Use `inherit-critic` when you want the critic to inherit the exact parent model and Max Mode state, or as the reliable fallback when named models are blocked and the user accepts same-model fallback.

## Known Limitations

- This uses more tokens and takes longer than normal review.
- Subagents start with clean context — see the **How Context Reaches Critics** section above for the `.adversarial-review/` files the parent must write so critics have anything to work with.
- Exact model IDs can change or be unavailable. Always report whether model diversity was verified; if named critics collapse to the parent model, the result is same-model multi-perspective review. Keep `inherit-critic` as the stable fallback when that is intentional.
- The skill slash form `/adversarial-multimodel-review` depends on Cursor 2.4+ skill discovery. Use `/mm-review` if the skill slash entry is not visible.
- Static review cannot prove deployment safety. If runtime was not restarted or logs are stale, require a runtime check.
- Open-ended adversarial critique on a constantly-changing diff has no natural stopping point. The skill caps automatic critic rounds at 2 per scope-stable commit and escalates to the user before round 3.

## Changelog

### 0.3.0

- **Durable model diversity.** The skill now resolves critic models at runtime: it picks one strong model from each distinct provider (OpenAI / Google / Anthropic) from the model list the environment currently exposes, and passes each as an explicit per-call `model`. This survives Cursor model-slug rotation, which previously made every critic silently clone the parent model.
- Updated the static frontmatter slugs to current-valid identifiers (`gpt-5.5-extra-high-fast`, `gemini-3.1-pro`, `claude-opus-4-8-thinking-max-fast`) and re-scoped them as fallback defaults rather than the authoritative route.
- Documented that Cursor has no provider-level "latest" alias, that an unrecognized slug clones the parent instead of erroring, and the request-based-plan `fast` restriction with its `inherit-critic` fallback.

### 0.2.x

- Files-on-disk evidence packet (`.adversarial-review/`), mandatory critic pre-flight, Hard Blockers vs Soft Suggestions, round cap, and the Model Diversity Check.
