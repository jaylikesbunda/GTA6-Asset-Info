# 4. PSO (`PSIN`) and the serialised object graph

## `PSIN` container

Section headers are big-endian. Sections are walked by a u32 length that **includes its own 8-byte
header**. Inside the sections (the `PSIN` block table, the references and the `LSCH` schema) the data
is little-endian; the full layout is in [formats §17](../formats/17-other.md#pso).

Evidence sample (99,744-byte `PSIN` payload) — first 32 bytes:

```
50 53 49 4e | 00 01 09 90 | 7a 07 d1 8b | 60 18 00 00
PSIN          0x00010990                    0x00001860
```

Read big-endian the first section length is `0x00010990` = 67,984 and the walk yields
`PSIN → LSCH → LENM → LSNS → LGID → LRSC → CHKS`, closing exactly at EOF. Read little-endian the
first length becomes `0x90090100` (2.4 GB) and the walk fails immediately. Two of three sampled
payloads close at EOF this way; the third is a 458,752-byte body whose first length is `0x0006F0E0`
and which does not close — recorded as an open case rather than assumed.

| Tag | Meaning |
|---|---|
| `LSCH` | schema: struct table and member records |
| `LENM` | enumeration table |
| `LSNS` | data section (see framing below) |
| `LGID` | global id section |
| `LRSC` | a copy of the meta resource header (`0DRP`) |
| `CHKS` | footer / checksum marker |

A second section family exists: `PMAP`, `PSCH`, `PSIG`, optional `STFN`/`STDN`, `STNS`, `CHKS`.

`LRSC` bodies begin with the bytes `0DRP` (u32 little-endian `0x50524430` when read as ASCII
`0DRP`). In the tested set these bodies are uniformly 80 bytes and contain no printable names.

### `LSNS` framing

Payload begins:

```
u32 LE 1 | 01 01 00 00 | u32 LE tail_length
```

`tail_length = size − 12`; tails are 16-byte aligned. A variant starts with a count word
`N ∈ {1,2,3,4}` instead of 1; for `N = 1` the `tail = size − 12` rule holds exactly.

### Walk failures

Where the section walk fails, the cause is a 65,536-byte page cut: the declared header offset
exceeds the stored chunk, or a section overruns at `LSNS`. Both are consequences of the
archive-level 64 KiB chain rule rather than separate format variants.

## Serialised object graph

One payload in the archive set (2,702,384 bytes) is a self-describing typed-object graph:

| Property | Value |
|---|---|
| record count | 30,683, stored at `0x14` (the only non-zero header word) |
| offset table | `0x28 .. 0x3BF00`, strictly ascending u64, `0x28 + count × 8 = 0x3BF00` exactly |
| tiling | records tile `0x3BF00 .. EOF` with zero gaps; every record 16-byte aligned |
| record sizes | 81 distinct values, minimum `0x20` |
| type slot | `rec + 0x08`, u64: 67 values (66 type hashes plus array schema `0x14`) |
| array schema | 10,294 records; `size = align16(0x30 + count × stride)` exactly, count at `+0x10` (duplicated at `+0x14`), stride at `+0x18` |
| pointers | 30,200, each a u32 followed by the marker `0x71717171`; value = target record start + 8 |
| roots | 483 records never targeted, each with a unique non-zero u32 id at `+0x00` |
| reachability | 30,683 of 30,683 from the roots; maximum depth 9 |
| entropy | maximum 5.48 bits/byte; no strings |

Every non-root record has exactly one parent, so the structure is a forest of 483 trees.

### Name resolution

Some slots hold joaat hashes of authored tokens. 289 slot values resolve: 277 of 483 root ids are
vehicle model names, 41 of 56 values in type `0xC48AB516` at `+0x10`, 10 keyref keys, 5 of 8 bone
slots. Only joaat exceeds chance (z ≈ +128); fnv1a, fnv1, crc32, djb2 and sdbm all sit at the
random expectation.

Unresolved after an expanded 1.3M-token universe: 206 roots and all 67 type hashes. A 64-bit slot
interpretation has not been ruled out for the type field.

## Open questions

- Semantics of the 67 type hashes (schema vocabulary is inferred, not proven).
- Whether the 483 trees are one object model or several.
