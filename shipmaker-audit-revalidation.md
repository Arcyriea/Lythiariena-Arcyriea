---
name: audit-revalidation
description: Revalidation of previous audit findings against current main branch
metadata:
  node_type: memory
  type: reference
  originSessionId: c29b82f5-7d95-4baa-a591-0bd3761e3aed
  modified: 2026-11-12T07:14:57.787Z
---

# Revalidation of Previous Audit Findings Against Current `main` Branch

**Current branch**: `main` at commit `95e6c21` ("Merge branch 'main' of https://github.com/Arcyriea/Redemptis-Tiariea")
**Working tree**: Clean — no uncommitted or untracked changes
**Test results**: 253 pass, 11 fail (failures in migration/remove-migrated-public-assets/phase2a-taxonomy tests — pre-existing, unrelated to core audit findings)

## Revalidation Table

### Finding 1 — Shipmaker three-definition architecture

**Previous classification**: Still valid / Not implemented  
**Corrected classification**: **Partially implemented**

**Evidence**:
- The canonical TypeScript definitions (`HullDefinition`, `ComponentDefinition`, `ShipLoadout`) and registries (`Game.hullDefinitions`, `Game.componentDefinitions`, `Game.shipLoadouts`) exist in `src/types/Shipmaker.ts` and `Game.ts:48-51`.
- `ShipDataAdapter.ingestLegacyData()` (called at `Game.ts:131`) populates these registries by converting legacy tuples into compatibility-derived typed records. These are adapter-generated records, not evidence that all game content has been authored canonically.
- `Ship.init()` contains a working canonical initialization path at `initCanonicalBlueprint()` (`Ship.ts:206-278`) that resolves a `ShipBlueprintDefinition` through `hullId` and `loadoutId`.
- Canonical mount resolution and component installation are implemented in `initCanonicalBlueprint()`: built-in components are materialized from hull mounting points, validated for compatibility, instantiated as fixed instances, and their points reserved; loadout installations are validated against remaining points and materialized (`Ship.ts:246-275`).
- The test `shipmaker-parity.test.ts:181-191` ("canonical geometry is used directly and loadout mounts resolve to declared hull points") and `shipmaker-parity.test.ts:203-228` ("built-in components are materialized as fixed instances and reserve their points") verify that when canonical definitions are manually registered, the runtime consumes them correctly.
- The current player blueprint (`Game.ts:133-168`) and ordinary legacy content have **not** completed migration to canonically authored definitions — they use the structured legacy blueprint path.

**Why it matters**: The canonical initialization mechanism is implemented and tested. The outstanding work is production-data migration (authoring canonical hull/component/loadout records for player and legacy ships) and broader integration, not rebuilding the canonical path from scratch. An agent should not assume the typed registries are universally empty — they are populated by the adapter, but with compatibility records, not authored canonical records.

**Classification**: **Partially implemented** — canonical runtime path exists and is functional; production content migration remains.

***

### Finding 2 — PlayerCannon not migrated to typed definitions

**Classification**: **Still valid**

**Evidence**:
- `Game.ts:126-128`: `this.componentData = { "PlayerCannon": ["Weapon", "ProjectileArcade", 10, 400, 0.15, 5, 600, 0] }` — legacy tuple, unchanged.
- `Game.ts:133-168`: Player blueprint registered as a structured `ShipBlueprintDefinition` with `gameplay.parts` and `gameplay.components` arrays — **no `hullId` or `loadoutId`**.
- `Ship.ts:157-166` `init()` resolves the player through the **structured legacy blueprint path** (`initStructuredLegacyBlueprint` at `Ship.ts:280-299`), NOT the canonical path (which requires `hullId`/`loadoutId`).
- `ShipDataAdapter.ingestLegacyData()` (line 131) adapts the `PlayerCannon` tuple into `componentDefinitions` Map as a `kind: 'legacy'` entry — this is a compatibility record, not a canonical `WeaponComponentDefinition`.
- The init hierarchy at `Ship.ts:157-166`:
  1. `if (blueprintDefinition && 'hullId' in blueprintDefinition)` → canonical (NOT taken for player)
  2. `else if (blueprintDefinition)` → structured legacy (TAKEN for player)
  3. `else` → legacy tuple fallback from `Game.shipData`
- Tests confirm PlayerCannon behavior is preserved on the legacy path: `tests/player-weapon.test.ts:24-53`, `tests/ship-blueprint.test.ts:48-109`, `shipmaker-parity.test.ts:134-147`.

