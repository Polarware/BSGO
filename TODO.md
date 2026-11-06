# TODO

Ordered by what unblocks the most. Each item says what is actually wrong, not just what to build.

## Broken now

### The sector-entry overlay still sticks

"Perform any action to enter the sector" was fixed on 4 Aug: both visibility gates accept the
never-started state, and `VisibilityTimer` lifts the ghost after ten seconds. It is still happening
in a live client (reported 5 Aug).

What is ruled out: the sector was healthy (29, ticking normally, the player was firing and looting),
`VisibilityTimer` is registered, and the log carries zero `ghost jump lifted from NOT_STARTED`
traces. So the lift is almost certainly being sent and the client is latching anyway. Next step is a
log line on the send itself plus a reproduction, because right now the server side looks correct and
that cannot both be true.

## Unflown

Nothing here is proven until it flies.

### From 5 Aug, the outpost and NPC pass

- **Outposts shoot in all directions.** Their four missile launchers carry a 90 degree arc against
  hardpoints mounted three to starboard and one to port, so only ever one hemisphere could answer.
  Stations now ignore firing arcs entirely. **Check:** sit off an outpost's bow and its stern, and
  confirm all four launchers engage from both.
- **Capitals fight beam-on.** The capital cannon arc went from 180 (meaning "anywhere") to 65, and
  NPC capitals now steer 90 degrees off the bearing and circle instead of pursuing nose-first.
  **Check:** watch a Kraken or Poseidon engage. Does it circle at a sensible radius, or wander?
  This is the change most likely to look wrong.
- **Player carriers are affected too.** Brimir, Pegasus, Galactica, Surtur, Basestar and Guardian
  share that cannon and those flank hardpoints, so they must now turn to bring guns to bear.
  **Check:** whether a capital still feels worth its rental. If not, the bosses can move to their own
  cannon card and leave the players' alone.
- **Flak kills no longer fake a hit.** `ObjectLeftHit` names what was struck, and for a missile
  killed by damage it was naming the killer, so your own flak swatting a warhead read to the client
  as that warhead hitting you. No damage was ever dealt. **Check:** shoot down a missile and confirm
  the screen no longer shakes.
- **Ring platforms repair after 15 minutes.** Both partial losses and full wipes. A full wipe used
  to hit an unconditional rebuild inside the 5-second tick, so hitting a ring harder brought it back
  faster. **Check:** clear a ring, confirm nothing returns for 15 minutes, then that all four do.
- **NPC wings stopped brawling on spawn.** Both wings shared one z band and differed only in the
  sign of x, leaving 1,600 m between their nearest edges against 1,000-2,500 m of auto-aggro, and
  the Cylon wing sat 1,285 m from the Colonial outpost. They now take opposite corners: worst
  wing-to-wing gap 2,973 m, worst wing-to-enemy-outpost gap 4,401 m. **Check:** that a contested
  system is quiet until someone starts something.
- **NPCs level out after a fight.** The levelling code sat below an early return that skipped every
  ship without patrol objectives. **Check:** no capitals cruising at 40 degrees of bank.

### From 5 Aug, the galaxy pass

- **Outpost control survives a restart.** Table `sector_outpost_states`, migration V8, written at
  shutdown and after every console fill or clear, restored at boot with the star-flag seeding as
  the fallback for sectors with no saved row. Verified server-side over a full stop and start.
  **Check:** capture something, restart, confirm it is still yours.
- **The console drives outposts.** `outpost_fill [colonial|cylon|both] [<sectorId>|all]`,
  `outpost_clear`, `outpost_status`, `outpost_save`. Both arguments default, so bare `outpost_fill`
  fills the galaxy. The underlying machinery is verified but the command dispatch itself has never
  been exercised, because that needs an admin client. **Check:** run `outpost_status` first. If it
  prints the per-sector table, the other three are on the same path.
- **Outposts moved to opposite corners**, 8,204 to 8,528 m apart on a regular system against 5,100
  before. Spawn corridors gained clearance rather than losing it: 1,378 m from the nearest outpost
  ring, up from 450. The cost is that your own outpost is now a corner trip from undock, roughly 3.5
  to 4 km instead of 1.7. **Check:** whether that trip is tedious.

### From 4 Aug, still unconfirmed

Flak screens thinning missiles in both directions is confirmed working, indirectly: the phantom
screen shake could only happen if flak was killing warheads. The rest still wants eyes on it.

