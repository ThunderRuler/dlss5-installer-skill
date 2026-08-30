# Picking the right stack

Two facts decide everything: **the game's graphics API** and **whether it already has DLSS**.
Establish both before touching a single file. Getting this wrong is how you lose an evening.

## Step 1 — determine the API and bitness

In rough order of reliability:

**a) RHI's cache** — fastest for a library sweep:
`%LOCALAPPDATA%\RHI\game_api_cache.json`, keyed by install path, e.g.
`{"Primary":"DirectX12","All":["DirectX12"]}`

**b) `ReShade.log`, if ReShade already loads** — definitive. It states bitness and the DLL
it loaded from:
```
Initializing crosire's ReShade version '6.8.0.2155' (64-bit) loaded from '...\dxgi.dll' into '...\ds.exe'
```
A line like `loaded from '...\bin\d3d9.dll'` and `(32-bit)` tells you immediately to stop.
The Feeder README also notes `Using feature level b100` = D3D11.

**c) The exe's imports** — works without launching anything:
```bash
grep -a -o -i -E "dxgi\.dll|d3d12\.dll|d3d11\.dll|d3d9\.dll" game.exe | sort -u -f
```

**d) Which DLLs the game folder ships** — `sl.interposer.dll` / `sl.dlss.dll` means
Streamline, so the game has native DLSS *and* DLSSTweaks will be bypassed.

## Step 2 — does it have DLSS?

Look for `nvngx_dlss.dll`, `sl.dlss.dll`, or a DLSS option in the game's video settings.
**DLSS injected by a mod counts** — the bridge's confirmed list includes Skyrim SE and
Fallout 4 (DLSS injector mods) and STALKER Anomaly (SSS24 upscaler injector), plus
Assetto Corsa via Custom Shaders Patch Preview 338+.

## Step 3 — read off the stack

| API | Bits | Has DLSS | Stack |
|---|---|---|---|
| D3D12 | 64 | yes | `renodx-dlss5.addon64` only |
| D3D12 | 64 | no | `renodx-dlss5` + **DLSS5-Feeder** |
| D3D11 | 64 | yes | `renodx-dlss5` + **dlss5-dx11-bridge** |
| D3D11 | 64 | no | `renodx-dlss5` + **DLSS5-Feeder** |
| D3D11 | 32 | either | **DLSS5-Feeder 32-bit path** (`.addon32` + `host64\`) |
| D3D12 | 32 | — | ✗ not supported (no D3D12 in the 32-bit runtime) |
| D3D9 wrapped→D3D11 | either | — | treat as D3D11 |
| D3D9 (real device) | — | — | ✗ impossible |
| D3D10 / Vulkan / OpenGL | — | — | ✗ impossible |

Never run the Feeder and the DX11 bridge on the same game.

## Why the dead ends are genuinely dead

Straight from the DLSS5-Feeder README: **"DX9 / DX10 / Vulkan won't work"** and
**"NGX is 64-bit only."**

The one D3D9 carve-out is *"A game whose D3D9 calls are wrapped to D3D11 works fine"* —
that means a game that internally renders through a D3D11 wrapper, like BioShock
Remastered. It does **not** mean you can bolt a wrapper on:

- **DXVK** translates D3D9 → **Vulkan**, and Vulkan is explicitly unsupported.
- A real D3D9 device lacks shared fences and has restricted shared-surface formats, so
  the transport the Feeder relies on cannot be built.

Bitness is *not* the blocker people assume — the Feeder has a working 32-bit path with a
`host64\` helper, proven on Splinter Cell: Blacklist and BioShock Remastered. **The API is
the blocker.**

### Worked example: "it works on Skyrim, so it'll work on Half-Life 2"

Both halves are checkable, and they give opposite answers:

- **Skyrim Special Edition** — 64-bit **D3D11**, DLSS via injector mod. Squarely in the
  DX11 bridge's confirmed list. Works.
- **Half-Life 2** — its own `bin\ReShade.log` shows ReShade loading **32-bit** from
  `bin\d3d9.dll` into `hl2.exe`. A real D3D9 device, no `bin\x64\`. Impossible.

Same "old game", opposite outcome, because the API differs. Always check the API — never
reason from the game's age or reputation.

## IMPORTANT correction: RHI's API cache says what a game *can* do, not what it *does*

`game_api_cache.json` reports `Primary` from static inspection of the binaries. It is a fine first
filter but it is **not authoritative about the API actually in use at runtime**.

Caught on **Routine** 2026-08-30. RHI reported:
```
"Routine\Routine\Binaries\Win64": {"Primary":"DirectX12","All":["OpenGL","DirectX12","DirectX9"]}
```
The game actually launches in **D3D11**:
```
Redirecting D3D11CreateDevice(...)
Using feature level b100          <- D3D_FEATURE_LEVEL_11_0
```

Cost: a long detour chasing `HOOKS ARMED - NO DLSS CREATE SEEN` and blaming Streamline routing,
the in-game upscaler setting, and the NVIDIA App override - when the real cause was that
`renodx-dlss5` hooks **D3D12 NGX only** while the game was creating **D3D11** NGX features. Every
symptom fit: DLSS DLLs loaded, `core present: yes`, `NGX modules detoured: 3`, and still
`creates 0`, because the creates were on an API the addon never watches.

**Always confirm the runtime API from `ReShade.log` before concluding anything:**

| Log line | Meaning |
|---|---|
| `Redirecting D3D12CreateDevice` / `D3D12` device lines | real D3D12 - Scenario A works |
| `Redirecting D3D11CreateDevice` + `Using feature level b100` | **D3D11** - needs the DX11 bridge |
| `loaded from '...d3d9.dll'` + `(32-bit)` | D3D9 - dead |
| `d3d12.dll` merely appearing in a `LoadLibrary` line | **not** proof of D3D12 - engines probe for capability without using it |

The same trap applies to the **UE `-dx12` launch option**: a UE title that lists D3D12 support may
still default to D3D11. If a game reports D3D11 but you expect D3D12, try `-dx12` in Steam launch
options and re-read the log.

**Rule of thumb:** a D3D11 game with native DLSS is Scenario B (bridge), and the symptom that
distinguishes it from a broken install is `creates 0` *with* `core present: yes` and the game's own
`nvngx_dlss.dll` showing up in the detoured-module list.
