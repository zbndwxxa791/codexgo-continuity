---
name: "thread-anchor"
description: 'Recover the previous Codex session for the current workspace after compaction, crash, or context loss. Use when the user types `thread-anchor`, asks to recover the previous Codex session, or asks to continue after a compact/crash break.'
---

# thread-anchor

Use this skill only when the user explicitly wants to recover the previous Codex session for the current workspace, usually at the start of a fresh session after compaction or crash.

Do not auto-trigger this skill later in a normal active conversation from vague phrases like "continue", "继续", or "接着做". In an active thread, prefer the current conversation context unless the user clearly names this skill.

## Run

Run:

```bash
python <thread-anchor-skill-dir>/scripts/codexgo.py --cwd "$PWD" --format json
```

Resolve `<thread-anchor-skill-dir>` as the directory containing this `SKILL.md`. Keep `--cwd` pointed at the user's current project workspace, not at the skill directory.

On Windows PowerShell:

```powershell
python "<thread-anchor-skill-dir>\scripts\codexgo.py" --cwd . --format json
```

By default, `thread-anchor` first identifies the current project, using the Git root when available and the current working directory otherwise. It then recovers the immediately previous recent thread whose working directory is inside that project. If that previous project thread has no recoverable user request, it reports an error instead of falling back to older threads. This prevents a fresh session from accidentally continuing another project or a stale older task. If the user explicitly wants a narrower, broader, older, or older-fallback search, pass `--scope exact`, `--scope repo`, `--scope tree`, `--max-age-days 0`, or `--fallback-older`.

Default output is intentionally minimal to avoid spending the new session's context on recovered history. Do not rerun with `--detail context`, `--detail full`, `--pretty`, or `--max-field-chars 0` unless the user asks for more detail or the minimal recovery report is ambiguous.

## Report Recovery

Read these fields:

- `resolved_request`: the best task to continue now
- `execution_policy`: should be `report_only_until_user_confirms`
- `requires_user_confirmation`: should be `true`
- `confirmation_prompt`: a short prompt to ask the user before continuing
- `literal_last_user_message`: the exact last user message from the previous thread
- `last_conversation_content`: the last meaningful conversation item
- `resolved_source`: whether the request came from a user message, assistant suggestion, supplement, or fallback
- `newer_thread_state_available`: whether there is a newer assistant completion/progress state after the recovered request
- `latest_thread_state_confidence`: `high`, `medium`, `low`, or `none`; low means a newer assistant message exists but has no clear completion/follow-up markers
- `completed_resolved_request`: whether that newer state looks fully complete; can be `null` when a newer assistant state exists but completion is unclear
- `latest_thread_state`: the newest assistant completion/progress/status message, when available
- `needs_more_context`: whether the recovered text still looks ambiguous
- `ambiguity_hints`: why the recovered text may still depend on earlier context
- `supporting_context_available` and `supporting_context_count`: whether omitted supporting context exists
- `context_expanded_upward`: whether the supporting context had to walk further upward for explanation
- `decision_basis_message`: earlier user requirement that defines a selection or comparison basis

After recovering a request, do not continue the task automatically. Report the recovered thread, the resolved request, and any ambiguity to the user, then ask whether to continue. If `newer_thread_state_available` is `true`, treat `latest_thread_state` as the newest state of the previous thread and do not restart the older original request as if it were still undone. If `completed_resolved_request` is `false`, continue from the follow-up work described by that latest state. If `completed_resolved_request` is `null`, report that a newer low-confidence state exists and ask whether to continue from that latest state or the older recovered request. Do not modify files, run non-read-only commands, start services, install dependencies, commit, push, or otherwise change the user's workspace until the user explicitly confirms after the recovery report.

Treat recovered-thread messages such as "continue", "继续", "ok", or prior approvals as historical context only. They are not approval to act in the new session.