- An outpost missile reading 200 HP when selected (it was 15).
- The Pegasus's guns refusing a selected missile while a Viper's still track one.
- A missile detonating ~260 units off the Brimir's hull rather than at it, and a missile actually
  hurting a gate basestar.
- Every bay on a Galactica rendering a turret.
- A rental surviving a relog with time on the clock.
- Locked systems greying out their jump button.
- Comets in sector 10.

Outpost Mode's fortress half is confirmed: `Fortify ON for player 1` in a live session on 5 Aug.

### From 13 Aug, the Outpost Mode beacon

Both halves exist and flew the same evening. What survived flight differs from what was built,
and the flight log rewrote the design:

- **The transponder jump works.** A live `RequestJumpToTarget` was accepted, charged, and landed
  the jumper 500 m off the fortified carrier. Verified end to end.
- **The green map flash is dead on arrival, client-side.** The type-10 galaxy updates reach the
  client and are applied every second - proven by type-9 updates riding the same batch behind
  them - but `SectorDesc.FilteredJumpTargetTransponders` still returns empty for any viewer
  outside the carrier's sector. Every server-side link was verified live (broadcast, wire bytes,
  aspect delivery re-pushed on each map open, subscription); an all-sector broadcast lit
  nothing. The one working flash all evening was a viewer standing inside the beacon's sector.
  RESOLVED 15 Aug by reading the decompiled client instead of flying experiments: the partyId=0
  wildcard was never the problem (`PartyId == 0` passes the filter outright, SectorDesc.cs:118),
  the type-10 updates were applying all along, and the gate really was the aspect - see the
  15 Aug section below for the mechanism and the fix. The party experiment is cancelled.
- **The beacon ICON channel works fleet-wide.** `SectorBeaconState` (type 9) renders the light
  blue jump-beacon icon on the carrier's sector tile for the whole faction, live within a
  second, cleared on unfortify. This is now the map indication for Outpost Mode; the code sits
  in `Galaxy.getJumpTargetTransponders` beside the transponder broadcast.
- **The beacon jump button: built 15 Aug, unflown.** Handler, range gate and the 1.7x price are
  in - see the 15 Aug section below.

- **Fortify lights the galaxy map.** Every fortified carrier is broadcast to its own faction on
  the 1s galaxy tick as a jump transponder, partyID 0 so the whole faction sees it. Clearing is
  free: the list is rebuilt from live fortify state each tick, so unfortify, death, dock and
  disconnect all clear it within a second with no cleanup code. **Check:** fortify and watch a
  second account's map green-flash. Your own map staying dark is correct; the client hides your
  own beacon.
- **Every pilot now carries the TransponderJump aspect**, pushed through the property buffer at
  spacing. The client filters beacons on the viewer's hangar ship having that aspect, and
  nothing ever sent it: the hangar ship only learns aspects from update type 13, which this
  server never wrote. **Check:** the jump button appears at all.
- **RequestJumpToTarget(75) has a handler.** No range check, deliberately: jumping to a beacon
  from anywhere is the feature. It refuses enemy base sectors, targets that are not fortified
  friendly carriers, and a requester that is itself fortified. Tylium is
  ceil(FtlCost x distance x 2.2), which is the price the client's button quotes. **Check:** jump
  from far beyond FTL range and compare the charge to the quote.
- **Arrival is a ring 500 m off the carrier**, radius growing 150 m per collision retry, and it
  beats the 15-minute stored-position replay. A carrier that unfortified or died during the
  charge costs the jumper nothing worse than the lost-transponder notice and a normal spawn
  point, so a stale beacon cannot strand anyone. **Check:** unfortify while a friend is
  mid-charge; they should land at the spawn area with the notice, not in limbo.

### From 15 Aug, the beacon flash diagnosis and the beacon hop

The client filter mystery is solved, from the decompiled client rather than a live session.
Both changes that follow from it compile and pass the suite; nothing here has flown.

- **The type-10 verdict from 13 Aug was a misdiagnosis.** GalaxyMapMediator applies every
  update type in the batch unconditionally - `star.JumpTransponders = descs`, no gate of any
  kind - and the type-9 icon landing proves it, because type-9 rides BEHIND type-10 in the same
  batch and a parse failure would have taken both. The descs reached every client all along.
