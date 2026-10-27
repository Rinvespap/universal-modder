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
date: 2026-10-28
links: []
tags: [refractor-2, bots, ai-bridge, dedicated-server, network-protocol, version-mismatch, vtable-hooks, rtti, navmesh, reverse-engineering, crash-dumps, ballistics, level-conversion, ladders, reconstructed-source, direct3d9, shaders, shadows]
---

# Battlefield 2142's bots inside Battlefield Play4Free, on a 2010 server taught to serve the 1.65 client

> Play4Free (Refractor 2, the Battlefield 2 family) shipped without bots and its service is gone. A plugin DLL
> inside the 2010 dedicated server hosts Battlefield 2142's own AI library by rebuilding the engine-side bridge
> DICE cut out, and translates the server's network format and behaviour to what the last client (1.65)
> expects. It runs in the real game: a human plays rounds with 32 bots that fight, drive, fly, man fixed
> weapons, use class gadgets, form squads under a commander and ferry each other by helicopter, on the game's
> own levels (Rush included) and on about twenty converted Battlefield 2 levels. A copy for other people, with
> the plugin's source, is made by a packaging script. Still in progress: vehicle gunners on the ground, bots
> that leave a vehicle they have just entered.

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
  numbers moved: HUD events are one higher from 48 on in 1.65, core events from 15 on. WHO gets an event
  differs too: the 1.65 server code posts some events to every client and some only to clients already in the
  game - list the "post" calls of both exes, not only the events.
- **What 1.65 added is mostly client-side arithmetic the server must repeat.** Weapon attachments and
  "training" abilities are items whose `upgradeWeapon.*` lines the 1.65 client applies to its own copy of every
  player's weapon: damage, magazines, sight field of view (the engine scales mouse turn speed by it on both
  sides), recoil, deviation, muzzle velocity. The 2010 server knows a few of those sixty properties.
- **Template properties only 1.65 knows** (explosion fall-off, items that go with their owner's death, supply
  objects that serve one ability ...) are silently dropped by the 2010 server when it executes the files. The
  plugin stands in the console's executor while the files load and keeps those lines per template: the numbers
  come from the game's own files, no generated tables to go stale.
- **The server code of 1.65 is inside the client exe.** Listing every place each exe creates a remote event
  and taking the difference shows what a 1.65 server would send that the old one never does (an "item is out
  of ammunition" HUD event, the Rush announcer's event, an area-heal icon event). Aligning the two builds'
  vtables of one class by function fingerprints shows which slots were inserted (Player: one before slot 24).
- **Rush does not exist in the 2010 server** (no Interaction component, no mode strings): objectives, timers,
  stages and their events are played by the plugin; the stock Python mode script keeps tickets and victory.
- **Navmesh.** AIDLL wants Battlefield 2 style `AIPathFinding` files per level. It takes a mesh of several
  separate pieces only after a small host patch, and its path search never leaves a piece. The BF2 editor's
  generator can be run unattended on Play4Free levels.
- **The AI's questions are the specification of the bridge.** AIDLL's behaviour choice is a table per bot:
  the behaviours, a weight for each, the one in use, and for each behaviour the information record of its last
  target. Reading those out of the library (which behaviour should have won, what its target is) turns "the
  bots do not do X" into "the host answered question Y wrongly" - see Gotcha 15.
- **One function says along what a weapon fires.** A fire component's "fire matrix" method (the same slot in
  the server and, for four component classes, in the client) leaves right / up / ahead / place in the component
  and returns it; the shot that follows starts the projectile there with the carrier's velocity plus the muzzle
  velocity along "ahead". Projectiles are integrated 30 times a second: vertical speed minus 14.73 x the
  template's gravityModifier x dt, the velocity scaled by a keep factor (with drag: a loss of 3.66e-5 x
  mass^1.46 a second), then the place. Measured by following real projectile objects, not read from code.
