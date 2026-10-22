# Config reference

**Updated 2026-09-29.** Set things in each tool's overlay panel where possible; if you edit
an ini by hand, do it with the game **closed** (ReShade rewrites `ReShade.ini` on exit).

## `[RenoDX.DLSS5]` in `ReShade.ini` — the RenoDX DLSS5 addon

Key names below are from the v8.5.0-rc10 source (a third-party copy,
`PEQHUB/RenoDX-DLSS5-Generic`, branch `v8.5dev`, a development snapshot — treat as
indicative, confirm in the addon's panel). v8
carries a config schema marker: an old v4.x config is migrated automatically and
`ReShade.ini` is backed up first.

**Placement and passes**

| Key | Notes |
|---|---|
| `NRHookPoint` | **0** = after upscaling (default, needs DLAA to avoid partial-frame artefacts) · **1** = before upscaling (NR at render resolution — cheaper) · **2** = on the final Present frame, UI included, before ReShade effects. Replaces the old `NRPreUpscale` (read only for migration). |
| `NRPasses` / `NRChainedHistory` | Multi-pass NR. Start with 1 pass. |
| `NRPass2..4*` | Per-pass overrides (Style, Intensity, Structure, Skin, LocalTone, AutoMask, UICorrection, Transfer, Color, FollowPass1). |
| `NRResolutionScale` / `NRFollowInputRes` | Model working resolution. |
| `NREnableUpscaling` | Older builds' switch for NR's own upscaling — **keep 0** where present. |

**Look**

| Key | Notes |
|---|---|
| `NRIntensity` | Overall strength. Higher = stronger, more artefact risk. |
| `NRPreset` / `NRStyle` | Model hints. On the leaked 310.8 runtime a changed hint is **not** proof of a changed model — the OptiScaler fork labels these "effect unverified". `NRStyle=2` has crashed under Smooth Motion. |
| `NRColorStrength`, `NRTransferMode` / `NRTransferStrength` | Colour-domain / transfer-function strength. |
| `NRLocalStructure`, `NRLocalTone`, `NRSkinStructure` | Structure, tone, skin handling. |
| `NRLook*` (Mode, Strength, Detail, Tone, Halo, Stabilize…) | Post-look stack added in v8. |
| `NRUICorrection`, `NRAutoMask` | UI correction / automatic masking — **both off by default**; the first things to try for HUD ghosting. |
| `NRCodecMode` (2 = Auto), `NRNormGovernor` (2 = Stable) | Leave at defaults. |

**Present hook (`NRHookPoint=2`) only:** `NRPresentGuides`, `NRPresentEncoding`,
`NRPresentWhiteNits`, `NRPresentWhiteSource`, `NRPresentUICorrection`,
`NRPresentFrames`, `NRPresentFgSource`, `NRPresentPreFg`, `NRPresentMVecUnits`,
`NRAutoFgFallback`.

**Other:** `NRToggleKey` (default F6), `NRScreenshot*`, `NRVkStateRestore`,
`NRDisableRelease`. Diagnostics: `NRGpuTimers`, `NRCostMeter`, `NRCrashDump`,
`NRReportHost` (the upload button — third-party host). Debug builds write
`RenoDX-DLSS5-debug.log`.

`NRMVecScaleX` / `NRMVecScaleY` (motion-vector scale) are still present in v8 and relevant
to guide/colour resolution mismatch. Older (v4.x) builds also exposed
`NRComputeScalingRatioC`, which is gone in v8.

Runtime log reports the active set, e.g.:
```
DLSS5 active settings: upscaling=OFF intensity=1.000000 color_strength=1.000000
transfer=1.000000 paper_white=1.000000 preset=0 style=0 enabled=ON
```

`EnableHooks=2` in the load banner means "NGX hooks only, Streamline modules left unpatched".

## `dlss5-feed.cfg` — DLSS5-Feeder v1.17.0

Auto-created next to the exe, re-read while the game runs, also editable from the overlay's
Add-ons → DLSS 5 Feed page.

**Core**

| Key | Default | Meaning |
|---|---|---|
| `enabled` | 1 | 0 = fully off (since 0.13; on Vulkan re-enabling needs a restart) |
| `mode` | 2 | 0 inert · 1 transport test, no NGX · 2 full DLSS path |
| `hdr` | -1 | -1 autodetect (FP16 / R11G11B10 ⇒ HDR) · 0 force SDR · 1 force HDR |
| `hdr_bridge` / `hdr_paper_white` | -1 / 203 | HDR10 bridge (all 64-bit paths) and its paper white |
| `depth_inverted` | -1 | -1 follow ReShade · 0/1 force |
| `flags` | -1 | raw `DLSS.Feature.Create.Flags` override |
| `preset` | 0 | DLSS SR preset: 5/6 = CNN E/F, 10/11 = J/K |
| `mv_scale_x` / `mv_scale_y` | 1.0 | extra motion-vector multipliers |
| `native_dlss_ok` | 0 | 1 = let the D3D12 same-device path run in a game that loaded its own DLSS (it should normally be on Scenario A instead) |

**Work resolution (D3D11 only)** — a cost knob, not upscaling; lower values look blurry.

| Key | Default | Meaning |
|---|---|---|
| `work_resolution` | 100 | 50–100 % |
| `work_upscale` | 0 | 0 bilinear · 1 FSR 1 EASU+RCAS · 2 synthetic-jitter DLSS SR (measured: saves nothing, shimmers) |
| `work_sharpness` | 0.3 | RCAS sharpness |
| `jitter_sign` / `jitter_phases` | 1 / 0 | for `work_upscale=2` |

For a sharp picture at lower cost, keep 100% and lower the **consumer's** model resolution
instead (OptiScaler `WorkingScale`, RenoDX `NRResolutionScale`).

**Feature lifecycle and stability**

| Key | Default | Meaning |
|---|---|---|
| `create_delay` | 60 | frames to wait before creating the feature (consumer arms asynchronously) |
| `warmup_rebuild` | 180 | re-create once after N frames — workaround for STANDBY/FAILED on first create (RenoDX consumer only) |
| `rebuild` | 0 | change the value to trigger one manual rebuild |
| `reset_every` | 0 | rebuild every frame — diagnostic, kills temporal history |
| `settle_evals` | 0 | diagnostic (0–8 extra evaluates of the same frame). Proved the post-stop "settling" is *not* temporal history; only costs frame time — leave at 0 |
| `hold_strength` / `hold_tolerance` | 0 / 0.04 | **output stabiliser** (1.17) — damps frame-to-frame flicker |
| `gpu_timeout_ms` | 2000 | GPU wait timeout |

**Logging:** `log_frames=3`, `stall_log_ms=50` (use to diagnose stutter).

**Vulkan 64-bit:** `vk_present_sync=1`, `passthrough=0`, `buffer_home=0`, `sync_home=0`.

**32-bit only:** `host_window=0` — 0 helper window hidden *behind* the game (enables the
cast panel; auto-switches to no-window if the game is exclusive fullscreen), 1 its own
visible window (press Home there), 2 no window at all, 3 like 0 but never auto-hides (for
dgVoodoo, whose swapchain falsely reports fullscreen). `async_home=1`, `host_gpu_priority=0`, `hold_input=0`, cast panel `cast_key`,
`cast_mods`, `cast_scale=100`, `cast_mode=0`, `cast_anchor=1`.

**Wine/Proton:** `wine_fence_import=0`.

Helper-window size and DRED live in `host64\ReShade.ini` under `[DLSS5Host]`
(`WindowWidth`, `WindowHeight`, `Dred=1`), not in this cfg.

Motion-vector sign/scale **uniforms** (`MV_SIGN`, `MV_SCALE`) and the geometry/mask knobs
live in the `DLSS5_Feed.fx` UI and are saved to the **preset** — see `motion-vectors.md`.

## `dlss5-bridge.cfg` — NIGos bridge v1.4.x

Renamed from `dlss5-dx11-bridge.cfg` (the old name is still read if no new file exists).
Re-read about once a second. A cfg from a different version resets to defaults unless its
first line is `# dlss5-bridge keep`.

| Key | Default | Meaning |
|---|---|---|
| `source` | auto | auto · mirror · synth · off |
| `vk_mirror` | 1 | Vulkan route for Vulkan games with DLSS |
| `synth` / `synth_after` | 0 / 0 | substitute path for games without DLSS (Optical Flow) — prefer the Feeder |
| `ofa_grid` / `ofa_perf` / `mv_sign_x` / `mv_sign_y` | 2 / 20 / 0 / 0 | substitute-path tuning |
| `stage` | 3 | 0 inert · 1 input copies · 2 + depth conversion · 3 full. **Set 0 to prove the bridge isn't the culprit.** |
| `mode` | 2 | 0 read-only · 1 transport without DLSS · 2 full |
| `skip_game` | 1 | suppress the game's own DLSS evaluate |
| `flags` | -1 | `-1`/`107` = use the game's; `108` forces; **RDR2 needs `75`** |
| `subrects` | 1 | fallback output subrect |
| `shape` | 0 | `1` fixed `0xBAD00005` in PSO2 |
| `ngx_loader` | 0 | workaround for drivers 32.0.16.1664/1686 crashing with dlssnr 310.8.0 |
| `unwrap` / `unwrap_list` | 1 / 0 | set `unwrap=0` on Linux/Proton substitute setups |
| `reset_every`, `pixels`, `dred`=1, `skip_exe`=1, `probe`, `stall_test`, `hash_out`=1 | | diagnostics (`pixels` stalls the GPU) | *(diagnostic keys — no functional default; no change is required)

`drain` was removed in v1.4.0.

## `OptiScaler.ini` — DLSS-NR fork (Scenario E)

`[DlssNr]`: `Enabled`, `RunBeforeSR` (NR before upscaling), `FinishedPicture` (native D3D12
only; keep **false** when fed by the Feeder), `Passes` (1–3; `UnlockPasses` for more),
`WorkingScale` (model resolution), `TransferStrength`, `ColourStrength`,
`WhitePointScale`, `MaxRatio`, `Precision`, `SkinProtection`, `Pass2*`/`Pass3*`,
`Preset`/`Style` (effect unverified), `DebugView`, `ToggleKey`.

Also: `[Upscalers] Dx12Upscaler=dlss` / `Dx11Upscaler=dlss_12`,
`[ProcessFilter] TargetProcessName=auto`, and for a missing menu
`[Log] LogToFile=true, LogLevel=2`, `[Hotfix] ManualInputPolling=true`. Dagherbou's
original build additionally needs `ScanExposure=false` under the Feeder.

## `dlss5-d3d12-fix.cfg` — mip fix (obsolete ≥ NR v3.3.5)

`fix`, `sub_output`, `sub_depth`, `preload_output`, `copyback`, `nr_quality` (-2 = unchanged).

## Relevant `ReShade.ini` sections

```ini
[ADDON]
AddonPath=.\                               ; required for Vulkan (layer) installs
DisabledAddons=Effect Runtime Sync         ; the Feeder REQUIRES Generic Depth enabled

[GENERAL]
EffectSearchPaths=.\reshade-shaders\Shaders\**
TextureSearchPaths=.\reshade-shaders\Textures\**
PreprocessorDefinitions=DLSS5_MV_PROVIDER=3,RESHADE_DEPTH_INPUT_IS_REVERSED=1

[INPUT]
KeyOverlay=36,0,0,0        ; 36 = Home

[RenoDX.DLSS5]
NRHookPoint=0
```
