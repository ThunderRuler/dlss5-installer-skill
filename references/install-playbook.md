# Install playbook

## Scenario A — D3D12 game with native DLSS (simplest)

Example: Death Stranding Director's Cut.

Files next to the game exe:
```
game.exe
dxgi.dll                  <- ReShade 6.8+ Addon build, renamed
renodx-dlss5.addon64
nvngx_dlssnr.dll          <- must be Authenticode Valid
nvngx_dlss.dll
```

1. Install ReShade 6.8+ **Addon build** against the game exe, choosing Direct3D 10/11/12,
   with "Enable loading of add-ons" ticked. Or in RHI, set the game's **RS DLL** dropdown
   to `dxgi.dll`.
2. Drop `renodx-dlss5.addon64` next to the exe.
3. Ensure `nvngx_dlssnr.dll` and `nvngx_dlss.dll` are present.
4. Launch. Confirm in `ReShade.log`:
   ```
   Registered add-on "DLSS 5 Neural Rendering" ...
   inline feature 18 evaluation succeeded (...)
   ```
5. **Home** for the overlay → Add-ons → DLSS 5 Neural Rendering. **F6** toggles the effect.

**Force DLAA.** The neural pass runs after upscaling, and mismatched render/output
resolution causes partial-frame artefacts (see [troubleshooting](troubleshooting.md)).
Use **NVIDIA App → Graphics → *game* → DLSS Override – Super Resolution → DLAA**, and leave
DLSS *enabled* in the game — the add-on hooks the game's own NGX call, so turning DLSS off
kills neural rendering entirely.

## Scenario B — D3D11 64-bit game with DLSS (native or modded)

Everything from Scenario A, plus:
```
dlss5-dx11-bridge.addon64
```
Drop it next to ReShade. It writes `dlss5-dx11-bridge.cfg` on first run with defaults that
need no adjustment. Enable the NR toggle in the add-on's overlay panel or in `reshade.ini`.

Diagnostics land in `dlss5-dx11-bridge.log`: the contract read from the game, which
resource-sharing directions the driver accepted, the result of every NGX call, and timing
every 600 frames.

To prove the bridge is or isn't at fault, set `stage=0`. If the problem persists, it isn't
the bridge.

## Scenario C — D3D11/D3D12 64-bit game with NO DLSS (Feeder)

```
game.exe
dxgi.dll
dlss5-feed.addon64
renodx-dlss5.addon64
nvngx_dlssnr.dll
nvngx_dlss.dll
reshade-shaders\
  Shaders\
    DLSS5_Feed.fx
    MartysMods_LAUNCHPAD.fx
    MartysMods\            (whole folder)
  Textures\
    iMMERSE_bluenoise_opt.png
```

1. ReShade 6.8+ Addon build as `dxgi.dll`.
2. `dlss5-feed.addon64` next to the exe, `DLSS5_Feed.fx` into `reshade-shaders\Shaders\`.
3. iMMERSE LaunchPad from https://github.com/martymcmodding/iMMERSE — copy
   `MartysMods_LAUNCHPAD.fx`, the `Shaders\MartysMods\` folder, and
   `Textures\iMMERSE_bluenoise_opt.png`.
4. `renodx-dlss5.addon64` + both nvngx DLLs next to the exe.
5. In game (Home):
   - enable **Generic Depth** and pick the correct scene depth
   - enable **MartysMods_Launchpad**
   - enable **DLSS 5 Feed** — it must sit **below** LaunchPad in the technique list
   - enable neural rendering in the DLSS 5 panel
   - turn the game's **MSAA/SSAA off**

Pipeline: `LaunchPad → DLSS5_Feed → [private D3D12 device: DLSS DLAA + neural] → written
back → remaining effects → present`.

Verify in `dlss5-feed.log`: `feature ready … DLAA` and `frame N delivered`.

## Scenario D — 32-bit D3D11 game (Feeder, beta)

```
game.exe                   (32-bit)
dxgi.dll                   (32-bit ReShade)
dlss5-feed.addon32
DLSS5_Feed.fx  -> reshade-shaders\Shaders\
reshade-shaders\           (LaunchPad files, as Scenario C)
host64\
  dlss5-feed-host64.exe
  dxgi.dll                 (64-bit ReShade)
  renodx-dlss5.addon64
  nvngx_dlssnr.dll
  nvngx_dlss.dll
```

The first fed frame spawns `host64\dlss5-feed-host64.exe` in a small window titled
"DLSS 5 Feed host". **Press Home in that window** to reach the DLSS 5 panel — not in the
game. Once tuned, set `host_window=0` in `dlss5-feed.cfg` to hide it.

Logs are split: `dlss5-feed.log` (32-bit game side), `host64\dlss5-feed-host.log` (helper),
`host64\ReShade.log` (where the DLSS 5 add-on reports appear).

If the helper dies the pipe breaks, the addon disables itself, and the game keeps rendering.

## Post-install checklist

- [ ] `ReShade.log` exists and names the right DLL and bitness
- [ ] `Registered add-on "DLSS 5 Neural Rendering"` present
- [ ] `nvngx_dlssnr.dll` signature is **Valid**, not HashMismatch
- [ ] `inline feature 18 evaluation succeeded` appears
- [ ] Render resolution == output resolution (DLAA), so guides match colour
- [ ] Only the add-ons this game actually needs are deployed