- **The real gate: the hangar ship's aspect list, which the client wipes.** The filter asks
  `Game.Me.ActiveShip.HasAspect(TransponderJump)` - the HANGAR ship. Its aspect list is empty
  on construction, every `AddShip` batch rebuilds the hangar from scratch (login, dock, ship
  upgrade), and exactly one thing ever fills it: a Stats(32) message carrying update 13. Our
  server sent update 13 at spacing (rides the sector-join scramble) and on map subscribe (races
  whatever opened the map) - neither reliable. Hence the pattern: a fresh spacing next to the
  carrier flashed, everything else stayed dark. **The fix:** `writeHangarShipStats` now appends
  the update-13 aspect entry. That message already follows every hangar rebuild and every ship
  switch, so the aspect is re-delivered in the same breath that wipes it. Wire-checked against
  the client parser: same message id (32), same update type (13), same aspect value (4).
  **Check:** fortify, and from a second account far across the galaxy - after docking
  somewhere, which forces the wipe - open the map. The carrier's sector should green-flash and
  offer the jump button.
- **The beacon jump button works on paper.** `RequestJumpToBeacon(85)` has a real handler: it
  resolves the fortified friendly carrier in the target sector server-side (the wire names only
  the sector), refuses requests beyond FtlRange (the client only offers the button in range, so
  the server enforces the same line - this path is the range-gated hop, not jump-from-anywhere),
  and reuses the transponder arrival plumbing, stale-beacon fallback included.
  `JumpScheduleItem` grew the cost-multiplier field the 13 Aug notes called for: 2.2 on the
  transponder path, 1.7 on the beacon path, both matching the client's quoted price.
  **Check:** with a beacon icon lit within FTL range, press "jump to beacon"; confirm arrival
  500 m off the carrier and a charge of ceil(FtlCost x distance x 1.7). Try the own-sector case
  too - the client offers it, and a same-sector pass through the jump plumbing has never run.
- **Still open: the group variant.** `GroupJumpToBeacon` (msg 89) is in the ClientMessage enum
  but has no case in parseMessage, so a party leader's group-beacon window does nothing. The
  solo button is the built half.

**Flight results, 15 Aug evening.** Two accounts live, carrier fortified in sector 52:

- **The beacon button works.** Account 2 pressed it at 19:38 and landed off the carrier -
  `beacon jump to carrier ... in sector 52` in the server log. Range gate and 1.7x price rode
  along. First msg-85 jump ever on this server.
- **The green flash appeared mid-session** after the beacon jump respaced the viewer, having
  stayed dark through two map opens at 19:22-19:23 despite the server pushing the aspect on
  both (log shows `[Dogfight, TransponderJump]` sent twice). Reading: every server-side
  delivery fires, but whether the CLIENT applies a given aspect message is a race - each
  delivery is a lottery ticket, and today's additions (login, dock, switch) plus the spacing
  and map-open pushes make wins frequent but not guaranteed. The player-visible symptom of a
  lost race is the transponder channel (green flash + pilot-name jump buttons) going dark
  until the next respacing.
- **The CompleteJump re-send shipped the same evening** (server restart 19:52): msg 76 arrives
  precisely when the client's scene load is done, so every jump arrival now re-delivers the
  aspect at the one moment that cannot race the loader.
- **And the next test STILL found the pilot-name button missing.** The Vidofnir screenshot
  settled the entire chain: the beacon icon showed on 276 Exrun (type-9 applied, so type-10
  landed with it), the Sector button quoted 181 tylium and the Beacon button 308 - exactly
  ceil(180.9) and ceil(180.9 x 1.7) for the star-table distance - and the pilot was
  resource-capped, killing the affordability theory. Every term of the client filter is now
  proven good except one: `ActiveShip.HasAspect(TransponderJump)` reads false on a client the
  server has aspect-pushed six different ways. The decompile says those pushes must stick. The
  running client disagrees. Server-side work on this is exhausted.
- **So the fix moved into the client: GameBridge 0.9.29 carries an OPM watchdog**
  (OpmBeacon.cs in the Fulgar repo). Once a second it re-adds TransponderJump to the active
  hangar ship if missing - reflection into ShipAspects' private list, the same technique as
  the Anti-AFK clock pin - and logs a state line whenever the aspect, the stored descs or the
  filter output change. The heal makes the flash immune to whatever eats the aspect; the
  witness lines name the eater for the day it can be shot properly.
  **Check:** with 0.9.29 deployed, fortify and look for the green flash and the pilot-name
  button from ANY sector. Then read BepInEx/LogOutput.log for `[OPM]` lines. `aspect=MISSING`
  followed by `HEALED` means the watchdog is earning its keep and the eater is real;
  `aspect=ok` from login onward means the wire was always fine and earlier sessions raced
  something now closed.

