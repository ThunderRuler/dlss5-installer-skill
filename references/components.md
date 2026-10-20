# Components

Every moving part, what it does, and what it requires. **Updated 2026-09-29** — the
ecosystem moved fast in September; version-sensitive claims carry a date.

## The one-paragraph model

A DLSS 5 install has two roles. The **neural consumer** is the thing that actually calls
NVIDIA's neural runtime (`nvngx_dlssnr.dll`): the RenoDX DLSS5 addon, ShortFuse's RenoDX
DLSS addon, or the OptiScaler DLSS-NR fork. A **feeder/bridge** is only needed when the
game cannot hand the consumer what it wants: DLSS5-Feeder synthesises DLSS inputs for
games with no DLSS; the dlss5-bridge carries a D3D11/Vulkan game's DLSS call over to D3D12.
**Exactly one consumer per game.** Two consumers conflict (one goes silently inert).

## Official DLSS 5 — check this first

NVIDIA shipped DLSS 5 officially on **2026-09-03** (Game Ready Driver **616.64**).
Launch title: **NBA 2K27**, with more announced for fall 2026 (AC Shadows, Hogwarts
Legacy, Starfield, Oblivion Remastered, RE Requiem, Phantom Blade Zero, Delta Force,
Naraka, Where Winds Meet, AION 2, Black State, Cinder City, Justice, NTE, Sea of Remnants,
EA Sports FC). Official support is **RTX 50 only**; RTX 40 is promised "later"
with no date. There is **no** NVIDIA App override for DLSS 5.

**If the game has an official "DLSS Neural Rendering" option, use it and do not mod.**
The official path has developer-tuned models and per-pixel masks the leaked runtime lacks,
and several of those titles run anti-cheat (Delta Force, Naraka, AION 2, EA Sports FC — Gate 2 refuses).

## Core

### ReShade 6.8+ — **Addon build**
The host the addons plug into. Must be the build with add-on support
(`ReShade_Setup_X.X.X_Addon.exe`); the Feeder rejects anything else since 0.14.

