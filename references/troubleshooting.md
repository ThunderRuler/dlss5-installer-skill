# Troubleshooting

`ReShade.log` next to the game exe is the primary diagnostic. Read it before changing anything.

## Error code table

| Code | Meaning | Fix |
|---|---|---|
| `0xbad00001` | `feature 18 create failed` on an **RTX 20/30/40** — the signed 310.8.0 (and the Lecram build) only create the neural feature on RTX 50. DLSS itself still runs, so everything else looks fine. | Use the pinned **SF-v2** cross-generation runtime (`E67DEE20…`, see `known-hashes.md`). |
| `0xBAD00002` | (a) `feature 18 create failed` — `nvngx_dlssnr.dll` is an **unrecognised modified** build (HashMismatch and not a pinned hash). (b) Feeder 1.18+: `SuperSampling.FeatureInitResult` `0xBAD00002 (PlatformError)` — NGX cannot run **in the game's process** (e.g. emulators); the Feeder's 64-bit helper mode (`dlss5-feed-helper.addon64`) is the route | Replace with a pinned copy for this GPU. See below. |
| `0xBAD00005` | Either (a) DLSS output carries a **mip chain** the add-on needs single-mip, or (b) the runtime **rejected the low-resolution colour contract** when NR upscaling was attempted | (a) update NR add-on to ≥ v3.3.5, which handles mipped outputs; (b) leave `NREnableUpscaling=0` — upscaling is blocked on some runtime builds |
| `DLSS output format 9 is not a supported typed codec format` | Game allocates its DLSS output as `R16G16B16A16_TYPELESS`; the add-on needs a **typed** resource | Per-game typeless→typed upgrade addon, à la `dltb-dlss5-fix` |

## Symptom → cause

### Home does nothing / no overlay / "RenoDX isn't installed"

