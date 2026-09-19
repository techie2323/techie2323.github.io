---
layout: post
title: "LLM Plays Satisfactory: Devlog"
date: 2026-09-18
---

This is the story of building an LLM-driven agent that can watch and, eventually, actually *play* Satisfactory — not scripted automation, but a real Claude-driven control loop over a running game.

The project split into two halves from the start:

1. **Reading the factory** — can an agent see what's happening in a live save (machine status, item flow, problems)?
2. **Acting on the factory** — can an agent actually *do* anything — place a building, fix a starved line, walk over and inspect a machine — without a human's hands on the mouse and keyboard?

The first half turned out to be the easy one. The second took an entire day of crash debugging, a live debugger attached to the game process, and a breakthrough that came down to a single Unreal Engine thread-safety rule — and then, once solved, grew astonishingly fast: within another session, the same agent could select a building, walk to a spot, aim at it, verify what it was looking at, and place or dismantle or reconfigure it, all from nothing but coordinates written to a text file.

This devlog covers both halves, roughly in the order they happened.

## Chapter 1 — Reading the factory

Before touching anything action-related, the project started with **Ficsit Remote Monitoring (FRM)**, a read-only HTTP JSON mod that exposes live save-state over a local API — 58 documented endpoints covering everything from machine status to belt geometry to player position.

Three tools came out of this phase:

- **`frm_status.py`** — pulls every production machine and classifies it (running, at-capacity, starved, stalled, unconnected, paused, unpowered) using FRM's pre-computed `IsProducing`/`Productivity` fields rather than deriving uptime from raw save data.
- **`item_balance.py`** — aggregates production and consumption per item across the whole factory, with extractor headroom.
- **`biome_lookup.py`** — translates raw world coordinates into named biomes, calibrated against known anchor points (the HUB terminal, the Space Elevator).

A few real lessons came out of this phase too, not just tooling:

- A machine reading "not producing" on a single API poll doesn't mean it's actually idle — batch/fluid machines (refineries, blenders) can read that way mid-cycle. Checking recent `Productivity` alongside the instantaneous flag avoids false positives.
- A global net-positive item balance can hide a real local problem — a surplus in one part of a base doesn't help a starved machine on the other side of the map if nothing connects them.
- Some "problems" are intentional: a base copied from another player's save had accelerators sitting deliberately idle as standby capacity, not a bug to fix.

This half of the project was solved quickly and stayed solved — FRM's data turned out to be reliable and rich enough that no further debugging was needed here. The real difficulty was still ahead.

## Chapter 2 — The action-side problem

FRM is read-only by design — it can tell you a machine is starved, but it can't fix it. Getting an agent to actually *act* on the game meant going a level deeper: **UE4SS** (Unreal Engine 4/5 Scripting System), an injectable framework that hooks directly into the running game process.

Getting UE4SS itself running was its own multi-session fight — the stable release predated the game's engine build and its signature scans failed outright, requiring a switch to an experimental prerelease build; Windows Defender quietly quarantined the injected DLL; the install layout didn't match the documentation.

Once UE4SS was actually running, the real problem appeared: calling the game's own building-placement function (`PrimaryFire`) — whether through UE4SS's Lua bindings, or later through native C++ reflection — crashed the game, every time, no exceptions.

What followed was a full day of live crash forensics:

- Attaching **cdb.exe** (Windows' console debugger) directly to the running game process
- Discovering that inline debugger commands silently overflow and fail — command *files* were needed instead
- Filtering past benign first-chance exceptions with dozens of repeated `g` (go) commands to reach the real crash
- Capturing the actual stack trace at the point of failure

The stack trace pointed to Chaos, Unreal's physics engine, and a specific assertion: `bIsInGameThread`. The building-placement call was being made from a context that Unreal's physics system didn't consider the "real" game thread — not from a UE4SS input callback, not from a mod's `on_update()`, none of it counted.

Along the way, my own diagnostic instinct also mattered: disabling non-essential SML mods right before the eventual breakthrough, since one of them (`InfiniteZoop`) had shown up directly in an earlier crash's stack trace.

## Chapter 3 — The breakthrough

Once the root cause was known, the fix was almost anticlimactic: call the action from inside `Hook::RegisterEngineTickPreCallback` — a native C++ mod hook that runs on `UEngine::Tick` itself, which genuinely *is* the game thread, unlike everything tried before it.

The working pattern: an input keypress just sets a flag (`pending_fire_request = true`); the actual game-mutating call only ever happens inside the next EngineTick callback. Confirmed live, twice in a row, with the log line that ended the whole crash-debugging saga:

> `PrimaryFire call completed without crashing!`

This was the single most important unlock of the entire project — the moment a human keypress could reliably trigger a real, in-game building placement through native code instead of crashing the process.

## Chapter 4 — Building a real control interface

A hardcoded F9 test key proved the concept, but it wasn't a control interface. The next step was a **file-based command bridge**: the mod polls a `command.txt` file once per EngineTick, executes a single-line plain-text command, and writes a one-line result to `result.txt`. Plain text rather than JSON, deliberately — the command set is small and fixed, so a parser would have been unnecessary complexity.

A Python client (`bridge_client.py`) writes commands and polls for the result, so any script can drive the game the same way a human presses a key.

The command set grew quickly, each one hitting its own real obstacle:

- **`FIRE`** — place whatever's currently selected. Generalized cleanly across Constructor, Assembler, Manufacturer, and even conveyor belts (which turned out to need zero new code — belts are a multi-step click-to-click placement, and `FIRE` already maps 1:1 to a real click).
- **`DISMANTLE`** — same pattern, targeting the dismantle build-gun state instead.
- **`ROTATE`** — needed a different technique than position: a raw rotation write got silently overwritten every tick by the hologram's own internal rotation-step logic. The fix was calling `Scroll()`, the actual function the game uses for mouse-wheel rotation.
- **`SELECT`** — choosing *what* to build. The first attempt (`GotoBuildState`) silently did nothing; the missing piece turned out to be a required `GotoMenuState()` call first, discovered only once the mod started logging state transitions unconditionally instead of just on success.
- **`SNAP`** — world-grid and centerline snapping (what holding CTRL does manually) turned out to be the same single toggle, `ChangeGuidelineSnapMode`, behaving differently by context.
- **`SET_MACHINE_RECIPE`** — changing what an *already-built* machine produces, a genuinely different concept from `SELECT`. The real mechanism (`Server_PasteSettings`, an RCO/Remote Call Object — the same call the in-game copy/paste-settings button uses) couldn't be found in memory at all, until a native hook on the function itself, firing during a real in-game copy/paste action, revealed the actual bug: Unreal's reflection system drops the leading `A`/`U` type-prefix letter from a native class's runtime name, so every search for `UFGManufacturerClipboardRCO` was silently matching nothing — the real name was `FGManufacturerClipboardRCO`.
- **`LIST_RECIPES`** — answers "what could this machine make" by calling the machine's own `GetAvailableRecipes()`, confirmed (via a second save with fewer unlocks) to genuinely reflect the player's real research progress, not just list everything that exists in the game's data.

By the end of this chapter, the agent could select a building, place it exactly where and however oriented it wanted, dismantle things, and reconfigure existing machines — as long as a human was still doing the aiming.

## Chapter 5 — Teaching the agent to move and see

Every action so far still needed a human to physically aim the crosshair. This chapter removed that dependency entirely — with one important caveat carried through all of it: **the agent has no visual perception at all.** It cannot see a screenshot or the screen. Everything here is either pure math, or verification through the game's own internal state.

- **`LOOK_AT x y z`** — rotates the camera to face a world coordinate. Pure trigonometry (`atan2` on the delta vector, a rough eye-height offset) feeding `AController::SetControlRotation` — a plain client-side function, no special thread dispatch needed. Confirmed live: a steep downward look and a level horizontal look both matched the computed angles exactly.
- **`TELEPORT x y z`** — instant relocation via the same `K2_SetActorLocation` used for hologram positioning. Confirmed working — and also confirmed *dangerous*: a horizontal-only teleport that reused a stale height value landed the player over open space with no floor underneath, and they fell straight through the world (recovered fine via respawn, but a real lesson: terrain isn't flat, and a Z value valid at one X/Y can be empty air a few meters over).
- **`MOVE_TO x y z`** — real walking via `AddMovementInput`, the same input WASD feeds. Genuinely gradual, multi-tick movement, driven by the actual character movement component — so it respects collision and terrain naturally, solving `TELEPORT`'s fall-through risk for ordinary travel.
- **`WHERE_AM_I`** — a read-only position query, added after a stale-coordinate mistake caused that first fall-through incident. Now the standard way to get a real reference point before picking any target.
- **`GOTO_DISMANTLE` + `INSPECT`** — since there's no vision, this pair is how the agent confirms what a `LOOK_AT` call actually landed on: enter dismantle mode, then read `mCurrentlyAimedAtActor` (a genuine hit-trace the game already runs every tick). Tested live and it caught a real miss — a `LOOK_AT` target computed from a rough manual estimate ("facing WNW, about 10 meters away") actually landed on a nearby Foundation instead of the intended Constructor. That's exactly the point of the check: without it, there would be no way to know the aim was wrong at all.

By the end of this chapter, an agent with nothing but coordinates could relocate, aim, confirm what it was looking at, and act — a complete loop with zero required human mouse or keyboard input.

## Chapter 6 — First real automation

With the full move-aim-verify-act loop in hand, `auto_visit.py` combined it with FRM for the first time: pick a flagged machine from `getFactory`, walk near it, aim at it, then verify the aim actually landed on the right actor by matching FRM's own `ID` field against what `INSPECT` reports.

Building it surfaced two genuine bugs, not just design questions:

1. **A false-positive from stale log matching.** The mod only logs an `INSPECT` result on success — a failed aim writes nothing new at all — so the first version's "find the last matching log line" check was silently matching an hour-old line and reporting false success.
2. **A sub-second timestamp precision bug.** The log-timestamp parser only captured whole seconds, so a genuine fresh result written a few hundred milliseconds early (same second, technically earlier) was wrongly excluded, turning a real success into a reported failure.

Once fixed, the script correctly reported both outcomes on real tests: a clean match on a Smelter, and a correctly-identified miss when a conveyor belt physically blocked the line of sight to the same target on a later attempt.

That miss led to a deeper investigation. `MOVE_TO` has no pathfinding — it walks in a straight line — and got physically stuck against the belt. Checking FRM's own data for the belt turned up something specific: its `BoundingBox` field is literally a single point (`min` equals `max`) even though the belt's real length is very real, because the actual route lives in a different field (`SplineData`) that the script had never queried.

An automatic recovery (a sidestep maneuver) was tried and looked like it worked — but I had to manually jump to actually get clear during that same test, meaning the automatic recovery wasn't reliably the real cause. Per my own guidance at that point — *"if you are stuck just stop"* — and the broader point that people don't walk in straight lines either, we route around things — that was replaced with a simpler, honest behavior: detect no progress, stop, and let the caller decide what to do next.

A proper grid-based A* route planner (`pathfind.py`) followed, using real obstacle data — machine bounding boxes and belt spline segments — rather than reactive improvisation. It isn't fully reliable yet (documented openly rather than glossed over): a live run reported no route found and fell back to a direct walk, and a separate test in fly mode dodged the obstacle problem but introduced a new one — flying let the player end up far above the target, and the resulting steep-angle look missed. Both are open items for the next session.

## Chapter 7 — Route planning gets real, and hits a real wall

The route planner from Chapter 6 needed real tuning before it was actually useful. The safety buffers around obstacles were too conservative — 60cm around machines and 150cm around belts made "no route found" the common outcome in a real, densely-built factory. Progressive live testing (60/150, then 30/75, then even 10/30 and 1/1) showed 30/75 was the sweet spot: routes that had failed outright started succeeding, while still keeping real margin over the player's actual collision size.

Three more bugs came out of the same testing pass. The pathfinder never checked whether the start or goal point was itself inside an obstacle, so a player standing close to a machine could make the whole search fail before it began — fixed by spiraling outward to the nearest actually-clear point. Splitters and mergers turned out to be completely invisible to the obstacle model: a real 400×400cm footprint that isn't in the machine list or the belt list at all, found by teleporting to a stuck waypoint and asking what was physically there. And checking a single candidate approach point wasn't enough, since a point can be technically clear but sit in a pocket the pathfinder still can't reach — fixed by trying all four sides of a target and actually running the search against each one.

Together these took the planner from routine total failure to a real six-waypoint walk through an actual base.

Then a genuine, different kind of problem showed up: a Smelter boxed in on all four sides by neighboring machines, with a belt running overhead. No 2D route existed to it, and even an aerial teleport fallback didn't help, since the overhead belt blocked line of sight straight down too. Rather than guess, the answer came from asking a real player to solve it: walking over, live, while position was polled the whole way. The real solution turned out to be a ladder built into the Smelter itself, reached by climbing up from a neighboring rooftop — something a purely 2D grid search structurally cannot find, because it isn't a routing problem at all, it's a different axis of movement entirely.

A second, quieter discovery came out of the same investigation: foundations don't appear in the monitoring data at all. Almost certainly a deliberate performance tradeoff — a large base can have thousands of foundation tiles, and the same tooling had already visibly struggled loading very large bases. It explained why "ground level" readings throughout this testing were suspiciously flat: they were the top of a foundation layer, not real terrain, and there was no way to ask the data for that surface's existence or height.

Both findings were deliberately parked rather than chased further — ladder-aware or general 3D navigation is a bigger, separate problem from 2D pathfinding, worth its own dedicated session rather than a rushed bolt-on.

## Chapter 8 — Pipes and belts: the tacit knowledge problem

With the core action loop solid, the next question was how far it generalizes to other building types. Pipes were first, starting with a community-written manual on fluid mechanics (pressure, Head Lift, pumps, valves) for domain grounding. None of that ended up mattering for placement — pipes turned out to behave structurally just like belts, needing zero new code for either placing or verifying them.

Getting there involved a genuinely confusing detour. Aiming at a pipe junction kept reporting the same unrelated object no matter the angle, including when aiming at open sky — which looked exactly like a caching bug. A diagnostic command was added to count how many internal dismantle-state objects existed in memory, in case a stale one was being read by mistake. It found only ever one. The real cause was simpler and easy to miss: the target was just outside the tool's actual reach, and a genuine out-of-range miss happened to look identical to a stale result until the distance was checked directly.

A second issue was a real design bug, not a tooling one. Connecting a fresh pipe fully autonomously — selecting the part, aiming at a junction's reported center, firing — reported success, but the resulting pipe had a visible kink instead of running straight. Comparing it against the same connection placed by hand (and reading both back from the monitoring data) showed why: aiming at a junction's exact center lets the game's own snapping logic pick whichever connector port it prefers, not necessarily the one facing the incoming pipe. Aiming instead at a point on the near face of the junction — about a meter off-center, toward the incoming line — reconnected it straight, confirmed by comparing the raw path data before and after.

Belts turned into the more interesting half of this chapter, because they surfaced a real gap between how well someone knows a skill and how well they can explain it. There are three build modes — Default, Straight, and Curved — confirmed apart not by eye but by pulling the raw path data back out: the curved one showed a genuine ~120cm bulge away from a straight line, while the other two came back essentially dead straight even at an angle. A related quirk turned up too: a belt's starting direction is locked in at the moment it's placed and has to be deliberately rotated to match the intended path, or the belt runs at an angle with straightened ends — the visual signature of not having done that rotation.

The real lesson came from trying to place a belt spanning free space rather than connecting two buildings directly. What looked like a simple two-click action turned out, once someone who's built thousands of these had to narrate it step by step, to actually be five distinct actions: anchor to the start building, place a support pole, anchor that pole, reattach to the pole just anchored, then anchor to the end building. The fourth step is the one that's easy to lose entirely, since skipping it makes the final connection silently do nothing. It's a small, clean example of the classic problem of trying to write down instructions for a task that's become pure muscle memory — the person doing it correctly every time is often the last person who'd think to mention the step they no longer notice themselves taking.

## Lessons learned and fun facts

**The real time investment, reconstructed from git history.** Claude has no internal sense of elapsed time — no clock, no felt duration between messages. Asked how long this project had taken, the only honest way to answer was to read real commit timestamps. They revealed something neither of us had tracked: work happened across five separate sittings spanning parts of five real-world days, including a 46-hour gap and several multi-hour gaps in a single day, invisible from inside any one conversation. Reasoning from commit density alone landed an estimate around 17–18 hours; my own memory (excluding earlier scoping conversations in regular Claude chat) put it closer to 20; combined with everything before Claude Code, my total estimate was about 25 hours from first idea to a working autonomous-control mod.

**The `A`/`U` class-name-prefix bug, twice.** Unreal's C++ headers always keep a class's type-prefix letter (`AFGBuildGun`, `UFGManufacturerClipboardRCO`), but the engine's own reflection system drops it at runtime. This silently broke two different searches over the course of the project (`FindFirstOf`/`FindAllOf` for a recipe's manufacturer base class, and later the RCO lookup for `SET_MACHINE_RECIPE`) before a native function hook — capturing a real in-game action — finally exposed the real runtime name and confirmed the pattern.

**Verification matters more than confidence.** Nearly every "it works" moment in this project that mattered was one backed by the game's own data — a hook capturing a real call, a hit-trace confirming what's under the crosshair, a timestamp check ruling out stale results — rather than an assumption that a well-reasoned action must have succeeded. Several of the project's real bugs were specifically *false positives*: something that looked like success but wasn't, caught only because a verification step existed at all.

**Domain knowledge came from experience, not the manual.** Several breakthroughs traced back directly to ~4,000+ hours of hands-on Satisfactory experience rather than anything discoverable from code alone — the CTRL-snap/centerline-snap distinction, the copy/paste-settings UI as the real path to changing a machine's recipe, the multi-click mechanics of laying belt poles, and the tech-tier gating behind alternate recipes all came from direct play experience that shaped where to look next.

## What's next

Open items, roughly in order of what's already closest to done:

- **True 3D navigation.** 2D route planning is now solid, but a target boxed in with no ground route (needing a climb or a ladder instead) is a genuinely different problem, deliberately parked rather than solved.
- **Act on what's found, not just verify it.** `auto_visit.py` currently stops once it confirms it found the right machine. The natural next step is closing the loop for real — e.g. for a starved machine, finding a working supply and connecting a belt to it.
- **Fully autonomous multi-point belt/pipe runs.** The exact click sequence for free-space points is now understood (including the easy-to-miss reattach step), but a script has never yet planned and laid out a full multi-point belt or pipe run on its own.
- **Broader building-type coverage.** Pipes and belts are now confirmed working. Trains, vehicles, and drones remain untested.
- **A demo recording**, once the loop is polished enough to be worth showing off.
