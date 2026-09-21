# CLAUDE.md

Working notes for agents building or porting RPaks with this repo. Read the
README first for the layout. Everything below is what the manifests here
assume and what breaks when you deviate from it.

## 1. The two targets

| target | engine | pak dir | compression | asset universe |
|---|---|---|---|---|
| S21 client | Apex Season 21 client | `paks/Win64/` | uncompressed (`headerFlags: 32`, `compressLevel: 0`) | full: `mdl_ txtr matl shdr shds txan dtbl aseq arig txtx uiia wrap rmap efct stgs` |
| S3 dedi | Season 3 dedicated server | `paks/Win64_server/` | zstd (`compressLevel: 6`, build-list `"dedi": true`) | server only: `mdl_ aseq arig stgs stlt dtbl txls Ptch` |

The dedi never renders. Its models carry collision, bones, hitboxes and
physics only (`.rmdl` + `.phy`, no vertex groups, no starpak). Everything
visual is client-side.

Asset versions per target:

| 4cc | S21 client | S3 dedi | notes |
|---|---|---|---|
| `mdl_` | 17 | 10 | rmdlconv; the dedi convert also rewrites the `.phy` header and updates `phySize` |
| `aseq` | 11 | 7 | R5-AnimConv |
| `arig` | 6 | 4 | R5-AnimConv |
| `matl` | 23 | not packed | |
| `txtr` | 10 | not packed | streamed mips go to the starpak pair |
| `shdr` / `shds` | 15 / 12 | not packed | |
| `stgs` / `stlt` / `dtbl` / `txls` / `Ptch` | 1 / 0 / 1 / 1 / 1 | same | shared writers |

Donor versions you will meet:

| donor | `mdl_` | `aseq` | `arig` | `shdr` / `shds` | `matl` | rBSP |
|---|---|---|---|---|---|---|
| S3 / TF2-era | 8 (IDST 54) / 49 | 7 | 4 | 12 / 11 | 15 | 47 / 37 |
| S10 | 12.2 | 10 | 4 | 15 / 12 | 18 | 49 |
| S21 | 17 | 11 | 6 | 15 / 12 | 23 | 51 |
| S25 | 19.1 | 12.1 | 7 | 15 / 13.1 | 23 | 52 |
| S26 | 19.1 | 12.1 | 7 | 16 / 14 | 23.2 | 52 |
| S30 | 19 (S30 variant) | 13 | 7 | 16 / 14 | 23.2 | 52 |

Version numbers that match are not proof of compatibility. S25 `shdr` is v15
like S21 and still needs the cbuffer conversion (section 6). S10 `shdr` v15
really is packable as-is.

## 2. Manifests and build lists

- Build lists are the unit of a Repak run. The build settings (`dedi`,
  `keepClientOnly`, `version`, `outputDir`, stream file names) are read from
  the build list, not from the per-pak manifest. A per-pak manifest's
  `assetsDir` / `outputDir` resolve relative to the build list's directory,
  so run everything from the repo root.
- Manifest schema: `{version, name, assetsDir, outputDir, headerFlags,
  streamFileMandatory, streamFileOptional, files:[{_type, _path, $guid, ...}]}`.
  `_path` is the canonical asset name; `$guid` pins the GUID (otherwise
  `StringToGuid(_path)`). Textures hash WITH the `.rpak` suffix; named models
  and materials hash WITHOUT it. `StringToGuid` is case-insensitive.
- Material GUIDs hash the RSX export path, which carries a shader-type suffix
  the in-pak name lacks: `world/x/floor_trim_01_rgdp` hashes to the GUID,
  `world/x/floor_trim_01` does not. Key materials on the export path.
- Asset order is load-bearing. Repak assigns segments in first-use order and
  the S21 client faults in pointer relocation if a temp-flagged lump lands
  before the first model's lump. Order models first, then
  `dtbl shdr shds txan txtr txtx matl aseq arig rmap uiia`. The dedi manifests
  are models first, then rigs.
- `$animrigs` / `$sequences` on a model or rig entry write GUID references.
  `aseq` are auto-added from `$sequences`; the dedi manifests do not list
  them. A bare `"0x..."` string in those arrays is reference-only (no auto-add).
- `stgs` entries need their settings layout (`stlt`) present on disk as a
  build reference even though the layout is packed elsewhere; a missing layout
  is a silent Repak exit that leaves a 128-byte pak.
