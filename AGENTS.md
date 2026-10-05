# AGENTS.md

Guidance for coding agents (and people) working in this repository. A change is finished when
it meets [DEFINITION_OF_DONE.md](DEFINITION_OF_DONE.md).

## What this is

openOMSI is a from-scratch Rust recreation of the bus simulator OMSI 2 (target: 1:1 behaviour
with OMSI 2.2.032), rendering through wgpu (Metal / Vulkan / Direct3D 12 / GL). It ships **no
game content**: it reads maps, vehicles and mods from an installed OMSI 2 and won't start
without one. Never commit anything taken from an OMSI 2 installation (textures, models, sounds,
maps, scripts). File formats are described in `docs/FORMATS.md`, not copied.

## Commands

```sh
cargo build --release -p omsi-app                        # the game -> target/release/openomsi
cargo test --workspace                                   # must pass (CI builds every platform)
cargo test -p omsi-script <test_name>                    # a single test in one crate
cargo run --release -p omsi-check -- "/path/to/OMSI 2"   # load every content file, report failures
cargo test --release -p omsi-render --test rt_smoke -- --nocapture   # Enhanced+ ray tracing
scripts/version.sh                                       # version of the checked-out commit
```

- Packaged builds: `scripts/build-{macos,linux,android,server}.sh`, `scripts\build-windows.cmd`
  -> `dist/<platform>/` (see `docs/BUILDING.md`). Rust 1.85+; Windows needs the MSVC build
  tools and CMake (OpenXR).
- `profile.dev` is `opt-level = 1`; use `--release` for anything that runs the game.
- Tests needing real content read `OMSI_ROOT=/path/to/OMSI 2` and skip themselves (or are
  `#[ignore]`d) without it.
- Run `omsi-check` before and after larger changes: a change must not break a stock map or mod
  that worked.
- Ray tracing shaders only compile on Direct3D 12 with DXC (`WGPU_BACKEND=dx12`,
  `WGPU_DX12_COMPILER=dynamicdxc`, `dxcompiler.dll` on `PATH`). The smoke test prints
  `ray queries: true` when they were actually traced; without it the test proves nothing about
  them.

## Checking the running game

- `openomsi --offscreen out.png --cam x,y,z,yaw,pitch --drive secs` renders frames to files.
- `OMSI_INPUT=...`, `OMSI_LAUNCHER_SHOT` / `OMSI_LAUNCHER_INPUT` / `OMSI_LAUNCHER_EXIT`,
  `OMSI_PROFILE=1 ... --exit-after N` and the `OMSI_DEBUG_*` / `OMSI_NO_*` A/B switches are
  listed in the debug switches table of `docs/USER_GUIDE.md`.
- Logs go to `~/.openomsi/` (`game.log`). Players attach that file to issues; GPU and shader
  errors appear there as `ERROR omsi_render` / `ERROR wgpu`.

## Architecture

The crates mirror the units of the original Delphi program (mapping table in
`docs/ARCHITECTURE.md`):

- **Formats and parsing**: `omsi-cfg` (OMSI's keyword-line text format, code pages, the virtual
  file system with content roots and mounted `.zip` mods), `omsi-script` (OMSI script compiler
  and VM), `omsi-o3d`, `omsi-model`, `omsi-texture`, `omsi-scenery`, `omsi-map`,
  `omsi-vehicle`, `omsi-timetable`, `omsi-content` (weather, fonts, humans, situations, ...).
- **Runtime**: `omsi-geometry` (spline/terrain tessellation, mesh frame conversion), `omsi-sim`
  (vehicle runtime, AI traffic, humans, physics, text textures, HTML engine), `omsi-render`
  (wgpu renderer: the vanilla path reproduces Direct3D 9 output, `enhanced` is its own PBR
  path, Enhanced+ adds ray tracing in `rt.rs` / `rt*.wgsl`), `omsi-audio` (cpal), `omsi-net`
  (LAN UDP sessions), `omsi-plugin` (Lua plugins, OMSI plugin DLLs, 32-bit plugin host),
  `omsi-ui` (UI toolkit, `i18n::tr`).
