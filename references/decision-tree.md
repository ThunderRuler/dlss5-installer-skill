# Picking the right stack

Two facts decide everything: **the game's graphics API** and **whether it already has DLSS**.
Establish both before touching a single file. Getting this wrong is how you lose an evening.

> **Updated 2026-08-30.** DLSS5-Feeder v0.5.1+ added **working Vulkan support**, and a
> **D3D9 path via dgVoodoo2**. Earlier versions of this document (and the Feeder's own older
> README) said both were impossible. They are not. **D3D10 is now the only hard dead end.**

## Step 1 — determine the API and bitness

In rough order of reliability:

**a) The game's own log**, if it has one. Often the most direct answer:
```
gfx:D3D11, gpu:NVIDIA GeForce RTX 5070 Ti      <- BeamNG.drive's beamng.log
```

**b) `ReShade.log`, if ReShade already loads** — definitive for what ReShade sees:
```
Initializing crosire's ReShade version '6.8.0.2155' (64-bit) loaded from '...\dxgi.dll' into '...\game.exe'
Redirecting D3D12CreateDevice(...)              <- real D3D12
Redirecting D3D11CreateDevice(...) / feature level b100   <- D3D11
```

**c) The exe's imports** — works without launching anything, but only tells you what the
binary *can* do:
```bash
grep -a -o -i -E "dxgi\.dll|d3d12\.dll|d3d11\.dll|d3d9\.dll|vulkan-1\.dll" game.exe | sort -u -f
```
Unity's `UnityPlayer.dll` imports *every* API — it proves nothing on its own.

