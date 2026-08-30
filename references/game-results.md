# Game log

> Community results log, seeded from the author's test machine (RTX 5070 Ti, driver 616.56).
> PRs with your own results are welcome: game, engine, API, scenario, and the proof log line.

## Done

### Death Stranding Director's Cut — ✅ working
`C:\Program Files (x86)\Steam\steamapps\common\DEATH STRANDING DIRECTORS CUT` · `ds.exe` · Steam 1850570
**D3D12, 64-bit, native DLSS (Streamline)** → Scenario A, `renodx-dlss5.addon64` alone.

Proof of life in `ReShade.log`:
```
Initializing crosire's ReShade version '6.8.0.2155' (64-bit) loaded from '...\dxgi.dll' into '...\ds.exe'
Registered add-on "DLSS 5 Neural Rendering" v0.2026.828.2110
RenoDX DLSS5 Generic v4.1.5 (build Aug 29 2026 15:54:01)
Running on NVIDIA GeForce RTX 5070 Ti Driver 616.56.
inline feature 18 evaluation succeeded (count=1, NR input 2560x1440 ... output 2560x1440 [native])
```

Three problems hit, in order — all three are the archetypes in
[troubleshooting](troubleshooting.md):

1. **Overlay wouldn't open.** RHI had installed ReShade as `ReShade64.dll` (OptiScaler's
   chain-load name) while OptiScaler itself had been removed, so nothing loaded ReShade.
   `OptiScaler.ini` also still had `LoadReshade = auto`, which means false. Fixed by
   installing ReShade as `dxgi.dll`.
2. **Effect covered only the top ~58% of the screen.** Guide/colour resolution mismatch —
   feature created with `guides 2560x1440`, evaluated with `guides 1485x835`, and
   `835/1440 = 0.58` = the game's DLSS Balanced ratio. Fixed by forcing DLAA.
