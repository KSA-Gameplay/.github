# Missions Mod — Detailed Milestone Task Plan

**Project:** Kitten Space Agency — Gameplay Mods  
**Spec version:** missions.md v1.0  
**Plan version:** 1.0  
**Status:** Draft

> Each task has a unique ID (`M1-T##`, `M2-T##`, `M3-T##`), a scope, requirements, and a Definition of Done (DoD). Tasks within a milestone are ordered; earlier tasks are prerequisites for later ones unless noted.

---

## Milestone 1 — Model: Data, Load, Save

**Goal:** Mission blueprints load from XML, validate, and instance state round-trips through save/load. No gameplay loop or player UI.

**Exit criteria (milestone DoD):**
- Valid mission XML loads without error.
- Invalid XML is logged and skipped — no crash.
- Save/load restores all mission instance fields without data loss.
- `IMissionRegistry` is callable by other mods.

---

### M1-T01 — Project scaffold & solution setup

**Scope:** Infrastructure

**Requirements:**
- Create a new C# class library project `GPMissions` targeting .NET 10.
- Add Harmony 2.4.2 NuGet reference.
- Add reference to KSA game assemblies (as local references, not NuGet).
- Set output path to the KSA mods directory for fast iteration.
- Add a "Hello World" print to test if the mod is loading.

**Definition of Done:**
- Solution builds without warnings.
- The plugin loads in-game and the startup log line appears.
- Repository has a `.gitignore` excluding build artifacts and game binaries.

---

### M1-T02 — `MissionBlueprint` model class

**Scope:** Model

**Requirements:**
- Create `Models/MissionBlueprint.cs`.
- Properties mapping to every XML field in spec §6.1:
  - `string uid` (required)
  - `string title` (required)
  - `string? synopsis`
  - `string? description`
  - `float? expiration` (seconds; 0 = none)
  - `bool isRejectable` (default `true`)
  - `float? deadline` (seconds; 0 = none)
  - `bool isAutoAccepted` (default `false`)
  - `Prerequisite? prerequisite`
  - `List<Objective> objectives` (required, min 1)
  - `List<MissionAction> actions`
- All properties use camelCase.
- All properties that are serializable have XML atributes for XML deserialization

**Definition of Done:**
- Class compiles cleanly.
- All spec §6.1 fields are represented with correct CLR types.
- XML doc comments on each property summarising the spec constraint.

---

### M1-T03 — `Prerequisite` model classes

**Scope:** Model

**Requirements:**
- Create `Models/Prerequisites/` folder.
- Abstract base `Prerequisite`.
- Mission-state concrete types (spec §6.2):
  - `MissionCompletedPrerequisite` — `string missionUID`
  - `MissionFailedPrerequisite` — `string missionUID`
  - `MissionAcceptedPrerequisite` — `string missionUID`
  - `MissionNotOfferedPrerequisite` — `string missionUID`
  - `PrerequisiteGroup` — `GroupLogic logic`, `List<Prerequisite> children`
- Instance-count concrete types (spec §6.2):
  - `MaxCompletedPrerequisite` — `int max`
  - `MaxFailedPrerequisite` — `int max`
  - `MaxConcurrentPrerequisite` — `int max`
  - `MaxRejectedPrerequisite` — `int max`
- All properties use camelCase.
- All properties that are serializable have XML atributes for XML deserialization

**Definition of Done:**
- All types compile. Each maps to a row/element in the spec prerequisite tables.

---

### M1-T04 — `Objective` model classes

**Scope:** Model

**Requirements:**
- Create `Models/Objectives/` folder.
- Abstract base class `Objective` with:
  - `string uid`
  - `bool optional` (default `false`)
  - `bool hidden` (default `false`)
- Concrete leaf subclasses (one per type in spec §6.3):
  - `HasPartObjective` — `string part`, `int count`
  - `HasResourceObjective` — `string resource`, `float amount`
  - `HasDeltaVObjective` — `float amount`
  - `LandOnObjective` — `string body`
  - `AchieveOrbitObjective` — `string body`, plus nullable `float` for all apoapsis/periapsis/eccentricity/period/LAN/inclination/argument bounds, plus `string? orbitType`
  - `RecoverVesselObjective` — no extra fields
  - `TimedSurvivalObjective` — `float seconds`
- `ObjectiveGroup` (not a leaf) — `string uid`, `GroupLogic logic` (enum `All`/`Any`), `List<Objective> children`, `bool optional`
- Enum `GroupLogic { All, Any }`
- All properties use camelCase.
- All properties that are serializable have XML atributes for XML deserialization

**Definition of Done:**
- All classes compile without warnings.
- Each concrete type maps 1:1 to a row in the spec §6.3 type table.

---

