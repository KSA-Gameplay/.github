# Missions Mod — Specification

**Project:** Kitten Space Agency — Gameplay Mods  
**Document version:** 1.0  
**Status:** Draft  
**Depends on:** KSA base game

---

## 1. Overview

The **Missions Mod** adds reusable mission contracts to Kitten Space Agency: objectives, prerequisites, lifecycle (offer → accept → complete/fail), and optional actions. Missions XML is stored in `missions/<name>.xml` and can run in sandbox or on other gameplay modes.

The mod is structured as **Model–View–Controller (MVC)**:

| Layer | Responsibility |
| :--- | :--- |
| **Model** | Blueprints, runtime instances, XML load/validation, save data |
| **View** | IMGUI panels (offer, objectives, editors) |
| **Controller** | Mission loop: prerequisites, evaluation, state transitions, actions |

---

## 2. Goals

- Provide a standalone mission system to provide enhanced gameplay experience in the form of missions the player can embark upon.
- Keep missions fully data-driven (XML, no code changes for new missions).
- Expose a stable API for other mods.
- Persist mission instance state via the KSA save system.

---

## 3. Non-Goals (v1)

- Mission dependency graphs or story packaging.
- In-game mission editor (Milestone 3).
- Procedural missions (future).
- Localisation beyond English (future).

---

## 4. Concepts & Terminology

| Term | Definition |
| :--- | :--- |
| **Mission blueprint** | Static definition loaded from `missions/<name>.xml`. |
| **Mission instance** | Runtime state for one play-through of a blueprint (offered, accepted, etc.). |
| **Objective** | Measurable goal tied to the mission and an active vessel. |
| **Prerequisite** | Conditions that must be true before a mission is offered. |
| **Mission state** | Per-save record of instances and objective progress. |

---

## 5. Architecture (MVC)

```text
┌─────────────┐     events      ┌──────────────────┐
│    View     │ ◄────────────── │    Controller    │
│  (IMGUI)    │ ──────────────► │ MissionController│
└─────────────┘   user input    │ MissionEvaluator │
                                └────────┬─────────┘
                                         │ read/write
                                ┌────────▼─────────┐
                                │      Model       │
                                │ MissionBlueprint │
                                │ MissionInstance  │
                                │ MissionReader    │
                                │ MissionSaveData  │
                                │ MissionWriter    │
                                └──────────────────┘
```

**Public API:**

See open question (§12, OQ-01) on how other mods can access the data available in this mod.

- `IMissionRegistry` — lookup blueprint by `uid`, list all loaded missions.
- `IMissionController` — offer / accept / reject; query instance state.
- Events — `MissionOffered`, `MissionCompleted`, `MissionFailed`, `ObjectiveChanged`, etc.

---

## 6. Data specifications

### 6.0 File layout and discovery

| Rule | Detail |
| :--- | :--- |
| **Path** | `missions/<name>.xml` under each mod root (one `<MissionBlueprint>` per file). |
| **Packaging** | Loose XML on disk, not embedded in `.dll` (§10). |
| **Discovery** | All `missions/*.xml` scanned on game start. |
| **UID** | Globally unique across mods; stable across releases (NFR-04). |
| **Filename** | `<uid>` should match filename (recommended). |

**Mod layout:**

```text
Missions/
  Missions.dll
  missions/
    first_steps.m01_launch.xml
    first_steps.m02_suborbital.xml
```

### 6.1 Mission blueprint — `MissionBlueprint`

| Class field | XML element | Required | Default | Constraints / notes |
| :---------- | :---------- | :------- | :------ | :------------------ |
| `uid` | `<uid>` | Yes | — | Max 128 chars. |
| `title` | `<title>` | Yes | — | Max 128 chars. |
| `synopsis` | `<synopsis>` | No | — | Max 1024 chars. |
| `description` | `<description>` | No | — | Max 4096 chars. |
| `expiration` | `<expiration>` | No | none | Seconds while **offered**. |
| `isRejectable` | `<isRejectable>` | No | `true` | |
| `deadline` | `<deadline>` | No | none | Seconds after **accept**. |
| `isAutoAccepted` | `<isAutoAccepted>` | No | `false` | |
| `prerequisite` | `<prerequisite>` | No | — | §6.2 |
| `objectives` | `<objectives>` | Yes | — | §6.3 |
| `actions` | `<actions>` | No | — | §6.4 |

**Lifecycle:**

```text
[locked] ──prerequisite met──► [offered] ──accept──► [accepted] ──objectives──► [completed]
                                  │                      │
                                  └──reject──────────────┴──► [failed] / [expired]
```