3. **STANDBY/FAILED after switching to DLAA.** `nvngx_dlssnr.dll` was Authenticode
   **HashMismatch** (`4B8D19BC…`). Replaced with the Valid copy from
   `%LOCALAPPDATA%\RHI\DLSS-NR\310.8.0\` (`E16BCF15…`). The `.original` backup in the game
   folder had the *same bad hash* — it was not a clean fallback.

Notes:
- RHI deleted `renodx-deathstrandingdc.addon64` (the HDR fix) when `renodx-dlss5.addon64`
  was dropped in. One RenoDX addon per game under RHI.
- `dlss5-dx11-bridge.addon64` and `dlss5-feed.addon64` are deployed but do nothing here —
  the game is D3D12 with native DLSS. Harmless; the Feeder just logs a missing-shader warning.
- DLSSTweaks is present but **not loading** (Streamline bypasses it). Ignore its settings.

### INSIDE - WORKING (Feeder / Scenario C) 2026-08-30
`D:\SteamLibrary\steamapps\common\INSIDE` · `INSIDE.exe` · Unity, **D3D11 64-bit, no DLSS**

Proof in `dlss5-feed.log`:
```
[feed] frame 21600 delivered (2560x1440, reset=0)
[feed] 600 frames: feed CPU 0.82 ms/frame | frame interval 5.65 ms (177.1 fps) | feed is 15% of the frame
```
Cost ~1 ms/frame CPU, 9-19% of frame time. First successful **Feeder** install - the harder path.

Final layout: `dxgi.dll`, `dlss5-feed.addon64`, `renodx-dlss5.addon64` (v4.1.5), `nvngx_dlssnr.dll`
(Valid), `nvngx_dlss.dll`, and in `reshade-shaders\`: `Shaders\MartysMods_LAUNCHPAD.fx`,
`Shaders\MartysMods\` (15 .fxh), `Shaders\DLSS5_Feed.fx`, `Textures\iMMERSE_bluenoise_opt.png`.

Problems hit, in order:

1. **No overlay - RHI installed ReShade as `opengl32.dll`.** Its api cache lists
   `"Primary":"DirectX11","All":["OpenGL"]` and it picked OpenGL. Unity on Windows renders D3D11;
   Windows loads `opengl32.dll` incidentally, so ReShade *initialised* but never got a GL context
   -> no runtime, no overlay, no add-ons. **The tell: `ReShade.log` looked busy but never said
   "Searching for add-ons".** That line is the real proof a loader is correct. Fixed with `dxgi.dll`.
   Third distinct wrong-proxy-name bug in one night (`ReShade64.dll`, then `opengl32.dll`).
2. **`WAITING FOR NGX` with `evaluations 0`.** Correct and expected - INSIDE never calls NGX. This
   is the state the Feeder exists to resolve; `renodx-dlss5` alone can only decorate an existing
   DLSS call. `NGX modules detoured: 0 | core present: no` is normal here.
3. **Shaders compiled but techniques not enabled.** `dlss5-feed.log` said
   `DLSS5_Feed.fx technique MISSING ... LaunchPad technique MISSING` while `ReShade.log` showed both
   compiling fine. **Compiled != enabled** - they must be ticked on the *Home* tab, LaunchPad first,
   DLSS5_Feed below it.
4. **`nvngx_dlssnr.dll` was the HashMismatch copy again** (`4B8D19BC...`). The NR panel prints
   `Runtime sha256:` - read it there rather than hashing the file. RHI deploys the bad copy by
   default; the Valid one (`E16BCF15...`) is in `%LOCALAPPDATA%\RHI\DLSS-NRÈ.8.0\`.

**RHI actively fights this setup:** it reverted `dxgi.dll` back to `opengl32.dll`, wiped the
`reshade-shaders\` folder (it manages it - see `Managed by RDXC.txt`), and removed
`dlss5-feed.addon64`. Set the per-game **RS DLL** dropdown to `dxgi.dll` and then leave the game
alone in RHI.

**Where the two Feeder shader files came from** (neither ships with RHI):
- `DLSS5_Feed.fx` - `raw.githubusercontent.com/jlrouzies-fr/DLSS5-Feeder/main/shaders/DLSS5_Feed.fx`
- `iMMERSE_bluenoise_opt.png` - `raw.githubusercontent.com/martymcmodding/iMMERSE/main/Textures/iMMERSE_bluenoise_opt.png`

RHI *does* ship LaunchPad and its support headers at
`%LOCALAPPDATA%\RHI
eshade\Shaders\iMMERSE\` - copy `MartysMods_LAUNCHPAD.fx` and the whole
`MartysMods\` folder from there.

## Ruled out

### Half-Life 2 — ❌ impossible
Real **D3D9** device, **32-bit**, no `bin\x64\`. Proven by its own `bin\ReShade.log`:
`ReShade ... (32-bit) loaded from '...\bin\d3d9.dll' into '...\hl2.exe'`.

D3D9/D3D10/Vulkan/OpenGL are all unsupported. DXVK doesn't help — it produces Vulkan.
The only D3D9 carve-out is a game that *internally* wraps D3D9→D3D11, which HL2 does not.
See [decision-tree](decision-tree.md#worked-example).

## Candidate targets in this library

APIs below are from RHI's `game_api_cache.json`. Re-verify before starting.

| Game | API | DLSS? | Stack | Why it's a good target |
|---|---|---|---|---|
| **Baldur's Gate 3** | D3D12 (also D3D11/Vulkan modes) | native | A (addon alone) in DX12 mode; B (+bridge) in DX11 mode | **Best next target.** On the bridge's confirmed list, tested extensively with DLAA + all quality presets. Run it in DX11/DX12, not Vulkan. |
| **Metro 2033 Redux** | D3D11 64-bit | none | C (Feeder) | The Feeder's flagship proven title — documented working end to end. |
| **Metro Last Light Redux** | D3D11 64-bit | none | C (Feeder) | Same engine family as the above. |
| **Fallout 4** | D3D11 64-bit | via injector mod | B (bridge) | On the bridge's confirmed list. Needs a DLSS injector mod first. |
| **7 Days To Die** | D3D11 64-bit | native | B (bridge) | On the bridge's confirmed list. |
| **BeamNG.drive** | D3D12 64-bit | none | C (Feeder) | Theoretically viable and relevant to the separate BeamNG DLSS effort — but no reports of anyone doing it, and BeamNG has no upscaler hooks of its own. Expect this to be genuinely hard. |

## Confirmed-working lists from the tool authors

**dlss5-dx11-bridge** — Baldur's Gate 3 (Divinity 4.0), Final Fantasy XIV Online,
The Legend of Heroes: Trails beyond the Horizon, Tainted Grail: Fall of Avalon (Unity),
7 Days to Die (Unity), Skyrim Special Edition (DLSS injector mod), Fallout 4 (DLSS injector
mod), S.T.A.L.K.E.R. Anomaly (SSS24 upscaler injector), Assetto Corsa (Custom Shaders Patch
Preview 338+).

The last four matter: they prove the bridge works off **injected** DLSS, not just native.

**DLSS5-Feeder** — Metro 2033 Redux (D3D11 64-bit), The Lord of the Rings: War in the North
Legacy Edition (D3D12 64-bit, no native DLSS), Splinter Cell: Blacklist (D3D11 32-bit,
beta), BioShock Remastered (D3D11 32-bit, beta — D3D9 wrapped to D3D11).

**dlss5-d3d12-fix** — Resonance: A Plague Tale Legacy (sole tested title).

**Per-game fix example** — Dying Light: The Beast via `dltb-dlss5-fix`.

**Community ports reported** — Control (the first), Skyrim, GTA V, Cyberpunk 2077, RDR2,
TLOU2 Remastered, Hogwarts Legacy, Black Myth Wukong, FF7 Rebirth.

## Template for a new game

```
Game:            
Install path:    
Exe:             
API / bitness:   (from game_api_cache.json, ReShade.log, or exe imports)
Has DLSS:        (native / injector mod / none)
Stack chosen:    (A / B / C / D from install-playbook.md)
Proxy DLL:       (dxgi.dll ...)
Result:          
ReShade.log evidence:
Problems + fixes:
```

## Open leads

### GMOD via RTX Remix + Vulkan bridge (Discord report, 2026-08-30)
Someone in the RenoDX Discord reported getting **Garry's Mod** working with **RTX Remix +
a Vulkan bridge**. Unverified and second-hand — but worth chasing, because if it holds it
attacks the exact wall that kills Half-Life 2.

Why it's interesting: RTX Remix replaces a fixed-function D3D8/D3D9 renderer wholesale
rather than wrapping it, so a Source-engine game stops being "a real D3D9 device" — the
condition that makes D3D9 impossible for every tool in this folder.

Why to stay sceptical until tested: Remix's runtime is DXVK-derived and presents **Vulkan**,
and Vulkan is explicitly unsupported by the Feeder ("DX9 / DX10 / Vulkan won't work"). So
the question to answer first is *what API ReShade actually sees* once Remix is in the chain.
If ReShade reports D3D12, this is a real route to Source-engine games. If it reports Vulkan,
it is another dead end wearing a new hat.

**Next step if picked up:** install Remix on GMOD, launch, and read `ReShade.log` for the
`Initializing crosire's ReShade ... loaded from '...'` line and the API it reports. That one
line settles it. Do not install any DLSS 5 components until that line is known.

