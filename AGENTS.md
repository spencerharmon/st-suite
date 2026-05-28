# AGENTS.md — st-suite

Guidance for AI coding agents working in this repository. Scope: the suite as a
whole. 

## What this repo is

`st-suite` is a modular, JACK-based live music performance rig written in Rust
(edition 2021). Each submodule is an independent JACK client process; they are
glued together by:

- **JACK** for audio/MIDI I/O and transport (BBT, tempo).
- **st-sync** — a tiny TCP protocol on `127.0.0.1:6142` carrying `u64`
  "next beat frame" timestamps from one producer to many consumers.
- **st-lib** — shared Rust types (currently just `OwnedMidi`).
- **NSM** (Non Session Manager) for session save/load (currently only st-loop).

## Submodules (top-level only)

| Submodule      | Role                                                                                  |
| -------------- | ------------------------------------------------------------------------------------- |
| `st-conductor` | JACK timebase master + st-sync TCP server. Owns tempo/time-signature/BBT.             |
| `st-sync`      | Library + bin. `Controller` (producer) / `Client` (consumer) of beat-frame `u64`s.    |
| `st-lib`       | Shared types crate. Currently defines `OwnedMidi { time, bytes }`.                    |
| `st-click`     | YAML-driven MIDI metronome / polyrhythm generator. JACK MIDI-out, st-sync client.     |
| `st-loop`      | 8-track × 8-scene live audio looper. MIDI-controlled, NSM-managed, st-sync client.    |

### Runtime topology

```
JACK server
  └─ st-conductor   (timebase master + st-sync server :6142)
        ├─ st-click  (JACK MIDI out,  st-sync client)
        └─ st-loop   (JACK audio I/O, st-sync client, NSM)
```

Start order: JACK → st-conductor → followers (st-click, st-loop).

## Conventions

- **Language:** Rust, edition 2021. Toolchain currently assumed nightly; see
  [plan.org](plan.org) (toolchain audit is a TODO).
- **Cross-crate sharing:** prefer adding types to `st-lib` over duplicating
  them. Sibling crates import via `path = "../st-lib"`.
- **Sync:** any new component that needs beat alignment should consume
  `st_sync::client::Client`, not roll its own.
- **Working style:** more conventions will be codified here as they arise.
  When in doubt, ask before inventing one.

## Working in this repo

- Submodules use **SSH** remotes (`git@github.com:spencerharmon/...`).
- After cloning fresh: `git submodule update --init --recursive`.
- Edits to a submodule are commits inside that submodule; the superproject
  then records the updated submodule SHA.
- Do **not** push, force-push, or amend without explicit user request.
- **Branch awareness.** The user frequently works in multiple branches
  across the submodules simultaneously. The checked-out submodule SHA in
  this superproject is *not* always the freshest code the user has in
  mind. When something doesn't add up (a `use` references a module that
  doesn't exist, a commit message describes code that isn't on disk,
  etc.), assume the user is talking about a branch you haven't fetched
  yet. Ask, or run `git -C <submodule> log --oneline --all` and
  `git -C <submodule> branch -a` to look around before concluding the
  tree is broken.

## Build-cache / disk hygiene

Rust `target/` dirs balloon fast across the suite's submodules and worktrees.
Keep the dev loop tight:

- **Lean dev debuginfo.** Each crate's (or a workspace's) root `Cargo.toml`
  should set:

  ```toml
  [profile.dev]
  debug = "line-tables-only"
  ```

  Panic backtraces still resolve to `file:line`; loses local-variable
  inspection in gdb/lldb. Override per-session when debugging:
  `CARGO_PROFILE_DEV_DEBUG=2 cargo build`. Typical `target/` shrink: 5-10×.

- **sccache as `RUSTC_WRAPPER`.** Per-developer shell export, e.g.
  `export RUSTC_WRAPPER=sccache` in `~/.bashrc`. Content-addressed,
  concurrency-safe, shares compiles across worktrees/submodules. Inspect
  with `sccache --show-stats`. **Do not** commit `rustc-wrapper` into
  `.cargo/config.toml` — breaks CI / environments without sccache.

- **Per-worktree `target/`, not a shared `CARGO_TARGET_DIR`.** Cargo takes
  a build lock per target dir, so sharing serializes concurrent builds and
  thrashes incremental caches across branches/features. Let sccache do
  the cross-worktree sharing.

- **Periodic GC** with `cargo install cargo-sweep` then
  `cargo sweep --time 30`. Safe — sccache rehydrates on next build.
  Manually delete unused cross-compile dirs (`target/aarch64-*/`, etc.);
  `cargo clean -p` won't touch them.

- **Scoped commands.** `cargo check -p <crate>` after edits,
  `cargo test -p <crate>` before committing that crate,
  `cargo build/test --workspace --all-features` only pre-push.

- **Don't** set `codegen-units = 1` or disable incremental in dev, and
  don't `--no-verify` / skip lints to paper over slow loops — fix the
  root cost.

## Planning

Active design notes, the GUI plan, and open TODOs live in
[plan.org](plan.org). Consult and update it as work progresses.
