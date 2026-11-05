**# Phase 3A: Versioned Binary Asset Container Design - Revised Specification**

**## Context**

Phase 3A specifies the versioned RTAB asset container and its generated manifest. Phase 3B adds packaging while preserving the existing loose-public-SVG runtime workflow; Phaser runtime loading remains unchanged until Phase 3C. Phase 3A is specification work only: it does not create bundles or modify implementation files.

**---**

**## 1. Container Identity**

**### 1.1 Format Identity**

- **Magic identifier**: `RTAB` (4 bytes, ASCII)
- **File extension**: `.bin` (physical file extension)
- **Format version**: v1 (unencrypted, engine-independent)
- **Rationale**: The `RTAB` identity was previously established in the project's design discussions; `RTIA` is not used.

**### 1.2 Version Compatibility Rules**

| Scenario | Behavior |
|----------|----------|
| **Any version other than v1.0** | Reject with an explicit unsupported-version diagnostic. V1.0 defines no forward-compatible extension mechanism; a later version must define one before readers may accept it. |
| **Nonzero reserved header bit or index flag** | Reject with an explicit error. |
| **Unsupported compression method** | Reject the affected bundle. |

**### 1.3 Format String**

`RTAB v1.0`

**---**

**## 2. Stable Asset Identity**

**### 2.1 Proposed Asset ID Namespace**

Explicitly assigned, persistent identifiers independent of filenames:

| Asset Type | Example Asset ID |
|-----------|-----------------|
| Ships | `rt.ship.human-frigate`, `rt.ship.alien-interceptor`, `rt.ship.human-dreadnought` |
| Components | `rt.component.aistiaria.side-hull-angular`, `rt.component.yutiarin.part-1` |
| Minimap | `rt.minimap.capitalship`, `rt.minimap.corvette`, `rt.minimap.fighter` |
| Root-level | `rt.root.favicon`, `rt.root.icons` |

**### 2.2 Asset Registry Design**

`scripts/asset-registry.json` is the sole source-controlled authority for stable asset IDs, logical paths, explicit texture keys, aliases, content types, and per-asset `bundleId` assignments. It is outside Vite's `public/` directory and is not exposed as a runtime resource. An asset ID is retained from this registry; it is never derived or recomputed from a path.

The registry also declares bundle-level identifiers and their output paths. Bundle membership is derived from each asset's `bundleId`; it is not repeated as asset-ID or texture-key arrays.

```json
{
  "registrySchemaVersion": "1.0",
  "assets": {
    "rt.ship.human-frigate": {
      "logicalPath": "assets/Ships/human-frigate.svg",
      "textureKey": "public-ships-human-frigate",
      "aliases": [],
      "contentType": "SVG",
      "bundleId": "ships1"
    },
    "rt.ship.human-dreadnought": {
      "logicalPath": "assets/Ships/human-dreadnought.svg",
      "textureKey": "public-ships-human-dreadnought",
      "aliases": ["human-dreadnought"],
      "contentType": "SVG",
      "bundleId": "ships1"
    },
    "rt.ship.alien-interceptor": {
      "logicalPath": "assets/Ships/alien-interceptor.svg",
      "textureKey": "public-ships-alien-interceptor",
      "aliases": ["alien-interceptor"],
      "contentType": "SVG",
      "bundleId": "ships1"
    }
  },
  "bundles": {
    "ships1": {
      "bundleFile": "/assets/Ships/ships.bin"
    }
  }
}
```

The top-level `registrySchemaVersion` identifies the schema of `scripts/asset-registry.json`; it is not the RTAB container version. RTAB v1.0 is identified independently by the binary outer header and generated bundle metadata. The registry's asset fields are authored. Bundle `assetCount`, per-source `sourceSha256`, and `bundleSha256` are generated; they are not separately maintained in the registry. `public/assets/asset-manifest.json` is generated output for runtime consumption, and `public/assets/asset-manifest-schema.json` describes that generated manifest instance. The manifest is not a second editable registry. The source registry may include registered assets not yet packaged.

Asset IDs are never reused after removal. A removed ID is retired rather than assigned to another asset.

**### 2.3 ID and Texture-Key Behavior**

| Scenario | Behavior |
|----------|----------|
| **Filename or logical-path changes** | The registered asset ID and texture key remain unchanged. Update only the registry's `logicalPath`; neither identity value is recalculated from the new path. |
| **Texture key migration** | Change a registered texture key only through an explicit migration. Retain the old key as a compatibility alias for as long as existing blueprints or saves may reference it. |
| **Existing legacy aliases** | Preserve the three exact asset-to-alias mappings in section 8.1. These legacy aliases are distinct from compatibility aliases retained during a texture-key migration. |
| **Asset removed or replaced** | Retire the asset ID and do not reassign it. |

