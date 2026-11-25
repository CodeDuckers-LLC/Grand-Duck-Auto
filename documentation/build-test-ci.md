# Build, test, CI

## Toolchain

- Rust **1.98.1** pinned in `rust-toolchain.toml`, target
  `x86_64-pc-windows-msvc` only, edition 2024, MSRV 1.88.
- Host: Windows 10/11 x64, AVX2 CPU, D3D12 GPU with SM 6.6 (+ mesh shaders
  for the world). Several GiB of free commit charge at startup.

## Profiles

| Command | Output | Use |
|---------|--------|-----|
| `cargo build --release -p victory` | `target\release\victory.exe` | Play / measure |
| `cargo build --profile diag -p victory --features diagnostics` | `target\diag\victory.exe` | Debugging aids |
| dev | | Dependencies built at `opt-level=2` (they chew gigabytes of data) |

## Tests

- ~520 `#[test]` functions. `cargo test --workspace` needs no game data.
- Tests that need real files read paths from env vars and **skip** when unset.
- Shader tests run emitted DXIL on the real GPU or WARP and skip without one;
  GPU tests are serialised.
- Differential tests (`emit/diff_tests.rs`) compile HLSL with both the
  console and PC compilers (paths via env) and compare results.
- `tests/fixtures/hle-probe` is a tiny DLL of our own, built on demand,
  used as a fake title by `victory-loader/tests/load_probe.rs`.

## CI (`.github/workflows/ci.yml`)

On every pull request and push to `main`, `windows-latest`, 60 min limit:

1. `cargo clippy --workspace --all-targets --locked -D warnings`
2. same with `--all-features`
3. `cargo test --workspace --locked` (includes the asset-policy test)

## Asset policy gate

- `cargo xtask install-hooks` sets `core.hooksPath=.githooks`; the
  pre-commit hook runs `cargo xtask policy --staged`.
- Allowed file types: `rs toml md txt lock yml yaml json hlsl hlsli ps1 cmd sh`
  plus a few names. Max 512 KiB. No binary magic. No hex/base64 run > 512.
- If game data ever lands in git: rewrite history, do not just delete it.
