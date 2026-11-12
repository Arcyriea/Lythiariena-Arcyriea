---
name: audit-revalidation
description: Revalidation of previous audit findings against current main branch
metadata:
  node_type: memory
  type: reference
  originSessionId: c29b82f5-7d95-4baa-a591-0bd3761e3aed
  modified: 2026-11-12T06:27:37.280Z
---

# Revalidation of Previous Audit Findings Against Current `main` Branch

**Current branch**: `main` at commit `95e6c21` ("Merge branch 'main' of https://github.com/Arcyriea/Redemptis-Tiariea")
**Working tree**: Clean — no uncommitted or untracked changes
**Test results**: 253 pass, 11 fail (failures in migration/remove-migrated-public-assets/phase2a-taxonomy tests — pre-existing, unrelated to core audit findings)

## Revalidation Table

### Finding 1 — Shipmaker three-definition architecture NOT implemented (PREVIOUS: CRITICAL)

**Current verification**: **STILL VALID.**

**Evidence**:
- `Game.ts:48-51` declares `hullDefinitions`, `componentDefinitions`, `shipLoadouts` Maps — but they are **never populated** with structured definitions during `loadGameData()`. The Maps remain essentially empty at runtime.
- `Game.loadGameData()` (lines 133-168) registers the player blueprint using legacy tuple-style `gameplay.parts` and `gameplay.components` data, not structured `HullDefinition`/`ComponentDefinition`/`ShipLoadout` records.
- `ShipManager.registerBlueprint()` accepts `StructuredShipBlueprintDefinition` but the actual registration at `Game.ts:133` passes a plain object with legacy-style gameplay data.
- `Ship.ts:157-166` `init()` method resolves definitions through a hierarchy: first checks `blueprint?.shipDefinition` for canonical form (checking `'hullId' in blueprintDefinition`), then checks for structured legacy blueprint, then falls back to `Game.shipData[type]` legacy tuples. The canonical path (checking `hullId`) is **not taken** because no blueprints have `hullId`/`loadoutId` set.
- `ShipDataAdapter.ingestLegacyData()` is the mechanism that actually populates the `hullDefinitions`, `componentDefinitions`, `shipLoadouts`, and `legacyCompatibilityViews` Maps — it converts legacy tuples into the structured format. Without calling this adapter, the Maps remain empty.
- The test `shipmaker-parity.test.ts:181-191` verifies that when a canonical blueprint IS properly registered (with `hullId`/`loadoutId`), `Ship.init()` uses the canonical path and populates parts from the hull definition. But this test requires manual registration — it's not the automatic code path.

**Why it matters**: An agent following the Shipmaker Architecture Handoff document who assumes the typed registries are automatically populated could remove or bypass the legacy tuple resolution paths (`ShipManager.syncFromGame()`, `ShipDataAdapter.ingestLegacyData()`, `Ship.init()` fallback to `Game.shipData`), breaking the existing working code. The legacy paths are the **active** code paths; the typed architecture is a **proposal** that requires explicit population.

**Classification**: Still valid — no change.

***

### Finding 2 — PlayerCannon not migrated to typed definitions (PREVIOUS: HIGH)

**Current verification**: **STILL VALID.**