**---**

**## 3. Binary Index Redesign**

**### 3.1 Layout Overview**

V1 contains one 64-byte outer header, one embedded index, and one payload section. The index starts at absolute file offset 64, and the payload immediately follows the complete index, including its trailing digest.

```
[Outer header: 64 bytes]
[Index: indexSize bytes]
[Payload: through end of file]
```

**### 3.2 Header Structure (64 bytes, little-endian)**

| Offset | Size | Field | Type and v1 meaning |
|--------|------|-------|---------------------|
| 0 | 4 | Magic identifier | ASCII `RTAB` |
| 4 | 2 | Format version | Little-endian unsigned 16-bit value; high byte is major and low byte is minor (`0x0100` = v1.0, stored as bytes `00 01`) |
| 6 | 2 | Flags | Bit 0 `hasIndex` and bit 1 `hasPayload`; both are 1 in v1; all other bits are reserved and zero |
| 8 | 8 | Payload offset | Absolute file offset of the payload section; equals `indexOffset + indexSize` |
| 16 | 8 | Index offset | Absolute file offset; exactly 64 in v1 |
| 24 | 8 | Index size | Total index bytes, including the 28-byte index header, records, string table, and final 32-byte digest |
| 32 | 32 | Reserved | Zero-filled |

V1 bundles contain at least one asset, so both section flags are set and neither section offset is zero. The payload section extends from `payloadOffset` through end-of-file.

**### 3.3 Index Structure**

All multibyte index and record fields are little-endian. The index is contiguous: its 28-byte header is followed immediately by `entryCount` fixed-size records, then the string table, then the 32-byte index digest.

**#### 3.3.1 Index Header (28 bytes)**

| Index-relative offset | Size | Field | Type and v1 meaning |
|-----------------------|------|-------|---------------------|
| 0 | 4 | Index magic | ASCII `INDX` (distinct from outer `RTAB`) |
| 4 | 4 | Entry count | Unsigned `uint32`; number of records and IDs |
| 8 | 2 | Record size | Unsigned `uint16`; exactly 72 in v1 |
| 10 | 2 | Index flags | Unsigned `uint16`; zero in v1 |
| 12 | 8 | String-table offset | Unsigned `uint64`; byte offset from index start to the first string-table byte |
| 20 | 8 | String-table size | Unsigned `uint64`; number of string-table bytes, excluding the final digest |

The record table starts at index-relative offset 28. Its size is `entryCount × recordSize`. In v1, `stringTableOffset` must equal `28 + entryCount × 72`; there is no padding between sections.

**#### 3.3.2 Index Record Layout (72 bytes per asset)**

Each record references its asset ID in the shared string table. ID offsets are relative to the beginning of the string table, and ID lengths are UTF-8 byte lengths. The string table contains no length prefixes or null terminators.

| Record-relative offset | Size | Field | Type and v1 meaning |
|------------------------|------|-------|---------------------|
| 0 | 4 | Asset ID string offset | Unsigned `uint32`; byte offset from string-table start |
| 4 | 4 | Asset ID byte length | Unsigned `uint32`; UTF-8 byte length, not character count |
| 8 | 8 | Payload offset | Unsigned `uint64`; relative to the payload-section start |
| 16 | 8 | Stored size | Unsigned `uint64`; payload bytes stored for this record |
| 24 | 8 | Original size | Unsigned `uint64`; original source bytes before compression |
| 32 | 2 | Content type | Unsigned `uint16`: 0=SVG, 1=PNG, 2=JSON, 3=Unknown; Phase 3B packaging support is restricted as specified in section 3.4.4 |
| 34 | 2 | Compression method | Unsigned `uint16`: 0=NONE, 1=ZLIB, 2=GZIP; v1 permits only 0 |
| 36 | 4 | Entry flags/reserved | Unsigned `uint32`; zero in v1 |
| 40 | 32 | Payload SHA-256 | Raw 32-byte SHA-256 digest of the stored payload bytes |

Record-size arithmetic is `4 + 4 + 8 + 8 + 8 + 2 + 2 + 4 + 32 = 72 bytes`. IDs are unique and stored once; each record's string offset and byte length identify its exact UTF-8 byte range.

**#### 3.3.3 String Table**

The string table is a concatenation of the UTF-8 bytes of each asset ID in record order. It has no count field, per-string length prefix, delimiter, or null terminator; the record table already provides the ID count, offsets, and byte lengths. The string-table offset and size in the index header delimit the entire table.

