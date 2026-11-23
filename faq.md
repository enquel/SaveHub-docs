---
layout: default
title: SaveHub FAQ
---

[Home](./) · [Actor opt-in guide](actor-opt-in.html) · [API reference](api.html) · [FAQ](faq.html) · [Changelog](changelog.html) · Support: [Discord](https://discord.gg/VNmNpzVAEB)

---

# FAQ

**My load did nothing / the level reverted to the live state.**
You called Load From Slot in a running level, and the level then unloaded and captured its live state over the loaded
data. Use **Load Slot And Open Level**, or **Load Slot On Next Level** right before your own Open Level.

**The actor does not save.**
It needs the SaveHub Saveable interface and `SaveGame` ticked on the variables. A component also needs the interface
itself. Check that Save World Actors is on.

**The player respawns at PlayerStart.**
A pawn is only saved if it, or one of its components, implements Saveable, or a Save Profile matches it. A plain
DefaultPawn has nothing to save. Restore Player Transform only applies when the loaded level is the level it was saved in.

**My visuals are wrong right after load.**
Refresh them in **On Saveable Loaded** (actors) or **On SaveHub Loaded** (participants), not only in BeginPlay.

**Streamed World Partition actors restore late.**
Their BeginPlay runs before the restore, by design. Use On Saveable Loaded.

**Delete Slot returns true for a slot that never existed.**
By design: true means the slot is gone afterwards. Use Does Slot Exist first to tell the cases apart.

**I changed the participant key and old saves lost the data.**
The key names the record in the file. Keep it stable. A rename needs a migration on your side.

**What is `SaveHubDemo_Legacy` in my settings?**
The plugin ships `Mirrored Legacy Slots = SaveHubDemo_Legacy` as a stand-in for the demo. Nothing else uses that name,
so it is harmless. Remove it if you do not ship the demo.

**Are slot files compatible between plugin versions?**
The container has a FormatVersion. Format changes ship with a migration and are listed in the [changelog](changelog.html).

**Can I save on a background thread / asynchronously?**
Yes: **Save To Slot (Async)**. Reading the actors (capture) runs on the game thread because it touches game objects;
compressing, the backups and writing the file run on a worker thread. On Saved fires when the file is safe on disk.

**Loading a big world hitches. What can I do?**
Set **Load Frame Budget Ms** (for example 4) and load with **Load Slot And Open Level (Async)**. Saved actors are then
applied over several frames; show a loading screen until On Completed / On World Restored. With a budget, actors begin
play with their defaults and get their state a few frames later (On Saveable Loaded tells each one).

**Performance**
Measured in the editor (Development build) on a Ryzen 9 7950X3D with an NVMe drive, an actor with eight SaveGame values
and a saveable component: see the table below. Your numbers depend on your actors and machine; every save and load logs
its own timing (`LogSaveHub`).

| Actors | Save: game thread (async) | Save: file work (worker) | Load, at once | Load over frames (4 ms budget): worst frame | File size |
|---:|---:|---:|---:|---:|---:|
| 100 | ~1.5 ms | ~1 ms | ~7 ms | ~1.3 ms | 3.4 KB |
| 1,000 | ~16 ms | ~3 ms | ~21 ms | ~4 ms (4 frames) | 28 KB |
| 5,000 | ~77 ms | ~16 ms | ~80 ms | ~4.2 ms (18 frames) | 133 KB |

A World Partition map with 5,008 saveable actors and real cell streaming: save ~85 ms game thread + ~12 ms worker,
load ~97 ms at once, 56 KB. Reading the actors (capture) is most of a save and runs on the game thread.

**Does it work in multiplayer?**
Yes, with the server in charge. On a listen or dedicated server the world and every player (by unique net id) are
saved and loaded; players who join or rejoin later get their state when their pawn is possessed; clients see loaded
state through replication. Clients cannot save the world (the calls return false) but can keep their own data in Local
Slots. Host migration is not supported. In Play In Editor all players share one machine, so give client local slots
different names when you test several clients.

**Can several people use one computer?**
Yes: **Set Active Profile**. Each profile has its own slots, local slots, thumbnails and backups.

**I renamed a map or moved a Blueprint. Are old saves lost?**
No, if you add a redirect: **Level Redirects** (old map package -> new) and **Runtime Class Redirects** (old class path
-> new class) in Project Settings > SaveHub. Engine class redirects (`[CoreRedirects]`) are followed too.

**Which platforms?**
Tested on Windows, Linux and Android. File access goes through Unreal's platform save system; Mac, iOS and consoles are
not tested.

**Android: saving fails on my phone.**
Android 11 and later do not let apps write to shared storage, where the engine puts game files by default. Enable
Project Settings > Platforms > Android > **Use ExternalFilesDir for UnrealGame files** (`bUseExternalFilesDir=True`) and
package again; saves then go to the app's own folder (`Android/data/<package>/files/...`). SaveHub logs this hint once
when a write fails on Android. This affects every save system, not only SaveHub.

**Slot names and upper/lower case**
Slot names may use letters, digits, `_` and `-` (1 to 64). Windows does not tell `Slot1` from `slot1`, Linux and
Android do: use one spelling per slot.

**Where are the files?**
`Saved/SaveGames/SaveHub_<SlotName>.sav` (prefix configurable) in the editor; a packaged Windows game uses
`%LOCALAPPDATA%/<Project>/Saved/SaveGames`, Android the app's own storage. A thumbnail is `SaveHubThumb_<SlotName>.sav`
next to it. In a Development build, the console command `SaveHub.SelfTest save` logs the folder.

**How do I put a screenshot in my slot list?**
Call **Save To Slot**, then **Request Slot Thumbnail** with the same name (it captures the game view without UI), and
show **Load Slot Thumbnail** in an Image widget. Capture before you open a menu on top of the game, or use **Save Slot
Thumbnail From Render Target** with a Scene Capture 2D of your own.

**A save file got damaged. Is the player's progress lost?**
Not if Backup Count is above 0 (default 2). Load uses the newest valid backup by itself. **List Slot Backups** and
**Restore Slot From Backup** let you pick one. **Inspect Slot** (or `SaveHub.Inspect <Slot>` in the console) shows what
is wrong and whether it is recoverable.

**What is inside my slot file?**
**Inspect Slot** lists the header and every record with its key, data version and size, without loading the slot.

**Can I save the GameMode or GameState?**
Yes, through the Participant interface (the engine creates both, so SaveHub never respawns them). See "GameMode and
GameState" in the [Actor opt-in guide](actor-opt-in.html).

**Do variables inside a struct get saved?**
Yes, when the struct variable itself is a `SaveGame` variable. Arrays, maps and structs inside structs are saved too.
A struct made in the Blueprint editor has no SaveGame tick box for its members; SaveHub saves all of its members
(tested). For a struct you write in C++, tick `SaveGame` on its members as well; the engine only writes nested C++
members that carry the flag.

**I added, removed or renamed a variable after players saved. What happens?**
Saved values are matched by variable name. A variable you added keeps its default. A variable you removed is skipped
(a warning is logged). A renamed variable is a removed one plus an added one, so its old value is not carried over: if
that matters, keep the old variable and copy it in **On Saveable Loaded** or **On SaveHub Loaded (SavedVersion)**.

**How can two players share a save?**
**Export Slot As Text** gives one piece of text (Base64) that can go through a chat or the clipboard; **Import Slot From
Text** writes it into a slot on the other player's machine. Damaged or cut-off text is rejected and nothing is written.

**How do I check a save is intact, copy it, or upload it?**
**Verify Slot** runs the integrity checks, **Copy Slot** duplicates a slot, **Export Slot Data** / **Import Slot Data**
move validated bytes to and from a cloud service.
