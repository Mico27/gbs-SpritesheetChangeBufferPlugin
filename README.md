# gbs-SpritesheetChangeBufferPlugin

**Version 4.3.1 — Requires GB Studio ≥ 4.3.0**

A GB Studio engine plugin that eliminates the one-frame visual glitch that occurs when using the **Set actor spritesheet** event. It double-buffers each reserved actor's sprite tiles: new tile data is written into a hidden back slot, and the actor only switches to that slot once the write is complete. The old sprite stays on screen cleanly for the frame the write happens on, and the new sprite appears on the next frame.

There are no new events or settings to configure — installing the plugin is all that is needed.

**Without the plugin:**

https://github.com/user-attachments/assets/f53a2621-c57a-45b0-8e64-609baf1817fa

**With the plugin:**

https://github.com/user-attachments/assets/de925525-5b96-4b94-8fbd-7c280441bc4c

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [Size Limits and Restrictions](#size-limits-and-restrictions)
4. [Memory Footprint](#memory-footprint)

---

## Concepts

### Why the glitch happens

When GB Studio changes an actor's spritesheet, it writes the new tile bitmaps into the actor's sprite VRAM slot, then updates the actor to use the new sheet. But the display reads those same tiles on the same frame, and for a large sprite the write is spread across more than one VBlank period. If the display reads while the write is still in progress, it shows a mix of the old and new tiles — the garbled-sprite glitch.

### Double-buffered VRAM slots

The plugin gives each actor that reserves sprite tiles **two** slots instead of one, and tracks which is currently being displayed. A spritesheet swap writes into the slot that is *not* being displayed, then switches to it on the following frame.

Because the new tiles land in a slot nothing is reading, the frame the write happens on still shows the old sprite cleanly, and the new sprite is visible from the next frame onward.

---

## Project Setup

1. Copy the plugin into your GB Studio project's `plugins/` folder.

That is all. The fix is active automatically for any actor with reserved tiles — which GB Studio sets up for you when a **Set actor spritesheet** event targets that actor. The plugin doubles that reservation to provide the buffer.

---

## Size Limits and Restrictions

### Only actors with reserved tiles are buffered

The double buffer applies to actors, including the player, that have reserved tiles. Actors drawing from the scene's shared sprite pool are unaffected and their VRAM allocation is unchanged.

### Sprite VRAM cost is doubled for buffered actors

Each buffered actor consumes twice as many sprite VRAM slots as before. With 256 sprite tile slots available, this halves the effective tile budget for spritesheets that use reserved tiles. Plan your tile allocations accordingly — if VRAM runs out, other actors' sprites get overwritten.

### The player is also buffered

If the scene reserves tiles for the player, the player's allocation is doubled too.

### No engine fields or events are added

The plugin has no settings, no events and no runtime-accessible variables. It is entirely transparent to game logic.

### Stock engine files are modified

The plugin replaces four core engine files, covering actor rendering, scene loading, the spritesheet-swap events and the actor data layout. Any other plugin that also patches those files must be merged manually, or an existing compatibility variant used.

---

## Memory Footprint

Measured against the stock GB Studio **4.3.0-e1** engine (per-file SDCC compile with GB Studio's build flags, default engine settings). Values are the plugin's *delta* versus the stock engine; DMG build, with CGB noted where it differs. ROM cost lands in banked ROM (GB Studio's autobanker spreads it across switchable banks); using the plugin's events additionally compiles a few bytes of GBVM script per call into your project's script banks.

| | Cost |
|---|---|
| WRAM | +21 bytes |
| ROM | +321 bytes |

- **WRAM:** 21 bytes — one byte per actor, at the default 21 actors. Projects that raise the actor limit pay 1 byte per extra actor.
- **Sprite VRAM:** buffered actors consume twice their reserved tile count. This is VRAM, not ROM or WRAM, but it is the plugin's real cost.
- **Engine WRAM headroom:** the stock GB Studio 4.3.0 engine leaves about **854 bytes** of WRAM free (usable engine WRAM is 7,776 bytes at 0xC0A0–0xDF00; the stock engine uses 6,922 bytes). With this plugin installed roughly **833 bytes** remain. This figure does not depend on how many global variables your project defines: the script memory array has a fixed size of VM_HEAP_SIZE + (VM_MAX_CONTEXTS × VM_CONTEXT_STACK_SIZE) words — 768 + 16 × 64 = 1,792 words (3,584 bytes) with stock engine settings.
- **SRAM:** not used.

---

<!-- BANK0:BEGIN -->
## Bank 0 (HOME) Usage

Bank 0 is the 16 KB non-switchable ROM bank that the GB Studio engine core,
the interrupt handlers and the GBDK runtime all share. Banked ROM is cheap
(add another bank), bank 0 is not, so it is usually the first thing a project
runs out of.

| | Bytes |
|---|---|
| Bank 0 used by this plugin | **+186** |
| Bank 0 free with this plugin installed | **1,265** of 16,384 (92% used) |

Everything else this plugin adds lives in banked ROM.

| Module | This plugin | Stock engine | Bank 0 cost |
|---|---|---|---|
| `actor.c` | 1,057 | 871 | +186 |

Modules that replace or patch a stock engine file only cost the *difference*:
the stock version's bank 0 bytes were being spent anyway.

<details><summary>How this was measured</summary>

GB Studio 4.3.2, DMG target, default engine settings. Each module's bank 0
contribution is the `A _HOME size` record that SDCC writes into its `.rel`
object, summed over the engine sources this plugin provides. Stock sizes come
from building projects whose only plugin ships no engine C, so every module in
them is the untouched engine; two such builds were compared and agreed on all
73 shared modules.

The "free" figure is a stock project with this plugin and nothing else. Your
own number will differ: other plugins, and any engine settings that change what
the core compiles, move it independently of this plugin.

</details>
<!-- BANK0:END -->
