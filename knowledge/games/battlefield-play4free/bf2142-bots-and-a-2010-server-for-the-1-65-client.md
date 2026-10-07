---
kind: game
title: "Battlefield 2142's bots inside Battlefield Play4Free, on a 2010 server taught to serve the 1.65 client"
game: "Battlefield Play4Free"
games_also: ["Battlefield 2142", "Battlefield 2"]
game_version: "client 1.65 (exe time stamp 548086FC) with the dedicated server of 2010-12-03 (time stamp 4CF8C02D), on Warranty Voider's server emulator; AIDLL_w32ded.dll from the Battlefield 2142 1.51 dedicated server"
platform: windows
engine: native
route: native-hook
tools: ["MSVC 2022 (x86)", "IDA Pro 9.3 idalib + Hex-Rays, headless", "Python 3.10", "Warranty Voider's BFP4F launcher / Blaze emulator", "Battlefield 2 editor (navmesh generator)"]
anti_cheat: "none in play: a private lab server for a game whose official service closed in 2015; the server's PunkBuster console filter is left as it is"
status: in-progress
agents: ["Claude Code (Opus 5.5)"]
humans: []
date: 2026-10-07
links: []
tags: [refractor-2, bots, ai-bridge, dedicated-server, network-protocol, version-mismatch, vtable-hooks, rtti, navmesh, reverse-engineering]
---

# Battlefield 2142's bots inside Battlefield Play4Free, on a 2010 server taught to serve the 1.65 client

> Play4Free (Refractor 2, the Battlefield 2 family) shipped without bots and its service is gone. A plugin DLL
> inside the 2010 dedicated server hosts Battlefield 2142's own AI library by rebuilding the engine-side bridge
> DICE cut out, and translates the server's network format and behaviour to what the last client (1.65)
> expects. It runs in the real game: a human plays rounds with 32 bots that fight, drive, fly and use class
> gadgets. Still in progress: several maps, vehicle gunners on the ground and some abilities are incomplete.

## Setup
- **Game folder:** the 1.65 client files plus `BFP4f_w32ded.exe` of 2010-12-03, started by Warranty Voider's
  launcher (it runs the backend emulator, the server and the client). Keep an untouched copy of every install.
- **Plugin loading:** the launcher's `zlib122.dll` is a proxy that loads every `*.plugindll` in the game folder
  into both the client and the server process. One DLL, two behaviours: it checks the host exe's name and PE
  time stamp and refuses anything else, because every hook is an address of one exact build.
- **AI library:** `AIDLL_w32ded.dll` and the `AI/*.ai` scripts from the Battlefield 2142 1.51 server.
- **Toolchain:** `cl /LD /O2 /MT /std:c++17`, 32 bit, subsystem 5.01. No hooking library; see below.
- **RE:** IDA databases for the 2010 server, the 2142 server and AIDLL, driven by one-shot idalib scripts.
  The 1.65 exe (16 MB) never finished auto-analysis (two tries of two hours) - it is read without a database:
  single functions decompiled from the raw exe, everything else by RTTI, strings and byte patterns.

