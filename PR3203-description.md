## What changes for the player?

Aligned ditch splines get continuous terrain shoulders and solid backing instead of holes, sharp vertical strips, or distorted banks. The transition follows the saved hillside through bends and grades, closes locally at the ditch ends, and limits new fill against visible road surfaces, including asphalt without vehicle paths.

This deliberately departs from Omsi.exe's projected hole under aligned splines: successful fitting supplies continuous backing through the trough and its extended ends, where a hole alone can expose the sky or leave sharp soil strips. If triangulation fails, the tile now restores the original saved ground **and the legacy spline rim cuts and soil walls**, including neighboring source tiles.

## How to test

1. Load a map containing narrow splines placed with `[spline_terrain_align]` or `[spline_terrain_align_2]`. For e.g. **Map 400 by P3ta**
2. Inspect the ditch beside a steep embankment, through a bend and a grade, at both terminals, and across a tile border. Check all four terminal modes and mirrored splines.
3. Inspect the adjacent carriageway, including an asphalt spline without `[path]` declarations whose traffic uses a separate invisible spline.
4. Expect continuous terrain backing and shoulders, locally closed ends, and no new fill over the road. Check that existing object/tunnel holes, bridges, terrain painting, and road collision surfaces still behave correctly.
5. Run `cargo test --locked -p omsi-geometry -p omsi-scene --lib` for the synthetic regressions. In particular, `aligned_ditch_shoulder_respects_asphalt_without_vehicle_paths` reproduces the road-surface selection mistake when only drivable splines are considered.

Validation of the review fixes on upstream base `caf95e8a`:

- Geometry and scene suites passed: 59 geometry tests and 192 scene tests, 5 ignored. This includes the aligned overhead-only saved-ground regression and injected fitting-error coverage for the production fallback handler.
- `cargo test --locked --workspace --no-fail-fast`: completed with 3,038 passed, 2 failed and 114 ignored. The benchmark environment names initially failed `omsi-cfg::flags::tests::flags_match_source`; they were renamed to `TERRAIN_BENCH_*`, and that test passed on a focused rerun. The outstanding failure is the unchanged `omsi-platform::updater::tests::a_broken_download_goes_on_where_it_stopped`: its loopback server receives HTTP 403 through the sandbox proxy. `NO_PROXY` does not override the updater's explicitly selected proxy. The normal-environment result still needs upstream CI.
- `cargo build --locked --release` with `CARGO_PROFILE_RELEASE_DEBUG=0`: passed on Linux (Rust 1.99.1).
- The opt-in tile-preparation benchmark passed a synthetic smoke test with automatic tile selection and seven measurement rows with stable vertex counts. This is harness validation, not a measurement of an installed map.
- Architecture guard, source-size budget and whitespace checks passed. No existing tests or assertions were disabled; the installed-map benchmark is opt-in.
- [Release on the new head `7123267`](https://github.com/openOMSI-org/openOMSI/actions/runs/40010326614) and all other checks report `action_required`. They need upstream maintainer approval; this contributor account has no upstream write/admin permission.

Stock-map screenshots and real-map timing/count CSVs are still pending. The existing Map 400 images below are retained. [Screenshot checklist and measurement procedure](https://github.com/p3taomsi/openOMSI/blob/fix/aligned-spline-terrain-v10/docs/testing/aligned-spline-terrain.md).

## Screenshots

Before:
<img width="1920" height="1080" alt="209241~1" src="https://github.com/user-attachments/assets/5f2fba4d-414c-40e1-a0d2-8f18db733e98" />
<img width="1920" height="1080" alt="2049AE~1" src="https://github.com/user-attachments/assets/654347df-4460-4641-8ef2-1a4d6452ae63" />

After:
<img width="1920" height="1080" alt="208518~2" src="https://github.com/user-attachments/assets/604f9ecd-2894-4f3f-be2c-a0dfa457f83f" />
<img width="1920" height="1080" alt="20A5E0~2" src="https://github.com/user-attachments/assets/4f245666-d9cb-4920-a213-30815744ff49" />

## What does this change, and why?

- Restore the pre-PR projected spline cuts and their soil walls when fitting returns an error, rather than displaying plain uncut ground over the ditch.
- Fit a complete terrain backing to the rendered spline section, retaining the floor and terminal extensions rather than cutting an alpha hole.
- Constrain a Delaunay triangulation by the saved terrain grid, spline backing, and graphical ground surfaces. Quantize at rendered vertex precision and planarize constraints before insertion to avoid degenerate triangles and intersecting-constraint failures.
- Blend displacement relative to saved ground, with a local tangent correction and independently bounded terminal transitions. This preserves the surrounding hill instead of extrapolating a spline plane across a cliff.
- Include visible asphalt without traffic paths when bounding new fill; those surfaces were missed by a drivable-only selection.
- Evaluate normals from the same height field with neighboring terrain available at tile edges.

Saved terrain files and physical terrain heights are unchanged. Object/tunnel cutters keep their explicit holes. The `spade` dependency provides constrained triangulation. Tests use small synthetic fixtures; no original OMSI 2 assets, user maps, generated builds, or caches are included.

## What could break?

Tile preparation now triangulates additional geometry and samples the local blend, which can increase loading time, vertex count, and memory on spline-heavy maps. Dense overlaps, unusual authored cutters, non-monotone sections, tile streaming, and interactions with nearby road/footway meshes deserve review. Stock maps and mods, Windows/macOS/Android runtime rendering, VR, and actual gameplay have not been comprehensively tested on this upstream branch. Online protocols, saved-file formats, and settings formats are unchanged.

## Performance

Real-map numbers are pending. `scripts/benchmark-terrain.ps1` compares the pre-PR base `caf95e8a` with the current source using release builds, one warm-up and seven measured preparations of one spline-heavy tile per map. It chooses the tile with the most saved alignment records by default, or accepts explicit coordinates. It exports preparation times, terrain vertex counts and triangle counts to CSV, and prints median/min/max time.

The measurement covers CPU tile preparation, including required neighbors, placement, fitting, cutting and wanted texture decoding. It excludes map opening, GPU upload and rendering. Both variants use one Rayon thread, warm filesystem caches and fresh scene caches. The script copies only the measurement test into an isolated baseline worktree, leaving the baseline terrain implementation unchanged. Use it once on Berlin-Spandau and once on Map 400, with the same date and content for before/after; see the [procedure](https://github.com/p3taomsi/openOMSI/blob/fix/aligned-spline-terrain-v10/docs/testing/aligned-spline-terrain.md).

Peak memory, frame times and actual gameplay performance remain unmeasured. Synthetic geometry checks do not establish those results.

## Checklist

- [ ] `cargo test --workspace` and `cargo build --release` pass (local workspace updater/proxy failure; CI confirmation pending)
- [ ] Upstream CI has run successfully on the current head
- [ ] Stock-map before/after screenshots cover a slope, bridge approach, cutting and tile border
- [x] The pull request contains a single change
- [x] Before/after screenshots are in the description if anything changes on screen
- [ ] Before/after tile-preparation times and terrain vertex counts are attached for Berlin-Spandau and Map 400
- [x] ~~New or changed interface texts are in `crates/omsi-app/locales/app.yml`, in every language I can~~ No interface texts changed
- [x] Online play, saves, settings and mods made for the previous release still work, or the description says what changes
- [x] No original OMSI 2 content and no file of a megabyte or more in the commits

