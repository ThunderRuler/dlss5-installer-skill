# Components

Every moving part, what it does, and what it requires.

## Core

### ReShade 6.8+ — **Addon build**
The host everything else plugs into. Must be the *full/unsigned* build with
"Enable loading of add-ons" — the standard build cannot load `.addon64` files.

- Installed as a **proxy DLL named after something the game imports**: `dxgi.dll` for
  D3D11/D3D12, `d3d9.dll` for D3D9, etc.
- Searches for add-ons **next to the DLL that loaded it**, and matches bitness:
  `.addon64` in a 64-bit process, `.addon`/`.addon32` in a 32-bit process.
- Writes `ReShade.log` next to the exe. That log is the primary diagnostic for everything
  in this folder.

### `nvngx_dlssnr.dll` — the neural rendering runtime
Version **310.8.0**. The leaked NVIDIA DLL that does the actual work.

**Must pass Authenticode verification.** A `HashMismatch` here is a top cause of
STANDBY/FAILED. Check with:

```powershell
Get-AuthenticodeSignature "path\to\nvngx_dlssnr.dll" | Select-Object Status
```

Known-good SHA256 on this machine: `E16BCF15E16E13F527491CDF7845B2FE6521A738D8F7C9C721866A8496E1FC8E`
Known-**bad** (HashMismatch) copy seen in the wild: `4B8D19BC3EFF58A084F5ECA7489C921501C203450169FB82FF4F649A4482BA05`

### `nvngx_dlss.dll` — super resolution
Standard DLSS DLL. Needed alongside the NR runtime. Any recent one works; DLSS Swapper
is the clean source.

### `renodx-dlss5.addon64` — DLSS 5 Neural Rendering
The add-on that hooks NGX and inserts the neural pass. **D3D12 only.** In a D3D11 game it
needs the bridge; in a game without DLSS it needs the Feeder.

- Current build seen here: v0.2026.828.2110 / *RenoDX DLSS5 Generic v4.1.5*
- Hotkeys: **F6** toggles neural rendering, **F5** screenshot
- Settings live under `[RenoDX.DLSS5]` in `reshade.ini` and in its overlay panel
- Snapshot URL: `https://github.com/clshortfuse/renodx/releases/download/snapshot/renodx-dlss5.addon64`

## Bridges and feeders

### `dlss5-dx11-bridge.addon64` (NIGos)
**For D3D11 games that already have DLSS.** Intercepts the game's
`NVSDK_NGX_D3D11_EvaluateFeature_C`, forwards it untouched, then reproduces the same DLSS
contract on a second NGX session running on its own D3D12 device.

- Requires: D3D11 game with DLSS, a GPU/driver supporting D3D12, and `ID3D11Device5` for
  cross-API shared fences
- Drop the `.addon64` next to ReShade; it writes `dlss5-dx11-bridge.cfg` with working defaults
- Inert in a D3D12 game — harmless but pointless
- Writes `dlss5-dx11-bridge.log` (verbose logging is always on)
- Fails safe: if anything goes wrong it disables itself and the game renders normally

### `dlss5-feed.addon64` (DLSS5-Feeder, jlrouzies-fr)
**For D3D11/D3D12 games with no DLSS at all.** Synthesises a DLAA contract from ReShade's
depth buffer plus iMMERSE LaunchPad optical-flow motion vectors, and runs it through the
NR add-on on a private D3D12 device.

