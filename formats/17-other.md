# 17. Other formats

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

| Format | State |
|---|---|
| PSO (`PSIN`; sections LSCH/LENM/LSNS/LGID/LRSC/CHKS) | **decoded** (2026-11-21): the plain-file form of the meta format of [§8](08-meta.md). Section headers are big-endian; the block table and references are little-endian; the `LSCH` schema names fields with 32-bit joaat (struct names are 64-bit). 2,225 of 2,281 files in `data.rpf` decode. See [PSO](#pso) and [Vehicle handling](#vehicle-handling) |
| RBF (`RBF0`) | a new variant with u16 indices and inline names; exported raw |
| `*_bvh4.rpf` | Xbox GPU ray-tracing acceleration structures and mount manifests. See [BVH4 archives](#bvh4-archives). |
| XML, text | plain; exported as is |
| shaders (`shaders.rpf`) | native RDNA2 programs, XDAT stage associations and PSV0 bindings. See [structure/06](../structure/06-models-shaders.md). |
| audio (AWC/ADAT) | container layout, chunk tags and Opus framing in [structure/03](../structure/03-audio-awc.md); not exported by the tool |
| animation (`anim.rpf`) | clip dictionaries and cutscene metadata; clip metadata decodes (confirmed); skeleton not in the clip dictionaries. See [Animation](#animation) |
| ped body shape | presets in `common.rpf`, a control-to-input PSO and pose rigs in `anim.rpf`; the rig program runs (confirmed). See [Ped body shape](#ped-body-shape) |
| music catalogue (`audio:/config/music.dat`) | title, artist and playlist records; title hash solved (confirmed). See [Music catalogue](#music-catalogue) |
| PSIN scene placements (`story000_script.rpf`) | 97,504 64-byte placement records (confirmed). See [PSIN scene placements](#psin-scene-placements) |
| movies (`movies.rpf`) | Bink 2: 515 movies, 505 `KB2n` and 10 `KB2j`, stored uncompressed; header, frame index and Bink Audio tracks parse (confirmed); video needs a Bink 2 decoder. See [Video](#video) |
| UI graphics | named texture dictionaries in `data.rpf` and UI texture stores. See [UI graphics](#ui-graphics) |
| text (GXT) | `0DRP` type `0x7a` key/text resources; 54,118 keys (confirmed). See [Text (GXT)](#text-gxt) |

## BVH4 archives

**Confirmed.** The BVH4 entries are dehydrated Xbox GPU ray-tracing
acceleration structures. CPU collision uses the separate
[`phBound` resources](13-collision.md). Each carved BVH4 entry consists of
adjacent zlib streams for a 208-byte system block and a graphics block.
The system block stores a 64-bit asset hash at `+0x40` and the graphics
size at `+0xA0`.

A 64-byte version-6 BVH4 header starts at graphics `+0x100`. The size
identity `0x140 + totalSizeIn64BNodes * 64` matches the declared graphics
size on all 1,233 full-carved entries in `shared_minigame000_bvh4.rpf`
and all 1,412 entries in a held-out `core_bvh4.rpf` sample.
Hierarchy words include 24-bit values and an 8-bit count. Leaf lists
contain packed 24-bit indices. Rehydration needs the external geometry
buffers to reconstruct runtime leaf vertices and bounds. The asset-hash
join to the source drawable remains open.

`story000_bvh4.rpf` contains four XML streams registering the environment
and per-tile terrain BVH packs as streamed map content.

## PSO

PSO files are the plain-file form of the self-describing meta format
([§8](08-meta.md)). Each section is a 4-byte magic and a **big-endian** u32
size that includes the 8-byte header
([structure/04](../structure/04-pso-objectgraph.md)). Everything inside is
little-endian.

| Section | Contents | Confidence |
|---|---|---|
| `PSIN` | instance data: a block table from +8, entries {u32 struct hash (low 32 bits of the 64-bit struct name), u32 length, 8 × `0xAA`}, then the blocks, each 16-byte aligned. References are u64 `(offset << 20) \| block` with a 1-based block | confirmed |
| `LSCH` | schema: 32-byte struct entries {u64 name hash, u32 info, u32 0, 8 × `0xAA`, u32 size, u16 0, u16 member count}, then all members in table order as 16-byte records **identical to §8**: {u32 name, u32 offset, u8 type, u8 sub, u16 reference, u32 extra} | confirmed |
| `LRSC` | a copy of the meta resource header (`0DRP`, type 0x7a): +0x38 u16 struct count, +0x3A u16 root block, +0x3C u16 block count | confirmed |
| `LENM` | enums, 64-bit names | not decoded |
| `LSNS`, `LGID`, `CHKS` | — | not decoded |

- **Names**: struct names are 64-bit hashes (function unknown). Member
  names are 32-bit **case-sensitive joaat**, and GTA V's names match
  (`fMass`, `fInitialDragCoeff`, `fGearRatios`, `fSteeringLock`). A struct
  member's `extra` is the low 32 bits of the struct name (confirmed).
- **info**: 0x200 / 0x300 / 0x400 = alignment 4 / 8 / 16; low byte 0x0C or
  0x24 marks generated helper types such as hash-table storage, named
  `<hash32>_000001xx` (inferred).
- **Pointers** are always pool references. There are no `0x5xxxxxxx`
  resource pointers; a reference whose offset is 0x500–0x5FF looks like
  one, so do not test for them (confirmed).
- In `data.rpf`, 2,225 of 2,281 PSO files decode this way. The other 56 use
  the older GTA V layout (`PSIN` / `PMAP` / `PSCH`) and are not decoded.

Type codes not in [§8](08-meta.md):

| Code | Type | Confidence |
|---|---|---|
| 0x01 | bool (1 B) | confirmed |
| 0x4d | 64-bit hash or id (8 B) | confirmed |
| 0x50 | inline fixed array: `extra` elements of member `reference` | confirmed |
| 0x75 | map {u32 count, u32 capacity, u64 ref} to a hash-table struct: control bytes `b8794c89` (u8 × N; 0x80 = empty, else 7 hash bits, as in a Swiss table) and slots `93c495af` {`6098a50e` key, `063fa3f2` value} | confirmed |
| 0x62 | tagged value (16 B: u32 tag, then the payload) | inferred |
| 0x31 | 4 floats | inferred |

In PSO, `0x65` bit sets take 16 bytes (the first u64 is used) and `0x44`
takes 16 bytes, probably a string reference into `LSNS` (not decoded).

## Vehicle handling

One PSO in `data.rpf` holds the handling data: 1,097 records of struct
`1ddb46e190779a14` (confirmed).

| Field | Meaning |
|---|---|
| `93db5464` | joaat of the lower-case handling name (`phoenix`); 301 match model names |
| `49a01a86` | a second name hash, often 0 |
| `41c8dec7` | array of polymorphic components, one struct type per component |

| Component | Contents (inferred from the field names) |
|---|---|
| `b4e0f8b0` | body: `fMass`, `fInitialDragCoeff`, 3-vectors |
| `6fdd2f18` | transmission: `fGearRatios`, `fFinalDriveRatio`, `fDriveBias*` |
| `88d3a1a3` | engine: RPM limits (e.g. 5,800), a torque curve as a map of {rpm, value} records, `fOilVolume` |
| `70317f27` | fuel |
| `edf48412` | wheels, steering and brakes: `fSteeringLock`, `fBrakeForce`, `fWheelMass` |
| `bdaf2dc0` | suspension: travel, damping |
| `c746f5ed`, `82173b9e`, `0da11ed9` | unknown |

Mass, drag, gear ratios, final drive, brakes, steering lock, wheel mass,
suspension travel and damping, anti-roll bar force and fuel are named; most
other fields, the torque curve's included, are decoded values with hashed
names. The record's root block is a small map of two keys to 64-bit ids.

Another PSO holds lights and sirens (`Lights`, `Sirens`: cone angles,
intensity, flash timings), like GTA V's `carcols`. The paint colours are in
the game-data meta resource instead ([§11](11-vehicle-paint.md#vehicle-tables)).

## Video

`movies.rpf` has a plain payload (only the table of contents is encrypted,
as everywhere), and its videos are **stored uncompressed**, so a search for
zlib streams finds nothing in it (confirmed).

Each video is a standard Bink 2 file starting on a 4 KiB boundary; the next
starts at the first 4 KiB boundary after the previous one ends. That gives
515 files, the entry count in the archive header, covering 8,831,336,892 of
the archive's 8,832,481,776 bytes: 505 `KB2n` and 10 `KB2j`, about 236
minutes, all at 30 fps (confirmed).

Header (the public Bink container layout; all u32):

| Off | Field |
|---|---|
| 0 | signature (`KB2n`, `KB2j`) |
| 4 | file size − 8 |
| 8 | frame count |
| 12 | largest frame size |
| 16 | frame count again |
| 20, 24 | width, height |
| 28, 32 | fps numerator, denominator |
| 36 | video flags (`0x40013` typical) |
| 40 | audio track count n |
| 44 | Bink 2 only: one more u32 (`0005e987` seen) |
| then | n × u32 largest decoded audio size, n × (u16 sample rate, u16 flags; `0x7000` on 48 kHz stereo), n × u32 track id |
| then | (frame count + 1) × u32 frame offsets: bit 0 = keyframe, the last one is the file size |

Every frame table checks out against the file size.

| Resolution | Files |
|---|---|
| 1280 × 720 | 246 |
| 1920 × 1080 | 130 |
| 720 × 1280 (portrait) | 67 |
| 512 × 512 | 23 |
| 1024 × 1024 | 19 |
| 2560 × 1440 | 14 |
| 512 × 256, 2048 × 558 | 2 each |

Audio: 319 files have none, 165 one track, 30 two, 1 four; all 48 kHz
stereo. The longest video runs 534 s. Original file names are not
recovered.

## UI graphics

- **Named texture dictionaries in `data.rpf`** (75): 36 loading screens,
  `hud`, `blip`, `map_scripted_movie`, `generic_movie`, `credits`, `phone`,
  `controller_test`, `analog_instruments`, `generic_instruments` and 25
  `vit_*` vehicle gauge sets (dashboards, motorcycle displays). Their 880
  textures decode with the dictionary rules of [§3](03-texture-dictionaries.md)
  and [§4](04-xbox-tiling.md); texture names are placeholders (`hud_i8d3`).
- **UI texture store** `data/ttdb/ui/ttdb_ui.ttpf` ([§5](05-ttpf.md)): 10,013
  textures, one mip each, named only by 64-bit hash. Mostly sprite atlases:
  shop and brand logos, tattoo art, hair, beard and wheel icons, character
  portraits, in-game arcade and website art. Formats: BC1 3,801, BC7 3,383
  plus 1,828 sRGB, BC3 928, BC4 57, A8 16. With BC1 BCPack decoded
  ([§6](06-bcpack.md)), 9,012 decode completely, 345 partly (some 64 KiB
  chunks are BCPack in another format: the top of the image decodes, the
  rest is blank) and 656 lie only in such chunks, mostly BC7.
- **Fonts and HUD shapes**: `data/ttdb/graphics` holds single-channel font
  glyph atlases and HUD shapes.
- **Minimap / pause map**: no tile set found in these stores or
  dictionaries. The map may be drawn from world data at runtime (not
  established). Menu layouts (which sprite goes where) are not decoded.

## Animation

`anim.rpf` holds two families. Bodies in the low region of the archive are
`0DRP` v122 cutscene/scenario metadata; above them, bodies whose
`+0x10..+0x3F` is all zero are clip dictionaries (inferred from a sample of
seven windows).

Clip names, durations and sequence channels decode (confirmed). Sampled rotation
tracks validate as unit quaternions: 3.95 M values, worst |q| − 1 about 1.2e-7
(confirmed). The skeleton is not stored in the clip dictionaries.

## Ped body shape

Body shape is a **sparse additive pose rig** on helper bones (`BM_*`,
`MH_*`, `SEC_*`, `RB_*`) of the shared ped skeleton
([§7.5](07-models.md#ped-skinning-bone-palette-confirmed)); the clothing is
skinned to those bones. Three pieces:

**Presets** (plain XML in `common.rpf`, `CPedVariationBodyTypeManager`):
height presets (1.52–2.02 m), scale profiles and 16 body-shape presets
(THIN, HALFTHIN, REGULAR, HALFHEAVY, HEAVY, MUSCLE*, BELLY*, HEAVY_EXTREME,
DANCERFEMALE, LINEBACKERMALE …). Each preset is a list of controls:
`gcweight<region>` (−1 thin to 1 heavy) and `gcmuscle<region>` (0 to 1) for
back, belly, butt, calf, chest, forearm, hip, muffin, neck, shoulder,
thigh and upperarm, plus `GCBreasts`, `GCCleavage` and `GCButt`. REGULAR is
all 0 (confirmed).

**Controls → rig inputs** (a preset PSO in `anim.rpf`: 922 of its 8,192
items carry rig inputs; confirmed). A negative `gcweight` drives the region's
thin input by |v|, a positive one its heavy input by v; `gcmuscle` drives one
input directly.

| Region | Heavy | Thin | Muscle |
|---|---|---|---|
| back | `722b3991` | `e79422cf` | `083e8443` |
| belly | `efc858c0` | `74a5940e` | `860a629a` |
| butt | `b65f7951` | `1d70bfc2` | `2267feeb` |
| calf | `4c92b04a` | `b3925c52` | `798aa11f` |
| chest | `fbc18785` | `47164eb7` | `f8ca1901` |
| forearm | `7e5f130b` | `49c5ac98` | `33c4e1ea` |
| hip | `0978bf99` | `596b89f1` | `fc1aaed3` |
| muffin | `b0c81e7f` | `9ffec641` | `3f088cd7` |
| neck | `334633c1` | `efc4dbff` | `97d111cf` |
| shoulder | `6d8264ca` | `fb039566` | `ab82da7b` |
| thigh | `525a6395` | `7636ca63` | `9e37dc2e` |
| upperarm | `c8a41369` | `5a7029e5` | `37657a6b` |

`GCButt` drives `5a502ae2`; `GCBreasts` drives `377b0101` when positive and
`48f90678` when negative. The input names themselves are unresolved.

**Rig dictionaries** in `anim.rpf` (magic `RIG` v2 at +0x70): one with 1,256
rigs (`anim/rig/generated/…`) and one with 543 (`anim/rig/canonical/…`).
Header +0x48 points to 0x50B hash buckets; each entry is {u32 joaat of the
full rig path, u64 pointer to a 0x30-byte wrapper (+0x10 count 1, +0x20
pointer to the rig), u64 next}. Body rigs include
`LOD0`–`LOD3_Body_Shapes_Female`, `Body_PoseDeformation_Female`,
`Dancer_Body_Shapes_Female`, `BreathingStyle0_Body_Shapes_Female`, XXL and
bodybuilder variants and garment rigs (`*_Garment_Shapes_Female`).

A rig is wrapped as `RIG` + byte 0x02, a u32 size (= the inner blob's
size), and the `RIG0` blob at +0x100: version bytes, size, a section count
at +0x10 and section offsets from +0x14. Sections are tagged {u32 type hash,
u32 size}:

| Section | Contents | Confidence |
|---|---|---|
| `e636c3e9` | block table: count, register count (1,252 floats on `LOD0_Body_Shapes_Female`), then u32 entries `type << 24 \| offset` into the program body; type 0xFF ends | confirmed |
| `2316833f` | inputs: records {u8 type, u8 kind, u16 register, u32 hash}. Kind 3 = a bone track (joaat bone name; type 0/1/2 = translation / rotation / scale), kind 1 = a float input | confirmed |
| `7224f987` | outputs: the same bones | confirmed |
| `b00f0959` | the program (about 25 KB); its body starts after type, size and u32 `0x00020000` | confirmed |
| `74d00508` | 5 named blocks {hash, offset, size} | inferred |
| `6257c371` | the rig path | confirmed |

Program blocks:

| Type | Block | Confidence |
|---|---|---|
| 0x85 | constants: u16 first register, u16 count, then f32 values (registers 0, 1 = 0.0, 1.0) | confirmed |
| 0x46 | u16 dst, a, b: probably dst = a × b | inferred |
| 0x81 | u16 dst, src, then a 0..1 range: probably a clamp | inferred |
| 0x86 | one pass: u16 counts nT, nS, nQ, 0; then nT + nS + nQ u16 triples (track register, weight register, the track register again); then the deltas, 3 floats per translation and scale entry and 4 per rotation entry (unit quaternions). Each entry does T += w·dT, S += w·dS or q = q·dq^w on the track, in place; a track appears at most once per pass | confirmed |

`LOD0_Body_Shapes_Female` has 12 passes (360 down to 3 entries) over 360
helper-bone tracks (translation vec3 at registers 8–429, scale vec3 at
430–775, rotation quaternions at 776–1229) and 53 float inputs. Most deltas
are on scale. The tracks are also inputs, so the rig adds to the animated
pose. `LOD0_Body_Shapes_Male` has 1,294 registers, 379 tracks and 53
inputs, 50 of them shared with the female rig, all the control inputs
among them.

**Axes.** Deltas are in the game's Z-up bone frames. For a glTF skeleton
converted with (x, y, z) → (x, z, −y), convert translation deltas to
(x, z, −y), scale to (x, z, y) and quaternions to (x, z, −y, w). With that,
all 196 left/right translation pairs mirror exactly, and the shapes look
right: heavy widens torso and pelvis and pushes the belly forward, thin
slims (checked by eye on a female top).

Related: about 210 resources in `packs.rpf` with hashed `GM_U01_*` bone
references are soft-body / secondary-motion data, not body shape: particles
(x, y, z, w with w 400–3000) and edges {u16 a, u16 b, f32 rest length, f32,
u32 flags} (inferred).

Open: the exact 0x46 / 0x81 ops, the lower-LOD rigs, the input names, and
why the belly spikes on some male T-shirts whose palettes name bones in
neither skeleton.

## Music catalogue

`music.dat` is compiled parsable data: a root count, then the magic u32
`0x045A88EA` at `0x08`, then type-tagged records with `0xAAAAAAAA` as the null
sentinel (confirmed).

A title record (tag `0xAAAAAA0C`) has a u32 title hash at `+0x08`, a char[64]
title at `+0x0C`, a u32 credit count at `+0x60`, then 8-byte credits
(confirmed).

The title hash is joaat of the **lowercased** title with seed `0xC3CCF6AD`
(804/804 titles, confirmed). AWC stream ids use unseeded joaat of the title, so
the two never coincide.

## PSIN scene placements

64-byte records at a 16-byte stride inside PSIN tag blocks: 97,504 records in
2,017 documents of `story000_script.rpf` (confirmed).

| Off | Meaning | Confidence |
|---|---|---|
| 0x00 | u32 handle, repeated at `0x30` | confirmed |
| 0x10 | f32 × 3 position, Z-up metres; x repeated at `0x1C` | confirmed |
| 0x20 | f32 × 3 rotation, radians; x repeated at `0x2C` | inferred (meaning) |
| 0x34 | u32 archetype = joaat(lowercase name) | confirmed |
| 0x38 | u64 id; zero in about half the records | inferred (meaning) |

The PSIN and [map-metadata](09-placements.md) records partly overlap.
In a sweep of `levels.rpf`, 953 of the 6,568 distinct PSIN hashes also
occurred as map archetype names, covering 12,441 PSIN records. The median
distance to the nearest same-hash map entity was about 518 m, compared with
1,089 m for a density-matched shuffled control. There were 1,076 matches
within 10 m, compared with 67 for the control. This supports a shared subset
of names. The two placement corpora have different coverage.

## Text (GXT)

GXT resources are `0DRP` type `0x7a`. A tag table points at a `0x101` key
table, `0xA8478889` locators, `0x956A9DF5` / `0x10075946` locale-variant groups
and items, and `0x10` text pages (confirmed). Keys are joaat, stored in 4,096
or 2,048 buckets (confirmed).

---

← [16. Road graph, junctions and flags](16-roads.md) · [Index](README.md)
