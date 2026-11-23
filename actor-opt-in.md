---
layout: default
title: SaveHub: Making things saveable
---

[Home](./) · [Actor opt-in guide](actor-opt-in.html) · [API reference](api.html) · [FAQ](faq.html) · [Changelog](changelog.html) · Support: [Discord](https://discord.gg/VNmNpzVAEB)

---

# Making things saveable

Pick the row that matches what you want to save.

| You want to save | Use |
|---|---|
| A placed or spawned actor (door, pickup, switch) | **SaveHub Saveable** on the actor |
| A component on such an actor | **SaveHub Saveable** on the component too |
| The player pawn and controller | Setting **Save Player** plus Saveable on the pawn or a component |
| A subsystem, manager or plain object | **SaveHub Participant** + Register Participant |
| A class you cannot edit | A **Save Profile** in the settings |
| Another plugin's fixed-name save file | **Mirrored Legacy Slots** in the settings |

## Actor in 3 steps (Blueprint)

1. Class Settings > Interfaces > Add > **SaveHub Saveable**.
2. Tick **SaveGame** (Instance Editable is not needed) on each variable to keep.
3. Add the event **On Saveable Loaded** and refresh whatever the variable drives (mesh pose, light, text).

No registration and no save code. On a level load the event runs before BeginPlay, so also run your refresh in
BeginPlay for a fresh start. The Blueprint sample door shows both.

## What gets saved

- SaveGame variables of the actor.
- The transform, if the root component is movable.
- Saveable components with their actor. A component on an actor without the interface is ignored.
- A placed actor destroyed during play stays destroyed after load.
- A runtime-spawned actor is spawned again by class on load.

## Tips

- Give placed actors stable names; they are found again by name.
- Use **On Saveable Pre Save** when state lives somewhere that is not a variable (flush it into one).
- Streamed actors (World Partition) get their state when their cell loads. Their BeginPlay runs before the restore,
  so do the visual refresh in On Saveable Loaded.
- A spawner that saves and restores its own runtime actors tags them `ISaveHubSaveable::ExternallyOwnedTag`. SaveHub
  then neither saves them nor destroys them on load.

## Participant (your own systems)

1. Add the **SaveHub Participant** interface.
2. Implement **Get SaveHub Key** (unique, stable) and mark variables `SaveGame`.
3. **Register Participant** (BeginPlay or Initialize). **Unregister Participant** before destroying it.

Change your data layout? Return a higher **Get SaveHub Version** and migrate in **On SaveHub Loaded (SavedVersion)**.

## GameMode and GameState

The engine creates the GameMode and the GameState itself, so SaveHub does not respawn them. Save them as participants:
add the **SaveHub Participant** interface to the Blueprint, return a unique key from **Get SaveHub Key**, mark the
variables `SaveGame`, call **Register Participant** in Begin Play and **Unregister Participant** in End Play. The record is
applied to the GameState that registers after a load or a level change, so state survives travel. A GameMode exists on
the server only; a GameState exists everywhere, so decide where the data should be saved. The C++ demo
(`ASaveHubDemoGameState`, map `SaveHubDemo_Persistent`) counts pad entries this way and shows them on its HUD.

## Structs and changing your variables

- A struct variable marked `SaveGame` saves its members, arrays, maps and nested structs. A Blueprint struct is saved
  with all its members. For a C++ struct, mark the members `SaveGame` as well.
- The Blueprint sample has a Participant made in Blueprint: `BP_SaveHubSample_Scoreboard` (implements SaveHub
  Participant, registers in Begin Play, unregisters in End Play, counts the saves of the current game).
- Values are matched by variable name. An added variable keeps its default, a removed one is skipped and logged, a
  renamed one counts as removed plus added: keep the old variable and copy its value in **On Saveable Loaded**.
- For a change of layout that needs code (merging two variables, new units), return a higher **Get SaveHub Version** from
  a participant and migrate in **On SaveHub Loaded (SavedVersion)**.

## Save Profile (class you cannot edit)

Project Settings > SaveHub > Save Profiles. Pick the class (subclasses match too), list property names, optionally
parameterless functions to run before save, before load and after load, and an adapter class for the rest. Profiles on
a class and its base class merge, base first.

## Mirrored Legacy Slots

Add the slot name the other system passes to SaveGameToSlot. On save the file is copied into your slot; on load it is
written back before the level opens; on new game it is deleted. **Warning:** New Game, and loading a slot that has no
copy of that file, delete the mirrored file from disk. Back up files you care about first. **On Legacy Slot Restored**
tells the other system to reload.
