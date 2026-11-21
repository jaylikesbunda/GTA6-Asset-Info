# 6. Models and shaders

## Meshlet geometry

Drawable geometry is stored as meshlets, not as raw vertex arrays. Every raw-coordinate
interpretation fails arithmetic or range tests, so a decoder must follow the meshlet structure
rather than search for float triples.

Observed layout, worked example body of `0x40000` bytes:

- Body header, with a system-end pointer at `+0x08` and a u16 count at `+0x4C`.
- That count gives the number of `{id/hash, size, pointer}` records that follow.
- Sorted by pointer, the records tile the body with gaps only of `{0, 4, 8}` plus the record
  array itself, and the chain terminates on the header's own `+0x08` target.

Meshlet buffer:

| Field | Meaning |
|---|---|
| `u32 @ buffer+0x00`, low 16 bits | meshlet count |
| `buffer + 0x0C` | start of the per-meshlet 16-byte record array |
| `buffer + 0x0C + count × 16` | triangle base |

Per-meshlet records carry the vertex and triangle offsets into the shared buffers; triangle spans
run to the next meshlet's start.

### Drawable name

The drawable's name pointer sits at `drawable + 0xA8`. Following it yields the authored name —
`name.#dr`, `name.#td`-style suffixes included. Scanning only for a `.#dr` string in the resource
finds a minority of drawables; the pointer is the reliable route.

### Containers

`0DRP` bodies in mesh-bearing archives are page-multiple in size (`0x20000`–`0x150000`) and carry
the same descriptor at `+0x10` seen elsewhere: a `0x7A` constant at `+0x14` and `0x50xxxxxx`
relocated page pointers.

In `PSIN` files the `LRSC` section instead holds an 80-byte `0DRP` stub with no pointers, so the
geometry for those entries is not in the `PSIN` payload itself.

## Shader cache

Shaders are **not** DXBC or DXIL. They are native AMD **RDNA2 (GFX10.3)** machine code inside a
`PS\x00\x09` pipeline-state-object cache.

Observed cache layout:

| Region | Size |
|---|---|
| header | `0x80` |
| graphics PSOs | `count × 0x70` |
| compute PSOs | `count × 0x18` |
| meta index | `count × 8` |
| shader index | `count × 8` |
| meta blob | variable |
| raw code | variable |

Example counts from one cache: 25,471 graphics PSOs, 2,693 compute PSOs, 54,576 meta entries,
21,522 shaders.

Instruction-level markers: `s_endpgm` (`0xBF810000`), `s_code_end` (`0xBF9F0000`), `s_clause`,
Wave32 VOP2/VOP3 forms, `image_sample_l` with 12-byte non-sampled-address operands, NGG/meshlet
`exp prim`, and hardware ray-tracing `image_bvh64_intersect_ray`.

Binding metadata: chunk `0x41` carries user-data root-signature records (`SRV 0x300`,
`CBV 0x302`, `UAV 0x303`, `Sampler 0x002`), `0x42` register slots, `0x43` 128-bit shader hashes,
plus `COMPUTE_PGM_RSRC1/2` and the `[numthreads(x,y,z)]` triple.

Two independent counts of the same cache disagree on stage totals, so stage classification depends
on the classifier used; the instruction stream itself is unaffected.

## Open questions

- The meaning of the eight-hex suffix on compiled shader wrapper names.
  A test of 6,896 stems against nine hash families, including JOAAT, produced
  zero suffix matches. The [32-bit drawable material name fields](../formats/12-materials.md#material-name-hashes)
  use lowercase JOAAT.
- Whether meshlet vertex data uses a dictionary encoding (a small shared u16 vocabulary is
  observed across bodies, suggesting one).
- BCPack decoding for compressed surfaces.
