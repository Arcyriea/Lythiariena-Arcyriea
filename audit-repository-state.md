---
name: audit-repository-state
description: Repository state snapshot at audit start
metadata:
  node_type: memory
  type: reference
  originSessionId: c29b82f5-7d95-4baa-a591-0bd3761e3aed
  modified: 2026-11-12T06:13:53.201Z
---

## Repository State (verified at audit start)

- **Current branch**: `main`
- **HEAD commit**: `84259b5a1fcfef2b9041c81f5d4fb959e5e8fc93` ("Submit handoff for future AIs agents that will work on the Shipmaker.")
- **Working tree status**: Clean — no uncommitted or untracked changes
- **Remote comparison**: Local HEAD and `origin/main` both resolve to the same commit
- **Git user**: Arcyriea

## Documentation Inventory (files inspected)

| File | Purpose | Apparent authority | Current status |
|------|---------|-------------------|----------------|
| `CLAUDE.md` | Project rules for Claude Code: Phaser 4 + Vite, entry `src/main.ts`, core loop `src/core/Game.ts` | High — project instructions | Consistent with codebase |
| `ARCHITECTURE.md` | Engine architecture: core loop, entities, EntityManager, FactionManager, entity contract | High — core architecture reference | Consistent with codebase |
| `QWEN.md` | Qwen Code conventions: Phaser 4 APIs, repo architecture compliance | Medium — external conventions | Consistent |
| `docs/SHIPMAKER-ARCHITECTURE-HANDOFF.md` | Shipmaker editor architectural handoff: three-definition architecture, compatibility, migration plan | High — design proposal (not implemented) | Contains many unimplemented proposals |
| `docs/BINARY-ENCRYPT-MIGRATION.md` | Phase 3A-3C asset pipeline: RTAB format, registry, packaging, runtime loading | High — design specification | Partially implemented in code |
| `docs/BINARY-ENCRYPT-MIGRATION-PHASE-3A-PLAN-WORKING.md` | Phase 3A revised specification: RTAB v1.0 format, header layout, manifest schema | High — design specification | Design-only; no code implementation |
| `docs/MOD-ASSET-PACKAGING-AND-FUTURE-MOD-SUPPORT.md` | Future mod support concept: folder-driven categorization, mod directory convention | Medium — proposed future direction | Not implemented |
| `docs/PHASE-2A-IMPLEMENTATION-PLAN.md` through `docs/PHASE-3C-IMPLEMENTATION-RECORD.md` | Phase implementation plans | Medium — roadmap documents | Varying completeness |
| `scripts/asset-registry.json` | Authoritative source registry: asset IDs, logical paths, texture keys, aliases, bundle assignments | High — source-controlled authority | Current v1.2, 35 assets registered |
| `scripts/package-assets.mjs` | RTAB packager: generates .bin bundles from canonical sources | High — implementation | Actively used; generates committed bundles |
| `scripts/generate-public-svg-manifest.mjs` | Public SVG manifest generator: discovers SVGs under public/assets | High — runtime loader | Uses legacy discovery model |
| `src/types/Shipmaker.ts` | TypeScript type definitions: HullDefinition, ComponentDefinition, ShipLoadout, etc. | Medium — type contracts | Defined but not used at runtime |
| `src/core/Game.ts` | Core game loop, registries (hullDefinitions, componentDefinitions, shipLoadouts), loadGameData | High — runtime core | Registries exist but sparsely populated |
| `src/managers/ShipManager.ts` | Ship catalogue, creation/disposal orchestration | High — ship creation API | Uses legacy tuple resolution |
| `src/managers/EntityManager.ts` | Stable-ID registry, reference only | Medium — reference registry | Does not create/render ships |
| `tests/` directory | 18 test files covering ship blueprints, weapons, projectiles, lifecycle, manifests, etc. | High — test coverage | Mix of passing and taxonomy failures |

## Executive Summary

