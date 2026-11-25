# How the parts communicate

There is no network or IPC between components: it is **one process**. The
title's code and ours share an address space. Communication is by function
pointers, shared memory and a few background threads.

## 1. Title -> our code: import binding

The title calls Windows/console DLL functions through its import table. The
loader fills that table itself (`victory-loader/src/windows.rs`, through the
`ImportResolver` trait in `resolve.rs`). For every `dll!function`:

1. **HLE**: an implementation in `victory-hle::modules` (always wins).
   Each module declares `HleModule { name, dlls: &[patterns], exports }`
   using the `hle_export!` macro; `registry.rs` collects them
   (`modules::all()`), and DLL patterns may end in `*` for API sets.
2. **Host**: the real Windows function, if `HostPolicy`
   (`victory-hle/src/host.rs`) allows the DLL. Allowed: core API sets,
   kernel32/ntdll, CRT, crypto, winsock (network refused later in
   `modules::net`), XAudio2, Media Foundation, dbghelp, etc. Loaded from
   System32 only.
3. **Title DLL**: an export of a DLL the title ships, mapped like the exe.
4. **Stub**: a 32-byte generated thunk (`victory-loader/src/thunks.rs`) that
   names the missing import when called. `--unimplemented trap` aborts;
   `zero` logs once and returns 0.

Runtime lookups follow the same rules: `LoadLibrary` of a console DLL returns a
pseudo handle and `GetProcAddress` finds our function; `QueryApiImpl` hands
out our runtime objects (XUser, XTaskQueue, ...) by interface id
(`modules/xgameruntime.rs`).

`victory census` prints this classification without running the title.

## 2. HLE modules: what the title sees

`crates/victory-hle/src/modules/`, one file per console API area. The
console is presented as **one offline retail console with one signed-in
user, full licence, no add-ons**.

| Area | Files |
|------|-------|
| Memory | `xmem.rs` (console memory from host memory, write watching for GPU copies) |
| Graphics entry | `d3d12x.rs` (device creation -> `victory-gpu`) |
| Texture layouts | `xg.rs` (console `xg_xs.dll`; uses `victory-formats::texture`) |
| Files / storage | `files.rs` (virtual drives over host folders), `storage.rs`, `dstorage.rs` (DirectStorage queues, zlib/BCPack decode) |
| Saves | `xgamesave.rs` (crash-safe writes under the data dir) |
| Runtime / identity | `xgameruntime.rs`, `xgame.rs`, `xsystem.rs`, `xuser.rs`, `xpackage.rs`, `xstore.rs`, `lifecycle.rs` |
| Async | `xasync.rs` (XTaskQueue, XAsync) |
| Network | `net.rs` (loopback only), `xnetworking.rs` (stack up, reaches nothing) |
| Input | `gameinput.rs` (XInput pads, keyboard stand-in pad) |
| Audio | `audio.rs` (AcpHal, XMA voices), `apu.rs` (Opus), `dsp.rs` (reverb, silent), `speech.rs` (empty WAVE) |
| UI / shell | `xgameui.rs` (text entry dialog), `display.rs`, `media.rs` (local-only source resolver) |
| Windows overrides | `kernel.rs` (APIs whose console meaning differs) |

Process-wide shared state lives in `victory-hle/src/runtime.rs` (main image
base, image paths, title manifest data, late DLL loader) as statics.

## 3. Title -> GPU: D3D12.X objects

- The title calls `d3d12_xs.dll` device creation, served by
  `hle::modules::d3d12x::create_device`, which first injects into
  `victory-gpu` the title's allocator (`XMemAlloc`), the page mapper, and a
  GPU-failure hook, then calls `victory_gpu::device::create`.
- GPU objects (`device.rs`, `queue.rs`, `list.rs`, `resource.rs`, ...) are
  **binary-compatible structs** with a function table the title's inlined
  code expects (`object.rs`: vtable pointer, flags, refcount; IUnknown first).
- D3D12.X normally writes command packets straight into memory while a put
  pointer is below a limit. **Our lists keep every limit at zero**, so every
  command arrives as a method call (`list.rs`), which records onto a host
  D3D12 command list.