The v1 maximum string-table size is `UINT32_MAX` bytes (4,294,967,295 bytes), matching the maximum representable `uint32` offset and byte length. An ID may be at most `UINT32_MAX` bytes, and every offset, length, and end position must be representable and within that table limit. The `uint64` header field does not increase this v1 limit.

For every record, the validator must check with overflow-safe arithmetic that `assetIdStringOffset + assetIdByteLength <= stringTableSize`, decode that range as valid UTF-8, and reject duplicate IDs. V1 records reference the concatenated strings in record order without gaps or overlap: the first offset is zero, each following offset equals the previous range's end, and the final range ends at `stringTableSize`.

**#### 3.3.4 Index Size and Integrity Digest**

`indexSize = 28 + (entryCount × recordSize) + stringTableSize + 32`; v1 requires `recordSize = 72`.

`indexSize` includes the index header, all records, the string table, and the final digest. The digest is stored as the last 32 bytes of the index at absolute file offset `indexOffset + indexSize - 32`. It is the raw SHA-256 digest of the exact half-open byte interval `[indexOffset, indexOffset + indexSize - 32)`: this includes the complete index header, all records, and the entire string table, and excludes only those final 32 digest bytes.

V1 requires `indexOffset = 64` and `payloadOffset = indexOffset + indexSize`. The string-table range must end exactly where the digest begins.

**#### 3.3.5 Record-Size Compatibility**

V1.0 readers require `recordSize = 72`, `indexFlags = 0`, and zero in all reserved outer-header and entry-flag bits; any other value is rejected. V1.0 has no extension mechanism and readers do not skip unknown fields or accept another record size. A later format version must explicitly define an extension mechanism before such data can be accepted.

**### 3.4 Payload Section**

**#### 3.4.1 Layout**

The payload section starts at the absolute offset in the outer header and extends to end-of-file. Its exact size is `fileSize - header.payloadOffset`; the outer header has no separate payload-size field. Payload records are ordered by asset ID and occupy this entire section contiguously in that order.

**#### 3.4.2 Payload Offsets and Bounds**

Every record's `payloadOffset` is relative to the payload-section start. The absolute file position of an asset is `header.payloadOffset + record.payloadOffset`. No record stores an absolute payload offset.

For each record, the validator must check with overflow-safe arithmetic that its payload range `[payloadOffset + record.payloadOffset, payloadOffset + record.payloadOffset + storedSize)` lies wholly within the bundle payload section. V1 ranges must be contiguous: the first record offset is zero, every next record offset equals the previous record's end, and the final record ends exactly at `fileSize - header.payloadOffset`. Thus ranges neither overlap nor leave gaps or trailing unindexed bytes. Unsupported compression methods are rejected; for v1 `NONE` requires `storedSize = originalSize`.

**#### 3.4.3 Payload Content (v1)**

- **Raw source bytes** are stored without compression.
- **Compression method** is `NONE` (value 0).
- **Stored size** equals **original size**.
- The payload digest is SHA-256 of exactly these stored bytes.

**#### 3.4.4 Payload Validation**

Phase 3B packages SVG assets only. Although the content-type field identifies SVG (0), PNG (1), and JSON (2), the Phase 3B packager must reject PNG, JSON, `Unknown` (3), and any unassigned content type; this phase does not define PNG or JSON payload validation.

Every SVG source must be a non-zero-byte, well-formed XML document whose document element is an SVG root. Malformed XML, a non-SVG root, or a zero-byte file rejects the package before any output is replaced. A valid non-zero-byte SVG such as a minimal `<svg/>` document is allowed even if it draws nothing.

V1 rejects any source asset larger than `4,294,967,295` bytes (`UINT32_MAX`); the validator must also reject sizes, offsets, or computed ranges that overflow their declared integer fields. Duplicate asset IDs are rejected.

**#### 3.4.5 Deterministic Payload Ordering**

Assets are sorted by asset ID (alphabetical/namespaced order) before being written to the payload. This ensures reproducible output.

**---**

**## 4. Registry and Generated Manifest**

**### 4.1 Authoritative Source**

`scripts/asset-registry.json` is the sole editable source for asset IDs, logical paths, texture keys, aliases, content types, bundle assignments, and bundle output paths. It is source-controlled and outside Vite's `public/` directory. Registry asset records contain `bundleId`; they do not contain bundle membership arrays.

**### 4.2 Generated Runtime Manifest**

`public/assets/asset-manifest.json` is generated output, not an authoring source. It contains currently packaged asset records and bundle records. `public/assets/asset-manifest-schema.json` describes the generated manifest. A manifest asset's identity and compatibility fields are copied from the registry; `sourceSha256`, bundle counts, and bundle hashes are generated from packaging inputs and outputs.

**### 4.3 Manifest Membership and Bundle Records**

