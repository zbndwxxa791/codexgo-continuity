# Origin and Changes

## New Project Name

This repository is published as `codexgo-continuity`.

The name keeps the relationship to `codexgo` visible while making the new purpose explicit: safer Codex session continuity across fresh sessions after compaction, crash, or context loss.

## Reference Project

- Referenced project: `codexgo`
- Reference repository: https://github.com/JY0xLU/codexgo
- Reference license: Apache-2.0

This repository was developed after reviewing the above project and keeps the same zero-dependency, local-state-only design.

## Problems Addressed

The referenced recovery model was useful for finding a previous actionable request, but local usage exposed several practical failure modes:

- Project bleed: when multiple Codex conversations were open, recovery could land in another nearby project or an internal approval/review thread.
- Stale continuation: the recovered request could point to an older task even when the previous thread had already completed later work.
- Repeated work: after a compact/crash break, the new thread could restart work that had already been done.
- Overly old fallback: searching too far back could recover a stale task rather than the immediately previous useful project thread.
- Context pressure: expanding too much old conversation consumed context in the new session.

## Improvements in This Fork

- Uses the current project boundary by default, based on the Git root when available and the current working directory otherwise.
- Does not silently use broad parent/child tree matching unless `--scope tree` is requested.
- Filters Codex internal approval/review threads before selecting the previous project thread.
- Limits default recovery to recent candidates and does not fall back to older threads unless `--fallback-older` is requested.
- Preserves newer assistant state after the recovered request as `latest_thread_state`.
- Adds `latest_thread_state_confidence` to distinguish clear completion, clear follow-up work, and low-confidence newer state.
- Uses `completed_resolved_request` to avoid treating completed older requests as still undone.
- Keeps default output minimal while exposing richer context on demand with `--detail context` or `--detail full`.
- Adds regression tests for project isolation, stale-thread avoidance, internal-thread filtering, latest-state preservation, and low-confidence continuation.

## Compatibility Notes

The script remains `scripts/codexgo.py` for compatibility with the existing project layout. The skill name and recommended repository/install directory are `codexgo-continuity`.