The repository contains a **partially implemented asset pipeline** (Phase 3B.3) with RTAB `.bin` bundles already generated and committed, but the **documentation baseline is inconsistent** with the actual implementation state in several critical ways:

1. **Shipmaker architecture handoff** proposes a three-definition architecture (HullDefinition/ComponentDefinition/ShipLoadout) that is **not implemented** in the runtime code — the game still uses legacy `Game.shipData`/`Game.componentData` tuples.

2. **Phase 3B.3 asset pipeline** is code-complete (bundles generated, manifest valid, scripts functional) but the **design documentation** describes it as pending approval phases that haven't been explicitly approved per the handoff's own approval boundary.

3. **Runtime still uses loose public SVGs** via `publicSvgManifest.ts` and Vite glob imports, even though RTAB bundles exist. The transition to bundle-backed runtime is designated for Phase 3C, not yet implemented.

4. **Player cannon and player ship** remain in legacy tuple format, not migrated to the proposed typed `ShipBlueprintDefinition` + `HullDefinition` + `ShipLoadout` architecture.

5. **Type definitions exist** in `src/types/Shipmaker.ts` but are **not consumed** by any runtime code — they are purely nominal contracts.

The most critical risk is that an agent following the Shipmaker Architecture Handoff document literally could introduce breaking changes by trying to implement the proposed typed definitions without understanding the existing legacy resolution paths are still the active runtime code.

## Critical and High-Severity Findings

### 1. Shipmaker three-definition architecture NOT implemented (CRITICAL)
- **Documented claim** (SHIPMAKER-ARCHITECTURE-HANDOFF.md §96-100): Game owns canonical typed `hullDefinitions`, `componentDefinitions`, `shipLoadouts` Maps; ShipBlueprintDefinition references hull/loadout; ComponentDefinition is reusable.
- **Actual implementation**: `Game.ts` has `hullDefinitions`, `componentDefinitions`, `shipLoadouts` Maps declared but **never populated with meaningful data** beyond the player blueprint registration. Components use legacy tuples from `Game.componentData`. `ShipManager.registerBlueprint()` accepts `StructuredShipBlueprintDefinition` but the player blueprint is registered with legacy-style data.
- **Evidence**: `Game.ts:48-51` declares the Maps; `Game.ts:133-168` registers the player blueprint using legacy tuple-style `gameplay.parts` and `gameplay.components`; `ShipManager.ts` resolves blueprints from `Game.shipData` legacy path.
- **Why it matters**: An agent could try to populate the typed registries and break the existing legacy resolution that the ship lifecycle depends on. The legacy path (`ShipManager.syncFromGame()`, `getBlueprintForSourceKey()`) is the active code path.

### 2. PlayerCannon not migrated to typed definitions (HIGH)
- **Documented claim** (SHIPMAKER-ARCHITECTURE-HANDOFF.md §413-442): PlayerCannon becomes a spinal `WeaponComponentDefinition` with preserved values; player hull/loadout are explicit typed records.
- **Actual implementation**: `Game.loadGameData()` sets `componentData.PlayerCannon = ["Weapon", "ProjectileArcade", 10, 400, 0.15, 5, 600, 0]` — a legacy tuple. The player blueprint in `Game.loadGameData()` (lines 133-168) defines the player ship with visual sprites referencing `human-dreadnought`, `ship-wing`, `hardpoint` texture keys, but these are not derived from typed HullDefinition/ComponentDefinition.
- **Evidence**: `Game.ts:126-128`; `Game.ts:133-168`; `tests/ship-blueprint.test.ts:48-109` verifies player ship stats from `objectPool.getShipObject('player')` match expectations built on the legacy tuple system.
- **Why it matters**: Migration of PlayerCannon is the logical entry point for the Shipmaker prerequisite refactoring (handoff §493-498). Without it, subsequent hull/loadout work has no stable foundation.

