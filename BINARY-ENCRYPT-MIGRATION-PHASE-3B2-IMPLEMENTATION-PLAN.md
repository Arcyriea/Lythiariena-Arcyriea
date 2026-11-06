# Phase 3B.2 — Full Game SVG Migration and Development Asset Rebuild Plan

**Status:** Planning only. This document does not authorize implementation, copying assets, changing source mappings, assigning additional bundle IDs, generating bundles, changing Git staging, or committing. Wait for explicit approval before starting Phase 3B.2.

**Audit date:** 2026-11-06  
**Authoritative specifications:** [Phase 3A specification](./BINARY-ENCRYPT-MIGRATION-PHASE-3A-PLAN-WORKING.md) and [Phase 3B implementation plan](./BINARY-ENCRYPT-MIGRATION-PHASE-3B-IMPLEMENTATION-PLAN.md).

## 1. Verified repository and Phase 3B.1 status

### Git state at planning start

- Root: `C:/GitHub/Redemptis-Tiariea`
- Remote: `origin https://github.com/Arcyriea/Redemptis-Tiariea.git`
- Branch: `main`
- HEAD: `405ec63a7cb0d322b559f029a03d4734dddd8937`
- Worktree was already dirty. Staged and unstaged files were inspected before planning.

At inspection time, **only** the following generated Phase 3B.1 outputs were staged:

- `public/assets/Ships/ships.bin`
- `public/assets/asset-manifest.json`
- `public/assets/asset-manifest-schema.json`

The approved Phase 3B implementation plan was ignored and was **not staged** in the current index, despite a previous report that it had been. This plan's requested path did not exist and is also ignored by `.gitignore`'s `docs/*` rule. No Git staging operation is part of this planning task.

Pre-existing unstaged modifications:

- `.gitignore`
- `package.json`
- `scripts/add-svg-copyright.mjs`
- `scripts/generate-public-svg-manifest.mjs`

Pre-existing untracked files:

- `docs/BINARY-ENCRYPT-MIGRATION.md`
- `scripts/asset-registry.json`
- `scripts/asset-registry.mjs`
- `scripts/asset-roots.mjs`
- `scripts/package-assets.mjs`
- `scripts/prepare-assets.mjs`
- `tests/add-svg-copyright.test.ts`
- `tests/package-assets.test.ts`

The ignored `private/` tree currently contains ten ship SVGs only. The existing migration specifications and other ignored documentation are not tracked. All pre-existing staged, unstaged, untracked, and ignored work must be preserved; do not reset, clean, restore, stage, unstage, or commit it.

### Phase 3B.1 acceptance audit

| Requirement | Verified state |
|---|---|
| Authoritative registry contains 31 game SVGs under `assets/` | Pass: 19 Components, 10 Ships, 2 Turrets-Platforms |
| Initial `ships1` bundle contains exactly the ten intended ships | Pass: ten registry assignments; generated manifest contains ten assets |
| Other 21 registered game SVGs remain unbundled | Pass: 19 Components and 2 Turrets-Platforms have no `bundleId` |
| Existing texture keys and all three aliases | Pass: all 31 registry identities match the 31 generated TypeScript entries; exact aliases are `central-hull`, `alien-interceptor`, and `human-dreadnought` on their approved assets |
| Minimap and Vite scaffold assets excluded | Pass: none are registered or included in the RTAB manifest |
| Existing Phaser loading unchanged | Pass: `src/main.ts` still loads `publicSvgTextures` using `this.load.svg(asset.key, asset.url, ...)`; `src/main.ts` and `src/ui/MiniMap.ts` have no current diff |
| Private-source lifecycle states | Pass: missing `private/` takes read-only validation; present complete sources package; present but incomplete private ships fail without public-ship fallback |
| RTAB v1.0 and fingerprints | Pass: current read-only output validator accepts the schema, manifest, and `ships1`; manifest and bundle SHA-256 are both `5a88e030dfbd0cecc8d2b51acf7e80dce2bdb36862838ba0ea113d84707fd65b`; bundle size is 111,817 bytes |
| Focused packaging and manifest tests | Pass: 18/18 |
| Full current test suite | 142 passed, 2 failed, 144 total. The same two existing failures remain in `tests/phase2a-taxonomy.test.ts`: legacy Skill/Passive/Spawn compatibility and PlayerCannon behavior. This matches the previously recorded Phase 3B.1 baseline; no new failure was observed. |

