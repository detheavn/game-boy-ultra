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
| M-cycles per frame | 17,556 | 35,112 | **~67,000** |
| Screen resolution | 160×144 | 160×144 | **240×160** |
| BG palettes | 1 × 4 colours | 8 × 4 colours | **12 × 6 colours** |
| OBJ palettes | 2 × 4 colours | 8 × 4 colours | **12 × 6 colours** |
| Colour depth | 2bpp index | 4bpp (15-bit RGB) | **4bpp (RGB565)** |
| Tile map size | 32×32 tiles | 32×32 tiles | **64×64 tiles** |
| Unique tiles | 256 | 384 | **512** |
| VRAM | 8KB × 2 banks | 8KB × 2 banks | **16KB flat** |
| WRAM | 8KB | 32KB banked | **32KB flat** |
| OAM sprites | 40 | 40 | **64** |
| Shoulder buttons | — | — | **L + R** |
| SID audio channels | — | — | **2 (CH5 + CH6)** |
| MOD streaming | — | — | **1 stream, 16 sub-channels** |
| External audio in | Pin 31 (unused) | Pin 31 (unused) | **Pin 31 (active)** |
| Hicolour potential | No | Yes (STAT trick) | **Yes (enhanced STAT)** |

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
**Refresh rate: 59.7 fps** (same as DMG/GBC)
**Scanlines: 0–159 visible, 160–176 V-Blank** (adjusted for taller pixel count)

The GBU display is the same physical size category as the GBC but wider,
matching the Game Boy Advance screen dimensions. This is intentional — it
creates a natural horizontal expansion of the playing field without requiring
developers to rethink vertical game design.

**STAT interrupt timing (stable, exploitable):**
The GBU STAT interrupt fires at a guaranteed cycle offset within each scanline,
documented here so developers can rely on it for mid-scanline palette effects.

```
Per-scanline cycle budget (M-cycles):
  Mode 2 (OAM scan):    20 M-cycles   (scanline start)
  Mode 3 (render):      43 M-cycles   (pixel output)
  Mode 0 (H-Blank):     51 M-cycles   (until next scanline)
  Total per scanline:   114 M-cycles

STAT interrupt fires at: end of Mode 2 (cycle 20 of scanline)
H-Blank interrupt fires at: start of Mode 0 (cycle 63 of scanline)
```

The H-Blank period (51 M-cycles at 16 MHz = ~3.2µs) is the window for
mid-scanline palette swaps. At 16 MHz this is a wider window than GBC,
giving more time for the hicolour technique.

**Hicolour potential:**
By swapping BG palette registers during H-Blank, a developer can display
more unique colours per frame than the 12 × 6 = 72 simultaneous colours
the palette system officially provides. This is a supported and documented
technique — the timing is stable and will not change between hardware revisions.

---

## 4. Graphics — Tiles and Maps

### Tile Data

**512 unique tiles**, stored in 16KB of flat VRAM.
Each tile is 8×8 pixels at 4bpp = 32 bytes per tile.
```
512 tiles × 32 bytes = 16,384 bytes = 16KB (exactly fills VRAM)
```

Tile indices 0–511 are valid. Tile data format is identical to GBC:
two bitplanes per row, 2 bytes per row, 8 rows = 16 bytes per tile.

**Tile addressing:**
The GBU uses a single flat VRAM bank (no bank switching needed).
VRAM base: `$8000`. Tile n is at `$8000 + (n × 32)`.
The 8000-method and 9000-method addressing modes from DMG/GBC are preserved
for compatibility, but both address the same flat 16KB space.

### Tile Maps

**Map size: 64×64 tiles = 512×512 pixels of scrollable space**

Each map entry is 2 bytes:
- Byte 0: Tile index (0–255 in low map, 256–511 accessed via attribute bit)
- Byte 1: Tile attributes (same layout as GBC, extended)

```
Tile attribute byte:
  Bit 7:   BG-to-OBJ priority (0=OBJ on top, 1=BG on top)
  Bit 6:   Y flip
  Bit 5:   X flip
  Bit 4:   Tile bank select (0=tiles 0-255, 1=tiles 256-511)
  Bit 3:   (reserved)
  Bits 2-0: Palette number (0–11, values 12–15 are undefined)
```