### The OPM map design, settled 15 Aug

The pilot's call, matching the original game: OPM's map presence is the flashing green tile
plus a jump button carrying the carrier pilot's name, jump-from-anywhere at 2.2x. The "Beacon"
button and the static beacon icon belong to beacon items, not to fortified carriers.

The precondition is MET: at 21:43:35 on 15 Aug the full transponder loop ran live - green flash
visible from Vidofnir, pilot-name button pressed, 2.2x jump from 322 units (far beyond FTL
range), arrival in the carrier's sector. So, ready for the next server stop:

- **Stop emitting SectorBeaconState for fortified carriers** - one line in
  Galaxy.getJumpTargetTransponders. That removes the beacon icon AND the "Beacon"/group-beacon
  buttons for carrier sectors in the same stroke, since the client gates all of them on that
  state. The pilot's standing rule: the beacon channel belongs to actual beacon ITEMS used by
  a ship; OPM is the transponder channel only. The msg-85/89 handlers stay for the item
  feature.
- **Log galaxy-map subscribe and unsubscribe server-side.** A subscriber that dies mid-session
  is currently silent, and an open map never re-subscribes - that combination cost the last
  hour of 15 Aug ("no flash" with every other link healthy; closing and reopening the map
  fixed it instantly). Visibility first, then consider making the drop less trigger-happy
  than one failed send.

### Next for Outpost Mode: carrier docking for squad strikes

The remembered original behaviour: strikes in the carrier pilot's squad could dock inside the
fortified carrier and repair with no delay. Scoped 15 Aug from the decompiled client, and it is
much closer than expected:

- **The client already does most of it.** Dock-at-player-ship is the ANCHOR path: the dock
  button appears when the target PlayerShip's Owner card is IsDockable, the viewer's hull is
  TIER 1, the carrier's pilot is in the viewer's PARTY, and the viewer is inside DockRange
  (SpaceObject.CanDock, SpaceObject.cs:417). Pressing it sends `RequestAnchor(78)`.
- **The server handler is already written and deliberately parked.** GameProtocol's
  RequestAnchor case carries the full flow - party check, dockable+range check, visibility
  change, anchored-id bookkeeping, the Anchor reply and the party broadcast - behind an
  `if (true) return;` marked "ignore this for now" (GameProtocol.java:174). RequestUnanchor is
  implemented and live.
- **What actually needs building:** flip the six carrier hulls' Owner cards to
  `IsDockable: true` with a sane DockRange in cards.js:1771 (every hull is `false` today;
  outposts use 400 on a 250-unit model), remove the parked guard, then work out
  repair-while-anchored - the client-side tier/party gates come free. `RequestLaunchStrikes`
  (79) for the carrier pilot wants checking in the same pass.

### From 15 Aug late, the audit and its fix batch

A six-segment adversarially-verified audit of the OPM chain (server wire against the decompiled
client, both directions) returned 30 confirmed findings and root-caused all three open symptoms
in one pass. The fix batch below is applied, compiles clean, and passes the suite; none of it
has flown.

- **The beacon flash killer was a thread race in `writeHangarShipStats`.** It serialized the
  LIVE stats map - counting it, then iterating it - while the sector thread's applyStats
  cleared and rebuilt that same map under a lock the writer never took. The client reads
  exactly the declared count, so every entry behind the divergence vanished silently, most
  visibly the trailing aspect entry; a concurrent rehash could also kill the send with a CME
  and CLOSE THE CONNECTION. Fixed: `getStatsSnapshot()` copies under the lock, and the aspect
  entry now rides FIRST so no stat entry's client-side handler can take it down. This is why
  the same client worked against another server: theirs does not race its own stats pipeline
  during fortify. Also closed: the ship-UPGRADE path rebuilt the client hangar with no aspect
  follow-up at all (`selectShip` early-returns on the active slot).
