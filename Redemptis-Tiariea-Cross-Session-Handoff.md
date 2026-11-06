# Redemptis-Tiariea — Cross-Session Project Handoff

**Export date:** 2026-11-06  
**Purpose:** Transfer the project context, architectural decisions, current priorities, and open questions from the previous ChatGPT session into another ChatGPT session.

---

## 1. Project identity and collaboration expectations

**Project:** Redemptis-Tiariea  
**Repository:** https://github.com/Arcyriea/Redemptis-Tiariea  
**Local checkout referenced in prior work:** `C:\GitHub\Redemptis-Tiariea`  
**Primary development environment:** Windows PowerShell, TypeScript, Phaser, Node test runner / `tsx`.

The game must develop its own identity. The broad direction discussed is:
- **Starsector:** a useful reference for the space-combat / fleet scale and general gameplay direction.
- **Battleship Forever:** a reference for modular ship construction ideas.
- **ShellCore Command:** explicitly not the design to reproduce.

These are inspirations, not templates to copy. The goal is an original game with its own systems and architecture.

When helping with repository work:
- Inspect the current local checkout before assuming a previous state is still current.
- Preserve existing user changes. Do not stage, commit, push, or modify unrelated files unless explicitly asked.
- Separate proposals and design documentation from implementation authorization.
- Prefer small, ordered implementation phases with regression tests.
- Preserve existing gameplay behavior unless a change is explicitly intended.
- Be precise about what was actually inspected, tested, changed, or not verified.
- Do not assume that an audit report or an earlier ChatGPT summary describes the latest checkout.

---

## 2. Overall architectural direction

The project is migrating from legacy data and behavior toward typed, canonical definitions while retaining compatibility where needed.

Important ownership boundaries established in earlier work:

- `Game` owns canonical registries for hulls, components, and loadouts.
- `ShipManager` remains responsible for the ship blueprint catalogue and ship creation/disposal lifecycle; it should not become a competing canonical-definition registry.
- `ShipBlueprintDefinition` selects the hull and loadout and carries ship-specific identity/stat information.
- `ShipLoadout` describes installations only. It should not independently select or own a hull.
- `Ship` is the runtime owner of its assembled ship state and installed components.
- `FactionManager` remains responsible for faction identity and relationships.
- Avoid adding new managers when an existing owner can reasonably handle the responsibility.

The intention is to build canonical typed contracts and runtime behavior first, then build the `/shipmaker` editor on top of those stable contracts. The editor should not become a place where core game rules are improvised or duplicated.

---

## 3. Work completed in Phases 2A–2C (as previously reported)

These summaries describe work reported in the prior session. Verify against the current checkout before relying on them as present-day facts.

### Phase 2A — Canonical Shipmaker taxonomy and installation foundation

Reported work:
- Introduced typed hull/component/loadout records.
- Added canonical mount initialization and component compatibility checks.
- Added explicit built-in component installation and reservation behavior.
- Retained legacy structured and tuple compatibility paths.
- Kept canonical registries on `Game`; did not introduce a competing manager.

Important limitation: Phase 2A did **not** complete the full Shipmaker handoff requirements. In particular, the default player/cannon migration, full validation, shared weapon transforms, deterministic visual composition, and Ship-owned rendering lifecycle remained incomplete in the later audit.

### Phase 2B — Hull modifications

Reported work:
- Added hull-modification/stat handling on the existing `Ship` and hull ownership model.

This phase did not complete weapon transforms, dimension validation/projection, or visual composition.

### Phase 2C — Active Module runtime

Reported work:
- Added `src/abilities/ActiveModuleRuntime.ts`.
- Added canonical Active Module operation, targeting, and payload types in `src/types/Shipmaker.ts`.
- Canonical Active Module `Component` instances own an `ActiveModuleRuntime`.
- The runtime owns activation readiness, charging, cooldown, target resolution, cancellation, and energy refund state.
- Resource restoration is clamped to target maxima.
- Target relationships are filtered through `FactionManager`.
- Area targeting discovers active ships from both `Game.shipQueue` and `EntityManager`, allowing registered player ships to be found even though the player is not in `shipQueue`.
- `Game.updateQueues(delta)` propagates Phaser's millisecond delta to queued ships.
- `Ship.updateComponents(delta)` updates all installed components.
- The player component update path calls `updateComponents(delta)` directly, while queued NPC ships update through the managed queue path. The intended invariant is exactly-once component updates.
- `Ship.resetIt()` and `Ship.init()` reset components before references/state are discarded, preventing stale charging/refund state on pooled ship reuse.
- Legacy Skill and Spawn remain on their separate `Ability` path in the current reported implementation.

