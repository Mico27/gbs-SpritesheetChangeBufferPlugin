# gbs-SpritesheetChangeBufferPlugin
Plugin for GBStudio that create a buffer for spritesheet swapping

Actors that uses the set actor spritesheet event have reserved space in the tileset VRAM. When using that event, 
the updating of the tileset might not happen on the same frame as the update of the tiles in the OAM, which can result in garbled tiles for a brief moment. 

This plugin create a buffer in the tileset VRAM to prevent the display of that visual glitch, 
this however means that the reserved space in the tileset VRAM will be doubled for that actor.

Example with the buffer:
<img width="748" height="452" alt="image" src="https://github.com/user-attachments/assets/d90a6cdb-5581-4d27-ac59-7bf4bea5b5f5" />
<img width="751" height="468" alt="image" src="https://github.com/user-attachments/assets/cc8bc395-50b0-4727-86d9-6d22dcfe3a28" />

Example without the buffer:
<img width="783" height="466" alt="image" src="https://github.com/user-attachments/assets/669f4d10-614c-419f-9769-b2c8bc54d3f3" />
<img width="800" height="468" alt="image" src="https://github.com/user-attachments/assets/10e33fdc-36a8-4a5f-bbba-f6dd69f7cefd" />
<img width="749" height="478" alt="image" src="https://github.com/user-attachments/assets/1921a667-9ad3-4803-b484-99a6a1612b13" />

