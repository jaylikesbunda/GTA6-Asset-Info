# 17. Other formats

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-20; confidence: confirmed / inferred / unknown.*

| Format | State |
|---|---|
| PSO (`PSIN`; sections LSCH/LENM/LSNS/LGID/LRSC/CHKS) | **mixed endianness** (corrected 2026-11-20): section headers are big-endian, and a section's length includes its 8-byte header ([structure/04](../structure/04-pso-objectgraph.md)); the block table inside is little-endian (u32 hash, u32 length, `0xAA` padding), and so are the 64-bit `(offset << 20) \| block` references; hashed field names (64-bit in the vehicle tables); exported raw |
| RBF (`RBF0`) | a new variant with u16 indices and inline names; exported raw |
| `core_bvh4.rpf` | BVH4 collision with packed 24-bit indices; not decoded |
| XML, text | plain; exported as is |
| shader cache (`shaders.rpf`) | one compiled cache, no names; layout notes in [structure/06](../structure/06-models-shaders.md) |
| audio (AWC/ADAT) | container layout, chunk tags and Opus framing in [structure/03](../structure/03-audio-awc.md); not exported by the tool |
| animation (`anim.rpf`) | clip dictionaries and cutscene metadata; clip metadata decodes (confirmed); skeleton not in the clip dictionaries. See [Animation](#animation) |
| music catalogue (`audio:/config/music.dat`) | title, artist and playlist records; title hash solved (confirmed). See [Music catalogue](#music-catalogue) |
| PSIN scene placements (`story000_script.rpf`) | 97,504 64-byte placement records (confirmed). See [PSIN scene placements](#psin-scene-placements) |
| movies (`movies.rpf`) | Bink 2: 515 movies, 505 `KB2n` and 10 `KB2j`; header, frame index and Bink Audio tracks parse (confirmed); video needs a Bink 2 decoder |
| text (GXT) | `0DRP` type `0x7a` key/text resources; 54,118 keys (confirmed). See [Text (GXT)](#text-gxt) |

## Animation

`anim.rpf` holds two families. Bodies in the low region of the archive are
`0DRP` v122 cutscene/scenario metadata; above them, bodies whose
`+0x10..+0x3F` is all zero are clip dictionaries (inferred from a sample of
seven windows).

Clip names, durations and sequence channels decode (confirmed). Sampled rotation
tracks validate as unit quaternions: 3.95 M values, worst |q| − 1 about 1.2e-7
(confirmed). The skeleton is not stored in the clip dictionaries.

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