**Two maps are available:** BG map and Window map.
Map base addresses selected via LCDC bits (same as GBC):
- BG map:     `$9800` (LCDC bit 3 = 0) or `$9C00` (LCDC bit 3 = 1)
- Window map: `$9800` (LCDC bit 6 = 0) or `$9C00` (LCDC bit 6 = 1)

Each map occupies 64×64×2 bytes = 8,192 bytes.
Both maps fit within the 16KB VRAM space alongside tile data.

```
VRAM layout:
  $8000–$BFFF  Tile data    (16,384 bytes, tiles 0–511)
  $C000–$DFFF  BG map       (8,192 bytes,  64×64 entries)
  — Note: This layout differs from DMG/GBC. —
  — GBU games must not assume GBC VRAM layout. —
```

**Scroll registers:**
Standard SCX/SCY handle 0–255. For the full 512-pixel map range,
extended high-bit registers provide bits 8 of SCX and SCY.
See `rGBU_SCX_HI` and `rGBU_SCY_HI` in section 16.

---

## 5. Graphics — Palettes and Colour

### Palette System

**12 BG palettes, 12 OBJ palettes**
**6 colours per palette (indices 0–5)**
**Colour format: RGB565 (16-bit)**

Palette indices 6–15 in tile data are treated as **transparent** for OBJ tiles
and as a **defined border colour** for BG tiles (configurable, default black).
This is the hardware enforcement of the 6-colour constraint.

The asymmetry with power-of-2 values is intentional — 12 palettes require
4 bits to address (tile attribute bits 2–0 address 0–11; values 12–15 are
undefined behaviour). This mild awkwardness is a creative constraint: palette
planning is part of the design work.

### Palette RAM

```
BG palette RAM:   12 palettes × 6 colours × 2 bytes = 144 bytes
OBJ palette RAM:  12 palettes × 6 colours × 2 bytes = 144 bytes
Total:            288 bytes
```

Palette RAM is accessed via index/data register pairs (same pattern as GBC):

```
rGBU_BGPI ($FF68)  BG palette index
  Bits 5–0: Index (0–71, addressing all 12×6 colours linearly)
  Bit 7:    Auto-increment (1 = index advances after each write to BGPD)

rGBU_BGPD ($FF69)  BG palette data (write low byte, then high byte)

rGBU_OBPI ($FF6A)  OBJ palette index (same structure)
rGBU_OBPD ($FF6B)  OBJ palette data
```

Linear palette index layout:
```
Index 0–5:   Palette 0, colours 0–5
Index 6–11:  Palette 1, colours 0–5
...
Index 66–71: Palette 11, colours 0–5
```

Colour 0 of every OBJ palette is **always transparent** (same as GBC).
Colour 0 of BG palettes is the background fill colour for that palette.

### Colour Format: RGB565

The GBU uses RGB565 (5 bits red, 6 bits green, 5 bits blue) stored as a
16-bit little-endian value, matching the ST7796S display controller natively.
This differs from the GBC's 15-bit BGR555 format.

```
RGB565 layout (16 bits):
  Bits 15–11: Red   (0–31)
  Bits 10–5:  Green (0–63)
  Bits 4–0:   Blue  (0–31)
```

---

## 6. Graphics — Sprites (OAM)

**64 sprites maximum** (up from 40 on DMG/GBC)
**10 sprites per scanline limit** (same as DMG/GBC, unless disabled via config)
**Sprite sizes: 8×8 or 8×16** (LCDC bit 2, same as GBC)

OAM layout per sprite (4 bytes, same as GBC):
```
Byte 0: Y position (sprite top + 16)
Byte 1: X position (sprite left + 8)
Byte 2: Tile index (0–511 via attribute bit 4)
Byte 3: Attributes
  Bit 7:   BG/OBJ priority
  Bit 6:   Y flip
  Bit 5:   X flip
  Bit 4:   Tile bank (0=tiles 0-255, 1=tiles 256-511)
  Bits 3-2:(reserved)
  Bits 1-0: Palette number (0–11; note only 2 bits, see below)
```