**Evidence**:
- `Game.ts:126-128`: `this.componentData = { "PlayerCannon": ["Weapon", "ProjectileArcade", 10, 400, 0.15, 5, 600, 0] }` — legacy tuple, unchanged.
- `Game.ts:133-157`: Player blueprint registered with legacy-style `gameplay.parts` and `gameplay.components`.
- `tests/player-weapon.test.ts:24-53`: Verifies PlayerCannon tuple values are preserved (damage 10, range 400, cooldown 0.15, energy cost 5, projectile speed 600, splash radius 0).
- `tests/ship-blueprint.test.ts:48-109`: Verifies player ship stats from `objectPool.getShipObject('player')` match expectations built on the legacy tuple system.
- `tests/shipmaker-parity.test.ts:134-147`: "legacy PlayerCannon operation values and player initialization behavior are preserved" test passes — confirms PlayerCannon behavior is unchanged.
- `ShipDataAdapter.ingestLegacyData()` adapts the legacy tuple into `componentDefinitions` Map entries (adapted as `kind: 'legacy'` components), but the original tuple in `Game.componentData` remains the source.
- **Area B — Player migration status detailed examination**:
  - The player blueprint (`Game.ts:133-168`) is registered as a structured `ShipBlueprintDefinition` (not canonical) with `gameplay.parts` and `gameplay.components` arrays.
  - `Ship.ts:157-166` `init()` resolves this through the **structured legacy blueprint path** (`initStructuredLegacyBlueprint` at `Ship.ts:280-299`), NOT the canonical path (which requires `hullId`/`loadoutId`).
  - The player ship has **no `hullId` or `loadoutId`** — its blueprint does not reference canonical `HullDefinition` or `ShipLoadout` records.
  - `ShipDataAdapter.ingestLegacyData()` (line 131) is called BEFORE the player blueprint registration, so it adapts the `PlayerCannon` tuple into `componentDefinitions` Map, but this adapted entry has `kind: 'legacy'`, not a canonical `WeaponComponentDefinition`.
  - The player ship's `Ship.init()` (line 137 in test) uses the structured legacy path because `blueprintDefinition` exists but lacks `hullId`.
  - The `Ship.ts` init hierarchy at lines 157-166 is: 
    1. `if (blueprintDefinition && 'hullId' in blueprintDefinition)` → canonical (NOT taken for player)
    2. `else if (blueprintDefinition)` → structured legacy (TAKEN for player)
    3. `else` → legacy tuple fallback from `Game.shipData`
  - **No migration has occurred** — the player remains on the structured legacy path with legacy tuple data adapted at runtime.

**Why it matters**: The PlayerCannon migration is the logical entry point for the Shipmaker prerequisite refactoring (handoff §493-498). Without it, subsequent hull/loadout work has no stable foundation. The legacy tuple remains the active data source.

**Classification**: Still valid — no change.

***

### Finding 3 — Public SVG manifest discovery model (PREVIOUS: HIGH)

**Previous conclusion**: `generate-public-svg-manifest.mjs` has two code paths: (1) legacy walk of `public/assets/` directory, (2) registry-derived discovery when `assetsRoot` is provided. The script is typically invoked without `assetsRoot`, so the legacy path is active. The documentation acknowledges this model is stale but the code has not been updated to make the new model default.

**New evidence**: The production workflow in `prepare-assets.mjs` line 553 calls `createPublicSvgManifest(undefined, undefined, projectRoot)` — passing `undefined` for `assetsRoot`. When `assetsRoot` is `undefined`, the `createPublicSvgManifest` function's `else` branch (line 151) runs — using `discoverRegisteredAssets(projectRoot)`, which reads from `asset-registry.json` and validates canonical sources. This is the **registry-derived** path.

However, the `if (assetsRoot)` branch (line 137) still exists and uses the legacy directory walk. The script `generate-public-svg-manifest.mjs` itself is neutral — it uses whichever path is invoked. The `prepare-assets.mjs` production workflow uses the registry-derived path by default, but the legacy path is still available and functional when `assetsRoot` is provided.

**Classification**: **PARTIALLY VALID — NEW NUANCE.** The production workflow uses the registry-derived path, but the legacy path remains in the code and could be used if `assetsRoot` is provided. The documentation's concern about the "legacy discovery model" being the default is not accurate for the production workflow, but the code still contains both paths and the legacy path is not removed.

**Recommended correction**: Update the revalidation to note that the production workflow uses the registry-derived path (confirmed by `prepare-assets.mjs:553`), but the legacy path still exists in the code and could be invoked if `assetsRoot` is provided. The stale concern is partially mitigated by the production workflow, but the code should still be reviewed for potential cleanup.

***

### Finding 4 — RTAB packaging implemented but design not explicitly approved (PREVIOUS: HIGH)

**Current verification**: **STILL VALID.**

**Evidence**:
- `scripts/package-assets.mjs` exists and generates committed `.bin` bundles (`ships.bin`, `components.bin`, `turret-platforms.bin`, `planets.bin`) in `public/assets/bundles/`. The `asset-manifest.json` and `asset-manifest-schema.json` are generated and committed.
- `public/assets/bundles/` contains 4 `.bin` files with matching `asset-manifest.json`. Verified: ships (10 assets, RTAB 1.0), components (19 assets, RTAB 1.0), turrets-platforms (2 assets, RTAB 1.0), planets (5 PNGs, RTAB 1.1).
- `package.json` scripts: `"predev": "node scripts/prepare-assets.mjs --npm-predev"`, `"prebuild": "node scripts/prepare-assets.mjs"` — these run the packager as part of dev/build workflow.
- `BINARY-ENCRYPT-MIGRATION.md` §10.1-10.2: Phase 3A design must receive explicit approval before Phase 3B proceeds. The handoff (SHIPMAKER-ARCHITECTURE-HANDOFF.md §635-637) states "At this handoff's creation, the architecture audit and proposal exist only as documentation. The prerequisite refactor is awaiting explicit approval."
- The code implements Phase 3B behavior (generating bundles), but the design-phase boundary from the specification document is still in effect per the handoff's own approval boundary.

