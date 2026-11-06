# Redemptis-Tiariea Project Handoff

**Document purpose:** Transfer the accumulated project context into a new ChatGPT conversation.  
**Prepared:** 2026-11-06  
**Project repository:** [Arcyriea/Redemptis-Tiariea](https://github.com/Arcyriea/Redemptis-Tiariea)  
**Planning/documentation repository:** [Arcyriea/Lythiariena-Arcyriea](https://github.com/Arcyriea/Lythiariena-Arcyriea)  
**Status:** Context handoff only. This document does not authorize repository edits, Git operations, implementation, or Phase 3C.

---

## 1. Project overview

### 1.1 Vision and game direction

Redemptis-Tiariea is a Phaser 4 2D space-combat / fleet-oriented game project. Its broad design references include **Starsector**, **Battleship Forever**, and **ShellCore Command**, but the user explicitly wants the game to become its own design rather than a clone of any of them.

Important design direction:

- Modular ship construction is a central feature, with visible ship parts, hardpoints, and modular hull extensions.
- Fleet-scale space gameplay and multiple entity layers are planned.
- The project is currently heavily AI-assisted. The user is actively learning Phaser 4 concepts while building the project, especially the camera/rendering model; do not assume the user has already implemented or deeply mastered every engine subsystem.
- The user has considered small lore-related mini-games as a possible way to build practical Phaser experience, but no particular mini-game has been approved or implemented in the project context captured here.

### 1.2 Technology stack

Latest known project toolchain from the conversation:

- **Phaser:** `^4.2.1` — Phaser 4 only; do not introduce Phaser 3 APIs or architecture.
- **Vite:** `^8.3.0`.
- **Runtime/language:** TypeScript, JavaScript ES modules (`"type": "module"`).
- **Package scripts:** `predev` and `prebuild` prepare/validate assets before Vite dev/build.
- **Testing:** `tsx --test`, invoked through `npm test`.
- **Other technologies in the project:** custom GLSL shaders, SVG assets, RTAB `.bin` asset bundles, and generated JSON manifests/schemas.

The project is a browser-based Phaser game at the current phase. Phase 3C runtime loading of RTAB bundles has **not** begun.

### 1.3 Repository and environment

Implementation repository:

- GitHub: `https://github.com/Arcyriea/Redemptis-Tiariea`
- Local Windows path: `C:\GitHub\Redemptis-Tiariea`
- Remote: `origin https://github.com/Arcyriea/Redemptis-Tiariea.git`
- Latest known branch: `main`
- Latest known HEAD: `405ec63a7cb0d322b559f029a03d4734dddd8937`

Planning/documentation repository:

- GitHub: `https://github.com/Arcyriea/Lythiariena-Arcyriea`
- Some migration and planning documents are stored here rather than in the implementation repository.
- The implementation repository's `.gitignore` ignores most of `docs/*`, so a file existing in the other repository or in a local ignored docs directory must not be assumed to be tracked in Redemptis-Tiariea.

The latest HEAD/status above comes from the Phase 3B.3 Copilot planning response and the immediately preceding audit output. It has not been rechecked locally in this handoff operation. Always inspect the real repository state before implementation.

### 1.4 High-level repository areas

Paths and files that have featured in the work:

```text
Redemptis-Tiariea/
├── docs/                         # Mostly gitignored planning / architecture documents
├── public/
│   ├── assets/                   # Public loose SVGs plus generated RTAB bundles/manifests
│   └── minimap/                  # Minimap SVGs; separate from game asset registry
├── private/
│   └── assets/                  # Canonical private source tree; intentionally gitignored
│       ├── Ships/
│       ├── Components/
│       │   ├── Aistiaria/
│       │   └── Yutiarin/
│       └── Turrets-Platforms/
├── scripts/
│   ├── asset-registry.json
│   ├── asset-registry.mjs
│   ├── asset-roots.mjs
│   ├── migrate-assets-private.mjs
│   ├── package-assets.mjs
│   ├── prepare-assets.mjs
│   ├── generate-public-svg-manifest.mjs
│   └── add-svg-copyright.mjs
├── src/
│   ├── generated/publicSvgManifest.ts
│   ├── shaders/                  # e.g. planets.glsl?raw
│   └── ... Phaser scenes, managers, entities and UI
├── tests/
│   ├── add-svg-copyright.test.ts
│   ├── migrate-assets-private.test.ts
│   ├── package-assets.test.ts
│   └── ... other project tests
├── package.json
└── vite.config.mjs               # Proposed for future watcher; not yet implemented per Phase 3B.3 plan
```

This is a contextual outline, not a verified exhaustive directory listing. Re-read actual source files before relying on any path or API.

---

## 2. Architectural decisions

This section separates explicit user decisions from designs that are only proposals.

### 2.1 Explicit decisions by the user

1. **Phaser 4 only.** Use Phaser 4 APIs and verify any version-sensitive API against Phaser 4. Do not introduce Phaser 3.
2. **Do not clone another game.** Starsector, Battleship Forever, and ShellCore Command are references, not targets to reproduce.
3. **Modular hull extensions are distinct and important.** A modular hull extension normally has built-in structure of its own. It deserves its own mount point on a ship, and each extension's connector points must match the connector type of the ship's installation point. This is intended to be more structured than simply attaching arbitrary components to sprites.
4. **Thruster vents are decorative/functional emission points, not destructible ship parts.** For now, the hull defines where thrust effects emit. In a future combat system, disabling vents in a particular direction should affect turning in that direction only; vents themselves are not destroyed.
5. **Three conceptual render layers:** Background, Foreground, and UI Elements. This is the user's conceptual rendering organization; do not imply it is already implemented as three dedicated Phaser cameras or managers.
6. **Asset source migration is private-first.** Registered game SVG originals are mirrored under `private/assets/`. Public loose SVG copies are retained for the current runtime until Phase 3C.
7. **Folder-driven bundle grouping.** Top-level folders under `private/assets/` determine category bundle membership; nested folders are included recursively within that category.
8. **Three distinct category bundles, not one `nonships.bin`:**
   - `private/assets/Ships/` → `ships.bin`
   - `private/assets/Components/` → `components.bin`
   - `private/assets/Turrets-Platforms/` → `turrets-platforms.bin`
9. **No arbitrary numeric suffixes.** Do not create names such as `ships1.bin`, `components1.bin`, or `turrets-platforms1.bin` as the naming convention. The existing logical registry ID `ships1` is legacy state, not a desired filename convention.
10. **Possible future mod support should use mirrored folder/registry conventions.** Modders may supply their own mirrored asset category folders and corresponding registry metadata. Their assets should eventually be merged into the corresponding category's effective bundle with base assets. This is a future goal, not current implemented capability.
11. **Base-game and mod sources must remain independently owned.** Combined bundles should be generated outputs; the packager must not destructively merge mod source files into base-game private sources.
12. **No commits until the binary migration is complete.** The user wants the packaging/runtime migration finished before committing. Preserve the existing Git index and all unstaged, untracked, and ignored work in the meantime.
13. **Approval gates matter.** The preferred workflow is inspect → plan → report → request explicit approval → implement only the approved scope.

### 2.2 Camera and rendering direction

The user has recently been understanding Phaser's camera system through Redemptis-Tiariea. The described setup includes:

- Gameplay/main camera follows the player and supports zoom around `0.3` to `2.5`.
- Background camera remains at zoom `1.0` and scroll `(0, 0)` for `spaceBackground.graphics`.
- HUD/minimap elements use `scrollFactor(0)` to remain fixed to the screen.
- Planets remain in world space and may use parallax behavior.

These are details previously discussed, but the exact current implementation should be re-inspected before making camera changes. The high-level user concept remains Background / Foreground / UI Elements.

### 2.3 Entity and manager architecture discussed

The project's intended architecture has included:

- `EntityManager` as generic lifecycle/identity management.
- Entity IDs use a prefix-plus-number convention and are not reused.
- Manager stores entities in a `Map` and tracks managed instances with a `WeakSet`.
- `EntityManager` should not absorb movement, rendering, combat, or AI behavior.
- `ShipManager` should extend `EntityManager` and work with a `ShipBlueprint` database.
- `ProjectileManager` owns projectile lifecycle and tracks faction/source-ship context.
- `WeaponManager` handles spinal/turret mount relationships.
- Future/considered systems include Faction, Sector, SolarSystem, GalacticSector, Fleet, Station, Tank, Fighter, Infantry, extraction, asteroid, and platform layers.
- Fighters were envisioned as a layer above ordinary ships; point-defense turrets could interact with them.
- Player ship construction has used a Phaser `Container` concept, with pooled objects and `EntityManager` ownership. A synchronization point before `engine.update` was discussed.
- F1/F2/F3 spawn/testing harness controls were part of the described development setup.

Do not assume every listed manager/system is fully implemented. Treat them as established architectural direction or plans according to the current source files.

### 2.4 Shipmaker and ship data conventions

Historical ship-data conventions discussed:

```ts
Game.shipData[type] = [
  partsData,
  compData,
  superType,
  subType,
  name,
  categoryType
];
```

- Part positions had a legacy scale conversion: stored positions divided by `5`, then multiplied by `5` during `Ship.init`.
- `compData` includes a `none` representation.
- The historical `PlayerCannon` tuple included cooldown `0.15`.
- The Shipmaker direction is a dev-only `/shipmaker` editor that can export/copy JSON for ship data.
- The editor should show hull sprites and component sprites, support spinal/turret hardpoints, and not silently invent built-in components when not explicitly configured.
- Thruster vent points are emission locations and should be treated as decorative/non-destructible under the current design.

A `Shipmaker` implementation plan and Phase 1 work have been discussed. Historical implementation artifacts include:

- `src/types/Shipmaker.ts`
- `ShipDataAdapter.ts`
- `Game` registries such as `hullDefinitions`, `componentDefinitions`, `shipLoadouts`, and `legacyCompatibilityViews`
- `docs/SHIPMAKER-ARCHITECTURE-HANDOFF.md`

The last saved project memory placed Shipmaker Phase 1 implementation underway and Phase 2 not yet authorized until Phase 1 completion. This status is historical and must be refreshed from the repository.

### 2.5 RTAB and asset packaging architecture

The asset binary format is **RTAB v1.0**, an engine-independent unencrypted container. Phase 3B packages raw SVG payloads; runtime integration is reserved for Phase 3C.

#### RTAB v1.0 outer header — 64 bytes

- Bytes `0–3`: magic `RTAB`.
- Bytes `4–5`: little-endian `uint16` version `0x0100` (major in high byte, minor in low byte; serialized LE as `00 01`).
- Bytes `6–7`: flags; bits 0 and 1 set (`0x0003`), all other bits zero.
- Bytes `8–15`: absolute payload offset.
- Bytes `16–23`: absolute index offset, currently `64`.
- Bytes `24–31`: index size, including index header, records, string table and digest.
- Bytes `32–63`: reserved and zero.

#### RTAB index

- Index begins at offset `64` and has a 28-byte `INDX` header.
- Header fields: magic `INDX`, `uint32` entry count, `uint16` record size `72`, `uint16` index flags `0`, `uint64` string-table offset relative to the index, `uint64` string-table size.
- Every record is exactly 72 bytes:
  - `0–3`: `uint32` asset-ID string offset.
  - `4–7`: `uint32` asset-ID byte length.
  - `8–15`: `uint64` payload-relative offset.
  - `16–23`: `uint64` stored size.
  - `24–31`: `uint64` original size.
  - `32–33`: `uint16` content type.
  - `34–35`: `uint16` compression.
  - `36–39`: `uint32` flags, must be zero.
  - `40–71`: SHA-256 of stored payload bytes.
- String table stores contiguous UTF-8 asset IDs without prefixes, separators, or null terminators.
- Records and payload ordering are deterministic (payload records sorted by asset ID).
- `indexSize = 28 + entryCount * 72 + stringTableSize + 32`.
- Final 32 bytes of the index contain SHA-256 over all preceding index bytes.
- Payload starts at `indexOffset + indexSize`.
- Version 1 stores uncompressed raw bytes: compression `NONE=0`; `storedSize === originalSize`.
- The format enumerates content types SVG=0, PNG=1, JSON=2, Unknown=3, but the Phase 3B packager is restricted to SVG only and rejects other content types.
- Strict validation: v1.0 only; wrong versions, invalid flags, malformed SVG/XML, invalid index structure, bad digests, gaps/overlap/trailing data, and unsupported values should fail.

### 2.6 Asset registry responsibilities

The intended separation is:

- Folder hierarchy determines category/bundle membership.
- Asset registry owns stable asset identity and metadata: `assetId`, `logicalPath`, `textureKey`, aliases, content type, etc.
- Packager validates the folder inventory against registry entries, derives bundle group metadata, and writes `.bin` outputs.
- Generated manifest describes the packaged bundles/assets and their fingerprints. It is generated output, not a second authoring authority.

The proposed Phase 3B.3 design removes authored `bundleId`/bundle table fields from the source registry because they duplicate folder-derived grouping. Generated manifest may still contain a derived `bundleId` for each packaged entry. **This schema change is proposed in the plan but has not been approved or implemented.**

---

## 3. Development history

### 3.1 Core Phaser/game architecture work

- Project moved to Phaser 4 and Vite; the user stresses Phaser 4-only architecture.
- Camera architecture was explored: gameplay camera with player following and zoom; static background camera; HUD/minimap fixed to view; planets/world-space parallax.
- Rendering was conceptually organized into Background, Foreground, and UI Elements.
- Manager boundaries were discussed, emphasizing that `EntityManager` should remain lifecycle/identity-focused rather than becoming a god object.
- Projectile pooling issues were addressed in prior work: Arcade Physics body disable/re-add/reset behavior. At one point focused tests were reported as 64/64, and subsequent project test count reached 73; exact current test inventory needs rechecking.
- A public SVG manifest generator and minimap asset flow were developed. Three minimap SVGs remain separate from the 31 registered game assets. Two root-level Vite scaffold SVGs (`favicon.svg`, `icons.svg`) also remain outside the game registry.
- Minimap markers currently include Capital-Ship, Corvette, and Fighter SVGs; at one point some gameplay markers used Capital-Ship assets because Corvette/Fighter game entities were not yet available. `MiniMap.ts` has been intentionally left untouched through the binary source migration.

### 3.2 Shipmaker direction

- Shipmaker handoff documentation was created earlier.
- Phase 1 work introduced typed shipmaker concepts and an adapter from legacy ship data toward canonical definitions.
- The longer-term editor target is `/shipmaker`, development-only, and exports portable JSON.
- Hardpoint behavior and extension/connector architecture remain important design work. Do not leap into Phase 2 until Phase 1 is verified complete and approved.

### 3.3 Phase 3A — RTAB specification

Phase 3A established the RTAB v1.0 binary container specification described above. It also separated version concepts:

- RTAB container format version.
- Registry schema version.
- Generated manifest schema version.

These must not be conflated. The format is designed to be deterministic, uncompressed in v1, and independently verifiable through per-payload SHA-256, index digest, and full bundle fingerprint.

### 3.4 Phase 3B.1 — Initial packaging implementation

Reported completed Phase 3B.1 implementation included:

- `scripts/asset-registry.json`
- `scripts/asset-registry.mjs`
- `scripts/package-assets.mjs`
- `scripts/prepare-assets.mjs`
- `tests/package-assets.test.ts`
- Changes to `package.json`, `scripts/generate-public-svg-manifest.mjs`, and SVG copyright handling.
- Generated outputs:
  - `public/assets/Ships/ships.bin`
  - `public/assets/asset-manifest.json`
  - `public/assets/asset-manifest-schema.json`
- Current implementation uses an authored bundle identity `ships1` for the ten ship assets. The other 21 registered game assets were not yet packaged in a bundle.
- Exactly 31 game SVG assets were registered: 10 Ships, 19 Components, 2 Turrets-Platforms.
- Three minimap assets and two root-level Vite scaffold assets were deliberately excluded.
- The initial ship bundle was reported as 111,817 bytes with SHA-256:
  `5a88e030dfbd0cecc8d2b51acf7e80dce2bdb36862838ba0ea113d84707fd65b`
- Index size: `1042`; payload offset: `1106`; index has 28-byte header + ten 72-byte records + 262-byte string table + 32-byte digest.
- Phase 3B.1 tests: focused 18/18 passed. `npx tsc --noEmit`, syntax checks, `npm run predev`, and `npm run build` were reported passing. Vite emitted a non-blocking large-chunk warning.
- Historical full suite at the earlier snapshot: 142 passed, 2 failed. The two known failures were in `phase2a-taxonomy.test.ts` (legacy Skill/Passive/Spawn compatibility and PlayerCannon behavior).

Phase 3B.1 kept runtime loose SVG loading. It did not integrate RTAB into Phaser.

### 3.5 Phase 3B.2 — Private source migration

Phase 3B.2 copied the remaining 21 public game SVG sources into mirrored private paths and changed canonical source resolution. The source files are under:

```text
private/assets/
├── Ships/                       # 10 existing private ship sources
├── Components/
│   ├── Aistiaria/               # 8 new files
│   └── Yutiarin/                # 11 new files
└── Turrets-Platforms/           # 2 new files
```

Newly copied files:

- Aistiaria: `01_side_hull_angular.svg`, `02_hex_cockpit.svg`, `03_side_wing.svg`, `04_rear_engine_pod.svg`, `05_nose_section.svg`, `06_swept_wing.svg`, `07_connector_module.svg`, `08_tail_fuselage.svg`.
- Yutiarin: `Part 1.svg` through `Part 10.svg`, and `Wing 1.svg`.
- Turrets-Platforms: `Yutiarin-Defense Turret.svg`, `Yutiarin-Perimeter Turret.svg`.

Phase 3B.2 implementation added `scripts/migrate-assets-private.mjs` and `tests/migrate-assets-private.test.ts`, and updated `scripts/asset-roots.mjs`. The inventory expectation in `tests/add-svg-copyright.test.ts` was also updated.

Reported verification:

- All 31 private sources resolved canonically under `private/assets/`.
- All 31 private files matched their public counterparts by SHA-256; public loose originals remained present.
- 21 new copies total 102,279 bytes.
- All 31 sources total 212,990 bytes.
- Breakdown: Ships 110,711; Aistiaria 41,221; Yutiarin 33,098; Turrets-Platforms 27,960 bytes.
- Three minimap SVGs and two root-level scaffold SVGs remained excluded and unchanged.
- `src/generated/publicSvgManifest.ts` was reported byte-for-byte unchanged by Copilot, although the independent audit could not establish a separate historical pre-migration snapshot for a direct comparison.
- `MiniMap.ts`, Phaser loading, registry identities, texture keys, aliases, content types, and bundle assignments were reported preserved.
- Migration and RTAB-focused tests: 21/21 passed.
- `npx tsc --noEmit`: passed.
- `npm run build` including asset preparation: passed; Vite warned about a 1,464 kB chunk.
- Full test suite: 150 passed, 2 failed. The two failures were reported as the same known `phase2a-taxonomy.test.ts` baseline failures. This was an implementation report; Phase 3B.3 plan correctly asks for a fresh baseline before future changes.
- No collisions or rollback events occurred.
- No commits, staging, or unstaging were performed during Phase 3B.2.

### 3.6 Independent audit of Phase 3B.2

Claude Code performed a read-only audit in two passes. Initial audit had some inaccurate claims. The addendum and final correction established:

- The first addendum incorrectly reported group sizes based on SHA-256 output rather than file metadata. `stat` then confirmed the correct byte sizes listed above.
- The `git ls-files --others --exclude-standard` output represents untracked, **non-ignored** files. The audit corrected this classification.
- `git check-ignore -v private/assets/...` confirmed `/private/` ignore rule in `.gitignore` (line 36 in the audited snapshot).
- The tests did include a partial-install failure cleanup test, correcting Claude's earlier assertion that it did not.
- Final post-install hash verification was retained; it checks a different failure window than the staging hash check and should not be removed as redundant.
- A case-mismatched fixture existed, but that is not equivalent to directly testing Windows case-insensitive filesystem behavior.
- Repeated successful migration execution remained an untested scenario.
- The `SOURCE_ROOT_PATTERNS` regex observation was downgraded conceptually to a maintainability concern absent a demonstrated wrong mapping/failure. Claude's reports were inconsistent about exact file attribution; do not treat it as an established major defect without reinspection.
- Overall Phase 3B.2 was accepted by the assistant as safe to accept, with non-blocking observations and no demonstrated data-loss defect.

### 3.7 Phase 3B.3 — Folder-driven packaging plan (latest)

The user asked to continue with folder-driven packaging and a future listener compatible with potential mod support. Copilot was asked to prepare a plan only. The published plan is:

[Phase 3B.3 — Folder-Driven Asset Packaging Plan.txt](https://github.com/Arcyriea/Lythiariena-Arcyriea/blob/main/Phase%203B.3%20%E2%80%94%20Folder-Driven%20Asset%20Packaging%20Plan.txt)

Copilot stated it only inspected files; no files were edited or created, no tests were run, and Git staging was untouched.

Plan findings:

- Current registry has 31 assets, schema version 1.0, ten entries authored with `bundleId: "ships1"`, and a `ships1` bundle declaration. The other 21 assets have no bundle assignment.
- `asset-registry.mjs` currently validates an explicit bundle table, defaults to exactly 31 assets, and requires exactly three aliases; these constraints obstruct easy addition of registered assets/categories.
- `package-assets.mjs` only packages registry entries assigned to declared bundles; it does not discover folder categories or reject unregistered SVGs.
- `prepare-assets.mjs` selects package vs read-only validation according to whether `private/` exists. Its publisher rolls back caught publication errors but does not remove obsolete generated bundles.
- Current private categories: 10 Ships, 19 Components, 2 Turrets-Platforms.

The plan recommends:

**Stage A — one-shot folder-driven packager**

1. Discover immediate child directories of `private/assets/` as categories.
2. Recursively enumerate supported SVG sources.
3. Match every discovered file to exactly one registered logical path, preserving exact path casing.
4. Fail before publication for missing registered sources, unregistered SVGs, case/path collisions, symlink/reparse points, unsafe names, and normalized category collisions.
5. Empty category directories produce no bundle.
6. Derive bundle IDs and filenames from category folder names; no numeric suffixes.
7. Current mapping:
   - `Ships/` → `ships` → `/assets/ships.bin`
   - `Components/` → `components` → `/assets/components.bin`
   - `Turrets-Platforms/` → `turrets-platforms` → `/assets/turrets-platforms.bin`
   - Future `Stations/` → `stations` → `/assets/stations.bin`
8. Proposed slug normalization: Unicode NFKD, strip combining marks, lowercase, replace each run of non-ASCII-alphanumeric characters with `-`, trim hyphens; reject empty results and collisions.
9. Proposed registry change: remove authored `bundleId` fields and authored bundle table, derive bundle IDs from folder hierarchy, retain `bundleId` only as generated manifest metadata.
10. Proposed registry schema bump to 1.1; keep RTAB v1.0 and generated manifest schema 1.0 if its shape does not need to change. These are proposals, not yet user-approved details.
11. Relax hard-coded runtime expectation of exactly 31 assets; assert today's 31 expected identities in tests. Preserve existing three alias mappings while allowing future aliases.
12. Generated manifest should list all 31 assets grouped into three bundles; `src/generated/publicSvgManifest.ts` must remain unchanged and Phaser continues loose SVG loading.
13. Generate/validate all outputs before publication. Publish bundles/schema first and manifest last as a commit marker. Use safe rollback and manifest-owned stale bundle removal. Never wildcard-delete `.bin` files.
14. Old generated bundle currently staged at `public/assets/Ships/ships.bin`; desired output path becomes root-level `public/assets/ships.bin`. Remove the old nested generated bundle from the worktree only after safe successful publication, while preserving the Git index. The resulting staged/unstaged status must be reported. This is an important Git-state edge case.

Expected current outputs at the Phase 3B.3 audit snapshot:

| Bundle | Assets | Raw source bytes | Expected RTAB bytes |
|---|---:|---:|---:|
| `ships.bin` | 10 | 110,711 | 111,817 |
| `components.bin` | 19 | 74,319 | 76,407 |
| `turrets-platforms.bin` | 2 | 27,960 | 28,296 |

The existing ship bundle should remain byte-identical if source data, ordering, and format remain unchanged; verify actual bytes/hash rather than assume.

**Stage B — continuous development watcher, separate from Stage A**

1. Add `scripts/watch-assets.mjs` as a small coordinator.
2. Attach it to Vite's dev-server lifecycle via a proposed `vite.config.mjs`, using Vite's existing watcher initially.
3. Explicitly add/watch `private/assets/` and `scripts/asset-registry.json`; test ignored private paths and recursive Windows watching.
4. Add Chokidar only if Vite watcher fails the required Windows/platform tests, with separate justification.
5. Startup full reconciliation; watch SVG add/change/unlink, directory changes, registry edits.
6. Debounce event bursts, serialize with a single-flight queue, coalesce events arriving during a build, schedule one follow-up if dirty.
7. Rebuild only the affected category for a stable registered asset modification when safe. Use full reconciliation for registry edits, category add/remove/rename, moves, ambiguous event batches, and invalid previous output.
8. Validate source stability before publication; handle duplicate Windows events, case-only renames, writes in progress, and transient sharing violations.
9. Preserve prior valid outputs on failures; report errors without tight retry loops; detach handlers and finish/cancel queued work on server shutdown.
10. Keep `predev`/`prebuild` as full reconciliation boundaries so missed events do not leave stale production output.
11. Watcher-generated bundle changes do not visually hot-reload Phaser artwork; until Phase 3C, runtime still loads loose SVG URLs.

**Future mod compatibility boundary:** No mod discovery, installation, enablement, loading, merging, or overrides are implemented in Phase 3B.3. The plan recommends making category normalization/grouping independent of the base source root so a future composer can group assets from multiple independently owned source roots. Future registry merges should be in-memory; mod sources are never copied into `private/assets/`. A namespaced ID convention such as `mod.<modId>.<assetName>` is only a future candidate. Override/conflict policy and mod manifest versions remain separate design questions.

**Files Stage A expects to change:** `scripts/asset-registry.json`, `scripts/asset-registry.mjs`, `scripts/package-assets.mjs`, `scripts/prepare-assets.mjs`, `tests/package-assets.test.ts`, and the mod-support planning document to correct filename spelling if needed. Expected generated outputs are the manifest/schema and three root-level `.bin` files. No public loose SVGs, private SVGs, Phaser runtime, minimap, or Vite scaffold assets should change.

**Files Stage B expects to add:** `scripts/watch-assets.mjs`, `vite.config.mjs`, `tests/watch-assets.test.ts`; `package.json` only if a tested need for a separate watcher dependency or script is established.

**Plan sequence:** recheck/preserve Git state → implement Stage A only → tests/validation/audit → review Stage A → implement Stage B separately → tests for event handling, concurrency, Windows behavior, failure preservation, shutdown → full verification. No Git index changes or commits.

---

## 4. Current implementation state

### 4.1 Latest known phase/status

- Phase 3A: RTAB v1.0 specification established.
- Phase 3B.1: initial ship bundle and manifest/schema generation implemented and reported complete.
- Phase 3B.2: all 31 game SVG sources migrated to private canonical locations and reported complete; accepted after read-only audit corrections.
- Phase 3B.3: **planning only**. Copilot published the plan linked above. No Stage A or Stage B implementation has been approved or started in the conversation.
- Phase 3C: not started. Phaser still loads loose public SVGs; there is no RTAB runtime bootstrap/integration in the context available here.

### 4.2 Current asset inventory

| Category | Files | Raw source bytes |
|---|---:|---:|
| Ships | 10 | 110,711 |
| Components/Aistiaria | 8 | 41,221 |
| Components/Yutiarin | 11 | 33,098 |
| Turrets-Platforms | 2 | 27,960 |
| **Total** | **31** | **212,990** |

The 21 non-ship assets total 102,279 bytes. Components as a category total 74,319 bytes. Turrets-Platforms total 27,960 bytes.

All 31 private/public source pairs were reported SHA-256 identical, and both filesystem listings showed matching sizes. Public originals remain present. `private/` is ignored by Git. Three minimap SVGs and two root-level scaffold SVGs are excluded from the game registry.

### 4.3 Current runtime and generated state

- Current runtime still uses `src/generated/publicSvgManifest.ts` and public loose SVG URLs.
- `MiniMap.ts` was intentionally unchanged in Phase 3B.2.
- Existing `ships1` bundle assignment/manifest is still the Phase 3B.1/3B.2 legacy state in the latest plan; it has not yet been replaced with folder-derived `ships`, `components`, and `turrets-platforms` identifiers.
- Current generated ship bundle path was identified by the Phase 3B.3 plan as `public/assets/Ships/ships.bin`, with 111,817 bytes. It is a staged Phase 3B.1 output in the latest reported Git state.
- Current root-level output names (`public/assets/ships.bin`, `components.bin`, `turrets-platforms.bin`) are desired Phase 3B.3 results, not yet implemented.

### 4.4 Latest known Git snapshot (not freshly checked)

At the latest audit snapshot:

- Branch: `main`; expected HEAD `405ec63a7cb0d322b559f029a03d4734dddd8937`.
- Staged Phase 3B.1 outputs:
  - `public/assets/Ships/ships.bin`
  - `public/assets/asset-manifest-schema.json`
  - `public/assets/asset-manifest.json`
- Modified tracked files:
  - `.gitignore`
  - `package.json`
  - `scripts/add-svg-copyright.mjs`
  - `scripts/generate-public-svg-manifest.mjs`
- Untracked, non-ignored files shown by `git ls-files --others --exclude-standard`:
  - `docs/BINARY-ENCRYPT-MIGRATION.md`
  - `scripts/asset-registry.json`
  - `scripts/asset-registry.mjs`
  - `scripts/asset-roots.mjs`
  - `scripts/migrate-assets-private.mjs`
  - `scripts/package-assets.mjs`
  - `scripts/prepare-assets.mjs`
  - `tests/add-svg-copyright.test.ts`
  - `tests/migrate-assets-private.test.ts`
  - `tests/package-assets.test.ts`
- `private/` is ignored; `dist/` and dependencies are also ignored.
- Some planning documents are ignored by `docs/*`.
- No commits/staging/unstaging were performed during Phase 3B.2.

The preceding audit incorrectly called the untracked list nine files; the actual displayed list has ten entries. Use current Git output, not these notes, as authority.

**Important planned Git consequence:** Phase 3B.3 proposes replacing the nested `public/assets/Ships/ships.bin` output with root-level `public/assets/ships.bin`. It also says to leave the Git index unchanged. If the tracked worktree copy is removed while the index still contains the staged addition, Git may report an unstaged deletion in addition to the staged addition. Do not stage, restore, or reset anything to make the status look clean. Report the exact resulting status and preserve the index.

### 4.5 Verification history and known tests

- Phase 3B.1 focused tests: 18/18 passed.
- Phase 3B.2 migration + RTAB focused tests: 21/21 passed.
- `npx tsc --noEmit`: reported passed.
- `npm run build` and preparation: reported passed; Vite large chunk warning (1,464 kB in later report).
- Full test suite reported 150 passed, 2 failed; the two failures were the existing `phase2a-taxonomy.test.ts` cases also observed at earlier baseline snapshots. Historical counts changed as tests were added (142 passed + 2 failed at an earlier snapshot, later 150 + 2). Do not treat either as a fresh baseline for Phase 3B.3.
- Phase 3B.3 plan explicitly requires rerunning the current suite before implementation to establish an actual baseline, then compare any new failures against it.

---

## 5. User requirements, preferences, and constraints

### 5.1 Approval and scope control

- Default to inspecting and planning before editing.
- For large or architectural tasks, deliver a plan and wait for explicit approval.
- Respect phase boundaries. Do not jump to later phases because they seem convenient.
- A planning document is not implementation authorization.
- If the user approves only Stage A, do not implement Stage B watcher in the same pass.

### 5.2 Git and working-tree safety

- Preserve all user work: staged, unstaged, untracked, and ignored.
- Never use `git reset`, `git clean`, destructive checkout/restore, or broad cleanup operations without explicit permission.
- Do not stage/unstage or commit while the binary migration is still underway.
- The user's stated policy is to commit only once the complete binary migration is done (including the later runtime migration phase).
- Private source files must remain ignored and must not be committed.
- When generated output paths change, do not silently tidy the index. Explain the staged/worktree effects and preserve the index.

### 5.3 Asset migration/runtime boundaries

- Keep public loose SVG originals through Phase 3B and until Phase 3C is properly implemented and verified.
- No public SVG deletion during Phase 3B.3.
- No Phaser runtime integration in Phase 3B.3.
- No minimap or Vite scaffold assets in the registered game asset system unless separately approved.
- Do not automatically invent stable asset IDs, texture keys, aliases, or metadata for discovered SVGs. Unregistered assets must fail/reported until the registry is deliberately updated.
- RTAB v1.0 remains unchanged unless the user explicitly approves a format revision.

### 5.4 AI coding instruction preferences

- Ask Copilot to inspect the real current repository, not rely on remembered or summarized state.
- Copilot prompts should list scope, constraints, exact files/behavior, acceptance criteria, and stop conditions.
- Require Copilot to preserve the index and working tree; ask for before/after status.
- Use read-only audits independently rather than accepting an implementation report as proof.
- Avoid overly broad implementation requests. Split one-shot packager and continuous listener into separate reviewable stages.
- Do not claim a test passed unless it was actually run; distinguish static inspection from execution.

### 5.5 Things explicitly not approved or rejected

- `nonships.bin` as one combined 21-asset bundle was explicitly rejected by the user's clarification. Components and Turrets-Platforms must be separate.
- Numeric bundle filename suffixes such as `ships1.bin` or `components1.bin` are not desired.
- Phase 3C runtime integration is not approved/started.
- A live watcher is a planned capability but must be implemented after the one-shot folder-driven packager is reviewed, not mixed into Stage A.
- Full mod loading, discovery, installation, enable/disable, conflict/override handling, and runtime asset composition are not part of current Phase 3B.3.
- No arbitrary auto-generated IDs/texture keys.

---

## 6. AI development workflow

### 6.1 GitHub Copilot

**Primary implementation assistant.** The user generally prefers Copilot for implementation because it tends to follow scoped coding instructions and can perform repository edits.

Best practice for Copilot prompts:

- Start with repository root, branch/HEAD/status checks.
- Require it to read architecture and plan files first.
- Say whether the task is planning-only or implementation-authorized.
- Define no-touch files and preserve unrelated work.
- Specify exact test/build commands and acceptance evidence.
- Explicitly forbid staging, commits, resets, cleanups, and Phase creep.
- Require it to stop after the requested phase and report every changed file.

### 6.2 Claude Code

**Independent read-only audit/review assistant.** It has been useful for repository inspection and audit work. Its first Phase 3B.2 audit included inaccuracies and needed evidence corrections; the follow-up corrected several of them. It should be instructed to distinguish verified evidence from assumptions and not to edit files during audits.

The user has previously found Claude Code reliable at following through on repository tasks, but has also noted it can be less dependable when editing existing files. For this project, read-only audit is a strong use case.

### 6.3 Qwen Code / Gemma

- Local `qwen3.5:9b` through Ollama/Qwen Code previously failed to follow through consistently and sometimes looped or did not use tools effectively.
- User has used cloud models through Ollama's Qwen Code launcher.
- **Gemma 4 31B Cloud** was preferred for documentation, architecture review, and independent auditing; it appeared more coherent than local Qwen for this type of work.
- Gemma/local models should not be assumed to have completed a task simply because they describe a plan. Require concrete tool evidence and changed-file/test reports.

### 6.4 Aider and other tools

- Aider v0.86.2 was used with a local FreeLLMAPI-compatible endpoint at `localhost:31415/v1`, model noted as `openai/gemma-4-12b`.
- The user prefers avoiding unnecessary system-wide installations, PATH modifications, global config edits, MCP/GitHub CLI/mcporter additions, or other tooling changes without approval.

### 6.5 Recommended division of labor

- Copilot: implementation, after approval.
- Claude Code: read-only code audit and verification.
- Gemma 4 31B Cloud: architecture/doc review or second opinion.
- ChatGPT: maintain project continuity, challenge assumptions, prepare precise prompts, compare reports against approved constraints, and help decide phase boundaries.

---

## 7. Important technical details

### 7.1 Asset counts and exact source paths

**Ships (10):**

- `alien-interceptor.svg`
- `alien-leviathan.svg`
- `alien-mothership.svg`
- `curved-corvette.svg`
- `faceted-cruiser.svg`
- `human-dreadnought.svg`
- `human-frigate.svg`
- `void-dreadnought.svg`
- `Yutiarin Kinetic Lance Frigate.svg`
- `yutiarin-starblade-destroyer.svg`

**Aistiaria Components (8):**

- `01_side_hull_angular.svg`
- `02_hex_cockpit.svg`
- `03_side_wing.svg`
- `04_rear_engine_pod.svg`
- `05_nose_section.svg`
- `06_swept_wing.svg`
- `07_connector_module.svg`
- `08_tail_fuselage.svg`

**Yutiarin Components (11):**

- `Part 1.svg` through `Part 10.svg`
- `Wing 1.svg`

**Turrets-Platforms (2):**

- `Yutiarin-Defense Turret.svg`
- `Yutiarin-Perimeter Turret.svg`

### 7.2 Asset sizes

| Group | Files | Bytes |
|---|---:|---:|
| Ships | 10 | 110,711 |
| Aistiaria Components | 8 | 41,221 |
| Yutiarin Components | 11 | 33,098 |
| Turrets-Platforms | 2 | 27,960 |
| Components combined | 19 | 74,319 |
| Non-ship combined (historic comparison only; not the chosen bundle layout) | 21 | 102,279 |
| All game SVGs | 31 | 212,990 |

Expected deterministic RTAB outputs at the Phase 3B.3 plan's source snapshot:

| Output | Assets | RTAB bytes |
|---|---:|---:|
| `ships.bin` | 10 | 111,817 |
| `components.bin` | 19 | 76,407 |
| `turrets-platforms.bin` | 2 | 28,296 |
| **Total** | **31** | **216,520** |

Do not confuse raw SVG source bytes with RTAB bundle bytes.

### 7.3 Registry/manifest identities and aliases

The 3B.1/3B.2 registry established 31 stable asset entries. Three important legacy aliases were explicitly preserved:

- `Ships/human-dreadnought.svg` → `human-dreadnought`
- `Ships/alien-interceptor.svg` → `alien-interceptor`
- `Components/Aistiaria/01_side_hull_angular.svg` → `central-hull`

Keep current texture keys and aliases intact unless a separate explicit schema change is approved. The Phase 3B.3 plan proposes preserving these three mappings but allowing new aliases, rather than hard-coding the total alias count to three.

`src/generated/publicSvgManifest.ts` remains the runtime loose-SVG manifest and should remain unchanged during Stage A. The generated RTAB `asset-manifest.json` / schema is a separate asset-packaging manifest.

### 7.4 Package lifecycle

At Phase 3B.1/3B.2:

- `package.json` `predev` and `prebuild` call `node scripts/prepare-assets.mjs`.
- When private sources are available, preparation packages assets and generates output.
- When the private root is absent (clean clone), preparation follows a read-only validation path based on the generated bundle/manifest outputs.
- A present-but-incomplete private root should fail, not fall back to public loose SVG files.

The Phase 3B.3 plan should preserve this behavior and make reconciliation folder-driven.

### 7.5 Current generated asset naming vs intended names

Legacy/current at latest audit:

- Bundle logical ID `ships1`.
- Current generated bundle path: `public/assets/Ships/ships.bin`.

User-approved target naming/grouping:

- Logical ID derived as `ships`; output `public/assets/ships.bin`.
- Logical ID derived as `components`; output `public/assets/components.bin`.
- Logical ID derived as `turrets-platforms`; output `public/assets/turrets-platforms.bin`.

The existing `ships1` logical assignment is legacy and expected to be reconciled during approved Stage A. Do not rename or remove it outside that implementation gate.

### 7.6 Shader and rendering detail

Prior shader details include:

- `src/shaders/planets.glsl?raw`
- `PlanetShader` and `planetSphere`
- `uMainSampler`
- Time uniform sourced from `scene.time.now * 0.001`

Blender 4.5.5 LTS EEVEE rendering freezes were previously resolved by switching OpenGL to Vulkan; this is background project context, not a current asset pipeline task.

### 7.7 Other gameplay design concepts

Ideas discussed for future gameplay include:

- Starsector-style energy flux/overload.
- Baseline energy regeneration target `energyRegen = 20`.
- Ammunition represented as a fleet stockpile.
- Asteroid capture.
- Alien particle condensation.
- Ship/fleet/station/tank/fighter/infantry and extraction/platform systems.

These are design ideas or targets, not necessarily implemented systems. Validate current source before treating any as done.

---

## 8. Future plans

### 8.1 Immediate asset pipeline work

The intended continuation is Phase 3B.3, structured as two separate stages:

**Stage A — one-shot folder-driven packager**

- Replace hand-authored bundle grouping with category discovery from `private/assets/`.
- Keep registry asset identity and metadata explicit.
- Verify every file is registered exactly once; reject unknown SVGs and missing registered assets.
- Derive output names from top-level category folder names.
- Create the three current category bundles: ships, components, turret-platforms.
- Generate a deterministic RTAB v1.0 bundle set and manifest.
- Add safe stale output detection/removal and transactional publication.
- Keep loose SVG runtime behavior unchanged.

**Stage B — development filesystem listener**

- Watch source files and registry changes during Vite development.
- Debounce and serialize packaging work.
- Rebuild only affected category bundles when safe; otherwise full reconciliation.
- Preserve the last valid outputs after failure.
- Handle Windows event/case/rename/write-in-progress behavior.
- Shut down cleanly.
- Do not confuse bundle regeneration with Phaser visual hot reload.

Stage A should be implemented, verified, and reviewed before Stage B begins.

### 8.2 Future mod support

A separate planning archive was created:

[MOD-ASSET-PACKAGING-AND-FUTURE-MOD-SUPPORT.md](sandbox:/mnt/data/MOD-ASSET-PACKAGING-AND-FUTURE-MOD-SUPPORT.md)

It records the intended concept:

```text
mods/
└── MyFactionMod/
    ├── mod.json
    ├── asset-registry.json
    └── assets/
        ├── Ships/
        ├── Components/
        │   └── MyFaction/
        └── Turrets-Platforms/
```

Future intent:

- Mod folders mirror the base-game asset category hierarchy.
- Modders supply registry metadata for their assets.
- Mod assets are grouped by category and merged into the effective category `.bin` output alongside base assets.
- Base game and mod source files remain independently owned and recoverable.
- Disabling/removing a mod regenerates the combined output from the remaining valid sources.
- Category grouping logic should eventually be independent of a single source root to support multiple source sets.
- Namespaced IDs such as `mod.<modId>.<assetName>` are a proposal, not a finalized convention.
- Collision/override/priority/dependency policy, mod registry versioning, trust/safety checks, and runtime installation are unresolved.

No mod loading, discovery, installation, enablement, runtime merging, or override system exists in the current phase.

### 8.3 Phase 3C — runtime bundle integration

Phase 3C is still future work. The broad intent from the earlier migration specification:

- Integrate asynchronous RTAB bundle bootstrap before Phaser scene preload.
- Validate every required bundle and fingerprint before runtime use.
- Register exact texture keys/aliases for the runtime.
- Treat missing/corrupt/mismatched required bundles as fatal to bootstrap; no public loose fallback.
- Only after successful runtime validation and rendering verification should public game SVG copies be removed.

Do not start Phase 3C as part of Phase 3B.3.

### 8.4 Game feature/learning direction

- Finish/verifiably close Shipmaker Phase 1 before Phase 2.
- Continue learning Phaser 4 cameras, render order, containers, and update lifecycle through manageable tasks.
- Potentially use lore mini-games as scoped practice, without abandoning the larger game.
- Continue development on modular hull extensions/connectors, weapon mounting, ship behavior, fleet combat systems, and the game's own identity.

---

## 9. Unresolved decisions and risks

### 9.1 Phase 3B.3 decisions still requiring explicit approval

The Copilot plan is a plan only. The user has approved the **conceptual direction and three-way output grouping**, but has not yet approved the exact implementation plan or all of its technical details.

Items to review/approve before Stage A:

1. **Registry schema bump:** Copilot recommends registry schema `1.1` when authored bundle fields are removed. This is reasonable but not yet explicitly approved.
2. **Remove authored `bundleId` and bundle table:** Proposed because grouping becomes folder-derived. Confirm that no external consumer depends on the authored table.
3. **Generated manifest schema:** Copilot proposes leaving it at 1.0 if the structure remains unchanged; verify against the actual schema.
4. **Slug normalization:** Proposed algorithm is Unicode NFKD → remove combining marks → lowercase → replace non-ASCII-alphanumeric runs with `-` → trim. This may reject category names that normalize to an empty slug or produce collisions. Confirm whether ASCII-only category names are acceptable, and whether any future localization/non-Latin category folder names matter.
5. **Unknown top-level folders:** Plan says any immediate child folder can become a category if valid and registered. Consider whether all category folders should be allowed or whether categories need a separate allowlist/registration declaration.
6. **Empty directories:** Proposed behavior is no empty bundle. This seems sensible but should be explicit.
7. **Symlink/reparse points:** Plan proposes rejection. Important on Windows and for untrusted mod sources later; confirm error behavior.
8. **Output ownership:** Proposed rule refuses to overwrite a target `.bin` unless the prior generated manifest establishes ownership. This is an important safeguard; require robust validation of manifest paths and ownership before deletion/replacement.
9. **Stale bundle recovery:** Need safe behavior if a process crashes between multi-file publication steps. The plan acknowledges that filesystem-wide atomicity across multiple renames is impossible; use manifest-last commit marker and rollback for handled exceptions, and document crash recovery.
10. **Windows watcher:** Try Vite's existing watcher first; only add Chokidar if Windows behavior tests demonstrate a failure. No dependency change without evidence.
11. **Incremental watcher rebuilding:** Rebuild one category only when the inventory/registry/previous outputs can be fully reconciled; fall back to full pass for registry/category structure changes.
12. **Future mod roots:** Stage B should watch only base private assets and registry initially. Generalize discovery so future mod asset roots can be added without redesign, but do not implement mod discovery now.

### 9.2 Discrepancy in mod-support archive filename

The earlier mod-support archive uses `turret-platforms.bin` (singular `turret`). The user's most recent explicit naming decision and the Phase 3B.3 plan use **`turrets-platforms.bin`** (plural `turrets`). The earlier archive must be corrected if it is revised or copied into the repository. The newest explicit user decision takes precedence.

### 9.3 Stale bundle/index interaction

The Phase 3B.3 plan proposes replacing `public/assets/Ships/ships.bin` with root-level `public/assets/ships.bin`, while retaining the Git index unchanged. Because the old nested file is currently staged as an addition, removing it from the working tree can produce a staged add plus an unstaged deletion. This is expected if index preservation is honored. Do not stage/unstage to hide it. The user intends to perform Git commits only after the complete binary migration.

### 9.4 Historical concerns not proven as defects

- The `SOURCE_ROOT_PATTERNS` regex was flagged by Claude during audit but not tied to a demonstrated incorrect mapping. Treat as a maintainability observation until proven otherwise.
- Staging-data preservation after failures is optional. Safe cleanup is preferable if errors are clear and no recovery data is lost.
- Keep final post-install SHA-256 verification; it checks for corruption after staging verification.
- Case-mismatch fixture coverage is not the same as actual Windows case-insensitive filesystem behavior.
- Repeated successful migration execution was not explicitly tested in the earlier audit.
- Do not overstate exact pre-Phase-3B.2 equality for `publicSvgManifest.ts`; the audit lacked a reliable independent snapshot, even though Copilot reported it unchanged.

### 9.5 Test baseline

The last historical full-suite result was 150 passing / 2 failing, with the same two known `phase2a-taxonomy.test.ts` failures. The Phase 3B.3 plan requires a fresh baseline run before Stage A. Future reports must distinguish existing baseline failures from newly introduced failures.

---

## 10. Immediate next steps

1. **Review the Copilot Phase 3B.3 plan against this handoff and current user decisions.** The plan generally aligns with the folder-driven design and explicitly splits one-shot packaging from the listener.
2. **Resolve the outstanding plan decisions** in Section 9.1, especially registry schema and bundleId removal, folder slug normalization, unknown top-level category behavior, output ownership/stale cleanup, and the Git index consequence of moving the nested ship bundle to root-level `ships.bin`.
3. **Correct the mod-support archive spelling** from `turret-platforms.bin` to the now-approved `turrets-platforms.bin` if that archive is revised or incorporated into the docs repository.
4. **Ask Copilot for a fresh read-only preflight and test baseline** after the plan is approved, using the actual local repo state. No preparation/build command should run during preflight if it can mutate generated outputs.
5. **Approve Stage A only**, with exact file scope and acceptance criteria. Do not approve Stage B automatically.
6. Copilot implements Stage A; no index modification or commits. Review actual generated outputs, manifest, bundle hashes, rollback behavior, stale bundle cleanup, and test/build results.
7. Run an independent read-only audit (Claude Code) of Stage A before authorizing Stage B.
8. Approve and implement Stage B listener separately, including Windows behavior, debounce/queue tests, failure preservation, and shutdown.
9. Only after Phase 3B.3 is complete should the project plan the Phase 3C runtime integration.

---

# HANDOFF STATUS AND NEXT ACTION

**Current agreed state:** Phase 3B.2 private source migration is accepted. The project has 31 canonical private SVG sources with matching public loose copies retained. RTAB v1.0 packaging exists for the ten ship assets only. Runtime still uses loose SVGs.

**Latest user-approved bundle layout:**

- `Ships/` → `ships.bin`
- `Components/` → `components.bin`
- `Turrets-Platforms/` → `turrets-platforms.bin`

No numeric suffixes; no combined `nonships.bin`.

**Latest Phase 3B.3 status:** Copilot published a planning document only. Stage A (one-shot folder-driven packager) and Stage B (continuous development watcher) have not been approved for implementation. Mod support remains a future compatibility goal, not a feature in this phase.

**Immediate next action for the new ChatGPT conversation:** Read the linked Phase 3B.3 plan and this handoff, discuss any remaining design corrections with the user, and request approval for **Stage A only** when the user is ready. Do not edit repository files, alter Git staging, or begin Phase 3C without explicit authorization.