**Example — `missions/first_steps.m02_suborbital.xml`:**

```xml
<?xml version="1.0" encoding="utf-8"?>
<MissionBlueprint>
  <uid>first_steps.m02_suborbital</uid>
  <title>Suborbital Hop</title>
  <synopsis>Reach space and return safely.</synopsis>
  <description>Build a small rocket and complete a suborbital flight.</description>
  <prerequisite>
    <MissionComplete missionUID="first_steps.m01_launch" />
  </prerequisite>
  <objectives>
    <objective uid="reach_space" type="AchieveOrbit" body="Earth" />
    <objective uid="recover" type="RecoverVessel" />
  </objectives>
</MissionBlueprint>
```

### 6.2 Prerequisites — `Prerequisite`

Pull-based offering: when prerequisites become true, `MissionController` offers the mission.

**Mission-state prerequisites:**

| XML element | Attributes | True when |
| :---------- | :--------- | :-------- |
| `<MissionCompleted>` | `missionUID` | Referenced mission completed at least once this save/session. |
| `<MissionFailed>` | `missionUID` | Referenced mission failed at least once. |
| `<MissionAccepted>` | `missionUID` | Referenced mission accepted or was accepted. |
| `<MissionNotOffered>` | `missionUID` | Referenced mission never offered. |
| `<group>` | `logic` (`All` \| `Any`) | Nested prerequisites. |

**Instance-count prerequisites** — gate offers by how many **instances** of a **specific** mission blueprint exist in a given state. `missionUID` must resolve to a loaded blueprint (§6.5). `max` is a positive integer (≥ 1). Counts are per save/session and re-evaluated on instance state changes.

The evaluating mission’s not-yet-created instance is **not** counted unless an instance already exists in the counted state.

| XML element | Attributes | True when |
| :---------- | :--------- | :-------- |
| `<MaxCompleted>` | `max` | Only offer this mission if it has been **Completed** &lt; `max` (cumulative). `max="1"` = at most one successfully completed mission. |
| `<MaxFailed>` | `max` | Only offer this mission if it has been **Failed** &lt; `max` (cumulative). Limits retries (e.g. `max="3"` allows three failures before a fourth offer (retry) is blocked). |
| `<MaxConcurrent>` | `max` | **Concurrent** instances in **offered** or **accepted** state &lt; `max`. `max="1"` = only one active mission at a time. |
| `<MaxRejected>` | `max` | Only offer this mission if it has been **Rejected** &lt; `max`. `max="1"` = mission won't be offered again after it was rejected. |

**Counting rules:**

- Each offer → complete/fail cycle is one instance; cumulative totals do not include rejected instances.
- **Concurrent** counts only **offered** + **accepted** for the given `missionUID` (failed runs are not active parallels).
- Reference another mission’s `missionUID` to gate this offer, or the same `uid` to cap retries on this contract.


**Retry:** `<MissionFailed missionUID="…"/>` in an OR group with the normal parent `<MissionComplete>` (§10, D-M02).

**Scope:** All loaded missions participate in prerequisite resolution. Other mods can change (using harmony) the method to check for offered missions (TODO: add the function name here.).

**Examples:**

```xml
<!-- Fork path -->
<prerequisite>
  <group logic="Any">
    <MissionComplete missionUID="first_steps.m03a_science" />
    <MissionComplete missionUID="first_steps.m03b_commercial" />
  </group>
</prerequisite>

<!-- Join path -->
<prerequisite>
  <group logic="All">
    <MissionComplete missionUID="first_steps.m03a_science" />
    <MissionComplete missionUID="first_steps.m03b_commercial" />
  </group>
</prerequisite>

<!-- Retry mission after failure a maximum of 3 times (don't have te reference self for failed) -->
<prerequisite>
  <MaxFailed max="3" />
</prerequisite>

<!-- Offer recovery mission m02b after failed m02 mission -->
<prerequisite>
  <MissionFailed missionUID="first_steps.m02_suborbital" />
</prerequisite>

<!-- Offer m03 only after m02 completed, with at most 1 completed and concurrent, and ≤ 3 failures on m03 -->
<prerequisite>
  <MissionComplete missionUID="first_steps.m02_suborbital" />
  <MaxConcurrent max="1" />
  <MaxCompleted max="1" />
  <MaxFailed max="3" />
</prerequisite>

```

### 6.3 Objectives — `Objective`

Leaf goals: `<objective type="…">`. Groups: `<group logic="All|Any">` wrapping children.