- `dtbl` CSVs need CRLF and a trailing type row; empty tables need one data row.
- Build lists carry `//` comments. Strip them before parsing as JSON
  (`tools/manifest_index.py` does).
- Every S21 client pak needs header bit `0x20` (`headerFlags: 32`). Repak's
  default omits it and the client rejects the pak with `pak load failed`. Any
  `compressLevel > 0` writes zstd, which the S21 client cannot decode. Dedi
  paks are the opposite: zstd level 6.

## 3. Client map port (S25 / S26 / S30 donor)

Order matters. The gates at the end are the difference between a map that
loads and one that fatals on the first unresolved reference.

1. Inventory. `rsx -nogui --list inv/base.csv --listformat csv <map>.rpak`.
   The `type` column is the authoritative `--exporttypes` code.
2. Extract with `-exportfullpaths`, always. Load the map pak plus the donor's
   `common*` paks so shader children resolve, and restrict with
   `--exportpak <map>.rpak` so you get the map's closure, not every loaded
   pak. Materials twice: once plain (path mode) and once with
   `--texturenames guid -exportfullpaths` (real texture GUIDs; this is the
   one the manifest keys on). Without guid mode RSX writes synthetic texture
   paths and every material reference dangles.
3. RSX cannot export `txtx` (no exporter) or raw `uiia` (it writes a decoded
   image). Read those two from the pak pages directly. The uiia asset name is
   doubled (`ui_image/ui_image/0x...rpak`) on purpose; Repak opens
   `assetsDir + _path` verbatim.
4. Models: client models are the raw extract. Do not run rmdlconv on them
   unless the donor is v19+ (then `-sourceversion 19.1 -targetversion 17`).
   A correct v17 model starts `93 00 05 00`; an `IDST` v54 file is the dedi
   format and makes Repak stop at `building pak file` with no error line.
   `.vg` / `.vg_static` / `.phy` are byte-identical between v19.1 and v17.
5. Animations: `R5-AnimConv -i <donor season> -o 21` converts rigs and the
   sequences they own. Rigless sequences are skipped.
6. Shaders and materials: see section 6. Never pack S25+ bytecode as-is.
7. Textures: streamed textures need `$strmMips` / `$optMips` (from the RSX
   json `streamLayout`) and a `$hdrTail`. Derive the tail from the texture's
   own format and dimensions plus a material-slot lookup; do not copy tail
   bytes out of the source pak (its counts can be in the compressed domain
   while Repak writes raw mips). A streamed texture with no tail dies in-map
   in the stream thread.
8. Closure. The manifest built from `base.csv` packs the map's OWN assets.
   Anything the map references that lives in a donor `common*` pak is packed
   by nobody and absent from S21. List the S21 runtime set yourself
   (`common`, `common_mp`, `common_early`, their `(01)` patches,
   `common_flowstate`, `mp_lobby` + perm/temp, `startup`, `ui`), diff the
   map's reference set against it, pull the gap with
   `rsx --exportguids <list>` (the pak that OWNS the asset must be loaded),
   persist the pull list, regenerate, repeat until the gap is 0. Each pulled
   shaderset drags in its own shaders. Skip refs of type `msnp asqd efct
   vers Ptch rson`; Repak strips them and the built asset carries no such
   reference.
9. Same-GUID law. If a `shdr` / `shds` / `matl` GUID already exists in S21,
   the S21 copy IS the asset. Packing donor bytes under it stomps the live
   copy for every map that shares it. Skip those GUIDs (common-resident ones
   resolve at runtime); use the S21 original for map-pak-only ones.
   Census against common AND an S21 map inventory, not common alone.
10. Static props. The BSP `sprp` game lump names models by path; the rpak
    dependency graph never sees them. A name that is in neither the map
    manifest nor the loaded-pak set stays unresolved and crashes at signon.
    The miss is often an S21 other-map pak (district owns
    `construction_plastic_mat_white_01.rmdl`; a map that does not load
    district must pack it). Donor for those is the S21 extract, v17 as-is,
    plus the model's own `matl` + `txtr`.
11. Minimap: the `resource/overviews/<map>.txt` KV rides as a `wrap` in the
    perm pak and the image as a `uiia` at `ui_image/overviews/<map>.rpak`.
    Both, or the minimap is blank with no log line. BSP lump `0x2A` goes in
    the perm wrap pak and `0x2B` in temp; swapping them puts the cubemap
    ambient buffer in the wrong pak.
