# GUI

*Part of the [SixNG tool guide](README.md). Numbers and timings as of 2026-11-21.*

```powershell
sixng-gui
```

The **Game** and **Work** fields each have a **Browse…** button. The app
picks its folders in this order:

1. the `SIXNG_GAME` and `SIXNG_WORK` environment variables;
2. the folders saved with **💾 Save** (kept in `sixng-settings.json` next to the exe, or in
   `%APPDATA%\SixNG\settings.json` if that folder is not writable);
3. for the work folder only, a `work` folder next to the exe.

There is no default game folder. On first start (or if the saved folder no
longer exists) the app shows **Choose game folder…**; the folder you pick
is saved for next time, and the CLI uses it too.

It never searches parent folders, so it behaves the same wherever you start
it from. Restart the app after saving new folders. On first start the app
also builds the name dictionary (`sixng names`) in the background.

## Browser tab

1. The left panel lists the archives. Click **Index** on any that are not
   indexed yet.
2. The middle panel lists the selected archive's entries. Filter with a
   regex and the kind and type drop-downs. Tick rows, then **Extract
   selected** or **Extract all N filtered** into **Output**; the **PNG**
   and **DDS** boxes pick texture formats.
3. The right panel previews the entry: decoded textures for dictionaries,
   the JSON description for models, a text or hex view otherwise. **Open
   in model viewer** opens models.
4. The **World data** row runs the world steps (Placements, Batches,
   Collision, Road navmesh, Lane graph, Lights, Interiors, Surfaces) into
   **Output**. Run them left to right.

## Model viewer tab

Pick a model from the **Models** list on the left. It holds every
drawable and fragment in `levels.rpf` and `packs.rpf`, in these
categories: Vehicles (about 240 plus `_hi` copies), Vehicle parts, Wheels,
Weapons, Props, Vegetation, Interiors, Buildings & map, and Unnamed.
- Search box: words match names and ids.
- **hide LOD copies**, on by default.
- Favourites (☆) and a **Recent** list.
- Click to load. **Up / Down** or ◀ ▶ step through the list.

You can also open by entry id, by name regex (`^phoenix_hi$`; prefix
another archive as `data:^w_pi`), from an exported `.glb`, or from the
Browser. Entries are exported on the fly into `<work>/gui_cache_v4`;
older `gui_cache_v*` folders can be deleted.

- Shading: base colour, normal map (DirectX green), packed
  roughness/metal/AO with a channel-order switch, alpha / cutout / decal /
  glass classes, emissive.
- Car paint: primary, secondary, pearl colour and amount, finish (metallic,
  matte, chrome, gloss).
- Vehicle data (from `sixng vehicles`): a car opens with its first spawn
  configuration (body colour, rim, tyre); the other configurations are
  listed, and the paint-colour lists show as swatches with their shares.
  Rims and tyres are sized as 1.625 × `wheelScale`, front and rear
  separately (a fitted factor).
- **Head skin ◀ n / N ▶** (ped heads) switches through the faces; the
  **Head skin** field picks the face for exports (`--head-skin`).
