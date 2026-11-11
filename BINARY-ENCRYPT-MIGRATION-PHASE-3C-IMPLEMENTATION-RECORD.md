# Phase 3C Runtime Migration — Implementation Record

**Project:** Redemptis-Tiariea
**Record date:** 2026-11-11
**Scope:** Browser runtime migration from loose public game assets to the existing RTAB bundles.
**Status:** Implementation is present in the worktree; this record does not imply that the changes have been committed or that public loose assets have been removed.

## 1. Implementation overview

Phase 3C changes startup so the application fetches and validates the RTAB manifest and all four bundles, decodes all 34 game images, and only then constructs `Phaser.Game`. The prepared `HTMLImageElement`s are handed to BootScene. BootScene registers the bundled images and aliases into Phaser's global Texture Manager, verifies the keys, and starts GameScene only after successful registration.

The runtime assets are the 31 registered game SVGs and three registered planet PNGs. The three minimap ship icons remain independently loaded SVGs. Existing dummy textures, planet shader behavior, camera setup, and public source files remain in place. This is runtime loading work only; it does not remove public asset copies or change the RTAB producer or format.

The implementation report and prior browser run described in this record come from the Phase 3C work in this session. For this documentation task, the worktree, source files, tests, hashes, compiler, production build, and full test suite were re-inspected or rerun. The prior interactive browser checks were not replayed during this documentation-only task; they are explicitly identified in the verification section.

## 2. Complete file inventory

The Git inventory at the start of this documentation task was:

```text
 M src/main.ts
?? src/assets/rtabRuntimeLoader.ts
?? tests/rtab-parser.test.ts
?? tests/rtab-runtime-loader.test.ts
```

`git diff --stat` reported one tracked file with 60 insertions and 37 deletions. `git diff` therefore contains only the tracked changes to [src/main.ts](../src/main.ts); the three untracked files do not appear in that diff. `git ls-files --others --exclude-standard` reported the two tests and runtime loader. No deletion was reported. No staged changes were present.