## Triaged 2026-08-30 (second batch)

| Game | Path | API | Bits | DLSS | Verdict |
|---|---|---|---|---|---|
| The Wolf Among Us | `D:\SteamLibrary\...\The Wolf Among Us` | **DirectX9** | 32-bit | none | ❌ **Dead.** Telltale Tool, real D3D9 device. Same wall as HL2. |
| INSIDE | `D:\SteamLibrary\...\INSIDE` | DirectX11 (OpenGL also detected) | 64-bit | none | ✅ Feeder. Force D3D11 with `-force-d3d11` if it starts in OpenGL. |
| Subnautica | `D:\SteamLibrary\...\Subnautica` | DirectX11 | 64-bit | none* | ✅ Feeder. Best visual showcase of the batch. |
| BeamNG.drive | `C:\...\BeamNG.drive` | DirectX12 | 64-bit** | none | ⚠️ Feeder. Qualifies on API, but hardest — no upscaler hooks, nobody has done it. |

\* Subnautica **appears** to have DLSS files (`nvngx_dlss.dll`, `sl.*.dll`, even `nvngx_dlssnr.dll`) — but
these are byte-for-byte the same sizes RHI deployed to Death Stranding. They are **RHI-deployed, not
native**. Subnautica (Unity, 2018) does not call NGX itself, so those DLLs are inert. Treat it as a
no-DLSS game → Feeder, not the bridge. Good reminder: **the presence of `nvngx_dlss.dll` does not mean
the game has DLSS** — RHI sprays those DLLs around. Check whether the *game* calls NGX.

