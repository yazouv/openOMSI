# Definition of Done

A change is done when every point below that applies to it is true. It gathers the rules of
[CONTRIBUTING.md](CONTRIBUTING.md), the pull request template and the automatic checks in one
list; the reasons behind them are in CONTRIBUTING.md.

## Scope

- [ ] The pull request holds **one change**: one fix, one feature or one clean-up.
- [ ] It is based on the latest `main` of `openOMSI-Project/openOMSI`.
- [ ] It comes from a branch of its own on a fork owned by a user account - pull requests from
      a `main` branch or from an organisation's fork are closed automatically.
- [ ] A fix for a GitHub issue names it (`Fixes #123`) and the issue was retested on the latest
      release first - it may already be fixed.

## Builds and tests

- [ ] `cargo build --release -p omsi-app` builds.
- [ ] `cargo test --workspace` passes. CI builds every platform, so code behind `#[cfg(...)]`
      for another system must compile too.
- [ ] A fixed bug has a test that fails without the fix, where the bug can be reproduced
      without a graphics card or OMSI 2 content. Tests that need content read `OMSI_ROOT` and
      skip themselves without it.
- [ ] Changes to the ray tracing shaders (`crates/omsi-render/src/rt*`) pass
      `cargo test --release -p omsi-render --test rt_smoke` on Direct3D 12 with DXC
      (`WGPU_BACKEND=dx12`, `WGPU_DX12_COMPILER=dynamicdxc`) and on Vulkan. The test prints
      `ray queries: true` when the shaders were really traced.

## Compatibility and performance

- [ ] Larger changes: `cargo run --release -p omsi-check -- "/path/to/OMSI 2"` reports no new
      failures compared to `main` - no stock map or mod that worked is broken.
- [ ] The change was tried in the running game (map, bus, steps) and behaves like
      OMSI 2.2.032, unless it is an openOMSI-only feature.
- [ ] Changes to rendering, tile streaming, traffic or other per-frame code: fps and memory
      were measured before and after (`OMSI_PROFILE=1 ... --exit-after N`) and did not get
      worse.

## Code

- [ ] New code follows `rustfmt`'s default style, but `cargo fmt` is **not** run on whole
      files: most of the code base is not formatted by it, and reformatting buries the change.
- [ ] Comments explain *why*, in plain English; existing references to the original game's
      behaviour or addresses are kept.
- [ ] `OMSI_*` switches on a hot path are read through `omsi_cfg::env`, not `std::env`.
- [ ] No warnings were added to the build.

## Content and files

- [ ] Nothing taken from an OMSI 2 installation: no textures, models, sounds, maps or scripts.
- [ ] No file of 1 MB or more; no builds, caches, converted or downloaded content, screenshots
      or videos in a commit.
- [ ] Third-party assets with their own licence were agreed with the maintainers first.

## Texts and documentation

- [ ] New interface texts go through `tr("...")` and are in
      `crates/omsi-app/locales/app.yml`. A reworded English key is updated in every language,
      and placeholders (`%{x}`, `{}`) are kept.
- [ ] A new or changed behaviour, setting or `OMSI_*` switch is described in `docs/`
      (`USER_GUIDE.md` for players, `FORMATS.md` for file formats, `ARCHITECTURE.md` for the
      design).

## Commit and pull request

- [ ] The commit subject has an area prefix (`Graphics: ...`, `Weather: ...`, `LAN: ...`,
      `fix: ...`, `perf: ...`, `chore: ...`) and says what the player notices.
- [ ] `CHANGELOG.md` and `VERSION` are left alone - maintainers update them.
- [ ] The pull request template is filled in: what changes for the player, how to test it with
      the test build, and why the change is made this way.
- [ ] The PR checks (large files, translations) and the test builds are green.
