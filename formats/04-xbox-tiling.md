# 4. Xbox Series texture tiling (dictionaries)

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

Applies to textures in texture dictionaries and embedded in models. (The
bulk texture stores use AMD Gfx10 layouts instead, [§5](05-ttpf.md).)

| Item | Rule | Confidence |
|------|------|-----------|
| Tiles | 16 × 16 **elements** (a 4×4 BC block, or one pixel), row-major; row pitch padded to 16 elements | confirmed |
| In-tile bit order, 8-byte elements (BC1/BC4), LSB first | `x0 y0 y1 x1 x2 y2 x3 y3` | confirmed |
| In-tile bit order, 16-byte elements (BC2/3/5/6H/7) | `y0 y1 x0 x1 y2 x2 y3 x3` | confirmed |
| 4-byte elements (RGBA8) | a guess is used; output flagged `swizzle: "unverified"` | unknown |
| Mip order | **smallest first**; mip *m* starts at `align_up(sum of sizes of mips m+1..N−1, 4096)` | confirmed (mips 0–1 by search, rest by eye to 64 px) |
| Whole chain in one 4 KiB block | small textures with mips whose whole chain fits one `SW_4KB_S` mip-tail block (e.g. 32 × 32 BC3 HUD icons, 4 mips, packed 4 KiB apart) keep **mip 0 inside that block**, laid out as the Gfx10 mip tail of [§5](05-ttpf.md#layout-amd-gfx10-rdna2-swizzle-modes-confirmed). There is no separate mip-0 page after the tail; reading one gives the next texture's data (blank icons) | confirmed |
| Mips under 16 × 16 elements of larger textures | packed tail | unknown |

The bit orders were found by search (simulated annealing on image
smoothness) and checked on clouds, terrain, normals, window atlases and UI.

## Open questions

- RGBA8 / 2-byte equations; the packed tail for the smallest mips (DDS
  export stops at 32 × 32 elements, 128 px for BC formats).


---

← [3. Texture dictionaries](03-texture-dictionaries.md) · [Index](README.md) · [5. `ttpf` texture stores](05-ttpf.md) →