**No Phase 3B.1 defect against its approved scope was found in this audit.** The current runtime remains loose-SVG based; this is intentional, not a Phase 3B.1 defect. The test suite is not fully green because of the two known unrelated baseline failures.

The current inventory checks also confirmed that all 31 public `/assets` SVG copies exist. The ten private ship copies match their public counterparts byte-for-byte by SHA-256. No public asset was modified or removed for this audit.

### Current implementation responsibilities

- `scripts/asset-registry.json` is the authoritative 31-asset identity/key/alias registry; `ships1` is the only assigned bundle.
- `scripts/asset-roots.mjs` maps the logical assets to current canonical physical roots.
- `scripts/package-assets.mjs` reads canonical sources, packages only assigned assets, emits the RTAB manifest/schema, and validates generated/committed outputs.
- `scripts/prepare-assets.mjs` selects package or read-only validation mode and publishes fully prepared output files.
- `scripts/generate-public-svg-manifest.mjs` emits all 31 loose-runtime entries from registry metadata and canonical source resolution.
- `src/generated/publicSvgManifest.ts` still has all 31 existing public URLs, keys, and aliases.
- `scripts/svg-ownership.json` continues to use `public/<logicalPath>` ownership keys. The copyright script resolves ownership by stable logical path, so moving a canonical physical source does not by itself require changing those keys.

## 2. Complete current inventory and source mapping

Only the 31 SVGs under `assets/` are in this migration. Current source paths and sizes below were read from disk. Sizes are raw SVG bytes. A future target is always `private/assets/<path relative to the `assets/` logical root>`.