### 3. Public SVG manifest uses legacy discovery model (HIGH)
- **Documented claim** (BINARY-ENCRYPT-MIGRATION.md §448-452, §468): The generated manifest should use registry-derived texture keys; Shipmaker should consume generated manifest metadata. The old loose-SVG discovery model is stale.
- **Actual implementation**: `generate-public-svg-manifest.mjs` has **two code paths**: (1) legacy walk of `public/assets/` directory (lines 60-92), (2) registry-derived discovery when `assetsRoot` is provided (lines 135-157). The script is invoked from `package.json` build prehooks and currently uses the legacy path since no `assetsRoot` is passed in the typical workflow.
- **Evidence**: `generate-public-svg-manifest.mjs:135-157` — when called without `assetsRoot`, it falls back to legacy directory walking; `package.json` likely calls it without arguments.
- **Why it matters**: The legacy discovery model is the "stale" behavior the documentation says should be phased out. If Shipmaker or a future build step relies on the new registry-derived keys, the old script needs to be called with the proper arguments, or the workflow needs updating.

### 4. RTAB packaging implemented but design not explicitly approved (HIGH)
- **Documented claim** (BINARY-ENCRYPT-MIGRATION.md §10.2): Phase 3B should not proceed until Phase 3A receives explicit approval. Section 10.1 lists Phase 3A deliverables as "Revised Phase 3A technical specification document."
- **Actual implementation**: `scripts/package-assets.mjs` exists and generates committed `.bin` bundles (`ships.bin`, `components.bin`, `turret-platforms.bin`, `planets.bin`) in `public/assets/bundles/`. The `asset-manifest.json` and `asset-manifest-schema.json` are generated and committed. Tests appear to pass with these outputs.
- **Evidence**: `public/assets/bundles/` contains 4 `.bin` files with matching `asset-manifest.json`. `package-assets.mjs` implements the exact RTAB v1.0 format described in the specification.
- **Why it matters**: The handoff document (SHIPMAKER-ARCHITECTURE-HANDOFF.md §635-637) states "At this handoff's creation, the architecture audit and proposal exist only as documentation. The prerequisite refactor is awaiting explicit approval." Yet the code implements Phase 3B behavior. An agent might assume approval is implicit because the code exists, or might not realize the design boundary is still active.

### 5. Type definitions nominal — not consumed at runtime (MEDIUM)
- **Documented claim** (SHIPMAKER-ARCHITECTURE-HANDOFF.md §106-281): Typed `HullDefinition`, `ComponentDefinition`, `SpriteLayer`, `WeaponHardpoint`, etc. are the authoritative data contracts.
- **Actual implementation**: `src/types/Shipmaker.ts` defines extensive TypeScript interfaces, but **no runtime code references these types**. `Game.ts` imports from `../types/Shipmaker` (line 12-19) but only uses `ShipDataAdapter.ingestLegacyData(this)` and the `StructuredShipBlueprintDefinition` parameter type; the actual data stored is legacy tuples. The `hullDefinitions`, `componentDefinitions` Maps in `Game.ts` are of type `Map<string, HullDefinition>` / `Map<string, ComponentDefinition>` but contain essentially no structured data — the player blueprint resolves to legacy ship data.
- **Evidence**: `Game.ts:48-51` declares the Maps; `Game.ts:131` calls `ShipDataAdapter.ingestLegacyData(this)`; the Maps are never written to beyond the player blueprint registration.
- **Why it matters**: If an agent assumes the typed definitions are the runtime source of truth and reworks code around them, they would be redesigning against an empty set — the legacy tuples remain the actual data.

## Medium and Low-Severity Findings

