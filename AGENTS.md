# AGENTS.md — `dekuraan/winit` fork (branch `wam-0.30.13`)

This is a fork of [rust-windowing/winit](https://github.com/rust-windowing/winit) used by [wam](https://github.com/dekuraan/wam) as a git submodule at `vendor/winit`. The wam repo's `[patch.crates-io]` redirects every transitive `winit` dependency to this submodule, so any change here ships to every wam crate that touches `bevy_winit`.

## Why the fork exists

The branch `wam-0.30.13` is upstream tag `v0.30.13` plus **one** patch:

- `src/platform_impl/web/event_loop/runner.rs::handle_event` — replaces `self.0.runner.borrow_mut()` with `try_borrow_mut()`. Upstream's unconditional borrow panics when a user-supplied event handler dispatches a second event synchronously. Bevy 0.18 + heavy SSE-driven chunk dispatch on the WASM single-thread event loop reliably triggers this. On contention we push the second event onto the existing pending queue (`self.0.events`), mirroring the `RunnerEnum::Pending` branch — the outer dispatch's drain loop picks it up. Same observable behavior, no panic.

The reproducer is the wam WASM client with LOD 1 enabled and the dispatcher running at full speed; the panic was deterministic before the fix.

## Branch hygiene

- `master` tracks upstream `rust-windowing/winit` master.
- `wam-0.30.13` is the only branch wam consumes. It is **never** rebased onto upstream master; it sits on top of the released `v0.30.13` tag.
- New patches go on top of this branch as additional commits, not as squashed force-pushes — the wam parent repo pins this branch by SHA via the submodule, so a force-push would break older wam commits.
- Bumping winit later (e.g. to a hypothetical `v0.30.14`): create a new branch `wam-0.30.14` off the new tag, replay the patches via `git cherry-pick` from `wam-0.30.13`, push, then point the wam submodule at the new branch.

## Don't edit this file from inside wam unless the change is generic

If you find yourself wanting to add a wam-specific note here, it probably belongs in `wam/AGENTS.md` or a wam-side memory entry instead. This file is for facts about *this fork's* relationship to upstream.

## Verifying the patch is still load-bearing

Before considering whether to drop the fork:

```bash
git -C vendor/winit diff v0.30.13 -- src/platform_impl/web/event_loop/runner.rs
```

If that diff is empty (e.g. because the patch was upstreamed), this fork is no longer needed and the wam side can move to `winit = "<version>"` from crates.io and remove the submodule.
