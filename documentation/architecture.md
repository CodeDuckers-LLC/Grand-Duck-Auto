# Architecture

See also upstream's own `docs/architecture.md` (short) and the crate docs
(`//!` comments at the top of every `lib.rs` and module, which are good).

## Layout

```
Cargo.toml            workspace: crates/*, apps/*, xtask
apps/victory/         the CLI binary (`victory.exe`)
crates/victory-core       config, asset-policy guard, logging
crates/victory-pe         read-only PE32+ / import-library parsing
crates/victory-loader     prepare, map, bind and start the title image
crates/victory-hle        console system API implementations (Rust)
crates/victory-gpu        D3D12.X objects -> host D3D12
crates/victory-shader     console GPU shaders -> DXIL
crates/victory-formats    texture tiling, BCPack, XMA2 (pure, no I/O)
crates/victory-pipeline   converter trait + content-addressed cache
crates/victory-game       title-specific: SGD5 shader files, build ids
xtask/                    `cargo xtask policy`, `install-hooks`
tests/fixtures/hle-probe  our own stand-in "title" DLL for loader tests
docs/                     upstream + fork design notes
```

Size: about 111k lines of Rust. The big files are
`victory-shader/src/emit/lower.rs` (11.7k), `victory-gpu/src/host.rs` (6.6k),
`victory-shader/src/resource.rs` (3.8k), `victory-gpu/src/resource.rs` (3.4k),
`victory-gpu/src/shadow.rs` (3.2k).

## Crate dependency graph

```
                     apps/victory (CLI)
      ┌──────────┬──────┼───────────┬────────────┐
      ▼          ▼      ▼           ▼            ▼
 victory-core  victory-pe  victory-loader  victory-hle  victory-game
      ▲          ▲  ▲        │  │             │  │  │
      │          │  └────────┘  │             │  │  └─► victory-formats
      └──────────┼──────────────┘             │  └────► victory-loader, victory-pe
                 │                            ▼
                 │                       victory-gpu ──► victory-shader
                 │                            │   └────► victory-formats
                 │                            └────────► victory-pipeline ─► victory-core
```

Edges (from the `Cargo.toml` files):

| Crate | Depends on (internal) |
|-------|----------------------|
| `apps/victory` | core, pe, loader, hle, shader (`host` feature), game |
| `victory-hle` | gpu, loader, pe, formats |
| `victory-gpu` | formats, pipeline, shader |
| `victory-loader` | core, pe |
| `victory-pipeline` | core |
| `victory-shader`, `victory-formats`, `victory-pe`, `victory-game`, `victory-core` | none |

Notes:
- `victory-shader` and `victory-formats` are pure leaves: easy to test on any OS.
- `victory-gpu` does not depend on `victory-hle`; HLE calls into GPU, and GPU
  gets callbacks injected (allocator, page mapper, failure hook) at device
  creation (`victory-hle/src/modules/d3d12x.rs`).
- Everything that touches Windows is behind `cfg(windows)`; much of the code
  builds but does nothing elsewhere. Only `x86_64-pc-windows-msvc` is a
  supported target.

## What `victory run <exe> --execute` does

From `apps/victory/src/run.rs`, in order:

1. Wait for a previous instance if the title relaunched itself
   (`hle::modules::lifecycle`).
2. Load config (`victory_core::Config`), read the exe.
3. `victory_loader::prepare`: patch a copy of the image so Windows will map
   it as a DLL but run none of its code (no entry point, no imports, TLS
   callbacks detached). Hash it with BLAKE3 and look it up in
   `victory_game::identify` (known builds).
4. Point the shader cache at `cache_dir/shaders`.
5. `victory_loader::load`: write the prepared copy to `cache_dir/images`
   and map it as a real Windows module (debuggers and stack walks see it).
6. Read `MicrosoftGame.config` beside the exe (title id, store id,
   storage quota). Mount console drives: `G:` = game folder, `T:` = temp
   under the data dir. Persistent storage and saves go under
   `data_dir/<TITLEID>/`.
7. Pick the console model (Series X default, or Series S).
8. Map the DLLs the title ships beside its exe (those neither HLE nor the
   host provide).
9. Bind every import of every image through `HleResolver` (see
   [communication.md](communication.md)). Print counts:
   `hle / host / title / missing`.
10. Without `--execute`, stop here (a dry run).
11. Install crash reporting (`crashes.log`), 1 ms timers, diagnostics
    (if built with the feature), and a late-DLL loader for `LoadLibrary`.
12. `spawn_main`: run the title's DLL inits, TLS callbacks and entry point on
    a thread with the stack size the image asks for. Join it.

From there on, the title drives everything by calling the functions we bound.

## Other CLI commands

| Command | File | Purpose |
|---------|------|---------|
| `inspect` | `inspect.rs` | Headers, imports, exports of a PE |
| `census` | `census.rs` | Classify every import (hle/host/title/missing) without running |
| `lib-exports` | `main.rs` | Read a COFF import library |
| `shader-info` | `shader.rs` | Describe shader containers, optional disassembly |
| `translate` | `translate.rs` | Batch-translate every shader found (containers, SGD5 files), report failures, optional validation and pipeline creation |
| `run` | `run.rs` | Above |
| `config` | `main.rs` | Show resolved paths |
