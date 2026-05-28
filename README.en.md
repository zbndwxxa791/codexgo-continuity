<p align="center">
  <img src="assets/codexgo-logo.png" alt="codexgo-continuity logo" width="132">
</p>

<h1 align="center">codexgo-continuity</h1>

<p align="center">
  <strong>A steadier Codex recovery skill.</strong><br>
  When a thread disappears, it uses the project boundary and latest state to bring the task back. (｀・ω・´)
</p>

<p align="center">
  <a href="README.md">中文</a>
  ·
  <a href="ORIGIN_AND_CHANGES.md">Origin and changes</a>
  ·
  <a href="https://github.com/JY0xLU/codexgo">Reference project</a>
</p>

<p align="center">
  <img alt="Python" src="https://img.shields.io/badge/python-3.10%2B-3776AB?style=flat-square&logo=python&logoColor=white">
  <img alt="Dependencies" src="https://img.shields.io/badge/deps-zero-10B981?style=flat-square">
  <img alt="Local only" src="https://img.shields.io/badge/privacy-local--only-0F766E?style=flat-square">
  <a href="LICENSE"><img alt="License" src="https://img.shields.io/badge/license-Apache--2.0-blue?style=flat-square"></a>
</p>

## What It Is

`codexgo-continuity` solves one specific, mildly cursed problem: you already explained the task, Codex started working, and the thread vanished because of compaction failure, crash, or lost context. Open a fresh session, type `codexgo-continuity`, and it reads local Codex state plus rollout records to recover the most likely continuation request and latest progress.

No cloud memory. No magic. No database writes. Just a tiny recovery buddy rummaging through local history and saying, "hey, we were doing this." (｡•̀ᴗ-)✧

## Origin and Changes