**OBJ palette addressing note:**
With 12 OBJ palettes and only 2 bits in the attribute byte, palettes 0–3 are
directly addressable. Palettes 4–11 require the `rGBU_OBJ_PAL_HI` register
($FF66) which provides 2 additional bits, giving a 4-bit effective palette
select. This is a deliberate hardware quirk — accessing the upper palettes
costs an extra register write, making them slightly more expensive to use.
Sprite overlaying (the Mega Man technique) is fully valid and expected.

---

## 7. Memory Map

```
Address Range   Size    Description
─────────────────────────────────────────────────────────────
$0000–$3FFF     16KB    ROM Bank 0 (fixed)
$4000–$7FFF     16KB    ROM Bank N (switchable via MBC)
$8000–$BFFF     16KB    VRAM — tile data (flat, no banking)
$C000–$DFFF      8KB    VRAM — tile maps (BG + Window)
$E000–$FFFF     The following is new/changed vs DMG/GBC:
$E000–$FFDF     8KB     WRAM lower bank (always mapped)
$FFE0–$FFFF           (see below — high page)

High page ($FF00–$FFFF):
$FF00           JOYP — joypad (same as DMG)
$FF01–$FF02     Serial (same as DMG, not implemented in V1)
$FF04–$FF07     Timer DIV/TIMA/TMA/TAC (same as DMG)
$FF0F           IF — Interrupt Flag (same as DMG)
$FF10–$FF26     APU — DMG sound registers (preserved exactly)
$FF30–$FF3F     Wave RAM (CH3 waveform, same as DMG)
$FF40–$FF4B     PPU registers (LCDC, STAT, SCY, SCX, LY, etc.)
$FF4F           VRAM bank — not used (GBU has flat VRAM)
$FF60           rGBU_ID — returns $47 ('G'), confirms GBU hardware
$FF61           rGBU_FLAGS — system mode and feature flags
$FF62           rGBU_INPUT — shoulder buttons L/R
$FF63           rGBU_SCX_HI — SCX bit 8 (high scroll bit)
$FF64           rGBU_SCY_HI — SCY bit 8
$FF65           rGBU_MAP_CFG — map size configuration
$FF66           rGBU_OBJ_PAL_HI — OBJ palette high bits
$FF67           (reserved)
$FF68–$FF6B     Palette RAM access (BGPI/BGPD/OBPI/OBPD)
$FF6C           rGBU_VOL_APU — DMG APU master volume
$FF6D           rGBU_VOL_SID — SID channels master volume
$FF6E           rGBU_VOL_MOD — MOD stream master volume
$FF6F           rGBU_VOL_EXT — External input volume
$FF70–$FF7D     SID synthesis registers (CH5 + CH6)
$FF7E–$FF7F     (reserved)
$FF80–$FF8A     MOD stream control registers
$FF8B–$FF8F     (reserved)
$FF90–$FF9B     Per-channel volume and fade (CH1–CH6)
$FF9C–$FF9F     (reserved)
$FFA0–$FFAF     MOD sub-channel volume (channels 0–15)
$FFB0–$FFBF     MOD sub-channel fade (channels 0–15)
$FFC0–$FFFE     HRAM (high RAM, same as DMG)
$FFFF           IE — Interrupt Enable (same as DMG)
```

