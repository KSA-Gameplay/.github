# Project overview

## Identity

The project aims to improve player gameplay experience through mods.

## Mod ideas

- Missions: Standalone mission contracts (objectives, prerequisites, lifecycle). Spec: [missions.md](specs/missions.md).
- Campaigns: Story playthroughs that group missions from the Missions mod. Spec: [campaigns.md](specs/campaigns.md).
- Mission control: An overview of all the missions, space vessels (on a mission), plan missions, send commands etc.
- Communication Satellites: Add capability and simulation of (digital) radio communication through satellites
- Tracking Satellites: Add capability and simulation of tracking the orbit of vessels through ground tracking stations and satellites. Also, hide orbit information from player when it can't be observed.
- Agencies: Add agencies that can compete for missions
- Actors: Add actors which are players or AI driven actors that can control a vessel through a Kitten (astronaut) or the guidance computer. These actors could decide how to control the vehicle to achieve a certain goal (i.e. objective) either from a mission or from commands from mission control.
- Training Center: Add skills, experience and training to gatonauts
- Technology research: Research technology, it unlock more and improved parts.

## Gameplay modes

- single player
  - sandbox (exists already in the game)
  - story mode (to be created)
  - realistic (to be created)
- multiplayer (future)
  - cooperative: players can play within same agency
  - adverserial: players play different agencies and compete against each other.
- player mode
  - 3rd person full control (current mode)
  - mission control; only from mission control with limited information, and can only send commands (with delay).
  - 1st person; control an agent (kitten) directly.
  - mix of mission control and 1st person (switchable)


### Sandbox mode

Anythink is possible mode. No limitations on available resources, parts etc. Likely, no limitations on the gatonauts skills, science etc.

### Story mode

Story mode is a (selection of) campaign(s) that will guide the player through a series of missions to have a more guided and story rich gameplay style.

### Realistic mode

Kitten Space Agency is a simulation game. However currently the whole simulation is fully observable and fully controllable, ofter refered as god-mode. Mods like ComSat, TrackSat etc. would limit the observability. While mods like Mission Control, and Actors would limit the controlability of the simulation.