| State at inspection | File | Role in Phase 3C |
|---|---|---|
| **Tracked, modified** | [src/main.ts](../src/main.ts) | Replaces loose game-asset loads with preflight, passes prepared images into BootScene, and moves core `Game` construction into GameScene `create()`. |
| **Untracked addition** | [src/assets/rtabRuntimeLoader.ts](../src/assets/rtabRuntimeLoader.ts) | Browser manifest validator, RTAB reader, SHA-256 checks, SVG sizing/decode path, planet checks, and Phaser texture registration. |
| **Untracked addition** | [tests/rtab-parser.test.ts](../tests/rtab-parser.test.ts) | Manifest and binary-format tests using the committed manifest and bundle outputs. |
| **Untracked addition** | [tests/rtab-runtime-loader.test.ts](../tests/rtab-runtime-loader.test.ts) | Preflight, planet pin, image dimension/decode, texture registration, collision, and rollback tests. |
| **New ignored file from this documentation task** | [docs/PHASE-3C-IMPLEMENTATION-RECORD.md](./PHASE-3C-IMPLEMENTATION-RECORD.md) | This implementation record. It is excluded from ordinary Git status by `.gitignore`'s `docs/*` rule. |
| **Unchanged, runtime identity source** | [src/generated/publicSvgManifest.ts](../src/generated/publicSvgManifest.ts) | Existing 31 SVG keys, public URLs, and aliases; imported by the runtime loader as an SVG identity check. |
| **Unchanged, registry source** | [scripts/asset-registry.json](../scripts/asset-registry.json) | Registry schema 1.2 identities and runtime metadata imported by the browser loader to form its expected inventory. |
| **Unchanged, emitted manifest** | [public/assets/asset-manifest.json](../public/assets/asset-manifest.json) | Manifest schema 1.2; 34 records and four bundle records. |
| **Unchanged, emitted schema** | [public/assets/asset-manifest-schema.json](../public/assets/asset-manifest-schema.json) | JSON Schema draft 2020-12 description for manifest 1.2. The browser loader does not fetch or execute this schema file; it performs explicit checks in TypeScript. |
| **Unchanged, emitted bundles** | [public/assets/components.bin](../public/assets/components.bin), [public/assets/ships.bin](../public/assets/ships.bin), [public/assets/turrets-platforms.bin](../public/assets/turrets-platforms.bin), [public/assets/planets.bin](../public/assets/planets.bin) | RTAB inputs consumed by the runtime. |
| **Unchanged, Node-side packaging and validation** | [scripts/package-assets.mjs](../scripts/package-assets.mjs), [scripts/asset-registry.mjs](../scripts/asset-registry.mjs), [scripts/asset-roots.mjs](../scripts/asset-roots.mjs), [scripts/validate-png.mjs](../scripts/validate-png.mjs), [scripts/prepare-assets.mjs](../scripts/prepare-assets.mjs), [scripts/asset-output-publisher.mjs](../scripts/asset-output-publisher.mjs) | Existing canonical-source, packaging, PNG, committed-output, and publication checks. No Node validator or binary producer was modified for Phase 3C. |
| **Unchanged, shader and planet behavior** | [src/entities/background/Planet.ts](../src/entities/background/Planet.ts), [src/shaders/planets.glsl](../src/shaders/planets.glsl) | Existing Shader Game Object setup, planet geometry, sampler, colors, and parallax behavior. |
| **Unchanged, core game engine** | [src/core/Game.ts](../src/core/Game.ts) | The project's `Game` implementation and its managers/data initialization are unchanged; only the timing of its construction in `main.ts` moved. |
| **Unchanged, minimap** | [src/ui/MiniMap.ts](../src/ui/MiniMap.ts) | Existing separate minimap icon paths and `hardpoint` arrow texture use. |
| **Unchanged, validation references** | [tests/package-assets.test.ts](../tests/package-assets.test.ts), [tests/validate-png.test.ts](../tests/validate-png.test.ts), [tests/public-svg-manifest.test.ts](../tests/public-svg-manifest.test.ts), [tests/ship-lifecycle.test.ts](../tests/ship-lifecycle.test.ts) | Existing packaging, PNG, SVG manifest, and ship texture identity coverage. |
| **Unchanged, app shell/configuration** | [index.html](../index.html), [package.json](../package.json), [tsconfig.json](../tsconfig.json), [ARCHITECTURE.md](../ARCHITECTURE.md), [README.md](../README.md), [QWEN.md](../QWEN.md), [CLAUDE.md](../CLAUDE.md), [docs/BINARY-ENCRYPT-MIGRATION.md](./BINARY-ENCRYPT-MIGRATION.md), [.gitignore](../.gitignore) | Existing page container, dependencies/scripts, architecture and tool-specific guidance, phase boundary notes, and private directory ignore rule. |
| **Unchanged, installed Phaser reference** | [SVGFile.js](../node_modules/phaser/src/loader/filetypes/SVGFile.js), [TextureManager.js](../node_modules/phaser/src/textures/TextureManager.js), [phaser.d.ts](../node_modules/phaser/types/phaser.d.ts), and [node_modules/phaser/package.json](../node_modules/phaser/package.json) | Installed Phaser 4.2.1 loader/texture APIs and version used as compatibility references; these are dependency files, not project modifications. |

No source file was modified by this documentation task. No private source file or private image content is included here.

No repository `AGENTS.md` or `CONVENTIONS.md` was present at inspection. `QWEN.md` references a conventions file; `CLAUDE.md` marks itself as Claude Code-specific. Both were inspected as repository guidance context.

## 3. Source-file documentation

### `src/assets/rtabRuntimeLoader.ts`

This untracked module owns browser-side validation and prepared texture registration. It imports the checked-in [asset registry](../scripts/asset-registry.json) and [generated SVG manifest](../src/generated/publicSvgManifest.ts). It does not import Node's package writer or Node crypto.

Important exported interfaces and functions:

- `RuntimeAssetMetadata`, `RuntimeBundleMetadata`, `RuntimeManifest`, `PreparedRuntimeAsset`, and `PreparedRuntimeAssets` describe validated manifest records and decoded browser images.
- `TextureManagerLike` is the small texture-manager shape accepted by the registration function and used by Node tests.
- `APPROVED_PLANET_SHA256` stores the three independent planet pins.
- `validateRuntimeManifest(value)` checks exact root/asset/bundle keys, schema version, field types, SHA formats, registry identity, SVG identity against `publicSvgTextures`, category-to-bundle assignment, counts, aliases, and approved planet manifest hashes.
- `validateRtabBundle(input, bundle, expectedAssets, subtle)` parses one complete bundle, checks its full SHA-256 and binary structure, checks each record and payload digest, matches the expected records, and returns decoded records containing the asset metadata and payload bytes.
- `resizeSvgForPhaser(svgText, scale)` implements the current Phaser SVG sizing rules for the RTAB decode path.
- `preflightRtabAssets(options)` fetches the manifest and all required bundles, validates them, checks cross-bundle coverage, decodes every image, checks planet dimensions, and returns prepared assets.
- `expectedTextureNames(prepared)` requires 34 assets and 37 unique primary/alias texture names.
- `registerPreparedTextures(textureManager, prepared)` checks all keys for collisions before adding any images, adds primary keys and aliases, verifies all keys exist, and removes only expected names that exist if registration throws.