| Asset ID | Logical path | Current canonical source | Bytes | Current state |
|---|---|---|---:|---|
| `rt.component.aistiaria.side-hull-angular` | `assets/Components/Aistiaria/01_side_hull_angular.svg` | `public/assets/Components/Aistiaria/01_side_hull_angular.svg` | 13,155 | Public; migrate |
| `rt.component.aistiaria.hex-cockpit` | `assets/Components/Aistiaria/02_hex_cockpit.svg` | `public/assets/Components/Aistiaria/02_hex_cockpit.svg` | 6,930 | Public; migrate |
| `rt.component.aistiaria.side-wing` | `assets/Components/Aistiaria/03_side_wing.svg` | `public/assets/Components/Aistiaria/03_side_wing.svg` | 4,040 | Public; migrate |
| `rt.component.aistiaria.rear-engine-pod` | `assets/Components/Aistiaria/04_rear_engine_pod.svg` | `public/assets/Components/Aistiaria/04_rear_engine_pod.svg` | 4,343 | Public; migrate |
| `rt.component.aistiaria.nose-section` | `assets/Components/Aistiaria/05_nose_section.svg` | `public/assets/Components/Aistiaria/05_nose_section.svg` | 1,952 | Public; migrate |
| `rt.component.aistiaria.swept-wing` | `assets/Components/Aistiaria/06_swept_wing.svg` | `public/assets/Components/Aistiaria/06_swept_wing.svg` | 6,430 | Public; migrate |
| `rt.component.aistiaria.connector-module` | `assets/Components/Aistiaria/07_connector_module.svg` | `public/assets/Components/Aistiaria/07_connector_module.svg` | 2,250 | Public; migrate |
| `rt.component.aistiaria.tail-fuselage` | `assets/Components/Aistiaria/08_tail_fuselage.svg` | `public/assets/Components/Aistiaria/08_tail_fuselage.svg` | 2,121 | Public; migrate |
| `rt.component.yutiarin.part-1` | `assets/Components/Yutiarin/Part 1.svg` | `public/assets/Components/Yutiarin/Part 1.svg` | 7,058 | Public; migrate |
| `rt.component.yutiarin.part-10` | `assets/Components/Yutiarin/Part 10.svg` | `public/assets/Components/Yutiarin/Part 10.svg` | 1,456 | Public; migrate |
| `rt.component.yutiarin.part-2` | `assets/Components/Yutiarin/Part 2.svg` | `public/assets/Components/Yutiarin/Part 2.svg` | 5,780 | Public; migrate |
| `rt.component.yutiarin.part-3` | `assets/Components/Yutiarin/Part 3.svg` | `public/assets/Components/Yutiarin/Part 3.svg` | 1,896 | Public; migrate |
| `rt.component.yutiarin.part-4` | `assets/Components/Yutiarin/Part 4.svg` | `public/assets/Components/Yutiarin/Part 4.svg` | 1,254 | Public; migrate |
| `rt.component.yutiarin.part-5` | `assets/Components/Yutiarin/Part 5.svg` | `public/assets/Components/Yutiarin/Part 5.svg` | 1,313 | Public; migrate |
| `rt.component.yutiarin.part-6` | `assets/Components/Yutiarin/Part 6.svg` | `public/assets/Components/Yutiarin/Part 6.svg` | 1,360 | Public; migrate |
| `rt.component.yutiarin.part-7` | `assets/Components/Yutiarin/Part 7.svg` | `public/assets/Components/Yutiarin/Part 7.svg` | 1,384 | Public; migrate |
| `rt.component.yutiarin.part-8` | `assets/Components/Yutiarin/Part 8.svg` | `public/assets/Components/Yutiarin/Part 8.svg` | 1,301 | Public; migrate |
| `rt.component.yutiarin.part-9` | `assets/Components/Yutiarin/Part 9.svg` | `public/assets/Components/Yutiarin/Part 9.svg` | 1,295 | Public; migrate |
| `rt.component.yutiarin.wing-1` | `assets/Components/Yutiarin/Wing 1.svg` | `public/assets/Components/Yutiarin/Wing 1.svg` | 9,001 | Public; migrate |
| `rt.ship.yutiarin-kinetic-lance-frigate` | `assets/Ships/Yutiarin Kinetic Lance Frigate.svg` | `private/assets/Ships/Yutiarin Kinetic Lance Frigate.svg` | 40,397 | Already private |
| `rt.ship.alien-interceptor` | `assets/Ships/alien-interceptor.svg` | `private/assets/Ships/alien-interceptor.svg` | 4,382 | Already private |
| `rt.ship.alien-leviathan` | `assets/Ships/alien-leviathan.svg` | `private/assets/Ships/alien-leviathan.svg` | 5,953 | Already private |
| `rt.ship.alien-mothership` | `assets/Ships/alien-mothership.svg` | `private/assets/Ships/alien-mothership.svg` | 4,406 | Already private |
| `rt.ship.curved-corvette` | `assets/Ships/curved-corvette.svg` | `private/assets/Ships/curved-corvette.svg` | 4,774 | Already private |
| `rt.ship.faceted-cruiser` | `assets/Ships/faceted-cruiser.svg` | `private/assets/Ships/faceted-cruiser.svg` | 5,329 | Already private |
| `rt.ship.human-dreadnought` | `assets/Ships/human-dreadnought.svg` | `private/assets/Ships/human-dreadnought.svg` | 9,714 | Already private |
| `rt.ship.human-frigate` | `assets/Ships/human-frigate.svg` | `private/assets/Ships/human-frigate.svg` | 5,280 | Already private |
| `rt.ship.void-dreadnought` | `assets/Ships/void-dreadnought.svg` | `private/assets/Ships/void-dreadnought.svg` | 7,214 | Already private |
| `rt.ship.yutiarin-starblade-destroyer` | `assets/Ships/yutiarin-starblade-destroyer.svg` | `private/assets/Ships/yutiarin-starblade-destroyer.svg` | 23,262 | Already private |
| `rt.turret.yutiarin-defense-turret` | `assets/Turrets-Platforms/Yutiarin-Defense Turret.svg` | `public/assets/Turrets-Platforms/Yutiarin-Defense Turret.svg` | 10,875 | Public; migrate |
| `rt.turret.yutiarin-perimeter-turret` | `assets/Turrets-Platforms/Yutiarin-Perimeter Turret.svg` | `public/assets/Turrets-Platforms/Yutiarin-Perimeter Turret.svg` | 17,085 | Public; migrate |