**Remaining migration work** (per handoff §493-498):
- Explicit `hull.player` `HullDefinition` with a spinal hardpoint.
- Explicit `loadout.player` `ShipLoadout` installing `PlayerCannon` at that hardpoint.
- A canonical spinal `WeaponComponentDefinition` for PlayerCannon preserving: `ProjectileArcade` firing mode, damage 10, range 400, cooldown 0.15s, energy cost 5, projectile speed 600, splash radius 0, dimensions omitted.
- Preserved player visual composition and existing texture keys (`human-dreadnought`, `ship-wing`, `hardpoint`).
- Preserved player statistics and weapon behavior.
- Preserved player initialization, Phaser body, input, synchronization, and queue separation from NPC `shipQueue`.

**Why it matters**: The adapter-generated compatibility record does not mean the player has completed canonical migration. The player remains on the structured legacy path. Migration requires authored canonical definitions, not adapter projections.

**Classification**: Still valid — no change.

***

### Finding 3 — Public SVG manifest discovery model

**Classification**: **Partially valid — new nuance** (unchanged from previous revalidation)

**Evidence**:
- The production workflow in `prepare-assets.mjs:553` calls `createPublicSvgManifest(undefined, undefined, projectRoot)` — passing `undefined` for `assetsRoot`, which selects the registry-derived discovery path (`discoverRegisteredAssets` reading from `asset-registry.json`).
- The legacy directory-walk path (`if (assetsRoot)` branch at `generate-public-svg-manifest.mjs:137`) still exists in code and is functional when `assetsRoot` is provided.
- The documentation's concern about "legacy discovery model being the default" is not accurate for the production workflow, but the code retains both paths.

**Classification**: Partially valid — production uses registry-derived path; legacy path remains available but not the default invocation.

***

### Finding 4 — RTAB packaging implemented but design not explicitly approved

**Previous classification**: Still valid / design not approved  
**Corrected classification**: **Completed and merged — separate from Shipmaker approval boundary**