**WRAM: 32KB flat**
The GBU provides 32KB of WRAM with no bank switching required.
It is mapped across two regions to avoid displacing the existing
high page: $E000–$FFDF (lower 8KB) and an extended bank at
$D000–$DFFF via `rGBU_WRAM_EXT` ($FF70, GBU mode only — shadows
the SID register range only when WRAM extension is active).

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
$FF70  rSID_CH5_FREQ_LO   CH5 frequency low byte
$FF71  rSID_CH5_FREQ_HI   CH5 frequency high byte
$FF72  rSID_CH5_WAVE      Bits 1-0: waveform select
$FF73  rSID_CH5_ADSR      Bits 7-4: attack, bits 3-0: decay
$FF74  rSID_CH5_SR        Bits 7-4: sustain, bits 3-0: release
$FF75  rSID_CH6_FREQ_LO
$FF76  rSID_CH6_FREQ_HI
$FF77  rSID_CH6_WAVE
$FF78  rSID_CH6_ADSR
$FF79  rSID_CH6_SR
$FF7A  rSID_FILTER_FREQ   Filter cutoff frequency (0–255)
$FF7B  rSID_FILTER_RES    Bits 7-4: resonance, bits 1-0: mode
$FF7C  rSID_FILTER_ROUTE  Bit 0: CH5 → filter, Bit 1: CH6 → filter
$FF7D  rSID_RINGMOD       Bit 0: ring mod CH5 × CH6
```

---

## 11. Audio — MOD Streaming

One MOD/XM format audio stream plays from the SD card, decoded on
the RP2350's second core. This provides cinematic, richly arranged
music independent of the synthesis channels.

### File Format: XM (Extended Module)

XM was chosen over raw PCM or OGG for the following reasons:
- Fits the "freedom through limitation" philosophy — XM is itself a
  constrained creative medium with fixed channels and pattern-based composition
- Compact file sizes (a complete game soundtrack may fit in 200KB)
- Supports up to 32 channels, each independently controllable at runtime
- Sounds distinctly different from both the DMG APU and SID channels,
  giving composers three different sonic textures to layer
- Widely supported by tracker software (OpenMPT, MilkyTracker, etc.)

The GBU caps XM channel control at **16 sub-channels** (channels 1–16
of the XM file). Channels beyond 16 play normally but are not individually
addressable via GBU registers.

### Track Index File: MUSIC.LST

Tracks are referenced by index number (0–255). A plain-text index file
on the SD card root maps indices to filenames:

```
; MUSIC.LST — one entry per line
; Format: filename (relative to SD root)
; Track index = line number (0-based), comments start with ;

TOWN.XM
DUNGEON.XM
BOSS.XM
ENDING.XM
```

A game writes `rMOD_TRACK = 2` to select `BOSS.XM` (line 2).

### MOD Stream Registers
```
$FF80  rMOD_CMD      Write: 0=stop, 1=play, 2=pause, 3=loop
$FF81  rMOD_TRACK    Track index (0–255, see MUSIC.LST)
$FF82  rMOD_STATUS   Read-only: 0=stopped, 1=playing, 2=track ended
$FF83  (reserved for seek/position, future use)
```

### Adaptive Music via Sub-Channel Control

The primary creative mechanism for adaptive music is **selective muting
and fading of individual XM sub-channels at runtime**, not switching
between files. A composer authors one XM file with all musical layers
present; the game reveals them dynamically based on game state.

Example XM channel layout convention (project-specific, document in
your own `game_audio.inc`):
```
XM ch 1:  Strings bed     — always present
XM ch 2:  Brass section   — builds with tension
XM ch 3:  Piano melody    — emotional highlight
XM ch 4:  Bass line       — enters with action
XM ch 5:  Percussion      — enters with action
XM ch 6:  Ambient texture — atmospheric layer
```

Sub-channel volume and fade registers are described in section 12.

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
```asm
; Enter haunted forest — begin with MOD bed only, all synth silent
    ld a, MOD_LOOP
    ld [rMOD_CMD], a

    xor a                        ; all synth channels silent
    ld [rGBU_VOL_CH1], a
    ld [rGBU_VOL_CH2], a
    ld [rGBU_VOL_CH3], a
    ld [rGBU_VOL_CH4], a
    ld [rGBU_VOL_CH5], a

; Mute MOD brass and percussion sub-channels (too lively for forest)
    ld hl, rMOD_CH1_VOL          ; brass on ch 2 (index 1)
    ld [hl], a
    ld hl, rMOD_CH4_VOL          ; percussion on ch 5 (index 4)
    ld [hl], a

; Tension rises — bass and melody enter with slow fade
    ld a, 180
    ld [rGBU_VOL_CH3], a         ; wave bass target
    ld [rGBU_VOL_CH1], a         ; pulse melody target
    ld a, 90                     ; 90 frame (~1.5s) fade in
    ld [rGBU_FADE_CH3], a
    ld [rGBU_FADE_CH1], a

; Climax — SID pad swells, MOD brass returns
    ld a, 200
    ld [rGBU_VOL_CH5], a
    ld a, 60
    ld [rGBU_FADE_CH5], a
    ld a, 160
    ld [rMOD_CH1_VOL], a         ; brass back in
    ld a, 45
    ld [rMOD_CH1_FADE], a
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
Internally they are standard Game Boy ROM format (the same binary
layout the GBX emulator loads via MBC), with GBU identification
provided by the ROM header.

### Physical Cartridge Options (custom board)