Reported focused Phase 2C test result: **16 passed**.

A previous full-suite run reported **120 passed, 2 failed**. The failures were described as legacy Skill/Passive/Spawn compatibility and PlayerCannon behavior in `tests/phase2a-taxonomy.test.ts`. A later audit at a different recorded checkout also reported two failures in that test file. Do not assume the failures remain identical or are still present; rerun the tests on the current checkout.

### Reported Phase 2C files / areas

- `src/abilities/ActiveModuleRuntime.ts`
- `src/entities/Component.ts`
- `src/entities/Ship.ts`
- `src/core/Game.ts`
- `src/main.ts`
- `src/types/Shipmaker.ts`
- `tests/phase2c-active-module.test.ts`
- `tests/phase2a-taxonomy.test.ts`
- `ARCHITECTURE.md`

---

## 4. Shipmaker Milestone 1 audit

A read-only completion audit was exported from Copilot and shared in the previous conversation. It compared the local checkout against `docs/SHIPMAKER-ARCHITECTURE-HANDOFF.md`.

### Audit provenance

The audit recorded:
- Repository: `C:\GitHub\Redemptis-Tiariea`
- Branch: `main`
- HEAD: `405ec63a7cb0d322b559f029a03d4734dddd8937`
- Working tree: clean before and after audit checks.
- No source, tests, documentation, configuration, or generated assets were modified during the audit.
- `npx tsc --noEmit`: passed.
- `npm test`: 120 passed, 2 failed of 122.
- `npm run build`: not run because its `prebuild` manifest-generation step could modify generated assets.
- The audit report itself was ignored by the existing `docs/*` rule; no ignore rule was changed.

**Important:** This is a historical snapshot, not proof that the current checkout is still at that HEAD or clean. Recheck `git status`, branch, and HEAD first.

### Audit verdict

**Shipmaker Milestone 1 was not complete.**

The audit's main remaining work, grouped by status:

#### Complete at audit time
- `Game` owns canonical hull, component, and loadout registries; `ShipManager` retains catalogue and lifecycle responsibilities.
- Explicit built-ins are resolved and installed at their declared points, including empty loadouts; they are fixed and cannot be replaced.
- Player remains outside `shipQueue`; managed NPC and player update paths are separate.

#### Partial at audit time
- Blueprint is the sole selector of hull/loadout, but the required shared-hull/two-blueprint regression test was missing.
- Stable unique ID namespaces, definition-local sprite IDs, reserved `legacy:` prefix, and shared hull point-ID namespace were not fully validated.
- Blueprint hull/loadout/component/point references were not comprehensively validated.
- Hardpoint configuration, vent propulsion category, and utility category matching existed, but the full spinal/turret/dual-compatible, vent, and utility test matrix was incomplete.
- Turret interval/speed behavior was incomplete: geometry fields and per-instance angle existed, but validation, initialization, update, and clamping were missing.
- Authored angle/order/scale/origin validation and `rotationFrame` restrictions were incomplete.
- Propulsion output/max validation, vent envelope checks, and exhaust orientation transform were incomplete.
- Manifest-backed public sprite-key validation was missing.
- Legacy adapter/projection compatibility and stale-projection/rebuild behavior needed more coverage.
- PlayerCannon behavior had legacy tests, but canonical migration and real muzzle-transform coverage were incomplete.
- PlayerCannon firing behavior had partial regression coverage but lacked the required cardinal transform integration checks.
- The overall Milestone 1 regression matrix remained partial.

#### Missing at audit time
- Optional-axis dimensions contracts and validation/projection implementation.
- The rule that dimension comparisons must not be misrepresented as off-center containment or physical-clearance proofs.
- Canonical migration of the default player hull/loadout/cannon to typed definitions.
- One shared attachment/muzzle transform used by both fallback and managed firing paths.
- Pure deterministic hull/component visual composition and stable layer ordering.
- Ship-owned Phaser display objects, cleanup, attachment context, turret rotating groups, and composition-refreshing add/remove operations.