### `src/main.ts`

This tracked file still defines both scenes and the Phaser config. It imports `preflightRtabAssets`, `registerPreparedTextures`, and the prepared-asset type. `bootstrap()` awaits preflight before calling `new Phaser.Game(config)`. It passes a prepared value and error callback to a constructed BootScene instance in the scene array. `displayStartupError()` replaces any prior RTAB error overlay with a fixed full-screen `role="alert"` / `aria-live="assertive"` DOM element; it places the error detail using `textContent`.

BootScene `preload()` continues to queue the three minimap SVGs with scale 0.5 and calls `createDummyTextures()`. It no longer queues the game SVGs or planet PNGs from public URLs. BootScene `create()` calls `registerPreparedTextures`, then calls `this.scene.start('GameScene')` only if registration succeeds; on an exception it reports the error and does not start GameScene.

GameScene's constructor now only calls `super('GameScene')`. Its `create()` constructs `new Game()` before existing gameplay-scene setup. The existing three `PlanetBackground.addPlanet()` calls retain `planetMap692`, `planetMap122`, and `planetMap111` with their former positions, sizes, depth, rotation, and parallax settings.

The module imports Phaser at module load time, but does not construct `Phaser.Game` until preflight resolves. The project `Game` object is also created only after BootScene starts GameScene.

### New parser and loader tests

[tests/rtab-parser.test.ts](../tests/rtab-parser.test.ts) reads the current committed manifest and bundles, validates the live expected inventory, and creates in-memory altered bundle copies to exercise failures. It does not rewrite the committed bundles.

[tests/rtab-runtime-loader.test.ts](../tests/rtab-runtime-loader.test.ts) injects a fetch function, Web Crypto implementation, and image decoder to test preflight behavior in Node. Its `FakeTextureManager` verifies registration logic and rollback without booting Phaser. These tests do not themselves prove browser image decoding or WebGL output; those checks are reported separately below.

## 4. RTAB parser specification

The browser parser in `validateRtabBundle()` implements the existing layout written by [scripts/package-assets.mjs](../scripts/package-assets.mjs). Integer fields are read little-endian through `DataView`; uint64 offsets are retained as `bigint` until checked or converted to a safe JS number.

### Header — 64 bytes

| Offset | Size | Implemented check/value |
|---:|---:|---|
| 0 | 4 | ASCII magic `RTAB`. |
| 4 | 2 | `0x0100` means RTAB 1.0; `0x0101` means RTAB 1.1. Unknown versions reject. Header version must equal the manifest bundle version. |
| 6 | 2 | Header flags must equal `0x0003`. |
| 8 | 8 | Payload section offset, uint64 LE. |
| 16 | 8 | Index offset, uint64 LE; must equal 64. |
| 24 | 8 | Index size, uint64 LE. |
| 32 | 32 | All reserved bytes must be zero. |

The complete byte array SHA-256 is checked against the manifest `bundleSha256` before deeper parsing. The index must immediately follow the 64-byte header; payload offset must immediately follow the index; the payload must be nonempty and within the fetched bundle.

### Index and string table

The index begins with a 28-byte header: `INDX` at offset 0, uint32 record count at 4, uint16 record size 72 at 8, uint16 index flags 0 at 10, uint64 string-table offset at 12, and uint64 string-table size at 20. The string table offset must equal `28 + recordCount * 72`. The full index size must equal its header, record table, string table, and 32-byte terminal digest. Its SHA-256 is computed over all index bytes except that final digest and compared to the terminal 32 bytes.

The string table is UTF-8. Record string offsets and lengths must be contiguous and in range; the accumulated referenced length must consume the entire string table. IDs must decode with fatal UTF-8, be nonempty, unique, and strictly ascending. Records and payload ranges may not contain gaps or trailing unindexed data.

### Record and payload validation