The 21 remaining files total **102,279 source bytes**: Components total 74,319 bytes and Turrets-Platforms total 27,960 bytes. The ten ship sources total 110,711 bytes. These measurements are an audit snapshot; implementation must re-inventory and remeasure rather than rely on stale sizes or hashes.

### Explicit 21-file migration map

Each public source maps to the mirrored private target shown below. Preserve exact filename case, spaces, and directory structure.

| Current public source | Future private target | Bytes |
|---|---|---:|
| `public/assets/Components/Aistiaria/01_side_hull_angular.svg` | `private/assets/Components/Aistiaria/01_side_hull_angular.svg` | 13,155 |
| `public/assets/Components/Aistiaria/02_hex_cockpit.svg` | `private/assets/Components/Aistiaria/02_hex_cockpit.svg` | 6,930 |
| `public/assets/Components/Aistiaria/03_side_wing.svg` | `private/assets/Components/Aistiaria/03_side_wing.svg` | 4,040 |
| `public/assets/Components/Aistiaria/04_rear_engine_pod.svg` | `private/assets/Components/Aistiaria/04_rear_engine_pod.svg` | 4,343 |
| `public/assets/Components/Aistiaria/05_nose_section.svg` | `private/assets/Components/Aistiaria/05_nose_section.svg` | 1,952 |
| `public/assets/Components/Aistiaria/06_swept_wing.svg` | `private/assets/Components/Aistiaria/06_swept_wing.svg` | 6,430 |
| `public/assets/Components/Aistiaria/07_connector_module.svg` | `private/assets/Components/Aistiaria/07_connector_module.svg` | 2,250 |
| `public/assets/Components/Aistiaria/08_tail_fuselage.svg` | `private/assets/Components/Aistiaria/08_tail_fuselage.svg` | 2,121 |
| `public/assets/Components/Yutiarin/Part 1.svg` | `private/assets/Components/Yutiarin/Part 1.svg` | 7,058 |
| `public/assets/Components/Yutiarin/Part 10.svg` | `private/assets/Components/Yutiarin/Part 10.svg` | 1,456 |
| `public/assets/Components/Yutiarin/Part 2.svg` | `private/assets/Components/Yutiarin/Part 2.svg` | 5,780 |
| `public/assets/Components/Yutiarin/Part 3.svg` | `private/assets/Components/Yutiarin/Part 3.svg` | 1,896 |
| `public/assets/Components/Yutiarin/Part 4.svg` | `private/assets/Components/Yutiarin/Part 4.svg` | 1,254 |
| `public/assets/Components/Yutiarin/Part 5.svg` | `private/assets/Components/Yutiarin/Part 5.svg` | 1,313 |
| `public/assets/Components/Yutiarin/Part 6.svg` | `private/assets/Components/Yutiarin/Part 6.svg` | 1,360 |
| `public/assets/Components/Yutiarin/Part 7.svg` | `private/assets/Components/Yutiarin/Part 7.svg` | 1,384 |
| `public/assets/Components/Yutiarin/Part 8.svg` | `private/assets/Components/Yutiarin/Part 8.svg` | 1,301 |
| `public/assets/Components/Yutiarin/Part 9.svg` | `private/assets/Components/Yutiarin/Part 9.svg` | 1,295 |
| `public/assets/Components/Yutiarin/Wing 1.svg` | `private/assets/Components/Yutiarin/Wing 1.svg` | 9,001 |
| `public/assets/Turrets-Platforms/Yutiarin-Defense Turret.svg` | `private/assets/Turrets-Platforms/Yutiarin-Defense Turret.svg` | 10,875 |
| `public/assets/Turrets-Platforms/Yutiarin-Perimeter Turret.svg` | `private/assets/Turrets-Platforms/Yutiarin-Perimeter Turret.svg` | 17,085 |