### Audit's dependency order

The audit recommended the following order:

1. Finish typed contracts and runtime validation: IDs/namespaces/references, dimensions, authored angles/sprite metadata, propulsion, and manifest keys.
2. Migrate the default player hull/loadout/cannon to canonical typed definitions while preserving stats, visuals, queue behavior, and cannon values.
3. Implement shared weapon/muzzle transforms for both managed and fallback firing paths.
4. Complete turret instance behavior: interval semantics, initial angle, per-instance updates, and clamping.
5. Add deterministic composition and Ship-owned rendering lifecycle.
6. Close remaining regression gaps and reverify with the full test suite and authorized build.

The audit explicitly said not to start editor/serializer milestones as part of these prerequisites.

### User's immediate priority from the audit discussion

The user chose to address the **three Missing prerequisites first**, then finish the **two remaining Partial items** they had identified in that discussion. The audit itself contains several partial rows, so clarify which exact two the user means if that distinction becomes relevant. Do not silently reinterpret this priority or start implementation without agreement.

---

## 5. Legacy combat systems: repurpose, do not simply delete

The user raised a broader unit-architecture direction:

- The old **Skill / Spawn / Weapon / Passive** system should be repurposed rather than discarded.
- **Infantry** should reuse the legacy Skill / Spawn / Weapon / Passive model as its starting framework.
- **Heroes** should use an expanded framework built from or compatible with that legacy foundation.
- **Tanks** should reuse the newer canonical system being developed for Ships, rather than being forced into the infantry legacy model.

This is a design direction, not a completed migration. The exact shared abstractions, type contracts, and runtime ownership boundaries remain to be designed.

### Squad behavior

The user considered whether tanks should be individual entities or squad-based:
- Small tanks may be grouped into squads, especially at smaller gameplay scales.
- Small tank squads may line up while travelling through urban areas.
- Larger and titan-class tanks should behave as individual units because their scale can approach that of small space warships.
- The user then leaned toward making the **squad mechanic globally splittable and rejoinable**.

This should be treated as a general squad capability, not a tank-only hardcoded feature. A unit may be part of a squad, split away, or rejoin under defined gameplay rules. Exact command semantics, formation behavior, squad membership ownership, and AI implications are still open.

---

## 6. Component, ManagedWeapon, and ActiveModuleRuntime relationship

The user asked whether `Component` is the common class through which both `ManagedWeapon` and `ActiveModuleRuntime` are used.

The reported design distinction:
- `Component` is the installed equipment/runtime wrapper.
- A weapon component uses the managed weapon path where appropriate.
- A canonical Active Module component owns and uses `ActiveModuleRuntime`.
- A component configured as an Active Module should not also run its ManagedWeapon behavior merely because both systems are reachable from the same `Component` class.
- Inactive functionality should remain inactive; component definition/type determines which behavior is active.
- Legacy Skill/Spawn continue to use the separate legacy `Ability` path in the reported implementation.

The user's example was: create a rotating turret, assign a `Component` that uses `ManagedWeapon`; create a separate Active Module and assign it through `Component`, where it uses `ActiveModuleRuntime` and leaves ManagedWeapon inactive.

This is a useful mental model: **one common installed-component abstraction, with behavior selected by the component's canonical definition/type**, rather than every installed component running every possible subsystem.

Verify the exact implementation before making code claims.

---

## 7. Modular hull extensions — newly clarified design proposal

A separate Markdown proposal was created and then revised in the previous session:

`Redemptis-Tiariea-Hull-Extensions-Proposal.md`

The proposal is a design document only. It is **not implementation authorization** and does not change the current Shipmaker Milestone 1 priority order.

### Core concept

A modular hull extension is a structural assembly attached to a ship. It is not an independent ship and not merely an ordinary equipment component.

Extensions can provide:
- Additional hull geometry or structure.
- Protection or other structural properties.
- Capacity or specialized functionality.
- Integrated equipment that makes the extension useful by itself.
- Additional component mounting points for optional equipment.
- Potentially additional structural connector points, subject to the future hierarchy rules.

An extension should usually have a meaningful purpose, but it does not need to be a weapon. Integrated equipment should reuse the existing component system rather than inventing a separate weapon/ability/active-module runtime.

