# QuanCompanions: feature list (draft for review)

Status: draft, 1 Nov 2026. Nothing is agreed or built yet. Named QuanCompanions on 2 Nov 2026 (was the working name QuanCompanions).

QuanCompanions would replace three mods on our server:
- **BetterTames 0.9.0** (xtavim): tame behaviour.
- **LetMeTameYou 0.1.0** (Meldurson): which creatures can be tamed, and how.
- **LetMeTameYouServerFix 1.0.1** (Qua8ion): dedicated-server setup for LetMeTameYou.

It would also cover what we lost by removing **TeleportCreatures 0.3.3** (OdinPlus).

How to use this doc:
1. Go through Parts A–C and mark each feature **Keep**, **Drop** or **Change**.
2. Pick extras from Part D.
3. Turn the agreed list into `design.md`.

Every feature comes with the design principle that applies to it, taken from what we learned in the 1 Nov investigations.

## Design principles

These come from the incidents of 1 Nov 2026: about 200 tames moved to a death spot, about 50 deer stuck in "recovering" for days, and the portal and recall problems. Every feature below follows them.

| # | Principle | Why (what went wrong) |
|---|---|---|
| **P1** | **Identify players by ID, never by name. Reject 0 or blank everywhere.** On the server, take the requester's identity from the network connection, never from the request itself. | BetterTames matched a blank name against every tame with no leader. MuchBetterTames' server recall would match ID 0 against every object in the world. |
| **P2** | **Only act on tames near a player.** The only exception is an explicit, validated recall of *that player's own* followers. Every scan checks "is this a tame" first, and none touches the whole world on an event. | Tames 4–5 km away were moved because the code could reach them. |
| **P3** | **Never take ownership as a side effect.** Change a creature only on the PC that owns it, and send a request to the owner instead of grabbing it. Work correctly whoever the owner is: a player, the server, or a player far away. | Tim's PC owned two bases through the old chunk loader. E presses on tames owned by a distant player were silently ignored. |
| **P4** | **Our state runs from our own tick, never from AI methods.** Knock-outs, timers and recovery use the world clock in our own update, the same for `MonsterAI`, `AnimalAI` and anything a mod adds. | Recovery hooked `MonsterAI.UpdateSleep`, which LetMeTameYou's deer AI never calls. |
| **P5** | **Every saved state times out and repairs itself.** An expired timer counts as finished, and old or orphaned flags are cleaned up on load. Nothing can be permanently invulnerable, stuck or "following". | Deer at 1 HP stayed invulnerable for days. "Following" links stuck around and pulled tames back through portals. |
| **P6** | **Decide where the facts are.** Weapon and attacker checks happen on the attacker's PC, and the result travels with the hit or RPC. | The butcher knife was judged on the owner's PC, which can't see what a remote player is holding. |
| **P7** | **One hook for every kind of teleport.** Portals, TargetPortal map trips, dungeon entrances and respawns all go through `Player.TeleportTo` or the matching vanilla transition. | BetterTames' portal following never fired with TargetPortal. |
| **P8** | **Don't replace vanilla components. Add to them.** Make creatures tameable by adding `Tameable` and `Procreation` and patching their existing AI, not by swapping the AI class. | LetMeTameYou's swapped AI broke BetterTames' recovery, and would break any mod that hooks `MonsterAI`. |
| **P9** | **Server and clients set up prefabs identically.** Every prefab change (tameable flags, offspring, eggs) happens the same way on the dedicated server and on every client. | Before the fix mod, the dedicated server deleted newborn fawns as unknown prefabs. |
| **P10** | **One job per mod, declared.** QuanCompanions owns tame behaviour and portal transport for tames. Overlapping mods are declared incompatible, or their overlapping feature is detected and switched off. | BetterTames and TeleportCreatures both moved tames on portal trips. |
| **P11** | **Messages say exactly what's true.** For example "No *nearby* tames following you", and hover text that shows the real state. | "No tames following you" appeared while 21 were still saved as following. |
| **P12** | **Log what we move and why.** The server logs every teleport, recall and knock-out with counts and reasons, at info level. A debug switch adds a reason for every creature skipped. | During the incident nothing logged anything; we had to rebuild events from saves and our chunk loader diagnostic. |
| **P13** | **No full-world scans on events.** Keep an index (followers by leader ID, tames by zone) updated as things change, and spread any unavoidable scan over several frames. | MuchBetterTames scans about 1.45 million objects on every recall. |
| **P14** | **Fit our chunk loader.** Our new loader leaves tames paused when no player is near (see QuanChunkLoader `design.md`, "Tames"). QuanCompanions never assumes the server is running tames. | The incident only happened because a loader kept distant tames live on one client. |
| **P15** | **Migrate existing worlds.** Read and convert the data the old mods left behind, then clean it up. | We have LetMeTameYou deer and fawns, BetterTames recovery flags and stale follow links in the live world. |
| **P16** | **Only save links that survive a restart.** Never save an object ID (ZDOID) to point at a ship, a player or another creature: the game gives every object a new ID each time the world loads. Save a player ID, or find the object again by position. If a saved link can't be resolved, treat it as broken and undo whatever depended on it. | PerfectPetTransit saves the ship's object ID on a boarded pet. After a server restart that ID points at nothing, but the pet's AI stays switched off until someone disembarks it by hand. |