- **The three fortify icons were stranded removals.** A forced release (death, dock, jump,
  disconnect) queued the RemoveBuff into per-subscriber buffers and then removed the ship from
  the only set the flush timer reads - composed, queued, discarded, every time. Fixed: the
  remover flushes synchronously right after unfortify. Two accomplices fixed with it: the
  update queue was an unordered HashSet (a fortify toggled on and off inside one 100ms tick
  could wire the remove BEFORE the add), and a resubscribe used to destroy a replaced buffer's
  queued updates.
- **The stuck "perform any action" overlay (4 Aug) was a double-send.** Every sector entry
  pushed the identical invisible state twice - constructor and CompleteJump - and the client
  stacks an overlay per push but lifts exactly one. Fixed: `startGhostJump` refreshes its
  window without re-issuing the unchanged state.
- **Every jump charged tylium twice.** A direct `decrementCount` AND the HoldVisitor deduction,
  so a 1.7x beacon hop really cost 3.4x - and a pilot who could afford the quote but not the
  double lost the money AND the jump with no message, with the map's jump buttons stuck hidden
  behind the client's never-reset jump-active flag. Fixed: single charge, pre-checked, and the
  first-ever sender of NotEnoughTylium(81), which is also what resets that client flag.
- **Sector 41's live `pitch is NaN` was the roll bug's sibling** - the time-to-peak divisions
  in moveToDirection divide by pitch/yaw acceleration, unguarded. Fixed with the same shape as
  rollCommand. The per-phase tick split contained it exactly as designed: sector 41 kept
  broadcasting beacons with movement down.
- **Hardening from the same audit:** unknown GameProtocol message ids no longer close the
  connection (they NPE'd the enum switch past the parse loop's typed catches); the galaxy
  broadcast survives a dead subscriber self-removing mid-loop (CME starved everyone after it
  in the map for that tick); the galaxy update lists are non-null from boot (a subscribe in the
  first second disconnected the subscriber); slot updates are filtered to card sectors (the
  client kills its whole batch on one unknown id - the sector-63 failure made structural);
  the lost-transponder notice sends a real sector guid for the first time (it always carried
  -1, so it has never displayed once); FTL charge times stopped truncating to whole seconds.
- **Still open from the audit:** the group beacon jump (msg 89) is offered by the client,
  silently dropped by the server, and leaves the party's client-side jump state stuck - it
  needs a real handler; the quick-reconnect window can still interleave sector flushes with
  sendCharacter's Reset-to-ActiveShip gap (rare, self-healing, documented); and two client-side
  quirks we can only note (SetInfo's transient SectorCaps NRE flicker, the quadratic button
  spacing in RepositionButtons).
- **Check, next flight:** fortify and confirm from the second account, from far across the
  galaxy: green flash, pilot-name button, jump lands 500 m off the carrier charged at
  ceil(FtlCost x distance x 2.2) - once, not twice. Fortify-cycle three times and confirm the
  plate shows one icon while fortified, zero after. Enter sectors repeatedly and confirm no
  00:00 overlay survives. The GameBridge witness stays deployed: `aspect=ok` with no HEALED
  lines is the server fix proving itself; the heal comes out once that is seen.

### From 15 Aug, the sector-10 NaN

The comet suspicion from 13 Aug was right. Fixed in three layers on 15 Aug; the chain is verified
by test and a clean build, but sector 10 has not flown since.

- **The cause: the comet's stats card had no flight stats.** Guid 23's NonShipStats carried only
  MaxHullPoints, MaxPowerPoints and Speed, so MovementController read every turn stat as 0, and
  the wasd roll term divided rollFading (400, from the Movement card) by RollMaxSpeed (0): 0/0 =
  NaN on the comet's first movement tick. The NaN survives every clamp because Mathf.clamp passes
  NaN through, enters the frame through Euler3's unvalidated setters, and two ticks later
  maxEulerSpeed builds an Euler3 whose yaw is NaN - the exact throw, from the exact 13 Aug stack.
  It repeated forever because the single-try tick died before its removal phase. A second defect
  hid behind the first: Acceleration was also absent, so a comet could never leave its spawn
  point. The card now carries the full flight block (cards.js regenerated, 12,250 validated).
  **Check:** boot, leave sector 10 alone for ten minutes, grep the log for `yaw is NaN`. Zero is
  the pass mark. Then look for comets actually crossing the sector.