### The defining rule: typed, two-sided structural connections

This was the user's important correction and should be preserved as a central design requirement:

**The ship defines structural extension installation points. Each modular hull extension defines its own connector points. The extension can connect only when its connector type matches the ship installation point's connector type.**

The two sides are distinct authored data:

- **Ship-side:** `HullExtensionInstallationPoint`
  - Stable ID.
  - Connector type identifier.
  - Local position and orientation.
  - Optional capacity/connection restrictions.
- **Extension-side:** `HullExtensionConnectorPoint`
  - Stable ID.
  - Connector type identifier.
  - Local position and orientation.
- **Connection record:** identifies both `installationPointId` and `connectorPointId`.

Illustrative example:
- Ship installation point type: `side-hull-rail`
- Extension connector point type: `side-hull-rail`
- This pair is type-compatible, subject to transform and capacity validation.
- An extension connector of type `engine-feed` must not connect to a `side-hull-rail` point even if its sprite appears to fit.

The connector type names above are examples only. The actual connector taxonomy should be authored for the game's own construction vocabulary.

### Structural connectors are not equipment mounts

Keep these interfaces separate:
- Structural extension installation points and connector points connect hull assemblies.
- Weapon hardpoints, turret mounts, thruster vents, utility sockets, and Active Module mounts install components/equipment.

Do not use one generic attachment-point type for both structural extension assembly and equipment installation.

Once attached, an extension may contribute its own **component mounting points**. Those are separate from its structural connector points.

### Placement and multiple connections

- Authored transforms determine placement; visual proximity or overlap alone is not a valid connection.
- If an extension uses multiple connector points to connect to multiple ship installation points, all pairs must resolve to one consistent rigid transform.
- Multiple connections must not stretch, skew, or deform the extension.
- The first implementation may support one connection per extension if that is the sensible scope, but the data design should not make future multi-connector extensions impossible.
- The coordinate-frame convention (including whether connector rotations are mating-facing or same-facing) must be specified once and shared by validation, runtime assembly, editor previews, and serialization.
- Initial scope: extension connectors attach to installation points owned by the ship/base structural design.
- Do not assume extension-to-extension chaining. That would require a later explicit hierarchy and cycle/ownership policy.

### Runtime ownership direction

- `Ship` should own the assembled ship and installed extension instances.
- A likely future `HullExtensionDefinition` describes reusable extension data.
- A runtime `HullExtension` instance represents one installed extension.
- `Game` may own a canonical extension-definition registry, consistent with the existing registry ownership pattern.
- No separate `HullExtensionManager` is currently proposed.
- Component installations on the base hull and extensions should resolve into the ship's existing component ownership/update path, avoiding separate extension component update loops.

### Blueprint/loadout distinction

- The ship blueprint selects the base hull, extension instances/configuration, loadout, and ship-specific stats/identity.
- Each extension instance should have a stable instance ID.
- Structural connection records identify the ship installation point and extension connector point.
- Loadouts contain optional component installations only; they do not select a hull.
- Component mount references on an extension need a scoped identity (for example, extension instance ID plus local component mount ID) so multiple copies of the same extension definition do not collide.
- Structural connection references and component-mount references are separate concepts.

### Validation requirements proposed

At minimum:
1. Extension definitions resolve from the authoritative registry.
2. Extension instance IDs are unique within the ship blueprint.
3. Ship installation points resolve on the intended ship/base hull.
4. Extension connector points resolve within the selected extension definition.
5. Connector type identifiers match exactly.
6. Point capacities/usage restrictions are respected.
7. Multiple connector pairs resolve to one consistent rigid transform.
8. Extension component mount references resolve to the correct instance and local point.
9. Integrated component IDs resolve through the canonical component registry.
10. Integrated components pass the same compatibility rules as other built-ins.
11. Loadout installations cannot replace integrated components or occupy reserved points.
12. Duplicate/ambiguous structural and component references are rejected.
13. Dimensions and transforms are finite and valid.
14. Invalid configurations fail predictably rather than leaving partially assembled ships.

### Design identity

The user's intended direction is not a copy of ShellCore Command. The system combines broad modular construction inspiration with an original explicit interface rule: the ship advertises compatible structural interfaces and the extension must provide matching typed connectors. This makes compatibility authored, inspectable, testable, and suitable for a future Shipmaker editor.