Each 72-byte record has: uint32 string offset (0), uint32 ID byte length (4), uint64 payload-relative offset (8), uint64 stored size (16), uint64 original size (24), uint16 content code (32), uint16 compression code (34), uint32 flags (36), and 32-byte payload SHA-256 (40).

- Content code 0 is SVG; code 1 is PNG. RTAB 1.0 accepts SVG only. RTAB 1.1 accepts SVG and PNG. Manifest identity/type checks further constrain this application to SVG assets in components/ships/turrets-platforms and PNG assets in planets.
- Compression code must be zero (`NONE`); record flags must be zero.
- Stored size must be nonzero, equal original size, and no greater than uint32 max.
- Record payload offsets must be contiguous starting at zero and remain inside the payload section.
- Each payload hash must match both the record digest and manifest `sourceSha256`.
- Each record ID must map to an expected manifest asset. Bundle count and sorted ID sequence must exactly match that bundle's expected manifest assets.

The implementation checks `bigint` ranges and safe-number conversions, but it has no additional explicit browser bundle-byte cap. The unchanged Node packager/validator retains its own uint32/uint64 and file-format checks.

## 5. Manifest and registry validation

The browser fetches exactly `/assets/asset-manifest.json`, then calls `validateRuntimeManifest()`. The implementation manually validates the runtime contract rather than fetching/evaluating `asset-manifest-schema.json`.

It requires exact root keys `manifestSchemaVersion`, `assets`, `bundles`, version string `1.2`, exactly 34 asset objects, and exactly four bundle objects. Asset and bundle objects must have exactly their expected property names. Hashes must match lowercase 64-character hexadecimal syntax. Runtime metadata is matched by asset ID against the statically imported registry JSON, including logical path, texture key, aliases, content type, and category-derived `bundleId`. It also rejects duplicate IDs and duplicate primary texture/alias names.

For SVGs it decodes each URL from `publicSvgTextures` into a logical path and compares that path's key and alias list with registry metadata. It requires 31 SVGs and three PNGs. The expected bundle constants are components (19, v1.0), planets (3, v1.1), ships (10, v1.0), and turrets-platforms (2, v1.0); manifest filenames, versions, and counts must match those constants. Assets are grouped by manifest bundle ID, sorted by asset ID for record matching, and after parsing all bundles the loader requires 34 unique record IDs and every manifest ID present.

The runtime imports the registry file as bundled JS data; it does not fetch the registry from the server. The registry schema version is not independently revalidated in the browser module; the checked-in registry's assets are used as the identity contract.

## 6. Integrity verification

Integrity checks in the actual implementation are:

1. Complete bundle SHA-256 against each manifest bundle's `bundleSha256`.
2. SHA-256 over index bytes excluding the final 32-byte digest against the index digest.
3. SHA-256 over every payload against its record digest.
4. The same payload SHA-256 against the manifest asset's `sourceSha256`.
5. For each PNG, the manifest source hash must independently equal the pinned hash, and the actual payload hash must equal that same pin.

The runtime pins are:

| Logical path | Pinned SHA-256 |
|---|---|
| `assets/Planets/Map-111.png` | `83d39b7a7603f574f47cee57c4d9b12b0b468b469fece0ffe7765d921f067ea1` |
| `assets/Planets/Map-122.png` | `35f8af626f4d7162a2926ef3dad6cdec36610cfd6f3acac86a1547d764081f7e` |
| `assets/Planets/Map-692.png` | `43652c4465b35a1e3ae4bf99bbb75308359179c9d1b3fd8711d38c806b1472a8` |

The manifest and bundle hashes provide consistency checks; they are not a digital signature or an independent authenticity mechanism for SVGs if both manifest and bundle are replaced by an attacker. The planet hashes are independently pinned in the JS runtime.

## 7. SVG and PNG decoding

`resizeSvgForPhaser()` uses `DOMParser` in `text/xml` mode, fails if a parser-error node or missing SVG root is found, and follows the current Phaser SVGFile scale behavior for `0.5`:

- Parses root width/height with `parseFloat`.
- If `viewBox` is absent and both dimensions are truthy, inserts `viewBox="0  0 <width> <height>"`.
- If a `viewBox` exists while both parsed dimensions are falsy, splits on whitespace or commas and uses entries 2 and 3.
- If both resulting dimensions are truthy, sets width and height to each dimension times 0.5 followed by `px`, serializing the SVG root.
- If dimensions remain unresolved, returns the original SVG text unchanged. A present `viewBox` is retained.