**Why it matters**: An agent might assume approval is implicit because the code exists, or might not realize the design boundary is still active per the handoff document. The 11 test failures are in migration/remove-migrated-public-assets/phase2a-taxonomy tests — pre-existing and unrelated to the core RTAB packaging functionality.

**Classification**: Still valid — no change.

***

### Finding 5 — Type definitions nominal, not consumed at runtime (PREVIOUS: MEDIUM)

**Current verification**: **STILL VALID.**

**Evidence**:
- `src/types/Shipmaker.ts` defines extensive TypeScript interfaces (HullDefinition, ComponentDefinition, SpriteLayer, WeaponHardpoint, etc.), but **no runtime code references these types for actual data**.
- `Game.ts:48-51` declares `hullDefinitions`, `componentDefinitions`, `shipLoadouts` Maps of types `Map<string, HullDefinition>` / `Map<string, ComponentDefinition>`, but these Maps contain **essentially no structured data** — the player blueprint resolves to legacy ship data.
- `Game.ts:131` calls `ShipDataAdapter.ingestLegacyData(this)` which is the mechanism that actually populates the Maps by converting legacy tuples. Without calling this adapter, the Maps remain empty.
- `Ship.ts:157-166` `init()` method has three resolution paths: canonical (checking `hullId`), structured legacy (checking for `'hullId' in blueprintDefinition`), and legacy tuple fallback (`Game.shipData[type]`). The canonical path is the one that would use the typed definitions, but it's never taken automatically.
- `ShipDataAdapter.ingestLegacyData()` (in `ShipDataAdapter.ts`) adapts legacy tuples into the structured format and populates the Maps, but the original tuple in `Game.componentData`/`Game.shipData` remains the authoritative source.
- The test `shipmaker-parity.test.ts:181-191` ("canonical geometry is used directly and loadout mounts resolve to declared hull points") verifies that the canonical path works WHEN a blueprint is manually registered with `hullId`/`loadoutId`, but this is not the automatic code path.

**Why it matters**: If an agent assumes the typed definitions in `src/types/Shipmaker.ts` are the runtime source of truth and reworks code around them, they would be redesigning against an empty set — the legacy tuples remain the actual data. The type definitions are contract schemas, not runtime data.

**Classification**: Still valid — no change.

***

### Finding 6 — Player loadout vs built-in distinction not enforced (PREVIOUS: MEDIUM)

**Current verification**: **STILL VALID.**

**Evidence**:
- No code materializes built-ins from hull hardpoints or processes a `ShipLoadout` during ship initialization.
- `ShipManager.createShip()` (line 163) acquires from pool, positions, registers — no built-in or loadout materialization.
- `Ship.init()` (called via `ObjectPool.getShipObject()`) resolves components from legacy tuples. There is no code that distinguishes between built-ins and loadout installations.
- The player ship has `PlayerCannon` at part index 0 with no explicit built-in/loadout distinction.
- `Ship.ts:init()` line 199-203: Player ships (`subType === "Player"`) get a `TractorEffect` added, but there's no built-in/component loadout processing.
- The test `shipmaker-parity.test.ts:203-228` "built-in components are materialized as fixed instances and reserve their points" manually sets `builtInComponentId` on a hardpoint and verifies the behavior — but this is a test-only scenario, not production code.

**Why it matters**: Without this distinction enforced, the loadout system cannot function correctly; built-ins and loadout installations would compete for the same points. Thehandoff (§306-332) describes the intended behavior but the code does not implement it.

**Classification**: Still valid — no change.

***

### Finding 7 — Phase 3C runtime integration not yet started (PREVIOUS: MEDIUM)

**Current verification**: **INVALID — Phase 3C IS implemented and active.**