---

## 8. Future final-phase code simplification and performance work

The user wants, **only after the main project is complete**, to use AI to simplify the codebase into something easier to learn, understand, and edit, while improving performance and preserving gameplay behavior.

This is a valid future goal, but it should be treated as a dedicated final refactoring/optimization phase, not mixed into current feature implementation.

Recommended safeguards for that eventual phase:
- Establish a full baseline of tests and gameplay behavior first.
- Profile before optimizing; do not guess where performance problems are.
- Refactor in small subsystems rather than rewriting the whole repository at once.
- Preserve public/runtime contracts or document intentional migrations.
- Use behavior-preservation tests, integration tests, and representative gameplay scenarios.
- Compare performance before/after with repeatable measurements.
- Keep changes reviewable and reversible.
- Improve naming, structure, comments, and documentation for human comprehension.
- Avoid reducing line count as a goal by itself; simpler code means clearer ownership and fewer concepts, not merely fewer lines.
- Do not allow an AI rewrite to silently alter balance, timing, physics, targeting, cooldowns, AI behavior, save compatibility, or visuals.

---

## 9. Recommended next actions

These are recommendations based on the previous discussion, not authorization to modify files.

1. **Reopen the repository in the new session and establish the actual state.**
   - `git status --short --untracked-files=all`
   - `git branch --show-current`
   - `git rev-parse HEAD`
   - Inspect `docs/SHIPMAKER-ARCHITECTURE-HANDOFF.md`, `ARCHITECTURE.md`, `CLAUDE.md`, and current tests.
2. **Reconcile the audit against the current checkout.** The shared audit is tied to the recorded HEAD above. Re-run a read-only audit if the checkout has changed.
3. **Keep the user-selected priority:** address the three Missing prerequisites first, then the two Partial items the user intended. Clarify the exact two partials if necessary.
4. **Do not begin `/shipmaker` editor implementation until the prerequisite milestone is actually complete.**
5. Keep the modular hull extension proposal as a separate future workstream until its interfaces and priority are approved.
6. Preserve the legacy-system repurposing direction for Infantry/Heroes and the newer canonical Ship-style system for Tanks, while designing the globally splittable/rejoinable squad mechanic separately.

---

## 10. Important open questions (not yet decided)

### Shipmaker / Milestone 1
- Which exact two Partial items did the user intend to complete after the three Missing items?
- What is the approved order and scope for each prerequisite implementation task?
- What is the complete validation error/reporting contract?
- What are the exact dimension projection conventions and permitted optional axes?
- What are the final turret interval semantics and authored initial-angle rules?
- What is the final shared muzzle transform coordinate convention?
- What is the intended Phaser display-object ownership/attachment context API?

### Modular hull extensions
- What connector type taxonomy should the game use?
- Are connector types exact string IDs, registry-backed definitions, or another typed contract?
- What are the mating coordinate-frame and rotation conventions?
- Does the first implementation support one connector pair per extension, or multi-connector rigid alignment?
- Can an extension ever attach to another extension? Initial proposal says not until explicitly designed.
- Which structural stats/behaviors can extensions contribute?
- How are integrated components represented and reserved?
- What are extension damage, detachment, repair, and destruction rules?
- Can extensions be removed/replaced during combat or only in construction/loadout contexts?

### Infantry / Heroes / Tanks / Squads
- How much of the legacy Skill/Spawn/Weapon/Passive runtime should be shared versus adapted?
- What makes the Hero framework expanded (unique abilities, progression, command aura, special slots, etc.)?
- How do individual tank units transition into and out of squads?
- What are squad movement/formation rules, especially urban line-up behavior?
- How do large/titan tanks remain individual while still interacting with the global squad system?

---

## 11. Suggested first message in the new ChatGPT session

You can paste this handoff file and then say:

> Read this project handoff as historical context, not as proof of the current repository state. First help me inspect the current local checkout and reconcile it with the Shipmaker Milestone 1 audit. Preserve my existing changes and do not modify, stage, commit, or push anything. Our immediate priority remains addressing the three Missing prerequisites first, then the two Partial items I selected. Keep the modular hull extension proposal and the Infantry/Heroes/Tanks design directions in context, but do not start implementing those workstreams unless I explicitly authorize them.

---

**End of handoff.**