- **Auto controllers.** A "player" with an id of 256 or more is an automatic one (sentry gun, drone): the
  Player constructor marks it and the player manager keeps such players in a list of their own. Battlefield
  2142's auto-controller AI drives them once the bridge hands them over.
- **The engine kept more of 2142 than it shows.** Play4Free's squad manager and commander-tools manager are
  alive in the server; only the menus were removed. Feeding them (instead of a squad list of the plugin's own)
  gave squads, leaders, orders and commander tools that the client draws by itself. Map and name-tag features
  of the older game are cut in three places at once: a mask in the client's code, the loading of the pictures,
  and the pictures themselves - all three have to be put back.
- **What a projectile does is an interface too.** The library's collision handler wants to be told of every
  hit, shot, heal and sighting; until the host calls it bots do not react to being shot at from behind.
- **Mines are a manager, not a behaviour.** The host reports every mine with a reach; twice a second the library
  asks "does this object lie still" and then makes the navmesh polygons under it five times as expensive.
- **To the AI a vehicle is a circle** (its physics sphere). A helicopter's circle is smaller than its hull: the
  tail boom lies outside it, and a bot on foot walks into it (Gotcha 30).
- **Battlefield 2 content converts.** Levels need the conquest layout rewritten for Play4Free's mode, the
  light settings lines the 1.65 renderer expects, strategic areas with neighbours for both home bases, and a
  generated navmesh; vehicles need Play4Free's camera and HUD lines and an AI template.
- **Score lines are engine events, not script.** "Kill", "Destroyed Vehicle +50" and the like are in-game events
  of an engine manager with their numbers in a settings file; the two builds agree on the numbers, but WHEN the
  2010 server raises one can differ (Gotcha 46). The stock Python mode scripts only keep tickets.
- **The renderer is Direct3D 9 behind one exported call.** Every device the client makes comes through
  `Direct3DCreate9`, so one hook there reaches the device's vtable: anisotropic filtering (sampler states of
  mipmapped colour textures only), a Direct3D 12 path through the system's D3D9On12 layer (Gotcha 52), pictures of
  the back buffer on request. Anti-aliasing and sharpening belong in the game's own post-processing shader, which
  runs before the HUD is drawn - a filter at Present would blur the HUD's text too. The shadows are one map the
  engine fits around its casters; the plugin gives it four cascades around the camera instead (Gotcha 54).

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
- **A feature that checks itself.** The gunship's fire control (the line of fire turned so that the shell lands
  where the sight looks) finds its own shell among the game's objects after each shot and logs how far from the
  aimed place it was last seen: 0.5 - 6 m in a human's battle, without anybody watching for it.
- **Reading the AI's mind instead of guessing.** For "bots fire at their own side": every bot with the trigger
  down is logged with the target the library itself keeps for his behaviour in use, and with any human of his
  side near his line of sight.
- **Count the reasons for "never".** A host rule that never fires gets a counter per reason it declined; one
  bots-alone run then says which condition is wrong (a "bot is in a fight" test judged by the behaviour's name
  refused 606 of 796 chances; judged by the library's own trigger wish it worked).
- **A controller needs a trace line, not a look.** Half-second lines of wanted against actual (place, speed,
  attitude, sticks) for every helicopter the host flies turned one played round into the data for tuning.
- **A table per battle.** Every death (place, side, vehicle, killer, weapon) and every destroyed vehicle goes to
  a CSV from the mode's Python handlers; a report over it found what no single look shows (on one level a tenth
  of one side's deaths had no killer: a squad posted on a hilltop drove its jeep up there and rolled - the rule
  that sends only squads on foot to such posts took those deaths from 23 to 3).
- **The human as the A/B judge.** When "everything got worse", all host additions to the AI were switched off for
  one battle by a settings file (with a backup of the old values) and on again: the human judged the additions
  smarter. Since then a "worse" is traced to the one addition that causes it, never answered by switching all off.