| Attribute | Required | Description |
| :-------- | :------- | :---------- |
| `uid` | Yes | Max 64 chars; HUD and action references. |
| `type` | Yes | See table below. |
| `optional` | No | Default `false`. |
| `hidden` | No | Default `false` (Milestone 2+). |

| Type | Required attributes | Optional attributes | Description |
| :--- | :------------------ | :------------------ | :---------- |
| `<group>` | `uid` | `logic` (`All` \| `Any`) | Container; `All` = every child complete (default). |
| `HasPart` | `part` | `count` (default `1`) | Active vessel has part(s). |
| `HasResource` | `resource`, `amount` | — | Vessel holds ≥ `amount` of `resource`. |
| `HasDeltaV` | `amount` | — | The vessel has atleast capability to generate `amount` of impulse (change in velocity) . |
| `LandOn` | `body` | — | Lands on celestial `body`. |
| `AchieveOrbit` | `body` | `minApoapsis`<br>`maxApoapsis`<br>`minPeriapsis`<br>`maxPeriapsis`<br>`minEccentricity`<br>`maxEccentricity`<br>`minPeriod`<br>`maxPeriod`<br>`minLongitudeOfAscendingNode`<br>`maxLongitudeOfAscendingNode`<br>`minInclination`<br>`maxInclination`<br>`minArgumentOfPeriapsis`<br>`maxArgumentOfPeriapsis`<br>`orbitType` | Orbit (Milestone 2).<br> `orbitType` one of: `elliptical`, `suborbit`, `escape` |
| `RecoverVessel` | — | — | Recovered after landing. (future) |
| `TimedSurvival` | `seconds` | — | Intact for `seconds` game time, often to be used in combination with other objectives. |

**Evaluation:**

- Active flight vessel, each physics tick.
- Complete/failed transitions latched.
- Mission completes when all non-optional root objectives complete; fails on non-optional failure or `deadline`.

**Examples by type:**

```xml
<!-- HasPart — single engine -->
<objective uid="has_engine" type="HasPart" part="LiquidEngine" />

<!-- HasPart — count of two decouplers -->
<objective uid="has_stages" type="HasPart" part="Decoupler" count="2" />

<!-- HasResource — at least 200 units of liquid fuel -->
<objective uid="has_fuel" type="HasResource" resource="LiquidFuel" amount="200" />

<!-- HasDeltaV — at least 9400m/s as delta-v to reach Low Earth Orbit -->
<objective uid="has_dv_for_LEO" type="HasDeltaV" amount="9400" />

<!-- LandOn — touchdown on the Luna -->
<objective uid="land_luna" type="LandOn" body="Luna" />

<!-- AchieveOrbit — basic stable orbit -->
<objective uid="orbit_Earth" type="AchieveOrbit" body="Earth" orbitType="elliptical" />

<!-- AchieveOrbit — with apoapsis / periapsis bounds (Milestone 2) -->
<objective uid="orbit_precise" type="AchieveOrbit" body="Earth" minApoapsis="80000" maxApoapsis="90000" minPeriapsis="75000" maxPeriapsis="85000" />

<!-- TimedSurvival — stay intact for 120 seconds -->
<objective uid="survive_launch" type="TimedSurvival" seconds="120" />

<!-- Optional objective — failure does not fail the mission -->
<objective uid="bonus_science" type="HasPart" part="ScienceExperiment" optional="true" />

<!-- Hidden until progress (Milestone 2+) -->
<objective uid="secret_orbit" type="AchieveOrbit" body="Earth" hidden="true" />
```

**Example — `<group>` with `All` and `Any`:**

```xml
<objectives>
  <!-- Prep: engine AND fuel required -->
  <group uid="prep" logic="All">
    <objective uid="engine" type="HasPart" part="LiquidEngine" />
    <objective uid="fuel" type="HasResource" resource="LiquidFuel" amount="100" />
  </group>

  <!-- Bonus: ANY one science part satisfies the group -->
  <group uid="bonus_instruments" logic="Any" optional="true">
    <objective uid="barometer" type="HasPart" part="Barometer" />
    <objective uid="thermometer" type="HasPart" part="Thermometer" />
  </group>
</objectives>
```

**Example — all objective types in one mission:**