---

## Part A: replacing BetterTames 0.9.0

### A1. Protection and health

| ID | Feature (what BetterTames does) | Principle and how we'd do it | Decision |
|---|---|---|---|
| A1.1 | A protected tame that would die is knocked out at 1 HP instead. | **P4, P5:** the knock-out is a timestamp in the ZDO, ended by our own tick on the owner, and self-clearing if it's overdue. | ☐ |
| A1.2 | Gets up at full health after a set time (60 s). | **P4:** world-clock based, so a tame that was unloaded catches up when it loads. Configurable health on waking. | ☐ |
| A1.3 | While knocked out: takes no damage, ignored by enemies, doesn't eat or breed. | **P5:** "no damage" only while the timer is running. Expired means normal. | ☐ |
| A1.4 | The butcher knife still kills a protected tame. | **P6:** decided on the attacker's side and sent with the hit. The game already only lets the knife hurt tames when PvP is off. | ☐ |
| A1.5 | Alt+E turns protection on or off for the tame you're looking at. | **P3:** a request to the owner, with no ownership grab. The flag lives on the tame. | ☐ |
| A1.6 | "Protect all tames" server default, with per-tame override. | Server-synced, admin-locked. | ☐ |
| A1.7 | BetterTames: health bar name is blue when protected, yellow while knocked out. | **Changed (Quandru, 2 Nov 2026):** name colours now show the relationship (see D4.8), so protection gets **no name colour**: it's a separate state, shown elsewhere (hover text). Knocked out: to decide whether it still overrides the colour. | ☐ |
| A1.8 | Downed tames can be hovered and targeted (raycast helper). | Keep it simple. Needed so you can first-aid or butcher them. | ☐ |
| A1.9 | Tames heal from empty to full in a set time (1 h); healing pauses while hungry. | **P4:** our own tick, not the AI's regeneration. | ☐ |

### A2. Commands

| ID | Feature | Principle and how we'd do it | Decision |
|---|---|---|---|
| A2.1 | E makes any tame follow or stay, not only wolves and lox. | **P8:** sets `m_commandable`, using vanilla's own command path. **P1:** the saved leader is a player ID, alongside vanilla's name field so vanilla code still works. | ☐ |
| A2.2 | Alt+F: every tame in range with no leader follows you (30 m). | **P2:** loaded tames in range only. **P3:** commands go to each owner. | ☐ |
| A2.3 | Alt+G: every tame in range following you stays (30 m). | Same as A2.2. | ☐ |
| A2.4 | Follow/stay range setting. | Server-synced. | ☐ |

### A3. Following and teleporting