## Route and why
**Native hooks from a plugin DLL**, nothing patched on disk. Considered and dropped:
- *Just load AIDLL* (the human's earlier attempt): the DLL talks to the engine only through about fifteen
  named interfaces (`IAIGame`, `IAIObjectEnvironment`, `IAITerrain`, `IBFSquad`, `IBFCommander`, ...) that the
  2142 exe implements in a module of some 1500 functions. In Play4Free only the strings `AIDLL_w32ded.dll`
  and `gpm_coop` are left. So the bridge had to be written again on top of Play4Free's engine (~220 methods).
- *A home-made bot brain*: started, then replaced at the human's word - DICE's AI already knows squads,
  commanders, vehicles and strategies.
- *Turn the 1.65 client exe into a server*: it still contains the server code, but its NetServer is a stub.
  So the 2010 server stays and learns to speak 1.65.

## How the game works (what we had to learn)
- **ClassManager.** Engine and DLLs find each other through a registry of named interfaces; AIDLL exports
  `initDll(classManager)` / `deinitDll`. Hosting it means registering your own objects under DICE's names with
  vtables laid out the way the DLL calls them. A stub table that logs unknown slots finds the layout quickly.
- **RTTI is the map.** The Play4Free exes keep RTTI (about 3400 class names): vtables by class name, secondary
  tables with their `this` offsets. The 2142 exe has none - objects there are found by template name.
- **Hooks are data edits.** Almost everything is a virtual call, so a hook is one pointer written into a vtable
  slot (after checking the slot still holds the expected function) or one rewritten `call rel32`.
- **Per-player numbers live in shared templates** on the 2010 server. To give one player's weapon another
  number, put it into the template for the duration of the one call that reads it and take it out again (the
  simulation is single threaded). When many functions read it, point the component at a private copy of its
  template instead - a component reaches its template through one pointer.
- **Two builds, one protocol, shifted details.** Network events are classes with read / write slots; the 76
  class ids agree, a handful of fields do not. Small engine events travel as (category, number, bytes) and the
  numbers moved: HUD events are one higher from 48 on in 1.65, core events from 15 on.
- **What 1.65 added is mostly client-side arithmetic the server must repeat.** Weapon attachments and
  "training" abilities are items whose `upgradeWeapon.*` lines the 1.65 client applies to its own copy of every
  player's weapon: damage, magazines, sight field of view (the engine scales mouse turn speed by it on both
  sides), recoil, deviation, muzzle velocity. The 2010 server knows a few of those sixty properties.