- Installed as a **proxy DLL named after something the game imports**: `dxgi.dll` for
  D3D11/D3D12. For **Vulkan** it is a *layer*, registered per-application by the
  installer; the machine-wide layer (`C:\ProgramData\ReShade\ReShade64.dll`) must itself
  be the Addon build, and `ReShade.ini` needs `AddonPath=.\` under `[ADDON]`.
- Loads add-ons next to the DLL that loaded it; `.addon64` in 64-bit, `.addon32` in 32-bit.
- Writes `ReShade.log` next to the exe — the primary diagnostic.
- **Rewrites `ReShade.ini` on exit** — only edit it with the game closed.

Not needed at all for the OptiScaler route (Scenario E).

### `nvngx_dlssnr.dll` — the neural runtime
Version **310.8** in all circulating builds; no newer NVIDIA build exists as of
2026-09-29 (the "310.9.x" numbers are the ordinary DLSS SR/RR SDK). There are several
310.8 variants — **which one is acceptable depends on the GPU, decided by SHA-256**. See
`known-hashes.md`; it is the single source of truth for this file.

### `nvngx_dlss.dll` — super resolution
Standard DLSS DLL. Needed alongside the NR runtime even in games with no DLSS of their
own. Any recent one works (3.1.13+ for the bridge's substitute path); DLSS Swapper or
`rhi-repo`'s `dlss-310.9.1` are clean sources.

### Driver
Minimum **616.56**. On **616.64 and newer** (i.e. any current driver) the RenoDX DLSS5
addon's **v4.6 and v4.7 ("lazy-adoption engine") builds fault** inside
`nvngx_dlssnr.dll` (Feeder issue #54). Per the Feeder's own `--test` table: classic-engine
v4.5 passes on 616.64; v6.1.0, v7.0.0-rc8 and v8.0.1 pass on 617.14; v5.x was never
tested. In a real game (Death Stranding DC, 616.92) classic **v4.1.5** also degraded —
NR on ~1 frame/sec, no error. **Use v6.1 or newer.** (6.1.0 and 8.0.1 aren't on rhi-repo; 6.4.1+, 7.0.0-rc8
and 8.5.0-rc10 are.) Staying on 616.56 is the other workaround.

## Neural consumers (pick exactly one)

### `renodx-dlss5.addon64` — RenoDX DLSS5 Generic (the default here)
The ReShade addon that hooks the game's NGX calls and inserts the neural pass.

- Current: **v8.5.0-rc10** (2026-09-26). Most versions mirrored on
  `RankFTW/rhi-repo` releases (`renodx-dlss5-<ver>`); not on clshortfuse/renodx's own
  releases (neither `snapshot` nor the new `nightly-YYYYMMDD` tags carry it). Also
  circulates via the RenoDX Discord and Nexus.
- **Recommended v6.1+ on driver 616.64+** (v4.6/v4.7 fail there).
- Hotkeys: **F6** toggles NR (configurable, `NRToggleKey`), **F5** screenshot.
- Settings under `[RenoDX.DLSS5]` in `ReShade.ini` and in its overlay panel. Recent
  builds (late v7 line onward) add a **hook point** choice (`NRHookPoint`: after upscaling / before upscaling / on the final
  Present frame), multi-pass, and a "look" stack — see `config-reference.md`.
- **Unconfirmed (from a third-party source copy, `PEQHUB/RenoDX-DLSS5-Generic`, branch
  `v8.5dev` — a development snapshot):** v8.x
  appears to carry its own D3D11 bridge and a native Vulkan path. Until a log proves it
  on a real game, keep using the dlss5-bridge for D3D11/Vulkan DLSS games.
- Its "Upload & copy link" diagnostics button sends a report archive to a third-party
  file host — tell the user before they press it.

### `renodx-dlss.addon64` — ShortFuse's RenoDX DLSS ("SF")
A separate addon by the RenoDX author, released as `renodx-dlss-SF-<date>` on
`rhi-repo`. Has a built-in cost scaler. The Feeder README now recommends it for 64-bit
D3D9/11/12 games that **already call DLSS themselves**. Same one-consumer rule applies.

### OptiScaler DLSS-NR fork (wilsjo2) — no ReShade at all
`wilsjo2/OptiScaler-DLSSNR-PreSR-Multipass` (built on Dagherbou's OptiScaler_DLSSNR). An
OptiScaler build that **drives `nvngx_dlssnr.dll` itself** as a pass around the game's
upscaler — it replaces the RenoDX addon rather than working beside it.

- Latest stable **v0.8.3**; pre-releases to v0.8.91. The Feeder author hosts a
  flicker-fixed **v0.8.92** (`jlrouzies-fr/OptiScaler-DLSSNR-PreSR-Multipass`) for model
  resolution below 100%.
- Use `OptiScaler-NR-v<ver>.zip`, **not** the `-rtx40-mfg` variant (unless you know you want it).
- Works on the game's **own upscaler input** (DLSS, FSR or XeSS). Native D3D12 preferred;
  D3D11/Vulkan only via OptiScaler's D3D12 bridges (e.g. `Dx11Upscaler=dlss_12`).
- Can run NR **before** super-resolution (`RunBeforeSR=true`) — i.e. at render
  resolution, which is the first real answer to DLSS 5's ~50% cost — or on the
  finished picture (`FinishedPicture=true`, native D3D12 only).
- Menu: **Insert**. Log: `OptiScaler.log`.
- Needs the full release zip plus a pinned `nvngx_dlssnr.dll`. wilsjo2 v0.8.1+ loads the
  model directly and ships **no** `nvngx.dll_dlssnr.dll` forwarder (its `INSTALL-DLSSNR.md`
  still mentions one — outdated). Only Dagherbou's original build uses the forwarder.

### Deep Fried Chicken — mention only
A consumer the Feeder README calls "recommended", but it has **no public download**
(Discord only) and is often quarantined by Defender. The skill does not install it; if
the user already has it, it counts as their one consumer.

## Feeders and bridges

### `dlss5-bridge.addon64` (NIGos) — games that already have DLSS
**Renamed** from `dlss5-dx11-bridge` (repo `NIGos/dlss5-bridge`; old URL redirects).
Stable **v1.4.12**, pre-release v1.4.13-pre8.

Three routes:
1. **D3D11 bridge** — intercepts the game's D3D11 DLSS evaluate, reproduces the contract on
   a private D3D12 NGX session where the consumer runs, copies the result back.
2. **Vulkan mirror** (`vk_mirror=1`, default) — the same for **Vulkan games with DLSS**
   (BG3 Vulkan, RDR2, Endfield).
3. **Substitute path** (`synth=1`, off by default) — fakes DLSS inputs for games without
   DLSS using NVIDIA Optical Flow. The README admits it looks worse; prefer the Feeder.

- **Delete the old `dlss5-dx11-bridge.addon64`** when upgrading — ReShade loads both and
  the old one wins. The old `.cfg` is read only if no `dlss5-bridge.cfg` exists.
- Writes `dlss5-bridge.log` (lists each loaded addon's SHA-256).
- Frame generation: DLAA + MFG 4x + NR reported working together on pre5 (Endfield).
- Fails safe: if anything goes wrong it disables itself and the game renders normally.
- **`dlss5bridge.com` is not the project** — its installer is flagged malicious (tries to add
  Defender exclusions). The real bridge has no installer.

### `dlss5-feed.addon64` / `.addon32` (DLSS5-Feeder, jlrouzies-fr) — games with no DLSS
Current stable **v1.17.0** (2026-09-27); **v1.18.0-beta.1** (2026-09-29) adds a 64-bit
*helper mode* (`dlss5-feed-helper.addon64` — NGX runs in `host64\` for 64-bit games whose
own process can't run it, e.g. shadPS4, and an untested route for 64-bit D3D10) and logs
NGX's own failure reason. Use `addon64` for every other 64-bit game. Synthesises a **DLAA** contract from ReShade's depth
buffer plus a motion-vector shader, and hands it to the consumer.

API coverage (1.17.0):

| Path | How |
|---|---|
| 64-bit D3D11 | private D3D12 device + shared textures/fences |
| 64-bit D3D12 | **same-device** — zero copies. Refuses if the game loaded its own DLSS/Streamline (override `native_dlss_ok=1`, but such games belong on Scenario A/B) |
| 64-bit Vulkan | hooks `vkCreateDevice` to add interop extensions; layer fallback bundled |
| **OpenGL** 32/64-bit | new since 0.7 — `GL_EXT_memory_object_win32`; game must render on the NVIDIA GPU |
| 32-bit D3D11/OpenGL/Vulkan | `addon32` + `host64\dlss5-feed-host64.exe` helper |
| **32-bit D3D10** | new since 0.13.1 — native, `addon32` only |
| D3D9 / D3D8 | dgVoodoo2 → D3D11 wrapper; **64-bit D3D9** can alternatively use DXVK → Vulkan |

- **DLAA only, permanently** — the README says upscaling modes are "not as a setting, and
  not as a future one either". `work_resolution` (D3D11) is a cost knob that looks blurry.
- **iMMERSE LaunchPad is no longer required** — it is just provider `1`. The recommended
  provider is LumeniteFX Kernel (`DLSS5_MV_PROVIDER=3`). See `motion-vectors.md`.
- Ships `Install-DLSS5Feeder.ps1` (full automated install) and `Verify-DLSS5Feeder.ps1`
  (read-only `[ OK ]/[WARN]/[FAIL]` check). See `install-playbook.md`.
- `addon32` and `host64\` must come from the **same zip** (IPC protocol v11; a mismatch
  refuses to connect).
- No frame generation of its own. NVIDIA Smooth Motion must be **off for Vulkan**.
- **Fake Feeder sites and copycat repos exist.** The project ships no `.exe`; the only real
  download is its GitHub Releases page.
- Writes `dlss5-feed.log`; crash dumps as `dlss5-feed-crash.dmp`.

## Fixes

### `dlss5-d3d12-fix.addon64` (NIGos) — *obsolete*
For D3D12 games whose DLSS output has a mip chain. Superseded since NR addon v3.3.5.
Symptoms it addressed: STANDBY/FAILED, guides 0x0, repeated `0xBAD00005`.

### `dltb-dlss5-fix` (markitzeroo) — the per-game fix pattern
Upgrades a `R16G16B16A16_TYPELESS` DLSS output to typed `R16G16B16A16_FLOAT` at creation
(`"DLSS output format 9 is not a supported typed codec format"`). The template for
per-game resource-upgrade work; selection filters must be retuned per game.

### `dlssnr-signature-repair` (kayle2203)
Replaces a HashMismatch runtime with a verified local copy. Note it treats *any*
HashMismatch as broken — **do not run it against the pinned SF-v2 build on a 20/30/40
card**, which is modified on purpose.

## Management

### RHI (RankFTW) — "ReShade HDR Installer"
Library-wide installer, current **2.7.8** (2026-09-27). Has a **Neural Rendering** section
with four methods: *DLSS5 Tool* (D3D12 with DLSS), *DLSS5 Tool + DX11 Bridge* (D3D11 or
Vulkan with DLSS), *ShortFuse DLSS Tool*, and *DLSS5 Feeder* (no DLSS, incl. 32-bit and
D3D9 via dgVoodoo2). Also offers the OptiScaler DLSS NR variant. Can **pin and hot-swap**
versions of the addon, Feeder, bridge and runtime (2.7.x), and writes `rhi_install.txt`
into each game folder it touches.

Runtime choices in its manifest: "310.8.0 (50xx)" = signed original, "310.8.2
(20/30/40/50)" = ShortFuse SF-v2, "310.8.3 (50xx)" = Lecram. Only the first two are
pinned in `known-hashes.md`.

State in `%LOCALAPPDATA%\RHI\`:

| Path | Contents |
|---|---|
| `installed.json` / `aux_installed.json` | per-game RenoDX addon / ReShade + OptiScaler **and the DLL name used** |
| `addon_deployments.json` | addons deployed per game |
| `game_api_cache.json` | detected API per game — a first filter, **not** authoritative |
| `DLSS-NR\<version>\` | cached runtime(s) — hash before trusting |
| `rdx5\` | cached consumer addon; `version.txt` / `version-sf.txt` hold its version |
| `optiscaler-dlssnr\` | cached OptiScaler DLSS-NR fork |
| `logs\session_*.txt` | every deploy, rename and delete it performed |

Gotchas:
- The per-game **"RS DLL" dropdown** decides the proxy name — `dxgi.dll` for D3D11/D3D12.
- Dragging `renodx-dlss5.addon64` onto a game **deletes** that game's per-game RenoDX
  addon (e.g. an HDR fix). One RenoDX addon per game.
- Its "global addon set" redeploys on refresh — hand-deleting files doesn't stick.
- Check `rdx5\version.txt` on any machine that updated its driver: a cached v4.6/v4.7
  addon faults on 616.64+, and other pre-v6 builds are untested there.

### DLSSTweaks (emoose) — **often bypassed**
Silently bypassed in Streamline games (`sl.interposer.dll` loads `_nvngx.dll` from the
driver store). Confirm `dlsstweaks.log` was written on the last launch. `ForceDLAA=true` is
disabled whenever `[DLSSQualityLevels]` is enabled. Prefer **NVIDIA App → DLSS Override →
DLAA**.

### Other managers (not used by this skill)
DLSS5-Swapper (rakanki911), DLSS5-Autopilot, DLSS5oneclick, DLSS-5-MANAGER. If a user
already ran one, **hash every runtime it deployed** — one was caught shipping an
unidentified HashMismatch "310.8.0". Beware lookalike repos with 0–1 stars copying these
names.

### AMD / Intel ports — out of scope
DLSS-NR-on-AMD (HIP, RDNA 3/4, alpha, ~30 fps at 1080p on a 9070 XT) and Arc experiments
exist. This skill does not support them; Gate 1 stops non-NVIDIA GPUs.