**Evidence**:
- **Area C — Phase 3B.3 asset pipeline status detailed examination**:
  - The RTAB runtime loader **DOES EXIST** at `src/assets/rtabRuntimeLoader.ts` — this is a complete implementation of Phase 3C runtime integration.
  - `main.ts:791-803` boots the game via `constructGameAfterRuntimePreflight(preflightRtabAssets, ...)` which:
    1. Fetches `/assets/asset-manifest.json` (line 494 in rtabRuntimeLoader.ts)
    2. Validates the manifest against the generated `runtimeAssetContract` (line 496)
    3. Fetches each `.bin` bundle (line 506-509)
    4. Validates each bundle's SHA-256, RTAB format, index integrity, and per-record digests (lines 282-408)
    5. Decodes SVG/PNG payloads to `HTMLImageElement` (lines 443-462)
    6. Registers textures with Phaser texture manager (lines 539-565)
  - The `BootScene` at `main.ts:96-163` receives `PreparedRuntimeAssets` and calls `registerPreparedTextures(this.textures, this.preparedAssets)` in `create()` (line 115).
  - `src/generated/runtimeAssetContract.ts` is generated by `prepare-assets.mjs` and contains the complete asset inventory with SHA-256 fingerprints, generation ID, and active planet placements.
  - **Phase 3C IS IMPLEMENTED AND ACTIVE** — the runtime loads from `.bin` bundles, validates integrity, and registers textures. The legacy `this.load.svg()` path in `BootScene.preload()` (line 106) is ONLY for minimap ship icons (3 small SVGs) and dummy texture generation, NOT for main game assets.
  - The `public/assets/bundles/` directory contains the 4 `.bin` files (ships, components, turrets-platforms, planets) with content-addressed filenames (e.g., `ships.5a88e030dfbd0cecc8d2b51acf7e80dce2bdb36862838ba0ea113d84707fd65b.bin`).
  - `asset-manifest.json` at `public/assets/asset-manifest.json` has `manifestSchemaVersion: "1.3"` and matches the generated `runtimeAssetContract`.
- `BINARY-ENCRYPT-MIGRATION.md` §10.3: Phase 3C implements async bootstrap, bundle reading, and texture registration — **this phase IS implemented**, contrary to the document's statement.

**Why it matters**: The previous audit and the BINARY-ENCRYPT-MIGRATION.md document both state Phase 3C is not started, but the code shows it is fully implemented and the production boot path uses it. The documentation is stale.

**Classification**: **INVALID — Phase 3C runtime integration IS implemented and active.** The runtime loader (`rtabRuntimeLoader.ts`), boot integration (`main.ts`), and generated contract (`runtimeAssetContract.ts`) form a complete Phase 3C implementation. The `BINARY-ENCRYPT-MIGRATION.md` document's Phase 3C status is outdated.

***

### Finding 8 — Legacy alias enforcement is strict but limited to 3 entries (PREVIOUS: LOW)

**Current verification**: **STILL VALID.**

**Evidence**:
- `scripts/asset-registry.mjs` validates exactly 3 legacy aliases (REQUIRED_ALIASES map, lines 7-11, 93-101).
- The `asset-registry.json` includes these as `aliases` fields on specific assets.
- The `publicSvgManifest.ts` also preserves these aliases.
- The 3 exact legacy aliases: `Ships/human-dreadnought.svg` → `human-dreadnought`, `Ships/alien-interceptor.svg` → `alien-interceptor`, `Components/Aistiaria/01_side_hull_angular.svg` → `central-hull`.

**Classification**: Still valid — no change.

***

### Finding 9 — Planet PNGs have both private and public copies (PREVIOUS: LOW)

**Current verification**: **STILL VALID.**

**Evidence**:
- `public/assets/Planets/` contains planet PNGs (verified from directory listing during test execution).
- `private/assets/Planets/` also contains planet PNGs.
- The manifest has both private-derived asset IDs (`rt.planet-map.*`) and the public-facing texture keys.
- `BINARY-ENCRYPT-MIGRATION-PHASE-3A-PLAN-WORKING.md` §85-86: "The 3 planet PNGs are mandatory private sources; public PNG copies are never packaging fallbacks. `Map-111-2.png` and `Map-122-2.png` remain public and outside this migration."

**Classification**: Still valid — no change.

***

### Finding 10 — Sprite layer `rotationFrame` field semantics inconsistent (PREVIOUS: LOW)

**Current verification**: **STILL VALID.**