The browser reference for this sizing rule is Phaser 4.2.1 `SVGFile.js`; the installed package reports 4.2.1. The runtime creates a Blob with MIME `image/svg+xml;charset=utf-8` for SVGs and `image/png` for PNGs, creates an object URL, assigns it to a new `Image`, and awaits `image.decode()`. The object URL is revoked in `finally`; the decoded `HTMLImageElement` remains in the prepared asset value for registration. Empty natural dimensions reject. Each planet must decode to exactly 2000×1000.

The implementation does **not** reproduce Phaser SVGFile's legacy data-URI retry path for browsers where the initial object-URL image load fails. Such failures reject preflight and display the startup error. The browser code does not reimplement PNG chunk CRC, RGBA profile, or inflated scanline validation: approved PNG bytes are hash-pinned and browser-decoded with dimension checks, while the existing Node-side `validate-png.mjs` checks remain unchanged.

## 8. Phaser integration and preserved behavior

The top-level `bootstrap()` awaits all manifest/bundle validation and image decoding before `new Phaser.Game(config)`. Phaser is statically imported before this call, but the Phaser application object is not constructed before preflight completes. Prepared assets are passed into `new BootScene(preparedAssets, displayStartupError)` in the GameConfig scene list; there is no module-level prepared-assets global.

BootScene still loads:

- `minimap-capital-ship` from `/minimap/Capital-Ship.svg`;
- `minimap-corvette` from `/minimap/Corvette.svg`;
- `minimap-fighter` from `/minimap/Fighter.svg`;
- and creates dummy `ship-core`, `ship-wing`, `hardpoint`, `bullet`, and `enemy-core` textures.

In `create()`, it registers each of the 34 prepared images under its primary key and registers the three aliases using the same image objects. It prechecks all 37 names for collision before any additions, verifies all names after registration, and only then starts GameScene. The project's `Game` core instance is constructed at the start of GameScene `create()`, not in the GameScene constructor. The `PlanetBackground`, `PlanetShader`, shader source, positions, sizes, rotations, parallax values, camera logic, minimap implementation, and gameplay APIs are otherwise unchanged.

Phaser 4.2.1's installed `TextureManager.addImage` API accepts `HTMLImageElement` and returns `null` on an already-used key. The runtime uses that API. Installed `TextureManager.remove()` removes and destroys the named texture; the rollback only iterates this attempt's expected 37 names and removes those that now exist after a caught registration failure. Preexisting keys are protected by the up-front collision pass.

## 9. Failure and rollback behavior

- A non-OK manifest or bundle response rejects preflight. Bundle requests are issued in parallel after manifest validation.
- Invalid JSON, manifest structure/identity, bundle hash, header/index, digest, version/content, payload hash, or asset coverage reject preflight.
- A planet manifest hash different from its pin, a changed planet payload, or decoded dimensions other than 2000×1000 rejects preflight.
- SVG XML errors, browser image decode errors, or zero image dimensions reject preflight.
- Preflight errors are caught by `bootstrap()` and rendered in a fixed, full-screen DOM overlay with an alert role. Since the catch occurs before `new Phaser.Game`, those failures create no Phaser canvas and no GameScene.
- Texture-key collision is detected in BootScene after Phaser exists but before registration. If registration or post-registration existence checks fail, the catch calls the same overlay callback, does not call `scene.start('GameScene')`, and rolls back keys created by the attempted registration. Phaser itself is not explicitly destroyed on this path; the BootScene and application remain running under the overlay.
- The Node tests verify helper-level collision and rollback behavior with a fake Texture Manager; browser failure checks described below cover missing and corrupted bundles before Phaser construction.

The overlay shows the exception message via `textContent`. It has no retry button or recovery flow in this implementation.

## 10. Tests and verification

### Committed new test cases

`tests/rtab-parser.test.ts` contains tests for:

1. Schema 1.2 and the exact registered 34-asset inventory.
2. Unsupported manifest version, duplicate identity, and unexpected identity.
3. RTAB 1.0 SVG and RTAB 1.1 PNG bundle parsing and record verification.
4. Altered complete-bundle fingerprint.
5. Unsupported header flags after recomputing the complete bundle hash.
6. Invalid index digest and truncated bundle.
7. RTAB 1.0 PNG rejection and nonzero record flags.
8. Payload corruption, unsafe offsets, and source-fingerprint mismatch.
9. Manifest and binary format-version mismatch.