Nothing is loading ReShade. See [README](README.md#the-single-most-common-failure).

Check in order:
1. Is there a proxy DLL named after something the game imports? (`dxgi.dll` for D3D11/12)
2. Is it next to the **exe that actually runs**? Some games launch from `bin\` or `bin\x64\`.
3. Is `ReShade.log` real content, or the "ReShade was not loaded by the game" placeholder?
4. If OptiScaler is meant to chain-load it: is OptiScaler actually present, and is
   `LoadReshade = true`? (`auto` means **false**.)

### Overlay opens but no add-ons listed

Bitness or location mismatch. ReShade searches next to the DLL that loaded it, and only
matches `.addon64` in 64-bit processes, `.addon`/`.addon32` in 32-bit. An `.addon64` in a
game root while ReShade loaded from `bin\` will never be found.

### `DLSSNR: STANDBY/FAILED`

Ranked by likelihood:

1. **Wrong or unrecognised runtime** — hash it and compare with `known-hashes.md`:
   ```powershell
   Get-FileHash "...\nvngx_dlssnr.dll" -Algorithm SHA256
   ```
   Not pinned → replace it with the pinned build for this GPU. RHI caches copies under
   `%LOCALAPPDATA%\RHI\DLSS-NR\<version>\` (hash those too). On RTX 50,
   `dlssnr-signature-repair` can pull a Valid copy from another game you own — **do not**
   point it at the SF-v2 build on a 20/30/40 card; that one is modified on purpose.

1b. **Old consumer on a new driver.** Driver **616.64+** with `renodx-dlss5` **v4.6 or
   v4.7** (the "lazy-adoption" builds) faults inside `nvngx_dlssnr.dll` (Feeder issue #54):
   the evaluate crashes or returns nothing. Classic v4.5 passes the Feeder's test, but
   classic **v4.1.5 degraded silently** in Death Stranding DC on 616.92 (NR on ~1
   frame/sec — see "Flicker" below); v5.x is untested. Check the addon version (RHI:
   `%LOCALAPPDATA%\RHI\rdx5\version.txt`;
   the load banner in `ReShade.log`). Upgrade to v6.1+ — this silently breaks installs that
   worked in August as soon as the user takes a Game Ready driver.

   *Back up first, and note that a `.original` backup may be equally corrupt — check its
   hash too rather than assuming it is clean.*

2. **Mip chain on the DLSS output** (`0xBAD00005`, guides shown as `0x0`, thousands of
   attempts and zero frames) — update the NR add-on to ≥ v3.3.5.

3. **Typeless output format** — needs a per-game fix.

4. **RTX 20/30/40 with the RTX-50-only runtime** — look for `0xbad00001` in `ReShade.log`
   (or `host64\ReShade.log`). Fix: the pinned SF-v2 runtime.

5. **Two consumers installed** (e.g. RenoDX addon + OptiScaler DLSS-NR, or + Deep Fried
   Chicken). One goes silently inert (Chicken reports `DISARMED` / `CONFLICT`). Keep
   exactly one.

### Effect covers only part of the screen (e.g. the top ~58%)

**Guide/colour resolution mismatch.** The neural feature is created expecting full-res
guides but evaluated with render-res ones. Look for these two lines disagreeing:

```
feature 18 created ... with guides 2560x1440
inline feature 18 evaluation succeeded (... guides 1485x835)
```

Do the arithmetic: `835 / 1440 = 0.58`, and the affected band is the top ~58% of the frame.
It matches the DLSS ratio the game is rendering at (0.58 = Balanced).

**Cheap falsifiable test:** raise in-game DLSS one quality step (0.58 → 0.667). If the band
grows to ~67% of screen height, confirmed.

**Fix:** make render resolution equal output resolution — DLAA — while keeping DLSS
*enabled*. NVIDIA App → DLSS Override – Super Resolution → DLAA.

Do **not** just turn DLSS off; the add-on hooks the game's NGX call and needs it running.

Fallback if DLAA isn't available: the guide-related knobs are `NRComputeScalingRatioC`,
`NRMVecScaleX`, `NRMVecScaleY`, `NRDepthMode`.

### Image doubles or smears during motion (Feeder only)

Motion vector sign is wrong. Flip the **MV_SIGN** component in the `DLSS5_Feed.fx` UI.
Use the "DLSS 5 Feed – debug view" technique to inspect: static areas grey, motion coloured.
`mv_scale_x` / `mv_scale_y` in `dlss5-feed.cfg` are extra multipliers.

### Feeder logs "DLSS5_Feed.fx is not loaded (technique/textures missing)"

Either the shader isn't installed, or the technique order is wrong. `DLSS5_Feed` must be
enabled **below** the motion-vector provider (`LUMENITE: Kernel 2.0` for provider 3,
`MartysMods_Launchpad` for provider 1). Also confirm Generic Depth is enabled with the
right depth buffer picked.

If the game already has native DLSS, you don't need the Feeder at all — this warning is
just noise and the addon can be removed.

### Effects reload constantly / swapchain churn

Exclusive fullscreen. Use windowed or borderless — the Feeder README calls out exclusive
fullscreen swapchain churn causing repeated effect reloads.

## Benign log noise — safe to ignore

- `Failed to find NVSDK_NGX_D3D12_EvaluateFeature_C` — one hook target of several missed;
  the others take and evaluation still succeeds.
- `CreateFeature was not intercepted ... registering lazily from evaluate contract` — the
  game created its DLSS feature before hooks were installed. It recovers.
- `Ignoring LoadLibrary('...') call to avoid possible deadlock` — normal ReShade behaviour.
- `warning X4000: use of potentially uninitialized variable` in SweetFX shaders — cosmetic.

### Flicker, or NR on only part of the image, while the log says "succeeded"

Not the guide/colour mismatch above if `guides` equals the output size. Check the
**evaluation rate**: the RenoDX addon logs milestone counts only (`count=1`, `60`, `600`…).

```powershell
Select-String ReShade.log -Pattern 'evaluation succeeded \(count=' | % { $_.Line.Substring(0,60) }
```

- Milestones arrive at frame rate (`1` → `60` in about a second) → NR runs every frame;
  look elsewhere (below).
- `1` → `60` takes tens of seconds → NR is landing on occasional frames only, which the
  eye sees as flicker and partial coverage. Seen with classic v4.1.5 on driver 616.92.
  Fix: addon **v6.1+** (v7.0.0-rc8 restored the full rate in the same game).

If the rate is fine and it still flickers (Death Stranding DC, 2026-09-29 — open), suspects
in order, one per launch: hand-tuned `[RenoDX.DLSS5]` values (especially non-default
`NRMVecScaleX/Y`, which mis-scale the motion vectors the model reprojects with, and
`NRAutoMask`) — reset to defaults; then the game's DLSS mode vs. forced DLAA; then the
hook point (`NRHookPoint`, v8).

## Warnings that matter

- `custom runtime accepted; untested build, NR failures may be specific to it` — the
  `nvngx_dlssnr.dll` is not a clean signed NVIDIA build. Suspect it first for any weird
  output, and hash it against `known-hashes.md` (the SF-v2 build fails Authenticode by
  design, so the signature alone tells you nothing).
- `NR upscaling fell back to native: the signed runtime rejected the low-resolution colour
  contract ... Upscaling cannot engage with this runtime build` — keep
  `NREnableUpscaling=0`. Not a bug you can fix from config.

## Things that quietly break a working install

- **RHI's global addon set** redeploys addons on refresh — deleting files by hand doesn't stick.
- **RHI drag-and-drop** of `renodx-dlss5.addon64` deletes the game's existing per-game
  RenoDX addon (e.g. its HDR fix).
- **Deleting `dxgi.dll`** takes the entire stack down; the addons then sit there inert.
- **DLSSTweaks is bypassed in Streamline games** — check `dlsstweaks.log`'s timestamp
  against your last launch before believing any of its settings apply.
- `ForceDLAA=true` is overridden whenever `[DLSSQualityLevels]` is enabled.

### `HOOKS ARMED - NO DLSS CREATE SEEN` (Streamline games)

Panel shows the runtime is fine and hooks are installed, but the game never creates a DLSS feature:

```
DLSSNR v310.8.0: HOOKS ARMED - NO DLSS CREATE SEEN
Runtime sha256: E16BCF15... (reference match)
NGX modules detoured: 3 | core present: yes
NGX hooks: creates 0 | evaluations 0
Streamline: DLSS/DLSSD evaluations 0 | all evaluations 0
```

Note `creates 0`, not "created but no evaluations" - the addon distinguishes these, and this is the
earlier state.

**Cause 1 - the game routes DLSS through NVIDIA Streamline**, which the addon leaves alone by
default. Its banner says so: `EnableHooks=2: NGX hooks only, Streamline modules left unpatched`.
Verbatim from the addon binary:

> "If the game uses NVIDIA Streamline, set EnableHooks=1 in the [RenoDX.DLSS5] section of
> ReShade.ini and restart; otherwise this title is not supported and NR stays off"

Fix - add to `reshade.ini`:
```ini
[RenoDX.DLSS5]
EnableHooks=1
```
**Set this only while the game is closed.** ReShade rewrites `reshade.ini` on shutdown and will
erase the line if you add it mid-session.

Also from the binary, the risk and the fallback:

> "WARNING: Streamline hooks ON (EnableHooks=1). Contested patch site - if the game crashes at
> boot, set EnableHooks=2 (NGX-only still covers Streamline calls)."

So: **crash at boot after setting 1 = expected failure mode, go back to 2.** `EnableHooks=0` is
safe mode / NR disabled by policy.

How to spot a Streamline game before you start: look for `sl.interposer.dll`, `sl.dlss*.dll`. In UE
titles they live under `<Game>/Plugins/StreamlineCore/Binaries/ThirdParty/Win64/`, **not** next to
the exe. UE games also keep DLSS itself in `<Game>/Plugins/DLSS/Binaries/ThirdParty/Win64/`.

**Cause 2 - the wrong upscaler is selected in-game.** UE titles often ship DLSS, FSR *and*
StreamlineCore plugins side by side. With FSR or TSR selected, NGX is never touched and no hook
setting will help. Check the in-game upscaler dropdown says DLSS.

**Useful confirmation:** when the runtime is the correct one, the panel prints
`Runtime sha256: E16BCF15... (reference match)`. "(reference match)" is the addon telling you the
DLL is the expected build - stronger evidence than an Authenticode check alone.

### "No effect files (.fx) found" on the Home tab

Not an error for Scenario A installs. That message is about **shaders**, and a game running only
`renodx-dlss5.addon64` needs none. The add-on lives on the **Add-ons** tab. Only the Feeder
(Scenario C/D) needs `reshade-shaders\`.

### `CreateFeature raised exception 0xC0000005` (Feeder, D3D12)

The Feeder reaches a fully working NGX session and dies on the last call:

```
[feed] NVSDK_NGX_D3D12_Init -> 0x00000001 (Success)
[feed] NGX capabilities: SuperSampling.Available=1
[feed] session ready (same-device)
[feed] building: 2560x1440 backbuffer R10G10B10A2_UNORM
[feed] CreateFeature raised exception 0xC0000005 (caught; nothing was submitted)
[feed] failure: resource build
```

This is an access violation *inside the neural add-on*, not a config error. Everything upstream
is correct - init succeeded, the capability bit is set, the session opened on the game's device.

**Prime suspect: a 10-bit backbuffer.** Note `R10G10B10A2_UNORM`. Every confirmed-working Feeder
title (INSIDE, Gunner HEAT PC, Cities Skylines) hands it an 8-bit RGBA buffer. ReShade corroborates
independently with `texture does not support sRGB sampling (back buffer format is not RGBA8)` from
the SweetFX shaders.

Not fixable from config. Feeder version is *not* the cause - verified on Routine by installing a
matched v0.3.0 addon+shader pair (MD5 4183E486) and reproducing the identical exception at the
identical step.

**Caution:** the addon's banner may keep printing an older version string (`[DLSS 5 Feed 0.2.0]`)
after a genuine upgrade. Verify the swap by **file hash**, never by the log banner or by file size -
v0.2.0 and v0.3.0 are both exactly 83,968 bytes.

### The DX11 bridge hooks D3D11 entry points only - useless in a D3D12 process

`dlss5-dx11-bridge` prints a confident, healthy-looking hook report that means nothing if the game
runs D3D12. Read the entry point names, not the `hooked: ... = yes` line:

```
NVSDK_NGX_D3D11_CreateFeature     = 00007FFE104D5160   <- D3D11!
NVSDK_NGX_D3D11_EvaluateFeature   = 00007FFE104D52C0
hooked: CreateFeature=yes EvaluateFeature=yes
```

Six layers can all report `hooked: yes` while the game calls `NVSDK_NGX_D3D12_CreateFeature`, which
the bridge never touches - producing `creates 0` with a log that looks perfect. If ReShade.log says
`Redirecting D3D12CreateDevice`, remove the bridge; it is by definition the wrong tool.

### UE5: the game can delete your Engine.ini

Some UE titles sanitise `%LOCALAPPDATA%\<Game>\Saved\Config\Windows\` on boot. Routine deleted a
freshly written `Engine.ini` and rewrote `GameUserSettings.ini` in the same second, so the
`[SystemSettings]` cvars never applied and the test silently never ran.

Set the file **read-only** (`attrib +R`) so the delete fails, and always re-read the file after the
next launch to confirm it survived.

### UE: DLSS lives inside the temporal-upscaler slot

The NVIDIA UE plugin registers as a temporal upscaler. With the game's AA method set to **NONE**
(`r.AntiAliasingMethod=0`) there is no upscale pass at all, so the plugin is never asked to create a
feature - `creates 0`, no matter how well the hooks are armed. A game that force-enables TAA when you
tick DLSS is showing you this dependency.

UE settings may not live in `GameUserSettings.ini`. Routine keeps them in
`Saved\SaveGames\SETTINGS\USERSETTINGS.sav`; the persisted value is readable as a raw FName string:
```bash
grep -a -o -E "ERoutineAAMethod::[A-Za-z0-9_]+" USERSETTINGS.sav
```

### Background jobs do not survive a tool call

A `Start-Job` that waits for a game to exit is destroyed when the invoking shell exits. If a file is
locked, ask the user to close the game and do the copy directly - a queued swap that never runs
leaves a **mismatched addon/shader pair**, which is worse than either version alone.

---

# Added 2026-08-30 (second session)

### `could not open included file 'ReShade.fxh'` — Feeder shader fails to compile

```
Failed to compile 'DLSS5_Feed.fx':
  (29,1): preprocessor error: could not open included file 'ReShade.fxh'
```

**DLSS5_Feed.fx v0.5.0+ `#include`s `ReShade.fxh`; v0.3.0 and earlier had no includes at
all.** If you assembled `reshade-shaders\` by copying a minimal Feeder-only folder from
another game, the standard ReShade headers are missing and only the new shader breaks —
LaunchPad compiles fine, which makes it look like a Feeder-specific fault.

Fix: copy a **complete** ReShade shader set (`ReShade.fxh`, `ReShadeUI.fxh`, `Macros.fxh`,
…), then put the matching `DLSS5_Feed.fx` back on top. RHI-managed `reshade-shaders\`
folders already contain the headers.

The overlay shows this as a red `[DLSS5_Feed.fx] failed to compile` on the Home tab, and
`dlss5-feed.log` says `technique MISSING`.

### Documentation drift: the README on `main` describes the *unreleased* build

The Feeder's GitHub README documents the tip of `main`, which can be a beta ahead of the
release you installed. Configuring a released build against those docs produces settings
that silently do nothing.

Caught on BeamNG: `DLSS5_MV_PROVIDER` is documented in the README but **does not exist in
v0.5.2** — the shader has zero occurrences of it. Every value set landed on a shader that
never reads it, and the log kept reporting `MV provider none`.

**Check the feature exists in the file you actually deployed:**
```bash
grep -c "DLSS5_MV_PROVIDER" DLSS5_Feed.fx     # 0 = your build has no provider system
```

Releases can also move fast — three Feeder releases were published on a single day.

### Preprocessor definitions: the preset overrides the global

ReShade reads preprocessor definitions from **both** `ReShade.ini` `[GENERAL]` and the
**preset file**, and the preset wins. Setting `DLSS5_MV_PROVIDER` only in `ReShade.ini` can
be silently overridden. Set it in **both**, or use the overlay's per-effect *Preprocessor
definitions* panel, which is what the Feeder README prescribes.

### Effects reload repeatedly — temporal history keeps resetting

`dlss5-feed.log` showing many `effect runtime ... destroyed / initialised` cycles means the
swapchain is being recreated. Every reload wipes the optical-flow history, so motion vectors
restart from zero and the image smears in bursts.

Usual cause: **exclusive fullscreen**. Use windowed or borderless.

### Technique names are case-sensitive and rarely match the display name

A preset naming a technique that does not exist produces:
```
Preset '...\ReShadePreset.ini' uses unknown technique 'X@Y.fx'
```
The display name in the overlay is *not* the technique identifier. Read the declaration:
```bash
grep -nE "^\s*technique\s+" shader.fx      # e.g. "technique Lumenite_Kernel <"
```
LumeniteFX Kernel shows as "LUMENITE: Kernel 2.0" but is declared `Lumenite_Kernel`.

### NGX reports `SuperSampling.Available=0` and blames your GPU

```
NGX capabilities: SuperSampling.Available=0
DLSS super sampling is not available on this GPU/driver
```

Misleading on an RTX 50-series card. **NGX resolves its feature DLLs from the running
process's own directory.** If `nvngx_dlss.dll` / `nvngx_dlssnr.dll` are in the game's root
while the exe runs from `Binaries\Win64`, NGX cannot see them.

Fix: put **every** file next to the exe that actually runs. Deploy `nvngx_dlss.dll` even
for games with no DLSS of their own.

### UI ghosts/smears while the world looks fine

`NRUICorrection` and `NRAutoMask` in the `renodx-dlss5` panel are **both off by default**.
Turn them on in the overlay (not by editing `reshade.ini` mid-session — ReShade rewrites it
on shutdown). See `motion-vectors.md` for the `UIMask.fx` fallback.

## ReShade silently reverts `ReShade.ini` edits made while the game is running

**Symptom:** you change a setting in `ReShade.ini` — most painfully
`PreprocessorDefinitions=DLSS5_MV_PROVIDER=N` — relaunch, and nothing changed. The new
provider's shader never appears in `ReShade.log` at all.

**Cause:** ReShade holds its own in-memory copy of `ReShade.ini` and **writes it back out
on exit**. Any edit you make to that file while the game is running is silently
overwritten by the stale in-memory version when the process closes. Your change is gone
before the next launch reads the file.

**Confirm it in two seconds** — compare mtimes after closing the game:

```bash
stat -c '%y  %n' "<gamedir>/ReShade.ini" "<gamedir>/dlss5-feed.log"
```

If `ReShade.ini`'s timestamp matches the moment the game *exited* rather than the moment
you edited it, ReShade rewrote it and your change is lost.

**Rule: only edit `ReShade.ini` with the game fully closed.** Verify the value after
closing, not after editing.

Note the asymmetry that makes this confusing: `ReShadePreset.ini` is **not** rewritten on
exit unless something changed it in the overlay. So a provider set in both files ends up
**mismatched** — preset says 3, global says 1 — which looks like the
preset-overrides-global problem and sends you debugging the wrong file.

Two related traps in the same family:

- A uniform changed in the overlay (`MV_SIGN`, `MV_SCALE`) is saved to the *preset*, not
  to `ReShade.ini`. If it did not persist, it was never committed — ReShade only writes
  the preset when a value actually changes.
- `DLSS5_MV_PROVIDER` is a **preprocessor define**, so it needs a full effect recompile.
  Changing it cannot take effect live; the game has to be restarted.

# Added 2026-09-29 (September ecosystem changes)

### The old bridge keeps loading after an upgrade

The bridge was renamed `dlss5-dx11-bridge` → `dlss5-bridge`. ReShade loads **both** addon
files if both are present, and the old one wins. Delete `dlss5-dx11-bridge.addon64` (and,
once `dlss5-bridge.cfg` exists, the old `.cfg`). The top of `dlss5-bridge.log` lists the
SHA-256 of every addon loaded — two bridge entries means this.

### Feeder on a D3D12 game does nothing and says the game has its own DLSS

Since v1.17.0-beta.2 the D3D12 same-device path refuses when the game has loaded
`nvngx_dlss.dll`, `nvngx_dlssd.dll` or `sl.dlss.dll`. That is deliberate: the game has
engine motion vectors, so it belongs on **Scenario A** (consumer alone) or **E**
(OptiScaler). `native_dlss_ok=1` overrides, but only do that with a reason.

### 32-bit: `the host in host64\ speaks protocol vX, this add-on vY`

`dlss5-feed.addon32` and `host64\dlss5-feed-host64.exe` came from different releases.
Replace both from the same zip. Related: a `dlss5-feed.addon64` placed **inside** `host64\`
is refused — the helper only takes the consumer.

### Agility SDK: `0x887E0003`

A game shipping its own `D3D12\` Agility SDK folder can conflict with the private D3D12
device. Test by renaming the game's `D3D12` folder to `D3D12_off`; revert if it changes
nothing.

### Vulkan: black screen, freezes or lost frames with NVIDIA Smooth Motion

Smooth Motion must be **off for Vulkan** games under the Feeder (NVIDIA Profile Inspector:
setting `0xB0CC0875` is a sum — 1 = DX12, 2 = DX11, 4 = Vulkan — so remove the 4, e.g.
`7` → `3`). It is compatible with D3D11/12 since Feeder
0.11.0-beta.2. Luma mods and `NRStyle=2` have also crashed under Smooth Motion.

### `### CRASH RECORDED ###` in `dlss5-feed.log`

The Feeder wrote `dlss5-feed-crash.dmp` next to the exe. Attach it to a report; don't retry
blindly.

### Stutter with Deep Fried Chicken as the consumer

Chicken ≤ 1.4.7 stutters; set `stall_log_ms` low to confirm stalls in the log, and update
Chicken or switch consumer.

### The bridge's substitute path (`synth=1`) looks soft and smeary

Expected — the bridge README says so. For games without DLSS use the Feeder instead.

## Scam and lookalike downloads (tell the user once, early)

- **`dlss5bridge.com` is not the bridge project.** Its installer has been flagged malicious
  (it tries to add Microsoft Defender exclusions). The real bridge has no installer.
- **Fake DLSS5-Feeder sites and copycat repos** distribute lookalike ZIPs and "installer
  .exe" files. The Feeder ships no `.exe`; the only real source is its GitHub Releases page.
- Lookalike repos with 0–1 stars copying popular names (Feeder, Swapper, "one-click",
  "NBA 2K27 free") have been seen. Check the owner matches `components.md` exactly.
- Any tool that asks to **add a Defender exclusion** is a red flag. (The Feeder installer
  asks before excluding Deep Fried Chicken, which Defender often quarantines — the skill
  doesn't install Chicken, so decline.)
