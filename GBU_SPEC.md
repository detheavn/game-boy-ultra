# Game Boy Ultra (GBU) — Hardware Specification
**Version 1.0 — Draft**

The Game Boy Ultra is a fantasy handheld console built on the RP2350 microcontroller.
It is spiritually and technically an extension of the Game Boy Color — the same
instruction set, the same memory map philosophy, the same creative constraints —
but with expanded display capabilities, a richer audio architecture, and a small
set of carefully chosen hardware extensions.

The guiding principle is **freedom through limitation**. Every specification decision
is made to expand creative headroom without removing the constraints that make the
platform interesting to develop for. A developer who knows Game Boy assembly can
target the GBU in an afternoon.

---

## Table of Contents

1. [Comparison Overview](#1-comparison-overview)
2. [CPU](#2-cpu)
3. [Display](#3-display)
4. [Graphics — Tiles and Maps](#4-graphics--tiles-and-maps)
5. [Graphics — Palettes and Colour](#5-graphics--palettes-and-colour)
6. [Graphics — Sprites (OAM)](#6-graphics--sprites-oam)
7. [Memory Map](#7-memory-map)
8. [Input](#8-input)
9. [Audio — DMG APU (preserved)](#9-audio--dmg-apu-preserved)
10. [Audio — SID Extension Channels](#10-audio--sid-extension-channels)
11. [Audio — MOD Streaming](#11-audio--mod-streaming)
12. [Audio — Per-Channel Mix Control](#12-audio--per-channel-mix-control)
13. [Audio — External Input (Pin 31)](#13-audio--external-input-pin-31)
14. [Cartridge Format](#14-cartridge-format)
15. [ROM Header](#15-rom-header)
16. [Hardware Register Map](#16-hardware-register-map)
17. [Developer Toolchain](#17-developer-toolchain)
18. [Implementation Notes (Emulator)](#18-implementation-notes-emulator)

---

## 1. Comparison Overview

| Feature | DMG | GBC | **GBU** |
|---|---|---|---|
| CPU speed | 4.19 MHz | 8.39 MHz | **16 MHz** |
| M-cycles per frame | 17,556 | 35,112 | **~67,002** |
| Screen resolution | 160×144 | 160×144 | **240×160** |
| BG palettes | 1 × 4 colours | 8 × 4 colours | **12 × 6 colours** |
| OBJ palettes | 2 × 4 colours | 8 × 4 colours | **12 × 6 colours** |
| Tile format | 2bpp indexed | 2bpp indexed | **4bpp indexed** |
| Palette colour format | 2-bit greyscale | RGB555 (15-bit) | **RGB565 (16-bit)** |
| Tile map size | 32×32 tiles | 32×32 tiles | **64×64 tiles** |
| Unique tiles | 256 | 384 | **512** |
| VRAM | 8KB × 2 banks | 8KB × 2 banks | **16KB flat** |
| WRAM | 8KB | 32KB banked (4KB banks) | **32KB banked (8KB banks)** |
| OAM sprites | 40 | 40 | **64** |
| Shoulder buttons | — | — | **L + R** |
| SID audio channels | — | — | **2 (CH5 + CH6)** |
| MOD streaming | — | — | **1 stream, 16 sub-channels** |
| External audio in | Pin 31 (unused) | Pin 31 (unused) | **Pin 31 (active)** |
| Hicolour potential | No | Yes (STAT trick) | **Yes (wider H-Blank window)** |

---

## 2. CPU

The GBU uses the same Sharp SM83-compatible 8-bit CPU as the DMG and GBC.
The instruction set is **identical** — all existing GB assembly knowledge applies.

**Clock speed: 16 MHz**

This is exactly 4× the DMG speed and 2× the GBC double-speed mode.
It was chosen to comfortably cover the additional rendering work of the wider
screen and expanded palette system, while remaining a clean memorable multiple.

**M-cycles per frame:**
```
16 MHz / 4 (T-cycles per M-cycle) = 4 MHz M-cycle clock
4,000,000 / 59.7 fps = ~67,002 M-cycles per frame
```

**Speed switching:**
GBU mode is always 16 MHz. There is no double-speed register (CGB KEY1 / $FF4D).
The `rGBU_FLAGS` register (see section 16) enables GBU mode at boot.

**Timing note for developers:**
All cycle counts from existing GB documentation remain valid in terms of
instruction duration in M-cycles. The wall-clock time per M-cycle is shorter
(62.5ns vs 238ns on DMG), but relative timing between instructions is unchanged.
STAT interrupt timing is documented in stable cycle counts — see section 3.

---

## 3. Display

**Resolution: 240×160 pixels**
**Refresh rate: 59.7 fps** (same as DMG/GBC/GBA)
**Scanlines: 0–159 visible, 160–169 V-Blank** (10 V-Blank scanlines, same count as GBC)

The GBU display matches the Game Boy Advance screen dimensions. This is
intentional — it creates a natural horizontal expansion of the playing field
without requiring developers to rethink vertical game design.

**STAT interrupt timing (stable, exploitable):**
The GBU STAT interrupt fires at a guaranteed cycle offset within each scanline,
documented here so developers can rely on it for mid-scanline palette effects.

```
Per-scanline cycle budget (M-cycles):
  Total scanlines:       170  (160 visible + 10 V-Blank)
  M-cycles per frame:    67,002
  M-cycles per scanline: 67,002 / 170 = 394

  Mode 2 (OAM scan):    20 M-cycles   (scanline start, same as GBC)
  Mode 3 (render):      60 M-cycles   (pixel output — wider at 240px)
  Mode 0 (H-Blank):    314 M-cycles   (until next scanline)
  Total per scanline:   394 M-cycles

  STAT interrupt fires at: end of Mode 2 (cycle 20 of scanline)
  H-Blank interrupt fires at: start of Mode 0 (cycle 80 of scanline)
```

The H-Blank period (314 M-cycles at 16 MHz = ~19.6µs) is the window for
mid-scanline palette swaps and raster effects. This is approximately 3× the
wall-clock duration of the GBC H-Blank, giving substantially more time for
the hicolour technique, per-scanline scroll manipulation, and software
affine-transform effects.

**Scanline timing note for GBC developers:**
The per-scanline M-cycle count (394) differs from GBC (114). Code that relies
on specific cycle counts within a scanline — such as timed STAT interrupt
handlers — must be recalculated for GBU. The extended H-Blank budget means
most timing-sensitive code has significantly more headroom than on GBC.

**Hicolour potential:**
By swapping BG palette registers during H-Blank, a developer can display
more unique colours per frame than the 12 × 6 = 72 simultaneous colours
the palette system officially provides. The 314-cycle H-Blank window makes
this substantially more reliable than on GBC. This is a supported and
documented technique — the timing is stable and will not change between
hardware revisions.

---

## 4. Graphics — Tiles and Maps

### Tile Data

**512 unique tiles**, stored in 16KB of flat VRAM (`$8000–$BFFF`).
Each tile is 8×8 pixels at **4bpp = 32 bytes per tile**.

```
512 tiles × 32 bytes = 16,384 bytes = 16KB (exactly fills VRAM tile area)
```

**4bpp tile format:**
Each row of 8 pixels is stored as 4 bytes — 2 bits per pixel across two
bitplane pairs, giving 4 bits (16 values) per pixel:

```
Row encoding (4 bytes per row, 8 rows per tile = 32 bytes total):
  Byte 0: Plane 0 low  (bits 0 of pixels 7–0)
  Byte 1: Plane 0 high (bits 1 of pixels 7–0)
  Byte 2: Plane 1 low  (bits 2 of pixels 7–0)
  Byte 3: Plane 1 high (bits 3 of pixels 7–0)

  Pixel colour index = (plane1_hi << 3) | (plane1_lo << 2)
                     | (plane0_hi << 1) | (plane0_lo)
  Range: 0–15
    0–11:  valid palette colour indices
    12–15: transparent (OBJ) or border colour (BG)
```

The 4bpp format was chosen deliberately. It provides 16 index values per
pixel, covering the 6-colour palette range (values 0–5) with values 6–15
used as transparent/border. It is also the natural format for efficient
transfer over HSTX to the display.

**How pixel values and palette numbers interact:**

For BG tiles: the tile attribute byte specifies which BG palette (bits 2–0,
selecting palette 0–7 from the 12 available). The pixel's 4-bit value then
selects a colour *within* that palette. Pixel value 0 = colour 0 of the
assigned palette. Values 6–15 = transparent/border.

For OBJ tiles: the OAM attribute byte specifies which OBJ palette (bits 3–0,
selecting palette 0–11). The pixel's 4-bit value selects a colour within
that palette. Pixel value 0 = always transparent (GBC convention preserved).
Values 6–15 = also transparent.

The palette system therefore has two levels: which palette is assigned
(attribute byte), and which colour within that palette (pixel value).
This matches GBC's model but with more palettes and more colours per palette.

**Tile addressing:**
VRAM base: `$8000`. Tile N is at `$8000 + (N × 32)`.
All 512 tiles addressed directly. No bank switching. No signed/unsigned
addressing mode distinction (the GBC $8800-method is not used in GBU mode).

### Tile Maps

**Map size: 64×64 tiles = 512×512 pixels of scrollable space**

Each map entry is 2 bytes:
- Byte 0: Tile index low (0–255)
- Byte 1: Tile attributes

```
Tile attribute byte:
  Bit 7:   BG-to-OBJ priority (0=OBJ on top, 1=BG on top)
  Bit 6:   Y flip
  Bit 5:   X flip
  Bit 4:   Tile index high (extends tile index to 0–511)
  Bits 2–0: BG palette number (0–7, selects one of the 12 BG palettes)
  Bit 3:   (reserved, write 0)

Note: The pixel's 4-bit value (0–5) selects a colour *within* the palette
specified by bits 2–0. Values 6–15 are transparent/border. BG tiles use
the attribute byte's palette field so the same tile graphic can be rendered
in different colour schemes on different parts of the map — useful for
indoor/outdoor lighting, zone colours, animated palette effects, etc.

**BG vs OBJ palette selection:**
BG tiles: palette chosen by tile attribute bits 2–0 (per-tile, set at map
          load time, changed by rewriting tilemap attribute bytes).
OBJ tiles: palette chosen by OAM attribute bits 3–0 (per-sprite, changed
           at any time by writing to OAM during VBlank).
Both reference the same pool of 12 palettes per layer (12 BG + 12 OBJ).
See section 5 for palette RAM loading (BGPI/BGPD), section 6 for OAM.
```

**Two maps available via bank switching:**
The GBU provides two physical 8KB map buffers sharing one address window.
The active map is selected by `rGBU_MAP_CFG` bit 2:

```
$C000–$DFFF   Active map window (8,192 bytes = 64×64×2 entries)
              Bit 2 of rGBU_MAP_CFG selects Map A (0) or Map B (1)
```

Switching maps mid-frame during H-Blank enables creative effects — different
map banks can be swapped per-scanline row for parallax, split-screen, or
other techniques. Both physical maps are always writable by selecting the
appropriate bank before writing.

The Window layer uses a separate fixed 32×32 map (2,048 bytes), sufficient
to cover the full 240×160 screen with one tile of overhang on each edge.
Window map address selected via LCDC bit 6 (same as GBC).

**Memory layout:**
```
$8000–$BFFF   Tile data    (16,384 bytes — 512 tiles × 32 bytes)
$C000–$DFFF   BG tile map  (8,192 bytes — active bank of 64×64 map)

Note: Maps live outside VRAM in the address space previously occupied
by GBC WRAM. GBU mode repurposes $C000–$DFFF for tile maps.
GBU games must not assume GBC VRAM or WRAM layout.
```

**Scroll registers:**
Standard SCX/SCY handle 0–255. For the full 512-pixel map range,
`rGBU_SCX_HI` and `rGBU_SCY_HI` provide bit 8 of each.
See section 16 for register details.

---

## 5. Graphics — Palettes and Colour

### Palette System

**12 BG palettes, 12 OBJ palettes**
**6 colours per palette (indices 0–5)**
**Colour format: RGB565 (16-bit)**

The GBU palette system works at two levels:

**Level 1 — Palette assignment** (which palette applies to a tile or sprite):
- BG tiles: bits 2–0 of the tile attribute byte select one of the 12 BG palettes
- OBJ tiles: bits 3–0 of the OAM attribute byte select one of the 12 OBJ palettes

**Level 2 — Colour selection** (which colour within the assigned palette):
- The tile's 4-bit pixel value (0–15) selects the colour
- Values 0–5: colours within the assigned palette
- Value 0 for OBJ: always transparent (GBC convention preserved)
- Values 6–15: transparent for OBJ, border colour for BG

This matches the GBC model — palette number from attribute byte, colour from
pixel value — extended to 12 palettes with 6 colours each.

Pixel values 12–15 are **transparent** for OBJ tiles and a **configurable
border colour** for BG tiles (default black).

### Palette RAM

```
BG palette RAM:   12 palettes × 6 colours × 2 bytes = 144 bytes
OBJ palette RAM:  12 palettes × 6 colours × 2 bytes = 144 bytes
Total:            288 bytes
```

Palette RAM is accessed via index/data register pairs:

```
rGBU_BGPI ($FF68)  BG palette index
  Bits 7–0: Index (0–255). Valid range for GBU: 0–143.
            Writes to BGPD always advance the index by 1.
            Index wraps from 143 back to 0 automatically.
            To write a non-sequential entry: write BGPI before each BGPD write.

rGBU_BGPD ($FF69)  BG palette data
  Write low byte then high byte of the RGB565 colour.
  Index advances automatically after each byte write.

rGBU_OBPI ($FF6A)  OBJ palette index (same structure as BGPI)
rGBU_OBPD ($FF6B)  OBJ palette data  (same structure as BGPD)
```

**OBPI/OBPD vs OAM palette bits — two separate things:**

- **OBPI/OBPD** — the *loading interface* for palette RAM. Used during VBlank
  to write RGB565 colour values into the 12 OBJ palettes. The full-byte index
  (0–255, valid 0–143) was chosen so future palette layout changes only require
  updating the valid range, not the register format.

- **OAM attribute bits 3–0** — *palette selection at render time*. Which of the
  12 loaded OBJ palettes the PPU applies to a specific sprite. See section 6.
  Changing OAM bits 3–0 instantly changes a sprite's colour without touching
  tile data or reloading palette RAM.

**No auto-increment flag.** Unlike the GBC, the GBU palette index always
advances after each write to the data register. This simplifies palette
loading — set the index once, then stream bytes into the data register.
For random access (writing a single colour without advancing), write BGPI
before each BGPD write.

Linear palette index layout:
```
Index   0–1:   Palette 0, colour 0 (low byte, high byte)
Index   2–3:   Palette 0, colour 1
Index   4–5:   Palette 0, colour 2
Index   6–7:   Palette 0, colour 3
Index   8–9:   Palette 0, colour 4
Index  10–11:  Palette 0, colour 5
Index  12–13:  Palette 1, colour 0
...
Index 142–143: Palette 11, colour 5
```

Formula: palette P, colour C → index `(P × 6 + C) × 2` (low byte),
`(P × 6 + C) × 2 + 1` (high byte).

**Full-byte index design note:**
The index register uses a full byte (0–255) rather than packing an
auto-increment flag into bit 7 as the GBC does. This was chosen deliberately:
the valid index range (0–143) doesn't map cleanly into power-of-2 bit
boundaries, and a full-byte index makes the register contract stable for
future palette layout changes. If the number of palettes or colours per
palette changes in a future revision, only the valid range documented here
changes — the register format stays the same.

Colour 0 of every OBJ palette is **always transparent** (same as GBC).
Colour 0 of BG palettes is the background fill colour for that palette.

### Colour Format: RGB565

The GBU uses RGB565 (5 bits red, 6 bits green, 5 bits blue) stored as a
16-bit little-endian value, matching the ST7796S display controller natively.
This differs from the GBC's 15-bit BGR555 format — GBC palette data cannot
be used directly on GBU without conversion.

```
RGB565 layout (16 bits, little-endian):
  Bits 15–11: Red   (0–31)
  Bits 10–5:  Green (0–63)
  Bits 4–0:   Blue  (0–31)
```

---

## 6. Graphics — Sprites (OAM)

**64 sprites maximum** (up from 40 on DMG/GBC)
**No per-scanline sprite limit** (the GBC's 10-per-scanline limit is removed)
**Sprite sizes: 8×8 or 8×16** (LCDC bit 2, same as GBC)

OAM layout per sprite (4 bytes, extended from GBC):
```
Byte 0: Y position (sprite top + 16)
Byte 1: X position (sprite left + 8)
Byte 2: Tile index low (0–255)
Byte 3: Attributes
  Bit 7:   BG/OBJ priority (0=sprite on top, 1=BG on top)
  Bit 6:   Y flip
  Bit 5:   X flip
  Bit 4:   Tile index high (extends tile index to 0–511)
  Bits 3–0: OBJ palette number (0–11 valid, 12–15 clamp to 0)
```

**OAM palette bits vs OBPI/OBPD — two separate things:**

- **OAM bits 3–0** — palette selection *at render time*. Which of the 12 OBJ
  palettes the PPU applies to this sprite when drawing it. Changing these bits
  in OAM instantly changes the sprite's colour scheme without touching tile data.
  Example: the same Firebrand tile renders red (palette 0), blue (palette 2),
  or green (palette 4) depending on which spell is equipped — just by writing
  a different value to the sprite's OAM attribute byte each frame.

- **OBPI/OBPD** (`$FF6A`/`$FF6B`) — palette data *loading interface*. How the
  game writes RGB565 colour values into OBJ palette RAM. Used during VBlank
  initialisation to define what each palette actually looks like. Not involved
  in rendering at all.

4 bits (0–15) cover the current 12 palettes with room to expand to 16 in a
future revision without changing the OAM format. Values 12–15 clamp to
palette 0. Same flexibility principle as the full-byte OBPI index register.

### Sprite sizes

**8×8 mode (LCDC bit 2 = 0):** Each OAM entry renders one 8×8 tile.
One OAM entry, one palette assignment, one tile index.

**8×16 mode (LCDC bit 2 = 1):** Each OAM entry renders two stacked 8×8 tiles,
forming an 8×16 pixel sprite. Tile index bit 0 is forced to 0 — tile N is the
top half, tile N+1 is the bottom half automatically. One OAM entry, one palette
assignment. Cannot mix 8×8 and 8×16 sprites within the same frame (global mode).

**Note — future per-sprite size select:** A `rGBU_OBJ_SIZE` register (8 bytes,
one bit per sprite) allowing individual 8×8/8×16 selection per sprite regardless
of LCDC bit 2 is planned as a future addition. Not implemented in V1.

### Composite sprites

The hardware provides no sprite size larger than 8×16. Larger characters and
bosses are built from multiple OAM entries positioned adjacently — called
**composite sprites**. The hardware renders each OAM entry independently;
the player perceives them as one logical object.

Example — a 16×32 character (8 OAM entries in 8×8 mode):
```
Entry 0: tile A, position (x,    y   )  ← top-left
Entry 1: tile B, position (x+8,  y   )  ← top-right
Entry 2: tile C, position (x,    y+8 )
Entry 3: tile D, position (x+8,  y+8 )
Entry 4: tile E, position (x,    y+16)
Entry 5: tile F, position (x+8,  y+16)
Entry 6: tile G, position (x,    y+24)  ← bottom-left
Entry 7: tile H, position (x+8,  y+24)  ← bottom-right
```

Each entry has its own independent palette, flip flags, and priority bit.
The SDK `GBU_Sprite` object groups constituent OAM entries so position,
palette, and flip state can be updated with a single call.

**Very large sprites (bosses):** For characters larger than a few tiles, the
BG+OBJ layering technique is preferred over very large composite sprites.
The BG layer handles large non-animated body parts; OBJ handles animated
elements (face expressions, weapon states, damage flashes). GBU's 12 palettes
and no scanline limit make this significantly more capable than on GBC.

### Sprite animation

Two approaches, both valid:

**Tile rewrite (Option A):** Load one set of animation-frame tiles into fixed
VRAM slots. Each frame, overwrite those slots with the next frame's tile data
during VBlank. OAM entries never change tile indices. Uses minimal VRAM but
requires VRAM writes every animation frame.

**Frame strip (Option B):** Pre-load all animation frames into VRAM at map
load time (frames in consecutive VRAM slots). Each frame, update OAM tile
indices to point at the current frame's slots. No VBlank tile writes, but
uses more VRAM proportional to frame count. Scales well when a sprite
appears many times or has many frames.

The SDK animation manager handles both approaches automatically.
Random phase initialisation is supported so multiple instances of the same
sprite (e.g. torches) animate out of sync naturally.

### The no-scanline-limit benefit

On GBC, the 10-sprite-per-scanline limit forced a flicker workaround: sprites
alternated between visible and off-screen positions on odd/even frames, making
every other frame show a different subset of sprites. This created intentional
flicker as a workaround for a hardware ceiling.

On GBU, all 64 sprites render every scanline. The flicker workaround pattern
is detected by the decomp pipeline and flagged `REMOVE_ON_GBU` — the
alternation code is removed entirely and all sprites are written every frame.

OAM DMA works the same as GBC — write source page to `$FF46`, 160 bytes
are copied from `(source × $100)` to OAM. Extended sprites (entries 40–63)
are written directly to `$FE00–$FEFF`.

---

## 7. Memory Map

```
Address Range   Size    Description
─────────────────────────────────────────────────────────────
$0000–$3FFF     16KB    ROM Bank 0 (fixed)
$4000–$7FFF     16KB    ROM Bank N (switchable via MBC)
$8000–$BFFF     16KB    VRAM — tile data (flat, no banking)
$C000–$DFFF      8KB    Tile map — active BG map bank
$E000–$FFDF      8KB    WRAM bank window (active bank)
$FE00–$FEFF     256B    OAM — 64 sprites × 4 bytes
$FF00–$FFFF             High page — I/O registers, HRAM, IE
```

```
High page ($FF00–$FFFF):
$FF00           JOYP — joypad (same as DMG)
$FF01–$FF02     Serial (same as DMG, not implemented in V1)
$FF04–$FF07     Timer DIV/TIMA/TMA/TAC (same as DMG)
$FF0F           IF — Interrupt Flag (same as DMG)
$FF10–$FF26     APU — DMG sound registers (preserved exactly)
$FF30–$FF3F     Wave RAM (CH3 waveform, same as DMG)
$FF40–$FF4B     PPU registers (LCDC, STAT, SCY, SCX, LY, etc.)
$FF4F           (unused in GBU — flat VRAM needs no bank select)
$FF60           rGBU_ID — returns $47 ('G'), confirms GBU hardware
$FF61           rGBU_FLAGS — system mode and feature flags
$FF62           rGBU_INPUT — shoulder buttons L/R
$FF63           rGBU_SCX_HI — SCX bit 8 (high scroll bit)
$FF64           rGBU_SCY_HI — SCY bit 8
$FF65           rGBU_MAP_CFG — map bank select and size configuration
$FF66           (reserved — rGBU_OBJ_PAL_HI removed, see section 6)
$FF67           rGBU_WRAM_BANK — WRAM bank select (bits 1–0, banks 0–3)
$FF68           rGBU_BGPI — BG palette index (0–143)
$FF69           rGBU_BGPD — BG palette data (RGB565, auto-advance index)
$FF6A           rGBU_OBPI — OBJ palette index (0–143)
$FF6B           rGBU_OBPD — OBJ palette data
$FF6C           rGBU_VOL_APU — DMG APU master volume
$FF6D           rGBU_VOL_SID — SID channels master volume
$FF6E           rGBU_VOL_MOD — MOD stream master volume
$FF6F           rGBU_VOL_EXT — External input volume
$FF70           (reserved — GBC SVBK shadow, writes ignored in GBU mode)
$FF71           (reserved — alignment pad before SID registers)
$FF72–$FF7F     SID synthesis registers (CH5 + CH6)
$FF80–$FF82     MOD stream control registers
$FF83–$FF8F     (reserved)
$FF90–$FF9B     Per-channel volume and fade (CH1–CH6)
$FF9C–$FF9F     (reserved)
$FFA0–$FFAF     MOD sub-channel volume (channels 0–15)
$FFB0–$FFBF     MOD sub-channel fade (channels 0–15)
$FFC0–$FFFE     HRAM (high RAM, same as DMG)
$FFFF           IE — Interrupt Enable (same as DMG)
```

**WRAM: 32KB in four 8KB banks**

The GBU provides 32KB of WRAM organised as four equal 8KB banks.
The active bank is visible at `$E000–$FFDF`. The bank is selected by
writing to `rGBU_WRAM_BANK` (`$FF67`, bits 1–0).

```
rGBU_WRAM_BANK = 0 → $E000 maps to WRAM bytes 0x0000–0x1FFF   (default)
rGBU_WRAM_BANK = 1 → $E000 maps to WRAM bytes 0x2000–0x3FFF
rGBU_WRAM_BANK = 2 → $E000 maps to WRAM bytes 0x4000–0x5FFF
rGBU_WRAM_BANK = 3 → $E000 maps to WRAM bytes 0x6000–0x7FFF
```

Bank 0 is the default after reset. A GBU game that never writes to
`rGBU_WRAM_BANK` has 8KB of flat WRAM at `$E000` — consistent with DMG
in size. Games requiring more RAM bank-switch explicitly.

Unlike GBC (which has a fixed 4KB lower bank at `$C000–$CFFF`), all four
GBU banks are equal and fully switchable. There is no fixed region.

**GBC-to-GBU WRAM migration:**
GBC WRAM occupies `$C000–$DFFF`. GBU WRAM starts at `$E000`. The constant
address offset between the two is `$2000`. For games being recompiled from
GBC to GBU, symbolic WRAM references migrate automatically via linker script
change. Literal address values require manual review. GBC banks 0+1 (8KB
combined) map directly to GBU bank 0. GBC banks 2–7 map to GBU banks 1–3
(two GBC 4KB banks per GBU 8KB bank). See `GBU_DECOMP_PIPELINE.md` for
the full migration strategy.

**Tile map at $C000:**
In GBU mode, `$C000–$DFFF` is repurposed from WRAM (GBC) to tile maps.
Games must not use this region as general-purpose RAM in GBU mode.
GBU-native code uses `$E000–$FFDF` (and banked extensions) for all
mutable data.

---

## 8. Input

### Standard Controls (via shift register)

The GBU reads all buttons through a 3-wire serial shift register interface
(SNES controller or two daisy-chained 74HC165 chips). The JOYP register
($FF00) behaves identically to DMG — games poll it the same way.

Standard button mapping:
```
D-Pad: Up, Down, Left, Right
Face:  A, B, Select, Start
```

### Shoulder Buttons (GBU extension)

```
rGBU_INPUT ($FF62)  — read-only
  Bit 0: L shoulder (0 = pressed, 1 = released)
  Bit 1: R shoulder (0 = pressed, 1 = released)
  Bits 2–7: always 1 (open bus)
```

Returns $FF on DMG/GBC hardware (open bus), so code that checks
`rGBU_ID` before using $FF62 will work correctly on all targets.

---

## 9. Audio — DMG APU (preserved)

The GBU DMG APU is bit-for-bit identical to the original DMG/GBC APU.
All four channels, all registers, all timing behaviour is preserved.
GBC-era timing fixes are included.

```
CH1  $FF10–$FF14   Pulse with frequency sweep
CH2  $FF16–$FF19   Pulse (no sweep)
CH3  $FF1A–$FF1E   Arbitrary waveform (32 × 4-bit samples in $FF30–$FF3F)
CH4  $FF20–$FF23   Noise (LFSR)
     $FF24          NR50 — master volume + VIN routing
     $FF25          NR51 — left/right enable per channel
     $FF26          NR52 — sound master enable + channel status
```

Pin 31 (External Audio Input / VIN) is active on GBU hardware.
See section 13.

---

## 10. Audio — SID Extension Channels

Two additional synthesis channels (CH5 and CH6) inspired by the
MOS 6581/8580 SID chip used in the Commodore 64. These channels
provide richer timbre options and a shared resonant filter — the
characteristic "sweep" sound of C64 music.

### Waveforms (per channel)
```
0 = Sawtooth    — bright, buzzy, harmonic-rich
1 = Triangle    — softer, hollow, flute-like
2 = Pulse       — duty cycle fixed at 50% (square wave)
3 = Noise       — white noise, independent LFSR from CH4
```

### ADSR Envelope (per channel)
Each channel has a full 4-stage amplitude envelope:
- **Attack:**  time to reach peak volume from silence
- **Decay:**   time to fall from peak to sustain level
- **Sustain:** volume level held while note is on (0–15)
- **Release:** time to fall from sustain to silence after note off

### Shared Filter
One multimode filter processes any combination of CH5 and CH6.

```
Modes (rSID_FILTER_RES bits 1–0):
  00 = Low-pass   — attenuates high frequencies (warm, muffled)
  01 = High-pass  — attenuates low frequencies (thin, nasal)
  10 = Band-pass  — passes a frequency band (nasal, telephonic)
  11 = Notch      — attenuates a frequency band
```

Filter resonance (bits 7–4 of rSID_FILTER_RES) controls the
emphasis peak at the cutoff frequency — higher resonance produces
the characteristic SID "wah" effect.

### Ring Modulation
CH5 can ring-modulate CH6 when `rSID_RINGMOD` bit 0 is set.
Ring modulation multiplies the two waveforms, producing sum and
difference frequencies — metallic, bell-like tones.

### SID Register Map
```
$FF70  (reserved — GBC SVBK shadow, ignored in GBU mode)
$FF71  (reserved — alignment pad)
$FF72  rSID_CH5_FREQ_LO   CH5 frequency low byte
$FF73  rSID_CH5_FREQ_HI   CH5 frequency high byte
$FF74  rSID_CH5_WAVE      Bits 1-0: waveform select
$FF75  rSID_CH5_ADSR      Bits 7-4: attack, bits 3-0: decay
$FF76  rSID_CH5_SR        Bits 7-4: sustain, bits 3-0: release
$FF77  rSID_CH6_FREQ_LO
$FF78  rSID_CH6_FREQ_HI
$FF79  rSID_CH6_WAVE
$FF7A  rSID_CH6_ADSR
$FF7B  rSID_CH6_SR
$FF7C  rSID_FILTER_FREQ   Filter cutoff frequency (0–255)
$FF7D  rSID_FILTER_RES    Bits 7-4: resonance, bits 1-0: mode
$FF7E  rSID_FILTER_ROUTE  Bit 0: CH5 → filter, Bit 1: CH6 → filter
$FF7F  rSID_RINGMOD       Bit 0: ring mod CH5 × CH6
```

---

## 11. Audio — MOD Streaming (MXU — Music eXtension Unit)

One MOD/XM format audio stream plays from the SD card or cartridge,
decoded on the RP2350's second core. This provides richly arranged,
tracker-based music that sounds distinctly different from both the DMG
APU and the SID channels — giving composers three separate sonic textures
to work with simultaneously.

The audio extension system is collectively referred to as the
**MXU (Music eXtension Unit)**. Track files use the `.xm` format.
The track list file uses the `.mxl` (Music eXtension List) extension.

### File Format: XM (Extended Module)

XM was chosen for the following reasons:
- Fits the "limitation breeds innovation" philosophy — XM is itself a
  constrained creative medium with fixed channels and pattern-based
  composition. Developers cannot just throw PCM samples at every problem.
- Compact file sizes (a complete game soundtrack may fit in 200KB)
- Widely supported by tracker software (OpenMPT, MilkyTracker, etc.)
- Sounds distinctly different from DMG APU and SID, giving the composer
  a third sonic register to layer — orchestral, atmospheric, or textural

XM supports up to 32 channels. The GBU provides **runtime control over
channels 1–16** via the sub-channel volume and fade registers. Channels
17–32 play at whatever the composer set in the tracker — they are
expressive tools but their mix is fixed at composition time.

The 16-channel control limit is deliberate. It follows the same philosophy
as the rest of the platform: enough to do interesting things, not so much
that the solution is always "throw more channels at it." A composer who
uses all 16 controllable channels has built a sophisticated adaptive music
system. A composer who uses channels 17–32 has committed to a fixed
arrangement for those elements. Both choices are valid. Neither is the
easy way out.

### The Creative Potential: Live Remix at Runtime

The most powerful use of the MXU is not switching between tracks — it is
**remixing a single track in real time** based on game state. The composer
authors one XM file with all musical possibilities present; the game
reveals and conceals layers dynamically.

A few examples of what this enables:

**Scene atmosphere:** A town theme has a strings bed, a piano melody, and
ambient crowd noise. Entering a building mutes the outdoor layers and
brings up the interior reverb texture without the music stopping.

**Tension building:** A battle theme has bass, drums, and a lead melody.
As the boss's health drops, percussion layers enter progressively. At the
final phase, a SID channel adds a piercing lead over the now-full arrangement.

**Genre remix:** The classic title screen square waves are muted. An XM
strings section that was always there but silent plays instead. Players
who find the right moment in the game hear the same song in a completely
different way.

This is the spirit of the MXU — not a music player with a skip button,
but a compositional instrument the game plays.

### MXU Folder Structure

Each game's music lives in a dedicated folder named identically to the ROM
file (without extension). This keeps multi-game SD cards tidy and allows
the table of contents to be game-specific.

```
SD root/
├── Gargoyles_Quest.gbu          — ROM file
├── Gargoyles_Quest/             — MXU folder for this game
│   ├── TOC.mxl                  — table of contents (track list)
│   ├── TITLE.XM
│   ├── OVERWORLD.XM
│   ├── DUNGEON.XM
│   └── BOSS.XM
├── My_Game.gbu
└── My_Game/
    ├── TOC.mxl
    └── ...
```

The `TOC.mxl` file maps track indices (0-based) to XM filenames within
the same folder. Lines starting with `;` are comments. Blank lines are
valid entries (index reserved, no track loaded):

```
; TOC.mxl — Music eXtension List for Gargoyles_Quest
; Format: XM filename, one per line, 0-indexed
; Blank lines reserve an index without assigning a track

TITLE.XM
OVERWORLD.XM
DUNGEON.XM
BOSS.XM
; index 4 reserved
```

A game writes `rMOD_TRACK = 2` to select the track at line 2 (`DUNGEON.XM`).

The `TOC.mxl` format is intentionally simple for V1. Future revisions may
extend it with metadata (loop points, default channel volumes, etc.).

### MOD Stream Registers
```
$FF80  rMOD_CMD      Write: 0=stop, 1=play, 2=pause, 3=loop
$FF81  rMOD_TRACK    Track index (0–255, see TOC.mxl)
$FF82  rMOD_STATUS   Read-only: 0=stopped, 1=playing, 2=track ended
$FF83  (reserved for seek/position, future use)
```

### Sub-Channel Volume Design

The 16 controllable sub-channels map directly to XM file channels 1–16.
Sub-channel index 0 = XM channel 1, index 15 = XM channel 16.
XM channels 17–32, if used, play at their composed volume without
runtime control.

Volume and fade registers are in section 12.

A suggested channel layout convention for game soundtracks (document your
own mapping in `game_audio.inc` — this is not enforced by hardware):

```
; Suggested MXU channel layout — adapt per project
XM ch 1:   Strings / ambient bed   — always present
XM ch 2:   Brass / power section   — builds with tension
XM ch 3:   Piano / lead melody     — emotional highlight moments
XM ch 4:   Bass line               — enters with action
XM ch 5:   Percussion              — enters with action
XM ch 6:   Atmospheric texture     — scene-specific layer
; ch 7–16: additional controllable layers (project-specific)
; ch 17–32: fixed-mix elements (composed, not runtime-controlled)
```

---

## 12. Audio — Per-Channel Mix Control

Every audio channel in the GBU system — DMG APU channels, SID channels,
and MOD sub-channels — has an independent software volume register and
a fade-speed register. This is the core mechanism for adaptive music.

**Volume:** 0 = silent, 255 = full output. Takes effect at the next
audio frame boundary (~60Hz). Setting volume does not affect the
channel's internal state (envelope, note, position).

**Fade:** Writing a non-zero value starts a linear fade from the
current volume to the target volume over N frames (at 59.7fps,
255 frames ≈ 4.3 seconds). Writing 0 makes the volume change instant.
The fade register auto-clears to 0 when the target is reached.

### DMG APU and SID channel volumes
```
$FF90  rGBU_VOL_CH1    CH1 (pulse + sweep) volume
$FF91  rGBU_VOL_CH2    CH2 (pulse) volume
$FF92  rGBU_VOL_CH3    CH3 (wave) volume
$FF93  rGBU_VOL_CH4    CH4 (noise) volume
$FF94  rGBU_VOL_CH5    SID CH5 volume
$FF95  rGBU_VOL_CH6    SID CH6 volume
$FF96  rGBU_FADE_CH1   CH1 fade speed (frames to target)
$FF97  rGBU_FADE_CH2
$FF98  rGBU_FADE_CH3
$FF99  rGBU_FADE_CH4
$FF9A  rGBU_FADE_CH5
$FF9B  rGBU_FADE_CH6
```

### MOD sub-channel volumes
```
$FFA0–$FFAF  rMOD_CH0_VOL – rMOD_CH15_VOL   (16 registers)
$FFB0–$FFBF  rMOD_CH0_FADE – rMOD_CH15_FADE (16 registers)
```

Sub-channel index 0 = XM file channel 1, index 15 = XM file channel 16.
See section 11 for the MXU folder structure and `TOC.mxl` format.

### Master layer volumes
```
$FF6C  rGBU_VOL_APU    DMG APU master (scales CH1–CH4 together)
$FF6D  rGBU_VOL_SID    SID master (scales CH5–CH6 together)
$FF6E  rGBU_VOL_MOD    MOD stream master (scales all sub-channels)
$FF6F  rGBU_VOL_EXT    External input volume (pin 31)
```

### Final mix
```
Left  = (APU_L × VOL_APU) + (SID_L × VOL_SID) + (MOD_L × VOL_MOD) + EXT_L
Right = (APU_R × VOL_APU) + (SID_R × VOL_SID) + (MOD_R × VOL_MOD) + EXT_R
```

NR50/NR51 APU panning registers remain active and apply within the APU layer
before the master APU volume scales the result.

### Example: Layered scene music

Note: sub-channel registers at `$FFA0–$FFBF` are outside the LDH range and
require `LD HL, addr / LD [HL], A` rather than `LDH`. Channel-specific
GBU registers at `$FF90–$FF9B` use `LDH` normally.

```asm
; Enter haunted forest — MOD bed only, all synth silent
    ld a, MOD_LOOP
    ldh [rMOD_CMD], a            ; $FF80 — LDH range

    xor a                        ; 0 = silent
    ldh [rGBU_VOL_CH1], a        ; $FF90 — LDH range
    ldh [rGBU_VOL_CH2], a
    ldh [rGBU_VOL_CH3], a
    ldh [rGBU_VOL_CH4], a
    ldh [rGBU_VOL_CH5], a

; Mute MOD brass (ch 2, index 1) and percussion (ch 5, index 4)
    xor a
    ld hl, rMOD_CH1_VOL          ; $FFA1 — above LDH range, use HL
    ld [hl], a
    ld hl, rMOD_CH4_VOL          ; $FFA4
    ld [hl], a

; Tension rises — bass and melody fade in over 1.5 seconds
    ld a, 180
    ldh [rGBU_VOL_CH3], a        ; wave bass target
    ldh [rGBU_VOL_CH1], a        ; pulse melody target
    ld a, 90                     ; 90 frames ≈ 1.5s at 59.7fps
    ldh [rGBU_FADE_CH3], a
    ldh [rGBU_FADE_CH1], a

; Climax — SID pad swells, MOD brass returns
    ld a, 200
    ldh [rGBU_VOL_CH5], a        ; SID CH5 target volume
    ld a, 60
    ldh [rGBU_FADE_CH5], a       ; 60 frame fade
    ld a, 160
    ld hl, rMOD_CH1_VOL          ; brass sub-channel back in
    ld [hl], a
    ld a, 45
    ld hl, rMOD_CH1_FADE
    ld [hl], a
```

---

## 13. Audio — External Input (Pin 31)

The original Game Boy cartridge connector exposes an analog audio input
on pin 31 (labelled EXT or VIN on schematics). On DMG and GBC hardware
this pin mixes directly into the headphone amplifier at the same level
as the internal APU output, but was almost never used commercially.

On GBU hardware, pin 31 is **active and documented**. A cartridge may
route any analog audio source to this pin — a secondary sound chip,
a piezo element, a sampler, or any other audio-generating circuit.
The GBU mixes pin 31 into the master output under volume control of
`rGBU_VOL_EXT` ($FF6F).

**Emulator behaviour:**
In the GBX emulator, pin 31 input is mapped to a configurable line-in
source. On the Pico 2W test hardware it is not connected. On the custom
board it will be wired to the cartridge slot pin 31 directly.

---

## 14. Cartridge Format

GBU games are distributed as ROM files with the `.gbu` extension.
Internally they are standard Game Boy ROM format — the same binary layout
the GBX emulator loads via MBC — with GBU identification provided by the
ROM header (`$014C = $47`).

### Distribution Options

**SD card** — The primary distribution and development format. ROM files
and their accompanying MXU music folders live on a standard SD card loaded
via the cartridge slot or an onboard SD socket. See section 11 for the
MXU folder structure.

**SD card cartridge** — A cartridge PCB containing only an SD card socket,
using the cartridge bus. This gives the SD card a physical cartridge form
factor. The emulator detects the SD cart and handles it identically to a
direct SD socket. This is a first-class distribution option, not a workaround.

**Flash cartridge** — A cartridge PCB with a NOR flash chip containing the
ROM. The MXU music data can be stored in a larger flash chip alongside the
ROM, or played from a companion SD cart.

### Physical Flash Cartridge (under development)

The flash cartridge design is currently experimental. NOR flash chips from
the Winbond W25Q series are the primary candidate, connected via QSPI
(Quad SPI — a 4-bit serial interface using CS, CLK, and IO0–IO3 lines).
The RP2350 reads the flash using the W25Q Fast Read command, auto-detected
via JEDEC ID.

This section will be updated once the physical cartridge design is
confirmed. The connector form factor, PCB layout, and supported chip
variants are still being determined.

Existing bootleg GBC cartridge shells (widely available) are a candidate
for the enclosure, with custom PCBs fitted inside. Connector sockets for
the GBC cartridge edge connector are readily available as repair parts.

Pin 31 (External Audio Input) is routed from the cartridge edge connector
to the audio mix on the main board regardless of cartridge type.

---

## 15. ROM Header

The GBU ROM header is a superset of the standard Game Boy header.
All standard header fields at their standard offsets are preserved.

### GBU Identification

**Byte $014C** (Mask ROM number — unused/zero in standard homebrew):
```
$47 ('G') = This is a GBU ROM — run in MODE_GBU
$00       = Standard GB/GBC ROM (default)
```

The emulator reads `$014C` first. If `$014C = $47`, GBU mode is activated
regardless of the GBC flag at `$0143`.

**GBC flag (`$0143`) for GBU ROMs:**
GBU-native games should set `$0143 = $C0` (GBC-only flag). This signals
to any real GBC hardware that the ROM is not intended for it — a real GBC
will display a black screen rather than attempting to run GBU code. Setting
`$0143 = $80` (GBC-enhanced) would allow the GBC to attempt to run the
ROM, which will produce incorrect results since GBU games target a
completely different memory map and timing model. There is no practical
benefit to making a GBU ROM appear GBC-compatible at the hardware level.

GBU games will not run correctly on real DMG or GBC hardware by design.
This is expected and intentional.

### Standard Header Fields (unchanged)
```
$0100–$0103  Entry point (NOP + JP to game start, same as always)
$0104–$0133  Nintendo logo (required for boot, same bytes)
$0134–$0143  Title (15 chars + GBC flag — use $C0 for GBU-only)
$0144–$0145  New licensee code
$0146        SGB flag ($00=none, $03=SGB compatible)
$0147        Cartridge type (MBC type, same codes as GBC)
$0148        ROM size code
$0149        RAM size code
$014A        Destination code
$014B        Old licensee code
$014C        **GBU identifier ($47 = GBU, $00 = standard)**
$014D        Header checksum (covers $0134–$014C, same algorithm)
$014E–$014F  Global checksum
```

### hardware-gbu.inc declaration
```asm
SECTION "Header", ROM0[$0100]
    nop
    jp EntryPoint
    ; Nintendo logo
    DB $CE,$ED,$66,$66,$CC,$0D,$00,$0B,$03,$73,$00,$83,$00,$0C,$00,$0D
    DB $00,$08,$11,$1F,$88,$89,$00,$0E,$DC,$CC,$6E,$E6,$DD,$DD,$D9,$99
    DB $BB,$BB,$67,$63,$6E,$0E,$EC,$CC,$DD,$DC,$99,$9F,$BB,$B9,$33,$3E
    ; Title (15 chars) + GBC flag
    DB "MY GBU GAME    "   ; 15 chars
    DB $C0                 ; GBC flag: $C0 = GBC-only (correct for GBU games)
    DB $00,$00             ; New licensee
    DB $00                 ; SGB flag
    DB $01                 ; MBC type (MBC1)
    DB $05                 ; ROM size (1MB)
    DB $03                 ; RAM size (32KB)
    DB $01                 ; Destination
    DB $33                 ; Old licensee (use new system)
    DB $47                 ; GBU identifier — activates MODE_GBU
    DB $00                 ; Header checksum (filled by rgbfix)
    DW $0000               ; Global checksum (filled by rgbfix)
```

---

## 16. Hardware Register Map

Complete register listing. All standard DMG/GBC registers at their
standard addresses are omitted here — refer to Pan Docs for those.
Only GBU-specific registers are listed.

### System
```
$FF60  rGBU_ID        Read-only. Returns $47 ('G'). Confirms GBU hardware.
                      Returns $FF on DMG/GBC (open bus).

$FF61  rGBU_FLAGS     R/W.
                      Bit 0: Widescreen enable (240×160 mode)
                      Bit 1: Fast CPU enable (16 MHz)
                      Bits 2–7: Reserved (write 0)

$FF65  rGBU_MAP_CFG   R/W.
                      Bit 0: BG map 64 tiles wide (1) or 32 (0)
                      Bit 1: BG map 64 tiles tall (1) or 32 (0)
                      Bits 3–2: Bus write target for $C000–$DFFF:
                                00 = BG Map Bank A (default)
                                01 = BG Map Bank B
                                10 = Window map (32×32, 2KB — $C000–$C7FF)
                                11 = reserved
                      Note: the PPU always reads the BG layer from the bank
                      selected by bit 2 and the window layer from win_map[]
                      regardless of this register. Bits 3–2 only affect
                      which physical buffer CPU writes go to.

$FF67  rGBU_WRAM_BANK R/W.
                      Bits 1–0: WRAM bank select (0–3)
                      Each bank is 8KB. Bank 0 is default after reset.
                      Bits 2–7: Reserved (write 0)
```

### Input
```
$FF62  rGBU_INPUT     Read-only.
                      Bit 0: L shoulder (0=pressed)
                      Bit 1: R shoulder (0=pressed)
                      Bits 2–7: Always 1
```

### Extended Scroll
```
$FF63  rGBU_SCX_HI   R/W. Bit 0 = SCX bit 8 (scroll X range 0–511)
$FF64  rGBU_SCY_HI   R/W. Bit 0 = SCY bit 8 (scroll Y range 0–511)
```

### Palette RAM
```
$FF68  rGBU_BGPI     R/W. BG palette index.
                     Bits 7–0: Index (0–255). Valid range: 0–143.
                     Index always advances by 1 after each write to BGPD.
                     Wraps from 143 to 0. No auto-increment flag — always on.

$FF69  rGBU_BGPD     R/W. BG palette data (RGB565 little-endian, low byte first).
                     Writing advances BGPI automatically.

$FF6A  rGBU_OBPI     R/W. OBJ palette index. Same structure as BGPI.
$FF6B  rGBU_OBPD     R/W. OBJ palette data. Same structure as BGPD.
```

### Audio Master Volumes
```
$FF6C  rGBU_VOL_APU  R/W. DMG APU layer master volume (0–255).
$FF6D  rGBU_VOL_SID  R/W. SID layer master volume (0–255).
$FF6E  rGBU_VOL_MOD  R/W. MOD stream master volume (0–255).
$FF6F  rGBU_VOL_EXT  R/W. External input (pin 31) volume (0–255).
```

### SID Synthesis
```
$FF70  (reserved — GBC SVBK shadow. Writes ignored in GBU mode.)
$FF71  (reserved — alignment pad before SID block.)

$FF72  rSID_CH5_FREQ_LO   CH5 frequency low byte
$FF73  rSID_CH5_FREQ_HI   CH5 frequency high byte
$FF74  rSID_CH5_WAVE      Bits 1–0: waveform (0=saw,1=tri,2=pulse,3=noise)
$FF75  rSID_CH5_ADSR      Bits 7–4: attack (0–15), bits 3–0: decay (0–15)
$FF76  rSID_CH5_SR        Bits 7–4: sustain (0–15), bits 3–0: release (0–15)
$FF77  rSID_CH6_FREQ_LO
$FF78  rSID_CH6_FREQ_HI
$FF79  rSID_CH6_WAVE
$FF7A  rSID_CH6_ADSR
$FF7B  rSID_CH6_SR
$FF7C  rSID_FILTER_FREQ   Filter cutoff frequency (0–255)
$FF7D  rSID_FILTER_RES    Bits 7–4: resonance (0–15), bits 1–0: mode
$FF7E  rSID_FILTER_ROUTE  Bit 0: CH5→filter, bit 1: CH6→filter
$FF7F  rSID_RINGMOD       Bit 0: ring modulation CH5×CH6
```

### MOD Stream
```
$FF80  rMOD_CMD      R/W. 0=stop, 1=play, 2=pause, 3=loop.
$FF81  rMOD_TRACK    R/W. Track index (0–255, references game's TOC.mxl).
$FF82  rMOD_STATUS   Read-only. 0=stopped, 1=playing, 2=track ended.
$FF83  (reserved)
```

### Per-Channel Volume and Fade
```
$FF90  rGBU_VOL_CH1    CH1 volume target (0–255)
$FF91  rGBU_VOL_CH2    CH2 volume target
$FF92  rGBU_VOL_CH3    CH3 volume target
$FF93  rGBU_VOL_CH4    CH4 volume target
$FF94  rGBU_VOL_CH5    SID CH5 volume target
$FF95  rGBU_VOL_CH6    SID CH6 volume target
$FF96  rGBU_FADE_CH1   CH1 fade speed (frames, 0=instant)
$FF97  rGBU_FADE_CH2
$FF98  rGBU_FADE_CH3
$FF99  rGBU_FADE_CH4
$FF9A  rGBU_FADE_CH5
$FF9B  rGBU_FADE_CH6
```

### MOD Sub-Channel Control
```
$FFA0–$FFAF  rMOD_CH0_VOL – rMOD_CH15_VOL
             Per-sub-channel volume (0–255). Index n = XM channel n+1.

$FFB0–$FFBF  rMOD_CH0_FADE – rMOD_CH15_FADE
             Per-sub-channel fade speed (frames, 0=instant).
```

---

## 17. Developer Toolchain

### RGBDS (Assembly)

GBU games are developed with [RGBDS](https://rgbds.gbdev.io/), the standard
Game Boy assembler/linker. No modifications to RGBDS itself are needed.

GBU-specific registers are provided via an include file:

```
hardware.inc       — standard GB/GBC register definitions (unchanged)
hardware-gbu.inc   — GBU extension registers (include after hardware.inc)
```

`hardware-gbu.inc` defines all registers listed in section 16 as named
constants, plus convenience constants for register values:

```asm
; Convenience constants defined in hardware-gbu.inc
MOD_STOP    EQU 0
MOD_PLAY    EQU 1
MOD_PAUSE   EQU 2
MOD_LOOP    EQU 3

SID_SAW     EQU 0
SID_TRI     EQU 1
SID_PULSE   EQU 2
SID_NOISE   EQU 3

GBU_ID_BYTE EQU $47    ; Expected value from rGBU_ID
```

### Runtime GBU Detection

A GBU-enhanced ROM that also runs on DMG/GBC should probe for GBU hardware
before enabling GBU features. `rGBU_ID` is at `$FF60` — use `LDH` for
the 2-byte fast form:

```asm
CheckGBU:
    ldh a, (rGBU_ID)          ; $FF60 — use LDH, not LD A,(nn)
    cp GBU_ID_BYTE             ; $47 = 'G'
    jr nz, .not_gbu
    ; GBU hardware confirmed — enable extensions
    ld a, %00000011            ; widescreen + fast CPU
    ldh (rGBU_FLAGS), a
    ret
.not_gbu
    ; Running on DMG/GBC — use standard feature set
    ret
```

Note: most GBU games will set `$0143 = $C0` (GBC-only) and `$014C = $47`
in the ROM header, so they boot directly into MODE_GBU without needing
runtime detection. Detection is only needed for ROMs that target both GBU
and earlier hardware simultaneously.

### C SDK (planned)

A C SDK for GBU development is planned. It compiles to SM83 machine code
constrained to the GBU memory map — a C developer targeting GBU gets access
only to the GBU fantasy console's capabilities, not the RP2350's full
capabilities. See `GBU_PHILOSOPHY.md` section 6.

The SDK is organised in phases. Core files compile for both ASM and C projects.

#### Phase 1 — Foundation
```
gbu.h          Hardware register definitions matching hardware-gbu.inc exactly
gbu.ld         Linker script: VRAM $8000, maps $C000, WRAM $E000, banks
gbu_crt0.asm   Startup stub: initialises GBU hardware, calls main()
```

Key functions:
```c
void GBU_WaitVBlank(void);          // Halt until VBlank interrupt
void GBU_LCDOff(void);              // Safe LCD disable (waits for VBlank)
void GBU_LCDOn(uint8_t lcdc_val);   // Re-enable LCD with given LCDC value
uint16_t GBU_ReadInput(void);       // Read all buttons including L/R shoulders
```

#### Phase 2 — Graphics
```c
// Tile loading
void GBU_LoadTiles(const uint8_t* src, uint16_t tile_start, uint16_t count);
void GBU_LoadTilesBank(const uint8_t* src, uint16_t tile_start,
                       uint16_t count, uint8_t rom_bank);

// Palette management
// Note: GBU_SetBGPalette/GBU_SetOBJPalette write RGB565 data via BGPI/BGPD.
// This is the LOADING INTERFACE — separate from per-tile/per-sprite palette
// SELECTION which happens via attribute bytes. See sections 4, 5, 6.
void GBU_SetBGPalette (uint8_t pal, const uint16_t* colours); // 6 colours
void GBU_SetOBJPalette(uint8_t pal, const uint16_t* colours); // 6 colours
void GBU_SetBGColour  (uint8_t pal, uint8_t idx, uint16_t rgb565);
void GBU_SetOBJColour (uint8_t pal, uint8_t idx, uint16_t rgb565);

// Scroll
void GBU_SetScroll(uint16_t scx, uint16_t scy); // 9-bit values, uses SCX_HI/SCY_HI
void GBU_SetWindowPos(uint8_t wx, uint8_t wy);

// Tilemap
void GBU_LoadTilemap(const uint8_t* data, uint8_t map_bank);
void GBU_SetTile(uint8_t x, uint8_t y, uint16_t tile_id, uint8_t attr);
void GBU_SetWinTile(uint8_t x, uint8_t y, uint16_t tile_id, uint8_t attr);
void GBU_SelectMapBank(uint8_t bank);  // 0=Map A, 1=Map B (rGBU_MAP_CFG bit 2)
```

#### Phase 3 — Sprite system
```c
// GBU_Sprite — logical sprite grouping multiple OAM entries.
// Abstracts composite sprites so large characters are managed as one object.
// Each OAM entry within a sprite can have its own palette (OAM bits 3-0),
// but GBU_SetSpriteOBJPalette updates all entries simultaneously.

typedef struct {
    uint8_t  oam_start;      // First OAM entry index for this sprite
    uint8_t  oam_count;      // Number of OAM entries (tiles) in this sprite
    uint8_t  width_tiles;    // Width in 8px tiles
    uint8_t  height_tiles;   // Height in 8px tiles
    uint16_t tile_base;      // Base tile index in VRAM (current anim frame)
    uint8_t  obj_palette;    // OAM palette (bits 3-0), applied to all entries
    uint8_t  anim_frame;     // Current animation frame index
    uint8_t  anim_speed;     // Frames per animation step
    uint8_t  anim_counter;   // Internal frame counter
    uint8_t  anim_count;     // Total animation frames
    // Internal: first tile for each animation frame in VRAM
    uint16_t anim_frames[8]; // Up to 8 frames (Option B — pre-loaded strips)
} GBU_Sprite;

void GBU_SpriteInit    (GBU_Sprite* s, uint8_t oam_start,
                        uint8_t w_tiles, uint8_t h_tiles);
void GBU_SpriteSetPos  (GBU_Sprite* s, int16_t x, int16_t y);
void GBU_SpriteSetPal  (GBU_Sprite* s, uint8_t palette);  // OAM bits 3-0
void GBU_SpriteFlip    (GBU_Sprite* s, bool flip_x, bool flip_y);
void GBU_SpriteUpdate  (GBU_Sprite* s);  // Advance animation, write OAM
void GBU_SpriteWriteOAM(GBU_Sprite* s);  // Write to OAM without advancing anim
```

#### Phase 3 — Map command stream (procedural maps)
```c
// GBU native map format: command stream + optional prefab table.
// Renders procedurally into the tilemap window ($C000–$DFFF) at load time.
// Much smaller ROM footprint than raw tile index maps for large worlds.

typedef enum {
    MAP_CMD_END      = 0x00,  // End of command stream
    MAP_CMD_TILE     = 0x01,  // Single tile placement
    MAP_CMD_SETPOS   = 0x02,  // Set cursor position
    MAP_CMD_OUTLINE  = 0x03,  // Rectangle outline
    MAP_CMD_FILL     = 0x04,  // Rectangle fill
    MAP_CMD_LINE_H   = 0x06,  // Horizontal line
    MAP_CMD_LINE_V   = 0x07,  // Vertical line
    MAP_CMD_PREFAB   = 0x08,  // Place a prefab (multi-tile composite)
    MAP_CMD_PALZONE  = 0x09,  // Set palette for a region (attr byte)
    MAP_CMD_COND     = 0x0A,  // Conditional: skip if script var out of range
} GBU_MapCmd;

void GBU_RenderCommandMap(const uint8_t* cmd_stream,
                          const GBU_Prefab* prefab_table,
                          uint8_t map_bank);

// GBU_Prefab — multi-tile composite placed by MAP_CMD_PREFAB.
// The "prefab" / "multi-tile prefab" concept: a named composite of tiles
// that can be placed as a single command, similar to a stamp tool.
// Replaces the need to individually paint every tile of recurring structures
// (houses, trees, wells, dungeon features) in every map that uses them.
typedef struct {
    uint8_t  width;           // Width in tiles
    uint8_t  height;          // Height in tiles
    uint8_t  flags;           // GBU_PREFAB_SOLID, GBU_PREFAB_HAS_DOOR, etc.
    const uint8_t* tiles;     // Tile indices (width × height entries)
    const uint8_t* attrs;     // Attribute bytes (palette, flip, priority)
} GBU_Prefab;
```

#### Phase 3 — Effects (H-Blank driven)
```c
// All effects run during H-Blank on Core 0. The 314 M-cycle H-Blank budget
// makes per-scanline manipulation comfortable without cycle counting.

// Palette animation — runs automatically each VBlank via animation manager
void GBU_FX_PaletteRotate(uint8_t pal, uint8_t layer,
                           uint8_t speed);            // Rotate colours within palette
void GBU_FX_PaletteLerp  (uint8_t pal, uint8_t layer,
                           const uint16_t* target,
                           uint8_t frames);           // Fade to target palette

// Raster effects — registered H-Blank callbacks
void GBU_FX_SineScroll   (uint8_t y_start, uint8_t y_end,
                           uint8_t amplitude, uint8_t frequency,
                           uint8_t phase);            // Per-scanline SCX wave
void GBU_FX_MapBankFlip  (uint8_t scanline);         // Switch BG map bank at scanline

// Tile animation manager (registered at map load time)
void GBU_AnimRegister    (uint8_t type, uint8_t tile_or_pal,
                          uint8_t speed, uint8_t frames,
                          const void* data);          // Up to 32 concurrent animations
void GBU_AnimUpdate      (void);                      // Call once per VBlank
```

#### Phase 4 — Audio
```c
// APU (DMG channels CH1–CH4) — wrappers over NR registers
void GBU_APU_PlayTone   (uint8_t ch, uint16_t freq, uint8_t vol);
void GBU_APU_Stop       (uint8_t ch);

// Per-channel volume and fade ($FF90–$FF9B)
void GBU_SetChannelVol  (uint8_t ch, uint8_t volume);  // 0–255
void GBU_SetChannelFade (uint8_t ch, uint8_t frames);  // 0=instant

// SID synthesis (CH5–CH6, $FF72–$FF7F)
void GBU_SID_SetFreq    (uint8_t ch, uint16_t freq);
void GBU_SID_SetWave    (uint8_t ch, uint8_t waveform); // SID_SAW/TRI/PULSE/NOISE
void GBU_SID_SetADSR    (uint8_t ch, uint8_t attack, uint8_t decay,
                          uint8_t sustain, uint8_t release);
void GBU_SID_SetFilter  (uint16_t freq, uint8_t resonance, uint8_t route);

// MOD streaming ($FF80–$FF82 + TOC.mxl)
void GBU_MOD_Play       (uint8_t track);
void GBU_MOD_Stop       (void);
void GBU_MOD_SetVol     (uint8_t channel, uint8_t volume); // sub-channel 0-15
void GBU_MOD_Fade       (uint8_t channel, uint8_t frames);

// Master volumes ($FF6C–$FF6F)
void GBU_SetMasterVol   (uint8_t apu, uint8_t sid,
                          uint8_t mod, uint8_t ext);
```

The SDK is not yet implemented. This section will be expanded as
implementation progresses. See `GBU_DECOMP_PIPELINE.md` for how the SDK
functions are used as C scaffold substitutions during game recompilation.

### Audio File Preparation

XM files are created with any tracker supporting the XM format:
- OpenMPT (Windows, free) — recommended
- MilkyTracker (cross-platform, free)
- Renoise (commercial, XM export)

Place XM files in the game's MXU folder on the SD card. Create `TOC.mxl`
listing one filename per line (0-indexed). Comments start with `;`.
See section 11 for the full MXU folder structure and format.

---

## 18. Implementation Notes (Emulator)
19. Optional Peripherals & Expansion
    - 19.1 Wireless Controllers (Console Mode)
    - 19.2 WiFi Features
    - 19.3 Link Cable / Local Multiplayer
    - 19.4 USB Host — Wired Controllers
    - 19.5 CEC System Menu
    - 19.6 Real-Time Clock
    - 19.7 Power Management — USB-C Charging
    - 19.8 Capability Register Summary

Notes for the GBX emulator implementation. Not relevant for game developers.

### Files affected by GBU implementation

| File | Status |
|---|---|
| `cpu.h` | ✅ GBU palette RAM, extended VRAM/WRAM, 64-sprite OAM, wram_bank |
| `bus.c` | ✅ GBU register reads/writes ($FF60–$FFBF), HDMA, banked WRAM |
| `ppu.c` | ✅ GBU render path: 64×64 map, 4bpp decode, 240px width |
| `mbc.c` | ✅ $014C check for GBU identification in mbc_init_from_header |
| `system.c` | ✅ CYCLES_PER_FRAME_GBU = 67,002; GBU scanline timing = 394 |
| `cpu_utils.c` | ✅ gbu_reset(), gbc_reset() with correct field initialisations |
| `display.inc` | ⏸ Pending display hardware decision |

Note: All source files have been refactored from `.inc` to `.c`/`.h`
compilation units. The opcode dispatcher files (`opcodes_*.inc`) remain
as `.inc` fragments included inside `cpu_step()` in `gbx.cpp` — this is
intentional.

### GBU mode activation sequence
1. `mbc_init_from_header()` reads `$014C` → sets `cpu.mode = MODE_GBU`
2. `gbu_reset()` initialises all GBU state (volumes, palette RAM, WRAM bank)
3. `cpu_reset()` sets registers to GBU power-on defaults
4. `system_run_frame()` uses `CYCLES_PER_FRAME_GBU` = 67,002
5. `update_ppu()` uses 240px render width, 160 visible scanlines, 394 M-cycles/scanline
6. `bus_read/write` routes `$FF60–$FFBF` to GBU register handlers

### Cycles per frame in each mode
```c
#define CYCLES_PER_FRAME_DMG  17556   // 4.194 MHz / 59.7 fps / 4
#define CYCLES_PER_FRAME_GBC  35112   // 8.389 MHz / 59.7 fps / 4
#define CYCLES_PER_FRAME_GBU  67002   // 16.0  MHz / 59.7 fps / 4
```

### Audio implementation priority
The audio subsystem is not yet implemented. Recommended implementation order:

1. DMG APU (CH1–CH4) — most games depend on this; highest impact
2. Per-channel volume/fade registers — needed for GBU adaptive music
3. MOD streaming on Core 1 — XM decoder + TOC.mxl reader + MXU folder structure
4. MOD sub-channel volume control
5. SID synthesis (CH5–CH6) — most complex, implement last

### Open TODOs

**Active — implement when emulator work resumes:**
- [x] Cart RAM persistence — implemented: dirty-flag idle counter, `.sav` /
      `.gbu.sav` extension convention, atomic write via temp file rename
- [x] Window layer map base selection — implemented: separate 32×32 win_map[]
      buffer, rGBU_MAP_CFG bits 3–2 route writes, PPU reads win_map directly
- [ ] Audio subsystem — see priority order above
- [ ] GBC retesting — retest with 144p test suite now that HDMA is implemented
- [ ] ROM selector subdirectory browsing
- [ ] SGB border support (`$0146 = $03`)
- [ ] Custom border support for non-SGB games (user-supplied image on SD)
- [ ] MBC3 RTC — needed for Pokémon Gold/Silver
- [ ] Save states — snapshot CPU registers + all RAM regions + VRAM + OAM +
      palette RAM + hardware register values to a file on SD card.
      Files named `<rom_stem>_<slot>.state` alongside the ROM.
      Accessible via an in-game overlay menu (future feature).
      Implementation: `fwrite` of `CpuState`, `MemoryState`, `GbcState`/`GbuState`,
      `MbcState.cart_ram`, `MbcState.ram_bank`, cycle counters.

**On hold — pending display hardware:**
- [ ] Display framerate — parallel display with 74LVC574 GPIO latches planned
      (DSi XL via transposer board, or parallel RGB panel direct)
- [ ] HDMI out via RP2350 HSTX — planned as primary development display output
- [ ] Hicolour mode — requires direct scanline control, not framebuffer;
      feasible with HSTX output

**Removed from scope:**
- ~~Touch screen driver~~ — touch screen does not align with the GBU's
  generational positioning (GBC → GBA had no touch input). Removed.
- ~~Theme support for ROM selector~~ — deferred indefinitely, not a
  platform feature.

---

## 19. Optional Peripherals & Expansion

Optional hardware features that extend GBU beyond its baseline capability.
All optional peripherals follow the same principle as the capability register
(`rGBU_HW_CAPS`) — games query what's available and gracefully degrade if
a feature is absent. No optional peripheral is required to run GBU software.

---

### 19.1 Wireless Controllers (Console Mode)

**Purpose:** Enable modern Bluetooth controllers when GBU is used in console
mode connected to a TV/monitor via HSTX. Allows play without the built-in
handheld controls.

**Supported controllers (target list):**
- Xbox Series / Xbox One (Bluetooth)
- PlayStation 4 / PS5 DualSense (Bluetooth)
- Nintendo Switch Pro Controller (Bluetooth)
- 8BitDo controllers (Bluetooth HID)
- Any standard Bluetooth HID gamepad

**Wireless module tiers:**

The wireless module is a separate optional component connected to the RP2350B
over UART. The RP2350B never handles the wireless stack directly — it receives
simple serialised input events regardless of which module is fitted. This means
modules are interchangeable without any RP2350B firmware changes.

```
ESP32-C3 (default / baseline)
  WiFi 4, 2.4GHz only
  Bluetooth 5.0 LE
  160MHz single core
  Lowest power draw
  Most mature SDK, widest availability
  Cheapest (~€1-2 as bare module)
  Sufficient for: BT controllers, link cable tunnelling,
                  ESP-NOW local multiplayer, basic online

ESP32-C6 (sidegrade — WiFi 6 efficiency)
  WiFi 6, 2.4GHz only
  Bluetooth 5.3 LE
  160MHz HP core + 20MHz LP core
  Low-power core handles background tasks during sleep
  Better battery life in active-listening scenarios vs C3
  Recommended for users who want WiFi 6 efficiency
  without needing 5GHz

ESP32-C5 (sidegrade — dual-band)
  WiFi 6, 2.4GHz + 5GHz
  Bluetooth 5 LE
  240MHz core + LP core
  For users on 5GHz-only or 5GHz-preferred networks
  Newest chip, SDK still maturing
  Slightly higher cost and power draw
```

The C3 is the right default for a handheld where battery life matters.
The C6 and C5 are sidegrades for users with specific network requirements
or who want to leave the module running more actively.

**Implementation:**

```
ESP32-Cx (Bluetooth HID host)
        │
        │  UART (2 GPIO pins)
        │  Protocol: simple button state packets
        │  [controller_id][button_bitmask_lo][button_bitmask_hi]
        ▼
RP2350B (GBU host)
        └── Maps to existing joypad input registers
            Same input path as built-in buttons
```

**GPIO cost:** 2 pins (UART TX/RX). Defined in `hw_pins.h` when PCB is
finalised — sourced from cartridge bus reserved region or spare GPIO pool.

**Console mode activation:** Detected automatically when HDMI HPD goes high
(monitor connected). GBU switches from handheld input to console input mode
and the wireless module begins scanning for Bluetooth controllers.

**Up to 4 simultaneous controllers:** Maps to 4 virtual joypad ports.
Single-player games always use port 0 regardless of which controller is active.

---

### 19.2 WiFi Features (via wireless module)

Optional features enabled when a wireless module is present. WiFi is off
by default — enabled via system menu. The module enters deep sleep when
WiFi is disabled, drawing negligible power (~10-15µA).

**Online multiplayer:**
Link cable protocol tunnelled over UDP/IP via a relay server. See section
19.3 for the full multiplayer transport stack.

**Leaderboards and achievements:**
GBU-native games can submit scores and achievement events to a community
server. Opt-in, privacy-respecting, no account required (anonymous device ID).

**OTA firmware updates:**
The GBU checks for updates at boot if WiFi is configured. Updates verified
with a signature before flashing. Triggerable via system menu without
needing a computer.

**Note on 5GHz networks:**
Some modern routers are configured 5GHz-first or 5GHz-only. The C3 and C6
modules are 2.4GHz only and will not connect to these networks. The C5
module handles both bands. Users on 5GHz-only networks who want WiFi
features should use the C5 sidegrade.

---

### 19.3 Link Cable / Local Multiplayer

**Background — original Game Boy link architecture:**

The original Game Boy link cable used a master/slave model — one unit
drives the clock and initiates all transfers, the other responds. This is
true for all GB/GBC games including multiplayer titles. Faceball 2000's
16-player cable chain extended this with a daisy-chain topology, still
with one master driving the entire chain.

The GBU's wireless multiplayer replicates this model naturally — one GBU
acts as host/coordinator, others as clients.

---

**Transport stack — three modes:**

```
Mode 1: Wired link cable
  Physical connector (3.5mm TRRS or proprietary)
  5V-tolerant GPIO (RP2350B0A4 variant)
  ~8KHz synchronous serial, byte-swap protocol
  Zero latency, maximum compatibility
  Original GB/GBC link cable games work without modification

Mode 2: ESP-NOW local wireless (no router needed)
  Direct peer-to-peer between GBU wireless modules
  No WiFi network or router required
  ~1ms latency — transparent to game logic
  Range: up to 200m in open field
  Up to 16 players simultaneously (see below)

Mode 3: Internet relay
  Link protocol tunnelled over UDP via relay server
  Any distance — works globally
  Variable latency (20-100ms typical)
  Rollback netcode for timing-sensitive games
  Requires WiFi-capable module (C3/C5/C6)
```

From the game's perspective all three modes are identical —
`rGBU_LINK_DATA` and `rGBU_LINK_STATUS` registers behave the same
regardless of transport. The wireless module handles mode selection
transparently.

---

**ESP-NOW multiplayer architecture:**

ESP-NOW is Espressif's connectionless peer-to-peer protocol — no router,
no pairing ceremony, direct device-to-device at 1Mbps. Critically it
supports broadcast mode which allows one device to reach an unlimited
number of receivers in a single transmission.

```
2-4 players (standard multiplayer):
  Unicast ESP-NOW with registered peers
  Each GBU's C3 module paired with up to 3 others
  Clean, authenticated, lowest latency

5-16 players (Faceball 2000 and similar):
  Broadcast ESP-NOW with custom message framing
  One GBU designated host (player 1 / first to start session)
  Host collates all inputs, broadcasts authoritative game state

  Host GBU
  C3 module (coordinator)
  ├── broadcast → Client GBU 2 (C3)
  ├── broadcast → Client GBU 3 (C3)
  ├── broadcast → Client GBU 4 (C3)
  └── ... up to 15 clients (16 total)

  Each client sends: [player_id][button_state] → host (unicast)
  Host broadcasts:   [frame_id][all_player_states][checksum]
  Each client:       filters broadcast for relevant state, updates game
```

**Payload sizing for 16 players:**
```
Per-player state:    ~4 bytes (buttons + position hint)
16 players:          64 bytes
Frame overhead:      ~10 bytes
Total per broadcast: ~74 bytes
ESP-NOW v1.0 limit:  250 bytes  ✓ comfortable
ESP-NOW v2.0 limit:  1470 bytes ✓ very comfortable
```

**Latency for 16 players:**
At ESP-NOW's 600kbps+ throughput with small payloads, round-trip from
input to all players seeing the updated state is well under 10ms locally.
The original Faceball 2000 daisy-chain serial protocol had comparable or
higher latency. 16-player wireless GBU Faceball 2000 is genuinely feasible.

**Discovery and session management:**
The wireless module firmware handles peer discovery — scanning for other
GBU modules broadcasting a session beacon. The system menu shows available
nearby sessions. Joining a session requires no pairing procedure.
The QuickESPNow library (or equivalent) manages peer registration
transparently, removing the 20-peer firmware limit for large sessions.

---

**GBU-native high-bandwidth protocol:**
For GBU-native multiplayer games that need more than the 2-byte link cable
exchange, the wireless transport supports larger payloads. GBU-native
multiplayer games can exchange up to 1470 bytes per frame (ESP-NOW v2.0),
enabling richer shared game state, voice chat metadata, or synchronised
procedural generation seeds.

---

**Internet relay (Mode 3) details:**
A lightweight relay server proxies link byte streams between two or more
GBUs over the internet. For games with tight timing requirements, rollback
netcode compensates for variable latency — each GBU predicts remote player
inputs and rolls back if wrong, the same technique used by modern fighting
game netplay (GGPO). Turn-based and less timing-sensitive games work
without rollback. The relay server is open-source and self-hostable —
no dependency on a central GBU service.

---

### 19.4 USB Host — Wired Controllers

**Purpose:** USB-A host port for wired USB HID controllers. No wireless
module needed. Any USB HID gamepad works via TinyUSB.

**Implementation:** TinyUSB HID host mode on the RP2350's native USB
peripheral. USB-A connector on the GBU PCB. Controller input mapped to
the same joypad register interface as all other input sources.

**Compatible devices:**
- Any USB HID gamepad (Xbox, PlayStation, 8BitDo, generic)
- USB keyboards (for development / text input)
- USB hubs (multiple controllers via one port)

**GPIO cost:** Zero additional GPIO — uses the RP2350's dedicated USB pins
(shared with the programming/charging USB-C connector via a USB switch IC
or separate physical connector).

---

### 19.5 CEC System Menu (TV Remote Control)

**Purpose:** When GBU is connected to a TV via HDMI, the TV remote can
navigate the GBU system menu via the HDMI CEC protocol. Useful when no
controller with a dedicated menu button is available.

**Implementation:** Single-wire PIO implementation on `PIN_HDMI_CEC`
(GPIO20 in dev config, GPIO47 in final config — see `hw_pins.h`).
The GBU listens for CEC User Control commands and maps them to system
menu navigation.

**CEC button mapping:**

| TV Remote Button | CEC Command | GBU Action |
|-----------------|-------------|------------|
| Menu / Home | ROOT_MENU | Open GBU system menu |
| Up/Down/Left/Right | DIRECTION | Navigate menu |
| OK / Select | SELECT | Confirm |
| Back / Return | EXIT | Close menu |
| Red button | F1 | Quick save state |
| Green button | F2 | Load save state |
| Yellow button | F3 | Screenshot |
| Blue button | F4 | Return to ROM selector |

**No controller required:** The system menu (ROM selection, save states,
display settings, battery status) is fully navigable via TV remote alone.

---

### 19.6 Real-Time Clock

**Purpose:** Hardware timekeeping for MBC3 RTC emulation (Pokémon Gold/Silver)
and GBU-native games that use real-world time for day/night cycles or
time-based events.

**Implementation:** DS3231 or similar I2C RTC IC. Battery-backed to maintain
time when GBU is powered off. Shared I2C bus with PMIC (section 19.7).

**GBU register interface:**
```
rGBU_RTC_SEC   ($FFxx)  Seconds (BCD)
rGBU_RTC_MIN   ($FFxx)  Minutes (BCD)
rGBU_RTC_HOUR  ($FFxx)  Hours (BCD, 24h)
rGBU_RTC_DAY   ($FFxx)  Day of week (0-6)
rGBU_RTC_DATE  ($FFxx)  Date (BCD)
rGBU_RTC_MON   ($FFxx)  Month (BCD)
rGBU_RTC_YEAR  ($FFxx)  Year low byte (BCD, offset from 2000)
rGBU_RTC_CTRL  ($FFxx)  Control (set time, alarm enable)
```

MBC3 RTC register reads are intercepted by the emulator and sourced from
the hardware RTC when present, falling back to a software counter when not.

---

### 19.7 Power Management — USB-C Charging

**Purpose:** Single USB-C connector for charging, power delivery, and
firmware updates simultaneously. Compatible with Nintendo Switch battery
(HAC-003, 3.7V 4310mAh) and Wii U GamePad battery as user-swappable
power sources.

**PMIC:** BQ25895 or equivalent. Handles:
- USB-C CC pin negotiation (identifies charger capability)
- Li-ion charging at programmable rate (target 1A for Switch battery)
- Regulated 3.3V system output
- Boost to 5V for peripherals
- I2C interface for accurate state-of-charge readout
- USB data pass-through to RP2350 (transparent to firmware update)

**Battery connector:** JST ZH 1.5mm 2-pin — matches Nintendo Switch HAC-003
battery connector directly. User-swappable without tools.

**Battery indicator:** RGB LED driven by 3 PWM GPIO pins:
```
Green          > 50% charge
Yellow/Orange  20–50%
Red            < 20%
Red flashing   < 5%  (save your game)
Blue           Charging
Purple         Charged (USB connected, battery full)
```

Software battery percentage available via I2C from PMIC, displayed in
system menu. Raw ADC fallback on `PIN_BATT_ADC` (GPIO45) if PMIC unavailable.

**Firmware update:** RP2350 BOOTSEL mode (USB mass storage, UF2 format)
triggered by holding BOOT button at power-on, or via system menu
(System → Firmware Update) for tool-free updates.

---

### 19.8 Capability Register Summary

All optional peripheral availability is reflected in the hardware capability
registers readable by GBU software:

```
rGBU_HW_CAPS_LO ($FF60 + TBD)
  Bit 0: GBU_CAP_WIRELESS     — ESP32-C6 module present
  Bit 1: GBU_CAP_BLUETOOTH    — Bluetooth controllers available
  Bit 2: GBU_CAP_WIFI         — WiFi available
  Bit 3: GBU_CAP_USB_HOST     — USB host port present
  Bit 4: GBU_CAP_LINK_CABLE   — Link port present
  Bit 5: GBU_CAP_RTC          — Hardware RTC present
  Bit 6: GBU_CAP_RUMBLE       — Rumble motor present (future)
  Bit 7: GBU_CAP_HDMI         — HDMI/HSTX output active

rGBU_HW_CAPS_HI ($FF60 + TBD + 1)
  Bit 0: GBU_CAP_CEC          — HDMI CEC available
  Bit 1: GBU_CAP_STEREO       — Stereo audio output (vs mono)
  Bit 2: GBU_CAP_EXT_AUDIO    — External audio input (pin 31)
  Bits 3-7: reserved
```

A game that uses no optional features never reads these registers and runs
identically on all GBU hardware. A game that uses Bluetooth multiplayer
checks `GBU_CAP_BLUETOOTH` and falls back to link cable or single-player
if the bit is clear.

---

*GBX Emulator / Game Boy Ultra — specification document*
*Maintained alongside the emulator source in the project repository*
