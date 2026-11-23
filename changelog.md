---
layout: default
title: SaveHub Changelog
---

[Home](./) · [Actor opt-in guide](actor-opt-in.html) · [API reference](api.html) · [FAQ](faq.html) · [Changelog](changelog.html) · Support: [Discord](https://discord.gg/VNmNpzVAEB)

---

# Changelog

## 1.0.0

First release.

- Slot-based saves: one container file per slot (`SaveHub_<name>.sav`, FormatVersion 1).
- World actors via the SaveHub Saveable interface; transform for movable roots; destroyed placed actors stay
  destroyed; runtime actors respawned by class.
- World Partition: per-world record, cell cache for stream-out and stream-in.
- Player record (pawn and controller), carried across level travel.
- Participants for your own systems, with data versioning for migration.
- Mirrored Legacy Slots for other plugins' fixed-name saves.
- Save Profiles and adapters for classes you cannot edit.
- Level-aware load and new game (`Load Slot And Open Level`, `...On Next Level`), autosave, coalesced save requests.
- Slot list with header values, without loading the slot.
- Copy Slot, Export / Import Slot Data (for cloud saves) and Verify Slot (integrity check).
- Slot thumbnails: capture the viewport or a render target, load as a texture; kept beside the slot, format unchanged.
- Backups: each save keeps the previous slot file (Backup Count, default 2, max 5); a damaged slot loads from the newest
  valid backup; List Slot Backups, Restore Slot From Backup.
- Export Slot As Text / Import Slot From Text: share a slot as one piece of text.
- GameMode and GameState save through the Participant interface (documented, tested, shown in the C++ demo).
- Inspect Slot: header, every record (key, version, size), thumbnail, backups and recoverability without loading the slot;
  `SaveHub.Inspect` console command.
- Async saving: Save To Slot (Async) node and Save To Slot Async; capture on the game thread, compression and file work
  on a worker thread, On Saved when the file is on disk. Autosave and Request Save use it.
- Compressed slot files (Oodle), container FormatVersion 2; FormatVersion 1 files load unchanged.
- Loads spread over frames: Load Frame Budget setting, Load Slot And Open Level (Async) and Load From Slot (Async) nodes,
  On Load Progress and On World Restored events; level-aware loads read the file while the level opens.
- Multiplayer: server-only world saves (clean refusal on clients), every player saved by unique net id and restored on
  join and rejoin, Local Slots for a client's own data.
- User profiles: Set Active Profile, Create / Delete / List Profiles; each profile has its own files.
- Level Redirects and Runtime Class Redirects for renamed maps and classes.
- Hardened reading: damaged or crafted files (also share codes) are refused without crashing.
- Blueprint structs: every member of a struct made in the Blueprint editor is saved (they have no SaveGame tick box).
- World Partition: after a load the player is held in place until the cells around them have streamed in.
- Mobile: files still being written finish when the app goes to the background or closes.
- Blueprint sample: Participant made in Blueprint (Scoreboard), Share as text / Import text in the menu, async save.
- Platforms: Windows, Linux and Android tested; `SaveHub.SelfTest` console command for device checks.
- Demo: C++ maps (persistent, World Partition) and a Blueprint-only sample. The Blueprint sample uses movable lights
  only (no lighting build needed); its menu fits 1280x720 and larger windows.
- 48 automation tests (`Automation RunTests SaveHub`), plus benchmarks (`Automation RunTests Bench`).