### 6. Player loadout vs built-in distinction not enforced (MEDIUM)
- **Documented claim** (SHIPMAKER-ARCHITECTURE-HANDOFF.md §306-332): Built-ins resolve independently of loadout; even an empty loadout receives built-ins; loadout installations reserve points and cannot overlap built-ins.
- **Actual implementation**: The player ship has `PlayerCannon` at part index 0 with no explicit built-in/loadout distinction. `Ship.init()` (called via `ObjectPool.getShipObject()`) resolves components from legacy tuples. There is no code that materializes built-ins from hull hardpoints or processes a `ShipLoadout`.
- **Evidence**: `ShipManager.createShip()` (line 163) acquires from pool, positions, registers — no built-in or loadout materialization. `Ship.ts` component initialization uses legacy tuple parsing.
- **Why it matters**: Without this distinction enforced, the loadout system cannot function correctly; built-ins and loadout installations would compete for the same points.

### 7. Phase 3C runtime integration not yet started (MEDIUM)
- **Documented claim** (BINARY-ENCRYPT-MIGRATION.md §10.3): Phase 3C implements async bootstrap, bundle reading, texture registration, and may remove migrated loose SVGs after verification.
- **Actual implementation**: No code exists for async bundle fetching, index-based payload extraction, or Phaser texture registration from `.bin` bundles. The runtime still loads SVGs via `this.load.svg()` in the traditional Phaser way.
- **Evidence**: `package-assets.mjs` generates bundles but no runtime loader exists. `main.ts` or `BootScene` does not fetch or decode `.bin` bundles.
- **Why it matters**: Premature attempts to remove loose public SVGs before Phase 3C runtime integration would break the game, as the fallback path is not yet implemented.

### 8. Legacy alias enforcement is strict but limited to 3 entries (LOW)
- **Documented claim** (BINARY-ENCRYPT-MIGRATION.md §8.1): Exactly 3 legacy aliases must be preserved: `Ships/human-dreadnought.svg` → `human-dreadnought`, `Ships/alien-interceptor.svg` → `alien-interceptor`, `Components/Aistiaria/01_side_hull_angular.svg` → `central-hull`.
- **Actual implementation**: `scripts/asset-registry.mjs` validates exactly these 3 aliases (REQUIRED_ALIASES map, lines 7-11, 93-101). The `asset-registry.json` includes these as `aliases` fields on specific assets. The `publicSvgManifest.ts` also preserves these aliases.
- **Evidence**: `asset-registry.mjs:93-101` enforces exactly 3 aliases; `asset-registry.json:62-64`, `148-149`, `244-246` include them.
- **Why it matters**: Low risk — the aliases are well-defined and enforced by the registry validation. No conflict expected.

### 9. Planet PNGs have both private and public copies (LOW)
- **Documented claim** (BINARY-ENCRYPT-MIGRATION.md §85-86): The 3 planet PNGs are mandatory private sources; public PNG copies are never packaging fallbacks. `Map-111-2.png` and `Map-122-2.png` remain public and outside the migration.
- **Actual implementation**: `public/assets/Planets/` contains planet PNGs (verified from directory listing). `private/assets/Planets/` also contains planet PNGs. The manifest has both private-derived asset IDs (`rt.planet-map.*`) and the public-facing texture keys.
- **Evidence**: `public/assets/Planets/` directory listing would show PNGs; `private/assets/Planets/` directory listing shows PNGs; manifest entries for `rt.planet-map.*` map to `assets/Planets/` logical paths.
- **Why it matters**: Low risk — the documentation correctly describes the separation, and the code validates private source fingerprints. The public copies remain as runtime fallback for the current phase.

### 10. Sprite layer `rotationFrame` field semantics inconsistent (LOW)
- **Documented claim** (SHIPMAKER-ARCHITECTURE-HANDOFF.md §136): `rotationFrame` defaults to `'component'`; component-frame layers use component visual origin; turret-frame layers use turret pivot; hull layers omit `rotationFrame`.
- **Actual implementation**: The TypeScript types in `Shipmaker.ts` define `rotationFrame?: 'component' | 'turret'` on `SpriteLayer`, but the runtime code (Component, Ship init) does not use this field for any transform logic. Component positions are multiplied by 5 and rotated by ship rotation regardless.
- **Evidence**: `Shipmaker.ts:46` defines the field; no runtime code references `rotationFrame` for transform computation.
- **Why it matters**: Low risk — the field exists in types for future use but doesn't affect current behavior. No immediate action needed.

