# Human-Like Autowalker — System Architecture

## Design principles

1. **Strict module boundaries.** Every module has one job, a defined input, a defined output, and no hidden side effects on other modules' state. If module B needs data from module A, it reads A's output — it never reaches into A's internals.
2. **Everything tunable lives in config, not code.** Magic numbers belong in named constants grouped by module, so tuning one system never requires touching another's code.
3. **Every module is independently testable/toggleable.** You can disable camera, sprint logic, jump lookahead, or interactables individually and the rest of the system still runs (possibly worse, but not broken).
4. **Every module logs its own decisions.** Debugging "why did it do that" should never require stepping through five files — the log should tell you which module made which call and why.

---

## Module map

```
┌───────────────────┐
│  block_costs.json │  (static config, no logic)
└─────────┬─────────┘
          │
┌─────────▼─────────┐
│  GlobalPlanner    │  coarse A* over sparse graph → strategic waypoints
└─────────┬─────────┘
          │  Waypoint[]
┌─────────▼─────────┐
│  PathAnnotator    │  tags waypoints with interaction/jump/hazard metadata
└─────────┬─────────┘
          │  AnnotatedWaypoint[]
          │
          ▼
┌───────────────────────────── AgentTickLoop (runs every tick) ─────────────────────────────────┐
│                                                                                               │
│   ┌────────────────┐   ┌────────────────┐   ┌────────────────┐   ┌───────────────────┐        │
│   │ SteeringCtrl   │   │ JumpController │   │ SprintCtrl     │   │ InteractablesCtrl │        │
│   │ (velocity/     │   │ (lookahead     │   │ (state machine,│   │ (door/button/     │        │
│   │  momentum)     │   │  jump timing)  │   │  hysteresis)   │   │  plate handling)  │        │
│   └───────┬────────┘   └───────┬────────┘   └───────┬────────┘   └─────────┬─────────┘        │
│           │                    │                    │                      │                  │
│           └────────────────────┴─────────┬──────────┴──────────────────────┘                  │
│                                          ▼                                                    │
│                                ┌──────────────────────┐                                       │
│                                │  MovementExecutor    │  ← single point of truth for          │
│                                │  (applies final      │    all agent input, resolves          │
│                                │   velocity/actions)  │    conflicts between controllers      │
│                                └──────────┬───────────┘                                       │
│                                           │                                                   │
│                                ┌──────────▼────────────┐                                      |
│                                │  CameraController     │  runs in parallel, reads agent       │
│                                │  (independent of      │  state + path, writes yaw/pitch only │
│                                │   movement)           │                                      │
│                                └───────────────────────┘                                      │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                ┌──────────▼───────────┐
                                │  DebugLogger/Overlay │  subscribes to all modules, no writes
                                └──────────────────────┘
```

---

## Module contracts

### 1. `block_costs.json` — static config
**Purpose:** single source of truth for block traversal cost and passability.
**Format:**
```json
{
  "dirt_path":     { "cost": 0.5, "passable": true, "category": "path" },
  "grass_block":   { "cost": 1.0, "passable": true, "category": "ground" },
  "oak_door":      { "cost": 1.0, "passable": true, "category": "interactable", "interactType": "door" },
  "lava":          { "cost": 9999, "passable": false, "category": "hazard" },
  "default":       { "cost": 1.0, "passable": true, "category": "ground" }
}
```
**Owns:** nothing dynamic. No code here, just data. Change traversal behavior by editing this file, never by editing planner logic.

---

### 2. `GlobalPlanner`
**Input:** `start: Vec3`, `goal: Vec3`, `world: WorldView`, `blockCosts`
**Output:** `Waypoint[]` — sparse list of coarse path points (not dense, not smoothed)
**Owns:** A* search, segment stitching for long-distance travel, per-journey cost jitter.
**Does NOT own:** movement execution, smoothing, jump/sprint decisions.
**Debug hooks:**
- `planner.lastSearchStats` → nodes expanded, time taken, fallback-triggered (bool)
- `planner.visualize()` → dumps searched node set + chosen path for overlay rendering

```
Waypoint {
  position: Vec3
  segmentIndex: int      // which planning segment this came from, for debugging long journeys
}
```