12. Build: `repak.exe build_list_<map>.json` from the repo root. The map's
    starpak pair is regenerated from scratch on every standalone build, so
    building map B alone overwrites a shared starpak and breaks map A. Either
    co-build in one list or use `streamCache` to append.

Gates before calling a client map done (all must be zero):

- consistency: unresolved references = 0. The engine checks every GUID
  reference at load and errors on the first non-resident one. "No worse
  than the old pak" is not a pass; the check is per reference.
- vertex format: every mesh is a superset of the format mask its vertex
  shader declares (`0x10` COLOR, `0x40` instanced, `0x200` nrm4, same bits
  as the VG `flags` word). The warning for a miss is stripped from the S21
  client, so a clean log proves nothing. Props that are full-bright or
  missing with everything else green are this.
- `blendStates[3]` never blend-enabled: render target 3 of the opaque pass
  is an integer format; a PSO create that blends into it fails silently and
  the null is dereferenced far away in D3D12. Legal masks for map content
  are `0x00 / 0x17 / 0x57`.
- material `unk_CC` nonzero on foliage / cloth / vertex-anim materials
  (retail: foliage 2 or 3, cloth 1). RSX does not export it and Repak zeroes
  it; 0 is invisible or full-bright foliage. Restore after every rebuild.
- material `unk_E8` = 1.0 except `particle/` and `effects/` materials.
- every sprp name resolves (manifest or loaded set); every streamed `txtr`
  has a tail; every client pak header reads `0x0020`.
- `rsx -validateshaders` with the map + perm + temp + `common*` + `mp_lobby`
  + `startup` + `ui` loaded: 0 flagged.

Symptom table:

| symptom | cause |
|---|---|
| `Pak file consistency error` at load | a referenced GUID is in no loaded pak: closure gap, or a skipped-but-referenced `txan` / `dtbl` |
| pak-load AV in pointer relocation, page index reads as `rmdl` ASCII | manifest asset order (models must come first) |
| Repak stops at `building pak file`, no error | a client model is `IDST` v54, or a `stgs` layout file is missing |
| props full-bright, pipeline otherwise green | vertex format contract (VS needs COLOR the mesh lacks) |
| foliage invisible or flickering with the camera still | `unk_CC` = 0, or `c_dof` still read from the camera cbuffer tail |
| one sky or glass layer solid black | blanket opaque dxState stamp hit a translucent material |
| whole map flat, no image-based lighting, every field measures clean | the shader converter never ran; ambient buffer still bound as cbuffer 6 instead of `t72` |
| a prop fully black (opaque) or invisible (alpha-tested) | its material shipped an all-zero `$textures` array |
| signon crash on a static prop | sprp model in neither the map pak nor the loaded set |
| in-map crash in the stream thread | streamed texture with no `$hdrTail` |
| `pak load failed` on a small pak | header flag `0x20` missing, or the pak is zstd |
| minimap blank | missing overviews wrap or overviews uiia |
| another map regressed after this port | same-GUID stomp of a `common*` asset |

## 4. Dedi pak (any S21-format map)

The dedi pak is built from the matching CLIENT map pak of the same name (the
client pak holds the map's full model set), reduced to server types.

1. Extract `mdl_ arig aseq dtbl` with `-exportfullpaths`. Extract `stgs`
   separately with `common.rpak` loaded (the settings layouts live there),
   then prune the exported settings to the map's own (`rsx --list` filtered
   to `stgs`), because loading common exports its 16k settings too. Extract
   `stlt` from common as a build reference.
2. `rmdlconv -v17 <src> <out> -nopause` converts every model to IDST v54 and
   writes the converted `.phy` next to it with `phySize` updated. Take BOTH
   the `.rmdl` and the `.phy` from the converted tree; a raw `.phy` is 16
   bytes short and Repak aborts with `Physics file has a size of N, but the
   model expected N+16`. Do not pass `-autogenbvh`; most BVH-less props are
   decals or foliage and would gain spurious collision.
3. `R5-AnimConv <src> -i 21 -o 3 -outpath <out>`. Point it at the whole
   extract, not just `animrig/`: self-rigged animated models drive their own
   sequence conversion. It writes `animseq/` AND `animseq_derived/`; stage
   both. It can crash on exit after writing complete output; diff the file
   set rather than trusting the exit code.
4. Assemble `s21-to-s3-server/<pak>/`: `mdl/<cat>/<name>.rmdl` + `.phy` from
   the converted tree, `.rson` from the raw extract (carries `rigs:` /
   `seqs:` for the manifest), `animrig/` + `animseq/` from the anim output.
