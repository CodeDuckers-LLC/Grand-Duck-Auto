# Documentation: initial repository scan

Notes from a first read-through of this fork (scan date 2026-11-25, branch
head `31714de`). They describe where the code is, what each part does, and how
the parts talk to each other. They add to `docs/`, they do not replace it:
`docs/` holds upstream's and the fork's own design notes; this folder is an
onboarding map.

| File | What it covers |
|------|----------------|
| [overview.md](overview.md) | What the project is, its lineage, current status |
| [architecture.md](architecture.md) | Crates, their dependency graph, the run sequence |
| [communication.md](communication.md) | How the pieces talk: imports, HLE, GPU objects, shaders, caches, threads |
| [crates.md](crates.md) | Each crate and its main modules, with purpose and size |
| [configuration.md](configuration.md) | Config file, paths, and the ~140 `VICTORY_*` environment switches |
| [build-test-ci.md](build-test-ci.md) | Toolchain, build profiles, tests, CI, the asset-policy gate |
| [findings.md](findings.md) | Observations, risks, gaps and suggested next steps |

Ground rule for anyone working here: **nothing from the game goes in the
repository** (see `docs/asset-policy.md`). These notes follow it: they
quote no game data and do not name the title.
