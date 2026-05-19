# Gameplay campaign mode — overview

**Project:** Kitten Space Agency — Gameplay Mods

Story-driven gameplay is split into two mods with separate specs and **independent MVC milestone tracks**:

| Mod | Spec | Purpose |
| :--- | :--- | :--- |
| **Missions** | [missions.md](./missions.md) | Mission blueprints, objectives, prerequisites, lifecycle, save — **standalone** in sandbox. |
| **Campaigns** | [campaigns.md](./campaigns.md) | Groups missions into campaigns; scope, selection, progress UI — **depends on Missions Mod**. |

## Dependency

```text
KSA (game)
  └── KSAMissions (missions/*.xml)
        └── KSACampaigns (campaigns/*.xml)   [optional]
```

## Shared conventions

- Loose XML on disk; UTF-8; stable `uid`s.
- Save/load via [KSA modding wiki — Save and Load Mod Data](https://modding.kittenspaceagency.wiki/en/Code/Save-and-Load-Mod-Data).
- **IMGUI** for all UI.
- Failed mission retry via `<MissionFailed missionUID="…"/>` prerequisites.

## Milestones (both mods)

Each mod follows **Model → Controller/View (game loop) → Editor/polish**:

| Milestone | MVC focus | Missions Mod | Campaigns Mod |
| :-------- | :-------- | :----------- | :-------------- |
| **1** | Model | Load/save mission blueprints & instances | Load/save campaigns; link to mission registry |
| **2** | Controller + View | Mission game loop & HUD | Campaign scope, selection, progress UI |
| **3** | View + tooling | Mission editor | Campaign editor + graph view |

Implement **Missions M1 → M2 → M3**, then **Campaigns M1 → M2 → M3**.

---

> **Note:** The original combined specification lived in this file (v1.0). It is superseded by `missions.md` and `campaigns.md` (v2.0).