`tests/rtab-runtime-loader.test.ts` contains tests for:

1. Four-bundle preflight, decoding 34 assets, and 37 texture names.
2. Altered planet payload pin rejection.
3. Changed planet manifest pin rejection before bundle requests.
4. Incorrect decoded planet dimensions.
5. Image decode failure.
6. Missing bundle rejection before image decoding.
7. Corrupted bundle rejection before decoding.
8. Registration of all primary keys and aliases using the same image.
9. Existing texture collision with no mutation.
10. Rollback of keys added by the failed registration attempt while preserving a preexisting texture.
11. Presence of the three planet names and three aliases.

### Commands rerun during this record task

- `npx tsc --noEmit` — passed.
- `npx tsx --test tests/rtab-parser.test.ts tests/rtab-runtime-loader.test.ts` — 20 passed, 0 failed.
- `npx vite build` — passed; Vite emitted its warning that the minified main chunk exceeds 500 kB. This command bypasses `prebuild`; the package script `npm run build` has a `prebuild` preparation hook.
- `npm test` — 231 passed, 2 failed, 233 total. The failures are the existing unrelated Phase 2A taxonomy tests: `unverified legacy Skill, Passive, and Spawn remain in compatibility path and not in canonical loadout` at `tests/phase2a-taxonomy.test.ts:299`, and `existing PlayerCannon runtime behavior is preserved` at `tests/phase2a-taxonomy.test.ts:377`.
- `git diff --check` — passed after implementation and again after this document is added.

### Browser and WebGL checks recorded from the prior Phase 3C verification run

These checks were previously performed through the integrated browser tools; they were not replayed during this documentation-only task:

- A browser Phaser 4.2.1 reference scene loaded each of the 31 loose SVGs with `load.svg(..., { scale: 0.5 })`. All 31 output dimensions matched the preflight-decoded RTAB images; zero mismatches were reported.
- A real Phaser WebGL Texture Manager probe registered all 34 primary textures and all three aliases; it reported 37 keys present.
- A separate WebGL verification scene rendered all three bundled planet maps with the existing `PlanetBackground`/`PlanetShader` implementation.
- The normal application resource list contained the manifest, four RTAB bundle requests, and the three `/minimap/*.svg` requests; no `/assets/...svg` or `/assets/Planets/*.png` runtime request was observed.
- A route that made `ships.bin` unavailable produced the visible overlay and zero `#app canvas` elements. A route that returned a corrupted components bundle likewise produced an overlay and zero canvases.
- Decode failure and registration failure/rollback were verified in the focused automated tests, not by separate browser fault injection.

## 11. Asset migration status

| Runtime group | Count | Current load path |
|---|---:|---|
| Components SVGs | 19 | `components.bin`, RTAB 1.0, then Phaser Texture Manager. |
| Ship SVGs | 10 | `ships.bin`, RTAB 1.0, then Phaser Texture Manager. |
| Turret/platform SVGs | 2 | `turrets-platforms.bin`, RTAB 1.0, then Phaser Texture Manager. |
| Planet PNGs | 3 | `planets.bin`, RTAB 1.1, independently pinned and decoded to 2000×1000. |
| Minimap ship icon SVGs | 3 | Still loaded separately by Phaser Loader from `/minimap/Capital-Ship.svg`, `/minimap/Corvette.svg`, and `/minimap/Fighter.svg`. |

All 31 loose public game SVG files and the three public copies of the planet PNGs remain present. They are no longer queued by BootScene and no longer serve as runtime fallback paths. `Map-111-2.png` and `Map-122-2.png` remain outside the RTAB inventory. The page's `/favicon.svg` link is also separate from the migrated game asset inventory.

The 34 registered runtime identities are listed below. Each logical path still has its loose public counterpart at `public/<logicalPath>`; the table describes bundle membership and runtime keys, not file deletion.