Excluded from both registry and migration: `public/minimap/Capital-Ship.svg`, `public/minimap/Corvette.svg`, `public/minimap/Fighter.svg`, `public/favicon.svg`, and `public/icons.svg`. No PNG or other non-SVG resource is in scope.

## 3. Destination layout and canonical source-root change

The completed private game-source tree should mirror the logical asset hierarchy:

```text
private/
  assets/
    Ships/                          # 10 existing private sources
    Components/
      Aistiaria/                    # 8 migrated SVGs
      Yutiarin/                     # 11 migrated SVGs
    Turrets-Platforms/              # 2 migrated SVGs
```

The source map in `scripts/asset-roots.mjs` currently has a broad private `assets/` root plus more-specific public `Components/` and `Turrets-Platforms/` roots. The longest-prefix rule intentionally leaves those 21 assets public during the pilot. Only after every destination copy is present and hash-verified should Phase 3B.2 remove the public category roots so all 31 `assets/...` logical paths resolve from `private/assets/`. Keep the minimap root and non-recursive public root mapping. Do not change any logical path, registry ID, texture key, alias, or runtime URL.

After mapping changes, verify that source discovery returns the expected 31 private game SVGs plus the three public minimap SVGs and two public root SVGs, with no duplicate logical paths. Public loose copies remain at their current paths for Phaser's existing `/assets/...` URLs.

## 4. Safe copy, hash, collision, and rollback procedure

Implementation should provide a repeatable migration utility with a non-mutating inventory/dry-run mode and an explicit apply switch. It must not be run until Phase 3B.2 is approved.

1. Re-read the registry and enumerate the full public game-SVG inventory and the entire existing `private/assets/` tree. Validate exact case-sensitive relative paths, duplicate paths under Windows case-insensitive comparison, file types, and unknown `.svg` files. Report missing sources, unregistered assets, case mismatches, and target collisions before copying.
2. Verify all ten existing private ships against their public mirrors. Recompute SHA-256 and size for all 21 public sources immediately before copying; do not embed this audit's snapshot hashes as future truth.
3. Inspect each intended private destination. If it is absent, mark it as a new destination. If it exists with exact expected case and identical bytes, treat it as already verified without overwriting. If it exists with different bytes, is a directory/symlink/reparse point, or differs only by path case, stop for user resolution. Never overwrite an existing user-owned private file.
4. Copy only the 21 registered SVGs into a unique staging area under `private/` but outside `private/assets/`, so the staging area cannot be mistaken for canonical sources and stays on the same volume. Do not stage under `public/`.
5. Hash every staged copy and require exact equality with its corresponding public source. Recheck source size/hash before publishing to catch concurrent edits. Reject short reads, unexpected files, path traversal, and case-insensitive destination collisions.
6. Publish only verified files using no-overwrite semantics. If any install step fails, do not change source-root mapping or registry bundle assignments. Preserve the public originals and all pre-existing private files. Clean only the migration run's temporary staging directory. Any newly created, verified target copies may safely remain and be treated as identical pre-existing copies on retry; do not recursively clean `private/assets/`.
7. Once all 31 private files verify, update the source-root mapping. Run the resolver against every registered asset and require all canonical paths to be the exact mirrored private paths. Regenerate the TypeScript manifest through the normal preparation flow and compare its content to the pre-migration version: 31 entries, same texture keys, three aliases, and same encoded `/assets/...` URLs.
8. If mapping or generation fails, restore only the prior source-root mapping if it still matches the value written by this run. Keep both source trees; never delete original public sources or pre-existing private files. Do not update bundle assignments until a grouping has been explicitly approved.
9. Run a final private/public hash comparison for all 31, inspect the complete private directory for missing/unregistered game SVGs, validate generated bundles/manifests, and record results before reporting Phase 3B.2 complete.

No phase in this procedure removes public sources. Phase 3B.2 ends with all 31 canonical source paths private while the 31 loose public runtime copies remain intact.

