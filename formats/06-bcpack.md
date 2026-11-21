# 6. BCPack

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-20; confidence: confirmed / inferred / unknown.*

BCPack is the texture codec of the Xbox Series hardware decompression
unit. There is no public specification or decoder, and no
plain copy of any affected texture exists elsewhere in the game. What
follows was mapped from the streams alone, on BC1. **0 % of BCPack data is
decoded as of 2026-11-20.** BC4, BC5 and BC7 streams have not been studied
beyond their headers.

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
5. Block order (linear or swizzled).
6. BC4, BC5, BC6H and BC7 streams.

## 6.4 How to resume

Any candidate decoder has a hard check: every lane must consume exactly its
byte budget and end at x = L, with the symbol count fixed (4N index
symbols for N blocks). Hits at a free symbol count are chance.

Ways in, best first:

1. **Reference blocks.** Decoded mip-0 BC blocks for a few BCPack textures
   (needs a reference decoder; none is public). With known symbols, each
   model and the read schedule can be fitted directly.
2. **A one-block-difference pair**: two textures that differ in exactly one
   known block. Few tail-sharing pairs exist (15 groups out of 3,259 BC1
   textures checked), so this is hard to find.
3. **Solve the varying-lane record layout first.** Their byte budgets would
   bound reads per round and make a search tractable.
4. **Full joint search.** Index models × lane-0 models × varying/extension
   models × round variants × start-state variants is about 3 × 10^8 full
   decodes per chunk (~6 × 10^12 symbol steps, roughly two weeks on one
   desktop), with no early pruning because every round mixes all lanes.

Until then, use the lower-mip recovery ([§5](05-ttpf.md)) and the flat fallbacks.


---

← [5. `ttpf` texture stores](05-ttpf.md) · [Index](README.md) · [7. Drawables, fragments and vehicles](07-models.md) →
