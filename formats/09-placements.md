# 9. Placements, LOD links, archetypes and batches

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

## 9.1 Map file and entity

**Map file** root struct `d3593fa6` (0x460 B): name hash +0x08, parent
hash +0x68, flags +0x80, four vec3 extents +0x90…+0xC0, `entities` array
+0xD0 (GTA V `CMapData`).

**Entity** struct `ce501483` (0x90 B, GTA V `CEntityDef`):

| Off | Field | Confidence |
|-----|-------|-----------|
| 0x08 | u64 per-archetype hash (function unknown) | confirmed field, unknown hash |
| 0x10 | rotation quaternion x, y, z, w: **the inverse** of the entity rotation | confirmed |
| 0x20 | u64 GUID | inferred |
| 0x28 | `extensions` array (lights, particles …) | confirmed |
| 0x38 | position vec3, world, Z-up, metres | confirmed |
| 0x44 | u32 **archetype = joaat(model name)** | confirmed |
| 0x48 | u32 flags | confirmed field |
| 0x54 / 0x58 | f32 scale XY / scale Z | confirmed |
| 0x4C, 0x5C–0x82 | other fields, kept by hash | unknown |

## 9.2 LOD hierarchy (confirmed)

| Field hash | Meaning | Values |
|---|---|---|
| `6c8f1715` | lodLevel | 0 HD, 1 LOD, 2/3/4 SLOD1–3, 5 ORPHAN_HD, 6 SLOD4 |
| `d89231bd` | parentIndex | LOD slot in the parent map file ([parents](#resolving-lod-parents)) |
| `e74c3c23` | parentInstance | −1, or an instance of a prop batch |
| `a687ac89` | numChildren | 0 on HD/orphans; 1–16 on LODs |
| `bf74dfa7` | lodDist | −1 = use the archetype's `lodDist` |
| `ca974bcd` | childLodDist | 0 on HD |
| `84799847` | priorityLevel | 0–3; −1 on orphans |
| `0dc5f270`, `540ca436` | AO multipliers | usually 255 (inferred) |

The parent index points into the entities of the map file whose name hash
equals this file's parent hash. Checked on 400 child files: every link goes
exactly one level up (HD → LOD, LOD → SLOD1), none is out of range, the
median child–parent distance is 0.47 m, 80 % of named pairs share a name
prefix (`x` → `x_lod` → `x_slod`). Of 2,089 files with a parent hash, 1,304
resolve.

### Archetype LOD distances (confirmed)

The archetype definition (`82d6fc83`, [§9.4](#94-archetype-definitions-and-unnamed-drawables))
carries the same names: `lodDist` +0x38, `bsCentre` +0x58, `bsRadius`
+0x64, `hdTextureDist` +0x70. 1,060,711 of the 1,158,518 HD entities have
`lodDist` −1. Where an entity sets it, it equals the archetype value for
52 % of HD and almost all LOD/SLOD entities, and overrides it for the rest.
1,492 of 278,174 archetype hashes have definitions with different `lodDist`
values.

### Resolving LOD parents

A map file's **LOD slots** are its entities followed by its prop batches
(`ba049770`). There are two parent forms.

**parentInstance = −1**: parentIndex is an entity of the parent map file.
Only 2,089 files name that file with a parent hash (the parent's name hash;
the 64-bit `e5cdb664` equals the parent's `8bfb6095`). For the others, the
parent file can be chosen per child file, since all children of a file
share it (inferred rule):

1. Candidates are files in the same region (map-data field `6c0f9c8c`,
   shared by a child file and its LOD files) whose entity at each child's
   index is one level up and has children. SLOD1 → SLOD2 links leave the
   region, so there any file near the children is a candidate.
2. Rank by the number of children inside the parent's bounding sphere,
   then by children per parent not above `numChildren`, then the nearest
   following file in the archive.

On the files that do have a parent hash, this picks the same file in 95 %
of cases; the misses are state variants of one LOD at the same place
(identical candidates).

**parentInstance ≥ 0** (about 1.05 M HD entities): the parent is instance
`parentInstance` of prop batch `parentIndex − <entity count>` in another
map file, a `<name>_batchlod` model (confirmed). No decoded field names
that file, but the instance sits exactly on the entity (median 0.0 m), so it
is found by position. About 3 % of prop batches do not decode (garbage
instance positions); for those, match by slot and by the name `<entity
name>_batchlod`, preferring the same region. Where both methods work they
pick the same batch in 99.9 % (one same-region candidate) and 99.0 %
(several) of cases.

Results on `levels.rpf`: batch parents 1,042,067 of 1,052,215 (99.0 %);
entity parents 214,517 of 214,705 (177,809 high, 30,444 medium and 6,264
low confidence).

## 9.3 Batches (instanced grass and props, confirmed)

Subclasses of the entity struct, listed in the map data's `f14a15be`
(+0x140: arrays at +0x08 / +0x18). The entity prefix gives the archetype
and a batch centre; instance arrays sit in the graphics page.

| Struct | GTA V equivalent | Fields |
|---|---|---|
| `ba049770` prop batch (368 B) | `PropInstanceList` | +0x60 LOD distance, +0x80 u16 array, **+0x90 instances `ea6b3a96` (64 B)**: 3 × vec4 rows of a 3×4 matrix (rotation × scale in xyz, world translation in w), +0x30 f32, **+0x34 tint RGBA**, +0x38 f32, +0x3C u16, +0x3E u8 × 2; +0xA0 bbox |
| `7c475b5d` grass batch (416 B) | `GrassInstanceList` | +0x80 bbox, +0xA0 vec3 scale range, +0xB8 LOD distance, **+0x178 instances `ed86f522` (16 B)**: u16 × 3 position quantised in the bbox, u8 normal x / y, u8 × 3 colour, u8 scale, u8 AO, u8, u16 |

906,922 batches: 24,880,278 prop and 219,484,128 grass instances; 99.76 %
of batches have a model name.

## 9.4 Archetype definitions and unnamed drawables

Archetype struct `82d6fc83` (184 B, GTA V `CBaseArchetypeDef`):

| Off | Field | Confidence |
|-----|-------|-----------|
| +0x40 / +0x4C | bbox min / max | confirmed |
| +0x74 | name (joaat) | confirmed |
| +0x78 | texture dictionary | confirmed |
| +0x7C | drawable dictionary (0 where checked) | inferred |
| +0x80 | physics dictionary | confirmed |
| +0x88 | asset name | confirmed |
| +0x90 | extension pointer array | confirmed |

92,925 archetypes have no named drawable because the name string was not
recovered. The drawable's bounds equal the archetype bbox, so they can be
linked by bbox (within 0.03 m). **The archive is stored in name order**,
which breaks ties (e.g. between 64 m ground tiles that share a bbox).

Two more rules (2026-11-20):

- **Embedded drawable name** (tried first, confirmed): about 8.8 K drawables
  that the archive scan left unnamed still carry their own name in the
  resource's name block. When joaat(name) is an unresolved archetype and
  only one drawable carries it, that is the link. These 8,616 links agree
  with all 14,777 earlier links they overlap. Their bounds often differ
  from the archetype bbox, which is why the bbox rule missed them.
- **Name-order gap** (fallback, inferred): an archetype whose name is
  guessed (joaat of sibling-name stems plus common suffixes) has its
  drawable between the named entries just before and after that name.
  With exactly one candidate in the gap it is linked; with 2–6, the one
  whose bounds clearly fit best within 0.5 m (3,645 links).

Result: **15,412** of 92,925 linked (8,616 by embedded name, 3,645 by name
order, 2,565 by unique bbox, 586 by bbox plus locality); 25,277 bbox
matches remain ambiguous.

An independent sweep of `levels.rpf` found 104 direct name-hash links
against a sample of 393 distinct drawable model hashes. One reproducible
example is `52ed145f`, the JOAAT of
`am_e_light_airport_rotatingbcn_01a_lod`. Read archetype `+0x74` from the
object at decompressed offset `0x60` in the resource at archive offset
`0x1e206d010`. Drawable model 0 at archive offset `0x1d9d8b010` stores the
same hash and the literal name. Its bounding-box components differ from
the archetype by at most 0.000237 m.

In that sample, a bbox-only search at 0.03 m tolerance gave 1,044 unique
candidates. Across 200 permutations of the drawable hash labels with their
bbox pool held fixed, 965–978 candidates were unique. Treat unique bounds
as a heuristic and use name-hash equality where both sides provide it.

`35abc248` (32 B, one per file, 6,304 files) is a named entity group
(prefab) in group space; no map data places these groups.

## Open questions

- The 64-bit per-archetype hash at entity +0x08.
- Where prefab groups (`35abc248`) are placed.
- 785 map files whose parent hash does not resolve.


---

← [8. Meta resources (type 0x7a)](08-meta.md) · [Index](README.md) · [10. Interiors (MLO)](10-interiors.md) →