### M1-T05 — `MissionAction` model classes

**Scope:** Model

**Requirements:**
- Create `Models/Actions/` folder.
- `MissionAction` class:
  - `string uid`
  - `ActionTrigger trigger` (enum)
  - `ActionType type` (enum)
  - `string? message`
  - `string? objectiveUID`
- Enum `ActionTrigger` — values matching all trigger strings in spec §6.4 (e.g. `OnMissionOffer`, `OnMissionAccept`, … `OnObjectiveComplete`, etc.)
- Enum `ActionType { ShowMessage, ShowBlockingPopup }`
- All properties use camelCase.
- All properties that are serializable have XML atributes for XML deserialization

**Definition of Done:**
- All enums and class compile. Every trigger/type in spec §6.4 has an enum member.

---

### M1-T06a — `ObjectiveInstance` model class

**Scope:** Model

**Requirements:**
- Create `Models/ObjectiveInstance.cs`.
- Properties:
  - `string uid` — UID string, generated on creation.
  - `string missionUID` — references a loaded `MissionBlueprint`.
  - `string objectiveUID` — references a loaded `Objective`.
  - `ObjectiveState state` — enum `{ FAILED, NOT_STARTED, TRACKED, MAINTAINED, ACHIEVED }`.
- Constructor accepting a `Objective` that initialises state to `NOT_STARTED` and populates child objectives by recursive walk for `ObjectiveGroup` type.
- All properties use camelCase.

---

### M1-T06b — `MissionInstance` model class

**Scope:** Model

**Requirements:**
- Create `Models/MissionInstance.cs`.
- Properties:
  - `string uid` — UID string, generated on creation.
  - `string blueprintUID` — references a loaded `MissionBlueprint`.
  - `MissionState state` — enum `{ Locked, Offered, Accepted, Completed, Failed, Expired, Rejected }`.
  - `double? offeredAtTimeS` — wall-clock timestamp when offered (for expiration calculation).
  - `double? acceptedAtTimeS` — for deadline calculation.
  - `List<ObjectiveInstance> trackedbjectives` — list of objectives.
- Constructor accepting a `MissionBlueprint` that initialises state to `Locked` and populates `ObjectiveStates` from the blueprint's objectives (recursive walk for groups).
- All properties use camelCase.

**Definition of Done:**
- Class compiles. Constructor correctly seeds `ObjectiveStates` for a blueprint with nested groups.
- Unit test: given a blueprint with 2 leaf objectives and 1 group of 2, constructor produces 4 entries in `ObjectiveStates`.

---

### M1-T07 — `MissionReader` — XML parsing

**Scope:** Model

**Requirements:**
- Create `Models/MissionReader.cs`.
- Static method `MissionBlueprint? LoadFromFile(string filePath, ILogger logger)`.
- Uses `System.Xml.Linq` (`XDocument` / `XElement`) — no third-party XML lib.
- Parsing rules (spec §6.1–6.4):
  - Reads `<MissionBlueprint>` root; returns null + logs error if root missing.
  - Maps all blueprint fields; applies defaults for optional ones.
  - Recursively parses `<prerequisite>` subtree into prerequisite model objects.
  - Recursively parses `<objectives>` subtree (handles `<group>` nesting).
  - Parses `<actions>` list.
  - On any parsing exception, logs the file path + error message and returns null.
- All properties use camelCase.

**Definition of Done:**
- Can parse the sample XML in spec §6.1 (`first_steps.m02_suborbital.xml`) into a correct `MissionBlueprint` object.
- Can parse all example XML snippets from §6.2, §6.3, §6.4 without throwing.
- Malformed XML (non-well-formed, missing required fields) returns null and logs clearly.

---

### M1-T08 — `MissionValidator` — validation rules

**Scope:** Model

**Requirements:**
- Create `Models/MissionValidator.cs`.
- Method `ValidationResult Validate(MissionBlueprint blueprint, IReadOnlyDictionary<string, MissionBlueprint> knownBlueprints)`.
- Returns a `ValidationResult` with `bool IsValid` and `List<string> Errors`.
- Implements all checks from spec §6.5:
  - Required fields not null/empty.
  - String length limits (uid ≤ 128, title ≤ 128, synopsis ≤ 1024, description ≤ 4096, action message ≤ 4096, objective uid ≤ 64, action uid ≤ 64).
  - Unknown objective types — not needed at parse time (type-safe); validate at load that all `AchieveOrbit` bound values are positive when present.
  - `missionUID` in prerequisites resolves to a key in `knownBlueprints`.
  - Instance-count `max` ≥ 1.
  - Action `message` present when `type` is `showMessage` or `showBlockingPopup`.
  - `objectiveUID` present and resolves to a leaf or group uid in the same blueprint when trigger starts with `onObjective`.
  - Objectives list is non-empty.