- **Body shape** field (ped clothing): the same controls as
  `--body-shape` ([CLI](02-cli.md#output-flags)), applied to exports. The
  viewer itself draws the bind pose, so the shape shows in the exported GLB
  (Blender and other glTF viewers), not in the viewer.
- **LOD** buttons; debug views (albedo, normals, UV checker, single texture
  slot, **BCPack gaps** green/amber/red, vertex colour, lighting only);
  wireframe and normal lines.
- Skeleton with joint axes and labels; **Rig** sliders spin the wheels and
  open the doors. Cars get a shared rim and tyre at their wheel mounts
  (their configuration's, or a default one without vehicle data).
- Orbit or fly camera; time of day, exposure, ambient, emissive and a wet
  look.
- Per-material list with the shader name, class, alpha mode, gap status
  and every texture slot (with its parameter name where known); show, hide
  and solo.
- **Screenshot** saves the 3D view and the window under the work directory.

## Map 2D tab

First bake the map from the world exports. **Bake map** / **Re-bake
map** at the top of both map tabs does it in the app:
- It runs as a separate process, with progress, **Cancel**, and a
  reload when done (about 2 minutes).
- It names any World data steps still missing (placements are required).

Or from the command line:

```powershell
sixng-gui --bake-map            # full bake, ~2 min, ~1 GB in <work>/map
sixng-gui --bake-map raster     # rasters only, ~45 s, reuses the collision cache
```

The bake writes a raster tile pyramid (`tiles/<layer>/<z>/<x>_<y>.png`, 256 px,
2 m per pixel at the top level; layers: hill-shaded terrain, surface
categories, building footprints, navmesh, vegetation/prop density,
texture-gap heatmap, LOD regions, water / no collision) plus vector data
(`roads.json`, `junctions.json`, `lights.bin`, `entities.bin`,
`strings.json`, `interiors.json`, `heights_8m.bin`).

In the tab:

- pan and zoom; a base layer (terrain, surfaces or water) with overlays at
  their own opacity;
- the lane graph coloured by speed limit, lane count or road class, with
  one-way arrows;
- junctions cycling through their signal phases; stop and yield entrances;
- lights with exterior/interior toggles and an hour slider (each light's
  hour mask);
- interior outlines;
- **Inspect** (click a placement: model, archetype, LOD level, position in
  game and glTF axes, map file, lights) and **Open in model viewer**;
- **Search** by name, model id or archetype hash;
- **Measure**; **Box select** shows the `--bbox` for region exports;
- a readout of game x/y/z and the glTF/BeamNG position (x, z, −y).

## Map 3D tab

A fly-through of the placed models, streamed from the baked map (bake
first, as above):

- drag to look, WASD/arrows to move, Q/E down/up, Shift faster (speed
  scales with height above ground);
- models are exported on demand into `<work>/gui_cache_v4` by background
  workers and drawn instanced (one GPU model per model id, its placements
  as instances). Loads go on-screen first, then big and near before small
  and far; models just outside the view stay loaded for turning round;
- **texture streaming**: a model first loads with 64 px textures, read
  from reduced copies (≤ 256 px, kept in `<work>/gui_cache_v4/lowtex`,
  capped at 4 GB), then reloads with sharper ones as the camera comes
  closer (full size in the HD band, then 1/2, 1/4, 1/8). Textures are
  shared between models;
- **Quality** (low / medium / high / ultra, saved in
  `<work>/gui_settings.json`) sets the detail, a triangle budget, a
  draw-call budget, the far-field error and the texture cap. Over either
  budget the smallest objects on screen are dropped first; both budgets
  have their own sliders;
- a **Detail** slider for the LOD distance bands (HD, LOD, SLOD1–4), prop
  and vegetation density and minimum screen size, each also adjustable on
  its own, plus the drawable LOD and a texture size cap;
- **HD fallback** (default on): most LOD-level placements have no linked
  model yet, so large HD models stay visible to 1.75× the HD distance
  and the LOD level is skipped;
- **whole-map terrain**: the whole map is drawn out to the horizon as a
  quadtree of tiles over the baked 8 m heights, coloured with the 2D map's
  terrain tiles. Tiles refine by screen-space error (**terrain error**, px);
  children replace a parent only once all four have loaded. Beyond the HD
  building range the tiles keep the roofs, so buildings without a loaded
  model show as blocks; near the camera a tile switches to ground heights
  once at least 90 % of its building models have loaded. A sea plane fills
  the rest;
- road and exterior-light overlays (by hour), a draggable **Minimap**
  (click to move) and **Stream** statistics window; click a model to
  inspect it and **Open in model viewer**;
- LOD hand-off from the placement LOD links: a parent LOD/SLOD is drawn
  until the camera is inside its `child_lod_dist` *and* all its children
  have loaded, then the children replace it, so the two are never drawn
  together. A child whose parent has no model stays visible out to the
  parent's distance. Placements without links use the distance bands;
- proxies and placeholders (see the placements `render_reason`) are
  hidden; **show proxies / placeholders** draws them for debugging;
- decal-bucket materials are drawn after opaque geometry by layer order
  with a small depth bias, and the renderer uses a reversed-Z 32-bit
  depth buffer, so coplanar markings, grime and signage and distant
  surfaces do not flicker; **decal depth bias** turns this off for
  comparison;
- **Memory** limits that scale with the installed RAM: past 20 % of RAM
  only nearby models load (at 64 px) and everything off screen is dropped;
  past 30 % the farthest visible ones are dropped too. The texture budget
  is RAM / 16 (0.5–3 GB) and the model cap 120 per GB (800–8,000). Untick
  **automatic** to set your own limit; it is saved in
  `<work>/gui_settings.json`. The process figure includes memory the
  graphics driver commits for GPU buffers and textures.

Coverage notes (confidence: high, checked on downtown): about 88 % of
LOD-level placements and most SLOD placements carry no model link or only
a placeholder unit-cube bounding box, and SLOD ground tiles ship without
normals (the viewer derives them).

## Textures tab

Every texture in the `ttpf` stores, without going through a model.

- Filter by store, by format and by minimum size; the count updates as you
  filter.
- Pick a texture to preview it: choose the **mip** and the **channel**,
  and zoom (0 = fit).
- **Export PNG** saves the shown mip to `<work>/out/textures/`
  (`<hash>_m<mip>.png`); **Show in Explorer** opens the folder.
- In the model viewer's material list, **view** on a texture slot opens
  that texture here.

BCPack BC1 textures decode; textures with no readable mip (the other
BCPack formats, [§6](../formats/06-bcpack.md)) cannot be previewed. Texture arrays and cube maps show slice 0 / face 0;
use `sixng ttpf --all-slices` for the others.

## More coming

Planned: a user-friendly library home and setup wizard, per-target
previews (BeamNG, Assetto Corsa), day/night, a change report, and an
audio browser.

## Command-line screenshots

For checks and docs:

```powershell
sixng-gui --model levels.rpf@10f266d010 --mode 5 --yaw 2.4 --pitch 0.25 --screenshot car.png --exit
sixng-gui --model levels.rpf@115e6c6010 --picker-cat vehicles --window-screenshot picker.png --exit
sixng-gui --tab map2d --start-bake --screenshot map.png --exit   # bakes first, shot after reload
sixng-gui --tab map2d --center -670,60 --mpp 2 --base 0 --overlays 2,3 --lights --hour 22 --screenshot map.png --exit
sixng-gui --tab map3d --center -640,-60 --height 450 --yaw 225 --pitch 30 --detail 0.5 --tod 17 --screenshot city.png --exit
```

Model options: `--lod N`, `--mode 0-7` (shaded, albedo, normals, UV, slot,
gaps, vertex colour, lighting), `--slot 0-3`, `--rma 0-3`, `--wire`,
`--skeleton`, `--normals`, `--tod H`, `--paint r,g,b`, `--finish 0-3`,
`--pearl A`, `--wet A`, `--exposure E`, `--spin R`, `--door R`, `--yaw R`,
`--pitch R`, `--zoom F`, `--size WxH`. Map options: `--center X,Y`,
`--mpp M`, `--base N`, `--overlays a,b`, `--lights`, `--hour H`,
`--search name`. 3D map options: `--center X,Y`, `--height H`, `--yaw DEG`,
`--pitch DEG`, `--quality low|medium|high|ultra`, `--detail 0-1`, `--tod H`, `--lights`, `--roads`,
`--no-terrain`, `--fog F`, `--max-tex PX`, `--no-textures`,
`--no-hd-fallback`, `--kinds a,b`, `--lods a,b`, `--ram-cap GB`,
`--no-windows`, `--no-decal-bias`, `--show-proxies`, `--fly X,Y,H,SECONDS`,
`--no-vsync`; the shot waits until every wanted model has loaded (15-minute
limit) and also saves `<name>_ui.png` with the panels. `--fly` waits for the
start view, flies to (X, Y, H metres above ground) and logs the frame
times; timings go to stderr and `<work>/map3d_perf.log`. The model viewer also takes
`--no-decal-bias`. `--tab textures` opens the Textures tab, and
`--texture <hash>` opens one texture there.

---

← [CLI](02-cli.md) · [Index](README.md) · [Output layout and coordinates](04-output.md) →