Three NOR flash chip packages are supported on the universal cartridge PCB,
all from the Winbond W25Q series with identical QSPI command sets:

| Part Number | Package | Capacity | Notes |
|---|---|---|---|
| W25Q128JVSIQ | SOIC-8 | 128Mbit (16MB) | Smallest, easiest to solder |
| W25Q256JVEIQ | WSON-8 | 256Mbit (32MB) | Compact, fine-pitch |
| W25Q256JVFIQ | SOIC-16 | 256Mbit (32MB) | Wide body, hand-solderable |

All three use the same 6-signal QSPI interface (CS, CLK, IO0–IO3).
The cartridge PCB provides footprints for all three; only one is
populated per board. The GBX firmware auto-detects the chip via
JEDEC ID and reads using the W25Q Fast Read command.

Pin 31 (External Audio Input) is routed from the cartridge edge
connector to the audio mix on the main board.

---

## 15. ROM Header

The GBU ROM header is a superset of the standard Game Boy header.
All standard header fields at their standard offsets are preserved.

### GBU Identification

**Byte $014C** (Mask ROM number — unused/zero in standard homebrew):
```
$47 ('G') = This is a GBU ROM
$00       = Standard GB/GBC ROM (default)
```

The emulator reads $014C after $0143 (GBC flag) to determine the
target hardware. If $014C = $47, GBU mode is enabled regardless
of the GBC flag value.

### Standard Header Fields (unchanged)
```
$0100–$0103  Entry point (NOP + JP to game start, same as always)
$0104–$0133  Nintendo logo (required for boot, same bytes)
$0134–$0143  Title (15 chars + GBC flag)
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
Games should declare themselves as GBU targets in their RGBDS source:
```asm
SECTION "Header", ROM0[$0100]
    nop
    jp EntryPoint
    ; Nintendo logo
    DB $CE,$ED,$66,$66,$CC,$0D,$00,$0B,$03,$73,$00,$83,$00,$0C,$00,$0D
    DB $00,$08,$11,$1F,$88,$89,$00,$0E,$DC,$CC,$6E,$E6,$DD,$DD,$D9,$99
    DB $BB,$BB,$67,$63,$6E,$0E,$EC,$CC,$DD,$DC,$99,$9F,$BB,$B9,$33,$3E
    ; Title
    DB "MY GBU GAME     "  ; 15 chars + GBC flag ($80 = GBC enhanced)
    DB $00,$00             ; New licensee
    DB $00                 ; SGB flag
    DB $01                 ; MBC type (MBC1)
    DB $05                 ; ROM size (1MB)
    DB $03                 ; RAM size (32KB)
    DB $01                 ; Destination
    DB $33                 ; Old licensee (use new system)
    DB $47                 ; **GBU identifier**
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
                      Bits 2–7: Reserved
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

### Extended OBJ Palette
```
$FF66  rGBU_OBJ_PAL_HI  R/W. Bits 1–0 = OBJ palette select bits 3–2.
                         Combined with attribute bits 1–0 for full 0–11 range.
```

### Palette RAM
```
$FF68  rGBU_BGPI     R/W. BG palette index. Bit 7 = auto-increment.
$FF69  rGBU_BGPD     R/W. BG palette data (RGB565, low byte first).
$FF6A  rGBU_OBPI     R/W. OBJ palette index.
$FF6B  rGBU_OBPD     R/W. OBJ palette data.
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
$FF70  rSID_CH5_FREQ_LO   CH5 frequency low byte
$FF71  rSID_CH5_FREQ_HI   CH5 frequency high byte
$FF72  rSID_CH5_WAVE      Bits 1–0: waveform (0=saw,1=tri,2=pulse,3=noise)
$FF73  rSID_CH5_ADSR      Bits 7–4: attack (0–15), bits 3–0: decay (0–15)
$FF74  rSID_CH5_SR        Bits 7–4: sustain (0–15), bits 3–0: release (0–15)
$FF75  rSID_CH6_FREQ_LO
$FF76  rSID_CH6_FREQ_HI
$FF77  rSID_CH6_WAVE
$FF78  rSID_CH6_ADSR
$FF79  rSID_CH6_SR
$FF7A  rSID_FILTER_FREQ   Filter cutoff frequency (0–255)
$FF7B  rSID_FILTER_RES    Bits 7–4: resonance (0–15), bits 1–0: mode
$FF7C  rSID_FILTER_ROUTE  Bit 0: CH5→filter, bit 1: CH6→filter
$FF7D  rSID_RINGMOD       Bit 0: ring modulation CH5×CH6
```