**d) Installer caches (RHI's `game_api_cache.json`)** — a fine first filter, **not
authoritative**. See the correction at the bottom of this file.

## Step 2 — does it have DLSS?

Look for `nvngx_dlss.dll`, `sl.dlss.dll`, or a DLSS option in the game's video settings.
**DLSS injected by a mod counts** — the bridge's confirmed list includes Skyrim SE and
Fallout 4 (DLSS injector mods) and STALKER Anomaly (SSS24 upscaler injector), plus
Assetto Corsa via Custom Shaders Patch Preview 338+.

### Beware the false positive

Library managers (RHI in particular) **spray `nvngx_*.dll` and the whole `sl.*` set into
game folders** whether or not the game uses them. Their presence proves nothing. Check:

- **File dates.** If the DLLs are dated days ago and the game's own files are years old,
  they were deployed by a tool.
- **Where they live.** A UE game with real DLSS keeps it in
  `<Game>\Plugins\{DLSS,StreamlineCore}\Binaries\ThirdParty\Win64\`, not the root.
  No `Plugins` folder at all = no native DLSS.
- **Strings in the engine binary.** Zero hits for `dlss|nvngx|streamline` means the game
  never calls it:
  ```bash
  grep -a -o -i -E "dlss|nvngx|streamline" game.exe | sort | uniq -c
  ```

Subnautica and BeamNG.drive both look like DLSS games by file listing alone. Neither is.

## Step 3 — read off the stack

| API | Bits | Has DLSS | Stack |
|---|---|---|---|
| D3D12 | 64 | yes | `renodx-dlss5.addon64` only |
| D3D12 | 64 | no | `renodx-dlss5` + **DLSS5-Feeder** |
| D3D11 | 64 | yes | `renodx-dlss5` + **dlss5-dx11-bridge** |
| D3D11 | 64 | no | `renodx-dlss5` + **DLSS5-Feeder** |
| **Vulkan** | 64 | either | `renodx-dlss5` + **DLSS5-Feeder** — see below, ReShade installs as a *layer* |
| D3D11 | 32 | either | **DLSS5-Feeder 32-bit path** (`.addon32` + `host64\`) |
| **D3D9** | either | — | **dgVoodoo2 → D3D11**, then treat as a normal install |
| D3D9 wrapped→D3D11 already | either | — | treat as D3D11 (e.g. BioShock Remastered) |
| D3D12 | 32 | — | ✗ no D3D12 in the 32-bit runtime |
| **D3D10** | — | — | ✗ **the only remaining hard dead end** |

Never run the Feeder and the DX11 bridge on the same game.

### The DX11 bridge is D3D11-only — check the entry point names

The bridge prints a confident hook report that means nothing in a D3D12 process. Read the
**entry point names**, not the `hooked: yes` line:

```
NVSDK_NGX_D3D11_CreateFeature = 00007FFE104D5160   <- D3D11!
hooked: CreateFeature=yes EvaluateFeature=yes
```

Six layers can all report `hooked: yes` while the game calls `NVSDK_NGX_D3D12_CreateFeature`,
which the bridge never touches — giving `creates 0` with a log that looks perfect.

## Vulkan (v0.5.1+)

Straight from the Feeder README: *"For Vulkan games this means: just install the add-on
like a 64-bit D3D11/D3D12 game."* Two differences:

1. **ReShade for Vulkan is a layer, not a `dxgi.dll`.** Its installer registers it globally
   and gates it per-application — point the installer at the game exe so it is on that list.
   The add-ons still sit next to the game exe, and `ReShade.ini` needs `AddonPath=.\` under
   `[ADDON]` so they are found.
2. Everything else is identical.

The transport needs KHR external-interop extensions on the game's `VkDevice`, which most
games do not request. The add-on hooks `vkCreateDevice` and appends them itself (ReShade
loads add-ons inside its `vkCreateInstance` hook, before the game creates its device). If
the driver refuses the extended list the call is retried untouched, so the hook can never
stop a game from starting. `dlss5-feed.log` lists each extension and where it came from.

Only if the log says the interop entry points are still missing, use the bundled
out-of-process layer (`feed-vk-layer.zip` → `layer\run-with-feed-layer.bat "path\to\game.exe"`).

**Confirmed:** DOOM (2016), 64-bit Vulkan, 4K.

## D3D9 via dgVoodoo2 (beta)

A *real* D3D9 device still cannot work directly: ReShade on D3D9 caps at Shader Model 3, so
no motion-vector provider (all SM5) will even compile, and D3D9 has no shared NT handles or
fences for the transport. **The fix is to translate D3D9 → D3D11 first with dgVoodoo2.**

Key points that trip people up:

- Use `MS\x86\D3D9.dll` from dgVoodoo's **`MS`** folder (`x86` for a 32-bit game).
- Set `DisableAndPassThru = false`. **The shipped default is `true`**, which forwards
  everything to the real D3D9 and does nothing — the single most common "dgVoodoo isn't
  working" cause.
- Set `VRAM = 1GB`. The default 256 MB causes "ran out of video memory" crashes regardless
  of your real GPU. **Do not use 2GB** — 2048 MB in bytes is `0x80000000`, which overflows a
  signed 32-bit integer and old engines mishandle it.
- **ReShade must be installed as `dxgi.dll`, never `d3d9.dll`** — dgVoodoo owns that name.
- Confirm you have a genuine D3D9 game first. If `ReShade.log` already says
  `D3D11CreateDevice` and `feature level b000/b100`, the game wraps to D3D11 on its own —
  skip all of this, it is a normal install.

**Confirmed:** Fable Anniversary (32-bit D3D9 via dgVoodoo2), BioShock Remastered
(self-wrapping).

Bitness is *not* the blocker people assume — the Feeder has a working 32-bit path with a
`host64\` helper. **The API is the blocker, and the list of blocked APIs is now just D3D10.**

## IMPORTANT correction: an installer's API cache says what a game *can* do, not what it *does*

`game_api_cache.json` reports `Primary` from static inspection of the binaries. It is a fine
first filter but **not authoritative about the API actually in use at runtime**.

Caught on **Routine** 2026-08-30. RHI reported:
```
"Routine\Routine\Binaries\Win64": {"Primary":"DirectX12","All":["OpenGL","DirectX12","DirectX9"]}
```
The game actually launched in **D3D11** (`Redirecting D3D11CreateDevice`, `Using feature
level b100`), while `renodx-dlss5` hooks **D3D12 NGX only** — so every symptom fit a broken
install (`core present: yes`, `NGX modules detoured: 3`) while `creates` stayed at 0, because
the creates were on an API the addon never watches.

Caught again on **BeamNG.drive**: RHI's cache said D3D12, an old game log said D3D11, and the
current version defaults to D3D12 with D3D11 marked *obsolete*. **A stale log is as
misleading as a stale cache** — check the log's date against the game's version.

| Log line | Meaning |
|---|---|
| `Redirecting D3D12CreateDevice` | real D3D12 |
| `Redirecting D3D11CreateDevice` + `feature level b100` | **D3D11** |
| `loaded from '...d3d9.dll'` + `(32-bit)` | real D3D9 — needs the dgVoodoo2 path |
| `d3d12.dll` merely appearing in a `LoadLibrary` line | **not** proof of D3D12 |

The same trap applies to the **UE `-dx12` launch option**, and to launchers that offer a
renderer choice (BeamNG's launcher lists D3D12 / Vulkan / D3D11). Ask which one the user
actually clicked.