## 5. Bundle grouping evaluation

Current code in `src/main.ts` eagerly queues every `publicSvgTextures` entry during `BootScene.preload()` and registers aliases during `create()`. No category-specific bundle loader or selective asset loading was found. Consequently, separate bundles will not reduce current startup payload if Phase 3C loads every declared required bundle. Separate groups can isolate updates/cache invalidation, but actual update cadence is not yet established. Recent Git history shows several asset-related commits from 2026-10-20 through 2026-11-03; it does not provide enough stable per-category frequency data to justify a final split. There is no current mod loader or optional-content contract.

The sizes below are calculated from the actual current asset bytes and IDs using the approved uncompressed RTAB v1 formula: 64-byte header + 28-byte index header + 72 bytes per record + UTF-8 asset ID bytes + 32-byte index digest + raw SVG payload bytes. They are exact for the inspected inputs and current IDs, not gzip estimates; future source changes will change them. Candidate rows are alternatives, not additive assignments.

| Candidate grouping | Assets | Raw SVG bytes | RTAB bytes at audit snapshot | Runtime/loading and maintenance rationale |
|---|---:|---:|---:|---|
| Existing `ships1` (retain) | 10 | 110,711 | 111,817 actual | Already approved and generated. Keep separate unless the user approves an explicit replacement/rename. |
| `components1` | 19 | 74,319 | 76,407 | Straightforward content grouping; isolates component changes, but current runtime still loads it at startup with the other assets. |
| `turrets-platforms1` | 2 | 27,960 | 28,296 | Small domain group and independently cacheable; no current selective-load behavior or update-frequency evidence supports making it separate. |
| `nonships1` (all Components + Turrets-Platforms) | 21 | 102,279 | 104,579 | Fewer requests than two new bundles and matches the current eager-load model; a change to any one asset invalidates the combined bundle. |
| `components-aistiaria1` | 8 | 41,221 | 42,208 | Possible faction/content split; no current runtime load boundary or evidence of independent update cadence. |
| `components-yutiarin1` | 11 | 33,098 | 34,323 | Possible faction/content split; same limitation as above. |
| One bundle for all 31 game assets | 31 | 212,990 | 216,272 | Lowest request count, but would absorb/replace the existing `ships1` boundary and increase unrelated bundle invalidation; not recommended without explicit approval. |

**Recommendation for discussion, not an assignment:** retain `ships1`; compare either one new `nonships1` bundle against separate `components1` and `turrets-platforms1`. The current runtime gives no evidence that one option improves startup. Bundle grouping and every `bundleId` beyond the existing ten `ships1` entries remain undecided until user approval. Do not change registry assignments based only on folder names.

## 6. Development asset watcher evaluation

The watcher is **not part of Phase 3B.2**. Keep the migration deterministic and bounded; treat automatic rebuild/watch behavior as a separate optional development enhancement (for example, Phase 3B.3) with its own approval and tests.

If approved later, the watcher should:

- Watch only `private/assets/**` and `scripts/asset-registry.json`; ignore public loose SVGs, generated bundles/manifests, `dist/`, and unrelated files.
- Use a Windows-tested watcher implementation with recursive directory support and rename event normalization. No watcher dependency is currently declared; prefer a maintained library such as Chokidar as a development dependency rather than relying on inconsistent native recursive `fs.watch` behavior on Windows.
- Debounce event bursts with a quiet interval and a maximum delay. Serialize packaging: never run two publishers concurrently. If events arrive during a build, mark the run dirty and schedule one coalesced follow-up after the current run.
- On startup and after relevant events, compare the exact case-preserving private SVG inventory with the registered private logical paths. Report unknown SVGs as unregistered; never auto-add them or infer IDs/keys/bundle assignments. A deleted/renamed registered file must fail source validation, not generate a reduced bundle.
- Re-read/validate the registry and all required canonical inputs; build every output in staging; recheck source hashes and watcher generation before publication. Publish only a complete validated set using the existing rollback-capable publisher (or an equivalent tested transaction). On any error, log the asset/path and reason clearly, keep the last valid bundle/manifest, and remain able to process later corrections.
- Handle case-only rename, duplicate notifications, file replacement/write-in-progress, registry edits, missing paths, and transient Windows sharing violations explicitly. Do not silently downgrade errors or publish partial output.
- Close/dispose watchers and await or cancel pending work on `SIGINT`/`SIGTERM`; do not leave background handles keeping Node alive.

