# Sources

All researched 2026-08-30. The GitHub READMEs are the authoritative documentation — the
news coverage is context only.

## Primary tools

| Source | Good for |
|---|---|
| https://github.com/clshortfuse/renodx | The RenoDX framework. The DLSS 5 addon ships via its `snapshot` release tag: `releases/download/snapshot/renodx-dlss5.addon64`. The main README does **not** document the DLSS 5 addon — that lives in the Discord and Nexus. |
| https://github.com/NIGos/dlss5-dx11-bridge | D3D11 bridge. Excellent README: full cfg table, confirmed games list, limitations, diagnostic workflow via `stage`. |
| https://github.com/jlrouzies-fr/DLSS5-Feeder | Feeder for games with no DLSS. The single best document in the ecosystem — exact file layouts for 64- and 32-bit, full cfg table, shader chain order, explicit API support/non-support statements. |
| https://github.com/NIGos/dlss5-d3d12-fix | Mip-chain fix. Mostly obsolete (superseded by NR v3.3.5) but documents the `0xBAD00005` / guides-`0x0` symptom set. |
| https://github.com/markitzeroo/dltb-dlss5-fix | The per-game fix pattern: typeless→typed output upgrade, RenoDX-based. Read this before writing any new per-game fix. |
| https://github.com/kayle2203/dlssnr-signature-repair | The `0xBAD00002` / HashMismatch signature repair. |
| https://github.com/martymcmodding/iMMERSE | LaunchPad motion vectors — a hard dependency of the Feeder. |
| https://github.com/RankFTW/RHI · [releases](https://github.com/RankFTW/RHI/releases) | The installer/manager. Release notes are where the per-game addon picker, DLL rename dropdowns and Neural Rendering column are documented. |
| https://reshade.me | ReShade. **Must be the Addon build** (`ReShade_Setup_X.X.X_Addon.exe`). |
| https://github.com/beeradmoore/dlss-swapper | Clean source for `nvngx_dlss.dll`. |

## Community / context

| Source | Notes |
|---|---|
| https://www.nexusmods.com/site/mods/2224 | "Applying RR and DLSS 5 RenoDX for the games" — the main community how-to. **Blocks automated fetching (HTTP 403); open in a browser.** |
| https://www.nexusmods.com/control/mods/140 | Control — the first game DLSS 5 was ported to; the reference implementation. |
| https://www.nexusmods.com/deathstranding/mods/95 | RenoDX HDR fix for Death Stranding DC. Warns that OptiScaler / SpecialK / PureDark FG may conflict, since all hook at a low level. |
| https://github.com/zhubaohi/FF7R-DLSS5 | One-click DLSS 5 installer for FF7 Rebirth — example of a per-game packaging approach. |
| RenoDX Discord | Where the newest builds and game-specific implementations actually circulate. Not indexable. |
| https://videocardz.com/newz/breaking-modders-unlock-experimental-nvidia-dlss-5-hours-after-dll-discovery | Origin story: the DLL discovery in NBA 2K27. |
| https://wccftech.com/nvidias-dlss-5-cracked-into-control-hours-after-modders-found-a-hidden-dll-inside-nba-2k27/ | Same event, more detail. |
| https://wccftech.com/nvidia-dlss-5-neural-rendering-in-10-modern-games-the-best-unofficial-dlss-5-on-vs-off-comparisons-so-far/ | ON vs OFF comparisons across 10 games. |
| https://windowsforum.com/windows-news.4/dlss-5-runtime-cuts-rtx-5070-ti-control-4k-fps-in-half.443725/ | The ~50% FPS cost on an RTX 5070 Ti — same card as this rig. |

## Key verbatim quotes worth keeping

> ~~"DX9 / DX10 / Vulkan won't work."~~ — DLSS5-Feeder README, **superseded 2026-08-30**

The current README opens instead with: *"DLSS 5 neural rendering in games that ship without
any DLSS — D3D11, D3D12, Vulkan, 32-bit, even DirectX 9."* Vulkan gained native support in
v0.5.1, and D3D9 works through a dgVoodoo2 wrapper. **Only D3D10 remains unsupported.**
Treat any older quote about API support as historical.

> ~~"NGX is 64-bit only. DX9 / DX10 / Vulkan / 32-bit not supported."~~ — **superseded**;
> NGX itself is still 64-bit only, which is why 32-bit games need the `host64` helper.
> (the 32-bit part refers to the NGX runtime; the Feeder works around it with a `host64` helper)

> "A game whose D3D9 calls are wrapped to D3D11 works fine" — DLSS5-Feeder README, on BioShock Remastered

> "render resolution equals output resolution equals game backbuffer; no upscaling
> performance gain yet" — DLSS5-Feeder README, on the DLAA-only limitation

> "Every report of the neural pass never starting has come from RTX 40-series hardware, and
> there is so far no report of DLSS 5 neural rendering working below RTX 50-series."
> — dlss5-d3d12-fix

> "If anything goes wrong the bridge disables itself and the game renders on its own; it
> never leaves a broken frame on screen deliberately." — dlss5-dx11-bridge README

## Provenance caveat

RenoDX, the bridge, the Feeder and the fixes are open source and readable. The
`nvngx_dlssnr.dll` runtime is **not** an official NVIDIA distribution — it leaked from an
early-access build. Copies circulating in YouTube-linked archives are worth scanning
yourself, and a `HashMismatch` signature means the file was modified after NVIDIA signed
it. Prefer a copy that verifies as Authenticode **Valid**.
