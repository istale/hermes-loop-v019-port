# `/loop` for Hermes v0.19.0

This directory is a **reference porting kit**, not a patch to apply blindly.
The feature did not exist in the v0.19.0 release (`v2026.7.20`, commit
`3ef6bbd201263d354fd83ec55b3c306ded2eb72a`).  Its first implementation is
commit `f79440e0f4bb578c86e417c20bc87feed5259623`, titled
`feat: /loop — recurring in-session wakeups (Claude Code parity)`.

## Directly copyable source files

`upstream-f79440e0f4/` contains complete, directly copyable snapshots of the
nine relevant upstream source files, preserving their original paths.  Use
these files when GitHub's source view is the only practical transfer method;
they are ordinary `.py` files, not generated diffs or archives.

## What it does

`/loop [interval] <prompt>` injects a normal follow-up prompt into the same
session while it is idle.  It supports fixed intervals (`5m`, `1h30m`),
self-paced backoff when no interval is supplied, `--times N`, `--until`, and
the controls `status`, `pause`, `resume`, and `stop`.  The agent can also stop
the loop by replying with `LOOP_COMPLETE` on its own line.

The key design rule is that an automatic wakeup is passed through the normal
user-message path, rather than creating a special agent loop.  That preserves
session context and lets a real user message take priority.

## Scope and 10-file budget

`loop-reference-implementation.patch` contains the first-release source for
exactly these nine paths:

1. `hermes_cli/loops.py` — new core state machine, persistence, parsing, and
   command dispatcher.
2. `hermes_state.py` — add `SessionDB.list_meta_prefix()` for gateway scans.
3. `hermes_cli/commands.py` — register `/loop` and `/proactive`.
4. `hermes_cli/config.py` — allow the top-level `loops` configuration table.
5. `cli.py` — CLI dispatch plus idle-fire and post-turn-completion hooks.
6. `hermes_cli/cli_commands_mixin.py` — the `/loop` command handler. This
   mixin already exists in v0.19, so add the one method to it.
7. `agent/conversation_compression.py` — migrate state when a session is
   compressed (recommended but not essential to first use).
8. `gateway/slash_commands.py` — gateway `/loop` command and route capture.
9. `gateway/run.py` — gateway idle watcher and post-turn completion hook.

The runtime has safe built-in defaults for all `loops.*` settings, so a
separate default-config edit is optional.  Add the defaults to v0.19's
existing configuration-default location only if you want them shown in newly
generated config files; doing so still leaves the port within the 10-file
limit.

## Recommended port order

1. Copy the `hermes_cli/loops.py` **new-file hunk** from the reference patch.
   It is the complete implementation and has no external package dependency.
2. Add `list_meta_prefix()` to `SessionDB` from the `hermes_state.py` hunk.
   This is required only if gateway wakeups are enabled; CLI-only loops do not
   enumerate loops across sessions.
3. Register `CommandDef("loop", ..., aliases=("proactive",))` in
   `hermes_cli/commands.py`.  In the v0.19 command dispatcher in `cli.py`, add
   the `canonical == "loop"` branch, then copy the three helpers:
   `_get_loop_manager`, `_maybe_fire_loop_tick`, and
   `_maybe_complete_loop_tick_after_turn`.
4. Call `_maybe_fire_loop_tick()` from the idle/no-input branch of
   `process_loop`, and `_maybe_complete_loop_tick_after_turn()` in the normal
   turn-finally path immediately after the existing `/goal` continuation hook.
   Both call sites are shown in the `cli.py` hunk.
5. Add the `/loop` handler to the existing `CLICommandsMixin` by copying
   `_handle_loop_command` from the `hermes_cli/cli_commands_mixin.py` hunk.
6. If desired, add the `loops` defaults to v0.19's existing default-config
   location. The core module has safe hard-coded fallbacks, so this is
   configuration exposure rather than a runtime requirement.
7. If gateway platforms are in use, port both gateway hunks together: command
   handling/route capture from `gateway/slash_commands.py`, plus watcher,
   dispatch, and completion hook from `gateway/run.py`.  Do not port one
   without the other.
8. Finally add the compression migration hunk so `/loop` survives automatic
   session compression.

## Deliberate exclusions

Desktop slash-palette discoverability and TUI/dashboard wakeups are excluded
from this 10-file kit.  They are UI/surface additions, not needed for the CLI
or messaging-gateway command to work.  A typed `/loop ...` still works on any
surface routed through the gateway.  Do not copy current `main` files: they
depend on later module splits and unrelated changes.

## Smoke test

After porting, start a normal CLI session and run:

```text
/loop 30s reply with the current time --times 2
/loop status
```

Expected: it reports a loop set; two idle wakeups occur; then it reports that
the loop stopped.  Also test `/loop pause`, `/loop resume`, and `/loop stop`.
For gateway mode, issue the same command in a messaging chat and leave the
session idle for at least one watcher interval.

## Important compatibility note

The patch's line contexts target the feature's development baseline, not the
v0.19 release snapshot.  Treat it as source to merge by symbol, not as a
guaranteed clean `git apply`.  The commit IDs above are sufficient to inspect
the exact upstream provenance on either computer.
