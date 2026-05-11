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
  - `gemini-critic` requests `gemini-3-1-pro`
  - `gpt-critic` requests `gpt-5-5`
  - `opus-critic` requests `claude-opus-4-7`
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

The plugin requests these models:

```yaml
model: gemini-3-1-pro
model: gpt-5-5
model: claude-opus-4-7
```

Cursor may fall back if a model is unavailable on your plan, region, team policy, or Max Mode settings.

> **Honest disclaimer.** If your Cursor plan does not include `gemini-3-1-pro`, `gpt-5-5`, and `claude-opus-4-7`, or Max Mode is off, the named critics **silently fall back to the parent model**. In that case all three critics run on the same model, and the multi-model premise dissolves — you get one model with three different system prompts, not three independent perspectives. If you cannot verify your Cursor plan covers all three model IDs in Max Mode, use 2–3 invocations of `inherit-critic` with different review angles instead, or enable Max Mode.

Cursor's documented subagent frontmatter does **not** include separate `reasoning_effort`, `max_tokens`, or `context_length` fields. This plugin asks critics to use the maximum available effort/context in their prompts. For the biggest context window, turn on **Max Mode** in Cursor before running the review.

Use `inherit-critic` when you want the critic to inherit the exact parent model and Max Mode state, or as the reliable fallback when named models are blocked.

## Known Limitations

- This uses more tokens and takes longer than normal review.
- Subagents start with clean context — see the **How Context Reaches Critics** section above for the `.adversarial-review/` files the parent must write so critics have anything to work with.
- Exact model IDs can change or be unavailable. Without Max Mode, named critics silently fall back to the parent model. Keep `inherit-critic` as the stable fallback.
- The skill slash form `/adversarial-multimodel-review` depends on Cursor 2.4+ skill discovery. Use `/mm-review` if the skill slash entry is not visible.
- Static review cannot prove deployment safety. If runtime was not restarted or logs are stale, require a runtime check.
- Open-ended adversarial critique on a constantly-changing diff has no natural stopping point. The skill caps automatic critic rounds at 2 per scope-stable commit and escalates to the user before round 3.
