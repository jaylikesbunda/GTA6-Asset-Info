# 8. Meta resources (type 0x7a)

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-20; confidence: confirmed / inferred / unknown.*

About 188 K resources in `levels.rpf` (plus ~20 K in `data.rpf` and
`packs.rpf`) are `PRD0` type 0x7a: **self-describing**, each carrying its
own schema, like a PSO file packed into a resource. Map data, archetypes,
lights, interiors, paths and junctions all use it.

## Header (confirmed)

| Off | Meaning |
|-----|---------|
| 0x10 | `0DRP` + u8 0x7a |
| 0x20 | u32 number of root objects |
| 0x28 | ptr → struct table, 32 bytes per struct |
| 0x30 | ptr → enum table |
| 0x38 | ptr → instance table, 16 bytes per block |
| 0x48 | u16 struct count, u16 enum count, u16 instance-block count |
| 0x4A | u16 enum count |

## Tables (confirmed)

| Record | Layout |
|---|---|
| struct (32 B) | u32 name hash, u32 signature, u32 flags (0x300/0x400), u32 0, u64 ptr → members, u32 size, u16 0, u16 member count |
| member (16 B) | u32 field-name hash (0x100 = anonymous element of the next array), u32 offset, u8 type, u8 sub-type, u16 reference (arrays: index of the element member; bitsets: bit count), u32 extra (enum hash for enums) |
| enum (24 B) | u32 hash, u32 signature, u64 ptr → values, u32 count, u32 0; values 16 B {u64 value, u64 name hash} |
| instance block (16 B) | u32 struct hash (or a type code, making it a **pool**), u32 byte size, u64 ptr; size / struct size objects follow each other |

**Field and enum names can be resolved with JOAAT.** The exact input spelling
matters. In sampled `ce501483` entity member tables, the stored hash for
position at `+0x38` is `01166164` (hash of `position`) and the stored hash
for flags at `+0x48` is `67d26872` (hash of `flags`). The title-case
spellings `Position` and `Flags` hash to `0c928991` and `4b621302`.
GTA V field names often resolve directly.

**Struct names of the map formats resolve too** (2026-11-22): their 32-bit
struct hashes are the case-sensitive JOAAT of the GTA V type names, with
`::` written as `__` as in GTA V's meta names.

| Hash | Name | See |
|---|---|---|
| `d3593fa6` | `CMapData` | [§9](09-placements.md) |
| `d98bb561` | `CMapTypes` | [§15](15-lights.md) |
| `ce501483` | `CEntityDef` | [§9](09-placements.md) |
| `82d6fc83` | `CBaseArchetypeDef` | [§9](09-placements.md) |
| `f14a15be` | `rage__fwInstancedMapData` | [§9.3](09-placements.md#93-batches-instanced-grass-and-props-confirmed) |
| `ba049770` / `ea6b3a96` | `rage__fwPropInstanceListDef` / `rage__fwPropInstanceListDef__InstanceData` | [§9.3](09-placements.md#93-batches-instanced-grass-and-props-confirmed) |
| `7c475b5d` / `ed86f522` | `rage__fwGrassInstanceListDef` / `rage__fwGrassInstanceListDef__InstanceData` | [§9.3](09-placements.md#93-batches-instanced-grass-and-props-confirmed) |
| `10506455` | `CMloArchetypeDef` | [§10](10-interiors.md) |
| `09cc28be` | `CMloInstanceDef` | [§10](10-interiors.md) |
| `0b1811f1` / `995072ca` / `d6a799f9` | `CMloRoomDef` / `CMloPortalDef` / `CMloEntitySet` | [§10](10-interiors.md) |
| `f54b227b` | `CLightAttrDef` | [§15](15-lights.md) |
| `27922c43` | `CExtensionDefLightEffect` | [§15](15-lights.md) |

The vehicle and material structs (`e59aa857`, `51019114`, `fc1c7000`,
`f795e8ba`, `37e52277`, `a9d57751`) do not resolve against GTA V names and
may be new or renamed types. PSO struct names are a separate, 64-bit scheme
([§17](17-other.md#pso)).

**Method.** Every identifier in CodeWalker's GTA V sources (63,839 tokens)
was hashed as written and in lower case, and compared with the 189 hashes
quoted in these notes. 66 match. The expected number of chance matches is
about 0.006 (63,839 × 2 × 189 / 2³²), so a match is a name, not a
coincidence. Adding public RDR2 lists (native and enum databases,
`rdr3_discoveries`) resolved nothing further. The names above are confirmed
by hash; what a field means is still inferred from its use.

```python
def joaat(s):
    h = 0
    for c in s.encode():
        h = (h + c) & 0xffffffff
        h = (h + (h << 10)) & 0xffffffff
        h ^= h >> 6
    h = (h + (h << 3)) & 0xffffffff
    h ^= h >> 11
    return (h + (h << 15)) & 0xffffffff

assert joaat("CMloRoomDef") == 0x0b1811f1
assert joaat("rage__fwInstancedMapData") == 0xf14a15be
```

**Map root block.** In the resource at `levels.rpf` offset `0x1e0bed010`,
both `+0x4A` and `+0x4E` contain 2. The `d3593fa6` map root is block 1
(1-based). Block 2 is a `0x4f` pool. Identify the root by its struct hash
when reading resources with this layout.

**Pool reference (confirmed):** u64 with the **1-based block index in bits
0–12** and the **byte offset in bits 20+**.

## Type codes

| Code | Type | Confidence |
|---|---|---|
| 0x10 / 0x11 / 0x12 / 0x13 / 0x15 | s8 / u8 / s16 / u16 / u32 | confirmed |
| 0x4a | u32 hash | confirmed |
| 0x21 | f32 | confirmed |
| 0x32 / 0x33 | vec3 (12 B) / vec3 (16 B) | confirmed |
| 0x34 | vec4 / quaternion | confirmed |
| 0x08 | u64 | confirmed |
| 0x4f | u64 (sub 5) / 128-bit (sub 0x0f) | inferred |
| 0x60 | u8 enum | confirmed |
| 0x65 | bitset (reference = bit count, extra = enum hash) | confirmed |
| 0x52 | array {u64 ref, u16 count, u16 capacity} | confirmed |
| 0x5c | array {u32 count, u32 capacity, u64 ref} | confirmed |
| 0x44 | string: inline chars or u64 ref into a char pool | confirmed |
| 0x07 | pointer (polymorphic when extra = 0: takes the struct of the block it points into) | confirmed |
| 0x05 | embedded struct | confirmed |
| 0x71 | map (24 B): u64 1, u64 pool reference, u16 count, u16 capacity. The reference points at `count` consecutive entry structs (`00000100`–`00000106`), each {`6098a50e` key (`Key`), `063fa3f2` value (`Item`)}; the entry struct sets the value type (`00000101` = u32 key → u32 array). Empty maps point at block 0. PSO uses code 0x75 instead ([§17](17-other.md#pso)) | confirmed |

## Open questions

- Struct name hashes (in PSO they are 64-bit, [§17](17-other.md#pso)); header field 0x4E.
- Many fields are still known only by hash; they are exported by hash.


---

← [7. Drawables, fragments and vehicles](07-models.md) · [Index](README.md) · [9. Placements, LOD links, archetypes and batches](09-placements.md) →
