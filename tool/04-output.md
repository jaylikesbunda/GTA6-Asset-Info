# Output layout and coordinates

*Part of the [SixNG tool guide](README.md). Numbers and timings as of 2026-11-21.*

## Output layout

```
<out>/
  manifest.jsonl                      one line per exported item
  textures/<archive>/<name>_<off>/    a texture dictionary
    dictionary.json
    <texture>.png / .dds / .json
  textures/ttpf/<hash>.png / .json    store textures, shared by all models (written once;
                                      slice 0 / face 0 of arrays and cube maps)
  models/<archive>/<name>_<off>/      a drawable or fragment
    model.glb, model_lodN.glb         triangle meshes per LOD
    points.glb                        only for mesh blobs with no index buffer
    model.json                        description
    buffers/NN_<gfxoff>.bin           raw GPU buffers
    sys.bin                           decompressed system page
    textures/                         embedded textures
  raw/<archive>/<name>_<off>.<ext>    anything else (xml, pso, rbf, bin …)
  placements/<archive>/
    <off>.json                        one map file
    index.jsonl                       one line per map file
    batches.jsonl, batch_props.bin, batch_grass.bin   instanced batches
    interiors.jsonl                   interior instances
  collision/<archive>/
    <off>.glb                         one bounds file
    index.jsonl                       one line per bounds file
    surfaces.json                     triangle count per surface id
    surface_categories.json           inferred categories
  navmesh/<archive>/<off>.glb, index.jsonl
  paths/<archive>/graph.json          road graph
  lights/<archive>/sets.jsonl, world.jsonl   lights
  meta/<archive>/index.jsonl, <off>.json      meta schema index / dumps
  pso/data/<offset>.json              one decoded PSO file
  pso/handling.json                   vehicle handling records
  vehicles/vehicles.json              model info, configurations, colour lists, part groups
  movies/movies.json, *.bk2           Bink videos (list; files with --extract)
  texture_gaps.json                   BCPack census (path set by --file)
  texture_gaps_models.jsonl.gz        one line per affected model
```

Schemas: [`model.json`](05-schemas.md#modeljson) ·
[placements](05-schemas.md#placements-offjson-and-indexjsonl) ·
[batches](05-schemas.md#batches) ·
[`interiors.jsonl`](05-schemas.md#interiorsjsonl) ·
[collision](05-schemas.md#collision-indexjsonl-and-glbs) ·
[`surface_categories.json`](05-schemas.md#surface_categoriesjson) ·
[`graph.json`](05-schemas.md#graphjson) ·
[lights](05-schemas.md#lights) ·
[`texture_gaps.json`](05-schemas.md#texture_gapsjson)

The work folder holds the indexes that later steps pick up:

```
<work>/
  index/archetype_links.json          link-archetypes
  index/archetype_lod.jsonl           lod-links: archetype LOD distances
  index/lod_parents.jsonl             lod-links: per map file, entity index → parent
  index/interior_entities.jsonl       interior-entities: space and membership per non-world file
  names/names.json                    names: hash → name dictionary
  index/head_skins.json               head → face table (built from the packs index, ~5 s)
```

## Coordinates

- Source data is **RAGE: Z-up, right-handed, metres**.
- GLBs are **glTF Y-up**: `glb(x, y, z) = rage(x, z, −y)`. Blender's glTF
  importer converts back to Z-up. glTF forward is −Z (RAGE +Y).
- Model GLBs are in model space; collision and navmesh GLBs are in world
  space (no node transform needed).
- JSON positions are RAGE world space unless the field says `gltf`, except
  in map files whose `space` is `interior` or `group`: those are
  interior-local or group-local.
- **Placement rotations are stored inverted** (as in GTA V): use
  `rotation_world` or the ready-made `gltf` transform. Interior instance
  rotations are not inverted.

---

← [GUI](03-gui.md) · [Index](README.md) · [File schemas](05-schemas.md) →
