# 7. Drawables, fragments and vehicles

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

Checked on a vehicle body and on several map buildings and road pieces:
exported GLBs opened in a glTF viewer (shaded, normals, UV checker,
textures, skeleton).

## 7.1 Buffer descriptor (system page)

| Off | Meaning | Confidence |
|-----|---------|-----------|
| 0x08 | u32 size in dwords | confirmed |
| 0x0C | u16 4 | unknown meaning |
| 0x0E | u16 8 or 9 | unknown meaning |
| 0x10 | flags: 0x1601 = mesh blob (vertices), 0x1380 = meshlet index buffer, 0x0982 = ? | confirmed (first two) |
| 0x18 | graphics pointer | confirmed |

## 7.2 Mesh blob (vertex buffer)

u32 magic `0x4042` at 0.

| Item | Meaning | Confidence |
|------|---------|-----------|
| stream table | u32 slot k at +4k, k = 0..22. Value = `offset_in_2_byte_units << 8 \| format`; byte offset = `(v >> 8) × 2`. Slot 0 `0x4042` → offset 0x80, format 0x42 | confirmed |
| 0x60 | f32 × 3 position base | confirmed |
| 0x6C | u32: the model axis held in the flat 16-bit z slot. 2 = z: slots hold (x, y, z), identity. 1 = y: slots hold (z, x, y). 0 = x: slots hold (z, y, x), a swap and not a rotation. Before 2026-11-20 mode 0 was read as (y, z, x), which tipped wall panels by 90°. SixNG also scores all six permutations against the stored normals. | confirmed (2026-11-20) |
| 0x70 | f32 × 3 position step | confirmed |
| vertex count | V = (smallest stream offset > 0x80 − 0x80) / 8 | confirmed |

**Vertex streams** (tightly packed arrays of V elements):

| Slot | Fmt | Size | Meaning | Confidence |
|------|-----|------|---------|-----------|
| 0 | 0x42 | 8 | position (below) | confirmed |
| 1 | 0x26 | 4 | RGBA8, probably vertex colour | inferred |
| 2 | 0x21 | 4 | normal, octahedral 10:10 | confirmed |
| 3 | 0x28 / 0x42 | 4 / 8 | UV0: 2 × s16 / 8192, or 2 × f32 | confirmed |
| 4 | 0x28 | 4 | UV1 | confirmed |
| 6 | 0x26 | 4 | skin weights, 4 × u8 (sum 255), with slot 8 format 0x25 | confirmed |
| 8 | 0x00 | **1 or 2** | skin joint: one bone index, weight 1. Width = section size / V (next section offset − this offset). 1 = u8, **2 = u16 LE** | confirmed |
| 8 | 0x25 | 4 | skin joints, 4 × u8 (wheels, tyres, ped clothing) | confirmed |
| 7 | 0x26 | 4 | ped meshes: a second set of 4 skin weights (the 8 weights sum to 255) | confirmed |
| 9 | 0x25 | 4 | ped meshes: a second set of 4 skin joints, with slot 7 | confirmed |
| 11 | 0x27 | 4 | tangent R10G10B10A2: xyz = v / 1023 × 2 − 1 (model space); A2 = 3 → bitangent sign w = −1 | confirmed |
| 12, 13 | 0x26 | 4 | RGBA8 | unknown |
| 14, 15 | 0x28 | 4 | UV2, UV3 | confirmed |
| 17 | 0x42 | 8 | ? | unknown |

**Position** (one u64 q per vertex; RAGE Z-up, metres):

```
x = ((((q >> 16) & 0xFFFF) << 8) | ( q       & 0xFF)) * step.x + base.x
y = ((((q >> 32) & 0xFFFF) << 8) | ((q >> 8) & 0xFF)) * step.y + base.y
z =  (q >> 48)                                       * step.z + base.z
```

Bits 0–15 hold the low byte of x (bits 0–7) and of y (bits 8–15), giving x and
y 24 bits of precision (confirmed: with them, decoded geometry reproduces the
stored per-geometry bounding boxes to about 1.5e-5 relative over 782
geometries; without them x and y are off by up to 255 steps).

**Axis rotation (confirmed):** the quantiser puts
the flattest model axis of each blob in the 16-bit z slot, and the header
u32 at +0x6C names that axis (corrected 2026-11-20):

