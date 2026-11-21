# 5. `ttpf` texture stores

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-20; confidence: confirmed / inferred / unknown.*

`data/ttdb/**/*.ttpf`: 57 stores (maps, terrain, props, peds, vehicles, UI,
weapons…), magic `RDS5`. Each store is a virtual byte space (can exceed
4 GiB) split into independently compressed chunks, plus an index keyed by
the **64-bit texture hash that drawable shaders reference** ([§7.6](07-models.md)). Most
map and vehicle textures live here, not in texture dictionaries.

## Header

| Off | Meaning | Confidence |
|-----|---------|-----------|
| 0x00 | `RDS5` | confirmed |
| 0x04 | u32 n = chunk-table entries (last is a zero terminator) | confirmed |
| 0x08 | u32 virtual size (low 32 bits only) | confirmed |
| 0x0C | u32 0x16000000, or 0x16000001 on 18 stores (mostly the largest); meaning unknown | confirmed values |
| 0x10 | u32 ?, u32 0 | unknown |
| 0x18 | chunk table, 8 bytes per entry | confirmed |
| `align_up(0x10 + 8n, 0x100)` | `ETPF` texture index | confirmed |

## Chunk entry (u32 a, u32 b)

| Field | Formula | Confidence |
|-------|---------|-----------|
| file offset | `(a & 0xFFFFFF) << 8` | confirmed |
| virtual offset | `0x1000 + (b & 0xFFFFF) × 0x4000 + (a >> 30) × 0x1000` | confirmed (2,810 chunks contiguous) |
| codec id | bits 24–29 of `a` (table below) | confirmed |
| `b >> 20` | trailing zero padding of the chunk, in bytes: stored length = next file offset − this file offset − (`b >> 20`) | confirmed (541/541 sampled zlib chunks end exactly at the RFC 1950 end-of-stream; 535/535 sampled BCPack chunks) |

Use the indexed length, not a length found by stripping trailing zeros: some
zlib streams end in a real `0x00` byte (2 of 541 sampled), and stripping cuts
them short.

A chunk runs to the next entry in both spaces. Missing the 4 KiB
sub-offset in `a >> 30` puts a quarter of chunks 4–12 KiB early, which
looks like a scrambled swizzle.

| Codec id | Chunks | Content | Codec |
|---|---|---|---|
| 0x20 | 908,643 | any | zlib |
| 0x04 | 100,026 | BC1 | BCPack |
| 0x0C | 35 | BC3 | BCPack |
| 0x10 | 77 | BC4 | BCPack |
| 0x14 | 325 | BC5 | BCPack |
| 0x18 | 268 | BC6H | BCPack |
| 0x1C | 5,120 | BC7 | BCPack |
| 0x00 | 5 | any | stored (read as is) |

BCPack ids are the BC number × 4 (`0x04` BC1 … `0x1C` BC7); zlib is `0x20`.
Confirmed by joining every chunk to the texture whose data it holds and
reading that texture's descriptor format: `0x0C` chunks belong only to DXGI
77/78 (BC3) textures (35/35), `0x10` only to DXGI 80 (BC4) (77/77), `0x04` to
BC1, `0x14` to BC5, `0x18` to BC6H, `0x1C` to BC7.

The non-zlib ids are compressed, not raw pages: their payloads measure
about 7.95 bits/byte and do not match raw BCn block sizes (confirmed).

## `ETPF` index

+4 u32 version (0x27A), +8 u32 texture count t, +0xC u32 table length, then
t × 16-byte descriptors, then t × u64 texture hashes **sorted** (descriptor
i belongs to hash i), then more tables (a u32 list, t × 32-byte digests:
unknown).