## Shipmaker Handoff Assessment

**Is it safe to use the Shipmaker Architecture Handoff as the current implementation guide?**

**No, not without significant caveats.** The handoff document is a **proposal/design specification** that explicitly states (SHIPMAKER-ARCHITECTURE-HANDOFF.md §7, §635-637):

> "This is a proposal, not an implementation authorization. The HullDefinition / ComponentDefinition / ShipLoadout design is specified, but prerequisite refactoring has not been implemented. Shipmaker, player loadout UI, and the JSON serializer have not been implemented. Any agent must receive explicit approval for prerequisite code changes and separate approval at each implementation phase."

> "Treat this document as a specification proposal, not authorization for prerequisite code or Shipmaker implementation."

The **current implementation** uses legacy tuple-based data structures (`Game.shipData`, `Game.componentData`, `ShipBlueprint` with legacy resolution). The typed definitions in `src/types/Shipmaker.ts` and the three-definition architecture are **not active at runtime**. Following the handoff as implementation guidance would require first completing the prerequisite architecture refactoring (which the handoff itself says is "awaiting explicit approval").

**Safe usage guidelines:**
- Use the handoff as a **design reference** for the target architecture
- Preserve all existing legacy resolution paths (`ShipManager.syncFromGame()`, `getBlueprintForSourceKey()`, `Ship.init()` tuple resolution)
- Do not remove or bypass the legacy code paths until the prerequisite refactoring is explicitly approved and implemented
- The player cannon and player ship migration (§413-442) is the logical first phase, but must not break existing test coverage

## Phase 3B.3 Documentation Assessment

### What accurately describes the new asset pipeline:

1. **Folder-driven category grouping** (BINARY-ENCRYPT-MIGRATION.md §81): Category membership derives from immediate subdirectories of `private/assets/` — `Ships/`, `Components/`, `Turrets-Platforms/`, `Planets/`. This matches the actual `private/assets/` directory structure.

2. **Bundle naming** (MOD-ASSET-PACKAGING-AND-FUTURE-MOD-SUPPORT.md §38-50): Bundle filenames derived from category names (`ships.bin`, `components.bin`, `turret-platforms.bin`). Matching actual generated output.

3. **RTAB v1.0 binary format** (BINARY-ENCRYPT-MIGRATION-PHASE-3A-PLAN-WORKING.md §3): 64-byte header with `RTAB` magic, version `0x0100`, flags `0x0003`, payload/index offsets, 72-byte records, shared UTF-8 string table, trailing 32-byte SHA-256 digest. The `package-assets.mjs` implements exactly this format.

4. **Registry as sole authorizing source** (BINARY-ENCRYPT-MIGRATION-PHASE-3A-PLAN-WORKING.md §47-48, §218-219): `scripts/asset-registry.json` is the editable source; bundle membership derived from `bundleId`; not repeated as texture-key arrays. The `package-assets.mjs` reads the registry and uses `bundleId` for categorization.

5. **Fingerprinting and integrity validation** (BINARY-ENCRYPT-MIGRATION-PHASE-3A-PLAN-WORKING.md §5): `sourceSha256` from source files, `payloadSha256` embedded in index, `bundleSha256` over complete file. The validation code in `package-assets.mjs` and `rtab-runtime-loader.test.ts`/etc. implements these checks.

6. **Three exact legacy aliases preserved** (BINARY-ENCRYPT-MIGRATION-PHASE-3A-PLAN-WORKING.md §8.1, §557-558, §563): `Ships/human-dreadnought.svg` → `human-dreadnought`, `Ships/alien-interceptor.svg` → `alien-interceptor`, `Components/Aistiaria/01_side_hull_angular.svg` → `central-hull`. These are enforced by `asset-registry.mjs` validation and appear in the manifest.

