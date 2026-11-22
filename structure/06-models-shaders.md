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

The shipped shader programs are native AMD RDNA2 machine code. With wave32
enabled, the LLVM `gfx1013` target decoded all reachable words in a corpus
of 2,559 unique shipped compute programs.

Reported `PS\x00\x09` pipeline-state-object cache layout (independent
archive location unconfirmed):

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

An independent scan of `shaders.rpf`, `common.rpf`, `script.rpf`,
`appdata0.rpf`, `data.rpf` and `packs.rpf` found no structurally valid
`PS\x00\x09` cache. Two signature hits in `packs.rpf` failed the
layout and instruction checks. The scan did not cover all of `levels.rpf`
or `story000.rpf`. The shipped shader records support a separate join
using XDAT fingerprints (1,259 groups with both VS and PS) and PSV0 binding
metadata. The shipped root-signature layout remains unresolved.

## Translation checks

Of 2,559 compute programs, 46 translated to HLSL with the strict translator.
Of those, 27 matched the reference interpreter on an RTX 3060 and 26 also
matched on WARP. The remaining compute program involves a cross-lane UAV
read-after-write. A separate `VS_Clear` / `PS_Clear` pair, at
`0x1b2392` and `0x5176e02` in `shaders.rpf`, matched all 4,096
reference pixels on both backends. These counts cover the tested programs.
SDWA operand handling was the first blocker for 1,239 compute programs.

## Open questions

- The meaning of the eight-hex suffix on compiled shader wrapper names.
  A test of 6,896 stems against nine hash families, including JOAAT, produced
  zero suffix matches. The [32-bit drawable material name fields](../formats/12-materials.md#material-name-hashes)
  use lowercase JOAAT.
- The meaning of the reported 64-bit hashes in the shader cache; these
  are not the confirmed 32-bit drawable material name fields.
- Whether meshlet vertex data uses a dictionary encoding (a small shared u16 vocabulary is
  observed across bodies, suggesting one).
- BCPack decoding for [formats beyond BC1](../formats/06-bcpack.md#60-status).
