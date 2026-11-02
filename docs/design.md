# QuanCompanions: design (brief, draft)

Status: first draft, 2 Nov 2026. Nothing agreed or built. Valheim 1.0.21.

`feature-list.md` holds the full list of candidate features, each with the principle that applies. This doc is the
short version: what the mod is for, the rules it follows and what's still to decide. It grows into the full design
once the feature list has been reviewed.

## Purpose

One mod for tames on the Quanheim server, replacing:

| Mod | Role today | Why replace it |
|---|---|---|
| **BetterTames 0.9.0** (xtavim) | Protection and knock-outs, follow/stay for all, teleport with you, levelling, feeding from chests, breeding tuning. | On 1 Nov 2026 it moved about 200 tames to a death spot (blank-name follower match), and left 52 tames stuck in recovery. |
| **LetMeTameYou 0.1.0** (Meldurson) | Makes deer, bears, skeletons and more tameable, with young and eggs. | Swaps `AnimalAI` for its own AI, which breaks other mods' recovery: the stuck-deer bug. |
| **LetMeTameYouServerFix 1.0.1** (Qua8ion) | Makes the dedicated server set up the same prefabs. | Only needed because LetMeTameYou doesn't do it itself. |
| **TeleportCreatures 0.3.3** (OdinPlus), already removed | Tames came through portals with you. | Overlapped BetterTames and moved tames too. |

Its Thunderstore description must include "tame", "taming" and "breed", and name the mods it replaces: Thunderstore's
search matches descriptions.

## Rules it follows

The 16 principles in `feature-list.md` (P1–P16), from the 1 Nov incidents. The ones that shape the whole design:
- **Players by ID, never by name** (P1), and only act on tames near a player, except a validated recall of your own (P2).
- **Never take ownership as a side effect** (P3): requests go to whoever owns the creature.
- **Our own tick and self-repairing state** (P4, P5): nothing can stay invulnerable, stuck or following forever.
- **Add to vanilla components, don't replace them** (P8), set up identically on server and clients (P9).
- **One teleport hook for every kind of trip** (P7), and links that survive a restart (P16).
- **Fit the chunk loader** (P14): tames in a loaded base stay paused with no player near. QuanCompanions never
  assumes the server runs them.
- **Migrate the live world** (P15): today's deer, fawns, recovery flags and follow links must load correctly.

## How it would work (outline)

- **Everyone must have it**, server included: the server sets up the same tameable prefabs, keeps the follower index
  and validates recalls; the owner's machine runs each tame.
- Config is server-synced and admin-locked. Creature rules come from a tame list (format still to decide).
- The server logs every teleport, recall and knock-out with counts and reasons (P12).

## Ideas noted

- **Followers at a glance** (agreed 2 Nov 2026): name colours by relationship, **white** wild, **green** tamed,
  **blue** following you; protection has no colour. Creatures following you show their health bar out to 20 m,
  always, with no fade. Feature list D4.8, D4.9 and A1.7.

## Where I'd like your input

1. **The feature list review.** Mark each feature in Parts A–C Keep, Drop or Change, and pick extras from Part D.
   That's the main step before this becomes a real design. We can go through it section by section in chat.
2. **Tame list format.** Keep LetMeTameYou's YAML so the current file carries over, or our own format?
3. **Portals.** Portal transport inside QuanCompanions (P10: one owner), or a separate mod?
4. **Creatures at launch.** Just the ones we've tamed (deer, leeches, skeletons, bears), or LetMeTameYou's whole list?
5. **Pets on boats** (Part D9). In for the first release, or later?