- Console memory is unified: the title's GPU addresses are CPU addresses.
  The host GPU cannot read them, so:
  - `shadow.rs` keeps host copies of title buffers;
  - `watch.rs` uses Windows write-watching (set up by `xmem`) to find what
    the title changed since the last copy;
  - `texture.rs` detiles title-written textures;
  - `inplace.rs` (experimental, `VICTORY_INPLACE`) opens title memory as a
    host heap to avoid copies;
  - `memory.rs` gives shaders raw-address access to title memory (ray
    tracing path).
- `draw.rs` brings the host list up to date at each draw (pipeline, root
  signature, user data, buffer copies, descriptor heaps, texture states).
- `indirect.rs` clamps indirect-draw arguments before the GPU reads them.
- `queue.rs` submits to the single host direct queue (`host.rs`). By default
  it waits, so fences complete at once. `VICTORY_LATE_WAIT` sends work ahead,
  and `VICTORY_RELAY` adds a thread that lands results as the GPU finishes.
- `frame.rs` paces frames on a clock; `dxgi.rs` provides one adapter/output;
  the host window and present live in `host.rs`.

## 4. GPU -> shaders: translation on pipeline creation

```
title creates a pipeline (bytes of a DXBC container)
  └► victory-gpu/pipeline.rs
       └► pipeline/make.rs  worker threads (not the title thread)
            └► shader_cache.rs   hit? use cached DXIL
                 miss: victory-shader
                   dxbc.rs     container parts
                   native.rs   GFX10 program inside the DXIL part
                   gfx10/      decode instructions
                   ir.rs       SSA, one inst per machine inst
                   structure.rs remove EXEC: lane masks -> per-lane bools
                   resource.rs  trace descriptors to root parameters
                   emit/        lower.rs -> DXIL bitcode -> container (SM 6.6)
                   emit/bitcode/opt.rs  clean-up passes (VICTORY_SHADER_OPT=0 off)
            └► pipeline/host.rs   host PSO with our own root signature
            └► pipeline/library.rs  compiled PSOs kept on disk (mapped file)
```

- Draws whose pipeline is not ready yet are skipped, after a wait of up to
  `VICTORY_PIPELINE_WAIT_MS` (default 500 ms) per frame.
- Shader cache key: container content + BLAKE3 of our own executable, so a
  rebuild invalidates old entries.
- Console waves can be 64 lanes; most host GPUs here use 32. `emit/wave.rs`,
  `own_bits.rs`, `structure_wide` handle that; `VICTORY_WAVE_WIDE` is a
  related switch that caused a device loss and is off by default.

## 5. Data on disk

| Where | What | Written by |
|-------|------|------------|
| `cache_dir/images` | Prepared copies of exe / DLLs | loader |
| `cache_dir/shaders` | Translated DXIL | gpu `shader_cache` |
| `cache_dir` (pipeline library) | Compiled host PSOs | gpu `pipeline/library` |
| `data_dir/<TITLEID>/storage`, `saves`, `temp` | Persistent storage, saves, `T:` | hle |
| `data_dir/<TITLEID>/crashes.log` | Crash and GPU failure reports | loader, gpu |

Every writer of derived data calls `victory_core::ensure_outside_repo`, which
refuses paths under a folder with `.victory-root`.

## 6. Threads

- Title main thread (`spawn_main`) and every thread the title creates; the
  loader's own TLS callback forwards thread attach/detach to mapped images
  (`threads.rs`).
- Pipeline workers (`pipeline/make.rs`).
- Optional relay thread (`VICTORY_RELAY`).
- Parallel decode for DirectStorage (zlib, BCPack).
- Diagnostics threads (`VICTORY_WATCH`, stack dumps).

## 7. CPU instruction gaps

Zen 2 has instructions some Intel CPUs lack (`CLZERO`, `RDPRU`, `MONITORX`,
SSE4a, sometimes `CLWB`/SHA). A vectored exception handler
(`victory-loader/src/console_insn*.rs`) emulates them and steps over.