- **Not verified:** the newest HUD events on screen, the core-event renumbering (never sent with one round per
  map), area heal, what a client sees of the commander's UAV, the corrected seating of Gotcha 19 on a client
  (the guard's silence will show it), most converted levels beyond a bots-alone tour; Gotchas 46, 50 and 54
  are in the game but not yet seen by the human.

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
15. **The hosted AI never uses something its own data says it should** (bots ignored fixed weapons and boats
    for days; a host-made "assist" was built and later switched off). **Cause:** three wrong answers of the
    bridge, none of them a missing behaviour: the "health" question returned 0 for objects without armour, so
    every fixed weapon counted as destroyed; the slot "where does one get into a fixed unit" was a stub (the
    original searches along the navmesh from the object to a point 12 m behind it); and "altitude above the
    ground" ignored water, so a boat on deep water was "more than 8 m up" and dropped as a candidate.
    **Fix:** before adding anything, log every behaviour's weight for one bot, see what the expected behaviour's
    evaluation asks the host, and read that same slot in the original exe (find it through a neighbouring slot
    you already know). Stubbed slots are the first suspects.
16. **The server goes silent: no crash, no log line, no exit.** **Cause:** a navmesh of about 100 separate
    pieces (roofs added as islands) sends AIDLL's path search round until its allocator opens a hidden
    "Memory Error" message box. **Fix:** dump the main thread's stack of the silent process (it sits in a
    dialog), keep the piece count low (eight work), keep the previous archive beside the new one.
17. **A bot on a detached piece of the mesh stands still for ever.** **Cause:** path search is per piece.
    **Fix:** the host finds jump links between pieces from the mesh's border edges and walks / jumps the bot
    across; or cut the piece away.
18. **A client crash dump "has no stack".** **Cause:** the automatic Windows dumps hold neither heap nor code
    pages, so a tool that validates return addresses by the bytes before them finds none. **Fix:** the fault
    address names the function; its caller's return address stands at esp + (the function's `sub esp` + its
    pushes) - judge stack words by module ranges alone. And list ALL dumps of the folder with their fault
    addresses first: the same address three times in an evening was one bug, one of the three had been
    written down as "the human left the game".
19. **The client dies with an access violation the moment some weapon fires.** **Cause:** the engine's fire
    matrix method takes the soldier ("default vehicle") of the weapon's player and uses it without a check.
    The player without a soldier was a bot the plugin had seated at a fixed weapon with the engine's
    enter-vehicle call and its "as the object's auto controller" argument set - copied from the code that
    seats a sentry gun's player, who has no soldier: that path sends no enter-vehicle event, the client loses
    the bot's soldier, and his gun's first shot ends the game. **Fix:** the argument is 0 for anybody with a
    soldier. **How it was found:** not from the heap-less dumps, but from a guard in front of that slot (all
    four tables) that asks the same question first, lets the shot go along the weapon's own matrix and writes
    down weapon, vehicle and player - one evening of play named a bot at a fixed weapon four times.
20. **The client stops at an engine check** ("soldier still carries a kit at destroy") or with C0000417 in the
    collision manager when a crate standing in the level is broken. **Cause:** the 1.65 client has its checks
    compiled in, and it throws away the collision templates of effects nobody used while the level loaded.
    **Fix:** step over the check and log the object; make the template again on demand.
21. **Weapons and soldiers are magenta on a converted level.** **Cause:** the level has none of the light
    settings lines the 1.65 renderer reads. **Fix:** add them with a tool to every converted level - and look
    up which level the server has really loaded before analysing a screenshot.
22. **Bots of one side never capture anything.** **Cause:** in the strategic areas the flags do not name that
    side's home base as a neighbour (the target's own list is what counts). **Fix:** generate the neighbour
    lists for both bases.
23. **A new shell's burst does nothing to vehicles.** **Cause:** Play4Free's material table has no damage
    entry for that projectile's material against armour. **Fix:** give the shell a material that has one.