**Definition of Done:**
- Valid blueprint + known blueprints → `IsValid = true`, no errors.
- Each invalid condition listed in spec §6.5 triggers at least one descriptive error string.
- Unit tests covering every validation rule, each in isolation.

---

### M1-T09 — `MissionsLoader` — discovery & batch load

**Scope:** Model

**Requirements:**
- Create `Models/MissionsLoader.cs`.
- Method `IReadOnlyList<MissionBlueprint> LoadAll(string modsRootPath, ILogger logger)`.
- Scans all `missions/*.xml` under each mod subfolder of `modsRootPath` (spec §6.0).
- Per file: calls `MissionReader.LoadFromFile`; on success calls `MissionValidator.Validate`.
- Skips (and logs) files that fail parse or validation.
- After all files parsed: second-pass validation of cross-blueprint prerequisite `missionUID` references.
- Detects duplicate `uid`s across files — logs error, keeps the first, skips the duplicate.
- Returns only the valid, non-duplicate blueprints.

**Definition of Done:**
- Given a directory with 3 valid and 2 invalid XMLs, returns exactly 3 blueprints.
- Duplicate `uid` logs an error with both file paths.
- No exception propagates to caller regardless of file content.

---

### M1-T10 — `MissionsData` model

**Scope:** Model / Save

**Requirements:**
- Create `Models/MissionsData.cs`.
- Wraps the full save payload:
  - `version`
  - `List<MissionInstance> offeredMissions`
  - `List<MissionInstance> rejectedMissions`
  - `List<MissionInstance> expiredMissions`
  - `List<MissionInstance> acceptedMissions`
  - `List<MissionInstance> failedMissions`
  - `List<MissionInstance> completedMissions`
  - `maxNumberOfOfferedMissions`
  - `maxNumberOfAcceptedMissions`
  - (Optional) `List<Mission> loadedMissions` 
- All properties use camelCase.
- All properties that are serializable have XML atributes for XML deserialization.

- Static helpers `Serialize(List<MissionInstance>) → XElement` and `Deserialize(XElement) → List<MissionInstance>`.
- Serialises/deserialises using `System.Xml.Linq`.

**Definition of Done:**
- Round-trip test: create 2 instances with varied state, serialize to XElement, deserialize back, compare all fields — no data loss.
- Enum strings are human-readable (e.g. `"Accepted"` not `"2"`).

---

### M1-T12 — Harmony save/load hooks

**Scope:** Model / Save