| ID | Feature | Principle and how we'd do it | Decision |
|---|---|---|---|
| A3.1 | A following tame teleports to you when it falls too far behind (50 m). | **P1, P2:** your followers by ID, loaded near you only. Safe landing behind you (see D3.2). | ☐ |
| A3.2 | After you die and respawn, your followers come to where you respawned. | **P1, P2, P13:** a server-validated recall of *your* followers, from the follower index, after spawn. Never while the character is being removed. | ☐ |
| A3.3 | Alt+H teleports every follower to you. | **P1, P11:** message says "N teleported" or "none of your followers are nearby/known". Distant followers come through the validated recall. | ☐ |
| A3.4 | Followers come into dungeons and back out. Option to leave big tames (lox) outside. | **P7:** dungeon transitions through the same teleport handling. Configurable exclusion list. | ☐ |
| A3.5 | Tames near you come through portals with you. | **P7:** hooks `Player.TeleportTo`, so TargetPortal works. See also Part C. | ☐ |

### A4. Leveling

| ID | Feature | Principle and how we'd do it | Decision |
|---|---|---|---|
| A4.1 | A tame that kills an enemy has a chance (10%) to gain a star, up to 2. | Owner-side, on the kill. | ☐ |
| A4.2 | Bonus chance per star the enemy has over the tame (+5%). | Same as A4.1. | ☐ |
| A4.3 | Healed to full on a new star. | Same as A4.1. | ☐ |

### A5. Feeding

| ID | Feature | Principle and how we'd do it | Decision |
|---|---|---|---|
| A5.1 | Hungry tames eat from player-built chests in range (5 m) when there's no food on the ground. | **P3:** taking food from a chest needs the chest's owner, so request it, don't grab. **P4:** not tied to one AI type. | ☐ |
| A5.2 | Creatures being tamed eat from chests too. | Same as A5.1. | ☐ |
| A5.3 | Option: only use chests that hold nothing but food. | Same as A5.1. | ☐ |
| A5.4 | Never touches dungeon or world chests. | Same as A5.1. | ☐ |

### A6. Taming and breeding tuning

| ID | Feature | Principle and how we'd do it | Decision |
|---|---|---|---|
| A6.1 | Taming time for every creature (30 min), modded ones included. | Global default, overridable per creature (B2). | ☐ |
| A6.2 | Love points needed for pregnancy (3). | Same as A6.1. | ☐ |
| A6.3 | Growth time for offspring (30 min). | Same as A6.1. | ☐ |
| A6.4 | Crowd limit: breeding stops with N of the same kind nearby (10 within 10 m). | Same as A6.1. | ☐ |
| A6.5 | Chance a newborn is one star above its parent (25%), up to 2. | Same as A6.1. | ☐ |
| A6.6 | Knocked-out tames don't breed and don't count as partners. | **P5:** only while knocked out. | ☐ |

### A7. Timers and hover

| ID | Feature | Principle and how we'd do it | Decision |
|---|---|---|---|
| A7.1 | Time left to tame, shown as paused while hungry or frightened. | **P11.** | ☐ |
| A7.2 | Time left in a pregnancy, or love points. | **P11.** | ☐ |
| A7.3 | Time left before offspring grow up. | **P11.** | ☐ |
| A7.4 | Time left for a knocked-out tame. | **P5, P11:** never "0 s" forever. | ☐ |
| A7.5 | Protection prompt in the hover text. | **P11.** | ☐ |

### A8. Combat behaviour

| ID | Feature | Principle and how we'd do it | Decision |
|---|---|---|---|
| A8.1 | Tames fight wild creatures of their own kind that are attacking (wolf against wolf). Calm ones and ones being tamed are left alone. | Owner-side AI targeting patch on `BaseAI.IsEnemy`, which works for any AI. | ☐ |

### A9. General

| ID | Feature | Principle and how we'd do it | Decision |
|---|---|---|---|
| A9.1 | Lock configuration: only admins can change server settings. | Server-synced, admin-locked. | ☐ |
| A9.2 | Debug mode. | **P12:** the server logs moves and knock-outs by default, and debug adds per-creature reasons. | ☐ |

---