Needs a whole chain, not just the addon:
- ReShade **Generic Depth** add-on enabled, with scene depth correctly picked
- **iMMERSE LaunchPad** (`MartysMods_LAUNCHPAD.fx` + `Shaders\MartysMods\` + `iMMERSE_bluenoise_opt.png`)
- `DLSS5_Feed.fx` in `reshade-shaders\Shaders\`
- Technique order in the ReShade list: **LaunchPad → DLSS5_Feed** (Feed must be below)
- Game MSAA/SSAA **off**

Supports 32-bit games via a `host64\` helper process. Writes `dlss5-feed.log`.

**DLAA only** — render res = output res = backbuffer. No performance gain; jittered
upscaling is future work.

Do **not** run the Feeder and the DX11 bridge together on the same game.

## Fixes

### `dlss5-d3d12-fix.addon64` (NIGos) — *usually obsolete*
For D3D12 games whose DLSS output carries a mip chain, which the NR add-on requires to be
single-mip and silently refuses.

**Superseded from DLSS 5 Neural Rendering v3.3.5**, which handles mipped outputs itself.
Upgrade the NR add-on rather than installing this. Only relevant if stuck on an old build.

Symptoms it addressed: `DLSSNR: STANDBY/FAILED`, thousands of NGX attempts with zero
successful frames, guides never resolving (shown as 0x0), repeated `0xBAD00005`.

### `dltb-dlss5-fix` (markitzeroo) — the per-game fix pattern
Dying Light: The Beast allocates its DLSS output as `R16G16B16A16_TYPELESS`. The NR add-on
needs a *typed* resource to sample in shaders and write through typed UAVs, so it rejects
it: `"DLSS output format 9 is not a supported typed codec format"`.

The fix upgrades that one resource to typed `R16G16B16A16_FLOAT` at creation time — same
format family and bit layout, only the view typing changes. Built on RenoDX's
`utils::resource::upgrade` / `SetUpgradeInfos()`.

**This is the template for per-game work.** The technique generalises, but the selection
filters (aspect ratio match to backbuffer, min 640×360, unordered-access usage) must be
retuned per game or you crash on unrelated texture views.

### `dlssnr-signature-repair` (kayle2203)
Fixes exactly one thing: `nvngx_dlssnr.dll` with an invalid NVIDIA signature, which
surfaces as STANDBY/FAILED and `feature 18 create failed with 0xBAD00002`.

Replaces the corrupted DLL with a verified copy **from another game you already own** —
no downloads. Verifies filename, version `310.8.0.0`, SHA-256 and Authenticode before
swapping, and takes a timestamped backup.

You often don't need the tool: if you have any valid copy locally (RHI caches one under
`%LOCALAPPDATA%\RHI\DLSS-NR\310.8.0\`), just copy it over.

## Management

### RHI (RankFTW) — "ReShade HDR Installer"
Library-wide installer. Detects games across eight storefronts and installs ReShade,
RenoDX, OptiScaler, Streamline, DLSS DLL swaps, frame limiters and add-ons.

State lives in `%LOCALAPPDATA%\RHI\`:

| Path | Contents |
|---|---|
| `installed.json` | per-game RenoDX addon |
| `aux_installed.json` | ReShade / OptiScaler, **and the DLL name each was installed as** |
| `addon_deployments.json` | which addons are deployed per game |
| `game_api_cache.json` | **detected render API per game** — the fastest way to triage targets |
| `logs\session_*.txt` | every deploy, rename and delete it performed |
| `DLSS-NR\310.8.0\` | cached (valid) `nvngx_dlssnr.dll` |
| `reshade\` | ReShade Addon build payload + installer |

Gotchas learned the hard way:
- The per-game **"RS DLL" dropdown** decides the proxy name. Set it to `dxgi.dll` for
  D3D11/D3D12 games. `ReShade64.dll` only works when OptiScaler is present as `dxgi.dll`
  **and** `LoadReshade = true` in `OptiScaler.ini` (default `auto` = false).
- Drag-and-dropping `renodx-dlss5.addon64` onto a game **deletes** that game's existing
  per-game RenoDX addon (e.g. the HDR fix). One RenoDX addon per game.
- It has a "global addon set" that redeploys addons on refresh — removing files by hand
  doesn't stick.

### DLSSTweaks (emoose) — **often bypassed, check before trusting**
Forces DLAA / custom DLSS ratios via an `nvngx.dll` wrapper in the game folder.

**It is silently bypassed in Streamline games.** When `sl.interposer.dll` is present, the
game loads `_nvngx.dll` straight from the driver store and never touches the game-folder
wrapper. Confirm it is actually running by checking that `dlsstweaks.log` was written on
your last launch — a stale timestamp means it never loaded.

Also note `ForceDLAA=true` is disabled whenever the `[DLSSQualityLevels]` section is
enabled; its own log says so: `ForceDLAA: true (overridden by DLSSQualityLevels section)`.

Prefer **NVIDIA App → DLSS Override → DLAA** for forcing native render resolution.