This project referenced [`JY0xLU/codexgo`](https://github.com/JY0xLU/codexgo) during implementation. It keeps the Apache-2.0 license and the same local-only, read-only, zero-dependency design.

This version focuses on recovery quality issues found in local use: project bleed when multiple conversations are open, stale task recovery after later work is already done, overly old fallback, excessive default context, and loss of the newest state when there are no obvious marker words.

See [ORIGIN_AND_CHANGES.md](ORIGIN_AND_CHANGES.md) for the full breakdown.

## The Problem

To keep long sessions manageable, Codex occasionally compacts background context. If that request is interrupted mid-stream, the failure can look like this:

```text
Error running remote compact task: stream disconnected before completion:
error sending request for url (https://chatgpt.com/backend-api/codex/responses/compact)
```

At that point the original thread may no longer be usable, but the conversation trail, workspace path, and task clues are still stored in local Codex records. `codexgo-continuity` reads those records, reconstructs the request you should continue from, and tries to avoid project bleed, stale tasks, and duplicated work.

## The Fix

After a compact interruption:

1. Stay in the same project workspace. Do not spend time reviving the broken thread.
2. Open a fresh Codex session.
3. Type `codexgo-continuity`.

```text
codexgo-continuity
```

It extracts the last actionable request and the newer thread state from the previous conversation so the new thread can continue in the right place. No manual recall, no re-explaining the requirement, no rebuilding context from scratch.

## How It Works

1. Opens the local Codex SQLite state database.
2. Matches a recent historical thread for the current workspace.
3. Reads the rollout timeline, restores message order, and identifies the real request.
4. Filters confirmation or placeholder messages such as `ok`, `continue`, and `继续`.
5. Walks upward for referenced context when the user says things like "that plan" or "do what we discussed".

## Highlights

| Feature | What it means |
| --- | --- |
| Tiny on purpose | One Python script, one skill file, standard library only |
| Quiet and safe | Local-only, read-only, no uploads, no database writes |
| Has a little memory | Skips low-signal replies and expands context for "that approach", "the previous plan", and similar references |
| Script-friendly | Supports both plain text and JSON output |
| Easy to poke at | Compact logic that is easy to read, study, and modify |

## Install

`codexgo-continuity` is a Codex skill, not a pip package. Put this repository in Codex's `skills/codexgo-continuity` directory, then restart Codex.

### Codex App

If you use the Codex desktop app, install into your active `CODEX_HOME`. When it is not set, the Windows app often uses `D:\CodexData\.codex`; regular CLI setups usually use `~/.codex`.

Windows PowerShell:

```powershell
$CodexHome = if ($env:CODEX_HOME) { $env:CODEX_HOME } elseif (Test-Path "D:\CodexData\.codex") { "D:\CodexData\.codex" } else { "$HOME\.codex" }
New-Item -ItemType Directory -Force "$CodexHome\skills" | Out-Null
git clone <your-repo-url> "$CodexHome\skills\codexgo-continuity"
```

Then fully restart the Codex app so it can rescan local skills.

### Codex CLI: macOS / Linux

```bash
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
git clone <your-repo-url> "${CODEX_HOME:-$HOME/.codex}/skills/codexgo-continuity"
```

### Codex CLI: Windows PowerShell

```powershell
$CodexHome = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { "$HOME\.codex" }
New-Item -ItemType Directory -Force "$CodexHome\skills" | Out-Null
git clone <your-repo-url> "$CodexHome\skills\codexgo-continuity"
```

Restart the Codex app, or open a fresh Codex CLI session, then type:

```text
codexgo-continuity
```

If you just crawled out of a broken thread, this is usually the first thing to say. No need to explain the whole task twice.

## Usage Flow

<p align="center">
  <img src="assets/codexgo-usage.png" alt="codexgo-continuity recovery flow" width="100%">
</p>

## What It Handles

Its job is to turn "human continuation noise" back into something Codex can actually continue. Small tool, useful little shovel. (ง •̀_•́)ง

| Last message before interruption | How codexgo-continuity resolves it |
| --- | --- |
| A real task | Returns that task directly |
| `continue` / `go on` / `继续` | Walks back to the previous real request |
| `ok` / `yes` / `好的` | Recovers the assistant plan you agreed to |
| `补充：...` | Merges the supplement with the previous context |
| "that approach" / "the previous plan" / "continue in that direction" | Expands supporting context upward automatically |
| Assistant has any newer state after the older request, even without clear marker words | Emits `latest_thread_state` and `latest_thread_state_confidence`; uses `completed_resolved_request: null` when completion is unclear |
| Selection or comparison prompts | Emits `decision_basis_message` as the decision basis |
| Automation use cases | Emits JSON for downstream tools |

JSON output also includes `context_expanded_upward`, which tells callers whether codexgo-continuity had to walk further upward to resolve an ambiguous reference. `matched_cwd` is the search target that matched; `thread_cwd` is the recovered thread's actual workspace.

## Example Output

Plain text output:

```text
Recovered Codex request
- matched search target: /path/to/project
- thread workspace: /path/to/project
- source: user_message
- needs more context: False
- context expanded upward: False

Resolved request:
Finish the README polish and run the tests.
```

JSON output for automation:

```json
{
  "status": "ok",
  "current_cwd": "/path/to/current/project",
  "scope_used": "repo",
  "matched_cwd": "/path/to/current/project",
  "thread_cwd": "/path/to/current/project",
  "resolved_request": "Finish the README polish and run the tests.",
  "resolved_source": "user_message",
  "newer_thread_state_available": false,
  "completed_resolved_request": false,
  "latest_thread_state_confidence": "none",
  "decision_basis_message": "",
  "context_expanded_upward": false
}
```

## Safety and Privacy

- Reads only local `~/.codex/state_*.sqlite` and rollout JSONL files.
- Does not upload conversations, call the network, or write to the Codex database.
- Does not modify your project files unless you pass its output into another automation.
- Returns an error when recovery fails instead of fabricating a request.

In plain words: it is not a cloud memory service. It is a local bookmark with a flashlight.

## CLI

```bash
python scripts/codexgo.py --cwd . --format text
python scripts/codexgo.py --cwd . --format json
```

Common options:

```text
--cwd <path>         Workspace path. Defaults to the current directory.
--codex-home <path>  Codex data directory. Defaults to CODEX_HOME or ~/.codex.
--scope <mode>       Search mode: auto, exact, repo, or tree. Defaults to auto.
--skip-current       Skip the current thread. Enabled by default.
--recent <n>         Number of recent user messages to include. Defaults to 3.
--lookback <n>       Nearby timeline entries to include as context. Defaults to 6.
--format <fmt>       text or json. Defaults to text.
```

`auto` searches only the current directory and the Git repository root. It does not silently fall back to parent/child tree matching. Use `--scope tree` explicitly when you want that wider fallback, because it can recover an unrelated thread from a nearby parent directory.

## Requirements

- Python 3.10+
- Codex local state in `~/.codex`
- No third-party Python packages

## Limitations

- Codex local state must exist; there is nothing to recover without history.
- If Codex changes its SQLite schema or rollout format, the parser may need an update.
- Ambiguous-reference recovery is rule-based, not LLM semantic reasoning.
- Recovery works best from the same workspace or Git repository.

It follows clues, but it does not pretend to be psychic. If it cannot recover the task, it says so.

## Development

Run tests:

```bash
python -m pytest tests/test_codexgo.py -p no:cacheprovider
```

## License

Apache-2.0
