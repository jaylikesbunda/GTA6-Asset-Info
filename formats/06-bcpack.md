# 6. BCPack

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

BCPack is the texture codec of the Xbox Series hardware decompression
unit. There is no public specification, and no plain copy of any affected
texture exists elsewhere in the game.

## 6.0 Status

**BC1 decodes** (as of 2026-11-21). A codec `0x04` chunk decodes to plain
BC1 blocks:

- block count = the chunk's virtual size / 8;
- the input is the stored bytes without the trailing zero pad (`b >> 20`
  in the chunk entry, [§5](05-ttpf.md#chunk-entry-u32-a-u32-b));
- the blocks come out in the texture's usual tiled order, so they are
  detiled and decoded exactly as for zlib chunks.

Confirmed: all 2,776 BCPack BC1 textures of one terrain store decode with
no errors. With BC1 decoded, mip 0 is readable for 98.4 % of store
textures (86.8 % before; [coverage](05-ttpf.md#coverage-all-57-stores-352461-textures)).

**Not decoded:** the BC3, BC4, BC5, BC6H and BC7 codecs (`0x0C`, `0x10`,
`0x14`, `0x18`, `0x1C`). About 0.8 % of textures have mip 0 in those.

The rest of this page is the earlier mapping of the BC1 stream, made from
the streams alone before BC1 decoded. It is partial and may not match a
working decoder in every detail (lane count, models); it is kept for the
other formats, which share the coder.

## 6.1 What is solved

**Identification (confirmed).** Chunk codec ids follow the BC format ([§5](05-ttpf.md))
and the size saving is ~12 %. The coder is rANS, with colour endpoints
coded apart from the indices (§6.1 below). Payloads are 7.9–8.0 bits/byte with flat bit autocorrelation and
are zero-padded to 256 bytes. No general-purpose wrapper (zlib, deflate,
LZMA, zstd, LZ4, bz2) applies.

**Lanes (confirmed for BC1).** A stream is made of **lanes**: independent
rANS streams, one per symbol class, whose bytes share one byte stream.

| Lane | Role | Confidence |
|---|---|---|
| 0 | mode `0x30`–`0x37`; carries 1.3–6 bits per block, probably per-block endpoint data | inferred |
| 1 | always mode `de`; a symbol that is the same in every block (its record depends only on the block count N). Not the 3-colour flag. | confirmed |
| 2–7 | endpoint channel planes **R0 G0 B0 R1 G1 B1** (565 colour 0, colour 1), each with its own model | confirmed |
| index lanes (4) | one per row of 2-bit indices, modes `0x33`–`0x38`; ~7.2 bits per row per block | inferred |
| extension lanes | one per plane of type `0x10`; models unknown | inferred |

**Mode byte (confirmed):** 3-bit parameter (top) + 5-bit type (bottom).
Plane types are `0x1a`–`0x1e` and `0x10`, with parameters 0–6. The top
bits fall as the plane's value range shrinks.

**Constant-lane coder (confirmed, byte for byte).**

| Parameter | Value |
|---|---|
| coder | byte-wise rANS, encoded in reverse, decoded forward |
| probability precision | M = 2^16 |
| state | 32-bit, lower bound L = 2^24, byte renormalisation |
| encoder start state | x = L (a lane decoded to its end lands exactly on L) |
| model | static: the repeated symbol at start 1 with frequency M − K; every other slot frequency 1 (one exception costs exactly 16 bits) |
| K by mode | `dc` K = 4, `dd` K = 8, `de` K = 32 (types `0x1c`/`0x1d`/`0x1e`, parameter 6) |

```
encode (per symbol, reverse order):
  while x >= ((L >> 16) << 8) * f:  emit(x & 255); x >>= 8
  x = (x // f << 16) + x % f + start
decode:
  s = x & 0xffff
  if 1 <= s < 1 + f:  x = f * (x >> 16) + s - 1     # the repeated symbol
  else:               x >>= 16                       # an exception slot
  while x < L:  x = (x << 8) | next_byte()
```

A constant plane with a few exceptions (constant in mip 1, not in mip 0)
solves exactly by re-encoding over every block and escape slot.

**Record layouts.**

| Lane kind | Record | Confidence |
|---|---|---|
| constant / fixed lanes (`dc`, `dd`, `de`, type `0x10`) | `[mode][n−1][state: 4 bytes big-endian]`, n = 4 + renorm bytes | confirmed |
| type `0x10` planes | `10 03 40 00 xx 00` on 6-bit planes, `.. 20 00 ..` on 5-bit planes (64 / 32 levels); parameters, not states. Their data is elsewhere in the stream. | inferred |
| lane 0 and index lanes | candidate seven-byte record `[mode][u16][state: 4 bytes]` | lane 0 record shape observed. The meaning of the middle word and the index-lane layout remain open. |
| varying plane lanes | `[mode][len lo][len hi]…` fits `3c` lanes, gives impossible lengths for `1c` lanes | unknown |
| extension lanes | length field not understood | unknown |

**Header layout (confirmed).** The first 48 bytes are the records of the 8
main lanes, **column-interleaved**: 6 rounds of 8 bytes, one byte per lane
per round (this is a consequence of the read-order rule below). For
planes-constant chunks a second set of records follows for the 4 index
lanes and any extension lanes, read the same way.

**Read-order rule (confirmed on the header and early bytes).** The decoder
reads in **rounds**. In each round, every lane that needs a byte reads
exactly one, in lane order: main lanes 0–7, then index lanes, then
extension lanes. A lane needs a byte while its record is incomplete, or
while its one-byte prefetch is empty and it still has byte budget (record
length) left. This explains:

- the 48-byte column-interleaved header;
- one "lookahead" byte per lane right after the header (the prefetch;
  constant lanes only take it if n − 1 > 3, length-coded lanes always) –
  checked on lane 1 in 41,988 of 42,000 chunks;
- the second header's rounds of 4 index-lane bytes, with 0–5 bytes from
  other lanes in between;
- the 8–9-byte gap between one lane's two renorm bytes at block 0.

**Other facts.**

- The index streams are encoded in reverse (rANS-like): two near-identical
  textures share only the last ~60 bytes of their streams.
- All lanes share one decode-ordered stream and end together (not
  concatenated).
- Chunk block count ≠ stream block count in 1,536- and 3,072-block chunks:
  they hold several smaller streams. For tests, use chunks whose lane 1 is
  the canonical `de` state for N.
- BC1 indices cost ~30 bits per block: BCPack saves almost nothing on
  indices; the saving comes from the endpoint planes.

### Constant lane 1 in BC1 and BC4

**Confirmed on the sampled base stores.** Let `N` be the declared chunk
output size divided by the descriptor's bytes per BC block. In BC1 and
BC4, the lane-1 constant record has the rANS state and renormalization byte
count for `S = min(N, 65536)` symbols. This check is independent of how the
other lanes are interleaved.

| Data | Exact state and byte-budget matches |
|---|---:|
| BC1 calibration store (`maps_0_7`) | 2,644 / 2,644 |
| Held-out BC1 stores | 97,376 / 97,376 |
| Held-out BC4 stores | 77 / 77 |

The counts support `ceil(N / 65536)` substreams for large chunks. All 918
multi-substream cases in the sample agree with the lane-1 cap. Of 97,904
constant states in plane lanes 2–7, 85,810 fit `S`, a tested division of
`S`, or a per-mip block count. The other 12,094 remain unexplained.

In all 5,748 sampled BC3, BC5, BC6H and BC7 chunks, lane 1 lacked the
constant record seen in BC1 and BC4. These 16-byte block formats need a
separate account of lane roles.

## 6.2 Ruled out

An independent test applied a literal little-endian byte-budget reading of
the middle word to the seven-byte lane-0 records. The modeled budget
disagreed in 2,562 of 2,565 lanes. In a separate 100,020-chunk test of the
proposed header and index-mode layout, two chunks satisfied the exact
header-plus-length partition. Both failed the proposed index-mode check.
The test did not reach a verified symbol decode. The 16-bit field needs a
separate validation before it can serve as a decoder byte budget.

Static frequency tables in the header; Huffman; byte-aligned planes;
shift-add and LZMA-style range-coder recurrences; static 4-symbol tables
for the index lanes; 81 adaptive index models (adaptive frequency tables
with rates 1–8, LZMA-style binary trees with shifts 1–8) × 5 contexts ×
every start offset, both per lane and with the 4 index lanes joint. Every
chunk that has index lanes also has lane 0 and at least one other lane with
an unknown model, so the index lanes cannot be tested alone.

## 6.3 Open unknowns

1. Models for lane 0, the index lanes (probably adaptive), the varying
   plane modes (parameter < 6) and the extension lanes.
2. Record layout (byte length) of varying and extension lanes.
3. What lane 0 codes.
4. Exact timing of decode versus read within a round.
5. Block order for formats beyond BC1.
6. Full decoding of BC4 and the lane roles of the 16-byte formats.

## 6.4 How to resume

Use the decoded BC1 blocks as a baseline. A candidate lane model must
consume exactly its byte budget and end at x = L, with a symbol count
fixed independently of the fit. For the proposed BC1 index model, that
means 4N index symbols in total for N blocks. A fit with a free count
is not validation.

1. **Start with reference blocks.** For formats beyond BC1, an exact
   decoded BC payload, or the same texture stored both as BCPack and plain
   blocks, would make read order and assembly testable. Re-encoding a
   visually similar image is not equivalent: one public encoder reproduced
   only about 42 % of blocks in a stored BC1 comparison. One-block-difference
   pairs are another option, but tail-sharing pairs were rare (15 groups
   out of 3,259 BC1 textures checked).
2. **Use the earlier failures to constrain the lane model.** In the
   independent harness, none of 2,566 all-constant BC1 chunks passed; the
   first failure was lane 0's expected symbol count. One 1,536-block
   chunk's state fits 257 symbols under the tested constant model. Do not
   treat the observed 16-bit words as byte budgets without validation;
   solve the varying and extension records and interleaved read order.
3. **Check reconstruction on held-out chunks.** Test each format's lane
   roles, endpoint/index assembly and exact output bytes, including mip
   tails, slices, tiling and multiple substreams.

The earlier BC1 joint-search estimate was about 3 × 10^8 full decodes
per chunk (~6 × 10^12 symbol steps, or roughly two weeks on one desktop).
It assumed no early pruning because each round mixes all lanes.

For BC3–BC7, use the lower-mip recovery ([§5](05-ttpf.md)) and the flat
fallbacks until then.


---

← [5. `ttpf` texture stores](05-ttpf.md) · [Index](README.md) · [7. Drawables, fragments and vehicles](07-models.md) →