| Asset ID | Bundle/type | Logical path | Runtime key | Alias |
|---|---|---|---|---|
| `rt.component.aistiaria.side-hull-angular` | components / SVG | `assets/Components/Aistiaria/01_side_hull_angular.svg` | `public-components-aistiaria-01_side_hull_angular` | `central-hull` |
| `rt.component.aistiaria.hex-cockpit` | components / SVG | `assets/Components/Aistiaria/02_hex_cockpit.svg` | `public-components-aistiaria-02_hex_cockpit` | — |
| `rt.component.aistiaria.side-wing` | components / SVG | `assets/Components/Aistiaria/03_side_wing.svg` | `public-components-aistiaria-03_side_wing` | — |
| `rt.component.aistiaria.rear-engine-pod` | components / SVG | `assets/Components/Aistiaria/04_rear_engine_pod.svg` | `public-components-aistiaria-04_rear_engine_pod` | — |
| `rt.component.aistiaria.nose-section` | components / SVG | `assets/Components/Aistiaria/05_nose_section.svg` | `public-components-aistiaria-05_nose_section` | — |
| `rt.component.aistiaria.swept-wing` | components / SVG | `assets/Components/Aistiaria/06_swept_wing.svg` | `public-components-aistiaria-06_swept_wing` | — |
| `rt.component.aistiaria.connector-module` | components / SVG | `assets/Components/Aistiaria/07_connector_module.svg` | `public-components-aistiaria-07_connector_module` | — |
| `rt.component.aistiaria.tail-fuselage` | components / SVG | `assets/Components/Aistiaria/08_tail_fuselage.svg` | `public-components-aistiaria-08_tail_fuselage` | — |
| `rt.component.yutiarin.part-1` | components / SVG | `assets/Components/Yutiarin/Part 1.svg` | `public-components-yutiarin-part-1` | — |
| `rt.component.yutiarin.part-10` | components / SVG | `assets/Components/Yutiarin/Part 10.svg` | `public-components-yutiarin-part-10` | — |
| `rt.component.yutiarin.part-2` | components / SVG | `assets/Components/Yutiarin/Part 2.svg` | `public-components-yutiarin-part-2` | — |
| `rt.component.yutiarin.part-3` | components / SVG | `assets/Components/Yutiarin/Part 3.svg` | `public-components-yutiarin-part-3` | — |
| `rt.component.yutiarin.part-4` | components / SVG | `assets/Components/Yutiarin/Part 4.svg` | `public-components-yutiarin-part-4` | — |
| `rt.component.yutiarin.part-5` | components / SVG | `assets/Components/Yutiarin/Part 5.svg` | `public-components-yutiarin-part-5` | — |
| `rt.component.yutiarin.part-6` | components / SVG | `assets/Components/Yutiarin/Part 6.svg` | `public-components-yutiarin-part-6` | — |
| `rt.component.yutiarin.part-7` | components / SVG | `assets/Components/Yutiarin/Part 7.svg` | `public-components-yutiarin-part-7` | — |
| `rt.component.yutiarin.part-8` | components / SVG | `assets/Components/Yutiarin/Part 8.svg` | `public-components-yutiarin-part-8` | — |
| `rt.component.yutiarin.part-9` | components / SVG | `assets/Components/Yutiarin/Part 9.svg` | `public-components-yutiarin-part-9` | — |
| `rt.component.yutiarin.wing-1` | components / SVG | `assets/Components/Yutiarin/Wing 1.svg` | `public-components-yutiarin-wing-1` | — |
| `rt.ship.yutiarin-kinetic-lance-frigate` | ships / SVG | `assets/Ships/Yutiarin Kinetic Lance Frigate.svg` | `public-ships-yutiarin-kinetic-lance-frigate` | — |
| `rt.ship.alien-interceptor` | ships / SVG | `assets/Ships/alien-interceptor.svg` | `public-ships-alien-interceptor` | `alien-interceptor` |
| `rt.ship.alien-leviathan` | ships / SVG | `assets/Ships/alien-leviathan.svg` | `public-ships-alien-leviathan` | — |
| `rt.ship.alien-mothership` | ships / SVG | `assets/Ships/alien-mothership.svg` | `public-ships-alien-mothership` | — |
| `rt.ship.curved-corvette` | ships / SVG | `assets/Ships/curved-corvette.svg` | `public-ships-curved-corvette` | — |
| `rt.ship.faceted-cruiser` | ships / SVG | `assets/Ships/faceted-cruiser.svg` | `public-ships-faceted-cruiser` | — |
| `rt.ship.human-dreadnought` | ships / SVG | `assets/Ships/human-dreadnought.svg` | `public-ships-human-dreadnought` | `human-dreadnought` |
| `rt.ship.human-frigate` | ships / SVG | `assets/Ships/human-frigate.svg` | `public-ships-human-frigate` | — |
| `rt.ship.void-dreadnought` | ships / SVG | `assets/Ships/void-dreadnought.svg` | `public-ships-void-dreadnought` | — |
| `rt.ship.yutiarin-starblade-destroyer` | ships / SVG | `assets/Ships/yutiarin-starblade-destroyer.svg` | `public-ships-yutiarin-starblade-destroyer` | — |
| `rt.turret.yutiarin-defense-turret` | turrets-platforms / SVG | `assets/Turrets-Platforms/Yutiarin-Defense Turret.svg` | `public-turrets-platforms-yutiarin-defense-turret` | — |
| `rt.turret.yutiarin-perimeter-turret` | turrets-platforms / SVG | `assets/Turrets-Platforms/Yutiarin-Perimeter Turret.svg` | `public-turrets-platforms-yutiarin-perimeter-turret` | — |
| `rt.planet-map.map-111` | planets / PNG | `assets/Planets/Map-111.png` | `planetMap111` | — |
| `rt.planet-map.map-122` | planets / PNG | `assets/Planets/Map-122.png` | `planetMap122` | — |
| `rt.planet-map.map-692` | planets / PNG | `assets/Planets/Map-692.png` | `planetMap692` | — |