### MOD Stream
```
$FF80  rMOD_CMD      R/W. 0=stop, 1=play, 2=pause, 3=loop.
$FF81  rMOD_TRACK    R/W. Track index (0–255, references MUSIC.LST).
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

### RGBDS

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
before enabling GBU features:

```asm
CheckGBU:
    ld a, [rGBU_ID]
    cp GBU_ID_BYTE          ; $47 = 'G'
    jr nz, .not_gbu
    ; GBU hardware confirmed — enable extensions
    ld a, %00000011         ; widescreen + fast CPU
    ld [rGBU_FLAGS], a
    ret
.not_gbu
    ; Running on DMG/GBC — use standard feature set
    ret
```

### Audio File Preparation

XM files are created with any tracker supporting the XM format:
- OpenMPT (Windows, free) — recommended
- MilkyTracker (cross-platform, free)
- Renoise (commercial, XM export)

Place XM files in the SD card root. Create `MUSIC.LST` listing one
filename per line (0-indexed). Comments start with `;`.

```
; MUSIC.LST
TITLE.XM
OVERWORLD.XM
DUNGEON.XM
BOSS.XM
; index 4 intentionally empty — reserved
```

---

## 18. Implementation Notes (Emulator)

Notes for the GBX emulator implementation. Not relevant for game developers.

### Files affected by GBU implementation

| File | Changes required |
|---|---|
| `cpu.h` | GBU palette RAM, extended VRAM/WRAM sizes, 64-sprite OAM |
| `bus.inc` | GBU register reads/writes ($FF60–$FFBF) |
| `ppu_utils.inc` | GBU render path: 64×64 map, 12-palette decode, 240px width |
| `mbc.inc` | Check $014C for GBU identification in `mbc_init_from_header` |
| `system_utils.inc` | CYCLES_PER_FRAME switches to 67,002 in GBU mode |
| `display.inc` | Already handles GBU widescreen viewport (no change needed) |
| `gbx.cpp` | No changes needed |

### GBU mode activation sequence
1. `mbc_init_from_header()` reads $014C → sets `cpu.mode = MODE_GBU`
2. CPU reset posts to GBU register defaults (all volumes 255, all fades 0)
3. `system_run_frame()` uses `CYCLES_PER_FRAME_GBU` = 67,002
4. `update_ppu()` uses 240px render width and 160 visible scanlines
5. `bus_read/write` routes $FF60–$FFBF to GBU register handlers

### Cycles per frame in each mode
```c
#define CYCLES_PER_FRAME_DMG  17556   // 4.194 MHz / 59.7 fps / 4
#define CYCLES_PER_FRAME_GBC  35112   // 8.389 MHz / 59.7 fps / 4
#define CYCLES_PER_FRAME_GBU  67002   // 16.0  MHz / 59.7 fps / 4
```

### Audio implementation priority
The audio subsystem is not yet implemented. When it is, the recommended
implementation order is:

1. DMG APU (CH1–CH4) — most games depend on this first
2. Per-channel volume/fade registers — needed for GBU adaptive music
3. MOD streaming on Core 1 — XM decoder + MUSIC.LST reader
4. MOD sub-channel volume control
5. SID synthesis (CH5–CH6) — last, most complex

### Open TODOs
- STAT mode bits (0–3) within scanline not yet tracked in `update_ppu()`
- LYC==LY coincidence interrupt not yet implemented
- MBC3 RTC stubbed — needed for Pokémon Gold/Silver
- Cart RAM persistence (.sav file on SD card)
- SGB border support (ROM header $0146 = $03)
- Custom border support for non-SGB games
- GBC colour mode (MODE_GBC) not yet emulated
- Audio subsystem (all of the above)
- Touch screen driver (FT6336U, I2C)
- ROM selector subdirectory browsing
- Theme support for ROM selector menu

---

*GBX Emulator / Game Boy Ultra — specification document*
*Maintained alongside the emulator source in the project repository*