24. **Bots hold "fire" in the first seconds of a round although nobody has seen anybody.** **Cause:** it is
    the horn: the only bots that did were drivers of vehicles whose AI template carries the type
    `ITHasCarHorn` (the library has a HonkHorn step for a blocked driver). **Fix:** none needed - but never
    give that type to a vehicle whose driver's fire key is a weapon.
25. **Everything the session started (server, client, a navmesh generator) is gone.** **Cause:** the agent's
    desktop app restarted and took its child processes with it. **Fix:** start anything long-lived through a
    detached, hidden launcher script; quotes inside its argument string do not survive.
26. **A patch script emptied a source file; comments in another alphabet turned to mojibake.** **Cause:** the
    target was opened for writing before the text had been encoded; a file went through PowerShell's
    `Get-Content | Set-Content`. **Fix:** encode first, write a file beside the target, replace it in one
    step; take the encoding from the code that reads the file.
27. **The hosted AI faults in a human's game but never in bots-alone runs.** **Cause:** a vehicle kind (the
    boat) was mapped to a navigation cluster graph the level does not register; the library then reads a layer
    number from memory nobody wrote - 160 000 wrong lookups in five minutes, a fault only when that memory is
    not mapped. **Fix:** choose the graph per level; and put a bounds-checking stand-in that LOGS in front of
    the table - it shows the wrong lookups that never fault.
28. **The client is thrown back to the menu when something ordinary happens** (a squad changes). **Cause:** one
    event of the old server is unreadable for the new client, which drops the connection; the backend's log
    holds only a "context" number. **Fix:** that number is the reason (29 = packet unreadable); log the event
    classes of every packet on the server to name the event, then compare its read / write in both builds.
29. **Bots cannot reach an objective one floor up.** **Cause:** a host "assist" walked the bot the last metres
    in a straight line. **Fix:** every walk the host makes asks the navmesh for the way.
30. **Bots stand at a helicopter's tail until they are removed.** **Cause:** a vehicle is a circle to the AI (see above), and the tail boom is outside it; a bot blocked for about nine seconds is given up on by the library. **Fix:** for any bot on foot
    beside a standing helicopter the host steers along the hull's box (out sideways first, then along it).
31. **The library's own troop transport drops everybody from 40 m and lets the helicopter fall.** **Fix:** the
    host flies transports: the pilot's own "bail out" radio line is kept back and used as the trigger; land or
    hover low by the threat at the place, only passengers leave. **What the controller needed:** attitude
    follows the stick about 1.5 s late (slew the wanted tilt, damp by the turn rate), hover trim measured in
    flight, a NEGATIVE collective to really descend; for a circle flown sideways with the nose on its centre,
    v^2 = g x height makes the bank that holds the circle equal to the angle down to the centre.
