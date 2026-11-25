# Configuration

## Paths

Copy `victory.example.toml` to `victory.toml` (git-ignored). Every path must
be **outside** the repository. Env vars win over the file.

| Key | Env | Default |
|-----|-----|---------|
| `game_dir` | `VICTORY_GAME_DIR` | none |
| `cache_dir` | `VICTORY_CACHE_DIR` | `%LOCALAPPDATA%\victory\cache` |
| `data_dir` | `VICTORY_DATA_DIR` | `%LOCALAPPDATA%\victory\data` |
| `test_titles` | `VICTORY_TEST_TITLES` (`;`-separated) | empty |
| (file) | `VICTORY_CONFIG` | `victory.toml` |

## Environment switches

About **140 distinct `VICTORY_*` variables** are read across 43 files. There
is no central list; each is documented in a comment where it is read
(`grep -rn '"VICTORY_' crates apps`). The README documents ~20 of them.

Counts by crate: gpu ~95, shader ~25, apps ~11, hle 9, core 6, loader 4,
formats 4.

### Most useful, by purpose

| Purpose | Switches |
|---------|----------|
| Logging | `VICTORY_LOG` |
| Host GPU debugging | `VICTORY_D3D12_DEBUG`, `_GBV`, `_DRED`, `_WARP` (software adapter, cannot hang the driver) |
| Frame capture | `VICTORY_CAPTURE`, `_CAPTURE_FRAMES`, `VICTORY_DUMP_TARGETS` |
| Pipelines | `VICTORY_PIPELINE_WAIT_MS` (default 500), `VICTORY_SKIP_PIPELINES`, `VICTORY_GPU_TIMING` |
| Shaders | `VICTORY_SHADER_OPT=0` (no clean-up passes), `VICTORY_SHADER_LOOP_LIMIT`, `VICTORY_EMIT_DUMP` |
| CPU-side debugging (diagnostics build) | `VICTORY_STACKS`, `VICTORY_WATCH`, `VICTORY_KEYS`, `VICTORY_CRASH_SAVE` |
| Audio | `VICTORY_AUDIO_CAPTURE`, `VICTORY_XMA_DUMP` |

### Fork speed experiments (off by default)

| Switch | Idea |
|--------|------|
| `VICTORY_LATE_WAIT=1..3` | Submit ahead of waiting for the GPU |
| `VICTORY_RELAY=1/2` | Thread that lands results when the GPU finishes; `2` waits only on flights that touch the memory needed |
| `VICTORY_WIDE_BACK`, `_WIDE_MERGE`, `_WIDE_REFRESH` | Wider copy-back of shader-written memory |
| `VICTORY_CARD_BUFFERS*` | Keep large title buffers in GPU memory, detect writes by hashing on the GPU |
| `VICTORY_INPLACE*` | Title memory opened as a GPU heap (first stage only) |
| `VICTORY_LAZY_TEXTURES`, `VICTORY_TEXTURE_*` | Texture upload budgets and staging |

README's best measured combo: `VICTORY_RELAY=2`, or
`VICTORY_LATE_WAIT=3 VICTORY_WIDE_BACK=1 VICTORY_CARD_BUFFERS_MIN_MB=1 VICTORY_CARD_BUFFERS_MB=1024`
(240-350 ms/frame vs 420-450 default).

### Ray tracing (off by default)

`VICTORY_BVH_BUILDER=1`, `VICTORY_BVH_QUARANTINE=1`, `VICTORY_WAVE_WIDE=1`
(the last caused a device loss once; keep off unless testing).

### Test-only

`VICTORY_PC_DXC`, `VICTORY_CONSOLE_DXC`, `VICTORY_DXIL`, `VICTORY_REF_VS/PS`,
`VICTORY_BCPACK_*`, `VICTORY_XMA_REFERENCE/STREAMS`, `VICTORY_MESH_TEST`,
`VICTORY_WARP`: point tests at tools or user data; tests skip when unset.
