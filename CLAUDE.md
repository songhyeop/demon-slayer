# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project Status

Pre-development. Unity 6 project created (6000.3.6f1). No game code yet.

**Read these files before writing any code, in this order:**
1. [plan.md](plan.md) — Game Design Document. Source of truth for all game rules and mechanics.
2. [dev-plan.md](dev-plan.md) — Tech stack decisions, architecture, folder structure, and phased development plan.
3. [build-test-guide.md](build-test-guide.md) — Build automation and test framework guidelines.

**Engine / Language:** Unity 6 (6000.3.6f1) / C#

---

## Architecture (from plan.md)

Demon Slayer is a **map-exploration roguelike** with turn-based combat and a time-pressure mechanic. The core loop is: navigate a node graph → trigger an event → resolve it (combat / shop / rest) → repeat until the boss is defeated or time runs out.

### Core Systems to Implement

| System | Responsibility |
|---|---|
| **GameManager** | Owns the global game state: `currentTurn`, `invasionGauge`, `maxInvasionGauge`, `isGameOver`, `gold`. Single source of truth for win/lose conditions. |
| **Map / Board** | A directed graph of nodes from `Start Node` (village) to `Boss Node` (demon king's castle). Each node has a type that determines what event fires on arrival. |
| **Combat** | Classic JRPG turn-based battle. Player acts first, then enemy. Player commands: Attack, Skill (costs MP), Item (uses a consumable), Flee (exits combat but adds +1 to invasion gauge). |
| **Invasion Gauge** | The central tension mechanic. Increments +1 on every map move and +1 when fleeing combat. Game over when it reaches `maxInvasionGauge` before the boss is killed. Displayed as `current / max` at the bottom of the screen. |

### Node Types

- **일반 전투 (Normal Combat)** — fight regular soldiers, earn gold + XP
- **엘리트 전투 (Elite Combat)** — stronger enemy, higher rewards / relics
- **상점 (Shop)** — spend gold on potions or equipment upgrades
- **휴식 (Rest)** — recover HP (still costs a turn / gauge tick)
- **보스 (Boss)** — the final objective (demon king)

### Initial Global State Shape

```json
{
  "currentTurn": 1,
  "invasionGauge": 0,
  "maxInvasionGauge": 30,
  "isGameOver": false,
  "gold": 100
}
```

---

## Unity Architecture Patterns

- **State Machine** is the central pattern. `GameManager` owns the active state. Each state implements `IGameState` with `Enter() / Update() / Exit()`. See [dev-plan.md](dev-plan.md) for the full state transition diagram.
- **Single Scene** — all states run in one Unity scene (`Main.unity`). UI panels are activated/deactivated per state. `InvasionGaugeUI` stays active at all times.
- **ScriptableObjects** for all static data (enemies, items, skills, map config). Never hardcode these values in code — they must be editable from the Inspector.

---

## Key Design Constraints (enforce these while coding)

1. **Invasion gauge is the only global timer.** It is not wall-clock time — it ticks discretely on map moves and flee actions only.
2. **Map is a DAG (directed acyclic graph), not a grid.** Node adjacency is explicit, not positional.
3. **Combat is strictly turn-based with player-first order.** No simultaneous or real-time elements.
4. **Flee is always available** in combat but always carries a +1 gauge penalty. It is an escape hatch, not a free action.
