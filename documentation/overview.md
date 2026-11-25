# Overview

## What it is

A Rust workspace that runs an Xbox Series X|S title natively on Windows, from
files the user supplies. It is a **compatibility layer**, not an emulator of
the CPU:

- Console executables are x86-64 PE32+ images built for a Zen 2 CPU, so the
  title's machine code runs **unchanged inside our process**.
- What differs is the platform underneath. The layer supplies it:
  - console system DLLs (`xgameruntime.dll`, `xmem.dll`, `d3d12_xs.dll`, ...)
    are reimplemented in Rust ("HLE", high-level emulation);
  - Windows API sets the console shares with PC pass through to the host;
  - the console's D3D12.X graphics API is mapped onto host Direct3D 12;
  - console GPU shaders (native RDNA2 / GFX10 machine code) are recompiled to
    DXIL (Shader Model 6.6) on first use and cached on disk;
  - console data formats (tiled textures, BCPack, XMA2, Opus) are decoded.

Platform-level code works for any Xbox Series title. Title-specific code lives
in one crate, `victory-game`.

## Lineage

```
Marvisak/VIctory (upstream, GPL-3.0-or-later)
   └── this fork  (one squashed commit "this fork's changes on top of VIctory main (ffe310e)")
         └── later fork commits, plus branches taken from other forks
```

- Upstream history is kept intact below commit `578d459`.
- `ffe310e` is the upstream `main` the fork is based on.
- Recent commits pull in branches from other people's forks as single commits
  (`3bff182`, `31714de`).
- Commit `4e17478` removed the title's name from the tree. Keep it that way.
- XCL (GPL-2.0-only) was used as a behavioural reference only; no code copied.

## Status (from README, 2026-11-25)

| Area | State |
|------|-------|
| Boot, menus, start screen | Works, 30 fps |
| Opening cutscene | Plays fully, lit |
| Saves | Autosave and load work |
| First gameplay scene | Renders; HUD, pause, pad input work |
| Shadowed areas | **Black**: the title's ray-traced bounce light and shadows do not run yet |
| World frame rate | **2-4 fps** (240-450 ms/frame) |
| Stability | No hangs in normal runs |
| Memory | ~24 GB during play |

Tested on one PC only: Windows 11, i7-14700KF, RTX 4070 12 GB, 32 GB RAM.
Upstream was tested on an AMD RX 6700 XT.

## The two big open problems

1. **Ray tracing.** The title does its own BVH build and traversal in compute
   shaders and uses one hardware instruction (`image_bvh*_intersect_ray`).
   The fork emulates that instruction in DXIL and translates most of the
   builder/trace programs, but nothing is lit by it yet. Details:
   `docs/fork-ray-tracing.md`.
2. **Speed.** Title memory has to be copied to the host GPU, submissions are
   waited on synchronously by default, and translated shaders are heavier
   than the originals. Many experimental speed switches exist, off by default
   (see [configuration.md](configuration.md)).