32. **A behaviour is never chosen although the bridge answers its questions.** **Cause:** the plan is only
    built for a bot whose weapon has a certain AI type (2142's mine-defusing tool); no weapon of this game has
    it. **Fix:** read the plan builder's precondition before debugging answers; give the bots the same result
    another way (they shoot enemy mines) and leave the dead switch off.
33. **Bots ignore ladders / fall into an empty pool and stay.** **Cause:** generated meshes have no ladder
    polygons, and most ladders are parts of other objects (65 of 71 on one level were found only by walking
    the object trees). **Fix:** a tool adds ladder polygons; the host climbs. Moving a bot by setting its
    position counts as movement for the engine and as "stuck" for the library - steer it instead.
34. **A client that joins while a bots-alone battle is running stays at LOADING.** Open. The normal start
    (bots are created when the first human enters - which the 1.65 client also needs to draw their name tags)
    works; do not leave a bots-alone test server up for a human.
35. **The lab folder is not a release.** It holds every session's logs, screenshots, the player's saved
    choices by name, backup and trial archives, a Windows profile path in two tool constants, and 200 comments
    quoting the human's requests. **Fix:** a packaging script: exclusion rules, a settings file written anew
    (described in English, values taken from the live one, test switches off), the quotes cut out of the copy
    (a rule where quote and attribution can simply go, a hand-written replacement keyed by the quote's hash
    where the sentence needs its sense kept - translating them was not what the human wanted), and a read-back
    that searches text files, executables and the text members of every archive for the profile path and a
    list of private words, and compiles every script of the copy.
36. **The second round on the same map leaves the 1.65 client on its loading screen.** **Cause:** at a round's end
    the 2010 server restarts the round in place (a "soft restart") unless its rounds-per-map counter says change
    the map; the 1.65 client expects a level load. **Fix:** whenever a round ends, call the server's own "next map"
    step with the same map as the next one - every round is a level load; the warmup that bots joining late start
    again is ended by the host once enough players are there.
37. **A reconstructed engine as the reference for a hand-made bridge.** Battlefield 2's engine was published
    reconstructed from its binaries (OpenBF2). Its engine-side AI interfaces have the same virtual order as
    Battlefield 2142's (the later game appended slots at the ends, or inserted one), and its headers give every
    slot a MEANING the stripped binary never had. The comparison found: a slot that must return a float in ST0
    answered with an integer in EAX (the library read an empty FPU register - NaN - and "our flag is being taken"
    never fired), a template getter read two slots too far (every flag's radius -1, so "inside the flag's area"
    never held), and spawn groups placed at their flag instead of at the mean of their spawn points (370 m off on
    one level). **Method:** match slots by meaning and by the bytes each pops, not by index; confirm every finding
    in the disassembly of the build that runs.
38. **A widescreen squeeze of the HUD cuts text and stretches pictures at the screen edges.** **Cause:** the client
    clips HUD text to the layout's sides (0 and 1280); a group shifted outward crosses that side, the part beyond
    is cut, and a clipped quad gets a corner exactly on the screen edge, which the squeeze leaves where it is -
    stretched. **Fix:** squeeze everything uniformly and keep the corner groups at their 4:3 places; moving them out
    needs the clip rectangle itself (not found yet).
39. **Ladders that are parts of objects are missing from the navigation mesh.** The editor exports only a level's
    own ladder objects; an aircraft carrier's 25 ladders are children of its hull parts. **Fix:** their places from
    the level's placement file times the parent templates' child entries (yaw only), their size from the ladder's
    collision mesh, each joined to the mesh with two "ladder" triangles. The carrier's deck then stops being an
    island: the library walks bots down through the ship to the sea and the boats by itself.
40. **The server stops on "memory.dll: sanity check: block size ... doesn't seem sane".** **Cause:** two ladders
    took the same 60-metre open edge of the mesh, so their triangles lay on top of each other (one edge shared by
    four triangles); the library's path search first faulted in a bot's update, then asked for 100 MB. **Fix:** a
    ladder joins a 1.5 m piece of the edge at its own place - the triangle that owns the edge is cut in three, so
    the mesh stays whole - and its edges are closed for the next ladder; before packing, every mesh is checked for
    triangles that are there twice and edges of three or more triangles, against the mesh that runs fine.
41. **Bots fall from a deck on to a ledge 1.4 m lower and jump at the edge until they are removed.** **Cause:**
    two thresholds of the host: "a ladder rises at least 1.5 m" (the ledge's ladders rise 1.4 - still more than a
    soldier jumps) and "the bot's way is followed only in a plan called GotoWaypoints..." (the navigator holds a
    way in other plans too, e.g. on the way to a vehicle). **Fix:** 1.2 m, above the jump height; any plan whose
    way the navigator holds. A ladder the bot only brushed past lets him off at the end he got on at - otherwise
    he goes up, his way sends him down, and the ladder takes him up again.
42. **"Merging the client and server archives breaks the game"?** In this game's original install no file name is
    in both halves (client: meshes, textures, sounds, shaders, fonts; server: .con / .tweak, collision meshes, AI,
    animations, XML), the client mounts both, and the extension alone decides the half. Merging changes no
    content; a split back can be done by extension.
43. **A bot spawned in a building pushes against a wall for ever.** **Cause:** the navmesh generator builds from an
    object's AI collision mesh when it has one (it prefers it over the soldiers' and the vehicles' meshes), and
    many buildings' AI meshes are a solid block without the floors inside - the mesh had only the roof above the
    spawn point. **Fix:** for objects whose soldiers' collision has floors the AI mesh lacks, give the generator
    the soldiers' mesh instead (generator input only; the game's files stay). A survey found 14-37 such object
    kinds per level, an aircraft carrier's insides among them.