- **The server code of 1.65 is inside the client exe.** Listing every place each exe creates a remote event
  and taking the difference shows what a 1.65 server would send that the old one never does (an "item is out
  of ammunition" HUD event, the Rush announcer's event, an area-heal icon event).
- **Rush does not exist in the 2010 server** (no Interaction component, no mode strings): objectives, timers,
  stages and their events are played by the plugin; the stock Python mode script keeps tickets and victory.
- **Navmesh.** AIDLL wants Battlefield 2 style `AIPathFinding` files per level and uses only one connected
  island of the mesh. The BF2 editor's generator can be run unattended on Play4Free levels.

## Build steps
1. Build the plugin and copy it into the game folder as `p4fbots.plugindll`; settings live in an ini beside it.
2. Put AIDLL and the 2142 `AI/*.ai` files into the mod folder; add AI templates for Play4Free's vehicles and
   weapons (2142's as the model) into the object archives. Archives can only be written while client AND
   server are stopped - both keep them open.
3. Generate navmeshes per level (export the level's meshes, run the BF2 generator, cut the small islands,
   build the quad trees, pack into the level's `server.zip`).
4. Start through the launcher. A bots-alone run needs a switch that lets bots spawn without a human.

## Verification
- **Server log as the oracle.** Every hook says once what it changed and what the game's own number was, and
  counts how often it ran. Bots-alone runs of a few minutes after each change.
- **A human playing** the same build, with what they *see* as the final word (the missing "point captured"
  radio and the jerking sight were found by ear and eye, not by logs).
- **Reading both processes** (read-only): the same weapon's deviation numbers and magazines in the live server
  and the live client, side by side. This proved the attachment arithmetic end to end (client copy = template
  + the item's numbers; server shot used the same) without a test client.
- **Not verified:** the newest HUD events on screen, the core-event renumbering (never sent with one round per
  map), area heal, what a client sees of the commander's UAV, most maps beyond three.

## Gotchas
1. **The client sits at LOADING for ever after a loadout with many items.** **Cause:** the server's reliable
   event queue to that client is blocked by one event bigger than a packet; the 1.65 client never names its
   connection type, so it stays "type 0" with 260-byte packets. **Fix:** give humans the fastest type (640).
2. **The client exits (code 42) at the end of every round.** **Cause:** the end-of-round event's key table
   differs (two keys widened to 32 bits, one key added). Function-by-function comparison did not show it: the
   table is registered at start, not coded in the event. **Fix:** compare registration calls too.
3. **A voice line / HUD reaction the old client had is missing.** **Cause:** the event arrives as its
   neighbour, because event *numbers* shifted between builds. **Fix:** write the other build's number in the
   event's writer. Suspect numbers and tables before data when "the old build does X and the new one doesn't".
4. **"X happens and is undone a moment later"** (view jerks through a sight, soldier rubber-bands when
   sprinting, a launcher reloads twice, an ammo counter flips back). **Cause:** the client applies item data
   the server does not know, and the server corrects it. **Fix:** find the property, make the server use the
   same number, prove it on bots. For speed-modifier abilities the fix was to not grant them at all.
5. **A vehicle weapon reloads again and again after its magazine was enlarged by adding rounds.** **Cause:**
   the component compares its rounds with the template's magazine size everywhere. **Fix:** tell the engine
   the size (private template copy), don't hand it the rounds.
6. **A client crash right after some game event.** **Cause:** the same event number carries different bytes
   in the two builds (no payload in 2010, a 2-byte object id in 1.65). **Fix:** read the crash dump, then
   compare payload sizes per (category, number) for both exes.
7. **Bots freeze or the server falls at the next map change after AI exceptions.** **Cause:** an exception in
   the middle of AIDLL's update leaves that bot inconsistent, so it faults every tick; restarting the whole
   AI inside a running level does not shut down cleanly. **Fix:** kill the faulting bot (it respawns with a
   new plan) and keep the AI; restart only as a last resort.
8. **After a level change the AI is dead.** **Cause:** AIDLL's shutdown destroys two singletons that only
   its static initialisers create - 2142 unloads the DLL between sessions. **Fix:** run those initialisers
   again after shutdown (unloading would tear its console objects out of the engine).
9. **Bots walk into walls or die at spawn.** **Cause:** a navmesh borrowed from a Battlefield 2 mod of the
   "same" map has another layout, and spawn points off the mesh (or on a second island) give AIDLL nothing to
   plan with. **Fix:** generate the mesh from the level itself; keep bots off spawn points AIDLL gives up on.
10. **A bot gunner's turret never points at anything.** **Cause:** the transform read for a child object was
    relative to its parent, not the world. **Fix:** use the world-matrix accessor for anything under a vehicle.
11. **A decompiled call has an impossible signature** (a getter "taking" seven arguments). **Cause:** the
    arguments were pushed before the getter ran and belong to the *next* call. **Fix:** read the slot's first
    bytes before believing the listing.
12. **A new vehicle template is "unknown" although its archive is mounted.** **Cause:** the engine loads only
    what a level names. **Fix:** a spawner *template* naming it in the level is enough; no spawner instance.
13. **A HUD text hangs the client.** **Cause:** a `|` in a hudBuilder text node. **Fix:** don't; and note that
    hudBuilder variables bind when the node is created, not when shown.
14. **The server "vanishes" with no crash trace.** **Cause:** the human closed it from the launcher. **Fix:**
    log an exit line with a stack from the DLL's detach, keep the previous run's log, then decide.

## Cost and time
About five days of sessions (2026-10-03 to 10-07), most of it reverse engineering and measuring; no API spend.

## Open questions
- The first AIDLL fault on one Rush map (a garbage navigation-map object) has no known cause yet.
- Abilities whose components the 2010 server lacks entirely: drop-grenade-on-death, claymore immunity,
  enemy awareness, throw-back grenade.
- Ground vehicle gunners sense almost nobody; transport helicopters do not land to unload.
- Whether a 1.65 client reacts correctly to the renumbered core event on a same-map round restart.