## Part B: replacing LetMeTameYou 0.1.0 and LetMeTameYouServerFix 1.0.1

### B1. Taming more creatures

| ID | Feature (what LetMeTameYou does) | Principle and how we'd do it | Decision |
|---|---|---|---|
| B1.1 | Make almost any creature tameable (deer, leeches, skeletons, necks, greydwarfs, bosses and more), from a list. | **P8:** add `Tameable` and `Procreation` to the prefab and patch its existing AI. **Don't swap `AnimalAI` for a `MonsterAI` subclass.** **P9:** set up identically on the server. | ☐ |
| B1.2 | Animals that use `AnimalAI` (deer and others) get their AI replaced with LetMeTameYou's own `MonsterAI` subclass so they can follow and eat. | **P8: replace this.** Give `AnimalAI` the follow, eat and calm behaviour through patches instead. This is the root of the stuck-deer bug. | ☐ |
| B1.3 | Adds a `Humanoid` component to creatures that lack one (needed for eating). | **P8:** only if unavoidable. Prefer patching the consume path. | ☐ |
| B1.4 | Remove taming from a creature that's normally tameable (`removeTameable`). | Per-creature setting. | ☐ |
| B1.5 | Maximum wild level that can be tamed (`maxTamingLevel`, −1 = any). | Server-synced. | ☐ |
| B1.6 | Tamed bosses stop being bosses: no boss bar, not aggravatable. | Per creature. | ☐ |
| B1.7 | Creatures a tame spawns when it dies (for example blobs splitting) are spawned tamed. | **P2:** only for that tame's own drops. | ☐ |

### B2. The tame list (YAML)

| ID | Feature | Principle and how we'd do it | Decision |
|---|---|---|---|
| B2.1 | YAML tame list with per-creature entries and reusable `group_` blocks applied in order (a default group, food groups, tier and size groups). | Server-synced, and the server's copy wins. Validate on load and log errors clearly (**P12**). | ☐ |
| B2.2 | Several prefabs in one entry (`A,B:`). | Same as B2.1. | ☐ |
| B2.3 | `tamingTime`, `fedDuration`. | Same as B2.1. | ☐ |
| B2.4 | `commandable` (follow and stay). | Replaces vanilla's per-species flag; see A2.1. | ☐ |
| B2.5 | `consumeItems`, `consumeSearchRange`, `consumeSearchInterval`, `consumeRange` (what it eats and from how far). | Same as B2.1. | ☐ |
| B2.6 | `procreation`, `pregnancyChance`, `pregnancyDuration`, `maxCreatures`, `size` (crowd range multiplier), `growTime`. | Overlaps with A6. One setting per thing, with per-creature overrides. | ☐ |
| B2.7 | `offspringName` and `offspringOverwrite` (name or replace the young prefab). | **P9.** | ☐ |
| B2.8 | `requiredWorldKeys`: taming gated behind bosses or world keys; shows "Feral" until met. | Same as B2.1. | ☐ |
| B2.9 | `hostileToSelf`: a tamed creature fights wild ones of its kind. | Overlaps with A8.1. Merge them. | ☐ |
| B2.10 | `specialGroup`: creatures in the same group never fight, tamed or wild. | Same as B2.1. | ☐ |
| B2.11 | `followDistMulti`: per-creature follow distance. | Same as B2.1. | ☐ |
| B2.12 | `trade`: tame by handing over items instead of feeding (unfinished in LetMeTameYou). | Same as B2.1. | ☐ |

### B3. Breeding, young and eggs

| ID | Feature | Principle and how we'd do it | Decision |
|---|---|---|---|
| B3.1 | Creates a young prefab for each new tameable (for example `MiniDeer` and fawns): the adult model, smaller. | **P9:** registered identically on server and clients. **P15:** keep existing young prefab names, so current fawns load. | ☐ |
| B3.2 | Egg-laying creatures: copy an egg type (drake or chicken) with hatch time, colour and size (`egg: drake(1200:#00FF00:0.3)`). | **P9.** | ☐ |
| B3.3 | Dragon eggs hatch into hatchlings, with a hatch time (1800 s). | Same as B3.2. | ☐ |
| B3.4 | Egg hover text shows hatch info. | **P11.** | ☐ |
| B3.5 | Records when a creature last procreated, for hover display. | **P11.** | ☐ |

