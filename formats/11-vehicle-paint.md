# 11. Vehicle paint and vehicle tables

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

| Item | Finding | Confidence |
|---|---|---|
| colour slot `c8e8c282` (`diffusetex`) on vehicle shaders (`standard_vehicle_2lyr`, `4aa38282`) | points at an engine-default **white** texture found in no store; the body colour is a runtime paint colour, as in GTA V | confirmed |
| `f151c063` | decal / sticker layer, RGBA | inferred |
| `d37026cb`, `640111c6` | dirt / grime / wear masks | inferred |
| `84d22f74` | fine scratches and flakes | inferred |
| `b9f4bdac` | grime | inferred |
| `fb03ae36` (`dusttexture`) | green-dominant mask (mask pack or normal detail) | unknown |
| `4d56d7fe` / `2bb2ede8` (`lyr1materialatex` / `lyr0materialatex`) | BC1 atlas, normal-like | inferred |
| `84c69350` / `58b6e707` (`lyr0normaltex` / `lyr1normaltex`) | BC5 normals | confirmed |
| shader constant buffer (+0x08) | ~0x160 bytes of floats, layout not keyed by the parameter table | unknown |

## Vehicle tables

The game-data meta resource in `packs.rpf` (type 0x7a, 26 MB) and about 330
per-vehicle pack resources with the same structs (rows repeated) hold the
vehicle tables. Field names are case-sensitive joaat and match GTA V's
`vehicles.meta` names where they exist (confirmed).

| Struct | Count (game-data resource) | Fields |
|---|---|---|
| `e59aa857` model info | 327 | `gameName` (`120e5836`, joaat of the model name), `handlingId` (`ec41bd9c`), `txdName`, `wheelScale` (`933301be`), `wheelScaleRear` (`76706238`), `wheelType` (`a2e90c68`), `layout`, `HDTextureDist`, `lodDistances`, `vehicleClass`, `swankness`, `plateType`, `defaultBodyHealth` … (52 names resolve) |
| `51019114` configuration | 3,879 | `fb1f80ff` model, **`5bdc13e3` body colour** (ARGB u32), `e89b8540` part models (joaat names, about 150: frame, bumpers, exhaust, `brakedisc_*`, `caliper_*`, and one `rim_*` and one `tyre_*` in 82 %) |
| `fc1c7000` variation set | 4,772 | `fb1f80ff` model, **`8c866829` weighted paint-colour list**: 1 to 400 ARGB colours with repeats (2,246 sets; this is the pool of 17,632 colours), `59a59c3e` weighted part groups: `37a6f9ef` slot, `a12ff3a2` list of {`8e6612bf` part, `dc732774` weight %} |

With the per-vehicle packs this covers 373 models (305 named), 4,063
configurations (3,339 with a rim and a tyre) and 2,377 colour lists. Handling is
a separate PSO ([§17](17-other.md#vehicle-handling)).

- **Colours** are all dark (`phoenix`: `ff091f30`, `ff122214`), so they are
  probably linear. Converted from linear to sRGB they give plausible paints
  (#356278 steel blue, #4b664f sage). Inferred; not checked in game.
- **Colour lists** (corrected 2026-11-21; earlier read as 100-colour
  palettes): colours repeat, e.g. `phoenix` has 400 entries with 10
  distinct colours, so the repeat count reads as a draw weight (inferred).
  They match the spawn-configuration body colours in only 168 of 4,024
  configurations, so they are a separate random-paint pool.
- **Wheel scale**: the shared rim models have radius 0.5 and the shared
  tyre 0.875 (width 0.585). Scaling both by **1.625 × `wheelScale`** (front
  and rear separately) fits `phoenix` (0.37 at `wheelScale` 0.2277), `mesa`
  and `ridgway` by eye. That factor is fitted, not decoded. It puts the
  `phoenix` rims at about 14.6 inch, close to a real car's 15 inch, so the
  earlier reading of `wheelScale` as the rim radius in metres (a dirt bike
  has 0.2538 / 0.2234, which would be 21 / 18 inch) is doubtful.

## Open questions

- Secondary and pearl colours, liveries; whether the colours are linear.
- The game's rule for sizing the shared rim and tyre from `wheelScale`
  (1.625 × is fitted); the tyre width.
- The shader constant buffer.


---

← [10. Interiors (MLO)](10-interiors.md) · [Index](README.md) · [12. Material classes and texture slots](12-materials.md) →