| +0x6C | flat axis | slots (x, y, z) hold model axes | order |
|---|---|---|---|
| 2 | z | x, y, z | (0, 1, 2) |
| 1 | y | z, x, y | (2, 0, 1) |
| 0 | x | z, y, x (a swap, not a rotation) | (2, 1, 0) |

Mode 0 was earlier read as the cyclic (1, 2, 0). Where the stored normals
could not tell the two apart, wall panels and slabs came out tipped by 90°
(on one tower drawable, 25 of 51 geometries fell outside the drawable
bbox). To check a blob, score all six permutations by how well the face
normals agree with the stored normals (always model space) and keep the
header's order unless another wins by more than 0.05. On that tower the
header's order scores 0.91–1.0 on every geometry.

**Normal:** `x = (w & 1023) / 1023 × 2 − 1`, `y = ((w >> 10) & 1023) / 1023
× 2 − 1`, `z = 1 − |x| − |y|`; if z < 0 fold x and y (standard octahedral
unfold), then normalise. The top 12 bits are unknown.

**Float-position variant (confirmed):** cables and pool/fountain water.
Slot 0 is `0x3063` (format 0x63 = 3 × f32 at 0x60), with no base/step
header; V = (next stream offset − 0x60) / 12. Slot 2 (also 3 × f32) is the
normal on water, and the unit direction along the wire on cables. Cable
vertex pairs share a position (zero-width ribbon widened by the shader;
UV0.x = ±1 gives the side). SixNG widens cables into two crossed 2 cm
ribbons; the real thickness is a shader parameter not yet read.

## 7.3 Meshlet index buffer (flags 0x1380, confirmed)

| Off | Meaning |
|-----|---------|
| 0x00 | u16 meshlet count n |
| 0x02 | u16 ? |
| 0x04 | u16 triangle count t |
| 0x08 | u16 remap count r |
| **0x0C** | n × 16-byte descriptors: u32 ?, **u16 remap start**, u16 triangle start, u32 ? (cone?), 2 × f16 ? |
| 0x0C + 16n | t × u32 triangles: local indices in bits 0–6, 7–13, 14–20; meshlet number from bit 21 |
| … + 4t | r × u16 remap (local → vertex) |

Vertex = `remap[remap_start[meshlet] + local]`. Drop degenerate padding
triangles. Descriptors start at 0x0C, not 0x10. About 97 % of triangles
agree with the stored normals; flip the rest.

## 7.4 Geometry, model and LOD table