---

### 3. `PathAnnotator`
**Input:** `Waypoint[]` from GlobalPlanner, `world: WorldView`, `settings`
**Output:** `AnnotatedWaypoint[]`
**Owns:** tagging each waypoint with metadata other modules consume. Does not make behavioral decisions itself — it only labels the path.

```
AnnotatedWaypoint extends Waypoint {
  heightDelta: int              // vs previous waypoint, for JumpController
  isGap: bool                   // horizontal gap requiring jump, vs step-up
  interaction: Interaction|null // for InteractablesCtrl
  hazardNearby: bool            // for SprintCtrl (no sprint near ledges) and camera pitch
  groundCategory: string        // "path" | "ground" | "water" | etc, from block_costs
}

Interaction {
  type: "openDoor" | "openGate" | "pressButton" | "walkPlate"
  target: Vec3          // block to interact with (may differ from waypoint position, e.g. button)
}
```
**Debug hooks:** `annotator.annotationsFor(waypointIndex)` — inspect why a given waypoint got a given tag.

---

### 4. `AgentTickLoop`
Runs once per game tick. Its only job is to call each controller in a fixed order and pass state between them — it contains **no decision logic of its own**.

```
function tick(agent, path, world, dt):
    steeringOutput      = SteeringController.update(agent, path, dt)
    jumpOutput           = JumpController.update(agent, path, world, steeringOutput)
    sprintOutput         = SprintController.update(agent, path, world, steeringOutput)
    interactionOutput    = InteractablesController.update(agent, path, world)

    MovementExecutor.apply(agent, steeringOutput, jumpOutput, sprintOutput, interactionOutput)
    CameraController.update(agent, path, world, dt)   // independent, side-channel

    DebugLogger.record(tick_number, {steeringOutput, jumpOutput, sprintOutput, interactionOutput})
```
This function should be short enough to read in one glance — if it's growing, logic is leaking into the loop instead of staying in modules.

---

### 5. `SteeringController`
**Input:** `agent: AgentState`, `path: AnnotatedWaypoint[]`, `dt`
**Output:**
```
SteeringOutput {
  desiredVelocity: Vec3
  currentWaypointIndex: int
  turnSharpnessAhead: float   // exposed for SprintController to consume
}
```
**Owns:** seek/arrive steering forces, momentum, waypoint advancement logic.
**Config constants:** `MAX_SPEED`, `MAX_FORCE`, `ARRIVE_RADIUS`, `WAYPOINT_ADVANCE_RADIUS`

---

### 6. `JumpController`
**Input:** `agent`, `path`, `world`, `steeringOutput`
**Output:**
```
JumpOutput {
  shouldJump: bool
  jumpType: "stepUp" | "gap" | "none"
  chainedJump: bool   // true if this jump follows immediately from a previous one (staircase)
}
```
**Owns:** lookahead scanning for climbable/gap waypoints, momentum-based trigger distance calc.
**Config constants:** `SPRINT_JUMP_BASE_DISTANCE`, `HEIGHT_JUMP_LEAD_MULTIPLIER`, `LOOKAHEAD_DISTANCE`

---

### 7. `SprintController`
**Input:** `agent`, `path`, `world`, `steeringOutput.turnSharpnessAhead`
**Output:**
```
SprintOutput {
  shouldSprint: bool
  shouldSneak: bool     // edge-safety sneaking lives here too, since it's the same "gait mode" concern
  reason: string         // debug-only: "clearAhead" | "sharpTurn" | "nearLedge" | "approachingGoal"
}
```
**Owns:** the sprint/walk/sneak state machine, hysteresis/debounce, ledge detection.
**Config constants:** `MIN_SPRINT_CLEARANCE`, `SHARP_TURN_THRESHOLD`, `SPRINT_DEBOUNCE_TICKS`, `LEDGE_CHECK_RADIUS`
**Note:** `reason` is not cosmetic — logging *why* the state changed is what makes this debuggable when the bot sprints somewhere it shouldn't.

---

