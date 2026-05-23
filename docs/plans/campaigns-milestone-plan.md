# Campaigns Mod — Detailed Milestone Task Plan

**Project:** Kitten Space Agency — Gameplay Mods  
**Spec version:** campaigns.md v1.0  
**Plan version:** 1.0  
**Status:** Draft  
**Depends on:** Missions Mod (`KSAMissions.dll`, `IMissionRegistry`, `IMissionController`)

> Each task has a unique ID (`C1-T##`, `C2-T##`, `C3-T##`), a scope, requirements, and a Definition of Done (DoD). Tasks within a milestone are ordered; earlier tasks are prerequisites for later ones unless noted.
>
> **Sequencing rule:** Start Campaigns M1 only after Missions M1 DoD is met. Start Campaigns M2 only after both Campaigns M1 DoD and Missions M2 DoD are met. Start Campaigns M3 only after Campaigns M2 DoD is met.

---

## Spike — Inter-mod Communication

**Goal:** Determine how KSACampaigns can obtain a live reference to `IMissionRegistry` and `IMissionController` from KSAMissions at runtime, given both are loaded by StarMap.API. This is a prerequisite to all of M1 and M2 — the chosen pattern will be used throughout the codebase.

**Context:** StarMap.API's inter-mod integration behaviour is currently unknown. Two candidate approaches exist: (A) cross-mod Harmony patching/reflection, and (B) StarMap.API's native mod service surface. Run both spikes in parallel; the decision task selects one and documents the pattern for all subsequent tasks.

**All subsequent tasks in M1 and M2 are blocked until C1-SP03 is resolved.**

---

### C1-SP01 — Spike: Harmony / direct assembly reference approach

**Scope:** Infrastructure / Research

**Requirements:**
- Create a minimal throwaway project `KSACampaigns.Spike` (separate from the main `KSACampaigns` project) that references `KSAMissions.dll` as a direct assembly reference (not a NuGet package).
- In `Spike.Awake()`, attempt to retrieve `IMissionRegistry.Instance` (the static singleton defined in M1-T10 of the Missions Mod plan) and call `GetAllBlueprints()`.
- Also attempt a Harmony postfix on a public method of `MissionController` from within the Campaigns spike project, to verify that cross-mod Harmony patches apply correctly.
- Document the following findings in a comment block at the top of `Spike.cs`:
  - Does StarMap.API enforce load order, or does `[BepInDependency]` / equivalent handle this?
  - Is the static singleton reference valid by the time `Awake()` runs in the dependent mod?
  - Does the cross-mod Harmony patch apply without error?
  - Are there any assembly isolation / versioning issues (e.g., type identity mismatch across load contexts)?

**Risks:** StarMap.API may use separate `AssemblyLoadContext` instances per mod, causing type identity mismatches — `IMissionRegistry` loaded in context A ≠ `IMissionRegistry` in context B even if the same DLL.

**Definition of Done:**
- Spike loads in-game alongside KSAMissions (both via StarMap.API).
- Log output confirms whether `IMissionRegistry.Instance` is non-null and returns blueprints.
- Log confirms whether the Harmony patch fires.
- Written findings note any errors, warnings, or unexpected behaviour.
- A `YES / NO / PARTIAL` verdict is recorded for: **(a)** static singleton accessible, **(b)** Harmony cross-mod patch works, **(c)** no type identity issues.

---

### C1-SP02 — Spike: StarMap.API native inter-mod service surface

**Scope:** Infrastructure / Research