| Structure | Field | Confidence |
|---|---|---|
| geometry record (0xA0) | +0x08 ptr → mesh-blob descriptor; +0x40 ptr → holder whose first u64 points to the index-buffer descriptor | confirmed |
| model (stride 0x40) | +0x00 ptr → geometry-record ptrs; +0x08 u16 count, u16 capacity; +0x0C u16 (peds: bone-palette count); +0x10 ptr (bounds?); **+0x18 ptr → u16 shader index per geometry**; +0x20 ptr (peds: **bone palette**, [§7.5](#ped-skinning-bone-palette-confirmed)); on vehicles +0x0C / +0x20 hold something else (+0x0C = 295 on a car body, entries not sorted); +0x28 u32, u8 0xFF, u8, u16 count | confirmed (0x00, 0x08, 0x0C, 0x18, 0x20) |
| model-array header | 16 bytes {ptr → model-pointer array, u16 count, u16 capacity} | confirmed |
| LOD group | pointers to model-array headers at **+0x50, +0x58, +0x60, +0x68** of the main drawable (high, med, low, vlow; empty = 0). Main drawable = the root of a `.#dr`, the struct at root+0x18 of a `.#ft` | confirmed (2026-11-20) |
| embedded drawables | other model arrays belong to embedded drawables: fragment children (root+0x10, records 0xC0 apart, each with its own +0x50 list) and nested sub-drawables in buildings. They are parts, not LODs | confirmed layout; placement of fragment-child parts inferred |
| drawable bounds | root +0x30 / +0x40 bbox min / max (drawables only) | confirmed |
| fragment bounds | fragment root +0x18 → its drawable, whose +0x30 / +0x40 hold the bbox | confirmed |
| stored bbox reliability | not always the geometry's extent: on route-sign prefab fragments it is shifted about 4.3 m in x, and on one scaffolding model it is rotated 90° about z. The archetype definition's bbox matches the decoded geometry in both cases (as of 2026-11-20) | confirmed (2 cases) |

Each model struct is preceded by 8 bytes and referenced at struct − 8 from
an array of model pointers. On a vehicle body, LOD1 is the same parts with
about 3× fewer vertices (not a damage set). Ranking every model array by
file offset gives wrong LODs: on fragment props (streetlights, signs) a
base plate or bracket became LOD0 and the full pole LOD2, and on buildings
trim strips from sub-drawables became LOD1–3. Use only the LOD group. In a
sample, 8 % of drawables and 2 % of fragments have such parts (aircraft:
5–16, flat rotor-height pieces in model space). Map buildings usually have
a single LOD: use the placement LOD hierarchy ([§9.2](09-placements.md))
for distance LODs.

## 7.5 Skeleton (fragments, confirmed)

Bone record (0x50 bytes):

| Off | Meaning |
|-----|---------|
| 0x00 | f32 × 4 local rotation quaternion (x, y, z, w) |
| 0x10 | f32 × 3 local translation, then u32 hash |
| 0x20 | f32 × 3 local scale |
| 0x30 | ptr → name |
| 0x40 | u16 flags |
| 0x42 | u16 ? (3 or 7) |
| 0x44 | u16 next sibling (0xFFFF none) |
| 0x46 | u16 parent (0xFFFF root) |
| 0x48 | u16 index |
| 0x4A | u16 name hash |

Skeleton header, found by its pointer to the bone array:

| Off | Meaning | Confidence |
|-----|---------|-----------|
| +0x00 | ptr → bone array | confirmed |
| +0x08 | ptr → **inverse bind matrices** (4 × 4 f32 per bone; row-vector convention `p_bone = p_model · M`) | confirmed |
| +0x10 | ptr → local bind matrices (w lanes hold other data) | inferred |
| +0x18 | ptr → u16 parent indices | confirmed |
| +0x20, +0x28 | ptrs | unknown |

Inverting the inverse bind matrix gives each bone's model-space pivot.
Checked symmetric and in place on a car: wheels, bonnet and boot hinges,
door hinges, lamps. Vertices are in model space (bind pose). Most vehicle
parts use one bone per vertex; wheels and tyres up to 4.

**u16 bone indices (confirmed, found 2026-11-20).** Slot 8 keeps format
0x00 but is **2 bytes per vertex** on some geometries of skeletons with
more than 255 bones (a high-LOD car body has 295). The format code does not
say so; only the section width does. Reading it as u8 put interior, lamp,
window and bumper geometry on the wrong bones. SixNG writes `JOINTS_0` as
u16 when any index exceeds 255. **Vehicle GLBs exported before this fix
should be re-exported.**

Bone, joint and attach-point names (`suspension_lf`,
`bonnet_catch_l_pos1_attach` …) are plain strings in the system page.

### Ped skinning: bone palette (confirmed)

Ped clothing has **8 influences per vertex**: slots 8 / 6 hold the first
four joints and weights, slots 9 / 7 a second four; the 8 weights sum to
exactly 255 on every vertex (checked on a protagonist top).

Its joint indices do **not** index the drawable's own bones. They go
through the model's **bone palette**: model +0x0C u16 count, +0x20 pointer
to that many u32 joaat bone-name hashes, sorted ascending (80 on a
protagonist top at LOD0, 28 at LOD1). Vertex joint index k is palette
entry k. Checked by body side: 110 of 110 joints land on the right side
with the palette, 50 % with the bone order.

The palette names two kinds of bones:

- the drawable's own `GM_U01_*` girth helper bones (69 on that top,
  parentless, with parent-relative binds whose parent bone is not known);
- bones of a **shared ped skeleton** that is not in the clothing file: 596
  bones from `SKEL_ROOT`, with `BM_*`, `MH_*`, `RB_*` and `SEC_*` helpers,
  in the `ambient_femalebody` / `ambient_malebody` fragments of
  `packs.rpf`. `femalexxl` has the same bind pose.

Some T-shirts name 8–9 palette bones that are in neither skeleton. Heads use
another skin format (slots 8 / 9 with format 0x8c, 16 bytes per vertex):
slot 9 reads as 4 × u32 joint indices (up to 50 on a protagonist head),
slot 8 as 4 × (u16, u16) with plausible joints in the low halves, and
slots 6 + 7 weights sum to 255 on every vertex (inferred; not decoded).

### Ped heads: face textures (confirmed)

Head drawables (`ped_standard_head`) **bind no textures**, and every head
shares one UV layout: the face in the middle, the ears at the sides and a
teeth and gums strip along the bottom. The face comes from a material
preset ([§12](12-materials.md#material-presets-peds)):

- a face preset (Category `378ad10c` = joaat("heads")) whose tag map lists the head
  (key `4bfd27dd`, joaat of the full head drawable name) is that head's
  face;
- its "base" parameter holds the **albedo in slot `ead733fc`** (1024² BC1
  sRGB) and a **BC5 normal map in slot `d689b17f`**.

Example: `cs_jada_watts_head_fr1_afr_000_y_full` (`eb9b7bee`) → albedo
`4aedb007f2b1ca3d`, normal `7742ffa7e241bd32`.

| | Heads |
|---|---|
| all head drawables | 1,657 |
| with a face through a preset (897 drawables tagged) | 893 |
| crowd heads `hc_*`: tagged, but by presets with empty texture slots | — |
| `a_head_base_*` and some `cs_*` heads (e.g. `cs_lucia`): not tagged in `packs.rpf` (perhaps in `story000.rpf`, not checked) | — |

About 1,000 face albedos are ordinary textures in the `ttdb_peds_*_3`
stores. `peds_loose.ttpf` holds clothing patterns, head-space masks (beard,
brows, mouth) and red / green UV remap maps.

**Body shape** moves helper bones of the shared skeleton through a rig
([§17](17-other.md#ped-body-shape)). Other ped facts:

- `ped_standard_muv` (clothing) and `ped_standard_head` dominate; the
  `ped_standard*_blendshape*` shaders (about 300 uses) are only on rigid
  accessories (piercings, helmets, dressings).
- Hair, brows, lashes and beards are about 2 KB drawables with 3 vertices;
  the real hair is built at runtime.

## 7.6 Shaders (materials)

| Off | Meaning | Confidence |
|-----|---------|-----------|
| 0x00 | u32 material effect (shader) name hash; lowercase JOAAT (`standard` = `bcf03fc1`) | confirmed (all 344 shaders; [name hashes](12-materials.md#material-name-hashes)) |
| 0x04 | `"atem"` | confirmed |
| 0x08 | ptr → constant buffer (~0x160 bytes of floats) | inferred (layout unknown) |
| 0x10 | ptr → per-parameter slot pointers | confirmed |
| 0x20 | ptr → parameter table: u8 ?, u8 count, …, then count × (u32 param hash, u32 data offset); param hash = joaat of the lower-case name (`diffusetex` = `c8e8c282`) | confirmed |
| 0x38..0x3F | flags, see [§12](12-materials.md) | confirmed (bytes 1, 4) |

A texture parameter's slot points to a 0x50-byte struct whose +0x30
pointer leads to a 32-byte texture reference: ptr, **u64 texture hash**
(the `ttpf` key), u32 6, u32 hash.

## 7.7 Vehicles

- **Wheels** are not in the car fragment. They are shared models
  `rim_<class>_NNN` and `tyre_<class>_NNN` (with `_hi`/`_mid`) authored for
  the left-front wheel at unit scale (rim radius 0.5, tyre 0.88). The game
  mirrors and scales them per vehicle. Each spawn configuration names its
  rim and tyre, and the model info has `wheelScale` / `wheelScaleRear`
  ([§11](11-vehicle-paint.md#vehicle-tables)); 1.625 × `wheelScale` fits by
  eye, but the game's rule is not established. The wheel bones' pivots give the mount
  points.
- **Damage parts**: separate fragments `<car>_dmg_mod_<part>_NNN`.
  **Mod-kit parts**: `<car>_<part>_<kit>`. Both with `_hi`/`_mid`.

## Open questions

- Normal top 12 bits; slots 12, 13, 17.
- The parent bone of the `GM_U01_*` helpers; the head skin format (0x8c);
  palette bones found in neither skeleton.
- The face of heads no preset lists (`a_head_base_*`, some `cs_*`), and of
  crowd heads whose presets have empty slots.
- Occluder and water-vegetation LOD buffers (another format).
- Shader constant-buffer layout (would give cable thickness, paint and
  material constants).
- The derivation of the 64-bit texture keys; 152 of 229 parameter names.
- How damage parts are swapped in; how `wheelScale` sizes the shared wheels.


---

← [6. BCPack](06-bcpack.md) · [Index](README.md) · [8. Meta resources (type 0x7a)](08-meta.md) →