## 12. Compatibility and preservation

- `publicSvgManifest.ts` is unchanged and remains imported by the runtime loader as a cross-check for all 31 game SVG key/path/alias identities. It is also consumed by existing tests.
- `Planet.ts` and `planets.glsl` are unchanged. The exact planet texture keys remain `planetMap111`, `planetMap122`, and `planetMap692`.
- The project's core `Game` class is unchanged. Only its construction moved from the GameScene constructor to the beginning of GameScene `create()`.
- `MiniMap.ts` is unchanged; its separate icon paths and `hardpoint` arrow texture remain supported.
- The Node-side package, registry, PNG, and output publication validators were not changed for Phase 3C. Existing RTAB files and manifest remain the inputs.
- Package dependencies and `package.json` are unchanged.
- The ignored `private/` directory remains ignored by [.gitignore](../.gitignore). No private file is part of the documentation inventory and no private image contents are recorded here.

## 13. Known limitations and follow-up work

### Confirmed implementation limitations

- The browser loader does not re-run the Node PNG chunk CRC, color profile, decompression, scanline, or aggregate file-size checks. It pins the exact approved PNG bytes, asks the browser to decode them, and validates natural dimensions. The Node validators remain unchanged.
- The runtime hashes establish manifest/bundle/payload consistency but are not a signed manifest. The three planet hashes are independently pinned; SVG `sourceSha256` values are taken from the manifest after registry identity checks.
- The Blob image path does not include Phaser SVGFile's fallback to an encoded data URI for browsers that reject the first object-URL load. In that case, this implementation fails closed and shows the startup error.
- A post-Phaser texture-registration failure leaves the Phaser application and BootScene running beneath the HTML overlay; it prevents GameScene start but does not destroy the application.
- The two unrelated full-suite taxonomy failures remain.

### Separate future gates or recommendations

- Removal of loose public game SVGs and planet PNGs remains a separate approval gate after successful end-to-end runtime verification. No deletion was performed.
- If stronger authenticity than same-origin deployment and pinned planet identity is required, a signed manifest or another independently trusted SVG hash source would be a separate design decision.
- Browser image decoding uses `Blob`, `URL.createObjectURL`, and `HTMLImageElement.decode`; browser-policy/legacy support beyond the verified browser run is not established here.

## 14. Git state at inspection

At the start of this record task:

- Branch: `feature/phase-3b3-stage-a`
- HEAD: `dcbeb0d1a32c6eb72c4f93c9875d69bf6009648a` (`fix Phase 3B.3 PNG packaging validation`)
- Tracked modification: `src/main.ts`
- Untracked additions: `src/assets/rtabRuntimeLoader.ts`, `tests/rtab-parser.test.ts`, `tests/rtab-runtime-loader.test.ts`
- Staged changes: none
- Deleted files: none
- Commit/push state: no Phase 3C commit or push was made

This document now exists at the requested path, but `.gitignore:11` (`docs/*`) ignores it, so it is not listed by `git status --short` or `git ls-files --others --exclude-standard`. The final reported Git inventory remains the one tracked modification plus the three untracked Phase 3C source/test files above, with no staged changes. No stage, commit, push, reset, revert, stash, or clean operation is part of this record task.