### B4. Taming tool and hover

| ID | Feature | Principle and how we'd do it | Decision |
|---|---|---|---|
| B4.1 | Craftable taming tool (raw meat, carrot, mushroom, 5 resin at a workbench); recipe and station configurable. | Same as B3.2. | ☐ |
| B4.2 | Holding the tool shows a taming overlay on creatures (progress, food, status). | Could merge with A7. | ☐ |
| B4.3 | Increased interact distance with the tool (20 m). | Same as B4.2. | ☐ |
| B4.4 | Wider hover text box to stop word wrap. | Same as B4.2. | ☐ |
| B4.5 | "Feral" instead of "Wild" when world keys are missing. | See B2.8. | ☐ |

### B5. Smaller tweaks

| ID | Feature | Principle and how we'd do it | Decision |
|---|---|---|---|
| B5.1 | Once tamed, stays fed longer (`TamedFedMultiplier`, ×4). | Server-synced. | ☐ |
| B5.2 | Global follow distance multiplier. | Same as B5.1. | ☐ |
| B5.3 | Hides tiny "+0" heal numbers. | Same as B5.1. | ☐ |
| B5.4 | Tamed creatures eat through a patched consume path. | **P8:** works for every AI type. | ☐ |
| B5.5 | Sends its config and version from server to clients, and refuses a mismatched client. | Jotunn's config sync and version check, like our other mods (see `docs/DEVELOPMENT.md`). A YAML tame list would sync as a registered custom config or a Jotunn custom RPC. | ☐ |

### B6. What the server fix adds

| ID | Feature (LetMeTameYouServerFix) | Principle and how we'd do it | Decision |
|---|---|---|---|
| B6.1 | The dedicated server runs the same tame setup as clients, so it stops deleting newborns and dropped taming tools as unknown prefabs, and server-run tames keep their tamed AI. | **P9:** built in from the start, so no separate fix is needed. | ☐ |
| B6.2 | Logs every tame set up on the server, with its offspring prefab, and warns if an offspring prefab is missing. | **P12.** | ☐ |

---

## Part C: what we lost with TeleportCreatures 0.3.3

| ID | Feature | Principle and how we'd do it | Decision |
|---|---|---|---|
| C1 | Tames come through portals with you, including TargetPortal map trips. | **P7.** Overlaps with A3.5. One implementation. | ☐ |
| C2 | Modes: all tamed, only followers, only named, all except named. | Per-player choice within server limits. | ☐ |
| C3 | Search radius and height around you, capped by the server. | Normal config settings, not just YAML. | ☐ |
| C4 | Per-portal rules (by portal tag or prefab) with allow and deny lists. | Same as C3. | ☐ |
| C5 | Dungeon and Basements transitions. | **P7.** | ☐ |
| C6 | Live counter of allies that will come while you stand in a portal. | **P11.** | ☐ |
| C7 | Safe placement at arrival (radius, spacing, height, water rules). | See D3.2. | ☐ |
| C8 | Optional enemy transport, or blocking portals with enemies nearby (off for us). | Same as C3. | ☐ |
| C9 | Diagnostics: per-creature reasons for skipping. | **P12.** | ☐ |

---

## Part D: extras from other mods (pick what you want)

Only features that the mods we're replacing don't already have. The source is in brackets.

### D1. Protection and health