| Descriptor field | Meaning | Confidence |
|---|---|---|
| u32 d0 bits 0–13 / 14–27 / 28–31 | width − 1 / height − 1 / mip count − 1 (on cube maps, tile modes 9–12, the field is the mip count itself: cubes marked "2" store one mip per face) | confirmed |
| u32 d1 bits 0–2 | flags | unknown |
| u32 d1 bits 3–9 | DXGI format | confirmed |
| u32 d1 bits 10–19 | array size − 1 (0 on 356,418 of 357,115 textures). "Bit 10" alone is the low bit of this field: it is set on every array with an even size (598 textures). Sizes: 6 = cube maps (567); 2 (19), 3 (99), 8 (3), 16 (1), 32 (4), 48 (2), 64 (2) = texture arrays. Layouts in [Arrays and cube maps](#arrays-and-cube-maps) | confirmed (the sizes match the store spacing; every slice decodes to a separate, coherent image) |
| u32 d1 bits 20–23 | ? | unknown |
| u32 d1 bits 24–27 | tile mode | confirmed |
| u64 loc bits 0–20 | first 4 KiB page, counted after the 0x1000 header page (virtual offset = page × 0x1000 + 0x1000) | confirmed |
| u64 loc higher bits | ? | unknown |

Textures are packed tightly and small ones can sit inside a larger one's
padding, so **the size comes from the mip chain**, not the distance to the
next texture.

## Layout: AMD Gfx10 (RDNA2) swizzle modes (confirmed)

| Tile mode | Gfx10 mode | Block | Element bit order in a block (low → high) |
|---|---|---|---|
| 0 | `SW_64KB_D` | 64 KiB | 16-byte: `x0 y0 x1 y1 y2 x2 y3 x3 y4 x4 y5 x5` · 8-byte: `x0 y0 x1 x2 y1 y2 x3 y3 x4 y4 x5 y5 x6` |
| 1, 2, 3 | `SW_4KB_S` | 4 KiB | 16-byte: `y0 y1 x0 x1 y2 x2 y3 x3` · 8-byte: `x0 y0 y1 x1 x2 y2 x3 y3 x4` |
| 9, 10, 11, 12 | `SW_4KB_S`, cube maps only: tile mode − 8 = number of stored copies (below) | 4 KiB | as modes 1–3 |

- Blocks are row-major; no pipe/bank XOR.
- Independent check of tile mode 0: decoded with these `SW_64KB_D` orders,
  textures are smoother than with the `SW_64KB_S` equations on 161/200 BC1/BC4
  and 191/199 BC3/BC5/BC7 textures (median block-edge error 5.2 vs 11.4 and
  2.9 vs 7.9).
- **Mips go smallest first; mip 0 is last.**
- **Mip tail:** a mip whose element size is ≤ (block width / 2, block
  height) and that is among the last `max_mips_in_tail` mips (8 for 4 KiB
  blocks, 12 for 64 KiB) is in the tail. All tail mips share one block at
  offset 0; the other mips follow, each padded to whole blocks.
- Tail mip k (0 = largest in the tail): `m = max_mips_in_tail − 1 − k`,
  `off = m > 6 ? 16 << m : m << 8`. Origin x = bits 9, 11, 13… of `off`
  × 8 (8-byte elements) or × 4 (16-byte); y = bits 8, 10, 12… × 4. Apply
  the block equation to (origin + x, y).

Checked per store and format: mip n + 1 equals mip n downsampled (mean
error 0–4 levels) on BC1, BC3, BC4, BC5 and BC7 in every mode.

An independent 352,461-record census of the 46 base TTDB stores found
298,354 supported full chains and 9,685 partial chains with a zlib base
mip. Of the remaining records, 40,105 had an opaque base payload, 2,857
had an unresolved uncompressed layout, 948 used unsupported codecs in
that decoder, 374 had unconfirmed layout nibbles, and 138 had no matching
surface span. These counts describe that decoder's coverage. RGBA8 and
BGRA8 test images remain provisional where no reference or discriminating
mip check establishes the layout.

Some 4 KiB RGBA8 entries have the shape of packed colour tables. For
example, `ttdb_graphics.ttpf` #342 (128 × 128) has 8,333 consecutive
texels with alpha 239, followed by 8,051 zero-alpha texels after
`SW_4K_S` decoding. A row-major read scatters that padding. Records
#339, #341 and #345–347 show the same contiguous alpha pattern with
different lengths. This supports the tiled ordering on these samples.
Their use by the game remains unknown.

## Arrays and cube maps

The array size is d1 bits 10–19 + 1. Arrays and cube maps are stored in two
different ways (both confirmed).

**Texture arrays (tile modes 0 and 1; 2–64 slices; 130 textures)** are stored
level by level, smallest first, with each level's slices back to back:
the mip-tail block of every slice, then the smallest non-tail mip of every
slice, and so on, ending with mip 0 of every slice. With the single-slice
layout above:

```
address(mip m, slice s) = offset(m) × slices + s × size(m)
```

where `offset(m)` and `size(m)` are the single-slice offset and padded size
(the whole tail block for tail mips). The total is slices × the single-slice
size, which matches the store spacing exactly (e.g. 256×256 BC5, 9 mips,
32 slices = 4 MiB). Decoding slice 0 with the single-slice layout reads
other slices' tail blocks where mip 0 should be, which is the mosaic. Seen:
cloth and detail-map sets (48 slices), pattern sets, and 2–3-slice ped face
wrinkle normal maps.

**Cube maps (tile modes 9–12; 6 faces; 567 textures)** use `SW_4KB_S` and
store **tile mode − 8 complete copies of the cube**, smallest first. Split
the mip chain into *units*, smallest first: the mip-tail block (if any),
then each non-tail mip. With u units and c = tile mode − 8 copies, copy k
(0 … c − 1) holds the smallest u − c + 1 + k units. Inside a copy the faces
follow each other, each face with its own chain (tail block, then the
non-tail mips up to that copy's top mip, as in the single-slice layout). The
last copy is the full-resolution cube.

- Each copy is one chunk of the store. A copy larger than 128 KiB is padded
  to a multiple of 64 KiB; smaller copies are not padded. So the full copy
  starts at the sum of the padded sizes of the copies before it, which is
  also the virtual offset of the chunk that starts c − 1 chunks after the
  texture. The two agree on all 533 cubes with a supported format.
- Example, 512×512 BC1, 7 mips (128×128 elements; tail + 32×32 + 64×64 +
  128×128 elements = 1 + 2 + 8 + 32 pages per face), tile mode 12: copies
  of 6, 18, 66 (padded to 80) and 258 pages; the full copy starts at page
  104.
- Example, 128×128 BC7, tile mode 11: copies of 6, 12 and 36 pages (padded
  to 48); the full copy starts at page 18.
- Tile mode 9 (one copy) is a plain face-by-face cube: e.g. 16×16 and 32×32
  cubes, whose whole chain is one tail block per face.

Seen: light cookies (`lighting.ttpf`), sky/star cubes and interior room
cubes (shop walls, ceiling and floor).

## Coverage (all 57 stores, 352,461 textures)

| | Textures |
|---|---|
| whole mip chain in zlib chunks | 298,332 (84.6 %): vehicles 96.7 %, maps 91.8 %, props/peds/interiors ~90 %, terrain 66 %, UI 57 % |
| chain touches BCPack, mip 0 still in zlib | 6,966 (2.0 %) |
| mip 0 in BCPack, a lower mip in zlib (exported at that mip) | 14,441 (4.1 %) |
| every mip in BCPack | 29,310 (8.3 %) |
| unsupported 4-/2-byte formats or special tile modes 9–12 | 3,412 (1.0 %) |

The coverage table predates the array and cube-map layouts. With them, 563 of
the 567 cube maps and 118 of the 130 arrays decode (the rest have 4-/2-byte
formats, except three cubes with more copies than their mip chain has units).

**Lower-mip recovery:** chunks are 4 KiB–4 MiB ranges, not whole textures,
and mips go smallest first, so the small mips of a BCPack texture often sit
in zlib chunks. SixNG exports the largest mip that lies entirely outside
BCPack chunks (tail mips count as the whole tail block).

**Who is hit:** BCPack hits linear `BC1_UNORM` almost only (normal, packed
and mask data), not colour. Over all materials in `levels.rpf`, 37.9 % have
a flat slot and 22.6 % a low-mip slot, but weighted by placements that is
5.0 % and 3.5 %; only 7,088 materials have a flat colour slot. Terrain is
worst: its layer normals and its RGB layer-blend mask are often flat.

## Open questions

- 4-byte (RGBA8/BGRA8) and 2-byte (R16) layouts: no equation found, and
  even row-major reads are scrambled, so the data location may also be
  wrong (another header?).
- Format 157 (one vehicle texture slot) is unknown.
- Three cube maps declare more copies (tile mode − 8) than their mip chain
  has units (256×256 with "3" or "4" mips, 128×128 with "3"); how they are
  stored is unknown.
- The other descriptor flag bits, the digest tables.
- The 64-bit texture key function. The [key layout and search results](../structure/05-ttpf-stores.md)
  describe which interpretations were tested.


---

← [4. Xbox Series texture tiling (dictionaries)](04-xbox-tiling.md) · [Index](README.md) · [6. BCPack](06-bcpack.md) →
