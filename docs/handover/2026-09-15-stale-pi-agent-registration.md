# Handover: stale Pi agent registration

**Date:** 2026-09-15

## Completed issue evidence

- Local issue filed: [“Pi loses agent_session after suspend and resume in a Herdr pane”](https://github.com/herdrdev/herdr/issues/4127), against Herdr 0.7.5.
- Follow-up confirmation posted on upstream [issue #1647](https://github.com/herdrdev/herdr/issues/1647#issuecomment-5681662909), the pre-existing “transient foreground takeover wipes a pane's persisted_agent_session” thread. It adds the Pi/macOS/Herdr 0.9.0 stable data point, confirming the defect on the captain's real daily 0.9.0 install and that #4127 is not stale.
- Both cases were independently, first-hand reproduced in isolated lab sessions, with no product changes and no LLM cost. The minimal trigger is a running Pi agent losing the pane foreground (Ctrl+Z or a nested interactive shell) and later returning as the same process. Herdr's foreground-process monitor misclassifies this as an agent exit and clears authoritative `agent_session` / hook authority. Pi returns as a detection-only agent (`agent_session: null`, `screen_detection_skipped: false`, `fallback_reason: "default_known_agent_idle_fallback"`) even though the same process remains alive throughout.

## Open integration proposal

The root mechanism is confirmed present through Herdr 0.9.0 stable (tag `b99002ac`). `src/pane.rs` derives `process_exited` from `pending_foreground_shell_clear && agent.is_some() && !foreground_shell_exit_reported`, which is true when Pi merely leaves the pane foreground, not only on a real exit. In `src/terminal/state.rs`, `set_detected_state_with_screen_signals_at` handles that signal by clearing `hook_authority` and `persisted_agent_session`. The Pi integration (`src/integration/assets/pi/herdr-agent-state.ts`) only re-reports session identity on `session_start` / `agent_start`, so suspend/resume never re-asserts it and the loss is permanent for that process.

The proposed fix direction (not implemented or authorized) is to decouple “agent left the pane foreground” from “agent process exited” in that derivation, retaining live hook authority / `agent_session` (or re-hydrating it from the still-live socket connection) while the same process remains alive.

This is an open proposal only. Implementation is out of scope until a maintainer authorizes it; this handover does not touch `src/`.

## Constraints

- Contributor account `pablontiv` is not listed in `.github/MAINTAINERS` or `.github/APPROVED_CONTRIBUTORS`. The available path today is filing/commenting on issues, not landing implementation PRs directly upstream; any future fix PR needs maintainer sign-off.
- `CONTRIBUTING.md` requires personal, first-hand reproduction before filing. Both linked issues satisfy that requirement; the proposed fix direction does not and must not be implemented without explicit authorization.
- This handover PR carries no product change and no `AGENTS.md`/`CLAUDE.md` change. It is a draft against the captain's fork only, never upstream, and must not be merged here.

## Next steps

- Await maintainer response on upstream [#1647](https://github.com/herdrdev/herdr/issues/1647) and local [#4127](https://github.com/herdrdev/herdr/issues/4127).
- Decide at the firstmate/captain level whether the nested-shell Ctrl+Z/fg reproduction should be filed as, or folded into, a separate upstream issue. It is a broader-trigger variant of the same mechanism; that decision is outside this task.
- If a maintainer authorizes implementation, decouple the `process_exited` derivation in `src/pane.rs` / `src/terminal/state.rs` and add regression coverage for suspend/resume and nested-shell displacement.
