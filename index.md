---
layout: default
title: SaveHub Documentation
---

[Home](./) · [Actor opt-in guide](actor-opt-in.html) · [API reference](api.html) · [FAQ](faq.html) · [Changelog](changelog.html) · Support: [Discord](https://discord.gg/VNmNpzVAEB)

Version 1.0 · Unreal Engine 5.8 · Tested on Windows, Linux and Android

---


Slot-based save orchestration for Unreal Engine 5. One container file per save slot, one call to save or load
everything: world actors, the player, your own systems, and saves of other plugins you cannot edit.

Blueprint first, C++ underneath. Calls work while the game is paused. The Async calls keep file work off the game
thread, and big loads can be spread over several frames. Multiplayer: the server saves the world and every player by
net id. Several user profiles can share one machine.

## What it does

| You want to save | How it joins | Needs |
|---|---|---|
| Doors, pickups, switches, spawned actors | **Saveable** interface on the actor | mark variables `SaveGame` |
| The player pawn, controller, their components | setting `Save Player` | pawn or its components implement Saveable (or a profile matches) plus `SaveGame` variables |
| A subsystem, component or plain object (inventory, quests, stats) | **Participant** interface + register | `SaveGame` variables + a unique key |
| The GameMode or GameState | **Participant** interface + register in Begin Play | `SaveGame` variables + a unique key |
| A plugin's own save file you cannot change | **Mirrored Legacy Slots** (settings) | the slot name that plugin uses |
| A class you cannot edit | **Save Profile** (settings) | property names to keep |
| Every player on a listen or dedicated server | automatic (setting `Save Players By Net Id`) | the same as the player record, plus the player state |
| A client's own data (settings, local progress) | **Local Slots** + Register Local Participant | a participant |

## Quick start (Blueprint)

1. Enable the plugin and restart the editor.
2. Make a door (or any actor) saveable: Class Settings > Interfaces > add **SaveHub Saveable**. Tick `SaveGame` on the
   variables to keep (for example `bOpen`). Optional events: *On Saveable Pre Save*, *On Saveable Loaded* (refresh
   visuals here; it runs before BeginPlay when a level loads).
3. In your menu or anywhere: **Get Game Instance Subsystem > SaveHub Subsystem**, then
   - **Save To Slot** `Slot1`
   - **Load Slot And Open Level** `Slot1`, level `None` (None = the current level)
   - **Start New Game And Open Level** `Slot1`, level `None`
4. Slot list: **List Slots** (newest first) gives name, UTC time, play time and your own key/values.

Open `SaveHubDemo_Persistent` and `SaveHubDemo_WorldPartition` under the plugin Content for a working C++ example
(F5 save, F9 load, N new game, 1/2/3 slot, Del delete), or `SaveHubDemo_Blueprint` for the Blueprint-only sample below.

## Blueprint sample (no C++)

Open `SaveHubDemo_Blueprint` under the plugin Content (`/SaveHub/Demo/Maps`) and press Play. It uses Blueprints only,
so you can copy any of it into your own project. Everything lives in `/SaveHub/Demo/Blueprint`.

| Asset | What it shows |
|---|---|
| `BP_SaveHubSample_Door` | **Make an actor saveable in 3 steps** (below). Walk into the trigger to open or close the door. |
| `BP_SaveHubSample_Counter` | A pad that adds one to a `SaveGame` int and shows it in a Text Render. |
| `BP_SaveHubSample_ColorPad` | A pad that cycles through four colors; the saved int picks the material and the label. |
| `BP_SaveHubSample_PushCube` | A physics cube. Push it, save, push it again, load: it returns to the saved spot. |
| `BP_SaveHubSample_Pickup` | A gold orb. Collect it and it stays gone after a load (a placed actor that was destroyed). |
| `BP_SaveHubSample_SpawnPad` | Step on it to spawn a crate at runtime (up to 8). Spawned crates come back by class on load; the pad's saved count shows how many. |
| `BP_SaveHubSample_Pawn` | The player character (capsule, walks with gravity: WASD, mouse look, Space jumps). It is saveable, so Load puts you back where you saved. |
| `BP_SaveHubSample_Scoreboard` | **A Participant made in Blueprint**: implements SaveHub Participant (key `Sample.Scoreboard`), registers itself in Begin Play and unregisters in End Play, counts the saves of the current game in a `SaveGame` int (On SaveHub Pre Save) and redraws its label in On SaveHub Loaded / Reset. Copy it for your own systems (quests, stats). |
| `WBP_SaveHubSampleMenu` | Menu: slot name field with **Save** (Save To Slot Async: the file is written on a worker thread), **New Game**, **Share as text** / **Import text** with a text box, a status line, and the list of slots. Save also asks for a slot thumbnail. |
| `WBP_SaveHubSampleSlotRow` | One row of that list: thumbnail, slot name, saved time, play time, **Load** and **Delete**. |
| `BP_SaveHubSample_PlayerController` | Creates the menu and the hint, and toggles the menu with **Tab** (it also shows and hides the mouse cursor). |
| `WBP_SaveHubSampleHint` | The always-visible tip at the top of the screen: "Press TAB to open the Save / Load menu". |
| `BP_SaveHubSample_GameMode` | Points the map at that controller and pawn. |

Keys: **Tab** opens and closes the menu (the controller uses a legacy Input Key event for it). Walk (WASD, mouse to look,
Space to jump) into the doors and pads, push the cubes by walking into them, collect orbs and spawn crates to change the world. Type a slot name,
press **Save**, change something, then press **Load** on the row. **Delete** removes the slot file (and its thumbnail).
**New Game** resets every Saveable actor. **Share as text** puts the slot named in the field into the box as text
(select it, Ctrl+A, Ctrl+C, send it to a friend); paste such a text into the box and press **Import text** to make it
the slot named in the field. Slot files are written to `Saved/SaveGames` as `SaveHub_<name>.sav`; the
row picture is `SaveHubThumb_<name>.sav` next to it.

Thumbnails: the menu calls **Request Slot Thumbnail** from its On Saved handler (`HandleSaved`) and redraws the list
from **On Thumbnail Saved** (`HandleThumbnailSaved`). The picture is the game view without the menu.

### The door pattern (copy this)

1. **Class Settings > Interfaces > Add > SaveHub Saveable** on your actor Blueprint.
2. Tick **SaveGame** on the variables to keep (the door has one: `bIsOpen`).
3. Add the event **On Saveable Loaded** and refresh whatever the variable drives. The door calls its
   `RefreshDoorVisual` function there and again in Begin Play, so the right pose shows after a level load and on a
   fresh start.

That is all: no registration and no save code in the actor. The menu only calls `Save To Slot`, `Load Slot And Open Level`
(level `None` = the current level), `Start New Game And Open Level`, `List Slots` and `Delete Slot` on the SaveHub
Subsystem, and listens to `On Saved` and `On Loaded` for its status line. Open the graphs in `WBP_SaveHubSampleMenu`
(`RefreshSlotList`, `HandleSaved`) to see each call; every graph section has a comment box.

The sample only needs the engine's basic shapes. To use it in your own level, set the GameMode override to
`BP_SaveHubSample_GameMode` (or reuse the controller), and drop doors and pads in.

## Loading during play: use the level-aware calls

Loading into the running level and then letting that level unload would overwrite the loaded data with the live
state. Use **Load Slot And Open Level** / **Start New Game And Open Level**. If your menu opens the level itself, call
**Load Slot On Next Level** / **Start New Game On Next Level** right before your own Open Level. Non-seamless travel only.


## More

| Page | Contents |
|---|---|
| [Actor opt-in guide](actor-opt-in.html) | Saveable, Participant, GameMode and GameState, structs, changing variables, Save Profiles, Mirrored Legacy Slots, World Partition tips |
| [API reference](api.html) | Every node, event, setting and the on-disk format |
| [FAQ](faq.html) | Why a load reverts, why the player respawns, performance, multiplayer, profiles, platforms, file locations |
| [Changelog](changelog.html) | Versions and format changes |

## Big worlds: async saving and loading over frames

- **Save To Slot (Async)** captures the actors on the spot (game thread), then compresses and writes the file on a worker
  thread. **On Saved** fires when the file is on disk. Autosave and Request Save use it too.
- **Load Slot And Open Level (Async)** reads the file while the level opens. With **Load Frame Budget** set
  (Project Settings > SaveHub, ms per frame), saved actors are applied over several frames; **On Load Progress** reports
  and **On World Restored** fires when the whole game is back (hide your loading screen there).
- Slot files are compressed (setting **Compress Slots**, on by default). Old uncompressed files keep loading.
- Measured numbers: see "Performance" in the [FAQ](faq.html).

## Multiplayer

The server owns the world. On a listen or dedicated server, Save To Slot / Load From Slot save and restore the world
and every connected player (keyed by unique net id). A player who joins later or rejoins gets their state back when
their pawn is possessed. Loaded state reaches clients through normal replication (replicate what clients must see).
On a client, the world calls are refused (return false, one log line); use **Save Local Slot** / **Load Local Slot** for
the client's own data. Host migration is not supported.

## User profiles

**Set Active Profile** switches every slot, local slot, thumbnail and backup to that profile's own files; the default
profile (empty name) uses the 1.0 file names. **Create / Delete / List Profiles**. The last active profile is active again
at the next start.

## Known limits

- Non-seamless travel for the level-aware load and new-game calls.
- Capturing the world (reading the actors) runs on the game thread; only the file work is moved off it.
- Host migration is not supported.
- Tested on Windows, Linux and Android. File access goes through Unreal's platform save system; Mac, iOS and consoles
  are not tested.

## Support

Discord: https://discord.gg/VNmNpzVAEB
