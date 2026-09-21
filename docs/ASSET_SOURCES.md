# Asset sources

Where each pak's inputs come from and what turned them into what the target
engine loads. "S21" is the bridge client build; "S3" is the dedicated server
build. Donor builds are named by the retail season they ship in.

## Tool pins

| tool | repo | commit |
|---|---|---|
| RePak | https://github.com/R5Flowstate/RePak | `bfb69e336` |
| rsx | https://github.com/R5Flowstate/rsx | `ab3b7d0ba` |
| rmdlconv | https://github.com/R5Flowstate/rmdlconv | `00f1bca86` |
| R5-AnimConv | https://github.com/R5Flowstate/R5-AnimConv | `ea0f88ec2` |
| bspconv | https://github.com/R5Flowstate/bspconv | `e0661f5a3` |

## S21 client paks (`paks/Win64/`)

| pak | manifest | donor | conversion |
|---|---|---|---|
| `mp_rr_district_mu1` + `_client_perm` + `_client_temp` | `s21-maps/mp_rr_district_mu1/` | S26 | rmdlconv `mdl_` v19 to v17; R5-AnimConv for `aseq` / `arig`; shader cbuffer conversion to the S21 layout; BSP via bspconv v52 to v51 |
| `mp_rr_district` (base rebuild) | `s21-maps/mp_rr_district/` | S21 (in-place rebuild) | none |
| `mp_rr_aqueduct`, `mp_rr_arena_composite`, `mp_rr_arena_skygarden` (+ perm / temp) | `s21-maps/<map>/` | S25 arenas | shader cbuffer conversion to the S21 layout; material dxState repair; `txtx` / `uiia` raw re-extract |
| `common_flowstate` | `s21-flowstate/common_flowstate/common_flowstate_s21.json` | authored (custom models, materials) | rmdlconv from the source model version; textures authored |
| `sdk_s30` + `pc_sdk_s30.starpak` pair | `s30_unified/sdk_s30.json` | S30 (effects, animations, settings) | `efct` operator / payload remap to S21; R5-AnimConv; `stgs` as-is |
| `sdk_axle`, `root_lgnd_skins_overdrive_classic` | `s30_axle/paks_out/` | S30 | rmdlconv v19 to v17; R5-AnimConv; shader sets from S21 common |
| `gcard_frame_overdrive_*` | `s30_overdrive/gcard_frames/` | S30 | `uiia` re-encoded (BC7 tiles) |
| `ui_sdk` | `s21-ui/ui_sdk/ui_sdk.json` | authored UI art | per-image `uiia` (BC1 / BC7 by alpha) |

Each map pak names its own `pc_all_<map>.starpak` + `.opt.starpak` pair in its
manifest. Rebuilding a map standalone rewrites its pair; `streamCache` in a
build list appends to an existing pair instead.

## S3 dedi paks (`paks/Win64_server/`)

All from `s21-to-s3-server/<pak>.json`, built with a `dedi: true` build
list. Donor is the matching S21 client pak of the same name, reduced to the
server-only asset types: `mdl_` v10 (rmdlconv, physics header rewritten),
`aseq` v7 / `arig` v4 (R5-AnimConv), `stgs`, `stlt`, `dtbl`, `txls`, `Ptch`.
No starpaks. zstd compressed.

| pak | notes |
|---|---|
| `common`, `common_early`, `common_mp`, `patch_master`, `mp_lobby`, `root_lgnd_skins` | core set; `common_hybrid` is a variant of `common` differing by two models |
| `mp_rr_district`, `mp_rr_district_mu1`, `mp_rr_thunderdome`, `mp_rr_canyonlands_staging_mu1`, `mp_rr_canyonlands_hu`, `mp_rr_desertlands_hu`, `mp_rr_olympus_mu2`, `mp_rr_divided_moon_mu1`, `mp_rr_tropic_island_mu2`, `mp_rr_party_crasher` | BR / firing-range maps |
| `mp_rr_aqueduct`, `mp_rr_arena_composite`, `mp_rr_arena_skygarden`, `mp_rr_arena_habitat`, `mp_rr_arena_phase_runner`, `mp_rr_freedm_skulltown` | arena / FFA maps |
| `sdk_axle_dedi`, `sdk_s30_dedi`, `common_flowstate_dedi` | SDK content twins of the client paks |