**RTAB rebuild is not visual hot reload.** Before Phase 3C, Phaser continues to load loose public SVG URLs via `this.load.svg()`. A watcher may refresh `.bin` bundles and generated metadata, but the running game's displayed artwork will not change from those bundle writes. Do not claim or implement runtime texture hot reload as part of this watcher.

## 7. Planned Phase 3B.2 files and responsibilities

| File | Planned action | Scope/constraint |
|---|---|---|
| `scripts/migrate-assets-private.mjs` | Add a safe inventory, dry-run, hash-verified copy, and no-overwrite apply workflow | Only registered game SVGs; report existing targets; no public deletion; no automatic registry/bundle changes |
| `scripts/asset-roots.mjs` | After all 21 copies verify, remove public Components and Turrets-Platforms canonical roots | Keep private `assets/`, minimap, and root-level public mapping; preserve public runtime URLs |
| `scripts/asset-registry.json` | Add bundle assignments only after grouping approval | Keep all asset IDs, paths, texture keys, aliases, and content types unchanged |
| `scripts/package-assets.mjs` | Reuse for expanded output; change only if full-coverage validation requires it | RTAB v1.0 binary contract and fingerprints remain unchanged |
| `scripts/prepare-assets.mjs` | Reuse source-aware lifecycle after root-map update; adjust only if full-source preparation needs a tested fix | Still fail on missing canonical source; preserve read-only clean-clone validation |
| `scripts/generate-public-svg-manifest.mjs` | Reuse registry-driven generation through the updated resolver | Result must preserve all existing 31 runtime entries byte-for-byte if rendering is unchanged |
| `scripts/svg-ownership.json` | No expected change | Ownership entries remain keyed to `public/<logicalPath>`; verify copyright tooling still resolves stable logical identities |
| `public/assets/asset-manifest.json`, schema, and approved `.bin` files | Regenerate after bundle groups and assignments are approved | Generated outputs only; preserve RTAB v1.0 and only include assigned bundle assets |
| `tests/migrate-assets-private.test.ts` | Add migration preflight/copy/hash/collision/rollback coverage | Use isolated temporary repositories; never mutate real `private/` or `public/` |
| `tests/package-assets.test.ts`, `tests/public-svg-manifest.test.ts` | Extend for all-31 private sources and full manifest invariance | Keep 3B.1 lifecycle cases and stable aliases covered |
| `src/generated/publicSvgManifest.ts` | Generated verification only; expected bytes unchanged | Do not change keys, aliases, or URLs |
| `src/main.ts`, `src/ui/MiniMap.ts`, public loose SVGs | No change in Phase 3B.2 | No bundle-backed runtime work or public-source deletion |
| `package.json` | No change expected for migration itself | Watcher script/dependency is deferred to separate approval |

Implementation must inspect and integrate the already dirty/untracked files instead of replacing them. The file plan is bounded to migration and packaging; any discovered need for unrelated changes requires a separate explanation before proceeding.

## 8. Phase 3B.2 tests and acceptance criteria

### Migration tests

- Dry-run inventory exactly matches all 21 public-to-private mappings and reports bytes/hashes without writing.
- Apply copies only the 21 registered files; every copied SHA-256 equals the corresponding public source hash; the ten existing private ships remain byte-identical.
- Missing public input, unreadable file, changed-during-copy input, incorrect hash, case mismatch, case-insensitive collision, unregistered private SVG, symlink/reparse point, non-file target, or differing pre-existing private target fails before source mapping changes.
- Identical pre-existing private targets are accepted without overwriting. Conflicting user-owned files are never replaced or deleted.
- Injected mid-copy/publish failure leaves public originals and pre-existing private files intact, leaves source mapping unchanged, cleans only the run's staging directory, and reports any newly installed verified copies for safe retry.
- The final private inventory has exactly the 31 expected registered game SVGs; no extra private game SVG is silently packaged. Non-SVG files are not copied.