Each manifest asset record has one `bundleId`. Bundle membership is derived by filtering asset records by that value. Bundle records contain only bundle-level fields and never duplicate asset IDs or texture keys. Every bundle declared in the v1.0 generated manifest is required; Phase 3C must not start gameplay unless every declared bundle loads and validates successfully.

The generated manifest root includes a `manifestSchemaVersion` field:

```json
{
  "manifestSchemaVersion": "1.0"
}
```

This excerpt shows the version field only; the manifest root also contains the `assets` and `bundles` collections described below.

**Generated asset record**:

```json
{
  "assetId": "rt.ship.human-frigate",
  "logicalPath": "assets/Ships/human-frigate.svg",
  "textureKey": "public-ships-human-frigate",
  "aliases": [],
  "contentType": "SVG",
  "sourceSha256": "abc123...",
  "bundleId": "ships1"
}
```

**Generated bundle record**:

```json
{
  "bundleId": "ships1",
  "bundleFile": "/assets/Ships/ships.bin",
  "formatVersion": "1.0",
  "assetCount": 10,
  "bundleSha256": "def456..."
}
```

`bundleFile` is always the root-relative URL served by Vite, including its leading slash (for example, `/assets/Ships/ships.bin`). The generated manifest has a top-level `manifestSchemaVersion` data field (for example, `"1.0"`) that identifies the generated manifest structure. The registry's `registrySchemaVersion` identifies the authoring registry structure, while each bundle record's `formatVersion` identifies the RTAB binary format (`"1.0"` for v1.0); these are three distinct version fields and may evolve independently. The JSON Schema document's standard `$schema` keyword identifies the JSON Schema dialect and is not a custom manifest-version field.

**---**

**## 5. Fingerprinting and Integrity**

**### 5.1 Three Distinct Concepts**

| Fingerprint | Computed From | Stored In | Purpose |
|------------|--------------|-----------|---------|
| **sourceSha256** | SHA-256 of the original source file bytes | Generated asset record in `asset-manifest.json` | Computed from canonical sources during packaging; a clean clone does not need the source file to validate committed bundles. |
| **payloadSha256** | SHA-256 of the exact stored payload bytes | Embedded in each index record at record-relative offset 40 (32 bytes) | Per-asset integrity check. In v1, raw payload means it normally equals `sourceSha256`, but the concepts and uses remain distinct. |
| **bundleSha256** | SHA-256 of the complete final `.bin` file (header + index + payload) | Generated bundle record in `asset-manifest.json` | Detects bundle changes and supports cache invalidation; checked against the manifest in bundle validation/runtime. |

**### 5.2 Rationale for Including payloadSha256 in Index**

- Compute `sourceSha256` from each original source file and `payloadSha256` from the exact bytes that will be stored; for v1 uncompressed assets these values normally match.
- Write each `payloadSha256` into its index record, then compute the index digest over the complete index except its final 32 digest bytes.
- Assemble the final RTAB file and compute `bundleSha256` over the complete file. This hash includes the embedded payload hashes and index digest.
- Store `bundleSha256` only in the external generated manifest. It does not feed back into the bundle bytes, so this sequence has no circular dependency.

**### 5.3 Fingerprint Check Phases**

| Phase | Checked Fingerprints | Purpose |
|-------|---------------------|---------|
| **Packaging from sources** | Compute `sourceSha256` and `payloadSha256`; verify v1 source/payload equality; compute index digest and final `bundleSha256` | Validate canonical source inputs and generate output |
| **Bundle validation / Phase 3C runtime** | Verify manifest `bundleSha256`, index digest, and each extracted payload against `payloadSha256` | Validate committed bundle bytes without requiring source files |
| **Clean-clone validation** | Same bundle, index, and payload checks; no `sourceSha256` recomputation | Validate committed manifest and bundle without private sources |

**### 5.4 Texture Key Stability**

- **Registered texture keys are stability data**: Once a texture key has been registered (e.g., `public-ships-human-frigate`), it is treated as stable compatibility data. A logical path change should not automatically rename the texture key.
- **Texture key preservation**: The existing `publicSvgManifest.ts` texture keys are preserved across all phases. They remain valid and are stored in the asset registry and manifest.
- **Explicit migration only**: A texture key is changed only through an explicit migration, while retaining the old key as an alias for as long as existing game data or blueprints need it.
- **Alias retention**: If a texture key is explicitly migrated, retain the old texture key as a compatibility alias for as long as existing blueprints or saves may reference it. This is distinct from the three fixed legacy aliases in section 8.1.

**---**

**## 6. Strict Packaging Behaviour**

**### 6.1 Mandatory Rejections**

The default packager **must** reject (with explicit error and exit code 1) if any of these conditions are detected:

| Condition | Error Message Example |
|-----------|----------------------|
| **Malformed XML/SVG or non-SVG root** | `"Fatal: asset \<path> is not a well-formed SVG document: \<details>"` |
| **Missing required canonical sources** | `"Fatal: could not find canonical source for asset \<id>: \<path>"` |
| **Zero-byte files** | `"Fatal: asset \<path> is a zero-byte file"` |
| **Oversized source asset** (larger than `UINT32_MAX` bytes) | `"Fatal: asset \<path> exceeds the v1 maximum size of 4294967295 bytes"` |
| **Duplicate asset IDs** | `"Fatal: duplicate asset ID "\<id>" detected. Asset IDs must be unique."` |
| **Duplicate texture keys** | `"Fatal: duplicate texture key "\<key>" detected. Texture keys must be unique."` |
| **Conflicting aliases** | `"Fatal: conflicting legacy alias "\<alias>" maps to both "\<key1>" and "\<key2>"` |
| **Unsupported content type** (including PNG, JSON, `3=Unknown`, and unassigned values for the SVG-only Phase 3B packager) | `"Fatal: unsupported content type \<type> for asset \<id>"` |
| **Incomplete bundle** (no assets packaged) | `"Fatal: no assets were packaged. Cannot generate incomplete bundle."` |

**### 6.2 Never Silent Failures**

- The packager **must never** silently generate an incomplete bundle.
- The packager **must never** overwrite a previously valid bundle with incomplete output.
- If any rejection condition is met, the packager exits with code 1 and does not modify any output files.

**### 6.3 Source-Available Packaging and Source-Unavailable Validation**

When all canonical source files required by the registry are available, packaging validates those sources, writes complete bundles, and generates the manifest. Missing canonical sources are fatal; old loose public copies are never substituted for missing canonical sources.

Only when the entire ignored `private/` source root is absent may the clean-clone workflow select explicit, read-only bundle-validation mode. It validates the committed manifest and bundles, including `bundleSha256`, the index digest, record bounds, and per-payload `payloadSha256`; it does not write or replace outputs and does not scan loose public copies for replacement content. If `private/` exists but any required canonical source is missing, treat it as an incomplete source tree and fail instead of entering validation-only mode.

**### 6.4 Never Silent or Partial Packaging**

- The default packaging operation never generates an incomplete bundle.
- Validation-only mode is explicit and read-only; it does not repackage, change the manifest, or overwrite bundles.
- Any packaging rejection exits with code 1 and leaves previously valid outputs unchanged.

**---**

**## 7. Phaser Runtime Compatibility**

**### 7.1 `this.load.svg()` Usage**

- **`this.load.svg()` must receive actual SVG data or a valid SVG URL, not the `.bin` container URL.**
- The `.bin` format is a packaging format; the runtime loads individual SVG assets, not the container directly.

**### 7.2 Future Runtime Design (Not Implemented in Phase 3A)**

The future runtime should:

1. **Fetch the bundle asynchronously** at app start (before Scene `preload()`).
2. **Validate its header, index and fingerprints** (`bundleSha256`).
3. **Extract individual SVG payloads** using the embedded index.
4. **Prepare data URLs or Blob URLs**, or use a compatible custom Phaser loader.
5. **Register the resulting textures** using the existing texture keys and aliases.

**Preparation must happen before Phaser's synchronous Scene `preload()` lifecycle requires the textures.**

**### 7.3 Bootstrap Failure Reporting**

- **Do not use process exit codes as browser runtime error handling.**
- All bundles declared in the v1.0 generated manifest are required; no separate `required` property is used. In Phase 3C, failure to load or validate any declared bundle is fatal to bootstrap: report the error to the console/UI and do not start gameplay. Do not silently fall back to loose assets or expose a partial texture collection. Phase 3B does not fetch or load bundles in Phaser.
  - `"Bootstrap error: required RTAB bundle not found or invalid at \<path>; gameplay cannot start."`
  - `"Bootstrap error: RTAB bundle fingerprint mismatch. See console for details."`
  - Present these as console messages or UI alerts, not process termination.

**### 7.4 Timeline**

- **Phase 3A**: Specification and design only.
- **Phase 3B**: Packaging, bundle generation/validation, and generated manifest; no Phaser runtime changes. Existing loose-SVG loading stays in place.
- **Phase 3C**: Phaser runtime integration (async bootstrap, bundle reading, and texture registration).

**---**

**## 8. Legacy Alias Documentation**

**### 8.1 Exact Asset-to-Alias Associations**

The `LEGACY_ALIASES` map must preserve its exact associations:

| Logical Source Path | Alias |
|---------------------|-------|
| `Ships/human-dreadnought.svg` | `human-dreadnought` |
| `Ships/alien-interceptor.svg` | `alien-interceptor` |
| `Components/Aistiaria/01_side_hull_angular.svg` | `central-hull` |

**### 8.2 Do Not Describe As**

- Do not describe these aliases as mappings between unrelated assets.
- Do not change the asset-to-alias mappings.
- Do not add new aliases without explicit user direction.

**### 8.3 Manifest Preservation**

The manifest must preserve these exact alias associations in each asset's `aliases` array.

**---**

**## 9. Version Compatibility Rules (Expanded)**

**### 9.1 Unsupported Major Version**

- **Behavior**: Reject with explicit diagnostic.
- **Example**: Runtime supports v1.0, file has header version v2.0 → error: `"Unsupported RTAB version 2.0 (0x0200). Supported version: 1.0 (0x0100)."`.

**### 9.2 Newer Minor Version**

- **Behavior**: Reject every version other than v1.0, including v1.1, with an explicit unsupported-version diagnostic such as `"Unsupported RTAB version 1.1 (0x0101). Supported version: 1.0 (0x0100)."`.
- **Compatibility condition**: V1.0 defines no unknown-field skipping or forward-compatibility mechanism. A later format version must define and version its extension rules before readers accept it.

**### 9.3 Header and Index Flags**

- Header bits 0 (`hasIndex`) and 1 (`hasPayload`) are the only recognized bits; both must be set in v1.0, and every other header bit must be zero.
- The index flags field must be zero in v1.0.
- **Behavior**: Reject any violation with an explicit diagnostic, such as `"Unsupported RTAB header flags"` or `"Unsupported RTAB index flags"`.

**### 9.4 Unsupported Compression Methods**

- **Behavior**: Reject the affected bundle.
- **Example**: Compression method field = 2 (GZIP), but runtime only supports 0 (NONE) → error: `"Unsupported compression method: 2. Supported methods: 0=NONE"`.

**### 9.5 Do Not Silently Turn Incompatible Bundles Into Empty Asset Collections**

- Every bundle declared in the v1.0 generated manifest is required. If any declared bundle is missing, corrupt, incompatible, or fails any fingerprint/bounds check, Phase 3C bootstrap must fail explicitly; gameplay must not start with missing or partial textures.
- Do not fall back to loose SVGs for a failed declared bundle. Phase 3B is unaffected: it does not load bundles in Phaser and continues using its existing loose-SVG runtime path.

**---**

**## 10. Phase Boundaries**

**### 10.1 Phase 3A — Design Approval**

**Activities**:
- Final format specification (this document).
- Exact header and index layouts (sections 3.1–3.4).
- Manifest instance and JSON Schema.
- Asset registry design (section 4).
- Fingerprint definitions (section 5).
- Validation and compatibility rules (sections 5–9).
- Test plan (section 11).

**Deliverable**: Revised Phase 3A technical specification document.

**Planned later-phase artifacts (not Phase 3A outputs)**:
- Phase 3B authors `scripts/asset-registry.json` and `public/assets/asset-manifest-schema.json`, then generates and commits `public/assets/asset-manifest.json` and `public/assets/Ships/ships.bin`.

**Do not implement**: Packager, bundle generation, runtime integration.

**### 10.2 Phase 3B — Packaging Implementation**

**Activities** (only after Phase 3A receives explicit approval):

- Implement the agreed packager (`scripts/package-assets.mjs`).
- Generate the initial Ships bundle containing all 10 required ship assets.
- Generate the committed asset manifest.
- Verify determinism and fingerprints.
- Test corruption and missing-source failures.
- Change the `predev` and `prebuild` hooks to use the source-available/source-unavailable behavior below.
- Keep runtime loading untouched and retain the loose public SVGs used by the current runtime. The generated `.bin` bundles are validation targets only; Phaser does not load them in this phase.

**Outputs**:
- `public/assets/Ships/ships.bin` — generated binary bundle.
- `public/assets/asset-manifest.json` — generated manifest.
- `tests/package-assets.test.ts` — test suite.
- Source-aware `predev`/`prebuild` hooks, with no Phaser runtime changes.

**Do not proceed**: Until Phase 3A specification receives explicit approval.

**### 10.3 Phase 3C — Runtime Integration**

**Activities** (only after Phase 3B completes successfully):

- Implement asynchronous bootstrap: fetch `.bin` bundles at app start before Scene `preload()`.
- Validate bundle headers, index and fingerprints (`bundleSha256`).
- Extract individual SVG payloads using the embedded index.
- Prepare data URLs or Blob URLs, or use a compatible custom Phaser loader.
- Register the resulting textures using the existing texture keys and aliases.
- Only after bundle-backed runtime loading is verified may Phase 3C remove the migrated loose public SVGs.