44. **Bots at a base are given up on although the new mesh covers it.** **Cause:** the editor's quad-tree save
    wrote polygon kind 0 for half of one level's polygons (its own text output had ground / water / ladder for
    all); kind 0 hides a polygon from the library. **Fix:** copy the kinds back from the text output, and count
    kind-0 polygons before packing any mesh. Making a long-hidden area walkable can wake a fault in the search:
    after such a change run bots alone for a long time (one deck made walkable hung the server in the library's
    open-list refill).
45. **The server exits at the level's start after a Python handler was added.** **Cause:** the handler for
    player deaths was registered before the mode's own scripts had set up theirs. **Fix:** register right after
    the scripts register their own handler for the same event (wrap their registration). The engine's Python is
    2.3 without the `os` module.
46. **"Destroyed Vehicle +50" is paid many times for one vehicle.** **Cause:** the 2010 server raises that event
    on every hit whose damage reaches the armour's hit points - also on a vehicle already destroyed; the 1.65
    server code also asks that the vehicle had hit points left before the hit. **Fix:** the same test in front of
    the old server's event. Read the condition in both exes, not the score settings.
47. **The sight jerks while turning only with a widened field of view.** **Cause:** the engine scales mouse
    speed by player field of view / camera field of view whenever the camera is wider; the client's camera had
    the player's wider view, the server's cameras never went over the default 1.1, so the two simulations turned
    by different amounts and the server corrected the client. **Fix:** on both sides compute the scale with
    min(camera field of view, 1.1). Confirmed by the human.
48. **A weapon's magazine flickers empty - full - empty after a reload.** **Cause:** the 2010 server's ammunition
    component starts the reload pause at the tick it is ASKED, and the engine asks again a few seconds into the
    pause when the player switches weapons; the 1.65 client counts the pause from the last shot. **Fix:** count
    it from the last shot on the server. A supply point refilling round by round is a second source of the same
    picture (still open for one aircraft).
49. **A late joiner sees no ring and no icon over neutral flags.** **Cause:** the renderer draws a control
    point's frame and its world icon only after the client's "set team" ran for it; the round's reset runs it for
    every point, but a client joining later gets a team only where it differs from the first state - a neutral
    point never. **Fix:** the client's "team was set" query answers yes. Confirmed by the human.
50. **Bots throw grenades with no enemy in sight.** **Cause:** this game's hand grenade fires the moment it is
    taken in hand (fire on enable, no delay), and the library's item choice in plans that are not combat - on
    the way to a vehicle, a medic going to a wounded man on a ladder - can pick it. **Fix:** each plan may take
    out only its own item (the medic his medic kit, the resupply plan its ammunition bag), and in combat a thrown
    item only while the target is within that item's reach (grenades were thrown at vehicles 70-90 m off).
51. **The library faults when told of a hit.** **Cause:** its per-side lists are indexed by the side without a
    check, and a vehicle's AI object had side 0 in the tick a human got in. **Fix:** tell only hits whose shooter
    and victim are on side 1 or 2, and log the ones left out - that log named the cause.
