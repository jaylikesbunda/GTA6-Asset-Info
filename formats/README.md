# SixNG format specification

GTA VI, Xbox Series build. Everything here was worked out from the data
itself: headers, statistics over many samples, and checking decoded output
by eye. Nothing has been confirmed against game code or official
documentation.

Numbers are **as of 2026-11-20**, from one retail Xbox Series install.

**Confidence levels** used in every table:

- **confirmed**: holds over many samples, and decoded output looks right
  when opened.
- **inferred**: consistent with the data, not proven.
- **unknown**: not worked out.

**Conventions:** offsets are hex from the start of the structure;
little-endian unless stated. Hashes are hex. "joaat" is the
Jenkins one-at-a-time hash used by RAGE; where a field name is known it is
the **case-sensitive** joaat of that name. "GTA V equivalent" names
(`CEntityDef`, `.ydr`, `.ynd` …) are the names the same structures have in
GTA V, used only to say what a structure corresponds to.

## Pages

**Containers**

- [1. RPF9 archives](01-rpf9.md)
- [2. Resources and pages](02-resources.md)

**Textures**

- [3. Texture dictionaries](03-texture-dictionaries.md)
- [4. Xbox Series texture tiling (dictionaries)](04-xbox-tiling.md)
- [5. `ttpf` texture stores](05-ttpf.md)
- [6. BCPack](06-bcpack.md)

**Models**

- [7. Drawables, fragments and vehicles](07-models.md)

**World data**

- [8. Meta resources (type 0x7a)](08-meta.md)
- [9. Placements, LOD links, archetypes and batches](09-placements.md)
- [10. Interiors (MLO)](10-interiors.md)
- [13. Collision and surface categories](13-collision.md)
- [14. Navigation meshes](14-navmesh.md)
- [15. Lights](15-lights.md)
- [16. Road graph, junctions and flags](16-roads.md)

**Materials**

- [11. Vehicle paint and vehicle tables](11-vehicle-paint.md)
- [12. Material classes and texture slots](12-materials.md)

**Other**

- [17. Other formats](17-other.md)

Each page ends with an **Open questions** list.

**Byte-level notes.** The [structure/](../structure/00-INDEX.md) pages
record layouts as raw byte facts with the test behind each one. They also
cover audio (`ADAT`), the PSO object graph and the shader cache, which these
pages only mention. Where the two sets disagree, both readings are given.