```xml
<objectives>
  <!-- Launch prep -->
  <group uid="launch_prep" logic="All">
    <objective uid="engine" type="HasPart" part="LiquidEngine" count="1" />
    <objective uid="fuel" type="HasResource" resource="LiquidFuel" amount="150" />
    <objective uid="clear_tower" type="TimedSurvival" seconds="30" />
  </group>

  <!-- Flight -->
  <objective uid="orbit" type="AchieveOrbit" body="Earth" minApoapsis="160000" minPeriapsis="160000" />
  <objective uid="transfer" type="AchieveOrbit" body="Luna" />

  <!-- Land on Luna & return -->
  <group uid="surface_ops" logic="All">
    <objective uid="land" type="LandOn" body="Luna" />
    <objective uid="land" type="LandOn" body="Earth" />
  </group>

  <!-- Optional: bring extra tank (failure ignored) -->
  <objective uid="extra_tank" type="HasPart" part="FuelTank" optional="true" />
</objectives>
```

### 6.4 Actions — `Action`

Optional `<actions>` on a mission blueprint. Each child `<action>` runs a side effect when its `trigger` fires. Actions do not change prerequisites, objective evaluation, or mission completion.

**Container:** `<actions>` holds one or more `<action>` elements.

**`<action>` attributes:**