| ID | Feature | Principle | Pick |
|---|---|---|---|
| D1.1 | First aid: press E on a knocked-out tame to shorten its recovery [MuchBetterTames] | **P3:** request to the owner. | ☐ |
| D1.2 | Faster recovery with a player within 15 m [MuchBetterTames] | **P4.** | ☐ |
| D1.3 | Recovery time shrinks per star [MuchBetterTames] | **P4.** | ☐ |
| D1.4 | Separate healing rates in and out of combat; well-fed tames heal faster [MuchBetterTames] | **P4.** | ☐ |
| D1.5 | Young can't be butchered by accident [TameableDeer] | **P6.** | ☐ |
| D1.6 | Tames inside an active ward take no damage from fire, poison or monsters [ArcadeHelper] | Owner-side. | ☐ |
| D1.7 | Opt in per tame by putting a symbol in its name (🛡️) [Pet Protection] | An alternative to A1.5. | ☐ |
| D1.8 | Tames that die if left hungry too long [TameDiesIfNotFed] | Off by default. | ☐ |

### D2. Commands

| ID | Feature | Principle | Pick |
|---|---|---|---|
| D2.1 | One key toggles all nearby tames: if any follow, all stay; otherwise all follow [MuchBetterTames] | **P2.** | ☐ |
| D2.2 | "Advance": followers move ahead in the direction you face for 3 s [Follower Commands] | **P2.** | ☐ |
| D2.3 | Order tames to attack the target you're hovering [WolfPack] | **P3.** | ☐ |
| D2.4 | Call tames by name or wildcard lists (for example only wolves) [WolfPack] | Same as D2.3. | ☐ |
| D2.5 | Tap E to pet, hold E to command [TameableAnimals, HoldToCommand] | Same as D2.3. | ☐ |
| D2.6 | Young can follow you too [LetsGo, BabyTames] | Same as D2.3. | ☐ |
| D2.7 | Per-creature exclusions from commands [TameCommands] | Same as D2.3. | ☐ |
| D2.8 | Key hint for the group command in hover text [MuchBetterTames] | **P11.** | ☐ |
| D2.9 | Controller bindings [MuchBetterTames, Follower Commands] | Same as D2.8. | ☐ |
| D2.10 | Tell tames to stand down and stop fighting, within a calm radius [PassiveWolves] | Same as D2.8. | ☐ |
| D2.11 | Radial menu (Shift+E on a tame): follow/stay, rename, saddle, board/disembark, with number keys 1–5 as shortcuts [PerfectPetTransit] | **P3:** each choice is a request to the owner. PerfectPetTransit takes ownership instead. | ☐ |

### D3. Following and teleporting

| ID | Feature | Principle | Pick |
|---|---|---|---|
| D3.1 | Catch-up after a delay (25 s) rather than instantly, including out of pits and across rivers [Heel] | **P2.** | ☐ |
| D3.2 | Arrivals in an arc behind you, spaced out, with ground, water, wall and ward checks [Heel, MuchBetterTames] | Safety. | ☐ |
| D3.3 | Stuck detection: teleport a follower that hasn't moved closer for a while [MuchBetterTames] | Same as D3.2. | ☐ |
| D3.4 | Longer teleport distance during combat (90 m), so tames aren't yanked mid-fight [MuchBetterTames, Heel] | Same as D3.2. | ☐ |
| D3.5 | Parked tames avoid fire, lava, smoke, traps and deep water [Heel] | Owner-side. | ☐ |
| D3.6 | Big tames keep a gap so they don't push you; walk through tames in dungeons [BetrTame] | Same as D3.5. | ☐ |
| D3.7 | Cap on how many followers come through a portal [TameableAnimals, TeamKoro BetterTames] | **P2.** | ☐ |
| D3.8 | Recall includes distant, unloaded followers, server-validated [Follower Commands] | **P1, P13**, as in A3.2. | ☐ |
| D3.9 | Bonding gate: a species follows through portals only after a per-player bond [TameBond] | Progression. Optional. | ☐ |
| D3.10 | Tames pull carts [LetsGo, BeastsOfBurden] | Separate feature. | ☐ |

### D4. Display

