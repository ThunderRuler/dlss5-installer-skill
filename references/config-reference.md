# Config reference

## `[RenoDX.DLSS5]` in `reshade.ini` — the NR add-on

Key names extracted from `renodx-dlss5.addon64` (v4.1.5). All are also exposed in the
add-on's overlay panel.

| Key | Notes |
|---|---|
| `NREnableUpscaling` | Master switch for NR's own upscaling. **Keep at 0** — blocked on current runtime builds (`0xBAD00005`). |
| `NRIntensity` | Overall strength of the neural enhancement. Higher = stronger, and more risk of artefacts. |
| `NRPreset` | **Currently inert** — the shipped DLL contains a single network, so there is nothing to switch between. |
| `NRStyle` | Also currently inert, same reason. |
| `NRColorStrength` | Colour-domain strength. |
| `NRTransferStrength` | Transfer-function strength. |
| `NRPaperWhiteScale` | HDR paper-white scaling. |
| `NRComputeScalingRatioC` | Scaling-ratio computation — **relevant to guide/colour resolution mismatch**. |
| `NRMVecScaleX` / `NRMVecScaleY` | Motion-vector scale — also relevant to guide mismatch. |
| `NRDepthMode` | Depth interpretation mode. |
| `NRLocalStructure` | Local structure preservation. |
| `NRLocalTone` | Local tone handling. |
| `NRSkinStructure` | Skin-specific structure handling. |
| `NRUICorrection` | UI correction pass. |
| `NRAutoMask` | Automatic masking. |

Runtime log line reports the active set, e.g.:
```
DLSS5 active settings: upscaling=OFF intensity=1.000000 color_strength=1.000000
transfer=1.000000 paper_white=1.000000 preset=0 style=0 enabled=ON
```

Hotkeys: **F6** NR toggle, **F5** screenshot.

`EnableHooks=2` in the load banner means "NGX hooks only, Streamline modules left unpatched".

## `dlss5-feed.cfg` — DLSS5-Feeder

Auto-created with working defaults, next to the game exe.

| Key | Default | Values | Meaning |
|---|---|---|---|
| `enabled` | 1 | 0/1 | 0 disables all processing |
| `mode` | 2 | 0/1/2 | 0 = inert · 1 = transport test, no NGX · 2 = full DLSS path |
| `hdr` | -1 | -1/0/1 | -1 = autodetect (FP16 / R11G11B10 ⇒ HDR) · 0 = force SDR · 1 = force HDR |
| `depth_inverted` | -1 | -1/0/1 | -1 = follow ReShade · 0/1 force orientation |
| `flags` | -1 | int | Raw `DLSS.Feature.Create.Flags` override |
| `reset_every` | 0 | 0/1 | Rebuild NGX feature every frame — diagnostic, kills temporal history |
| `warmup_rebuild` | 180 | int | Re-create feature once after N delivered frames. **Workaround for add-on STANDBY/FAILED on first create.** |
| `rebuild` | 0 | int | Change the value to trigger a single manual rebuild |
| `log_frames` | 3 | int | Initial frames logged in detail |
| `create_delay` | 60 | int | Frames to delay feature creation after runtime init — the DLSS 5 add-on arms its hooks asynchronously. 0 disables. |
| `mv_scale_x` | 1.0 | float | Extra motion-vector X multiplier |
| `mv_scale_y` | 1.0 | float | Extra motion-vector Y multiplier |
| `host_window` | 1 | 0/1 | **32-bit only** — show/hide the helper window |

Motion-vector sign lives in the `DLSS5_Feed.fx` UI (**MV_SIGN**), not the cfg.

## `dlss5-dx11-bridge.cfg` — DX11 bridge

Re-read at runtime; changes apply immediately or on the next NGX feature trigger.

| Key | Default | Values | Meaning |
|---|---|---|---|
| `stage` | 3 | 0–3 | 0 = inert · 1 = input copies only · 2 = + depth conversion · 3 = full. **Set 0 to prove the bridge is not the culprit.** |
| `mode` | 2 | 0–2 | 0 = read-only · 1 = transport without DLSS · 2 = full DLSS pipeline |
| `skip_game` | 1 | 0/1 | Suppress the game's own DLSS evaluate (its result is overwritten anyway) |
| `flags` | 107 | int | `DLSS.Feature.Create.Flags` for the bridge's feature |
| `subrects` | 1 | 0/1 | Fallback output subrect when the game doesn't set one |
| `reset_every` | 0 | 0/1 | Force NGX Reset every frame — diagnostic |
| `pixels` | 0 | 0/1 | Read pixels back to CPU for debugging (**stalls the GPU**) |
| `dred` | 1 | 0/1 | Device Removed Extended Data for crash diagnosis |
| `skip_exe` | 1 | 0/1 | Skip processing for certain executables |

## `dlss5-d3d12-fix.cfg` — mip fix (obsolete ≥ NR v3.3.5)

| Key | Default | Purpose |
|---|---|---|
| `fix` | 1 | Enable/disable substitution |
| `sub_output` | 1 | Replace output texture |
| `sub_depth` | 1 | Replace depth texture |
| `preload_output` | 1 | Pre-seed with game data |
| `copyback` | 1 | Return results to the game |
| `nr_quality` | -2 | Quality preset; -2 = unchanged |

## Relevant `reshade.ini` sections

```ini
[ADDON]
DisabledAddons=Generic Depth,Effect Runtime Sync   ; NOTE: Feeder REQUIRES Generic Depth

[GENERAL]
EffectSearchPaths=.\reshade-shaders\Shaders\**
TextureSearchPaths=.\reshade-shaders\Textures\**
PreprocessorDefinitions=RESHADE_DEPTH_INPUT_IS_REVERSED=1,...

[INPUT]
KeyOverlay=36,0,0,0        ; 36 = Home

[RenoDX.DLSS5]
NREnableUpscaling=0
```
