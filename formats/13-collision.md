# 13. Collision and surface categories

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-20; confidence: confirmed / inferred / unknown.*

## 13.1 Bounds (GTA V `phBound` with small changes, confirmed)

About 87 K of ~98 K unnamed resources in `levels.rpf` are physics bounds
(`.ybn` equivalent).

| Off | Field |
|---|---|
| +0x10 | u8 type: **5 = geometry (BVH), 6 = composite** |
| +0x14 | f32 bounding-sphere radius |
| +0x20 / +0x30 | bbox max / min (world space); +0x2C f32 margin 0.005; +0x3C u32 ref count |
| +0x40 / +0x50 | bbox centre / centre of gravity |

**Composite (6):** +0x70 child pointer array, +0x78 and +0x80 child
transforms (4 × 4 f32, w row stored 0), +0x88 child flags (16 B each),
+0x98 u16 child count (twice). Samples have 1–5 children with identity
transforms.

**Geometry (5):** +0x88 polygons, +0x90 vec3 quantum, +0xA0 vec3 centre,
+0xB0 vertices (s16 × 3; **position = s16 × quantum + centre**), +0xD0 u32
vertex count, +0xD4 u32 polygon count, +0xF0 materials (8 B each), +0x118
per-polygon material index (u8), +0x120 u8 material count, +0x130 / +0x138
further tables (BVH nodes; not needed for the mesh).

**Polygons** (16 B, type in byte 3):

| Type | Layout |
|---|---|
| 0 triangle | u16 v0, v1, v2 at +4 (top bit is a flag), i16 neighbours at +10 |
| 1 sphere | u16 vertex at +0, f32 radius at +4 |
| 2 capsule | u16 v0 at +0, f32 radius at +4, u16 v1 at +8 |
| 3 box | four u16 corners at +4, alternate corners; the other four mirror through the centre |
| 4 cylinder | as capsule |

**Material record** (8 B): **u16 surface id** at +0 (always < 0x8000), flag
bytes after it (unknown).

## 13.2 Surface ids (unresolved)

The observed surface IDs fit in 15 bits. Their relationship to the material
names in the partial XML from `common.rpf` remains unknown.

A search over 403 recovered names and 136 distinct IDs from 43,240 collision
records compared 4,593 candidate mappings. These included plain indices,
folded hashes, seeds, prefixes and compound names. The best candidate hit
8 of 136 IDs. The control averaged 1.7 hits with a standard deviation of 1.2.
Inverting a single seeded JOAAT model across every name/ID pair yielded
54,808 distinct candidate seeds without a shared seed. These results bound
the tested mappings and the recovered name list.

## 13.3 Inferred surface categories

Instead of names, each id gets an **inferred category** and confidence from
evidence over the whole map:

| Cue | What it measures |
|---|---|
| geometry | up-facing / vertical share, mean height, height vs water line |
| road overlap | share of up-facing area on navmesh road polygons |
| path-link density | road-graph links over the id, relative to its area share |
| plants on it | grass-batch instances standing on it, by plant family |
| props on it | reflectors, bollards, barriers, fences, rocks, trees on it |
| railway | surfaces under named railway entities (gravel / ballast ids) |

Categories: road surface, dirt track / unpaved road, hard surface around
roads, paved ground, grass / vegetated soil, forest floor, wetland, rocky
ground, walls, shore, seabed, gravel / ballast, and a generic default
surface (one id covers ~39 % of all area). **98.3 % of collision area is
categorised (22.7 % high, 68.1 % medium, 7.4 % low confidence), 243 ids**.
These are categories, not material names.

## Open questions

- The surface hash function and the full material list.
- Material record flag bytes; BVH node tables.
- A railway-only surface (none found).


---

← [12. Material classes and texture slots](12-materials.md) · [Index](README.md) · [14. Navigation meshes](14-navmesh.md) →