| Attribute | Required | Description |
| :-------- | :------- | :---------- |
| `uid` | Yes | Stable id (max 64 chars); unique among actions on the same mission. Latched after first successful run per save. |
| `trigger` | Yes | Lifecycle or objective event; see [triggers](#action-triggers) below. |
| `type` | Yes | Effect to run; see [action types](#action-types) below. |
| `message` | Conditional | Required when `type` is `showMessage` or `showBlockingPopup`. Player-facing text (max 4096 chars). |
| `objectiveUID` | Conditional | Required when `trigger` starts with `onObjective`. Must match a leaf `<objective>` or `<group>` `uid` on this mission. |

**Execution rules:**

- Each `uid` runs at most **once per save** (latched after success).
- Multiple `<action>` elements with the same `trigger` run in **document order**.
- Objective triggers apply to the **active mission instance** only.

#### Action triggers {#action-triggers}

| `trigger` value | Required attributes | Description |
| :-------------- | :------------------ | :---------- |
| `onMissionOffer` | `uid`, `trigger`, `type` | Mission instance enters **offered**. |
| `onMissionAccept` | `uid`, `trigger`, `type` | Player accepts (or `isAutoAccepted`). |
| `onMissionExpire` | `uid`, `trigger`, `type` | Offer expires (`expiration` elapsed). |
| `onMissionReject` | `uid`, `trigger`, `type` | Player rejects offer or aborts accepted mission (if allowed). |
| `onMissionComplete` | `uid`, `trigger`, `type` | Mission instance reaches **completed**. |
| `onMissionFail` | `uid`, `trigger`, `type` | Mission instance reaches **failed**. |
| `onObjectiveStarted` | + `objectiveUID` | Referenced objective becomes active for tracking. |
| `onObjectiveMaintained` | + `objectiveUID` | Objective met but must stay met (Milestone 2+). |
| `onObjectiveReverted` | + `objectiveUID` | Maintained objective no longer satisfied. |
| `onObjectiveComplete` | + `objectiveUID` | Referenced objective transitions to **complete**. |
| `onObjectiveFailed` | + `objectiveUID` | Referenced objective transitions to **failed**. |

#### Action types {#action-types}

| `type` value | Required attributes | Description |
| :----------- | :------------------ | :---------- |
| `showMessage` | + `message` | Non-blocking notification (toast / log); gameplay continues. |
| `showBlockingPopup` | + `message` | Modal dialog; game paused until dismissed. |

**Planned (not in v1):** `spawnVessel`, `giveReward`, `offerMission` — unsupported `type` values are validation errors until implemented.

**Examples:**

```xml
<!-- Mission complete toast -->
<actions>
  <action>
    <uid>m02_complete_toast</uid>
    <trigger>onMissionComplete</trigger>
    <type>showMessage</type>
    <message>Suborbital flight logged. Mission control is impressed with your save return.</message>
  </action>
</actions>

<!-- Objective failure blocks with a popup -->
<actions>
  <action>
    <uid>orbit_fail_popup</uid>
    <trigger>onObjectiveFailed</trigger>
    <objectiveUID>orbit_kerbin</objectiveUID>
    <type>showBlockingPopup</type>
    <message>Orbit not achieved. Check staging and delta-v to be at least 9400m/s.</message>
  </action>
</actions>
```

### 6.5 Validation

| Check | Severity |
| :---- | :------- |
| Well-formed XML | Error — file skipped |
| Required fields | Error |
| Duplicate mission `uid` | Error |
| Unknown objective `type` / prerequisite / action | Error |
| `missionUID` in prerequisite resolves to loaded blueprint | Error |
| Instance-count prerequisite `max`, invalid `max` | Error |
| Action payload complete | Error |
| String length limits | Error |

---

## 7. Save integration

Mission instance state serializes to XML during KSA save via Harmony Prefix/Postfix ([wiki guide](https://modding.kittenspaceagency.wiki/en/Code/Save-and-Load-Mod-Data), §10).

---

## 8. Requirements

**FR-M01** Discover and load all `missions/<name>.xml` on startup.  
**FR-M02** Validate XML; log errors; never crash on bad files.  
**FR-M03** Persist and restore mission instances with game save.  
**FR-M04** Per-tick objective evaluation.  
**FR-M05** Mission complete/fail when objective rules met.  
**FR-M06** Offer / accept / reject lifecycle with optional expiration and deadline.  
**FR-M07** Multiple simultaneous accepted missions.  
**FR-M08** Execute actions on triggers (latched once per save per action `uid`).  
**FR-M09** Instance-count prerequisites (§6.3) gate offers by per-`missionUID` completed, failed, and concurrent instance limits.

**NFR-M01** Objective loop ≤ 1 ms/frame.  
**NFR-M02** Hot-reload `missions/` in debug (Milestone 3).  
**NFR-M03** Harmony Prefix/Postfix only.  
**NFR-M04** Stable `uid`s.  
**NFR-M05** PascalCase methods; camelCase locals.  
**NFR-M06** IMGUI for all UI (§10).

---

## 9. Milestones (MVC)

### Milestone 1 — Model: data, load, save

**Goal:** Mission blueprints load, validate, and instance state round-trips through save/load. No gameplay loop or player UI.

| Layer | Deliverables |
| :---- | :----------- |
| **Model** | `MissionBlueprint`, `MissionInstance`, `Objective` models; `MissionReader`; validation (§6.5); `MissionSaveData` + Harmony save hooks. |
| **Controller** | Stub `MissionController` (register blueprints only). |
| **View** | None (debug log of loaded mission count optional). |

**Definition of Done:**

- Valid mission XML loads; invalid XML logged and skipped.
- Save/load restores mission instance fields without loss.
- `IMissionRegistry` exposes loaded blueprints to other mods.

---

### Milestone 2 — Controller & View: game loop

**Goal:** Standalone playable missions: prerequisites, evaluation, offer/accept UI, objective HUD, actions.

| Layer | Deliverables |
| :---- | :----------- |
| **Model** | Expanded objective types (`LandOn`, `AchieveOrbit`, `RecoverVessel`, `TimedSurvival`, groups). |
| **Controller** | `MissionController` — prerequisite checks, offer/accept/reject/expire/fail; `MissionEvaluator` per tick; action runner; parallel missions. |
| **View** | IMGUI mission offer/briefing; objective HUD; debrief toasts/popups. |

**Definition of Done:**

- Player can play a bundled ≥ 3-mission chain in sandbox mode.
- Accept/reject and objective HUD work in flight.
- Progress survives mid-play save/load.
- Retry via `<MissionFailed>` prerequisites works.

---

### Milestone 3 — View: editor & polish

**Goal:** Author and polish missions in-game.

| Layer | Deliverables |
| :---- | :----------- |
| **Model** | Mission XML writer; validation in editor context. |
| **Controller** | Play-test from editor; hot-reload (`F9` debug). |
| **View** | IMGUI mission editor (metadata, prerequisites, objectives, actions); `docs/mission-xml-guide.md`. |

**Definition of Done:**

- Mission created in editor loads and plays without hand-edits.
- Editor XML passes `MissionReader` validation.
- Hot-reload works without restart.

---

## 10. Design decisions

| # | Decision |
| :--- | :--- |
| D-M01 | Save via [KSA wiki](https://modding.kittenspaceagency.wiki/en/Code/Save-and-Load-Mod-Data) Harmony hooks. |
| D-M02 | Failed missions retry via `<MissionFailed>` prerequisites. |
| D-M03 | Loose `missions/*.xml` on disk. |
| D-M04 | **IMGUI** for all UI. |
| D-M05 | **Mod agnostic** missions mod is completely agnostic of other mods using this mod. |
| D-M06 | **Standalone** — full mission gameplay without other mods should be possible. |

---

## 11. Out of scope

- Packaging of missions into a more coherent set of missions.
- Multiplayer, procedural missions, voice acting, localisation.

## 12. Open questions

- **OQ-1** How can other mods access or query the available missions with the current Harmony/StarMap APIs.