**Outputs**:
- Updated `src/main.ts` with bundle loading integration.
- Verified end-to-end runtime operation with `.bin` bundles.

**Phase sequence**: 3A → 3B → 3C. No overlapping or skipped phases.

**### 10.4 Clean-Clone and Lifecycle-Hook Behavior**

The existing `predev` and `prebuild` hooks run `scripts/generate-public-svg-manifest.mjs`, which scans canonical source roots and writes `src/generated/publicSvgManifest.ts`. That generated TypeScript file is tracked by Git. The repository's `private/` source tree is ignored by Git and absent from a normal clean clone; its absence is intentional and is not an incomplete checkout.

- **Complete source tree available**: If the private source root exists, verify that every registry-required canonical private source is present before generation. When the complete canonical inputs are available, validate them, package assets, generate the asset manifest, and regenerate `src/generated/publicSvgManifest.ts` from canonical source discovery using the registry's explicit stable texture keys. Retained public copies continue to serve the existing Phase 3B runtime; they are not substituted for missing canonical private packaging inputs.
- **Private source tree intentionally absent**: Treat a wholly absent ignored `private/` root as the clean-clone case. The hooks run explicit read-only bundle-validation mode, do not package or regenerate the asset manifest, and leave the tracked `src/generated/publicSvgManifest.ts` unchanged. The existing Phase 3B runtime continues to use the retained loose public SVGs.
- **Private source tree present but incomplete**: Treat missing required files under an existing `private/` root as an incomplete source tree, not as the intentional clean-clone case. Fail `predev`/`prebuild` explicitly before writing any generated manifest or bundle; do not silently switch to validation-only mode or substitute public copies.
- **No stale-copy scanning for packaging**: Bundle packaging reads the registry's canonical source paths only. It never uses loose public copies as replacements for absent private canonical sources.

| Phase | Clean-clone capability |
|-------|------------------------|
| 3A | No implementation or runtime change; the existing game continues to use loose public SVGs. |
| 3B | With the entire ignored private source root absent, hooks validate committed manifest/bundles and leave the tracked generated TypeScript untouched. A present but incomplete private tree is an error. The normal game runtime still uses retained loose public SVGs; no bundle-backed game runtime is claimed. |
| 3C | After runtime integration is verified, a clone without `private/` can run from committed bundles and manifest, and migrated loose public copies may be removed. |

**---**

**## 11. Test Strategy**

**### 11.1 Unit Tests (to be added in Phase 3B)**

- Test `.bin` header parsing with valid and invalid files.
- Test asset index extraction and individual asset retrieval.
- Test fingerprint computation and verification.
- Test packaging pipeline with known input/output.

**### 11.2 Determinism Tests**

- Run packaging pipeline twice on identical input.
- Verify byte-identical `.bin` output both times.
- Verify `asset-manifest.json` output is identical.

**### 11.3 Integration Tests**

- In Phase 3B, test that a clean clone can validate committed `.bin` bundles and the manifest without canonical private sources; do not claim that Phaser loads those bundles.
- Verify the existing Phase 3B game continues to resolve loose public SVGs, current texture keys, and legacy aliases.
- In Phase 3C, test bundle-backed Phaser loading and error handling for missing/corrupt bundles.

**### 11.4 Verification Against Existing Code**

- For every currently packaged asset, verify the generated `publicSvgManifest.ts` retains the texture key recorded in the registry and generated asset manifest; unbundled assets need not appear in the bundle manifest.
- Verify the packaging pipeline covers canonical source paths returned by `listSourceAssets()` and computes `sourceSha256` from the exact source bytes.
- Verify 36-SVG inventory coverage matches existing expectations.
- Verify exact legacy aliases: `Ships/human-dreadnought.svg` → `human-dreadnought`; `Ships/alien-interceptor.svg` → `alien-interceptor`; `Components/Aistiaria/01_side_hull_angular.svg` → `central-hull`.

**### 11.5 Final Verification Checklist (Phase 3B)**

