# 15. Lights

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-20; confidence: confirmed / inferred / unknown.*

Lights hang off **archetypes** (`82d6fc83` +0x90 extension array) and some
**entities** (`ce501483` +0x28) as **light extensions** `27922c43` (80 B,
GTA V `CExtensionDefLightEffect`): +0x08 u32 name, +0x10 vec3 offset
(always 0 here), +0x20 and +0x30 arrays of **lights** `f54b227b` (672 B,
GTA V `CLightAttrDef`). Archetype files have root `d98bb561` (`CMapTypes`).
All three struct hashes are the JOAAT of these GTA V names ([§8](08-meta.md)).

Field names are the case-sensitive joaat of GTA V's `CLightAttr` member
names, matched against the struct's own member hashes (confirmed unless
noted).

| Off / hash | Field |
|---|---|
| +0x120 | position vec3, local to the archetype (Z-up, m) |
| +0x1A0 / +0x1AC | direction / tangent (unit vectors) |
| +0x288 | `colour` u8[3] RGB |
| +0x282 | `volOuterColour` u8[3] |
| +0x144 (`8609d52c`) | `timeFlags`: 24-bit hour mask, bit h = on at hour h (`ffffff` always; `e0007f` = night only) |
| `1cbbbe30` | `lightType`: 1 point, 2 spot, 4 capsule, 32 rare |
| +0x138 (`efcd993d`) | `intensity` |
| +0x148 (`cc6806e8`) | `falloff` = range in metres |
| +0x14C (`2bb342ba`) | `falloffExponent` |
| `455c3938` / `f8d9e344` | `coneInnerAngle` / `coneOuterAngle`, degrees |
| +0x154 / +0x158 / +0x15C / +0x16C | `volIntensity`, `volSizeScale`, `volOuterIntensity`, `volOuterExponent` |
| +0x168 / +0x174 / +0x178 / +0x170 | `coronaSize`, `coronaIntensity`, `coronaZBias`, `shadowNearClip` |
| +0x1DC | `extents` vec3 (capsules) |
| +0x25C | `boneId` |
| +0x28B / +0x28D / +0x28E / +0x28F / +0x290 | u8 `flashiness`, `lightFadeDistance`, `shadowFadeDistance`, `specularFadeDistance`, `volumetricFadeDistance` |
| +0x291 / +0x292 / +0x293 / +0x295 | u8 `lightType`, `groupId`, `shadowBlur`, `lightHash` |
| +0x248 (`575e428d`) | unnamed float (14 / 20 / 30 typical) — unknown |
| `b89daac1`, `ca684e56` | 90 for nearly all — unknown |

**World position** = entity position + rotate(entity world rotation, local
× (scale_xy, scale_xy, scale_z)); for interior entities, then apply the
interior instance transform. 774,847 placed lights: 447,756 exterior,
326,183 inside interiors, 907 in unplaced prefab groups (group space).

## Open questions

- `575e428d`, `b89daac1`, `ca684e56`; the two u64 hashes at +0x10 / +0x18.


---

← [14. Navigation meshes](14-navmesh.md) · [Index](README.md) · [16. Road graph, junctions and flags](16-roads.md) →