### 8. `InteractablesController`
**Input:** `agent`, `path` (reads `AnnotatedWaypoint.interaction`), `world`, `settings.useInteractables`
**Output:**
```
InteractionOutput {
  action: "rightClick" | "wait" | "none"
  target: Vec3|null
  blocksMovement: bool   // true while mid-interaction (e.g. waiting for door animation)
}
```
**Owns:** door/gate/button/plate logic exactly as designed earlier — reads `Interaction` tags from PathAnnotator, decides timing (reaction delay, animation wait), never decides *whether* a door is on the path (that's the annotator's job, done once, upstream).
**Config constants:** `INTERACTION_REACTION_DELAY`, `BUTTON_TO_DOOR_DELAY`, `DOOR_OPEN_ANIMATION_DELAY`, `CLOSE_DOOR_BEHIND_CHANCE`
**Toggle behavior:** when `settings.useInteractables = false`, this controller is skipped entirely in the tick loop — not called with a "do nothing" flag, actually skipped, so its absence is unambiguous in logs. In this mode, PathAnnotator/GlobalPlanner already routed around these blocks upstream via `block_costs.json` passability, so there's nothing for this controller to do anyway.

---

### 9. `MovementExecutor`
**Input:** all four controller outputs
**Output:** actual game input calls (movement vector, jump key, sprint key, right-click)
**Owns:** conflict resolution (e.g., can't jump and be mid-door-interaction at once — defines priority order), the single point where "decision" becomes "action."
**Priority order (example):**
```
if interactionOutput.blocksMovement:
    hold position, execute interaction only
else:
    apply steering velocity
    if jumpOutput.shouldJump: trigger jump
    apply sprint/sneak state from sprintOutput
    if interactionOutput.action != "none": trigger it (non-blocking interactions, e.g. plates)
```
This is the **only** module allowed to call actual input/movement APIs. Every other module produces data, not action — this makes it trivial to mock/replay for testing (feed recorded controller outputs back into the executor without needing a live world).

---

### 10. `CameraController`
**Input:** `agent`, `path`, `world`, `dt` — explicitly does **not** take controller outputs from movement, only raw state, to keep it decoupled as discussed.
**Output:** writes `agent.yaw`, `agent.pitch` directly (this is the one exception to "modules only output data" — camera has no downstream consumer, so writing directly is fine).
**Owns:** look-ahead targeting, POI glancing, contextual pitch, idle noise.
**Config constants:** `LOOK_AHEAD_DIST`, `MAX_YAW_SPEED`, `MAX_PITCH_SPEED`, `GLANCE_PROBABILITY`, `IDLE_JITTER_AMOUNT`

---

### 11. `DebugLogger` / overlay
**Input:** subscribes read-only to every module's output each tick.
**Owns:** nothing — pure observer.
**Suggested features:**
- Per-tick structured log line: `tick, position, velocity, sprintState+reason, jumpState, interactionState, currentWaypointIndex`
- Toggleable **path overlay** (render GlobalPlanner's chosen path + searched nodes in-world, e.g. via particles or an F3-style debug screen)
- Toggleable **per-module mute** — e.g. run with `CameraController` disabled to check if a movement bug is camera-related or steering-related
- Replay mode: dump full tick log to file, replay it against `MovementExecutor` without a live connection, to reproduce bugs offline

---

## Settings object (single shared config surface)

```
Settings {
  useInteractables: bool
  closeDoorsBehindChance: float
  sprintEnabled: bool          // master override, forces walk-only if false
  cameraGlanceEnabled: bool
  jumpLookaheadEnabled: bool   // false = reactive block-in-front jumping only, for comparison/testing
  debugOverlay: bool
  debugLogLevel: "off" | "summary" | "verbose"
}
```
Every controller reads from this shared object but never writes to it — settings are operator-controlled, not runtime-mutated by modules. This means you can flip one system off mid-run to isolate a bug without touching code.

---

## File structure (Fabric / Java)

Mirrors the module map 1:1 — one module, one class (or one package if a module grows enough to need internal splitting, e.g. `interactables/` with separate handlers per interactable type). Nothing shared implicitly; anything cross-module goes through `core/types/` or `config/`. This is a standard Fabric Loom (Gradle) project — generated via the Fabric Template Mod Generator — not a Node package.

```
src/main/java/com/yourname/autowalker/
├── config/
│   ├── BlockCosts.java             # module 1 — loads block_costs.json, exposes cost/passability lookups
│   ├── AutowalkerSettings.java     # shared Settings object, operator-editable (POJO or Cloth Config screen)
│   └── Constants.java              # all tuning constants, grouped by module (nested static classes)
│
├── core/
│   ├── types/
│   │   ├── Waypoint.java
│   │   ├── AnnotatedWaypoint.java
│   │   ├── Interaction.java
│   │   └── AgentState.java         # + one record/class per controller output shape
│   ├── WorldView.java               # thin wrapper around MinecraftClient world/chunk access (isolates client-API calls)
│   └── AgentTickLoop.java           # module 4 — hooks ClientTickEvents.END_CLIENT_TICK, orchestration only
│
├── planning/
│   ├── GlobalPlanner.java           # module 2 — A*, segment stitching, cost jitter
│   └── PathAnnotator.java           # module 3 — waypoint tagging (jump/interaction/hazard/ground)
│
├── controllers/
│   ├── SteeringController.java      # module 5
│   ├── JumpController.java          # module 6
│   ├── SprintController.java        # module 7
│   ├── InteractablesController.java # module 8
│   │   └── handlers/
│   │       ├── DoorHandler.java
│   │       ├── GateHandler.java
│   │       ├── ButtonHandler.java
│   │       └── PressurePlateHandler.java
│   └── CameraController.java        # module 10
│
├── execution/
│   └── MovementExecutor.java        # module 9 — only class allowed to write client.player.input / send interact packets
│
├── debug/
│   ├── DebugLogger.java             # module 11 — per-tick structured logging
│   ├── Overlay.java                 # in-world path/node visualization (WorldRenderEvents / HUD render)
│   └── Replay.java                  # offline replay of recorded tick logs against MovementExecutor
│
└── AutowalkerMod.java                # entry point: implements ClientModInitializer, wires modules together

src/test/java/com/yourname/autowalker/
├── planning/
│   └── GlobalPlannerTest.java
├── controllers/
│   ├── JumpControllerTest.java
│   ├── SprintControllerTest.java
│   └── InteractablesControllerTest.java
└── fixtures/
    └── MockWorldView.java            # fake WorldView for isolated controller testing, no live client needed

src/main/resources/
├── fabric.mod.json                   # mod metadata, entrypoints, dependencies (fabric-api, fabric-loader)
├── data/autowalker/block_costs.json  # module 1 static config, loaded by BlockCosts.java via Gson
└── assets/autowalker/...             # icon, lang files (only needed if you add a config screen)
```

**Rules that keep this structure honest as the project grows:**
- A class under `controllers/` may reference `core/types/` and `config/`, but never another class in `controllers/` directly — if two controllers need to share data, it flows through `AgentTickLoop`'s method parameters, not a cross-reference. This is what stops the "everything imports everything" tangle that makes debugging hard.
- Only `execution/MovementExecutor.java` and `core/WorldView.java` are allowed to touch the real Minecraft client API (`MinecraftClient`, `client.world`, `client.player`). Every other class works against the plain data types in `core/types/`. This is what makes the `fixtures/MockWorldView.java` approach possible — controllers never know or care whether they're talking to a real client or a fixture, so they're plain JUnit-testable without launching the game.
- `debug/` classes subscribe to data, never produce it — if a bug only appears with `DebugLogger` attached, that's itself a bug (logging must be side-effect-free).

## Suggested build order

1. `block_costs.json` + `GlobalPlanner` — get raw coarse pathing working, verify with overlay logging before anything else exists.
2. `SteeringController` + minimal `MovementExecutor` — confirm the bot moves smoothly between waypoints.
3. `SprintController` — layer in sprint/walk switching, verify against `reason` logs.
4. `JumpController` — add lookahead jump timing.
5. `InteractablesController` — doors/buttons/plates, toggleable.
6. `CameraController` — last, since it's fully decoupled and easiest to bolt on once movement already looks right.
7. `DebugLogger` — build incrementally alongside every step above, not at the end. Each module should be logging from the moment it exists.

Building the logger last is the one order swap I'd actively avoid — retrofitting logging onto five already-written modules is exactly the kind of pass that gets skipped once the bot "looks like it's working."
