# File schemas

*Part of the [SixNG tool guide](README.md). Numbers and timings as of 2026-11-20.*

Field lists below cover what downstream converters need. Every file also
carries `note` fields explaining itself; unknown fields are kept by hash.

## `model.json`

| Field | Meaning |
|---|---|
| `source` | `{archive, id, offset, streams}` |
| `name`, `type` | recovered name; `dr` / `ft` … |
| `glb`, `lod_glbs`, `parts_glb`, `points_glb` | file names written. `parts_glb` (`model_parts.glb`, added 2026-11-20) holds models of embedded drawables (fragment children, building sub-drawables); they are not LODs, and `meshes[]` marks them `part: true` with `lod: null` |
| `vertex_count`, `triangle_count` | totals |
| `geometry_bbox` | `[min, max]` in RAGE model space (Z-up), over the LOD0 meshes, computed from the decoded vertices (added 2026-11-20). Prefer it to `drawable.bounds` and `bounds_raw`, which are offset or rotated on some models |
| `coordinate_system` | axes and units note |
| `surface_hint` | `road`, `kerb`, `sidewalk`, `grass`, `dirt`, `vegetation` or null, guessed from the model name only |
| `meshes[]` | per geometry: `name` (`geomNN_<off>`), `geometry`, `vertex_count`, `triangle_count`, `exceeds_kn5_16bit_limit`, `material` (`shaderNN_<hash>`), `shader_index`, `uv_sets`, `uv_slots`, `normals`, `vertex_colors_raw`, `triangles_flipped_to_match_normals`, `triangles_dropped`, `position_axis_order`, `skinned`, `skin_influences`, `bones`, `bbox` (RAGE `[min, max]` from the vertices, added 2026-11-20) |
| `wheel_mounts[]` | `{bone, pivot}`: wheel centres in model space (RAGE Z-up) |
| `drawable` | parsed structure: `models[]`, `shaders[]`, `skeleton[]` (node, parent, `pivot`, `bind_matrix`), `bounds` |
| `materials[]` | per shader: `shader`, `shader_name_hash`, `shader_name` (the game's shader name, e.g. `standard_vehicle_2lyr`; added 2026-11-21), `classification`, `base_color`, `normal`, `normal_green`, `textures[]` (each slot with `param_name` where known, e.g. `diffusetex`), optional `texture_fallback`, a `paint` block on car paint, and a `multilayer` block on two-layer blend shaders |
| `buffer_files`, `texture_files` | written files |
| `head_skin` | ped heads (added 2026-11-21): the face used, `linked` (a preset names the head) or `auto`, its index and count, and the texture hashes |
| `skin_used` and the skin's bones | ped clothing (added 2026-11-21): which skin was written (the bone palette joined to the shared ped skeleton when at least 90 % of the palette resolves), its bones, and any unresolved joints. The shared skeleton is read from `packs.rpf`, so index `packs` first. GLBs get `JOINTS_1` / `WEIGHTS_1` for the second four influences. Ped clothing exported before this date is wrongly skinned; re-export it |

`classification` = `{class, render_bucket, alpha_mode, instanced_or_batch_lod, family, record_flags, decal}`;
`class` is one of `opaque`, `alpha`, `cutout`, `decal`, `glass`, `window`,
`emissive`, `water`, `terrain_multilayer`, `car_paint`.

`decal` (added 2026-11-20) is set on decal-bucket materials (road markings,
grime, signage, terrain blend layers) and null otherwise:
`{bias_m, order, order_source, depth_write}`. These surfaces are coplanar
with the geometry under them, so draw them after opaque geometry, in
ascending `order`, offset toward the camera by `bias_m` along the normal
(or an equivalent depth bias); `depth_write` is true for opaque/masked
decal layers. No per-material priority is known in the shader record, so
`order` comes from the class: terrain blend layers 0, other decals
(markings, grime, signage) 1 (confidence: medium). In the viewers a depth
bias of a few millimetres plus this order removes the fighting.

`multilayer` (added 2026-11-20) is set on the two-layer blend shaders
(walls, ground pieces, terrain blend decal; format spec §12):
`{layers: [base, layer2], overlay, extra_data, vertex_color_channels,
confidence, note}`. Each layer is `{name, colour, normal, data,
two_channel, uv_set, weight}`, and each texture is `{param, png, status}`.
Suggested blend: `lerp(base, layer2, COLOR0.a)` with base on TEXCOORD_0
and layer 2 on TEXCOORD_1, then the overlay by its own alpha (UV set
unknown). The grouping of slots into layers is confirmed. The weight
channel and the layer-2 UV set are inferred, the base UV set is a weak
inference, and the data and two-channel meanings are unknown.

Each `textures[]` entry has `param`, `param_hash`, `hash64`, `hash32`,
format, size, tile mode, `role_guess`, `slot_role`, the PNG path or the
reason it is missing, `mip_used` when exported at a lower mip, and
`texture_fallback {reason, role, rgba, source}` when no mip decodes.

The `paint` block: `colour` (null: runtime colour, not decoded),
`suggested_default_linear_rgb`, `masks` (decal, dirt, grime, scratches,
mask pack slot hashes) and short CSP / BeamNG hints.

**GLB contents:** nodes `root > modelN > geomNN_<off>`; attributes
`POSITION`, `NORMAL`, `TANGENT` (where the game has one), `TEXCOORD_0..3`,
`_COLOR0` (raw RGBA8), and for fragments `JOINTS_0` (u8, or u16 above 255
bones) / `WEIGHTS_0` with a glTF skin over the skeleton joints. One
double-sided material per shader, alpha mode from the class. Car paint gets
a grey `baseColorFactor`, metallic 0.6, roughness 0.25 and no albedo.

## Placements: `<off>.json` and `index.jsonl`

A map file JSON holds the file's own fields (extents, name and parent
hash), its `space` (`world`, `interior`, `world+interior` or `group`: interior
and group files are in local space, not world space), `interiors` (per
interior: bbox, rooms, entity sets with counts) and `entities[]`:

