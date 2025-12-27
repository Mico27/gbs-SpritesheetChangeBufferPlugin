# gbs-SpritesheetChangeBufferPlugin
Plugin for GBStudio that create a buffer for spritesheet swapping

Actors that uses the set actor spritesheet event have reserved space in the tileset VRAM. When using that event, 
the updating of the tileset might not happen on the same frame as the update of the tiles in the OAM, which can result in garbled tiles for a brief moment. 

This plugin create a buffer in the tileset VRAM to prevent the display of that visual glitch, 
this however means that the reserved space in the tileset VRAM will be doubled for that actor.

Example without the buffer plugin:



https://github.com/user-attachments/assets/f53a2621-c57a-45b0-8e64-609baf1817fa



Example with the buffer plugin:



https://github.com/user-attachments/assets/de925525-5b96-4b94-8fbd-7c280441bc4c

