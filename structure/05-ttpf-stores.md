# 5. Texture stores — `RDS5` / `ETPF`

`data/ttdb/**/*.ttpf` holds 57 stores. Each is a virtual byte space, potentially larger than
4 GiB, split into independently compressed chunks, plus a texture index.

## Header

| Off | Meaning | Confidence |
|---|---|---|
| 0x00 | `RDS5` magic | OBSERVED |
| 0x04 | u32 `n` — chunk-table entries (last entry is a zero terminator) | OBSERVED |
| 0x08 | u32 virtual size, low 32 bits | OBSERVED |
| 0x0C | u32 `0x16000000` (`0x16000001` on the 18 largest stores) | OBSERVED |
| 0x10 | u32, then a zero u32 | UNKNOWN |
| 0x18 | chunk table, 8 bytes per entry | OBSERVED |
| `align_up(0x10 + 8n, 0x100)` | `ETPF` texture index | OBSERVED |

## Chunk table

Entry is a pair `(u32 a, u32 b)`:

| Field | Formula |
|---|---|
| file offset | `(a & 0xFFFFFF) << 8` |
| virtual offset | `0x1000 + (b & 0xFFFFF) × 0x4000 + (a >> 30) × 0x1000` |
| codec id | bits 24..29 of `a` |
| trailing padding | `b >> 20` bytes after the chunk's data ([formats/05](../formats/05-ttpf.md)) |

A chunk runs to the next entry in both address spaces. Omitting the 4 KiB sub-offset
(`(a >> 30) × 0x1000`) places a quarter of chunks 4–12 KiB early, which presents as a scrambled
swizzle rather than as a wrong offset — a useful diagnostic.

### Codec ids

| Id | Content | Encoding |
|---|---|---|
| 0x20 | any | zlib (`78 DA` verified on extraction) |
| 0x04, 0x0C, 0x10, 0x14, 0x18, 0x1C | BCn surfaces | BCPack (hardware block compression) |
| 0x00 | any | stored as is |

Observed distribution in one 228 MB store: 0x20 ×4,334, 0x04 ×85, 0x1C ×57, 0x14 ×44,
0x18 ×4, 0x0C ×3. Non-zlib chunks measure ~7.95 bits/byte and do **not** match raw BCn block
sizes, which is consistent with a compressed block format rather than stored blocks.

## Texture index (`ETPF`)

Structure: magic `ETPF`, a constant `0x27A`, a count, then `count × 16`-byte records, then a
trailer. Section type bytes decide the encoding of each chunk. The type byte is the top byte of
`a`: bits 24..29 are the codec id and bits 30..31 the 4 KiB sub-offset, so `0x20`, `0x60`, `0xA0`
and `0xE0` are all codec `0x20` (zlib). The other ids are BCPack (see Codec ids above), plus `0x00`
for 5 chunks stored as is; none are raw pages.

The descriptor's second u32 holds the format in bits 3..9: `((u1 >> 3) & 0x7F)` is the DXGI format
enum. This held for 357,115 of 357,115 descriptors, with no unknown values.

Bits 10..19 of the same u32 are the array size − 1: 6 on 567 cube maps, 2–64 on 130 texture
arrays, 0 elsewhere. Arrays and cube maps have their own data layouts; see
[formats/05, Arrays and cube maps](../formats/05-ttpf.md#arrays-and-cube-maps).

Each store's index holds 64-bit texture keys. Read each entry as two
little-endian u32 words `a` and `b` and assemble `a | (b << 32)`.
Of 118 keys read from material texture references in two `data.rpf`
drawables, 107 matched the index in `ttdb_vfx.ttpf`. Reversing the word
order gave zero matches.

The derivation of the keys remains unknown. A search over 104,005 candidate
names ran about 24.4 million evaluations against 352,461 index entries
across 46 stores. Nine 64-bit hash families and 100 two-hash assemblies
gave zero full-key matches. The search recovered all five planted controls.
`0x5A82F9AE` is the 32-bit JOAAT of `hauler`. It appeared in neither
word of any key in those stores.

## Volumes

Across the 57 stores: 1,026,833 sections, 147,252,026,112 bytes of payload (137.1 GiB decoded),
zero unaccounted bytes in every file. Total stored size 57.94 GiB.

## Open questions

- [BCPack](../formats/06-bcpack.md#60-status) decodes for BC1, not the other
  observed BC formats; roughly 10% of archive bytes are BCPack-encoded.
- How the 64-bit texture keys are derived.