| Field | Meaning |
|---|---|
| `archetype` | joaat of the model name (hex) |
| `name`, `model_id` | model name and entry id, when resolved |
| `position` | RAGE world, Z-up |
| `rotation` | quaternion as stored (= inverse) |
| `rotation_world` | the real world rotation |
| `gltf` | `{translation, rotation, scale}` ready for placing `model.glb` in a Y-up scene |
| `scale_xy`, `scale_z` | scale |
| `flags`, `guid`, `hash64` | raw |
| `lod` | `{level: HD/LOD/SLOD1–4/ORPHAN_HD, parent_index, parent_link, num_children, lod_dist, child_lod_dist, priority}`. After `sixng lod-links` also `archetype_lod_dist`, `hd_texture_dist`, `effective_lod_dist` (own `lod_dist` if > 0, else the archetype's) and `parent` `{kind: entity \| batch, map, parent_index` or `batch` + `instance`, `archetype, name, model_id, distance, method, confidence, alternatives}`; `method` is `parent_hash`, `region`, `spatial`, `batch_position` or `batch_index_name`, `confidence` high / medium / low (added 2026-11-21) |
| `interior` | on entities of interior files: `{interior, room, room_name, entity_set, entity_set_name}` (added 2026-11-21) |
| `render`, `render_reason` | present only when the placement should not be drawn by default: `render: false` and a reason — `placeholder` (no model resolved), `vfx_proxy` (effect markers, e.g. `*vfx*proxy*`, `vfx_dummy*`), `interior_proxy` (interior-shell stand-ins `*_pxy`, `*shell_proxy`, `int_*_proxy`, which duplicate the interior shells), `shadow_proxy` (`*_shadow` shadow casters) or `light_proxy` (`*_lit_proxy`, zero-size geometry). Rules come from exporting examples; `gmt_*_proxy_*` shop items and `*_occl_*` cables are real geometry and are not flagged. Confidence: medium (name rules, checked on the examples). Added 2026-11-20 |
| `fields` | every other scalar field, by hash |

`index.jsonl`, one line per map file: `id`, `file`, `name_hash`,
`parent_hash`, `parent_id` (the map file `parent_index` points into),
`entities`, `resolved`, `bbox`.

## Batches

`batches.jsonl`, one line per batch: `kind` (`prop` / `grass`), `map`,
`archetype`, `name`, `model_id`, `bbox`, `lod_distance`, `scale_range`
(grass), and `first` / `count` into the binary file of its kind.

| File | Record | Layout (little-endian) |
|---|---|---|
| `batch_props.bin` | 44 B | position f32 × 3, quaternion xyzw f32 × 4, scale f32 × 3, tint RGBA u8 × 4 |
| `batch_grass.bin` | 20 B | position f32 × 3, normal x / y u8, colour u8 × 3, scale u8, AO u8, pad u8 |

## `interiors.jsonl`

One line per interior instance: `archetype`, `map`, `instance`, `position`,
`rotation_world`, `gltf`, `interior_file`, `entities` and
`entity_set_entities` (indices into the interior file's entities),
`bbox`, `collision` (exact bbox matches), `collision_candidates`, and
`collision_linked {id, method (exact / ordered / floor_fit / bbox), score,
fit, iou, margin, confidence}`. Place interior entities with their own
transform, then the instance transform.

## Collision: `index.jsonl` and GLBs

`index.jsonl`, one line per bounds file: `id`, `file`, `bbox` (world),
`stats` (polygon counts by type), `materials[]` (surface id, flags,
triangle count) and `primitives` (exact spheres, boxes, capsules,
cylinders).

Each GLB is in world space (Y-up), with one primitive per surface id,
material named `surface_<id>` with a stable colour and the 8-byte record
in `extras`. Boxes are 12 triangles, spheres octahedra, capsules and
cylinders 8-sided prisms. No normals (flat shading).

## `surface_categories.json`

| Field | Meaning |
|---|---|
| `note` | method summary |
| `area_covered` | share of collision area with a category (0.983) |
| `area_by_confidence` | share per `high` / `medium` / `low` |
| `samples` | sample counts per cue |
| `surfaces` | map of surface id (hex) → `{category, confidence, round, evidence}` |

`evidence` holds the numbers behind the label (`area_share`, `road_share`,
`up`, `vertical`, `mean_z`, `path_link_ratio`, `plant_families`,
`prop_families`, `railway_props`, `earlier_category` …).

## `graph.json`

Top level: `node_file`, `link_file`, `junction_file`, `coordinate_system`,
notes, and three arrays.

| Array | Fields |
|---|---|
| `nodes[]` | `p` (RAGE world), `address` [area, index], `speed` (m/s), `street` (hash), `flags` (hex), `flag_names`, `key` [area, index], `kind`, `road_class`, `num_links`, `density`, `bytes` (raw) |
| `links[]` | `from`, `to` (node indices), `length` (m), `lanes_to`, `lanes_from`, `flags`, `flag_names` (game names, or `[inferred]` labels, or `bitN`), `raw` |
| `junctions[]` | `node`, `entrances[] {node, lane, stop, yield, paths[] {to, start, end, length, lane}}`, `phases[] {duration, lights [[hash64, boxId]]}`, `u38` |

A junction with `phases` is signalised; the light hash matches the
`hash64` of a placed traffic-light prop.

## Lights

`sets.jsonl`, one line per light set: `id` (meta file), `owner`
(`archetype` / `entity`), `archetype`, `name`, `model_id`, `entity_index`,
`ext_name`, `ext_offset`, `lights[]` with `position` (local), `direction`,
`tangent`, `colour`, `colour2`, `colour3`, `light_type`, `intensity`,
`range`, `falloff_exponent`, `cone_inner`, `cone_outer`, `corona_size`,
`corona_intensity`, `vol_intensity`, `flashiness`, `light_fade_distance`,
`shadow_fade_distance`, `time_flags` (24-bit hour mask), `night_only`, and
the raw `fields`.

`world.jsonl`, one line per placed light: `p` (world), `dir`, `archetype`,
`light` (index in its set), `colour`, `type`, `intensity`, `range`, `cone`
[inner, outer], `night_only`, `map`, `entity`, `owner`, `space` (`world`
or `group_local`), and `interior_map` / `interior_instance` for lights
inside interiors.

## `texture_gaps.json`

`summary` tables (by class, format, slot role, terrain layer, usage and
area, top flat materials) and a `textures` table: each affected texture's
store, size, format, `status` (`flat` = every mip in BCPack, a fallback was
written; `lowmip` = exported at mip N), mip and export size.

`texture_gaps_models.jsonl.gz`, one line per affected model: `model_id`,
`model`, `status`, `placements`, `area_m2`, `footprint_m`, `lod_proxy`,
`name_surface`, `collision_under` (top 3 inferred surface categories under
its placements), and `materials[]` with `shader`, `shader_class`, `class`,
`status`, `colour_status`, `slots[]` and, for layered materials,
`layers[]` (`status`, `affected_roles`, `surface_type`, `colour_rgba` …).

---

← [Output layout and coordinates](04-output.md) · [Index](README.md)