**Requirements:**
- Follow the [KSA modding wiki save guide](https://modding.kittenspaceagency.wiki/en/Code/Save-and-Load-Mod-Data).
  - Create `Patches.UniverseDataPatch.cs`.
  - Create `UncompressedSavePatch.cs`.
  - Postfix patch on the KSA save method: serialize `MissionsData` and write it into the mod save slot.
  - Postfix patch on the KSA load method: read the mod save slot, call `MissionsData.Deserialize`, and reconstruct `MissionInstance` list from records + loaded blueprints.
- Handle missing or null mod savegame data gracefully.
- If a saved `blueprintUID` no longer resolves in the registry (mod removed), log a warning and skip that instance.

**Definition of Done:**
- Start a new game, accept a mission (manually set state in code for testing), save, reload — instance state is fully restored.
- If the mod save slot is absent, the game loads without error.
- Tested with the KSA game running (integration test).

---

### M1-T12 — `IMissionRegistry` interface & implementation

**DRAFT**  
Need to check how interfacing between mods works.

**Scope:** Model / Public API

**Requirements:**
- Create `Api/IMissionRegistry.cs`:
  ```csharp
  public interface IMissionRegistry
  {
      MissionBlueprint? GetBlueprint(string uid);
      IReadOnlyList<MissionBlueprint> GetAllBlueprints();
  }
  ```
- Create `MissionRegistry.cs` implementing the interface backed by the loaded blueprints list from `MissionLoader`.
- The registry is populated once at game-start; no mutation after that.
- Expose a static `MissionRegistry.Instance` (singleton) accessible to other mods.

**Definition of Done:**
- `GetBlueprint("unknown")` returns null without throwing.
- `GetAllBlueprints()` returns a read-only view (callers cannot modify the list).
- Another test assembly can reference `IMissionRegistry` and call both methods.

---

### M1-T13 — Stub `MissionController`

**DRAFT**  
Need to check how interfacing between mods works.

**Scope:** Controller (stub only for M1)

**Requirements:**
- Create `Controllers/MissionController.cs`.
- Holds a `public static MissionsData` in memory (populated from save on load).
- Single public method `RegisterBlueprints(IReadOnlyList<MissionBlueprint> blueprints)` — stores the registry reference.
- No prerequisite evaluation, no offer logic yet — those are M2.
- Expose `IReadOnlyList<MissionInstance> GetInstances()`.

**Definition of Done:**
- `RegisterBlueprints` stores blueprints without throwing.
- `GetInstances()` returns the in-memory list.

---

### M1-T14 — Plugin wiring & startup sequence

**Scope:** Infrastructure

**Requirements:**
- In `GPMissions.OnAllModsLoaded()`:
  - Call `MissionLoader.LoadAll(modsRootPath, logger)` → populate registry.
  - Call `MissionController.RegisterBlueprints(...)`.
- Install Harmony save/load patches. on `OnFullyLoaded()` and `Unload()` hooks.
- Log debug information: "GPMissions: N blueprint(s) loaded".
- `modsRootPath` resolved from the game's mod directory at runtime.

**Definition of Done:**
- Game starts; log shows `"GPMissions: N blueprint(s) loaded"`.
- Placing an invalid XML in `missions/` logs an error but does not crash.
- Placing a valid XML loads the blueprint (and it is retrievable via `IMissionRegistry`).

---

### M1-T15 — Demo mission XMLs (first_steps chain)

**DRAFT**  
Scope the example missions out more

**Scope:** Content / Test data

**Requirements:**
- Create at least 2 valid mission XML files under `missions/`:
  - `first_steps.m01_launch.xml` — no prerequisites; objectives: `HasPart` (engine), `HasResource` (fuel), `TimedSurvival` (30 s).
  - `first_steps.m02_suborbital.xml` — prerequisite: `MissionComplete(m01)`; objectives: `AchieveOrbit` (body=Earth, orbitType=suborbit), `RecoverVessel`.
- Both files must pass `MissionValidator`.

**Definition of Done:**
- XMLs load without errors in-game.
- `IMissionRegistry.GetAllBlueprints()` returns both.
- Validated by `MissionValidator` in a unit test (no game required).

---

## Milestone 2 — Controller & View: Game Loop

**Goal:** Standalone playable missions — prerequisites evaluated, missions offered/accepted, objective HUD displayed, actions fired. A ≥ 3-mission chain is playable in sandbox.

**Entry criteria:** Milestone 1 DoD met (`IMissionRegistry` working, save/load working).

**Exit criteria (milestone DoD):**
- Player can play a ≥ 3-mission chain in sandbox mode.
- Accept/reject and objective HUD work in flight.
- Progress survives mid-play save/load.
- Retry via `<MissionFailed>` prerequisite works.

---

### M2-T01 — Prerequisite evaluator

**Scope:** Controller

**Requirements:**
- Create `Controllers/PrerequisiteEvaluator.cs`.
  - ? Method `bool Evaluate(Prerequisite prerequisite, string missionUid, IReadOnlyList<MissionInstance> allInstances)`.
    - `Prerequisite.Evaluate()`
- Implement all prerequisite types from spec §6.2:
  - `MissionCompletedPrerequisite` — any instance of `missionUID` has `State == Completed`.
  - `MissionFailedPrerequisite` — any instance has `State == Failed`.
  - `MissionAcceptedPrerequisite` — any instance has `State == Accepted` or `Completed`.
  - `MissionNotOfferedPrerequisite` — no instance exists for `missionUID`.
  - `PrerequisiteGroup` — recurse; `All` = all children true, `Any` = at least one true.
  - `MaxCompletedPrerequisite` — count instances with `State == Completed` for `missionUID` < `max`. When `missionUID` is null/self, uses the evaluating mission's own uid.
  - `MaxFailedPrerequisite` — same but `State == Failed`.
  - `MaxConcurrentPrerequisite` — count instances with `State == Offered || Accepted` < `max`.
  - `MaxRejectedPrerequisite` — count instances with `State == Rejected` < `max`.
- Implicit `All` when `<prerequisite>` has multiple children with no wrapping `<group>` (spec §6.2 examples).

**Definition of Done:**
- Unit tests for every prerequisite type using a hand-crafted `List<MissionInstance>`.
- The retry example from spec §6.2 (`MaxFailed max="3"`) allows exactly 3 failures then returns false.
- Group logic (`All`/`Any`) tests with mixed-result children.

---

### M2-T02 — Mission offer logic

**Scope:** Controller

**Requirements:**
  - 
- In `MissionController`, add method `CheckForMissionsToOffer()`:
  - Iterates all loaded blueprints.
  - For each blueprint with no existing `Offered` or `Accepted` instance
  - `MissionBlueprint.CanOffer()`: Loops through it's Prerequisites using `PrerequisiteEvaluator`.
  - If prerequisite passes (or blueprint has no prerequisite), create a new `MissionInstance` with `State = Offered`, add to the instance list.
  - Do not re-offer if an instance already offered or accepted.
  - Check for auto-accepting mission.
- Call `CheckForMissionsToOffer()`:
  - On startup of mod.
  - Once after save/load.
  - After any mission state transition (complete, fail, reject).

**Definition of Done:**
- With m01 having no prerequisite, calling `EvaluateOffers()` on a fresh list creates an `Offered` instance for m01.
- After setting m01's instance to `Completed` and calling `EvaluateOffers()`, m02 (prerequisite: m01 complete) becomes `Offered`.
- Does not create a second `Offered` instance if one already exists.

---

### M2-T03 — Accept / reject / expire lifecycle

**Scope:** Controller

**Requirements:**
- Add to `MissionController`:
  - `bool AcceptMission(string instanceId)` — transitions `Offered → Accepted`; sets `AcceptedAt`; if `IsAutoAccepted` is already handled at offer time; fires `onMissionAccept` action; calls `EvaluateOffers()`.
  - `bool RejectMission(string instanceId)` — transitions `Offered → Rejected` (only if `IsRejectable`); fires `onMissionReject`; calls `EvaluateOffers()`.
  - `void CheckExpirations(float currentGameTime)` — for each `Offered` instance with an `Expiration`, if `currentGameTime - OfferedAt > Expiration`, transition to `Expired`, fire `onMissionExpire`, call `EvaluateOffers()`.
- Return false (do not throw) if the instance is not in the expected state.
- ? Raise C# events: `MissionAccepted`, `MissionRejected`, `MissionExpired` (event args carry `instanceId`).

**Definition of Done:**
- Accept transitions state correctly and fires the event.
- Reject on non-rejectable mission returns false, state unchanged.
- Expiration fires after simulated time advance in a unit test.

---

### M2-T04 — Expanded objective model types for M2

**Scope:** Model

**Requirements:**
- Ensure all types needed for in-flight evaluation exist (most created in M1-T03; verify `AchieveOrbit` bounds are all nullable floats and `OrbitType` is nullable enum).
- Add `ObjectiveEvaluationContext` class passed each tick:
  - `VesselSnapshot activeVessel` (wraps KSA vessel API fields needed for evaluation).
  - `float gameTimeDelta` (seconds since last tick).
- `VesselSnapshot`:
  - `bool isInFlight`
  - `bool isLanded`
  - `string? landedBody`
  - `bool isRecovered`
  - `Dictionary<string, int> partCounts` (part name → count)
  - `Dictionary<string, float> resourceAmounts`
  - `float deltaV`
  - Orbit fields: `float apoapsis`, `float periapsis`, `float eccentricity`, `float period`, `float longitudeOfAscendingNode`, `float inclination`, `float argumentOfPeriapsis`, `string orbitBodyName`, `string orbitType` (`"elliptical"` / `"suborbit"` / `"escape"`)

**Definition of Done:**
- `ObjectiveEvaluationContext` and `VesselSnapshot` compile.
- All fields addressable by the objective evaluator in the next task.

---

### M2-T05 — `MissionEvaluator` — per-tick objective evaluation

**Scope:** Controller

**Requirements:**
- Create `Controllers/MissionEvaluator.cs`.
- Method `void Tick(IReadOnlyList<MissionInstance> instances, ObjectiveEvaluationContext ctx, IReadOnlyDictionary<string, MissionBlueprint> blueprints)`.
- For each `Accepted` instance:
  - Retrieve blueprint from registry.
  - Recursively evaluate each objective:
    - `HasPartObjective` — `ctx.activeVessel.partCounts[part] >= count`.
    - `HasResourceObjective` — `ctx.activeVessel.resourceAmounts[resource] >= amount`.
    - `HasDeltaVObjective` — `ctx.activeVessel.deltaV >= amount`.
    - `LandOnObjective` — `ctx.activeVessel.isLanded && ctx.activeVessel.landedBody == body`.
    - `AchieveOrbitObjective` — `ctx.activeVessel.orbitBodyName == body` and all non-null bounds satisfied; `orbitType` matched if specified.
    - `RecoverVesselObjective` — `ctx.activeVessel.isRecovered`.
    - `TimedSurvivalObjective` — accumulate elapsed ticks while `isInFlight`; complete when accumulated ≥ `seconds`. (Survival time stored in `MissionInstance`, not `objectiveStates`.)
    - `ObjectiveGroup(All)` — complete when all non-optional children complete; fail when any non-optional child fails.
    - `ObjectiveGroup(Any)` — complete when any child completes; fail when all non-optional children fail.
  - Latch transitions: once `Complete` or `Failed`, do not re-evaluate.
  - Raise `ObjectiveChanged` event on each transition.
  - After evaluating all objectives, check mission-level completion:
    - All non-optional root objectives `Complete` → transition instance to `Completed`, fire `onMissionComplete`, call `EvaluateOffers()`.
    - Any non-optional root objective `Failed` → transition to `Failed`, fire `onMissionFail`, call `EvaluateOffers()`.
  - Check deadline: if `AcceptedAt` + `Deadline` < current game time → `Failed`.
- Performance: entire `Tick` for all active missions must complete in ≤ 1 ms (NFR-M01). Use early-exit on latched states.

**Definition of Done:**
- Unit tests for each objective type using a mock `VesselSnapshot`.
- Group `All` / `Any` logic tested with mixed children results.
- Deadline expiry test: set `AcceptedAt` far in the past, tick once → instance becomes `Failed`.
- `TimedSurvival` accumulates across multiple ticks.

---

### M2-T06 — Action runner

**Scope:** Controller

**Requirements:**
- Create `Controllers/ActionRunner.cs`.
- Method `void Run(ActionTrigger trigger, MissionInstance instance, MissionBlueprint blueprint, string? objectiveUID = null)`.
- Iterates blueprint `Actions` in document order.
- For each action matching `trigger` (and `objectiveUID` if trigger is objective-based):
  - If `action.Uid` already in `instance.FiredActionUids` → skip (latched).
  - Execute based on `ActionType`:
    - `ShowMessage` → raise `ShowMessageRequested` event (View will display).
    - `ShowBlockingPopup` → raise `ShowBlockingPopupRequested` event.
  - Add `action.Uid` to `instance.FiredActionUids`.
- `ActionRunner` is called from `MissionController` and `MissionEvaluator` at the correct trigger points.

**Definition of Done:**
- Unit test: action runs once then is latched (second call with same trigger skipped).
- Objective-trigger action only fires when `objectiveUID` matches.
- Both action types raise their respective events.

---

### M2-T07 — Harmony tick hook

**Scope:** Controller / Infrastructure

**Requirements:**
- Create `Controllers/MissionTickPatch.cs`.
- Harmony Postfix on KSA's physics update method (identify correct method from KSA API).
- In the patch: build `VesselSnapshot` from the current active vessel, construct `ObjectiveEvaluationContext`, call `MissionEvaluator.Tick(...)`.
- Also call `MissionController.CheckExpirations(currentGameTime)`.
- Guard: only run when a vessel is in active flight (not in VAB/menu).

**Definition of Done:**
- With a mission `Accepted` and a `TimedSurvival 30s` objective, the objective completes after 30 in-game seconds of flight.
- No tick patch runs in the VAB or main menu (verified by log).

---

### M2-T08 — `IMissionController` public interface

**Scope:** Controller / Public API

**Requirements:**
- Create `Api/IMissionController.cs`:
  ```csharp
  public interface IMissionController
  {
      IReadOnlyList<MissionInstance> GetOfferedMissions();
      IReadOnlyList<MissionInstance> GetAcceptedMissions();
      bool AcceptMission(string instanceId);
      bool RejectMission(string instanceId);
      event EventHandler<MissionStateChangedEventArgs> MissionStateChanged;
      event EventHandler<ObjectiveChangedEventArgs> ObjectiveChanged;
  }
  ```
- `MissionController` implements `IMissionController`.
- Expose `IMissionController.Instance` static accessor.

**Definition of Done:**
- A separate test assembly can use `IMissionController` without referencing internal types.
- Both event types carry the relevant `instanceId` and new state.

---

### M2-T09 — IMGUI Mission Offer Panel

**Scope:** View

**Requirements:**
- Create `Views/MissionOfferPanel.cs`.
- Subscribes to `MissionController.MissionStateChanged` for `Offered` transitions.
- When a mission is offered, displays an IMGUI window with:
  - Mission `Title` (header).
  - `Synopsis` (if present).
  - `Description` (if present, scrollable).
  - "Accept" button → calls `IMissionController.AcceptMission`.
  - "Reject" button (shown only if `IsRejectable = true`) → calls `IMissionController.RejectMission`.
- Panel is dismissible; dismissing without choosing leaves the mission in `Offered` state.
- If multiple missions are offered simultaneously, queue them — show one at a time.
- Opened from a "Missions" button in the KSA toolbar (or equivalent always-visible anchor).

**Definition of Done:**
- With m01 `Offered`, the panel appears in-game showing title, synopsis, and both buttons.
- Clicking Accept calls `AcceptMission` and closes the panel.
- Clicking Reject on a non-rejectable mission is not shown.

---

### M2-T10 — IMGUI Objective HUD

**Scope:** View

**Requirements:**
- Create `Views/ObjectiveHUD.cs`.
- Displays a compact in-flight overlay listing all `Accepted` missions and their objectives.
- Per objective:
  - Show `uid` (or a display label if added later) and state icon (`⬜` pending, `✅` complete, `❌` failed).
  - Hide objectives with `Hidden = true` until they reach `Complete` (Milestone 2+, spec §6.3).
  - `Optional` objectives shown with a `(optional)` suffix.
  - Groups: show group `uid` as a sub-header with its children indented.
- HUD auto-hides when no missions are `Accepted`.
- HUD updates on `ObjectiveChanged` event (no polling).

**Definition of Done:**
- With one mission accepted containing 3 objectives, all appear in the HUD.
- When an objective completes (simulated via direct state set in debug), the icon updates immediately.
- `Hidden` objectives do not appear until complete.

---

### M2-T11 — IMGUI Debrief (toast + blocking popup)

**Scope:** View

**Requirements:**
- Create `Views/MissionDebriefView.cs`.
- Subscribes to `ActionRunner.ShowMessageRequested` and `ActionRunner.ShowBlockingPopupRequested`.
- `ShowMessage` → renders a toast notification (IMGUI label) that auto-dismisses after 5 seconds.
- `ShowBlockingPopup` → renders a modal IMGUI window with the message and an "OK" button; game time paused while open (call KSA pause API).
- Both display the `message` text from the `MissionAction`.

**Definition of Done:**
- Trigger `onMissionComplete` action with type `showMessage` → toast appears for 5 s then disappears.
- Trigger `onMissionFail` action with type `showBlockingPopup` → modal opens, clicking OK dismisses and resumes.

---

### M2-T12 — Demo mission chain (≥ 3 missions)

**Scope:** Content

**Requirements:**
- Extend the M1 demo with at least one more mission:
  - `first_steps.m03_orbit.xml` — prerequisite: m02 complete; objectives: `AchieveOrbit(Earth, elliptical)`, `TimedSurvival(60)`.
- Add `<actions>` to each mission per spec §6.4 examples (at minimum one `onMissionComplete showMessage` per mission).
- Add a retry path: `first_steps.m02b_suborbital_retry.xml` — prerequisite: `MissionFailed(m02_suborbital)`, `MaxFailed max="3"`.

**Definition of Done:**
- Player can complete m01 → m02 → m03 in sandbox mode without a crash.
- Failing m02 causes m02b to be offered (retry).
- All actions fire at correct trigger points.

---

### M2-T13 — Save/load with active missions

**Scope:** Model / Save

**Requirements:**
- Extend `MissionSaveData` to persist `TimedSurvivalObjective` accumulated time per objective per instance.
- Verify `FiredActionUids` are saved and restored (actions don't re-fire after reload).
- Verify `ObjectiveStates` are restored — completed objectives stay complete after reload.
- Integration test: accept mission, complete one objective, mid-mission save, reload, verify objective stays `Complete` and HUD reflects it.

**Definition of Done:**
- All instance state survives a mid-mission save/reload cycle.
- No objective double-fires its `Complete` event after reload.

---

## Milestone 3 — View: Editor & Polish

**Goal:** Mission authors can create and edit missions in-game using an IMGUI editor. Hot-reload works without restart.

**Entry criteria:** Milestone 2 DoD met (full game loop working).

**Exit criteria (milestone DoD):**
- Mission created in editor loads and plays without hand-edits.
- Editor XML passes `MissionReader` validation.
- Hot-reload works without restart.

---

### M3-T01 — `MissionWriter` — XML serialization

**Scope:** Model

**Requirements:**
- Create `Loaders/MissionWriter.cs`.
- Method `XDocument Write(MissionBlueprint blueprint)`.
- Produces XML matching the schema used by `MissionReader` — round-trip safe (write then read returns equal object).
- Handles all prerequisite, objective, and action subtypes.
- Indents output for human readability (2-space indent).
- Saves to `missions/<uid>.xml` under the configured mod root via `Save(MissionBlueprint blueprint, string outputDir)`.

**Definition of Done:**
- Round-trip test: load `first_steps.m02_suborbital.xml`, write to temp file, reload → all fields equal.
- Saved file is valid UTF-8 XML with `<?xml version="1.0" encoding="utf-8"?>` header.

---

### M3-T02 — Hot-reload (`F9`)

**Scope:** Controller / Infrastructure

**Requirements:**
- In debug builds only (conditional compile `#if DEBUG`), register a key listener for `F9`.
- On `F9`:
  1. Call `MissionLoader.LoadAll(...)` again.
  2. Diff new blueprints against existing registry; update registry with new/changed ones.
  3. Do **not** drop existing `MissionInstance`s whose blueprint uid still exists.
  4. Log `"Hot-reload: N blueprint(s) reloaded"`.
- In release builds, `F9` does nothing.

**Definition of Done:**
- In debug: modify a mission XML title, press F9 in-game, `GetBlueprint(uid).Title` returns the new title.
- Existing accepted mission instance survives hot-reload with state intact.

---

### M3-T03 — IMGUI Mission Editor — metadata panel

**Scope:** View

**Requirements:**
- Create `Views/Editor/MissionEditorWindow.cs` — top-level IMGUI editor window.
- Left panel: list of all loaded blueprints (title + uid); "New Mission" button.
- Main panel when a mission selected: editable fields for all `MissionBlueprint` metadata (uid, title, synopsis, description, expiration, isRejectable, deadline, isAutoAccepted).
- Inline validation: red text under fields that violate length limits or are empty when required.
- "Save" button — calls `MissionWriter.Save(blueprint, outputDir)`; then hot-reloads that file.
- "Play-test" button — directly sets selected blueprint to `Offered` in the live `MissionController` (bypasses prerequisites for testing).

**Definition of Done:**
- Editor opens from a toolbar button (or debug menu).
- Editing and saving `title` of m01 persists to disk and hot-reloads correctly.
- Play-test button causes the mission offer panel to appear for the selected blueprint.

---

### M3-T04 — IMGUI Mission Editor — objectives panel

**Scope:** View

**Requirements:**
- Sub-panel within `MissionEditorWindow` for editing the `<objectives>` tree.
- Tree view showing groups and leaf objectives with drag-to-reorder (or Up/Down buttons as fallback).
- "Add Objective" dropdown — lists all `type` values from spec §6.3.
- Per objective: editable `uid`, `optional`, `hidden` checkboxes; type-specific fields (e.g. `part`, `body`, `amount`).
- "Add Group" button — creates a new `ObjectiveGroup` with `logic` dropdown (`All`/`Any`).
- Delete button per node (with confirmation).
- Changes are reflected immediately in the in-memory blueprint; committed to disk only on "Save".

**Definition of Done:**
- Add a `HasPart` objective, save, reload → objective appears in-game HUD when mission accepted.
- Group with two children correctly evaluates in M2 game loop when play-tested.

---

### M3-T05 — IMGUI Mission Editor — prerequisites panel

**Scope:** View

**Requirements:**
- Sub-panel for editing `<prerequisite>` tree.
- Tree similar to objectives: add/remove/reorder nodes.
- Prerequisite type selector: all types from spec §6.2.
- For mission-state types: `missionUID` text field with dropdown of known blueprint uids (for discoverability).
- For instance-count types: `max` integer field (min 1).
- For `PrerequisiteGroup`: `logic` dropdown + nested children.

**Definition of Done:**
- Author a new mission with `MissionComplete` prerequisite referencing m02; save; on fresh game play-through, mission is not offered until m02 is complete.

---

### M3-T06 — IMGUI Mission Editor — actions panel

**Scope:** View

**Requirements:**
- Sub-panel for editing `<actions>` list.
- Per action: `uid`, `trigger` dropdown (all triggers from spec §6.4), `type` dropdown (`showMessage`/`showBlockingPopup`), `message` text area (multiline), `objectiveUID` field (shown only for objective triggers).
- Inline validation: message required for showMessage/showBlockingPopup; objectiveUID required for objective triggers.

**Definition of Done:**
- Add a `showMessage` action on `onMissionComplete`, save, play-test → toast fires on completion.

---

### M3-T07 — Validation feedback in editor

**Scope:** View

**Requirements:**
- On every edit (or on demand via "Validate" button), run `MissionValidator.Validate(blueprint, registry)` against the in-memory blueprint.
- Display all errors in a collapsible "Issues" panel at the bottom of the editor.
- "Save" button is disabled (greyed out) if `IsValid = false`.

**Definition of Done:**
- Leave `uid` blank → "Issues" shows the required-field error and Save is disabled.
- Fix the error → Issues clears and Save becomes enabled.

---

### M3-T08 — `docs/mission-xml-guide.md`

**Scope:** Documentation

**Requirements:**
- Write `docs/mission-xml-guide.md` covering:
  - File location and naming convention.
  - Full annotated XML example (complete blueprint with all optional fields, multiple objective types, prerequisites, actions).
  - Field reference table (mirrors spec §6.1 but in user-facing language).
  - Objective type reference with required/optional attribute examples.
  - Prerequisite type reference with examples for each type.
  - Action trigger and type reference.
  - Common patterns: retry on fail, fork/join campaign paths, timed objectives.
  - Validation error reference (what each error means, how to fix).
  - Add examples.
  - Add best practices.

**Definition of Done:**
- Document is self-contained — a modder with no access to the spec can author a valid mission XML using only this guide.
- All XML examples in the guide pass `MissionValidator` in a unit test.

---