**Requirements:**
- Read the StarMap.API source or documentation to identify any built-in mechanism for inter-mod communication: service locator, shared event bus, `IModRegistry`, mod messaging API, or equivalent.
  - Start at [StarMap.API GitHub / wiki](https://modding.kittenspaceagency.wiki) (search for "inter-mod", "mod API", "service", "dependency").
- In the same `KSACampaigns.Spike` project, implement an alternative retrieval path using whatever StarMap.API surface is found:
  - If StarMap.API has a service locator: register `IMissionRegistry` in KSAMissions and resolve it in KSACampaigns via the locator.
  - If StarMap.API has a mod messaging / event bus: publish a "registry ready" event from KSAMissions and subscribe from KSACampaigns.
  - If no native surface exists: document that explicitly and mark this approach as `N/A`.
- Document findings in the same `Spike.cs` comment block:
  - Which StarMap.API mechanism was found (name, namespace, method signatures).
  - Whether it guarantees load-order safety (i.e., KSACampaigns can't resolve before KSAMissions is ready).
  - Whether it is type-safe or stringly typed.
  - Whether it survives save/load (reference remains valid after a scene reload).

**Definition of Done:**
- StarMap.API source / docs reviewed; findings documented with references (file path or URL).
- If a native surface exists: spike successfully resolves `IMissionRegistry` through it in-game, confirmed by log.
- If no native surface exists: documented clearly as `N/A` with evidence.
- A `YES / NO / N/A` verdict recorded for: **(a)** native surface exists, **(b)** reference resolved successfully, **(c)** load-order safety guaranteed by the API.

---

### C1-SP03 — Decision: adopt inter-mod communication pattern

**Scope:** Infrastructure / Architecture

**Entry criteria:** C1-SP01 and C1-SP02 both have written verdicts.

**Requirements:**
- Review the verdicts from both spikes.
- Select one canonical pattern for all inter-mod calls in KSACampaigns using the decision matrix below:

  | Condition | Choose |
  | :--- | :--- |
  | StarMap.API native surface exists, type-safe, load-order safe | StarMap.API (SP02) |
  | StarMap.API native surface exists but stringly typed or unsafe | Evaluate risk; prefer SP02 only if SP01 has type identity issues |
  | SP01 static singleton works, no type identity issues | Direct assembly reference + static singleton (SP01) |
  | SP01 has type identity issues, SP02 is N/A | Investigate shared `AssemblyLoadContext` or `AppDomain` workaround |

- Write a brief (< 1 page) Architecture Decision Record (ADR) as `docs/adr/001-inter-mod-communication.md`:
  - **Context:** what was unknown.
  - **Decision:** which approach was chosen and why.
  - **Consequences:** impact on all subsequent tasks (e.g., "all inter-mod references use `StarMap.API.ServiceLocator.Resolve<IMissionRegistry>()`").
- Update `C1-T01` (project scaffold) and `C1-T11` (plugin wiring) task descriptions in this document to reflect the chosen pattern.
- Delete the `KSACampaigns.Spike` project.

**Definition of Done:**
- ADR written and committed to `docs/adr/`.
- Chosen pattern documented with a concrete code example in the ADR.
- This plan updated: any task that references "retrieve `IMissionRegistry`" uses the chosen pattern's exact call site.
- Spike project removed from solution.

---

## Milestone 1 — Model: Data, Load, Save

**Goal:** Campaign blueprints load from XML, link to the Missions Mod registry, and campaign instance state round-trips through save/load. No campaign game loop or player UI.

**Entry criteria:** Missions Mod M1 DoD met (`IMissionRegistry` working, Missions save/load working) **and C1-SP03 resolved**.

**Exit criteria (milestone DoD):**
- Valid campaign XML loads without error; broken `missionUID` references are logged and the campaign skipped.
- `ICampaignRegistry` is callable by other mods.
- Active campaign `uid` survives a save/load cycle.
- No crash from missing or malformed campaign XML.

---

### C1-T01 — Project scaffold & solution setup

**Scope:** Infrastructure

**Requirements:**
- Create a new C# class library project `KSACampaigns` targeting .NET 10, in the same repository as `KSAMissions`.
- Add Harmony 2.4.2 NuGet reference.
- Add references to KSA game assemblies and to `KSAMissions.dll` (as a local project reference).
- Set output path to the KSA mods directory.
- Add `CampaignsPlugin` entry-point class with `[BepInPlugin]` (or equivalent) that logs `"KSACampaigns loaded"` on startup.
- Guard startup with a `[BepInDependency("KSAMissions")]` attribute (or equivalent) so KSACampaigns only loads after KSAMissions.

**Definition of Done:**
- Solution builds without warnings.
- Plugin loads in-game after KSAMissions and log line appears in the correct order.
- `.gitignore` excludes build artifacts and game binaries.

---

### C1-T02 — `CampaignBlueprint` model class

**Scope:** Model

**Requirements:**
- Create `Models/CampaignBlueprint.cs`.
- Properties mapping every XML field in spec §6.1:
  - `string Version` (required)
  - `string Uid` (required, max 128 chars)
  - `string Title` (required, max 128 chars)
  - `string? Synopsis` (max 1024 chars)
  - `string? Description` (max 4096 chars)
  - `string StartMissionUID` (required; must appear in `MissionUIDs`)
  - `List<string> MissionUIDs` (required, min 1 entry)
- All properties PascalCase. No serialization attributes (handled in `CampaignReader`).
- XML doc comments on each property summarising the spec constraint.

**Definition of Done:**
- Class compiles cleanly.
- All spec §6.1 fields represented with correct CLR types.

---

### C1-T03 — `CampaignInstance` model class

**Scope:** Model

**Requirements:**
- Create `Models/CampaignInstance.cs`.
- Properties:
  - `string InstanceId` — GUID string, generated on creation.
  - `string BlueprintUid` — references a loaded `CampaignBlueprint`.
  - `CampaignState State` — enum `{ Inactive, Active, Completed }`. `Inactive` = instance created but campaign not yet started (pre-start placeholder, e.g. created from a save record whose blueprint no longer exists or reserved for future use); `Active` = player is playing; `Completed` = campaign completion detected.
  - `DateTime? StartedAt` — wall-clock timestamp when the campaign was started.
  - `DateTime? CompletedAt` — timestamp when the campaign completed.
- Constructor accepting a `CampaignBlueprint` that initialises `State` to `Inactive`.

**Definition of Done:**
- Class compiles cleanly.
- Constructor correctly initialises all fields.

---

### C1-T04 — `CampaignReader` — XML parsing

**Scope:** Model

**Requirements:**
- Create `Loaders/CampaignReader.cs`.
- Static method `CampaignBlueprint? TryRead(string filePath, ILogger logger)`.
- Uses `System.Xml.Linq` (`XDocument` / `XElement`) — no third-party XML lib.
- Parsing rules (spec §6.1):
  - Reads `<CampaignBlueprint>` root; returns null + logs error if root missing.
  - Maps all blueprint fields; applies empty-list default for `<missions>` if absent.
  - On any parsing exception: logs file path + error, returns null.
- Parses the full example XML from spec §6.1 (`first_steps.xml`).

**Definition of Done:**
- Correctly parses the spec §6.1 example into a `CampaignBlueprint` with all fields populated.
- Malformed XML (non-well-formed, missing `<uid>`) returns null and logs clearly.

---

### C1-T05 — `CampaignValidator` — validation rules

**Scope:** Model

**Requirements:**
- Create `Loaders/CampaignValidator.cs`.
- Method `ValidationResult Validate(CampaignBlueprint blueprint, IMissionRegistry missionRegistry)`.
- Reuses the `ValidationResult` type from `KSAMissions` (same assembly reference already required). Do not define a second `ValidationResult` type.
- Returns `ValidationResult` with `bool IsValid`, `List<string> Errors`, and `List<string> Warnings`.
- Implements all checks from spec §6.2:
  - **Error:** Required fields null/empty (`uid`, `title`, `version`, `startMissionUID`, non-empty `<missions>`).
  - **Error:** String length violations (uid ≤ 128, title ≤ 128, synopsis ≤ 1024, description ≤ 4096).
  - **Error:** Duplicate campaign `uid` (checked at loader level; validator checks intra-file only — no duplicate `<missionUID>` within one campaign).
  - **Error:** Any `<missionUID>` does not resolve in `IMissionRegistry`.
  - **Error:** `startMissionUID` is not listed in `<missions>`.
  - **Error:** Any prerequisite `missionUID` in a scoped mission's blueprint does not exist anywhere in `IMissionRegistry` — no campaign or mod provides it; the mission can never be satisfied.
  - **Info:** A prerequisite `missionUID` exists in `IMissionRegistry` but is not in this campaign's `<missions>` list — cross-campaign dependency; valid (sequential or parallel campaigns), flagged for author awareness.
  - **Error:** The `startMissionUID` mission has a prerequisite referencing a `missionUID` that does not exist in `IMissionRegistry` — campaign can never start.
  - **Warning:** The `startMissionUID` mission has a prerequisite referencing a cross-campaign `missionUID` (exists in registry, not in this campaign) — campaign start depends on another campaign's progress; valid if intentional.
  - **Warning:** A `<missionUID>` is unreachable from `startMissionUID` following within-campaign `MissionComplete` prerequisite edges — may still be reachable via a cross-campaign prerequisite, but flagged for author awareness.
- Warnings are collected in a separate `List<string> Warnings` on `ValidationResult`.

**Definition of Done:**
- Valid blueprint + populated registry → `IsValid = true`, no errors.
- Each error condition in spec §6.2 triggers at least one descriptive error string.
- Warning for orphan `<missionUID>` fires correctly.
- Unit tests covering every error rule, each in isolation (using a mock `IMissionRegistry`).

---

### C1-T06 — `CampaignLoader` — discovery & batch load

**Scope:** Model

**Requirements:**
- Create `Loaders/CampaignLoader.cs`.
- Method `IReadOnlyList<CampaignBlueprint> LoadAll(string modsRootPath, IMissionRegistry missionRegistry, ILogger logger)`.
- Scans all `campaigns/*.xml` under each mod subfolder of `modsRootPath` (spec §6.0).
- Per file: calls `CampaignReader.TryRead`; on success calls `CampaignValidator.Validate`.
- Skips (and logs) files that fail parse or validation.
- Detects duplicate `uid`s across files — logs error with both file paths, keeps first, skips duplicate.
- Returns only valid, non-duplicate blueprints.
- Requires `IMissionRegistry` to be fully populated before this call (i.e., Missions M1 startup must complete first).

**Definition of Done:**
- Given 3 valid and 2 invalid campaign XMLs, returns exactly 3 blueprints.
- Duplicate `uid` logs an error naming both file paths.
- No exception propagates to caller regardless of file content.

---

### C1-T07 — `ICampaignRegistry` interface & implementation

**Scope:** Model / Public API

**Requirements:**
- Create `Api/ICampaignRegistry.cs`:
  ```csharp
  public interface ICampaignRegistry
  {
      CampaignBlueprint? GetBlueprint(string uid);
      IReadOnlyList<CampaignBlueprint> GetAllBlueprints();
  }
  ```
- Create `CampaignRegistry.cs` implementing the interface, backed by the loaded blueprints list from `CampaignLoader`.
- Registry populated once at game start; no mutation after that.
- Expose `CampaignRegistry.Instance` static singleton accessible to other mods.

**Definition of Done:**
- `GetBlueprint("unknown")` returns null without throwing.
- `GetAllBlueprints()` returns a read-only view.
- A separate test assembly can reference `ICampaignRegistry` and call both methods.

---

### C1-T08 — `CampaignSaveData` model

**Scope:** Model / Save

**Requirements:**
- Create `Save/CampaignSaveData.cs`.
- Wraps the campaign save payload:
  - `List<string> ActiveCampaignUids` — uids of all currently `Active` campaigns (empty list if none). Multiple campaigns can be active simultaneously.
  - `List<CampaignInstanceRecord> Instances`
- `CampaignInstanceRecord`:
  - `string InstanceId`
  - `string BlueprintUid`
  - `string State` (enum serialised as string, e.g. `"Active"`)
  - `string? StartedAt` (ISO 8601)
  - `string? CompletedAt` (ISO 8601)
- Static helpers `Serialize(CampaignSaveData) → XElement` and `Deserialize(XElement) → CampaignSaveData`.
- Uses `System.Xml.Linq`.

**Definition of Done:**
- Round-trip test: create save data with 2 instances in different states and 2 active campaign UIDs, serialize to `XElement`, deserialize back, compare all fields — no data loss.
- Enum strings are human-readable (`"Active"` not `"1"`).
- Deserialize with empty `<activeCampaignUids/>` → empty list, no null reference exception.

---

### C1-T09 — Harmony save/load hooks

**Scope:** Model / Save

**Requirements:**
- Create `Save/CampaignSaveHooks.cs`.
- Follow the [KSA modding wiki save guide](https://modding.kittenspaceagency.wiki/en/Code/Save-and-Load-Mod-Data).
- Postfix patch on KSA save method: serialize `CampaignSaveData` into the mod's campaign save slot.
- Postfix patch on KSA load method: read the campaign save slot, deserialize, reconstruct `CampaignInstance` list.
- Handle missing/null mod save slot gracefully (fresh save = no active campaign).
- If a saved `BlueprintUid` no longer resolves in `ICampaignRegistry` (mod removed): log warning, skip that instance, remove its uid from `ActiveCampaignUids` if present.

**Definition of Done:**
- Manually set an active campaign `uid` in code, save, reload — `uid` is restored.
- Missing save slot loads without error.
- Tested with KSA game running (integration test).

---

### C1-T10 — Stub `CampaignController`

**Scope:** Controller (stub only for M1)

**Requirements:**
- Create `Controllers/CampaignController.cs`.
- Holds the `ICampaignRegistry` reference and a `List<CampaignInstance>` in memory (populated on load).
- Methods (stubs, no logic yet):
  - `void RegisterRegistry(ICampaignRegistry registry)` — stores registry reference.
  - `CampaignInstance? GetActiveCampaign()` — returns the currently active instance (null if none).
  - `IReadOnlyList<CampaignInstance> GetAllInstances()` — returns all instances.
- No scope filtering or game loop integration yet (those are M2).

**Definition of Done:**
- `RegisterRegistry` stores the reference without throwing.
- `GetActiveCampaign()` returns null on a fresh start.

---

### C1-T11 — Plugin wiring & startup sequence

**Scope:** Infrastructure

**Requirements:**
- In `CampaignsPlugin.Awake()` (or equivalent entry point):
  1. Wait for / retrieve `IMissionRegistry.Instance` from KSAMissions (must be populated first).
  2. Call `CampaignLoader.LoadAll(modsRootPath, missionRegistry, logger)` → populate `CampaignRegistry`.
  3. Call `CampaignController.RegisterRegistry(CampaignRegistry.Instance)`.
  4. Install Harmony save/load patches.
  5. Log number of campaign blueprints loaded.
- `modsRootPath` resolved from game's mod directory at runtime.

**Definition of Done:**
- Game starts; log shows `"KSACampaigns: N campaign(s) loaded"` after `"KSAMissions: N blueprint(s) loaded"`.
- Invalid campaign XML logs an error but does not crash.
- Valid campaign XML loads and is retrievable via `ICampaignRegistry`.

---

### C1-T12 — Demo campaign XML (`first_steps.xml`)

**Scope:** Content / Test data

**Requirements:**
- Create `campaigns/first_steps.xml` following the spec §6.1 example.
- Lists the mission UIDs from Missions Mod demo XMLs (at minimum `first_steps.m01_launch`, `first_steps.m02_suborbital`).
- `startMissionUID` = `first_steps.m01_launch`.
- Must pass `CampaignValidator`.

**Definition of Done:**
- File loads without errors in-game.
- `ICampaignRegistry.GetAllBlueprints()` returns it.
- `CampaignValidator.Validate` passes in a unit test (with mock `IMissionRegistry` that knows the mission UIDs).

---

## Milestone 2 — Controller & View: Game Loop

**Goal:** Full "First Steps" campaign is playable start-to-finish. Mission scope is restricted to campaign missions. Campaign selection UI exists. Progress survives save/load.

**Entry criteria:** Campaigns M1 DoD met + Missions M2 DoD met.

**Exit criteria (milestone DoD):**
- Player selects a campaign, plays ≥ 6 missions (including a branch) from start to completion.
- Only missions in the active campaign's `<missions>` list are offered during play.
- Campaign + mission progress survives save/load.
- Campaign completion detected and shown.

---

### C2-T01a — Spike: scope filter via Missions Mod hook

**Scope:** Controller / Research

**Requirements:**
- Define `IMissionScopeFilter` interface in `KSACampaigns` (`Api/IMissionScopeFilter.cs`):
  ```csharp
  public interface IMissionScopeFilter
  {
      bool IsInScope(string missionUID);
  }
  ```
- Propose adding `void RegisterScopeFilter(IMissionScopeFilter filter)` to `IMissionController` in the Missions Mod (`KSAMissions`). Implement it: `MissionController.EvaluateOffers()` skips any blueprint where `filter.IsInScope(uid)` returns `false`. If no filter is registered, all missions are in scope (sandbox behaviour preserved).
- Prototype this in a branch: add the method to `IMissionController` and `MissionController`; register a stub filter from `KSACampaigns` that logs "scope filter called for uid X".
- Document findings:
  - Does adding to `IMissionController` break any existing callers of that interface?
  - Does the registered filter reference survive a scene reload (or does it need to re-register on load)?
  - Any type identity concerns given the inter-mod communication pattern from C1-SP03?

**Definition of Done:**
- Stub filter registered from `KSACampaigns`; log confirms `IsInScope` is called during `EvaluateOffers()`.
- Written verdict: **Go / No-go**, with reasons.

---

### C2-T01b — Spike: scope filter via Harmony patch

**Scope:** Controller / Research

**Requirements:**
- In `KSACampaigns`, implement a Harmony Postfix (or Prefix with skip) on `MissionController.EvaluateOffers()` that reads the active campaign scope from `CampaignController` and skips out-of-scope missions.
- No changes to the Missions Mod.
- Document findings:
  - Does the Harmony patch apply cleanly given the inter-mod communication pattern from C1-SP03?
  - Is `MissionController.EvaluateOffers()` a single call site or split across multiple methods that all need patching?
  - Any fragility concerns (e.g., Missions Mod refactor breaks the patch silently)?

**Definition of Done:**
- Patch applied; with a mock scope list of [A, B], mission C is not offered.
- Written verdict: **Go / No-go**, with reasons.

---

### C2-T01c — Implementation: mission scope filter

**Scope:** Controller

**Entry criteria:** C2-T01a and C2-T01b both have written verdicts.

**Requirements:**
- Select the approach using the following matrix:

  | Condition | Choose |
  | :--- | :--- |
  | Both Go | Hook (C2-T01a) — cleaner contract, no cross-mod patching |
  | Only C2-T01a Go | Hook |
  | Only C2-T01b Go | Harmony patch |
  | Both No-go | Escalate — find alternative before proceeding |

- Implement the chosen approach fully:
  - Add `void SetMissionScope(IReadOnlyList<string> missionUIDs)` to `CampaignController`. Scope is the union of all active campaigns' `MissionUIDs` lists (multiple active campaigns → union). With no active campaigns, scope is unrestricted.
  - `CampaignController` implements `IMissionScopeFilter`; `IsInScope` returns `true` if `missionUID` is in the union scope, or always `true` if the union scope is empty (no active campaign).
  - Wire filter registration or patch installation in `CampaignsPlugin`.
- Write decision to `docs/adr/002-scope-filter.md` (same format as ADR 001).

**Definition of Done:**
- With two active campaigns (scopes [A, B] and [B, C]), missions A, B, C are offered; mission D is not.
- With no active campaign, all missions are offered.
- Unit tests using a mock `IMissionController` verify both conditions.
- ADR written and committed.

---

### C2-T02 — Campaign start logic

**Scope:** Controller

**Requirements:**
- Add to `CampaignController`:
  - `bool StartCampaign(string campaignUid)` — creates a new `CampaignInstance` with `State = Active`, sets scope to the campaign's `<missions>` list, then calls `IMissionController.EvaluateOffers()`.
  - The start mission is offered through the normal `EvaluateOffers()` flow — **no prerequisite bypass**. The start mission must have prerequisites that are satisfiable on a fresh save (validated in C1-T05); `EvaluateOffers()` will offer it naturally.
  - Returns `false` if uid unknown or an `Active` instance for this specific campaign uid already exists (same campaign can't run twice in parallel; different campaigns can).
  - After creating the new instance, calls `SetMissionScope(union of all active campaigns' MissionUIDs)` to include the new campaign's missions in the filter.
- Raise C# event `CampaignStarted` (event args: `instanceId`, `campaignUid`).

**Rationale:** Prerequisites are always evaluated to keep mission state consistent across multiple concurrently loaded campaigns. Bypassing prerequisites for the start mission would break campaigns that share missions or have overlapping prerequisite graphs.

**Definition of Done:**
- Calling `StartCampaign("ksa.demo.first_steps")` on a fresh save: creates an instance, sets scope, and the start mission is `Offered` via `EvaluateOffers()` (not forced).
- Calling `StartCampaign` again for the same campaign returns `false` without changing state.
- Unit test: start mission has no prerequisites → offered after `StartCampaign`. A second campaign with a different start mission and no prerequisites → also offered after its `StartCampaign`; both are `Offered` simultaneously.

---

### C2-T03 — Campaign completion detection

**Scope:** Controller

**Requirements:**
- Create `Controllers/ReachabilityEvaluator.cs` with method:
  ```csharp
  bool CanEverUnlock(string missionUID,
                     IReadOnlyList<MissionBlueprint> scopedBlueprints,
                     IMissionRegistry fullRegistry,
                     IReadOnlyList<MissionInstance> allInstances)
  ```
  - `scopedBlueprints` — blueprints belonging to the campaign being evaluated.
  - `fullRegistry` — the complete `IMissionRegistry` (all loaded mods); used to look up cross-campaign missions.
  - `allInstances` — **all** mission instances across all active campaigns and sandbox, not just the current campaign's scope.
  - Returns `true` if there exists at least one assignment of future mission states (for missions not yet in a terminal state) under which the target mission's prerequisites would be satisfied.
  - Returns `false` when all paths to satisfying the prerequisites are permanently blocked by current terminal states. Blocking conditions:
    - `MissionCompletedPrerequisite(X)` — permanently false when X's instance is `Failed`, `Expired`, or `Rejected`.
    - `MissionFailedPrerequisite(X)` — permanently false when X's instance is `Completed`.
    - `MissionNotOfferedPrerequisite(X)` — permanently false when X has any instance (ever offered).
    - `MaxCompletedPrerequisite(max)` — permanently false when the completed count already equals `max`.
    - `MaxFailedPrerequisite(max)` — permanently false when the failed count already equals `max`.
    - `MaxRejectedPrerequisite(max)` — permanently false when the rejected count already equals `max`.
    - `PrerequisiteGroup(All)` — permanently false when any child is permanently false.
    - `PrerequisiteGroup(Any)` — permanently false when all children are permanently false.
  - **Cross-campaign missions:** a prerequisite referencing a mission not in `scopedBlueprints` is looked up in `fullRegistry`. If it exists in the registry but has no instance yet, it is treated as **potentially achievable** (not permanently false) — another campaign or sandbox could complete it in the future. If it has an instance, its actual state is evaluated using the same blocking conditions above. If it does not exist in `fullRegistry` at all, it is permanently false (unknown mission).
- Add `void CheckCampaignCompletion()` to `CampaignController`:
  - Called whenever any mission state transitions (subscribe to `IMissionController.MissionStateChanged`).
  - A campaign is **complete** when: no scoped mission is `Offered` or `Accepted`, **and** every scoped `Locked` mission returns `false` from `CanEverUnlock`.
  - Transition `CampaignInstance.State` to `Completed`, set `CompletedAt`, raise `CampaignCompleted` event.

**Definition of Done:**
- `CampaignCompleted` event fires exactly once per completion.
- Unit test — linear completion: 3 missions in a chain (m1 → m2 → m3); all reach `Completed` → campaign completes.
- Unit test — branch, main path taken: missions m1, m2a (`MissionCompletedPrerequisite(m1)`), m2b (`MissionFailedPrerequisite(m1)`). m1 completes, m2a offered and completed. m2b is `Locked` with `MissionFailedPrerequisite(m1)` — permanently false because m1 is `Completed`. `CanEverUnlock(m2b)` returns `false` → campaign completes despite m2b never leaving `Locked`.
- Unit test — branch, not yet decided: m1 is `Accepted` (not terminal). m2a and m2b are `Locked`. `CanEverUnlock` returns `true` for both (m1's outcome is still open) → `CheckCampaignCompletion` does **not** fire.
- Unit test — `MaxCompleted` cap reached: `MaxCompletedPrerequisite(max=1)` on a mission; 1 completed instance already exists → `CanEverUnlock` returns `false`.
- Unit test — cross-campaign, no instance yet: campaign B has a mission with `MissionCompletedPrerequisite(campaign_A_mission)`. Campaign A's mission has no instance yet (campaign A not started). `CanEverUnlock` returns `true` — it may be completed in the future.
- Unit test — cross-campaign, sequential completed: same setup, but campaign A's mission now has a `Completed` instance → prerequisite is satisfied → `CanEverUnlock` returns `true` (mission is in fact now unlockable).
- Unit test — cross-campaign, permanently blocked: campaign B has `MissionCompletedPrerequisite(campaign_A_mission)` but campaign A's mission is `Failed` and `MaxFailed max=1` is already hit → `CanEverUnlock` returns `false`.
- Unit test — unknown mission: prerequisite references a `missionUID` not in `fullRegistry` → `CanEverUnlock` returns `false`.
- `ReachabilityEvaluator` is unit-tested independently from `CampaignController`, using hand-crafted blueprint and instance lists.

---

### C2-T04 — `ICampaignController` public interface

**Scope:** Controller / Public API

**Requirements:**
- Create `Api/ICampaignController.cs`:
  ```csharp
  public interface ICampaignController
  {
      IReadOnlyList<CampaignInstance> GetAllInstances();
      IReadOnlyList<CampaignInstance> GetActiveCampaigns();
      CampaignInstance? GetCampaign(string campaignUid);
      bool StartCampaign(string campaignUid);
      event EventHandler<CampaignStartedEventArgs> CampaignStarted;
      event EventHandler<CampaignCompletedEventArgs> CampaignCompleted;
  }
  ```
- `GetActiveCampaigns()` returns all instances with `State == Active`; returns an empty list if none.
- `GetCampaign(string campaignUid)` returns the most recent instance for the given uid, or null if none exists.
- `CampaignController` implements `ICampaignController`.
- Expose `ICampaignController.Instance` static accessor.
- Event args carry `instanceId` and `campaignUid`.

**Definition of Done:**
- A separate test assembly can use `ICampaignController` without referencing internal types.
- `GetActiveCampaigns()` returns all active instances when two campaigns are running in parallel.
- Both event types carry expected data in unit tests.

---

### C2-T05 — Game mode hook & startup integration

**Scope:** Controller / Infrastructure

**Requirements:**

**Phase 1 — Research (timebox: 2 hours):**
- Identify the KSA method(s) to hook for new-game creation and game-load events:
  - Decompile KSA game assemblies (or search the modding wiki/community) for the new-game flow entry point — the method called when the player confirms "New Game" on the main menu.
  - Identify whether there is a distinct "new game" event vs "load game" event, or a single "session start" event with a flag.
  - Identify how to distinguish sandbox mode from campaign mode at hook time (e.g., a game mode enum, a save metadata field, or a flag set by the campaign selection UI).
- Document findings in `docs/research/ksa-newgame-hook.md`:
  - Method name, class, namespace, and signature.
  - How to detect new-game vs load-game vs sandbox.
  - Whether a Harmony Postfix, Prefix, or a native event/callback is appropriate.
  - Go / No-go for each approach found.

**Phase 2 — Implementation:**
- Patch the identified KSA new-game method: inject a call to show the campaign selection UI (C2-T06) before the game session starts.
- Patch the identified KSA load-game method: restore active campaign state from `CampaignSaveData` (already done in C1-T09) and re-apply scope filter via `SetMissionScope(union of all active campaign scopes)`.
- When the player selects sandbox mode, ensure scope filter is in pass-through mode (empty active campaigns list).

**Definition of Done:**
- Research doc written with identified method(s) and chosen approach.
- New save → campaign selection UI shown (C2-T06) → correct campaign starts.
- Loaded save → scope filter restored, all active campaigns match saved state.
- Sandbox save → all missions available (no scope filter).

---

### C2-T06 — IMGUI Campaign Selection screen

**Scope:** View

**Requirements:**
- Create `Views/CampaignSelectionView.cs`.
- Shown on new-game creation or via a "Change Campaign" option (accessible before a campaign is active).
- Lists all `ICampaignRegistry.GetAllBlueprints()` as selectable entries showing:
  - `Title` (header)
  - `Synopsis` (if present)
  - `Description` (if present, scrollable)
  - Mission count
- "Start Campaign" button → calls `CampaignController.StartCampaign(selectedUid)`, closes view. Button is greyed out for any campaign that is already `Active` (same campaign cannot run twice; a different campaign can be started in parallel).
- "Active campaigns" summary — if `GetActiveCampaigns()` is non-empty, display a list of currently active campaign titles above the campaign list with a "Continue (all)" button that re-enters the game without starting anything new.
- "Sandbox" button — starts a save with no campaign (scope filter pass-through).

**Definition of Done:**
- With 2 campaigns loaded, both appear in the list with title and synopsis.
- Clicking "Start Campaign" triggers `StartCampaign` and closes the panel.
- If one campaign is already active, its "Start Campaign" button is greyed out; the other campaign's button is still enabled.
- "Active campaigns" summary visible when at least one campaign is `Active`.

---

### C2-T07 — IMGUI Campaign Progress overview

**Scope:** View

**Requirements:**
- Create `Views/CampaignProgressView.cs`.
- Accessible from a "Campaign" button in the KSA toolbar (or equivalent) during active campaign play.
- Displays:
  - Campaign `Title` and current `State`.
  - A flat list of all scoped missions with their current `MissionState` (from `IMissionController`), shown as status icons. **Note:** IMGUI may not support these unicode glyphs (`🔒 📋 ▶ ✅ ❌`) depending on the font KSA bundles. First verify rendering in-game; fall back to ASCII labels (`[LOCKED]`, `[OFFERED]`, `[ACTIVE]`, `[DONE]`, `[FAILED]`) if glyphs do not render. If the game's font does support extended unicode, use the glyphs.
  - Optional `synopsis` per mission if available from `IMissionRegistry`.
- Updates on `IMissionController.MissionStateChanged` event (no polling).

**Definition of Done:**
- During an active campaign with 3+ missions in mixed states, all appear with correct icons.
- When a mission state changes (e.g., accepted → completed), the icon updates without reopening the view.

---

### C2-T08 — IMGUI Campaign Completion summary

**Scope:** View

**Requirements:**
- Create `Views/CampaignCompletionView.cs`.
- Subscribes to `ICampaignController.CampaignCompleted`.
- On campaign completion, displays a summary modal window:
  - Campaign `Title`.
  - Final mission states (completed vs failed vs expired breakdown).
  - "Return to Menu" button — saves and returns to main menu.
  - "Continue Playing" button — dismisses modal. The completed campaign's missions are removed from the active scope union (since it is now `Completed`, not `Active`). If no other campaigns remain `Active`, scope filter returns to pass-through (all missions available). The player can then start a new campaign or play sandbox freely.

**Definition of Done:**
- Trigger campaign completion manually in debug → modal appears with correct stats.
- "Continue Playing" dismisses without crashing; completed campaign removed from active scope; if no other campaign is active, scope becomes pass-through.

---

### C2-T09 — Demo campaign playthrough (≥ 6 missions, branching)

**Scope:** Content

**Requirements:**
- Extend the `first_steps.xml` campaign to include at least 6 missions, with at least one branch point:
  - The branch is encoded via mission prerequisites in the mission XML files (e.g., m04a and m04b have mutually exclusive prerequisites via `MissionComplete/MissionFailed`).
  - At least two possible "endings" (all core missions complete, or a retry/alternative path).
- Add any required new mission XML files to the Missions Mod `missions/` folder.
- All new missions must pass `MissionValidator`.

**Definition of Done:**
- Player can play the campaign start-to-finish on the "main path" (≥ 6 missions) without a crash.
- Taking the branch (failing a mission) causes the alternative mission to be offered.
- Campaign completion fires at the end of each path.

---

### C2-T10 — Save/load integration with mission state

**Scope:** Model / Save

**Requirements:**
- Verify that campaign `ActiveCampaignUid` + scope filter are restored correctly on load (C1-T09 + C2-T05 wiring).
- Integration test:
  1. Start campaign, accept mission, complete one objective, save mid-mission.
  2. Reload — verify: campaign instance is `Active`, scope filter is active, accepted mission objective states are restored (from Missions Mod save), campaign progress view shows correct states.
- Verify `CampaignCompleted` does not re-fire after reload of a completed campaign.

**Definition of Done:**
- All campaign and mission state survives a mid-campaign save/load cycle.
- No events double-fire after reload.

---

## Milestone 3 — View: Editor, Graph & Polish

**Goal:** Mission authors can create and edit campaigns in-game using an IMGUI editor. A graph view visualises mission structure and live progress. Hot-reload works without restart.

**Entry criteria:** Campaigns M2 DoD met.

**Exit criteria (milestone DoD):**
- Campaign created in editor plays to completion without hand-editing XML.
- Editor XML passes `CampaignReader` round-trip.
- Graph view matches prerequisites and live progress.
- Hot-reload works without restart.

---

### C3-T01 — `CampaignWriter` — XML serialisation

**Scope:** Model

**Requirements:**
- Create `Loaders/CampaignWriter.cs`.
- Method `XDocument Write(CampaignBlueprint blueprint)`.
- Produces XML matching the schema used by `CampaignReader` — round-trip safe.
- Indents output (2-space) and includes `<?xml version="1.0" encoding="utf-8"?>` header.
- Method `Save(CampaignBlueprint blueprint, string outputDir)` writes to `campaigns/<uid>.xml`. Creates `outputDir` if it does not exist; throws descriptive `IOException` if creation fails (do not silently swallow).

**Definition of Done:**
- Round-trip test: load `first_steps.xml`, write to temp file, reload → all fields equal.
- Saved file is valid UTF-8 XML.

---

### C3-T02 — Campaign hot-reload (`F9`)

**Scope:** Controller / Infrastructure

**Requirements:**
- In debug builds only (`#if DEBUG`), register a key listener for `Shift+F9` (separate from Missions Mod's `F9` to keep mods independent; both can be pressed together to reload all).
- On reload:
  1. Call `CampaignLoader.LoadAll(...)` against the current `IMissionRegistry`.
  2. Diff new blueprints vs existing registry; update changed entries.
  3. Do **not** drop an active `CampaignInstance` whose uid still exists.
  4. Log `"Hot-reload: N campaign(s) reloaded"`.
- In release builds, key does nothing.

**Definition of Done:**
- In debug: modify a campaign title in XML, press F9 in-game, `ICampaignRegistry.GetBlueprint(uid).Title` returns the new title.
- Active campaign instance survives hot-reload with scope and state intact.

---

### C3-T03 — IMGUI Campaign Editor — metadata panel

**Scope:** View

**Requirements:**
- Create `Views/Editor/CampaignEditorWindow.cs` — top-level IMGUI editor window.
- Left panel: list of all loaded campaign blueprints (title + uid); "New Campaign" button.
- Main panel when a campaign is selected: editable fields for all `CampaignBlueprint` metadata (`uid`, `version`, `title`, `synopsis`, `description`).
- Inline validation: red text for fields violating length limits or empty required fields.
- `startMissionUID` field: text input with dropdown of mission UIDs currently in this campaign's `<missions>` list.
- "Save" button — calls `CampaignWriter.Save(blueprint, outputDir)`, then hot-reloads that file.
- "Play-test" button — if no `Active` instance exists for this campaign, calls `CampaignController.StartCampaign(uid)`. If an `Active` instance already exists (campaign already running), shows an inline warning "Campaign already active — stop it first?" with a "Reset & Play-test" option that transitions the existing instance to `Completed` (not saved) and starts a fresh one. Debug builds only (`#if DEBUG`).

**Definition of Done:**
- Editor opens from a toolbar button or debug menu.
- Editing and saving `title` persists to disk and hot-reloads correctly.
- Play-test button starts the campaign when not active; shows reset prompt when already active.

---

### C3-T04 — IMGUI Campaign Editor — mission membership panel

**Scope:** View

**Requirements:**
- Sub-panel within `CampaignEditorWindow` for editing the `<missions>` list.
- Shows all `<missionUID>` entries with Up/Down reorder buttons and a Delete button (with confirmation).
- "Add Mission" — searchable dropdown of all missions in `IMissionRegistry` not yet in the list.
- Inline warning when a listed `missionUID` is not in `IMissionRegistry` (e.g., mod removed).
- `startMissionUID` dropdown automatically updates its available options from the current list.
- Changes reflected in-memory immediately; committed to disk only on "Save".

**Definition of Done:**
- Add a mission UID from the registry, save, reload → mission appears in campaign scope.
- Remove a mission UID, save, reload → mission is no longer offered during campaign play.
- Adding an unknown UID shows an inline warning (not an error; allows forward-authoring). The Save button remains enabled when only warnings are present; it is disabled only on validation errors (consistent with C3-T03).

---

### C3-T05 — Graph View: DAG layout (baseline)

**Scope:** View

**Requirements:**
- Create `Views/Editor/CampaignGraphView.cs` — IMGUI resizable window, default size 900×600 px, minimum 400×300 px. Usable both embedded in `CampaignEditorWindow` (as a docked sub-panel) and as a standalone floating window opened via the "Graph" button in `CampaignProgressView`.
- Implements a **left-to-right DAG layout**:
  - Assign each mission a depth via BFS from `startMissionUID` following `MissionComplete` prerequisite edges **within the campaign scope only** (cross-campaign edges ignored for layout). Missions with no within-campaign incoming edges default to depth 0. If multiple missions share depth 0 (e.g., because all their prerequisites are cross-campaign), they are stacked vertically at column 0 and a layout warning is logged.
  - Arrange missions into columns by depth and rows by insertion order within each column.
  - Node positions are computed from (column, row) slots using configurable constants for horizontal and vertical spacing.
  - If a cycle is detected (not expected but possible with malformed XML), log a warning and break the cycle by capping depth at the first repeated visit.
- Renders each mission as a labelled rectangle node; edges as straight lines between node centres with an arrowhead.
- Node size, colour, and spacing configurable via constants.
- Clicking a node in editor mode selects that mission in the membership panel.
- The graph is the **default** view; C3-T06b (if implemented) adds an enhanced view as an alternative selectable via a toggle button.

**Definition of Done:**
- `first_steps` campaign graph renders all missions as nodes with correct directed edges.
- A branch point (two successors) displays both outgoing edges correctly.
- Orphan missions (no edges) appear as isolated nodes at depth 0.
- Cycle in input → logged, no infinite loop, graph renders a best-effort layout.

---

### C3-T06 — Graph View: live progress overlay

**Scope:** View

**Requirements:**
- Extend `CampaignGraphView` with a "Progress" mode toggle (editor vs live play). Applies to whichever graph layout is active (C3-T05 baseline or C3-T06b enhanced, if present).
- In progress mode, node colour reflects current `MissionState` from `IMissionController`:
  - Grey = Locked
  - Yellow = Offered
  - Blue = Accepted
  - Green = Completed
  - Red = Failed / Expired
  - White = Rejected
- Updates on `IMissionController.MissionStateChanged` event (no polling).
- Accessible from `CampaignProgressView` (C2-T07) via a "Graph" button.

**Definition of Done:**
- During active campaign play, node colours match current mission states.
- When a mission transitions to `Completed`, its node turns green without reopening the view.

---

### C3-T06b — Graph View: enhanced visual layout *(optional, non-blocking)*

**Scope:** View / Research

> **Optional:** this task does not block C3-T07 or C3-T08. Implement after the baseline (C3-T05) and progress overlay (C3-T06) are working. C3-T05 remains the fallback if this task is not completed or is cut.

**Requirements:**

**Phase 1 — Research (timebox: 2–3 hours):**
- Investigate whether KSA's existing graph/plot rendering is exposed to mods via IMGUI or a dedicated API.
  - Inspect KSA's UI assemblies (decompiler or source) for any `ImPlot`, `ImNodes`, or custom graph-drawing types already used by the game (the game ships with at least one graph UI, e.g. delta-v / trajectory displays).
  - Determine if `ImPlot` or `ImNodes` is bundled with KSA's IMGUI build and callable from a mod.
  - If neither is available, assess feasibility of shipping `ImNodes` or a pure-IMGUI custom node renderer as part of the mod.
- Document findings in `docs/research/graph-rendering.md`:
  - What is available (class names, namespaces, version).
  - Whether mod access is possible (public API, reflection, or not at all).
  - Recommended approach with rationale.
  - Go / No-go recommendation for the rest of this task.

**Phase 2 — Implementation (proceed only on Go):**
- Create `Views/Editor/CampaignGraphViewEnhanced.cs`, replacing the IMGUI widget inside `CampaignGraphView` when the "Enhanced" toggle is active (baseline DAG layout remains available via toggle).
- **Background:** render a scrollable, zoomable canvas. The background is a tileable space/nebula image (asset path configurable; falls back to a solid dark colour if the asset is missing). Canvas scrolls horizontally and vertically by click-and-drag; scroll position persists while the window is open.
- **Node rendering:** mission nodes drawn on top of the background using whatever rendering approach the research identified. Each node shows the mission `Title` (or `uid` if no title). Node positions use the DAG column/row layout from C3-T05 as the initial default; nodes can be repositioned by drag and the positions saved to a per-campaign layout file (`campaigns/<uid>.layout.json`).
- **Edge rendering:** curved or straight arrows between nodes (prefer curved if the rendering API supports it).
- **Node interaction — click:** clicking a mission node opens a **side panel** (right side of the graph window) showing:
  - Mission `Title` and `uid`.
  - `Synopsis` and `Description` (scrollable).
  - Current `MissionState` (if in progress mode).
  - Prerequisite summary (human-readable, e.g. "Requires: m01 completed").
  - "Play-test from here" button (debug builds only; same behaviour as C3-T07).
- The side panel is collapsible; when no node is selected it is hidden (full canvas width used).

**Definition of Done:**
- Research doc (`docs/research/graph-rendering.md`) written with a clear Go / No-go verdict.
- *(If Go)* Enhanced view renders in-game with scrollable background, nodes, and edges.
- Clicking a node opens the side panel with correct mission data.
- Dragging a node repositions it; position persists across window close/open within the session.
- Layout file saves/loads correctly; missing layout file falls back to DAG column/row positions.
- "Enhanced" / "Simple" toggle switches between C3-T06b and C3-T05 layouts without losing selected node or progress state.
- *(If No-go)* Research doc explains why and C3-T05 remains the only graph layout.

---

### C3-T07 — Play-test from selected mission

**Scope:** Controller / View

**Requirements:**
- In `CampaignEditorWindow` and both graph views (C3-T05 baseline and C3-T06b enhanced side panel), expose a "Play-test from here" button per mission node.
- Action:
  1. Sets campaign scope (if not already active).
  2. Calls an override on `IMissionController` to directly set the selected mission to `Offered`, bypassing all prerequisites.
  3. All missions that would normally be offered before it remain `Locked` (tester skips ahead).
- Only available in debug builds (`#if DEBUG`).

**Definition of Done:**
- Click "Play-test from here" on m03 → m03 appears as `Offered` in-game regardless of m01/m02 state.
- No crash or invalid state if precondition missions are still `Locked`.

---

### C3-T08 — `docs/campaign-xml-guide.md`

**Scope:** Documentation

**Requirements:**
- Write `docs/campaign-xml-guide.md` covering:
  - File location and naming convention (`campaigns/<name>.xml`).
  - Full annotated XML example with all fields (mirrors spec §6.1 but in user-facing language).
  - Field reference table.
  - How campaigns link to missions (no back-link in mission files).
  - How to define branching paths (via mission prerequisites).
  - Common patterns: linear campaign, branching campaign, retry path.
  - Validation error and warning reference.
  - How to use the in-game editor vs hand-authoring XML.

**Definition of Done:**
- Document is self-contained — a modder with no access to the spec can author a valid campaign XML using only this guide.
- All XML examples in the guide pass `CampaignValidator` in a unit test (using a mock registry).

---

## Appendix — Task Summary Table

| ID | Milestone | Layer | Title |
| :-- | :-- | :-- | :-- |
| C1-SP01 | Spike | Infra | Inter-mod: Harmony / direct assembly reference |
| C1-SP02 | Spike | Infra | Inter-mod: StarMap.API native surface |
| C1-SP03 | Spike | Arch | Decision: adopt inter-mod communication pattern |
| C1-T01 | 1 | Infra | Project scaffold |
| C1-T02 | 1 | Model | `CampaignBlueprint` |
| C1-T03 | 1 | Model | `CampaignInstance` |
| C1-T04 | 1 | Model | `CampaignReader` |
| C1-T05 | 1 | Model | `CampaignValidator` |
| C1-T06 | 1 | Model | `CampaignLoader` |
| C1-T07 | 1 | API | `ICampaignRegistry` |
| C1-T08 | 1 | Save | `CampaignSaveData` |
| C1-T09 | 1 | Save | Harmony save/load hooks |
| C1-T10 | 1 | Controller | Stub `CampaignController` |
| C1-T11 | 1 | Infra | Plugin wiring |
| C1-T12 | 1 | Content | Demo campaign XML |
| C2-T01a | 2 | Controller | Scope filter spike — Missions Mod hook |
| C2-T01b | 2 | Controller | Scope filter spike — Harmony patch |
| C2-T01c | 2 | Controller | Scope filter implementation |
| C2-T02 | 2 | Controller | Campaign start logic |
| C2-T03 | 2 | Controller | Campaign completion detection |
| C2-T04 | 2 | API | `ICampaignController` (multi-campaign) |
| C2-T05 | 2 | Infra | Game mode hook & startup integration |
| C2-T06 | 2 | View | Campaign selection screen |
| C2-T07 | 2 | View | Campaign progress overview |
| C2-T08 | 2 | View | Campaign completion summary |
| C2-T09 | 2 | Content | Demo campaign playthrough (6+ missions) |
| C2-T10 | 2 | Save | Save/load integration |
| C3-T01 | 3 | Model | `CampaignWriter` |
| C3-T02 | 3 | Infra | Campaign hot-reload (F9) |
| C3-T03 | 3 | View | Editor — metadata panel |
| C3-T04 | 3 | View | Editor — mission membership panel |
| C3-T05 | 3 | View | Graph view — DAG layout (baseline) |
| C3-T06 | 3 | View | Graph view — live progress overlay |
| C3-T06b | 3 | View | Graph view — enhanced visual layout *(optional)* |
| C3-T07 | 3 | View | Play-test from selected mission |
| C3-T08 | 3 | Docs | `campaign-xml-guide.md` |