| ID | Feature | Principle | Pick |
|---|---|---|---|
| D4.1 | On-screen counter of how many tames follow you [MuchBetterTames] | **P11.** | ☐ |
| D4.2 | Health bars of *your* followers tinted, visible only to you [MuchBetterTames] | **P11.** | ☐ |
| D4.3 | Health shown as numbers, with taming and XP progress [BetrTame] | **P11.** | ☐ |
| D4.4 | Taming progress messages to nearby players at 25/50/75% [ServerTaming, ServersideQoL] | **P11.** | ☐ |
| D4.5 | Your tames on the minimap and map, with no pins added [TamedRadar] | **P11.** | ☐ |
| D4.6 | A panel listing all your tames, wherever they are [VikingDex] | **P2, P13:** server index. | ☐ |
| D4.7 | Why an animal is or isn't breeding, shown on hover [NearbyPanel] | **P11.** | ☐ |
| D4.8 | **Name colours by relationship** (Quandru, 2 Nov 2026), so you can pick out your followers in a crowd at a glance: **white** for wild, **green** for tamed, **blue** for following you. Protection has no colour (A1.7). Vanilla never colours names. | **P1, P11:** "following you" is decided by your player ID, on your own screen. Another player's followers show green (tamed). | ☐ |
| D4.9 | **Your followers always show their health bar, out to 20 m** (Quandru, 2 Nov 2026), persistent with no fade. Vanilla shows a creature's name and bar within 10 m and only for 60 s after you last hovered over it (`EnemyHud.TestShow`, `m_hoverShowDuration`); tames already get the green "friendly" bar. For creatures following you: 20 m and no time limit. | **P11.** Client-side display only. | ☐ |

### D5. Leveling

| ID | Feature | Principle | Pick |
|---|---|---|---|
| D5.1 | XP-based stars (from enemy health, level and boss) instead of a random chance [MuchBetterTames, BetrTame] | Owner-side. | ☐ |
| D5.2 | XP shared with your followers nearby; your own kills give them XP [MuchBetterTames, BetrTame] | **P1.** | ☐ |
| D5.3 | Stars above 2 (up to 30), with configurable health, damage, size and loot per star [BetrTame] | A balance decision. | ☐ |
| D5.4 | Stars passed to young [BetrTame] | Same as D5.3. | ☐ |
| D5.5 | Level-up effect and speech bubble [MuchBetterTames] | Same as D5.3. | ☐ |

### D6. Taming and breeding

| ID | Feature | Principle | Pick |
|---|---|---|---|
| D6.1 | Taming only progresses with a player nearby, so no taming from loot left lying [BetrTame] | **P14.** | ☐ |
| D6.2 | No young born while the parent is following you [BetrTame] | Same as D6.1. | ☐ |
| D6.3 | Newborns stay inside their pen [BreedingInPen] | Same as D6.1. | ☐ |
| D6.4 | Young of new species in a different colour tone [TameableAnimals] | **P9.** | ☐ |
| D6.5 | Natural sounds for petting, mating and birth [TameableDeer] | Same as D6.4. | ☐ |
| D6.6 | Passive animals stop fearing you once tamed, but still flee monsters [TameableAnimals] | **P8:** an `AnimalAI` patch. | ☐ |
| D6.7 | Affection from petting, required before breeding [TameCraft] | A balance decision. | ☐ |
| D6.8 | Global relative multipliers per species instead of absolute values [TameCraft] | Simpler config. | ☐ |
| D6.9 | Sterilised offspring, to stop runaway breeding [ZenBreeding] | Same as D6.8. | ☐ |

### D7. Feeding

| ID | Feature | Principle | Pick |
|---|---|---|---|
| D7.1 | A buildable feeding trough or bowl that animals eat from as if from the ground [TameableAnimals, Feeding Trough mods] | An alternative to A5. **P3.** | ☐ |
| D7.2 | Per-chest feed range set from a sign on the chest [ServersideQoL] | Same as D7.1. | ☐ |
| D7.3 | Always leave at least N of each food in a chest [ServersideQoL] | Same as D7.1. | ☐ |

### D8. Other

