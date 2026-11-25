# Findings from the first scan

Things noticed while reading. None were changed in this PR; it adds
documentation only.

## Where things stand

- The platform layer (loader, HLE, file/save/input/audio) is mature: menus,
  saves, cutscenes and input work.
- All active work is in **`victory-gpu`** and **`victory-shader`**
  (~80k of ~111k lines). The two blockers are ray-traced lighting (black
  shadows) and frame time (2-4 fps).
- `victory-game` is nearly empty: `KNOWN_BUILDS` has no entries and no hooks
  exist. Every fix so far is at the platform level.

## Documentation gaps

1. **Missing decision records.** Code cites "decision 0003" (shader
   translation, 10 places) and "decision 0007" (host GPU design, 16 places),
   but no such documents are in the repo. Worth recovering from upstream or
   writing down.
2. **No central list of env switches.** ~140 `VICTORY_*` variables; README
   lists ~20. A generated table (e.g. an `xtask` command that greps them with
   their doc comments) would help.
3. **Stale module docs**, written before the host GPU path existed:
   - `victory-formats/src/lib.rs` and `docs/architecture.md` call BCPack and
     audio codecs "planned"; `bcpack.rs` and `xma.rs` exist.
   - `victory-gpu/src/lib.rs` calls `present` "planned"; presenting is in
     `host.rs`.
   - `fence.rs`, `query.rs` say "nothing runs on a GPU yet"; it does now.
4. `victory-pipeline` is described as the offline conversion pipeline, but
   its only user is the GPU shader cache.

## Code health

- **Relaxed lints.** The fork turned off several clippy lints in the
  workspace `Cargo.toml`, including `undocumented_unsafe_blocks`. There are
  ~1,090 `unsafe {` blocks; upstream warns on any without a `SAFETY:` comment,
  and the `Cargo.toml` note says eleven of the fork's newer GPU blocks lack one.
- **158 `shortcut:` markers** flag places that approximate console behaviour
  for this title or this GPU. They are a good to-do list.
- Very large files: `emit/lower.rs` (11.7k lines), `gpu/host.rs` (6.6k).
  Hard to review; candidates to split.
- Many switches are off by default and interact (`LATE_WAIT` x `RELAY` x
  `WIDE_*` x `CARD_BUFFERS*`). Untested combinations are a risk.

## Testing and portability

- Tested on **one** PC (NVIDIA RTX 4070, 32-lane waves). Upstream used AMD.
  Wave-size code paths (`wave64` vs 32) are a known source of device loss.
- Windows-only target. Pure crates (`victory-shader` analysis,
  `victory-formats`, `victory-pe`, `loader::prepare`) could be tested on
  Linux too, if wanted.
- CI runs on hosted Windows runners with no real GPU: GPU tests skip or use
  WARP there, so GPU regressions are caught only on dev machines.
- This scan did not build or run anything (Linux container, Windows-only
  target).

## Process and provenance

- Recent history squashes branches from **other people's forks** into single
  commits (`3bff182`, `31714de`). Keep a note of the source and author of
  each, for GPL attribution and review.
- Upstream history is intact under `578d459`; rebasing onto newer upstream
  will collide mostly in `victory-gpu` and `victory-shader`.
- The asset policy is strict and enforced by hook + CI. Install the hook
  (`cargo xtask install-hooks`) before committing.

## Suggested next steps

1. Recover or write decisions 0003 and 0007 into `docs/`.
2. Generate an env-switch reference.
3. Fix the stale `//!` docs listed above.
4. Restore `undocumented_unsafe_blocks` and add the missing `SAFETY:` notes.
5. Track `shortcut:` sites and ray-tracing gaps as issues.
6. Get a second test machine (AMD) into the loop.