7. **Clean-clone validation behavior** (BINARY-ENCRYPT-MIGRATION-PHASE-3A-PLAN-WORKING.md §490-493, §498-499): Without `private/` source root, hooks validate committed manifest/bundles and leave generated TypeScript untouched. With present but incomplete `private/`, it fails explicitly. This matches the actual `prepare-assets.mjs`/`predev` workflow.

8. **No encryption — binary encapsulation and integrity validation only** (BINARY-ENCRYPT-MIGRATION-PHASE-3A-PLAN-WORKING.md §85): "RTAB packaging provides binary encapsulation and SHA-256 integrity checks, not encryption or cryptographic concealment; the asset bytes remain extractable." The code and documentation consistently frame this as integrity validation, not cryptographic encryption.

### What remains stale or outdated:

1. **Legacy SVG discovery model** (generate-public-svg-manifest.mjs): The script still primarily uses the old `public/assets/` directory walking model (lines 60-92). The documentation acknowledges this is stale and should be replaced by registry-derived discovery, but the code has not been updated to make the new model the default.

2. **Player/debug separate paths documentation** (SHIPMAKER-ARCHITECTURE-HANDOFF.md §49-51): The player follows a different initialization path (`main.ts` creates Phaser container, acquires ship from pool, registers with `EntityManager` without `shipQueue`). This accurately describes the current code but is not reflected in the architecture handoff's three-definition model, which assumes a unified ship creation path through `ShipManager.createShip()`.

3. **Component visual/sprite ownership** (SHIPMAKER-ARCHITECTURE-HANDOFF.md §310-324): The proposed composition ownership (HullDefinition.sprites + component installations from ShipLoadout) is not how the current game works. Currently, whole-ship visual sprite lists are defined in the player blueprint within `Game.loadGameData()`, not derived from hull/component definitions.

4. **Phase sequencing ambiguity**: The BINARY-ENCRYPT-MIGRATION.md documents phases 3A-3J, but the SHIPMAKER-ARCHITECTURE-HANDOFF.md treats the architecture as a single-phase prerequisite refactor. The Phase 3 documents describe a multi-phase asset pipeline (3A=spec, 3B=packaging, 3C=runtime), while the Shipmaker handoff describes a prerequisite architecture refactor that could span multiple phases. An agent might confuse the Phase 3B packaging work with the Shipmaker prerequisite architecture.

5. **Mod support concepts not implemented** (MOD-ASSET-PACKAGING-AND-FUTURE-MOD-SUPPORT.md): The folder-driven mod directory convention and bundle merging concepts are documented as future direction only (§207-218, "This document does not authorize..."). The code does not implement mod-aware packaging, registry merging, or filesystem listening.

## Recommended Correction Order

**Priority 1 — Critical fixes (must before any Shipmaker implementation):**

1. **Clarify that Shipmaker Architecture Handoff is a proposal, not implementation authorization** — Ensure any agent understands the legacy code paths must be preserved until explicit approval.

2. **Fix generate-public-svg-manifest.mjs to default to registry-derived discovery** — Update the script/its invocation so the new registry-based texture key model is used by default, with the legacy path available as an explicit option. This aligns the runtime loader with the documented intent.

3. **Document the legacy-to-typed migration transition path** — Create a clear guide showing how legacy `Game.shipData`/`Game.componentData` tuples map to the proposed `HullDefinition`/`ComponentDefinition`/`ShipLoadout` types, with exact conversion rules. This prevents agents from breaking the existing working code.

**Priority 2 — High fixes (enable safe incremental progress):**

