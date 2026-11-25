# Crates and modules

Line counts are approximate (2026-11-25). Every file opens with a `//!`
comment that explains it; read those first.

## apps/victory (~1.9k lines)

The CLI (`clap`). `main.rs` parses commands; `run.rs` is the launcher
(see [architecture.md](architecture.md)); `translate.rs` is the offline
shader batch (also used to measure translation coverage); `census.rs`,
`inspect.rs`, `shader.rs` are analysis commands. Feature `diagnostics`
turns on debug aids across crates.

## victory-core (~0.4k)

- `config.rs`: paths (`game_dir`, `cache_dir`, `data_dir`, `test_titles`).
  Order: env var, then `victory.toml`, then defaults under `%LOCALAPPDATA%`.
- `policy.rs`: `ensure_outside_repo` (runtime half of the asset policy).
- `logging.rs`: tracing setup, `VICTORY_LOG` / `RUST_LOG`.

## victory-pe (~0.8k)

Read-only PE32+ and COFF import-library parsing on top of `object`.
`layout.rs` exports raw header offsets for the loader's patching.

## victory-loader (~3.3k)

| Module | Purpose |
|--------|---------|
| `prepare.rs` | Patch a copy: DLL flag, no entry point, no imports, TLS callbacks detached (pure, testable anywhere) |
| `windows.rs` | Map as a real module, bind imports, run entry |
| `resolve.rs` | `ImportResolver` trait |
| `thunks.rs` | Stubs for unresolved imports |
| `tls.rs` | Redirect exe TLS slot 0 for an exe mapped as DLL |
| `threads.rs` | Forward thread attach/detach to mapped images |
| `crash.rs` | Vectored handler: `module+rva`, unwind, `crashes.log` |
| `console_insn*.rs` | Emulate Zen 2 instructions Intel lacks |
| `stacks.rs` | Scheduled stack dumps (`VICTORY_STACKS`) |

## victory-hle (~19k)

Console system APIs. `lib.rs` has `HleResolver` (HLE -> host -> missing),
`host.rs` the `HostPolicy` allowlist, `registry.rs` the module registry,
`runtime.rs` shared process state. The `modules/` table is in
[communication.md](communication.md#2-hle-modules-what-the-title-sees).
Largest: `dstorage.rs`, `gameinput.rs`, `xgamesave.rs`, `xasync.rs`,
`xmem.rs`, `audio.rs`, `net.rs`.

## victory-gpu (~39k)

D3D12.X on host D3D12. The biggest and busiest crate in the fork.

| Group | Modules |
|-------|---------|
| Objects the title sees | `object`, `device`, `dxgi`, `queue`, `list`, `fence`, `resource`, `descriptor`, `root`, `pipeline`, `signature`, `query`, `console`, `frame` |
| Host side | `host` (device, queue, window, submission, late wait / relay), `draw`, `descriptor/host`, `pipeline/{host,make,library}`, `shader_cache`, `sampler`, `primitive` (rect lists) |
| Title memory on the GPU | `shadow` (buffer copies), `watch` (write tracking), `texture` (title-written textures), `inplace` (title memory as a heap), `memory` (raw-address access), `indirect` (argument clamping) |
| Diagnostics | `capture` (PNG frames), `copystat` (copy statistics) |

## victory-shader (~41k)

Console GFX10 machine code -> DXIL.

| Group | Modules |
|-------|---------|
| Containers | `dxbc`, `dxil`, `native` (GFX10 program + `XDAT` blobs) |
| Decode | `gfx10/{mod,fields,opcodes,sp3}` (SP3 = AMD asm text, for disassembly) |
| Analysis | `ir` (SSA), `structure` (EXEC removal, control flow), `resource` (descriptor provenance), `io` (inputs/exports), `spill` (spilled user data) |
| Emit | `emit/mod`, `emit/lower` (11.7k: per-instruction lowering), `emit/shader`, `emit/bitcode` (+ `opt` clean-ups, `census`), `emit/container`, `emit/wave`, `own_bits`, `threading`, `intersect` (BVH test in DXIL), `rect` (rect-list GS), `sampler`, `validator` (`dxil.dll`), `host` (D3D12 checks) |
| Reference | `bvh.rs` (CPU reference for the ray/node test) |
| GPU tests | `emit/*_tests.rs` (run on real GPU or WARP; skip without one) |

## victory-formats (~4.4k, pure)

`texture.rs` (RDNA2 tiling, sizes, detile), `bcpack.rs` (BC1-7 packing,
decode only; uses the one allowed binary file `bcpack-tables.bin`), `xma.rs`
(XMA2 audio decode).

## victory-pipeline (~0.2k)

`Converter` trait (bytes in, bytes out, versioned) and a content-addressed
`Cache`. Only user today: the GPU shader cache.

## victory-game (~0.2k)

Title-specific. `sgd5.rs` reads the engine's shader archive (records found by
marker, deflate). `KNOWN_BUILDS` is **empty**, and no hooks exist yet.

## xtask (~0.3k)

`cargo xtask policy [--staged]` (asset allowlist: extensions, < 512 KiB, no
binary magic, no long hex/base64 runs), `cargo xtask install-hooks`.