- [ ] `scripts/package-assets.mjs` exists and runs without errors
- [ ] `public/assets/Ships/ships.bin` generated successfully with all 10 ship assets
- [ ] `public/assets/asset-manifest.json` generated with correct format
- [ ] Bundle contains all 10 ship SVG assets with correct fingerprints
- [ ] `asset-manifest.json` includes `sourceSha256`, and `bundleSha256` for each bundle
- [ ] Running the packager twice produces byte-identical `ships.bin` and `asset-manifest.json`
- [ ] A clean clone without canonical sources validates the committed manifest and bundle without rewriting them.
- [ ] In an isolated clean-clone fixture, omit `private/` and remove migrated loose public SVGs; run `npm run predev` and `npm run prebuild` and verify both hooks validate committed outputs without scanning stale public copies or changing `src/generated/publicSvgManifest.ts`. This fixture checks hook behavior only; it does not claim the game runtime works without loose SVGs before Phase 3C.
- [ ] At the start of Phase 3B, run and record the full existing test suite. The latest reported reference is 129/131 passing with two unrelated taxonomy failures; this document has not independently verified that report. Phase 3B must introduce no new failures, and its actual test results must be reported.
- [ ] `tsc --noEmit` still passes
- [ ] Legacy texture keys and all three exact aliases remain functional through the retained loose-SVG runtime.
- [ ] No private source paths or artwork bytes are exposed; the manifest contains only declared asset/bundle metadata and hashes.
- [ ] Asset IDs are persistent (`rt.ship.human-frigate`, etc.) not filename-derived
- [ ] Registry permits logical path changes without ID changes

**---**

**## 12. Confirmed Repository Facts (from exploration)**

These are **established facts** from the codebase exploration, not design decisions:

- **36 SVG assets total**: 31 in `assets/` + 3 minimap + 2 root-level (`favicon.svg`, `icons.svg`)
- **Source-root mapping**: `ASSET_ROOTS` in `asset-roots.mjs` with longest-prefix match semantics
- **Current texture keys**: Follow `public-{category}-{slug}` pattern (e.g., `public-ships-human-frigate`)
- **3 legacy aliases**: `Ships/human-dreadnought.svg` → `human-dreadnought`, `Ships/alien-interceptor.svg` → `alien-interceptor`, `Components/Aistiaria/01_side_hull_angular.svg` → `central-hull`
- **Private assets**: 10 ships copied from `public/assets/Ships/` to `private/assets/Ships/`, SHA-256 hashes verified matching
- **`.gitignore`**: Already includes `/private/`
- **`listSourceAssets()`**: Returns 36 entries comprehensively covering all SVG categories
- **`analyzeSvgMetadata()`**: Validates SVG metadata and extracts titles
- **Test baseline report**: The consistency review reports 129/131 passing with two unrelated taxonomy failures. This report has not been independently verified here; Phase 3B must record its own exact results and introduce no new failures.
- **`LEGACY_ALIASES` map**: Exactly 3 entries as documented in `generate-public-svg-manifest.mjs`

**---**

**## 13. Approved Design Decisions**

- The container uses the 64-byte little-endian `RTAB` outer header and an embedded `INDX` index with the exact v1 fields, 72-byte records, shared UTF-8 string table, and trailing digest defined in section 3.
- The version field's high byte is the major version and low byte is the minor version; v1.0 is `0x0100`. V1.0 rejects content type `Unknown` and all unassigned content-type values.
- Phase 3B packages SVG assets only; its packager rejects PNG, JSON, `Unknown`, and unassigned content types. V1.0 requires header bits 0 and 1 set, all other header bits zero, and index flags zero.
- The outer `payloadOffset` is absolute; record payload offsets are relative to the payload section.
- V1 stores raw, uncompressed asset bytes and rejects zero-byte, malformed, non-SVG-root, oversized, or incomplete source inputs.
- `scripts/asset-registry.json` is the sole authoring authority. The public manifest and schema are generated output; asset membership is represented by per-asset `bundleId` only.
- `sourceSha256`, embedded `payloadSha256`, the index digest, and external `bundleSha256` have the distinct inputs, storage locations, and verification stages defined in section 5.
- The registry schema, generated manifest schema, and RTAB format use distinct `registrySchemaVersion`, `manifestSchemaVersion`, and bundle-record `formatVersion` fields; the JSON Schema `$schema` keyword identifies only the schema dialect. All v1.0 manifest-declared bundles are required at runtime.
- Clean clones without canonical source files use explicit read-only bundle-validation behavior; they do not repackage from loose public copies.
- Phase 3B packages and validates bundles while retaining loose-SVG runtime loading. Phase 3C integrates bundle loading into Phaser and may remove migrated loose copies only after runtime verification.
- No partial/recovery packaging mode is part of v1.

No unresolved design decision is required to complete Phase 3A. Implementation details within the approved design are left to their designated later phase.

**---**

**## 14. Approval Required**

**Phase 3A is complete when the user explicitly approves the revised technical specification above.** No files may be created, modified, or implemented until explicit approval is given.

**After approval**, Phase 3B implements the packaging pipeline, generates the initial Ships bundle, and verifies all acceptance criteria.

**---**
*This specification is planning-only. The separately preserved original Claude plan at C:\Users\Arcyriea\.claude\plans\lexical-petting-orbit.md is not modified by this work.*