\** BeamNG needs `BeamNG.drive.x64.exe`. The plain `BeamNG.drive.exe` is a 32-bit launcher shim.

**Recommended order:** run **Metro 2033 Redux** first as a calibration run — it is the Feeder author's
own proven title, so if the chain fails there the fault is the setup, not the game. Once a known-good
Feeder install exists, repeat it on Subnautica / INSIDE with confidence.

## Batch 3 - triaged and installed 2026-08-30

| Game | API | Bits | DLSS | Scenario | Result |
|---|---|---|---|---|---|
| Gunner, HEAT, PC! | D3D11 | 64 | none | C Feeder | WORKING. Exe is in `Bin\` - install there, not game root. |
| Cities: Skylines (1) | D3D11 | 64 | none | C Feeder | WORKING. CPU-bound game; Feeder's ~1 ms/frame actually costs here. |
| Control | D3D12 | 64 | native | A | installed. `Control_DX12.exe` - must NOT launch the DX11 exe. |
| Routine | D3D12 | 64 | native | A | installed to `Routine\Routine\Binaries\Win64\` next to `Routine-Win64-Shipping.exe` (root `Routine.exe` is only a launcher). |
| Journey | D3D11 | 64 | none | C Feeder | installed, on Feeder **v0.3.0** (the other Feeder games are v0.2.0). |
| RFG Re-MARS-tered | D3D11 | **32** | none | D host64 | installed, worked but underwhelming - two stacked approximations (cross-process pipe + estimated MVs). |
| Alien Isolation | D3D11 | **32** | none | D host64 | viable, not yet installed. Same ceiling as Re-MARS-tered. |
| Red Faction Guerrilla (orig) | **D3D9** | 32 | - | - | DEAD |
| Mirror's Edge | **D3D10**/D3D9 | 32 | - | - | DEAD (D3D10 is separately unsupported, not just D3D9) |
| Garry's Mod | Source/D3D9 | 32 | - | - | DEAD by normal rules - see the RTX Remix open lead above |
| The Wolf Among Us | **D3D9** | 32 | - | - | DEAD |

**Why Control is the highest-value target:** the addon builds a *"Control-equivalent soft-clip/sRGB/
UpgradeToneMap codec"* - that string appears in every game's log. Control is the calibration
reference for the colour pipeline, so it is the title most likely to look correct rather than
approximated.

**Feeder v0.3.0** (2026-08-30 04:21) added the 32-bit path. Assets:
`github.com/jlrouzies-fr/DLSS5-Feeder/releases/download/v0.3.0/` - `dlss5-feed.addon32`,
`dlss5-feed-host64.exe` (64,000 bytes, **unsigned**), `dlss5-feed.addon64`, `DLSS5_Feed.fx`.
The v0.3.0 `addon64` (MD5 4183E486...) differs from v0.2.0 (B1F2EEB7...) - it is a real rebuild.

**Pattern worth remembering:** the win rate is set almost entirely by API + bitness, and every
install failure so far has been a wrong proxy DLL name, not a broken tool.

## Routine (UE 5.5) - FAILED, abandoned 2026-08-30

`C:\Program Files (x86)\Steam\steamapps\common\Routine\Routine\Binaries\Win64`

Ships DLSS + FSR + StreamlineCore plugins. **No `sl.dlss.dll`** - only `sl.dlss_g`, `sl.reflex`,
`sl.pcl`, `sl.deepdvc - so super resolution goes straight to NGX and Streamline is not the router.
`EnableHooks=1` is pointless here; leave it at 2.

Defaults to **D3D11** despite RHI reporting `"Primary":"DirectX12"`. `-dx12` does force D3D12.

Four separate problems, fixed in order, and it still failed:
1. RHI's API cache was wrong - game ran D3D11 while `renodx-dlss5` hooks D3D12 -> `-dx12`
2. `dlss5-dx11-bridge` hooked D3D11 entry points in the now-D3D12 process -> removed
3. AA persisted as `ERoutineAAMethod::NONE`, so the DLSS plugin never created a feature
4. Switched to Scenario C (Feeder). NGX session opened cleanly, then
   `CreateFeature raised exception 0xC0000005` on a **R10G10B10A2_UNORM** backbuffer

Reverted to stock (dxgi.dll + renodx-dlss5.addon64 only). **Retry when the add-on supports 10-bit
backbuffers** - that is the only remaining blocker, and it is add-on side.