- **wasd and moveToDirection now share a guarded rollCommand.** Roll options of 0 mean "this hull
  cannot roll" and produce a 0 command instead of a division by zero, so the next statless object
  handed a TurnManeuver drifts harmlessly instead of poisoning its sector.
  MovementSimulationTest reproduces the original comet setup, fails against the unguarded code,
  and passes with the guard.
- **Sector.run ticks in seven per-phase try blocks.** One bad object in movement no longer
  silences collision, abilities, timers, removal and zone management, and the error line names
  the failing phase, which would have pointed at the movement updater on day one.
  **Check:** any future per-tick error reads `crash in <phase> phase` with the other six phases
  still alive.

## Next

### Transponder consumables

The map-beacon half of Outpost Mode shipped on 13 Aug (see Unflown), but the deployable
transponder is still unbuilt: `ActivateJumpTargetTransponder` has no `AbilityActionFactory` case,
so `cards.js` keeps dropping the three jump-transponder systems, correctly, because the factory's
default branch throws inside the sector tick. The `JumpTargetTransponder` space object with its
arm timer exists as a class and is never spawned. Outpost Mode no longer needs any of it; build
it only if the consumable itself is wanted.

### Comets beyond sector 10

The timer only runs where a sector template asks for it, and only sector 10's does. Do a handful
first and watch for per-tick sector errors before turning it on galaxy-wide, since a comet kills any
ship it touches.

### Balance, once the above settles

Hull points span 850 on a Viper Mk II to 100,000 on a flagship, and the equipment ladder is the
original's own rather than a tier multiplier. Nobody has timed kills across tiers with a person in
the cockpit. The capitals are the open question, and the broadside arc has just changed the answer:
a twelve-bay Pegasus that has to turn to shoot is a different ship from one that did not.

Boss lairs pay six themed jackpots. Whether any of them is worth the fight is untested.

## Galaxy

All 58 accessible systems ship. Ids, names, guids, positions, threat levels and faction lockouts are
transcribed from the client; **sector contents are generated**. No source records where a single
asteroid sat in any system but 0, 6 and 10, so belts, clusters, junk fields and lairs are plausible
rather than recovered. The three upstream files are never regenerated.

`galaxy.js` holds the star table and `STAGE`, the number that grows the map; `emit-sector-templates.js`
writes the templates. They must move in the same commit: a star with no template on disk is a server
that does not boot.

On system size, since it comes up: the wiki says standard systems are 10,000 m across and only
Carillon, Exomera and 14 Toah are 40,000. That is what ships. Alpha Ceti's own upstream content
stops at 4,993 on x and 4,997 on z, which is a 5,000 half-extent to the metre. The one contradiction
is Tannhauser, whose upstream content reaches 7,368 and therefore cannot fit the wiki's figure,
which is why it carries a hardcoded 16,000 floor. `sizeM` in the star table is a blanket 10000 on
every row, so it is a placeholder rather than per-system data.

Still open here:

- Sector 10's hand-placed 1,360-asteroid field remains the only genuine sector content in existence.
  Anything that recovers more of the real layouts beats the generator.
- `SectorSlotData` sends 100/100 slots for every sector always, so a pilot sees a live jump button
  on a system the server will refuse, then a bare "sector not allowed" with no explanation.
- Comets stay off in generated sectors.

## Known-unfixable, documented so nobody re-investigates

**Ship scrapping.** `ScrapShip` is a logged no-op upstream and `RemoveShip` has zero cases in
`parseMessage`. Every hull is therefore `CanBeSold: false`, which is not a choice we made, and a
live footgun the day anyone wires it up, because `ContainerVisitor.sellItem` pays out on `CanBeSold`
alone with no second gate.

**Outpost retreat has no announcement.** `EmergencyMessage` does not carry text: the client wraps
whatever string it receives as `%$bgo.<s>.description%` and looks it up. Nothing in the client's
locale describes an outpost retreating, and no other broadcast writer renders literal text, so there
is no honest message to send. Passing a sentence renders the server-maintenance banner instead,
which is what used to happen on every outpost kill.

**Weapon hardpoint rotations are render transforms, not aiming axes.** Nearly every hull stores its
guns at 90 degrees off the nose, fighters included, because the rotation orients the muzzle flash.
Firing direction comes from the mount angle measured against that weapon's own `Angle` stat, which
is a deviation limit and not a cone width. Anything that reasons about where a ship's guns point has
to ask the second question, not the first. This already cost one session: a broadside check built on
mount angle alone classified every NPC in the game as a capital.