| ID | Feature | Principle | Pick |
|---|---|---|---|
| D8.1 | Wild-creature defence: anything that attacks a player or a tame becomes your tames' enemy [MuchBetterTames] | Owner-side. | ☐ |
| D8.2 | Tamed creatures don't damage player buildings [FriendlyLoxFriendly] | Same as D8.1. | ☐ |
| D8.3 | Server-only operation for vanilla and console clients [ServersideQoL, ServerTaming] | Not a goal for us: every player runs the modpack. | ☐ |
| D8.4 | Translations [MuchBetterTames: 14 languages] | Same as D8.3. | ☐ |
| D8.5 | Admin commands to safely recall a tame from anywhere [RecallTames] | **P1, P12.** | ☐ |

### D9. Pets on boats

All from PerfectPetTransit 0.0.59 (yadren1y, first released 1 Nov 2026). There's no source repository, no config file, and its messages are in Russian only. In vanilla, tames on a ship slide about or jump overboard as soon as a threat appears.

| ID | Feature | Principle | Pick |
|---|---|---|---|
| D9.1 | **Embark:** one command puts the tame onto a ship within 10 m, on a free deck spot laid out per ship and species, facing forward. It then moves with the ship. | **P3:** the tame's owner moves it. **P16:** link to the ship in a way that survives a restart. PerfectPetTransit saves the ship's object ID, which doesn't. | ☐ |
| D9.2 | **Disembark:** finds flat, clear ground 3–12 m from the ship (5 m minimum for a lox), no lower than the deck and no more than 4 m above it, and puts the tame there. If there's nowhere safe it refuses and says so. | **P11.** | ☐ |
| D9.3 | **Tethered on board:** the tame stays put during a fight instead of jumping into the sea, and a rope is drawn from its neck to the mast. It can still be hurt. | **P4, P5.** PerfectPetTransit switches off the whole AI while the link is set, which also stops anything that runs from the AI, such as recovery. That's the same trap as LetMeTameYou's deer. We'd pin the position and leave the AI running, or pause only movement. If the ship can't be found, release the tame. | ☐ |
| D9.4 | No bump damage between a tame and its own ship (collisions between the two are ignored) | Owner-side. | ☐ |
| D9.5 | **Capacity per ship.** Points: raft 1, karve 2, longship 6, drakkar 11. Boar 1, wolf 2, asksvin or moose 3, lox 11. Each species also has its own cap on each ship (for example a longship takes at most 2 wolves). | **P9:** limits set by the server. PerfectPetTransit hard-codes them and has no entries for our LetMeTameYou species (deer, bears and so on). | ☐ |
| D9.6 | A boarded tame won't follow until it has been disembarked | **P11:** a clear message. | ☐ |

How it would fit our other choices:
- **Chunk loader (P14).** A ship docked in a loaded zone with no player near is run by the server. Boarded tames must stay put however the ship's owner changes. That needs a test.
- **Portals and recall (P2, P7).** A boarded tame must not be pulled through a portal or recalled while it's tied to the ship. It should be disembarked first, or skipped with a message.

---

## Migrating the live world (P15)

These need handling whatever we pick:
- **Deer and other LetMeTameYou tames** are saved with `tamed = true`. Our setup must make those same prefabs tameable before the world loads, or they behave as wild.
- **Young prefabs** such as `MiniDeer`, and any fawns now in the world, must keep their names, or the game deletes them as unknown on load.
- **BetterTames recovery flags** (`BetterTames_RecoverUntil`, `BetterTames_Protection`): 52 tames are stuck. On load, clear an expired flag and restore health.
- **Stale "following" links** (`follow` names): convert to player IDs where the player is known, and drop the rest.
- **LetMeTameYou's own ZDO keys** (`LMTY_TameStep`, `LMTY_ProcTime`): ignore or clean up.

## Open questions

- ~~Name the mod.~~ QuanCompanions (2 Nov 2026). Its Thunderstore description must include "tame", "taming" and "breed": search matches descriptions as well as names.
- Keep the YAML tame list format so the existing file can be reused, or move to our own format?
- Portal transport in QuanCompanions, or a separate module? P10 says one owner.
- Which creatures to support at launch: just the ones we've tamed (deer, leeches, skeletons, bears), or the whole LetMeTameYou default list?