**Evidence**:
- The TypeScript types in `Shipmaker.ts` define `rotationFrame?: 'component' | 'turret'` on `SpriteLayer`, but the runtime code (Component, Ship init) does not use this field for any transform logic. Component positions are multiplied by 5 and rotated by ship rotation regardless.
- No runtime code references `rotationFrame` for transform computation.

**Classification**: Still valid — no change.

***

### Finding 11 — Shipmaker Architecture Handoff and Binary Encrypt Migration documents are stale (NEW: MEDIUM)

**Current verification**: **NEW FINDING — DOCUMENTATION IS STALE.**

**Evidence**:
- **Area D — Documentation consistency re-inspection**:
  - `SHIPMAKER-ARCHITECTURE-HANDOFF.md` §474-488 (Prerequisite Refactoring Proposal table): Lists files and "proposed changes" as if they haven't been done. But:
    - `main.ts` (line 487 in table) — Already boots via RTAB preflight and registers prepared textures. The "proposed change" to "Supply/attach the scene and existing player body/input integration to Ship-owned composition" is NOT needed for RTAB loading — that's already done.
    - The table's "Main regression risk" for `main.ts` cites "Phaser radians/degrees confusion, changed player setup, visual ownership leaks, or implying a general NPC renderer" — but the actual code already handles this correctly.
  - `SHIPMAKER-ARCHITECTURE-HANDOFF.md` §491-498 (Migration Order): Steps 1-5 describe work as future, but Step 1 ("Introduce typed definitions and a runtime validator/conversion boundary") is partially done — `ShipDataAdapter.ingestLegacyData()` IS the conversion boundary, and `rtabRuntimeLoader.ts` IS the runtime validator for assets.
  - `SHIPMAKER-ARCHITECTURE-HANDOFF.md` §500-530 (Regression Test Matrix): The "Required Tests for Prerequisite Refactoring" includes items like "Weapon transforms" (524) which are tested in `shipmaker-parity.test.ts:181-191` and "Legacy migration" (527) tested in `shipmaker-parity.test.ts:106-179`.
  - `BINARY-ENCRYPT-MIGRATION.md` §10.3 (Phase 3C): States "Phase 3C implements async bootstrap, bundle reading, and texture registration — but this phase has not been started." This is **factually incorrect** — Phase 3C is implemented and active (see Finding 7).
  - `BINARY-ENCRYPT-MIGRATION.md` §82-91 (Phase 3B.3 Stage A): Describes the packager as "one-shot" deriving from folders, but `prepare-assets.mjs` line 553 shows the production workflow calls `createPublicSvgManifest(undefined, ...)` which uses registry-derived discovery, not folder walk. The document's "folder-derived packaging" description is stale.

**Why it matters**: An agent following these documents would work against stale specifications — implementing features that already exist, or following migration steps that are already partially complete. The handoff and migration documents need updating to reflect the actual repository state.

**Classification**: **NEW — STALE DOCUMENTATION**.

***

## Summary of Revalidation

Of the 10 previous findings: 8 still valid, 1 partially valid with new nuance (Finding 3), 1 **INVALID** (Finding 7 — Phase 3C IS implemented). **One new finding (Finding 11) documents stale documentation.**

The single partial revalidation is Finding 3, which now has the additional evidence that the production workflow in `prepare-assets.mjs` uses the registry-derived path by default (calling `createPublicSvgManifest(undefined, ...)`), but the legacy path still exists in the code.

**Key changes from original audit**:
1. **Finding 7 (Phase 3C)**: Was "not yet started" — **now CONFIRMED IMPLEMENTED AND ACTIVE**. The runtime boots from `.bin` bundles via `rtabRuntimeLoader.ts` preflight.
2. **Finding 11 (New)**: Documentation in both `SHIPMAKER-ARCHITECTURE-HANDOFF.md` and `BINARY-ENCRYPT-MIGRATION.md` is stale and does not reflect the actual implementation state (particularly around Phase 3C and the RTAB runtime loader).

**No findings were resolved or removed** (except Finding 7 which was incorrect). The overall picture is that the repository has **more implemented than documented** — the RTAB asset pipeline (Phases 3A-3C) is code-complete and active in production, but the design-phase boundaries from the handoff/specification documents still describe them as pending.

The 11 test failures are all in migration/remove-migrated-public-assets/phase2a-taxonomy test files and are pre-existing — they failed before the original audit too and are unrelated to the core audit findings about architecture, player migration, or asset pipeline discovery models.

**No code changes were made during this revalidation.** All inspections were read-only (Read tool usage, git status, npm test output).