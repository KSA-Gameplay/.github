# Campaigns Mod — Specification

**Project:** Kitten Space Agency — Gameplay Mods  
**Document version:** 1.0  
**Status:** Draft 
**Depends on:** [Missions Mod](./missions.md) (`KSAMissions.dll` + `IMissionRegistry`)  
**Related:** Missions Mod provides mission blueprints, lifecycle, and objective evaluation.

---

## 1. Overview

The **Campaigns Mod** packages missions into story playthroughs. A campaign file lists which mission `uid`s belong to the story, defines the start mission, and tracks **campaign-level** progress. All mission gameplay (objectives, prerequisites, offer/accept) is delegated to the Missions Mod; this mod adds **scope**, **campaign state**, and **campaign UI**.

Structured as **Model–View–Controller (MVC)** (independent milestone track from Missions Mod).

| Layer | Responsibility |
| :--- | :--- |
| **Model** | `CampaignBlueprint`, `CampaignInstance`, loader, save data |
| **View** | IMGUI selection, progress list, graph (M3) |
| **Controller** | Active campaign, mission scope filter, start/completion rules |

---

## 2. Goals

- Guided story playthroughs distinct from sandbox.
- Branching via mission prerequisites (authored in mission XML).
- Data-driven campaigns (`campaigns/<name>.xml`).
- Clean dependency on Missions Mod API — no duplicate mission logic.

---

## 3. Non-Goals (v1)

- Defining objectives or mission lifecycle (Missions Mod).
- Standalone missions without campaigns (Missions Mod).
- In-game campaign editor (Milestone 3).
- Procedural campaigns, multiplayer, localisation.

---

## 4. Concepts & Terminology

| Term | Definition |
| :--- | :--- |
| **Campaign blueprint** | `campaigns/<name>.xml` — metadata + mission membership + start mission. |
| **Campaign instance** | Active playthrough: which campaign, which missions are in scope for this story. |
| **Mission scope** | Subset of loaded missions listed in the campaign; only these are offered during play. |
| **Campaign state** | Save data: active campaign `uid`, campaign-level flags (Missions Mod holds mission instances). |

---

## 5. Architecture (MVC)

```text
┌─────────────┐                 ┌────────────────────┐
│    View     │ ◄────────────── │ CampaignController │
│  (IMGUI)    │                 │  (scope, start,    │
└─────────────┘                 │   completion)      │
                                └─────────┬──────────┘
                                          │ uses
                                ┌─────────▼──────────┐
                                │   Missions Mod     │
                                │ IMissionController │
                                │ IMissionRegistry   │
                                └─────────┬──────────┘
                                          │
                                ┌─────────▼──────────┐
                                │      Model         │
                                │ CampaignBlueprint  │
                                │ CampaignInstance   │
                                │ CampaignReader     │
                                │ CampaignSaveData   │
                                │ CampaignWriter     │
                                └────────────────────┘
```

**Integration rules:**

1. On campaign start, `CampaignController` sets **mission scope** to the campaign’s `<missionUID>` list.
2. `MissionController` only offers missions in scope whose prerequisites are satisfied.
3. **Start mission** — on new campaign, offer the mission matching `<startMissionUID>` (no prerequisite or empty).
4. **Campaign complete** — no scoped mission is offered/accepted and no scoped locked mission can still unlock (FR-C09).

---

## 6. Data specifications

### 6.0 File layout

| Rule | Detail |
| :--- | :--- |
| **Path** | `campaigns/<name>.xml` — one `<CampaignBlueprint>` per file. |
| **Missions** | Defined only in Missions Mod (`missions/<name>.xml`). |
| **Linking** | Campaign lists `<missionUID>`; loader verifies each exists in `IMissionRegistry`. |
| **No back-link** | Mission files do **not** contain `campaignBlueprintUID` (D-C01). |

**Mod layout:**

```text
Campaigns/
  Campaigns.dll
  campaigns/
    first_steps.xml
```

Mission XML lives in **KSAMissions** (or any mod providing missions).

### 6.1 Campaign blueprint — `CampaignBlueprint`

| Class field | XML element | Required | Constraints / notes |
| :---------- | :---------- | :------- | :------------------ |
| `version` | `<version>` | Yes | Mod semantic version. |
| `uid` | `<uid>` | Yes | Max 128 chars; globally unique. |
| `title` | `<title>` | Yes | Max 128 chars. |
| `synopsis` | `<synopsis>` | No | Max 1024 chars. |
| `description` | `<description>` | No | Max 4096 chars. |
| `startMissionUID` | `<startMissionUID>` | Yes | Must appear in `<missions>`. |
| `missions` | `<missions>` | Yes | Wrapper for `<missionUID>` entries. |
| *(membership)* | `<missionUID>` | Yes | `uid` of a loaded mission blueprint. |

**Example — `campaigns/first_steps.xml`:**