52. **Direct3D 12 through D3D9On12: a black window, no loading picture.** `Direct3DCreate9On12` makes the game run
    on Direct3D 12 in full screen. In the game's own window nothing reaches the screen (the frame is in the back
    buffer; a test popup with the same parameters shows it); a device made by plain Direct3D 9 under the
    D3D9On12 object crashed the renderer. **Fix:** decide by the command line - a client started in a window gets
    plain Direct3D 9. With anisotropic filtering forced, return the game's own values when it reads its sampler
    states back, and leave render targets and depth / shadow formats alone.
53. **A post-processing shader that compiles with today's compiler is refused by the game.** **Cause:** the game
    compiles its effects at run time with its own old D3DX, which rejects a function with a return inside an
    `if` (fxc accepts it). **Fix:** branch-free code (step / lerp), and test every shader with the game's own D3DX
    DLL in a small harness before it goes into the archive. A runtime on / off switch: a sampler stage the effect
    does not use, bound by the plugin to a 1 x 1 texture, read by the shader as a blend factor.
54. **Shadows end 100 m from the camera** (a vehicle has one, the next one a little farther has none). **Cause:**
    four cascades in one map, each a fixed share of the coarsest square, and the coarsest square IS the shadow
    distance. **Fix:** a longer distance with the two nearest squares kept where they were (5 and 12.5 m) and the
    third between them and the farthest. Every share must stay a whole number: the cascades' centre moves in whole
    pixels of the coarsest square, which must be whole pixels of every cascade too, or the shadows crawl.
55. **One side's commander goes silent halfway through a round.** **Cause:** the library draws a strategy at
    random, weighted, among those whose conditions hold - and a "flag difference smaller than -1" is strict, so a
    side behind by exactly one flag had no strategy at all. **Fix:** thresholds half-way between whole numbers
    (+-0.5) and a catch-all strategy with an always-true condition and a tiny weight; check that every flag
    balance leaves at least one strategy. The running strategy cannot be asked from the console - log the
    commander's orders instead.

## Cost and time
About twenty-six days of sessions (2026-10-03 to 10-28), most of it reverse engineering and measuring; from
10-22 several agent sessions at once (infantry, vehicles, general AI, research), each owning its part of the
source and telling the others before it builds; no API spend.

## Open questions
- The first AIDLL fault on one Rush map (a garbage navigation-map object) has no known cause yet.
- Whether the "soldier goes with his kit" check (Gotcha 20) and a crash while joining (a customization update
  for an object that is not there) were the same wrong seating as Gotcha 19 - they have not come back yet.
- The plugin's own seating step stays for now: measured over 28 presses, the engine takes a bot in by his own
  "use" within about 1.3 m of a bipod gun's entry point (radius 1 m in its files; 2142's fixed guns have 2.2 m),
  and AIDLL sometimes presses up to 1.8 m short of the place, once, while still walking - one press in ten.
- The human sees bots of his own side fire at him at the start of a round while he sits in a vehicle; the log
  of whom the bots fire at has shown the horn and two shots at enemies past him, nothing aimed at him yet.
- Abilities rebuilt on the server (grenade dropped on death, throw-back grenade) are unverified with a human.
- Ground vehicle gunners sense almost nobody. Helicopter pilots rarely fire while the host flies a circle.
- Where the client builds the HUD's clip rectangle (Gotcha 38) - with it the corner groups could go to the edges.
- Bots enter a vehicle, leave and enter again. The library's data gives every vehicle kind the same weights
  (the "change vehicle" behaviour far above idling); crew weight sets per kind are installed, their effect is
  not proven - a counter of what plan a bot had when it left a seat is the next measurement.
- Once, a client started in a window had Direct3D 9 refuse to make its device three times (each with fewer
  hardware flags) and then crashed; the next start was fine. The error code is logged now.
- The library's own squad formation is alive (about two thirds of the asks say "keep your slot"), but its slots
  were some 26 m from the squad leader the host chose - whom it anchors on is not known.
