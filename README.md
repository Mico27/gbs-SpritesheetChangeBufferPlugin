# gbs-SpritesheetChangeBufferPlugin

**Version 4.3.0 — Requires GB Studio ≥ 4.3.0**

A GB Studio engine plugin that eliminates the one-frame visual glitch that occurs when using the **Set actor spritesheet** event. It does this by double-buffering the sprite VRAM allocation for each actor that has reserved tiles: new tile data is written into a hidden back slot, and only after writing completes does the OAM rendering switch to the new slot. The result is that the old sprite is still displayed on the same frame the write happens, and the new sprite appears cleanly on the next frame.

There are no new events or settings to configure — installing the plugin is all that is needed to activate the fix.

**Example without the buffer plugin:**

https://github.com/user-attachments/assets/f53a2621-c57a-45b0-8e64-609baf1817fa

**Example with the buffer plugin:**

https://github.com/user-attachments/assets/de925525-5b96-4b94-8fbd-7c280441bc4c

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [Technicalities and Restrictions](#technicalities-and-restrictions)
4. [Inner Workings](#inner-workings)

---

## Concepts

### The Glitch: Why It Happens

The Game Boy OAM hardware renders sprites by reading tile indices from the OAM table and looking up the corresponding 8×8 pixel patterns in sprite VRAM (0x8000–0x8FFF for sprites). When GB Studio changes an actor's spritesheet it:

1. Writes the new spritesheet's tile bitmaps into VRAM at the actor's `base_tile` slot (`SetBankedSpriteData`).
2. Updates the actor's `sprite` pointer and animations.

Steps 1 and 2 happen inside `vm_actor_set_spritesheet`, which runs during a script update. The OAM tile indices, however, are recalculated on the same frame in `actors_render`, which calls `move_metasprite` with the current `base_tile`. If VRAM writing is still in progress (spread across multiple VBlank periods for large sprites) when the LCD reads OAM, the hardware may display a mix of old and new tile data — the garbled-sprite glitch.

### The Fix: Double-Buffered VRAM Slots

The plugin doubles the VRAM tile reservation for any actor that uses `reserve_tiles`. Each such actor now gets two consecutive slots of `reserve_tiles` size in sprite VRAM:

- **Slot A** — offset 0 from `base_tile`
- **Slot B** — offset `reserve_tiles` from `base_tile`

A per-actor flag (`using_sprite_buffer`) tracks which slot is currently being displayed. When a spritesheet swap occurs:

1. The flag is **toggled** — the "back" slot becomes the write target.
2. The new tile data is written into the back slot.
3. On the next render call, `actors_render` computes: `base_tile = actor->base_tile + (using_sprite_buffer ? reserve_tiles : 0)` and passes that to `move_metasprite`.

Because the new tiles were written into the slot that OAM is *not* currently reading, the frame on which the write happens still displays the old sprite cleanly. The new sprite is visible from the following frame onward.

---

## Project Setup

1. Copy the plugin into your GB Studio project's `plugins` folder.
2. That is all — the feature is active automatically for any actor whose reserved tiles are set when a **Set actor spritesheet** event is on an actor. The plugin automaticaly doubles that amount to be used as a buffer when the spritesheet changes.

---

## Technicalities and Restrictions

### Only Actors with Reserve Tiles Are Buffered

The double-buffer only applies to actors (including the player) that have `reserve_tiles > 0`. Actors that share tiles from the scene's common sprite pool (`reserve_tiles == 0`) are not affected and do not have their VRAM allocation changed.

### Sprite VRAM Cost Is Doubled for Buffered Actors

Each actor with `reserve_tiles` now consumes `reserve_tiles * 2` sprite VRAM slots instead of `reserve_tiles`. On hardware with 256 sprite tile slots (0–255) this halves the effective tile budget for spritesheets that use reserved tiles. Plan your tile allocations accordingly — if VRAM runs out, other actors' sprites will be overwritten.

### Player Is Also Buffered

The player actor follows the same buffering logic as other actors. If the scene's `reserve_tiles` field is non-zero, the player's allocation is also doubled at scene load time.

### No Engine Fields or Events Added

The plugin has no engine settings, no GB Studio events, and no runtime-accessible variables. It is entirely transparent to the game logic.

### Modified Engine Files

The plugin replaces four core engine files. Any other plugin that also patches these files must be manually merged or an existing `engineAlt` combination must be used:

- `engine/src/core/actor.c` — `actors_render` reads `using_sprite_buffer` to choose the display slot.
- `engine/src/core/data_manager.c` — `load_scene` doubles the tile allocation for reserved actors and initialises `using_sprite_buffer = 0`.
- `engine/src/core/vm_actor.c` — `vm_actor_set_spritesheet` and `vm_actor_set_spritesheet_by_ref` toggle the buffer flag before writing.
- `engine/include/gbs_types.h` — the `actor_t` struct gains the `using_sprite_buffer` field.

---

## Inner Workings

### `actor_t` Struct Change (`gbs_types.h`)

A single byte field is added to the `actor_t` struct:

```c
uint8_t using_sprite_buffer;
```

This flag is 0 when the actor is currently displaying tiles in slot A and 1 when displaying from slot B. It lives in the same struct as `base_tile` and `reserve_tiles`, so both the render path and the spritesheet-swap path have direct access.

### Scene Load: Doubling the Allocation (`data_manager.c`)

During `load_scene`, the player's tile allocation is adjusted:

```c
UBYTE n_loaded = load_sprite(PLAYER.base_tile = 0, scn.player_sprite.ptr, scn.player_sprite.bank);
allocated_sprite_tiles = (n_loaded > scn.reserve_tiles) ? n_loaded : scn.reserve_tiles;
if (scn.reserve_tiles) {
    allocated_sprite_tiles = allocated_sprite_tiles << 1;   // double for buffer
}
```

For non-player actors that have `reserve_tiles`, the same doubling happens:

```c
actor->base_tile = allocated_sprite_tiles;
actor->using_sprite_buffer = 0;
UBYTE n_loaded = load_sprite(allocated_sprite_tiles, actor->sprite.ptr, actor->sprite.bank);
allocated_sprite_tiles += (((n_loaded > actor->reserve_tiles) ? n_loaded : actor->reserve_tiles) << 1);
```

The initial sprite data is loaded into slot A (`base_tile + 0`). Slot B (`base_tile + reserve_tiles`) starts uninitialised but is never displayed until a spritesheet swap occurs.

### Spritesheet Swap: Toggle and Write (`vm_actor.c`)

Both `vm_actor_set_spritesheet` and `vm_actor_set_spritesheet_by_ref` follow the same pattern:

```c
actor->using_sprite_buffer = !actor->using_sprite_buffer;
UBYTE base_tile = actor->base_tile + (actor->using_sprite_buffer ? actor->reserve_tiles : 0);
load_sprite(base_tile, spritesheet, spritesheet_bank);
```

1. The flag is toggled first — the back slot (the one OAM is **not** reading this frame) is selected as the write target.
2. `load_sprite` writes the new tile bitmaps into that back slot via `SetBankedSpriteData`.
3. The actor's `sprite`, `animations`, `bounds`, and animation frame are then updated to reflect the new spritesheet.

Because `using_sprite_buffer` was already flipped before the write, `actors_render` on this same frame will still compute `base_tile + 0` (or `base_tile + reserve_tiles`, whichever was the old display slot), showing the old sprite. On the next frame, `actors_render` reads the newly flipped `using_sprite_buffer` and switches to the freshly written slot.

### Render: Choosing the Display Slot (`actor.c`)

`actors_render` computes the OAM base tile for each actor:

```c
UBYTE base_tile = actor->base_tile + (actor->using_sprite_buffer ? actor->reserve_tiles : 0);
allocated_hardware_sprites += move_metasprite(
    *(sprite->metasprites + actor->frame),
    base_tile,
    allocated_hardware_sprites,
    screen_x,
    screen_y
);
```

This is identical for both the player and all other actors. The toggle ensures that `move_metasprite` always uses the slot whose VRAM data was fully written in a previous frame, never the slot currently being overwritten.