```xml
<?xml version="1.0" encoding="utf-8"?>
<CampaignBlueprint>
  <version>1.0.0</version>
  <uid>ksa.demo.first_steps</uid>
  <title>First Steps</title>
  <synopsis>Learn to fly, orbit, and land.</synopsis>
  <description>A guided introduction to Kitten Space Agency.</description>
  <startMissionUID>first_steps.m01_briefing</startMissionUID>
  <missions>
    <missionUID>first_steps.m01_launch</missionUID>
    <missionUID>first_steps.m02_suborbital</missionUID>
    <missionUID>first_steps.m03_luna</missionUID>
    <missionUID>first_steps.m04a_lab</missionUID>
    <missionUID>first_steps.m04b_launchpad</missionUID>
  </missions>
</CampaignBlueprint>
```

**Prerequisite scope:** `<MissionComplete missionUID="…"/>` in mission XML should reference `uid`s within the same campaign’s `<missions>` list (validated as warning/error at campaign load).

### 6.2 Validation

| Check | Severity |
| :---- | :------- |
| Well-formed XML | Error |
| Required fields | Error |
| Duplicate campaign `uid` | Error |
| Each `<missionUID>` exists in `IMissionRegistry` | Error |
| `startMissionUID` listed in `<missions>` | Error |
| Prerequisite `missionUID`s in scoped missions reference only listed missions | Warning / Error |
| Graph reachable from `startMissionUID` | Warning |
| Orphan `<missionUID>` (unreachable) | Warning |

---

## 7. Save integration

Campaign instance data (active campaign `uid`, campaign-level progress) saves via the same KSA Harmony hooks as Missions Mod (§9). Mission instances remain in Missions Mod save payload.

---

## 8. Requirements

**FR-C01** Load `campaigns/<name>.xml`; resolve missions via Missions Mod.  
**FR-C02** “Campaigns” game mode in startup config;
**FR-C03** Create a new or continue campaign on startup.
**FR-C04** Configure on creation which campaign(s) are enabled. (Can be edited later)  
**FR-C05** Campaign progress UI (list / graph).  
**FR-C06** Delegate offer/accept/objectives to Missions Mod within scope.  
**FR-C07** Campaign completion when no further scoped missions can unlock.  
**FR-C08** Validate campaign XML on load.  
**FR-C09** Persist campaign state with game save.

**NFR-C01–C06** Same as Missions Mod (performance, hot-reload campaigns in debug, Harmony, stable ids, naming, IMGUI).

---

## 9. Milestones (MVC)

*Implement after Missions Mod Milestone 1 is complete (registry + save API available).*

### Milestone 1 — Model: data, load, save

**Goal:** Campaign blueprints load, link to missions, campaign state saves/loads. No campaign game loop UI.

| Layer | Deliverables |
| :---- | :----------- |
| **Model** | `CampaignBlueprint`, `CampaignInstance`, `CampaignReader`, validation (§6.2), `CampaignSaveData` + save hooks. |
| **Controller** | `CampaignController` stub — register campaigns, set active campaign id. |
| **View** | None required (optional debug list of campaigns). |

**Definition of Done:**

- Campaign XML loads; broken `missionUID` references error clearly.
- Active campaign `uid` survives save/load.
- Requires Missions Mod M1 (`IMissionRegistry`).

---

### Milestone 2 — Controller & View: game loop

**Goal:** Full “First Steps” campaign playable start-to-finish using Missions Mod M2 loop within campaign scope.

| Layer | Deliverables |
| :---- | :----------- |
| **Model** | — |
| **Controller** | Scope filter on `MissionController`; start mission on new campaign; campaign completion detection; integrate save with mission state. |
| **View** | IMGUI campaign selection (new / continue); campaign progress overview (mission states); campaign completion summary. |

**Definition of Done:**

- Player selects campaign, plays ≥ 6 missions start-to-finish.
- Branching (two endings) via mission prerequisites.
- Campaign + mission progress survive save/load.
- Requires Missions Mod M2.

---

### Milestone 3 — View: editor, graph & polish

**Goal:** Author campaigns in-game; visualize structure and progress as a graph.

| Layer | Deliverables |
| :---- | :----------- |
| **Model** | Campaign XML writer; cross-validate with mission registry. |
| **Controller** | Play-test campaign from selected mission; hot-reload `campaigns/`. |
| **View** | IMGUI campaign editor; **graph view** (nodes = missions, edges = `<MissionComplete>` from mission files); live progress overlay (completed / offered / locked). |

**Definition of Done:**

- Campaign created in editor plays to completion.
- Graph matches prerequisites and live progress.
- Round-trip XML loads in `CampaignReader` unchanged semantically.
- Requires Missions Mod M3 for mission editing (or hand-authored mission XML).

---

## 10. Design decisions

| # | Decision |
| :--- | :--- |
| D-C01 | Missions are **campaign-agnostic** — membership only in campaign XML. |
| D-C02 | **Missions Mod first** — each mod has its own M1→M2→M3 track. |
| D-C03 | Same save/wiki/IMGUI/retry decisions as Missions Mod (§10 missions.md). |
| D-C04 | Demo content split: mission XML in Missions; `first_steps.xml` in Campaigns. |

---

## 11. Out of scope

- Objective types, actions, mission editor (Missions Mod).
- Multiplayer, procedural generation, localisation.

## 12. Future considerations

- Multiple campaigns; How to continue with a gamesave with another campaign.
- Campaigns can have prerequisites too?