5. Generate the manifest (models first, then rigs; `$animrigs` / `$sequences`
   from the `.rson`), then `repak.exe build_list_<pak>_only.json`.
6. Gate: the built pak's GUID set must be a superset of the pak it replaces
   (set diff, not counts). `rsx --list` auto-loads a `patch_master.rpak`
   sitting next to the listed pak and adds a phantom `Ptch` row; filter rows
   by `file_name` or list from an isolated directory.

Closure rule: the dedi always loads `common`, `common_early`, `common_mp`,
`mp_lobby`, `startup`. An asset the map references must be in one loaded
pak. The converted common paks are lossy relative to the donor's common, so
an asset dropped there has to be pulled into the map pak (district carries
six models from the donor's `common_mp` for this reason). The BSP `sprp` +
entity lumps are the authority for what the dedi must resolve; a missing
model is `Model "..." not found and "mdl/error.rmdl" couldn't be loaded`.

Collision part selector: the converted model's per-class collision part
indices (movement class, bullet class; negative = no collision for that
class) must be copied VERBATIM from the source, negative values included.
About 20% of retail props ship a negative movement slot on purpose
(walk-through decor); forcing them to `0` gives the server phantom collision
and prediction errors. Only auto-generated BVH legitimately reads `(0, 0)`.

Run the heavy converters serially on big paks; rmdlconv has crashed when run
concurrently with a Repak build and an RSX extract.

## 5. BSP and VPK

- Extract the BSP + lumps from the CLIENT paks (`<map>.rpak` +
  `_client_perm` + `_client_temp` loaded together, `--exporttypes wrap`). The
  newer server BSPs use a vis system the S3 engine cannot consume.
- Strip the `.client` suffix from lump filenames before bspconv or disk load.
- `bspconv <map>.bsp` targets v51 (S21 client); `-dedi` targets v47 (server
  lump strip + v121 to v8 brush collision downgrade). `-out <dir>` converts a
  copy; `-pack` writes one monolithic `.bsp`.
- Lump sidecar hex must be LOWERCASE (`.006a.bsp_lump`). The S3 filesystem
  lowercases the path before a case-sensitive VPK lookup; a miss reads the
  rBSP header as the lump and crashes on the first spatial query. RSX writes
  uppercase; bspconv renames, but verify before packing a VPK.
- Disk-loaded maps need lightmap lump `0x69` padded to `rpakSize * 13/12`
  (zero fill) or map load aborts with `Odd LUMP_LIGHTMAP_DATA_REAL_TIME_LIGHTS
  lump size`.
- The `sprp` game-lump version byte is cosmetic; the engine finds the lump
  by id and the static-prop record is the same size in every era.
- revpk re-splits its own command line on spaces, argv[0] included. Run it
  from a space-free exe path with cwd = the VPK dir and relative arguments.
  A new lump must be added to the VPK manifest (`preloadSize 0, loadFlags
  257, textureFlags 0, useCompression 1, deDuplicate 1`) or pack ignores it.
- Wiring a new map is data: playlist `maps` blocks, the map-name allowlist,
  a localization key named for the level, and a `<map>_loadscreen.rpak`.

## 6. Shaders and materials (S25+ donors)

The S21 engine cbuffer contract differs from S25+ even where versions match:

| | S25+ | S21 |
|---|---|---|
| `CBufCommonPerCamera` (cb3) | 928, `c_dof` at 880 | 880 |
| `CBufUberDynamic` (cb1) | 32 | 80, `c_dof` at +16 (`cb1[1..3]`) |
| cubemap samples | `CubemapSample_s` cb6[256] | `g_cubemapSamples` structured `t72`, stride 4 |
| stripped debug permutations | zero-size entries | null pointer with nonzero size (normal) |

The conversion passes (uber 32 to 80, exposure nop, cubemap cb6 to t72,
dither-discard nop, camera RDEF 928 to 880, `c_dof` relocation) rewrite the
port's OWN blobs in place with one DXBC checksum re-sign. Measure a pass by
the RDEF binding it produces (`type 5, bind 72` for the cubemap buffer),
never by a name string. Never transplant retail bytecode onto donor meshes:
the VS input signature is derived from the bytecode and a donor mesh
lacking a stream the retail VS requires crashes input-layout creation.

A header-only child shader (type `9`, no bytecode) is packed with
`$parentShader`; read the parent from the RSX `--depfilepath` adjacency
list rather than the pak bytes.

Material dxState from S25+ ships zero. Arbitrate per material against a
working S21 map, never blanket-stamp: the S21 opaque row is `blendStates`
`0xF0000000` x8, `blendStateMask 4`, `depthStencilFlags 0x17`,
`rasterizerFlags 6`, `unk_E8 1.0`, but a translucent material needs its own
row and retail repeats slot 0 in slot 4 (a name-free check for a stomped
slot 0). S26 v19 models carry gpu-bone records in the opposite field order
from what S21 reads.

The shader conversion toolkit and the per-map manifest generators that
import it are not in this repo; the manifests are their output and the
in-place fix stack must be re-run after any rebuild.

## 7. Small client paks

- Loadscreen: one `uiia` v2 asset, GUID =
  `StringToGuid("ui_image/loadscreens/<map>_widescreen.rpak")` (with the
  suffix). Fully resident (no starpak). `headerFlags: 32, compressLevel: 0`.
- UI images (`ui_sdk`): the client has two fixed tile atlases routed by
  `imgFlags & 3`: class 0 BC1 (4096x9216, 36,864 tiles) and class 1 BC7
  (4096x5120, 20,480 tiles, not growable). Retail `ui.rpak` images are
  protected from eviction; every other pak's images are evicted first, so
  budget BC7 to about a third of the pool and route opaque or binary-alpha
  art to BC1. Tile block order is Morton. Culling unused art beats capping.
- Particles (`efct`): three channels. A GUID not in S21 ships as-is; a GUID
  S21 owns must ship under a synthetic GUID with the same NAME (last
  published name wins; redeclaring a live GUID is a load-order crash); an
  effect whose materials S21 lacks is blocked. Uncompressed, header align 8,
  stride 24. An unresolved child GUID is left in place silently and the
  particle draws untextured forever; gate for zero unresolved before packing.
- Custom models into `common_flowstate`: rmdlconv `-sourceversion 8|49|12.2
  -targetversion 17`; bind a stock S21 animrig via `$animrigs` (do not pack
  a copy of a common rig); the placeholder sequence must carry
  `paramindex -1,-1`, `activity 0xFFFF` and the virtual-model flag, or the
  model T-poses.
- Navmesh: Respawn ships `small` and `med_short` for every S21 map as `wrap`
  assets in `<map>_server_temp.rpak` (`maps/navmesh/<map>/<hull>.nm`, v9).
  The S3 dedi reads v8; the converter swaps `dtPolyAreas` (v9 `JUMP=0,
  GROUND=1`, v8 the mirror) or every polygon arrives tagged JUMP. Bake the
  three wide hulls with recast from the dedi v47 BSP.

## 8. RSX habits

- Flags first, input files last; the first non-dash argument ends flag
  parsing.
- `-export` writes files; `-nogui` alone loads and exits.
- Streamed payloads need the `.starpak` beside the `.rpak`; the base map pak
  often holds little and the lumps live in perm/temp plus their starpaks.
- `-decompresspak` writes `<pak>.dec.rpak` for offset-based readers, but a
  decompressed Respawn pak segment-aligns its pages while a Repak pak packs
  them contiguously; prefer RSX's per-asset json and `--depfilepath` over
  re-reading pak bytes.
- RSX names exported hash-only files unpadded (`0x101F...`, not `0x0101...`).
- Headless RSX takes a named mutex; concurrent invocations serialise. It
  re-parses `common.rpak` on every invocation, so chain extractions.
- Two distinct materials can share a filename stem; index by full relative
  path, never by stem. RSX's guid-mode material export is correct; if
  references look like garbage the bug is downstream in the generator.

## 9. Publishing rules for this repo

- Recipes only. Never commit `.rpak`, `.starpak`, `.dds`, `.rmdl`, `.rseq`,
  `.rrig`, `.phy`, `.vg`, `.msw`, `.bsp*`, `.vpk` or any extracted asset;
  the `.gitignore` refuses them and that policy is deliberate.
- Only content from retail seasons that have shipped. Nothing from
  playtest or unreleased builds, by name or by asset.
- Paths inside manifests are repo-root relative. Never commit an absolute
  local path.
- No reverse-engineering detail: no engine addresses, symbol names, struct
  layouts or debug-database references in any file or commit message.
- Regenerate `docs/MANIFESTS.md` (`py -3 tools/manifest_index.py`) whenever a
  manifest or build list changes.
- Commit messages describe what the tree is.
