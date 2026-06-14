# Tree-Aligned Macro Index Implementation

This note documents the current experimental Macro Bound + Original Fine Traversal path.

## Goal

The physical streaming layout remains the existing RAD/RADC paging:

- `*.rad` header
- `*.radc` data chunks
- 65536 splats per page
- existing `SplatPager` fetch/decode/upload path

The macro traversal unit is not the 64K storage chunk. A separate tree-aligned sidecar file provides coarse LoD units:

```text
scene.rad
scene-0.radc
scene-1.radc
scene.macro-index.bin
```

This keeps I/O and visual traversal as two separate structures.

## Offline Macro Index

`rust/build-lod/src/main.rs` builds `*.macro-index.bin`.

The writer walks the existing LoD tree and cuts subtrees into macro units. A subtree becomes one macro unit when its descendant splats touch at most `DEFAULT_MACRO_UNIT_PAGES` RAD pages. The current value is `16`.

Each unit stores:

- `root`: raw LoD node index, in RAD tree index space
- `page_start`, `page_count`: slice into the global page list
- `sphere_center`, `sphere_radius`: conservative bound for the subtree
- `s_max`: maximum `feature_size()` inside the subtree
- page list: RAD pages covered by the subtree

Binary format:

```text
u32 magic = 0x5844494d  // "MIDX" little-endian bytes
u32 version = 1
u32 unit_count
u32 page_ref_count

unit[unit_count]:
  u32 root
  u32 page_start
  u32 page_count
  f32 center_x
  f32 center_y
  f32 center_z
  f32 sphere_radius
  f32 s_max

u32 pages[page_ref_count]
```

Commands:

```powershell
cargo run --manifest-path rust\build-lod\Cargo.toml --release -- --rad-chunked D:\path\scene.rad
cargo run --manifest-path rust\build-lod\Cargo.toml --release -- --macro-index-only D:\path\scene-lod.rad
```

`--macro-index-only` reads an existing monolithic LoD RAD and writes only the sidecar index. For header-only chunked RAD files, generate from the original monolithic RAD and copy/rename the sidecar to match the chunked RAD basename, provided the rechunk step did not reorder splats.

## Runtime Load Path

`src/SplatPager.ts` derives the sidecar URL from the RAD URL:

```text
foo.rad -> foo.macro-index.bin
```

If the sidecar fetch succeeds, `SparkRenderer.initLodTree()` sends it to the LoD worker. If the sidecar is missing, runtime logs a message and falls back to the original root-based traversal behavior.

WASM entry:

```text
set_lod_macro_index(lodId, macroIndexBytes)
```

The parser stores:

- `macro_units`
- `macro_pages`

on the same LoD tree record as the paged splat data.

## Macro Traversal

`lodTraverseMode: "macro"` calls:

```text
macro_traverse_lod_trees()
```

For each macro unit:

1. Compute a conservative bound:

```text
P_bound = s_max * lodScale / max(eps, distance(camera, sphere_center) - sphere_radius)
```

2. If `P_bound <= pixelScaleLimit`, coarse accept the unit:
   - output the unit root if its root page is resident
   - otherwise touch the root page to request it

3. If `P_bound > pixelScaleLimit`, the unit is active:
   - touch missing pages in the unit page list
   - if the root is resident, seed the original fine traversal heap with that root

4. The fine traversal loop is otherwise the existing Spark traversal:
   - same pixel-scale scoring
   - same max splat budget
   - same child-page residency checks
   - same output packing

If no macro unit seeds an instance, the code falls back to the original root seed.

## Debug Output

Offline `build-lod` prints:

```text
Wrote scene.macro-index.bin (38919 macro units, 352512 page refs, max_pages_per_unit=16)
Macro index stats: pages_per_unit min/avg/max=... s_max min/avg/max=...
```

Runtime load prints:

```text
[Spark LoD] fetched macro index url=... bytes=...
[Spark LoD] macro index loaded lodId=... bytes=... units=... pageRefs=...
```

Runtime traversal prints at most once per second:

```text
[Spark LoD] traversal {
  mode,
  ms,
  pixelLimit,
  totalLodSplats,
  chunks,
  outputSize,
  frontierSize,
  leafCount,
  macroCoarseChunks,
  macroActiveChunks,
  macroUnitsConsidered,
  macroSeededRoots,
  macroMissingPageRefs,
  macroFallbackInstances
}
```

Use the streaming LoD example's `Traversal` GUI control to compare `standard` and `macro` on the same camera path.