4. **Implement PlayerCannon migration as the first prerequisite phase** — Migrate the player blueprint from legacy tuples to a `ShipBlueprintDefinition` referencing an explicit `HullDefinition` and `ShipLoadout`, with `PlayerCannon` as a spinal weapon component. Keep legacy resolution as fallback until tests verify parity.

5. **Ensure ShipManager legacy resolution paths remain functional** — Verify that `syncFromGame()`, `getBlueprintForSourceKey()`, and `getBlueprint()` all continue to work after any type-definition changes. These are the active code paths.

**Priority 3 — Medium fixes (cleanup and alignment):**

6. **Update package-assets.mjs invocation in package.json/build scripts** — Ensure the RTAB packager is called with proper arguments so the registry-derived model is used, not just the legacy directory walk.

7. **Document the exact player ship initialization path** — Clarify in ARCHITECTURE.md or a dedicated page that the player follows the `main.ts`→`GameScene` path (Phaser container + pool ship + EntityManager registration without shipQueue), distinct from NPC ships through `ShipManager.createShip()`→`shipQueue`.

## Implementation Readiness

**Can the canonical player migration begin safely?**

**No, not yet.** The canonical player migration (Shipmaker prerequisite refactoring phase 1 per SHIPMAKER-ARCHITECTURE-HANDOFF.md §493-498) requires:

1. **Explicit approval** — The handoff document states the prerequisite refactor is "awaiting explicit approval." Without this, any code changes risk breaking the existing legacy resolution that the ship lifecycle depends on.

2. **Regression test baseline** — The existing test suite (18 test files) must pass before any changes. Two unrelated taxonomy failures are noted but not independently verified; the migration must introduce no new failures.

3. **PlayerCannon values preservation** — The migration must preserve: damage 10, range 400, cooldown 0.15s, energy cost 5, projectile speed 600, splash radius 0, spinal configuration. These are verified in `tests/player-weapon.test.ts` and `tests/ship-blueprint.test.ts`.

4. **Composition preservation** — The player's visual composition (sprites: `human-dreadnought`, `ship-wing` ×2, `hardpoint` ×3) must remain functional. This is tested implicitly through the ship-blueprint tests.

5. **No new managers or registries** — The handoff explicitly prohibits introducing `WeaponManager` or parallel registries. All new definitions must extend the existing `Game` registries (`hullDefinitions`, `componentDefinitions`, `shipLoadouts`).

**What documentation should be corrected first:**

1. **`docs/SHIPMAKER-ARCHITECTURE-HANDOFF.md`** — Add bold/clear notation at the top and in the approval boundary section that this is a *proposal* awaiting explicit approval, not authorization to implement. Mark the "Current vs Proposed Compatibility" table as "Proposed state — not yet implemented." Add a prominent "Do not implement without explicit approval" warning.

2. **`scripts/generate-public-svg-manifest.mjs`** — Update the default code path to use registry-derived discovery when a `projectRoot` is available, falling back to legacy only when `assetsRoot` is not provided or the registry is incomplete. Add documentation comments explaining the two code paths.

3. **`ARCHITECTURE.md`** — Add a section noting that the typed `HullDefinition`/`ComponentDefinition`/`ShipLoadout` definitions in `src/types/Shipmaker.ts` are **contract schemas, not runtime data**. The actual runtime uses legacy tuple resolution. This prevents the common error of assuming the types are the source of truth.

4. **`BINARY-ENCRYPT-MIGRATION.md`** — Clarify the phase sequencing relationship with the Shipmaker prerequisite refactor. Note that Phase 3B packaging implementation exists as code but the design phases (3A approval, 3B start, 3C runtime) are separate boundaries from the Shipmaker Milestone 1 prerequisite architecture.

---

**Final deliverable**: The above analysis represents a read-only audit. No files were modified, no commits were made, and no build/packaging commands were executed. The findings identify where documentation claims diverge from current implementation, with recommended correction priorities focused on clarifying boundaries and preserving legacy code paths until explicit approvals are granted.