**Evidence**:
- `scripts/package-assets.mjs` generates committed `.bin` bundles (`ships.bin`, `components.bin`, `turrets-platforms.bin`, `planets.bin`) in `public/assets/bundles/` with `asset-manifest.json` and `asset-manifest-schema.json`.
- `public/assets/bundles/` contains 4 `.bin` files: ships (10 assets, RTAB 1.0), components (19 assets, RTAB 1.0), turrets-platforms (2 assets, RTAB 1.0), planets (5 PNGs, RTAB 1.1).
- `package.json` scripts: `"predev": "node scripts/prepare-assets.mjs --npm-predev"`, `"prebuild": "node scripts/prepare-assets.mjs"` — these run the packager as part of dev/build workflow.
- `BINARY-ENCRYPT-MIGRATION.md` §1-7 (lines 1-45) and Phase 3B.3 Stage A (§79-91) describe Phases 1-7 and Stage A as completed deliverables. The Phase 3B.3 Stage A work (folder-derived packaging, RTAB v1.0/v1.1 bundles, SHA-256 integrity) is implemented and merged into `main`.
- The historical approval gate in `BINARY-ENCRYPT-MIGRATION.md` §1-7 (§10.1-10.2 in previous report referenced lines that don't exist in this 99-line version) pertained to the **asset pipeline specification** at the time of its writing. That pipeline work is now complete and merged.
- The separate **Shipmaker prerequisite refactoring approval boundary** in `SHIPMAKER-ARCHITECTURE-HANDOFF.md` §635-637 ("The prerequisite refactor is awaiting explicit approval") remains respected and is unrelated to the completed asset pipeline work.

**Why it matters**: An agent should not conflate the completed/merged asset pipeline work with the separate Shipmaker implementation approval boundary. The asset pipeline (Phases 1-7, 3B.3 Stage A/B) is done. The Shipmaker prerequisite refactor (typed hull/component/loadout authoring, player migration, canonical initialization integration) is the separate work awaiting approval.

**Classification**: **Completed and merged** — asset pipeline work is complete; Shipmaker approval boundary remains separate.

***

### Finding 5 — Type definitions nominal, not consumed at runtime

**Previous classification**: Still valid / Nominal only  
**Corrected classification**: **Partially implemented**

**Evidence**:
- `src/types/Shipmaker.ts` defines `HullDefinition`, `ComponentDefinition`, `ShipLoadout`, and related interfaces — these are TypeScript interface declarations (contract schemas).
- `Ship.ts` **consumes canonical definitions** at runtime through `initCanonicalBlueprint()` (`Ship.ts:206-278`): it reads `HullDefinition` parts, weapon hardpoints, thruster vents, utility sockets; resolves `ComponentDefinition` (non-legacy) via `requireCanonicalComponent()`; validates mount compatibility via `validateCanonicalMount()`; and materializes components via `installCanonicalComponent()`.
- `ShipDataAdapter.ts` creates **compatibility-derived typed records** from legacy tuples at `ingestLegacyData()` (`ShipDataAdapter.ts:17-113`). These populate `Game.hullDefinitions`, `Game.componentDefinitions`, `Game.shipLoadouts` with records that have `kind: 'legacy'` for components and synthesized hull/loadout IDs — they are adapter output, not authored canonical data.
- Manually authored canonical definitions **can be registered and consumed** — verified by `shipmaker-parity.test.ts:181-228` which registers `hull.canonical`, `CanonicalCannon` (spinal weapon), `loadout.canonical`, and a canonical blueprint, then calls `Ship.init('canonical')` which successfully uses the canonical path.
- Current production player content follows the **structured legacy blueprint path** (`initStructuredLegacyBlueprint` at `Ship.ts:280-299`), not the canonical path.

**Distinction**:
- **Interface declarations**: `src/types/Shipmaker.ts` — contract schemas.
- **Adapter-generated compatibility data**: `ShipDataAdapter.ingestLegacyData()` output — legacy tuples projected into typed shape, tagged `kind: 'legacy'`.
- **Manually authored canonical data**: Test fixtures in `shipmaker-parity.test.ts:35-104` — explicit `HullDefinition`, `ComponentDefinition` (non-legacy), `ShipLoadout`, canonical blueprint with `hullId`/`loadoutId`.
- **Production content actually used**: Player blueprint at `Game.ts:133-168` — structured legacy path.

**Why it matters**: The typed definitions are consumed by the runtime when present. The gap is that production content has not been migrated to authored canonical definitions. The types are not "merely nominal" — the canonical initialization path is implemented and functional.

**Classification**: **Partially implemented** — canonical consumption path exists; production content migration remains.

***

### Finding 6 — Player loadout vs built-in distinction not enforced

**Previous classification**: Still valid / Not implemented  
**Corrected classification**: **Implemented in the canonical runtime path; not yet integrated into current player content**

**Evidence**:
- `Ship.initCanonicalBlueprint()` (`Ship.ts:206-278`) implements the complete built-in/loadout distinction:
  - **Built-in materialization** (`Ship.ts:246-255`): Iterates all hull mounting points (weapon hardpoints, thruster vents, utility sockets, active module mounts); for each `builtInComponentId`, resolves the canonical component definition, validates mount compatibility, constructs a fixed component instance with `installationSource: 'built-in'`, reserves the point.
  - **Loadout materialization** (`Ship.ts:257-275`): Iterates `ShipLoadout.installations`; rejects duplicate mount IDs; rejects installations at reserved (built-in) points; validates mount exists and component/mount compatibility; materializes with `installationSource: 'loadout'`, `fixed: false`.
  - **Compatibility validation** (`Ship.ts:399-434`): `validateCanonicalMount()` enforces kind/category/configuration matching between component and mount point type.
- Tests verify this behavior: `shipmaker-parity.test.ts:203-228` ("built-in components are materialized as fixed instances and reserve their points") sets `builtInComponentId` on a hardpoint, clears loadout installations, and asserts the component is materialized as `built-in` + `fixed: true`; a second test asserts that attempting a loadout installation at the same point throws "replaces built-in component".
- The current player blueprint (`Game.ts:133-168`) does not use this system — it has no `hullId`/`loadoutId`, no canonical hull with hardpoints, no `builtInComponentId`, and no `ShipLoadout`. It follows the structured legacy path where `PlayerCannon` is attached by `partIndex` with no built-in/loadout distinction.

**Why it matters**: The underlying mechanism is implemented and tested in the canonical path. The outstanding integration is migrating the player (and other content) to use canonical hull/loadout definitions so this mechanism becomes active for production ships.

**Classification**: **Implemented in canonical path; production integration outstanding**.

***

### Finding 7 — Phase 3C runtime integration

**Previous classification**: Not yet started  
**Corrected classification**: **Implemented and active**

**Evidence**:
- `src/assets/rtabRuntimeLoader.ts` — complete Phase 3C runtime implementation.
- `main.ts:791-803` boots the game via `constructGameAfterRuntimePreflight(preflightRtabAssets, ...)` which:
  1. Fetches `/assets/asset-manifest.json` (`rtabRuntimeLoader.ts:494`)
  2. Validates manifest against generated `runtimeAssetContract` (`rtabRuntimeLoader.ts:496`, `validateRuntimeManifest` at lines 128-253)
  3. Fetches each `.bin` bundle (`rtabRuntimeLoader.ts:506-509`)
  4. Validates each bundle's SHA-256, RTAB magic/version/flags, index integrity, per-record digests (`validateRtabBundle` at lines 282-408)
  5. Decodes SVG/PNG payloads to `HTMLImageElement` (`decodeBrowserImage` at lines 443-462, with SVG scaling at `resizeSvgForPhaser` lines 410-441)
  6. Registers textures with Phaser texture manager (`registerPreparedTextures` at lines 539-565)
- `BootScene` at `main.ts:96-163` receives `PreparedRuntimeAssets` and calls `registerPreparedTextures(this.textures, this.preparedAssets)` in `create()` (line 115).
- `src/generated/runtimeAssetContract.ts` generated by `prepare-assets.mjs` — complete asset inventory with SHA-256 fingerprints, generation ID, active planet placements.
- `public/assets/bundles/` contains 4 content-addressed `.bin` files.
- `asset-manifest.json` at `public/assets/asset-manifest.json` has `manifestSchemaVersion: "1.3"` matching `runtimeAssetContract`.
- The legacy `this.load.svg()` in `BootScene.preload()` (line 106) is ONLY for minimap ship icons (3 SVGs) and dummy texture generation — NOT for main game assets.

**Clarification**: The RTAB runtime validator validates **asset bundle integrity and manifest consistency** (SHA-256, format version, index structure, per-record digests, generation ID). It is **not** a Shipmaker hull/component/loadout contract validator — those are separate TypeScript-level definitions validated at authoring/ingest time, not at RTAB bootstrap.

**Documentation correction**: The `BINARY-ENCRYPT-MIGRATION.md` at this commit is 99 lines and does not contain a §10.3. The phase list (§1-10, lines 1-65) lists Phase 8 as "Bundle-based runtime loading" with deliverable "src/assets/AssetProvider.ts (BundleAssetProvider / SourceAssetProvider); bootstrap decode before BootScene" — this corresponds to the implemented `rtabRuntimeLoader.ts` + `main.ts` preflight integration. The Phase 3C-equivalent work is implemented.

**Classification**: **Implemented and active** — RTAB bootstrap, validation, and texture registration are the production boot path.

***

### Finding 8 — Legacy alias enforcement is strict but limited to 3 entries

**Classification**: **Still valid** (unchanged)

**Evidence**:
- `scripts/asset-registry.mjs` validates exactly 3 legacy aliases (REQUIRED_ALIASES map, lines 7-11, 93-101).
- The `asset-registry.json` includes these as `aliases` fields on specific assets.
- The `publicSvgManifest.ts` also preserves these aliases.
- The 3 exact legacy aliases: `Ships/human-dreadnought.svg` → `human-dreadnought`, `Ships/alien-interceptor.svg` → `alien-interceptor`, `Components/Aistiaria/01_side_hull_angular.svg` → `central-hull`.

***

### Finding 9 — Planet PNGs have both private and public copies

**Classification**: **Still valid** (unchanged)

**Evidence**:
- `public/assets/Planets/` and `private/assets/Planets/` both contain planet PNGs.
- Manifest has private-derived asset IDs (`rt.planet-map.*`) and public-facing texture keys.
- Phase 3A plan: "The 3 planet PNGs are mandatory private sources; public PNG copies are never packaging fallbacks. `Map-111-2.png` and `Map-122-2.png` remain public and outside this migration."

***

### Finding 10 — Sprite layer `rotationFrame` field semantics inconsistent

**Classification**: **Still valid** (unchanged)

**Evidence**:
- `Shipmaker.ts` defines `rotationFrame?: 'component' | 'turret'` on `SpriteLayer`.
- Runtime code (Component, Ship init) does not use this field for transform logic. Component positions are multiplied by 5 and rotated by ship rotation regardless.
- No runtime code references `rotationFrame` for transform computation.

***

### Finding 11 — Shipmaker Architecture Handoff and Binary Encrypt Migration documents are stale

**Classification**: **New finding — documentation is stale** (unchanged from previous revalidation, with corrections)

**Evidence**:
- `SHIPMAKER-ARCHITECTURE-HANDOFF.md` §474-488 (Prerequisite Refactoring Proposal table): Lists `main.ts` as needing "Supply/attach the scene and existing player body/input integration to Ship-owned composition" — but `main.ts` already boots via RTAB preflight and registers prepared textures. The RTAB loading concern is already handled.
- `SHIPMAKER-ARCHITECTURE-HANDOFF.md` §491-498 (Migration Order): Step 1 ("Introduce typed definitions and a runtime validator/conversion boundary") — `ShipDataAdapter.ingestLegacyData()` IS the conversion boundary for Shipmaker data; `rtabRuntimeLoader.ts` IS the runtime validator for **assets** (not Shipmaker hull/component/loadout contracts).
- `SHIPMAKER-ARCHITECTURE-HANDOFF.md` §500-530 (Regression Test Matrix): "Weapon transforms" (524) and "Deterministic visual composition and Ship-owned rendering" (530) are listed as required tests — these are **not** verified by `shipmaker-parity.test.ts:181-191` (which only verifies hull coordinates and loadout mount resolution) nor by any existing test. The shared muzzle-position/direction transform between `Weapon` and `ManagedWeapon` is not implemented; Ship-owned Phaser rendering composition is not implemented.
- `BINARY-ENCRYPT-MIGRATION.md` §82-91 (Phase 3B.3 Stage A): Describes "folder-derived packaging" deriving category from `private/assets/` subdirectories — but `prepare-assets.mjs:553` production workflow calls `createPublicSvgManifest(undefined, ...)` using **registry-derived discovery**, not folder walk. The legacy discovery branch exists but is not the normal production invocation.

**Why it matters**: An agent following these documents would work against stale specifications. The handoff and migration documents need updating to reflect: (a) RTAB Phase 3C-equivalent bootstrap is implemented; (b) canonical Shipmaker initialization path exists; (c) production manifest generation uses registry-derived discovery; (d) weapon transforms and Ship-owned rendering are outstanding implementation areas.

***

## Corrected Implementation Status Summary

| Area | Correct Status |
|------|----------------|
| Typed Shipmaker definitions (`HullDefinition`, `ComponentDefinition`, `ShipLoadout`) | Implemented (interface contracts) |
| Legacy-to-typed compatibility adapter (`ShipDataAdapter.ingestLegacyData`) | Implemented, with migration limitations (produces `kind: 'legacy'` records) |
| Canonical Ship initialization (`initCanonicalBlueprint`) | Implemented |
| Canonical mount validation and loadout installation | Implemented in the canonical path |
| Built-in component materialization and point reservation | Implemented in the canonical path |
| Canonical player hull / loadout / PlayerCannon | Outstanding — player uses structured legacy path |
| Shared weapon/muzzle transforms (`Weapon` + `ManagedWeapon`) | Outstanding — not implemented |
| Deterministic visual composition and Ship-owned Phaser rendering | Outstanding — not implemented |
| Comprehensive typed-contract validation (authoring-time) | Partially implemented (TypeScript + adapter) |
| Operational turret instance behavior (per-instance angle, limits) | Partially implemented (types exist; runtime integration partial) |
| RTAB Phase 3B.3 packaging (bundles, manifest, SHA-256) | Implemented and merged |
| RTAB Phase 3C-equivalent runtime integration (bootstrap, validation, texture registration) | Implemented and active |
| RTAB encryption | Not implemented; separate future milestone |
| Shipmaker editor and JSON serializer | Not implemented |

***

## Summary of Corrections

1. **Finding 1**: Changed from "Not implemented" → **Partially implemented** — canonical path exists and is functional; production content migration remains.
2. **Finding 2**: **Still valid** — player uses structured legacy path; migration requires authored canonical definitions.
3. **Finding 3**: **Partially valid** — production uses registry-derived path; legacy path retained.
4. **Finding 4**: Changed from "Design not approved" → **Completed and merged** — asset pipeline done; Shipmaker approval boundary is separate.
5. **Finding 5**: Changed from "Nominal only" → **Partially implemented** — canonical definitions consumed at runtime when present; production content not yet migrated.
6. **Finding 6**: Changed from "Not implemented" → **Implemented in canonical path** — built-in/loadout distinction works in canonical initialization; player not yet integrated.
7. **Finding 7**: **Implemented and active** — RTAB bootstrap is production boot path; not a Shipmaker contract validator.
8. **Finding 8-10**: Unchanged — still valid.
9. **Finding 11**: **Stale documentation** — updated with corrected specifics (no §10.3 in current BINARY-ENCRYPT-MIGRATION.md; weapon transforms and Ship-owned rendering are not tested by cited tests).
10. **Weapon transform / rendering claims corrected**: `shipmaker-parity.test.ts:181-191` verifies hull coordinates and mount resolution only — does not prove shared muzzle transforms or Ship-owned rendering.

**No code changes made** — all inspections read-only.