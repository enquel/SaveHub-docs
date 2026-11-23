---
layout: default
title: SaveHub API reference
---

[Home](./) · [Actor opt-in guide](actor-opt-in.html) · [API reference](api.html) · [FAQ](faq.html) · [Changelog](changelog.html) · Support: [Discord](https://discord.gg/VNmNpzVAEB)

---

# SaveHub API reference

Everything public lives in `Source/SaveHub/Public`. All Blueprint nodes are under the category **SaveHub**.
Calls need no world and work while the game is paused. The plain calls finish before they return; the Async calls
(category **SaveHub|Async**) do the file work on a worker thread and report through events.

Get the subsystem in Blueprint with **Get Game Instance Subsystem > SaveHub Subsystem**, in C++ with
`USaveHubSubsystem::Get(WorldContextObject)` (null when there is no game instance or the hub has shut down).

## SaveHub Subsystem (`USaveHubSubsystem`)

### Saving

| Node | Returns | Meaning |
|---|---|---|
| Save To Slot (`SlotName`) | bool | Saves every participant, world actor and mirrored legacy save; the slot becomes current. False if the write failed. |
| Save Current Slot | bool | Saves to the current slot. False if no slot is active. |
| Request Save | none | Saves the current slot (async) at the end of this frame. Many requests in one frame make one save. |
| Save To Slot Async (`SlotName`) | bool | Captures now; compresses and writes on a worker thread. On Saved fires when the file is on disk. False (and On Saved false) if refused at once. |
| **Save To Slot (Async)** node | On Completed (`SlotName`, `bSuccess`) | The same as a latent Blueprint node. |
| Is Save In Progress | bool | An async save is still writing. |

### Loading and new game

| Node | Returns | Meaning |
|---|---|---|
| Load Slot And Open Level (`SlotName`, `LevelName`) | bool | Opens the level (None = current) and loads the slot while it opens. False and no travel if the slot is missing. **Use this to load during play.** |
| Start New Game And Open Level (`SlotName`, `LevelName`) | none | Opens the level (None = current) and starts a new game in the slot. |
| Load Slot On Next Level (`SlotName`) | bool | For a menu that opens the level itself: call right before your own Open Level. |
| Start New Game On Next Level (`SlotName`) | none | Same, for a new game. |
| Load From Slot (`SlotName`) | bool | Applies the slot to what is registered right now. If the running level unloads afterwards, it captures its live state over the loaded data. Prefer the level-aware calls. |
| Start New Game (`SlotName`) | none | Empties the current game, resets every participant, clears mirrored legacy saves. Writes nothing to disk. |
| **Load Slot And Open Level (Async)** node | On Progress, On Completed | Load Slot And Open Level that completes at On World Restored, with progress while actors are applied over frames. |
| Load From Slot Async (`SlotName`) / **Load From Slot (Async)** node | bool / pins | Read on a worker thread, actors applied within Load Frame Budget per frame. Same warning as Load From Slot. |
| Is Load In Progress / Get Load Progress | bool / 0..1 | A load is reading or applying over frames. |

Level-aware loads use non-seamless travel. They read the slot file on the worker while the level opens.

### Multiplayer

| Node | Returns | Meaning |
|---|---|---|
| Can Save World | bool | False on a network client. There, every world call (save, load, new game, their level-aware forms) is refused with a log line. |
| Save Local Slot / Load Local Slot / Delete Local Slot / Does Local Slot Exist / List Local Slots | | Slots of this machine only (`SaveHubLocal_<SlotName>`), separate from the world slots, in every net mode. |
| Register Local Participant / Unregister Local Participant | | Participants saved in the local slots. |

On a server, every player's pawn, controller and player state (their saveable parts) is a record keyed by the player's
unique net id (record `SaveHub.Players`). Settings: Save Players By Net Id, Restore Rejoining Player Transform.

### Profiles

| Node | Returns | Meaning |
|---|---|---|
| Set Active Profile (`ProfileName`) | bool | Every slot, local slot, thumbnail and backup is now that profile's. Empty = default profile. Clears the current slot; remembered for the next start. |
| Get Active Profile | string | Empty for the default profile. |
| Create Profile / Delete Profile / List Profiles / Does Profile Exist | | Delete removes all files of the profile (not the active one). |

Profile names: letters, digits, `_` and `-`, up to 32 characters.

### Slots

| Node | Returns | Meaning |
|---|---|---|
| List Slots | array of Slot Meta | Newest first. Only files with the configured prefix. |
| Get Slot Meta (`SlotName`) | bool + Slot Meta | Reads the header without loading the slot. |
| Does Slot Exist (`SlotName`) | bool | True if the slot file exists. |
| Delete Slot (`SlotName`) | bool | True when the slot is gone afterwards, **including when it never existed**. Use Does Slot Exist to tell the cases apart. |
| Has Current Slot / Get Current Slot | bool / string | The slot last saved, loaded or started. |
| Get Play Time Seconds | double | Play time of the current game, restored on load. |
| Set Slot Meta Value (`Key`, `Value`) | none | Written to the header with the next save. Shows up in List Slots. |
| Get Slot Meta Value (`Key`) | string | Value set for the current game, empty if unset. |

### Copy, export, verify

| Node | Returns | Meaning |
|---|---|---|
| Copy Slot (`SourceSlot`, `DestSlot`) | bool | Copies a slot, with its thumbnail, over `DestSlot`. The source is validated first; the current game and current slot are untouched. False for a missing or unreadable source, an invalid name, or the same name twice. |
| Export Slot Data (`SlotName`) | bool + bytes | Validated slot bytes, for a cloud service or your own file handling. |
| Import Slot Data (`SlotName`, `Bytes`) | bool | Writes exported bytes into a slot, replacing it. Damaged or truncated bytes are rejected whole and nothing is written. Does not load the slot. |
| Export Slot As Text (`SlotName`) | bool + text | The slot as one piece of Base64 text, for a chat message or the clipboard. Same content as Export Slot Data. |
| Import Slot From Text (`SlotName`, `Text`) | bool | Writes a slot from that text (spaces and line breaks are ignored), replacing it. Damaged or truncated text is rejected whole and nothing is written. Does not load the slot. |
| Verify Slot (`SlotName`) | bool + error | True if the slot passes every integrity check (magic, format version, size, CRC) and can be loaded. The error text says what is wrong. |

### Backups

Set **Backup Count** (Project Settings > SaveHub, default 2, max 5). Each Save To Slot first copies the previous slot
file into a side file (`SaveHubBackup<n>_<SlotName>`, n = 1 is the newest), so Slot files and their format are
unchanged. Only a valid file becomes a backup. If the slot file is later damaged, Load uses the newest valid backup by
itself (logged as a warning); a slot you deleted stays deleted. Delete Slot removes the backups, List Slots ignores them.

| Node | Returns | Meaning |
|---|---|---|
| List Slot Backups (`SlotName`) | array of Slot Meta | Headers of the valid backups, newest first. |
| Restore Slot From Backup (`SlotName`, `BackupIndex` = 1) | bool | Replaces the slot with a backup (1 = newest valid, as listed). Validated first. Does not load the slot. |

### Diagnostics

| Node | Returns | Meaning |
|---|---|---|
| Inspect Slot (`SlotName`) | Slot Report | What the file holds without loading it: valid or why not, recoverable from a backup, file size, header, every record (key, data version, bytes), thumbnail, backup count, and all of it as text in `Summary`. |

Console: `SaveHub.Inspect <SlotName>` prints the summary to the log.

### Thumbnails

A thumbnail is a small PNG kept in a side file next to the slot (`SaveHubThumb_<SlotName>`), so the slot file and its
format version are unchanged. Delete Slot removes it, Copy Slot copies it, List Slots ignores it.

| Node | Returns | Meaning |
|---|---|---|
| Request Slot Thumbnail (`SlotName`, `MaxWidth` = 320) | bool | Captures the game viewport (no UI) at the end of this frame. Call it right after Save To Slot. False if there is no game viewport or a capture is pending. The result arrives in On Thumbnail Saved; a capture that never arrives times out after 5 seconds with `bSuccess` false. |
| Save Slot Thumbnail From Render Target (`SlotName`, `RenderTarget`, `MaxWidth` = 320) | bool | Stores a render target (for example from a Scene Capture 2D). |
| Does Slot Thumbnail Exist (`SlotName`) | bool | |
| Load Slot Thumbnail (`SlotName`) | Texture 2D | Null if the slot has none. Feed it to an Image widget. |

`MaxWidth` is clamped to 16..1024; larger images are scaled down, never up.

### Participants

| Node | Meaning |
|---|---|
| Register Participant (`Participant`) | Object must implement SaveHub Participant and return a unique key. False if it does not. |
| Unregister Participant (`Participant`) | Call before the object is destroyed. |

### Events

| Event | Fires |
|---|---|
| On Pre Save (`SlotName`) | Before a slot is written. |
| On Saved (`SlotName`, `bSuccess`) | After a save. |
| On Pre Load (`SlotName`) | Before a slot is applied. |
| On Loaded (`SlotName`, `bSuccess`) | After a load. With the level-aware calls it runs in the new world before any actor's BeginPlay and before saved actor state is applied. |
| On New Game (`SlotName`) | After participants were reset. |
| On Legacy Slot Restored (`SlotName`) | A mirrored legacy save was written back or removed; its owner may need to reload it. |
| On Thumbnail Saved (`SlotName`, `bSuccess`) | A Request Slot Thumbnail capture finished. |
| On Load Progress (`SlotName`, `Progress`) | Once per frame while saved actors are applied over frames. |
| On World Restored (`SlotName`) | The loaded game is fully in the world: every actor has its state, the level plays and, in World Partition, the cells around the player streamed in. After every successful load. |
| On Profile Changed (`ProfileName`) | After Set Active Profile. |

With the Async calls On Saved fires when the file is on disk and On Loaded when everything is applied.

## Slot Meta (`FSaveHubSlotMeta`)

`SlotName`, `SavedAtUtc` (UTC), `PlayTimeSeconds`, `FormatVersion` (container format the file was written with),
`Values` (your key/values from Set Slot Meta Value).

## SaveHub Saveable (`ISaveHubSaveable`)

Interface for actors and their components. SaveGame variables (and the transform of an actor with a movable root)
are saved with the level. Details: [Actor opt-in guide](actor-opt-in.html).

| Event | Meaning |
|---|---|
| On Saveable Pre Save | Just before the variables are written: flush pending state into them. |
| On Saveable Loaded | After saved state was applied. On a level load it runs before BeginPlay; a load into a running level, or a respawned runtime actor's components, runs after it. Refresh visuals here. |

Actor tag `ISaveHubSaveable::ExternallyOwnedTag`: SaveHub leaves a runtime actor with this tag alone.

## SaveHub Participant (`ISaveHubParticipant`)

Interface for any object that keeps state: subsystem, component, actor, plain object.

| Function | Meaning |
|---|---|
| Get SaveHub Key | Unique, stable name of the record in the slot. Changing it orphans existing saves. |
| Get SaveHub Version | Version of your data layout. Default 1. Passed back on load. |
| On SaveHub Pre Save | Flush pending state into SaveGame variables. |
| On SaveHub Loaded (`SavedVersion`) | After the record was applied. Migrate old layouts here. |
| On SaveHub Reset | After SaveGame variables were reset to defaults (new game, or the slot has no record for this key). |

C++ only: override `WriteSaveHubRecord`, `ReadSaveHubRecord`, `ResetSaveHubRecord` for a custom format. Default is the
object's SaveGame properties.

## SaveHub Profile Adapter (`USaveHubProfileAdapter`)

Abstract, Blueprintable. Subclass it for logic a Save Profile cannot express. Events: On Pre Save, On Pre Load,
On Post Load (each gets the `Target`). A short-lived instance is created per call: keep no state in it.

## Settings (`USaveHubSettings`)

Project Settings > Plugins > SaveHub. Stored in `DefaultGame.ini`.

| Setting | Default | Meaning |
|---|---|---|
| Slot File Prefix | `SaveHub_` | Prefix of every slot file. Changing it hides existing slots. |
| Backup Count | 2 | Previous versions kept per slot (0 = off, max 5). |
| User Index | 0 | Platform user. |
| Autosave Interval Seconds | 0 | Saves the current slot every N seconds. 0 = off. |
| Mirrored Legacy Slots | empty (see FAQ) | Fixed-name saves of other systems, kept in step with the slot. |
| Save World Actors | on | Saveable actors, per level. |
| Save Player | on | Local player's pawn and controller as one record. |
| Restore Player Transform | on | Only when the loaded level is the level it was saved in. |
| Save Profiles | empty | Classes made saveable from config: class, property names, pre-save / pre-load / post-load functions, adapter. |
| Compress Slots | on | New files are compressed (Oodle) on the worker thread. Either kind loads. |
| Load Frame Budget Ms | 0 | Milliseconds per frame for applying saved actors in async and level-aware loads. 0 = at once, before BeginPlay. |
| Save Players By Net Id | on | Servers: one record per player, keyed by unique net id. |
| Restore Rejoining Player Transform | on | A rejoining player is put back where they left (same level). |
| Level Redirects | empty | Renamed maps: old package -> new package, so old slots restore the renamed map. |
| Runtime Class Redirects | empty | Renamed classes of runtime-spawned actors: old class path -> class to spawn. Engine class redirects work too. |

## On-disk format

Slot files are `Saved/SaveGames/<prefix><SlotName>.sav`; a thumbnail, if any, is `SaveHubThumb_<SlotName>.sav` (PNG
bytes), backups are `SaveHubBackup<n>_<SlotName>.sav` (full copies of an older slot file). The container carries a FormatVersion (currently 2: optional
Oodle compression, CRC32 over the stored bytes); version 1 files load unchanged. Local slots are `SaveHubLocal_<SlotName>`
(side files `SaveHubLocalThumb_`, `SaveHubLocalBackup<n>_`). A profile puts `<Profile>~` after each prefix, for example
`SaveHub_Anna~Slot1`. The profile list is `SaveHubProfiles`. The file prefix, the player record key `SaveHub.Player`
and older format versions stay readable across releases; a format change comes with a migration.

Every file goes through the engine's platform save system (`ISaveGameSystem`), so the location follows the platform:
`Saved/SaveGames` in the editor and on desktop, the app's own storage on Android. A packaged game logs it with the
console command `SaveHub.SelfTest save` (Development builds).