### Mapping, identity, manifest, and packaging tests

- All 31 registry logical paths resolve exactly to `private/assets/<relative path>` after source-map change; source discovery has 36 total SVGs with 31 game, 3 minimap, 2 root-level entries and no duplicates.
- All 31 private game sources match their public originals by SHA-256. Public loose copies remain present and byte-identical until Phase 3C.
- `src/generated/publicSvgManifest.ts` remains byte-for-byte unchanged: same 31 keys, encoded public URLs, and exact three aliases, including `central-hull`.
- Registry IDs, logical paths, texture keys, aliases, and content types do not change. Minimap, favicon, icons, PNG, and other non-SVG resources remain outside migration and RTAB packaging.
- Only user-approved bundle assignments appear in the generated manifest. Repeated packaging from unchanged inputs is byte-identical.
- Validate every generated RTAB v1.0 header, index, range, ordering, SVG payload, source/payload/index/bundle fingerprint, manifest, and schema; validate clean-clone mode with `private/` absent and confirm it is read-only.
- Existing lifecycle behavior remains: missing entire private root validates committed outputs read-only; a present complete private root packages; a present but incomplete private tree fails without public fallback or output replacement.

### Project checks and release gate

- Run focused migration, registry, packager, manifest, and lifecycle tests.
- Run `npm test`; report the two known taxonomy baseline failures separately and introduce no additional failures.
- Run `npx tsc --noEmit`, `npm run predev`, and `npm run build` only after verified mapping and approved bundle assignments are in place. These commands may publish generated files, so they are not planning-task validation commands.
- Inspect final `git status`, staged and unstaged diffs without rearranging them. No public loose SVG may be deleted during 3B.2.
- Report a complete changed-file list, per-asset source/copy hash result, bundle IDs/counts/sizes/hashes, test and build results, and any remaining failures. Stop for explicit Phase 3B.2 review and approval before Phase 3C.

## 9. Phase boundaries, Git preservation, and open decisions

### Phase 3B.1 — completed Ships pilot

Keep the current ten-ship bundle, 31-entry registry/runtime manifest, RTAB generation and source-aware lifecycle. Do not alter loose Phaser loading.

### Phase 3B.2 — full private source migration and expanded packaging

Copy and verify the remaining 21 sources, update canonical roots only after every copy passes, approve grouping and assignments, generate/validate expanded RTAB outputs, and retain every loose public game SVG. No Phaser bundle integration and no public SVG deletion.

### Phase 3C — runtime migration

Only after explicit Phase 3B.2 approval, implement asynchronous RTAB bootstrap, integrity validation, SVG extraction, and texture/alias registration before gameplay. Verify every game asset and fail bootstrap explicitly on required-bundle errors. Remove public game SVG copies only after end-to-end Phase 3C verification. Keep minimap and Vite scaffold assets outside all stages.

### Git and commit policy

- Phase 3B.2 implementation must not change staging or commit. Preserve the three currently staged generated outputs and every existing unstaged/untracked change exactly as found.
- `private/` remains ignored; its sources are not staged or committed unless the user explicitly changes that policy.
- This planning document is ignored by `.gitignore`'s `docs/*` pattern. At creation it was absent and untracked; do not stage it during this task.
- The user intends to commit only after the complete binary migration is finished. Make no commit during Phase 3B.2 or Phase 3C; after each approval gate, report current index/worktree state and wait for direction.

### Decisions requiring explicit approval

1. Choose the additional bundle grouping and per-asset `bundleId` assignments. The measured candidate sizes above support comparison but do not decide membership.
2. Decide whether to pursue a separate development-only watcher after the source migration. Recommendation: defer it from Phase 3B.2; no watcher or runtime hot reload is included in this plan's immediate implementation.
