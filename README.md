# Section 2. Practical Assignment 10.7

`BP_Movable` is a pawn that moves inside a rectangle at `Speed`. Leaving `Min X`, `Max X`, `Min Y`, or `Max Y` picks a new random point inside the area. Static mesh cubes are avoided by the navmesh. A pawn ahead of the actor is sidestepped. Movement uses only `Floating Pawn Movement` and `AI MoveTo`.

## Setup

`BP_Movable` inherits from `Pawn`.

- `AI Controller Class` is `AIController`.
- `Auto Possess AI` is `Placed in World or Spawned`.
- The root component is `Capsule` with the `Pawn` collision preset. The skeletal mesh is the cat. `Floating Pawn Movement` drives the actor. On `BeginPlay`, `Speed` is copied into `Max Speed`.

The level is `Content/Maps/FTEST_Map`. It contains `BP_Movable`, the cube obstacles, a `Nav Mesh Bounds Volume`, and `BP_Test`. In `Config/DefaultEngine.ini` the navmesh agent radius is `22`, the startup map is `FTEST_Map`, and the game mode is `ThirdPersonGameMode`, so the mannequin can walk up to the cat.

## Variables

| Variable | Role |
|---|---|
| `Min X`, `Max X`, `Min Y`, `Max Y` | Rectangle bounds. Leaving them picks a new random point inside the area. |
| `Speed` | Units per second. Copied into `Max Speed` on `BeginPlay`. |
| `Origin`, `Box Extent` | Center and half size of the rectangle for `Random Point in Bounding Box`, inset by the capsule radius. |
| `Target Point` | Current `AI MoveTo` destination. Also the step-length reference for a sidestep. |
| `Turn Right` | Side of the next pawn sidestep. `Pick Sidestep Point` reads it. Tick flips it after the step. |
| `Ticks Until Trace` | Frames left before the pawn trace runs again, so the current path is not replaced. |

`Is Actor in Boundaries 2D` is the template bounds check. It reads `Min X`, `Max X`, `Min Y`, and `Max Y` itself.

## Implementation

### BeginPlay

`Speed` is written into `Floating Pawn Movement` `Max Speed`. `Origin` and `Box Extent` are computed once: the center is `(Min + Max) / 2`, and each half extent is `(Max - Min) / 2` minus the capsule radius. `Pick Random Point` then calls `Move To Point`.

### Move To Point

Custom event and the only movement order. `AI MoveTo` uses `Self` as the pawn and an acceptance radius of `80`. The top exec pin is left empty: it fires in the same frame, and wiring it back would loop `BeginPlay` before the actor starts moving. `On Success` and `On Fail` pick a new random point and call this event again. `Target Point` stores the destination so a sidestep can measure its length. The path follows the level navmesh, which already cuts out the `Static Mesh Actor` cubes.

### Tick

Every frame, `Is Actor in Boundaries 2D` checks the rectangle. A false result means the actor left the area: a new random point is chosen, `Move To Point` is called, and the trace is skipped for that frame.

Otherwise, while `Ticks Until Trace` is above zero, the counter is reduced by `1` and the trace is skipped. This keeps the current `AI MoveTo` from being replaced by the next hit.

When the counter is `0`, a `Sphere Trace For Objects` runs for `Pawn` only. The player is not baked into the navmesh, so cubes stay on the navmesh and the trace is only for other pawns. The start is the actor location. The end is one capsule radius forward. The sphere radius is the same. Ignore Self is on.

If the hit is ahead of the actor (`Dot` > `0.71`), `Pick Sidestep Point` builds a destination to the left or right and keeps it inside the area. That point goes to `Move To Point`. `Turn Right` flips, and `Ticks Until Trace` is set to `10`.

### Pick Random Point

Returns a random point inside the rectangle computed on `BeginPlay`. Z comes from the actor, so the goal stays at the same height. `Box Extent` Z is zero.

### Pick Sidestep Point

Builds the destination for a pawn sidestep. `Turn Right` selects `90` or `-90`. Forward is rotated around the Z axis. The step length is the maximum of the distance to `Target Point` and the capsule radius. X and Y are clamped to `Min` / `Max` with a capsule-radius margin. Z stays at the current height.

## Verification

1. Open `FTEST_Map` and press Play. The cat moves inside the rectangle, routes around the cubes on the navmesh, and steps aside when the character is in front of it.
2. Run the automation test: Window → Session Frontend → Automation, filter `BP_Test`, and start it on `FTEST_Map`.
3. Success is `Result={Success}`. The test fails on `Object stuck.`, `Object left boundaries.`, `Object overlapped with Static Mesh Actor.`, or `Time's Up`.