- **`omsi-app`**: the `openomsi` binary - window and main loop, launcher (`launcher/`, the
  game's own wgpu window replacing the original's dialogs), HUD, tile streaming (`tiles.rs`),
  schedule/timetable runtime (`schedule.rs`), traffic/humans glue, LAN game, dedicated server
  (`--server`), offscreen mode. `omsi-launcher-core` is the launcher's data side plus the
  `openomsi-launcher` CLI.

Design points to keep in mind:

- Tile loading, parsing and tessellation run on the rayon pool; a dedicated streamer thread
  stages tiles; each frame uploads only a slice of a tile to avoid stutter. Object and spline
  types are shared through `Arc` caches.
- Textures decode and BC1/BC3-compress on worker threads, which also make the wgpu device
  calls, so the draw thread only inserts into the scene. A texture memory budget drops and
  restores mip levels by distance.
- The renderer rewrites only changed per-draw entries (`Scene::changed`); full rebuilds happen
  only on instance adds or origin moves. Performance regressions matter: fps and memory are
  measured per change.
- Coordinate frames differ: world is x east / y north / z up; `.cfg` is x right / y forward /
  z up; meshes are converted from the Direct3D frame on load; map splines store
  (x, height, y). Read "Coordinate frames" in `docs/ARCHITECTURE.md` before touching
  transforms.
- Read `OMSI_*` environment switches through `omsi_cfg::env` (cached), never `std::env`, on any
  hot path.
- Behaviour is matched to `Omsi.exe`; comments often cite the original's behaviour or addresses
  as justification. Keep that rationale when editing.
- WGSL is translated to HLSL, MSL and SPIR-V by naga, which keeps constant names. Don't name
  WGSL items after built-ins of those languages (for example HLSL's `RAY_FLAG_*`): DXC rejects
  the shader and Enhanced+ stops drawing on Direct3D 12.

Some files are very large (`omsi-app/src/scene.rs`, `omsi-render/src/lib.rs` ~14k lines,
`traffic.rs`, `humans.rs`, `schedule.rs`): search them instead of reading them whole.

## Conventions

- Style: `rustfmt` defaults for new code, but don't run `cargo fmt` over whole files - most of
  the code base isn't formatted by it. Comments explain *why*.
- Prose (comments, docs, commit messages, changelog) is plain, player-facing English describing
  what changes for the player.
- Commit subjects carry an area prefix: `Graphics: ...`, `Weather: ...`, `LAN: ...`,
  `Changelog: 0.1.N`, or `fix:` / `perf:` / `chore:`.
- `CHANGELOG.md` is maintained by hand in "Changelog: <version>" commits (`### New` /
  `### Fixes` / `### Merged pull requests` per version) - by the maintainers.
- Versions are `MAJOR.MINOR.COMMIT`: `VERSION` holds `MAJOR.MINOR` (bumped by maintainers only);
  `COMMIT` counts commits on `main` since `VERSION` changed. Every push to `main` is released by
  CI.
- UI strings live in `crates/omsi-app/locales/app.yml`, keyed by the English text and looked up
  with `tr("...")`. Rewording an English key breaks the lookup for every translation - update
  all entries. Keep placeholders (`%{x}`, `{}`); CI checks them
  (`.github/scripts/check_locales.py`).
- No file of 1 MB or more in a pull request (CI rejects it); no generated or downloaded
  content, builds or screenshots in commits. New third-party licensed assets need the
  maintainers' agreement.
- User-facing docs live in `docs/` and are published as the website (`site/`); update them when
  behaviour or switches change.
- Pull requests: one change each, from a branch other than `main` on a fork owned by a user
  account; the template asks what the player notices and how to test it.

## This fork

`origin` is the fork (`yazouv/openOMSI`), `upstream` is `openOMSI-Project/openOMSI`. This file,
`DEFINITION_OF_DONE.md` and the link to it in `CONTRIBUTING.md` belong to the fork only and must
never reach a pull request to `upstream`:

- start every pull request branch from `upstream/main`, never from the fork's `main`;
- before pushing it, `git diff upstream/main --stat` must list none of these files.

## Before fixing an issue

Issues pile up quickly after a release, and many are fixed by the time someone looks. Check the
version the reporter used against the latest release and the commits since
(`git log v<version>..upstream/main`) before writing a fix.
