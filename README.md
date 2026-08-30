# dlss5-installer

**A copy-paste skill that turns Claude (or any capable LLM with file access) into a
guided DLSS 5 Neural Rendering installer for your games.**

You paste one prompt, tell it which game, and it does what an experienced modder would:
checks your hardware, refuses anti-cheat games, figures out the game's graphics API,
installs the right stack (RenoDX DLSS5 + ReShade + DLSS5-Feeder or the DX11 bridge),
and then **verifies from the log files** instead of just saying "done!". If something
breaks, it debugs one variable at a time — and it can cleanly remove everything.

> 🌐 **Start here: [the project page](https://thunderruler.github.io/dlss5-installer/)** —
> it has the copy-paste prompt and a plain-English explanation.

## Quick start

### Option 1 — paste this into Claude (or your LLM of choice)

> Fetch https://raw.githubusercontent.com/ThunderRuler/dlss5-installer/main/SKILL.md and
> follow it exactly. This whole session is for installing DLSS 5 into my games.
> First game: **<game name>**

Works in Claude Code, Claude Desktop (with file access), or any assistant that can fetch
a URL and run commands on your PC.

### Option 2 — install it properly (one command)

```bash
npx skills add ThunderRuler/dlss5-installer
```

Installs into Claude Code and ~20 other agents (Cursor, Codex, Copilot, Gemini CLI,
Cline, Windsurf, Zed…) via the [Agent Skills](https://skills.sh) standard. Then just ask
to "get DLSS 5 working on <game>" and it triggers on its own — or invoke
`/dlss5-installer` directly in Claude Code.

Prefer git? `git clone` this repo into `%USERPROFILE%\.claude\skills\dlss5-installer`
works too.

## What you need

| Requirement | Why |
|---|---|
| **NVIDIA RTX 50-series** + driver 616.56+ | No confirmed report of DLSS 5 NR working on anything older |
| Windows + Steam/GOG/Epic games | Game Pass installs are locked and can't be modded this way |
| A single-player game **without anti-cheat** | The skill hard-refuses anti-cheat titles — injecting DLLs gets you banned |

## What to expect (honestly)

DLSS 5 Neural Rendering is **not an upscaler**. It's a neural detail/beauty pass that
costs roughly **half your framerate**. Every current path is effectively DLAA. People run
it because it looks striking, not because it's fast. The skill tells your assistant to
make sure you know this before installing anything.

## What's in the repo

```
SKILL.md                     the procedure the LLM follows (the skill itself)
references/
  components.md              every tool in the ecosystem and where to get it
  decision-tree.md           picking the right stack from the game's API + bitness
  install-playbook.md        exact file layouts for scenarios A/B/C/D
  troubleshooting.md         error codes, symptom → cause, real solved cases
  config-reference.md        every config key for every component
  known-hashes.md            good/bad neural-runtime hashes (a corrupt copy is common!)
  game-results.md            per-game results log — PRs welcome
  sources.md                 primary sources for everything claimed here
```

## Contributing game results

Got it working — or definitively *not* working — in a game that isn't logged yet?

**[Open a game result issue](https://github.com/ThunderRuler/dlss5-installer/issues/new/choose)**
and fill in the form. No git required, and it gets added to
[`references/game-results.md`](references/game-results.md) for you. Prefer to edit the file
yourself? That works too — see [CONTRIBUTING.md](CONTRIBUTING.md).

Failure reports are just as valuable as successes: knowing a game *can't* work saves the
next person an evening. Please include the proof line from your log rather than just
"it worked" — that's what makes the table trustworthy.

Two things that get declined, and they're why this project can exist publicly: **no
binaries** (above all `nvngx_dlssnr.dll`) and **no links to runtime downloads**.

## Credits & provenance

This skill orchestrates other people's excellent work:
[RenoDX](https://github.com/clshortfuse/renodx) (clshortfuse),
[DLSS5-Feeder](https://github.com/jlrouzies-fr/DLSS5-Feeder) (jlrouzies-fr),
[dlss5-dx11-bridge](https://github.com/NIGos/dlss5-dx11-bridge) (NIGos),
[iMMERSE LaunchPad](https://github.com/martymcmodding/iMMERSE) (Marty McFly),
[ReShade](https://reshade.me) (crosire), and
[RHI](https://github.com/RankFTW/RHI) (RankFTW).

**No NVIDIA binaries are included in this repo, ever.** The `nvngx_dlssnr.dll` runtime is
unreleased NVIDIA code that leaked from a game's early-access build. The skill only
locates and verifies copies already present on your machine — it never downloads the DLL
and instructs the assistant to refuse tampered (`HashMismatch`) copies.

Use at your own risk. Modding game files can break games; Steam's *Verify integrity of
game files* restores them.

## License

Documentation and skill text: [MIT](LICENSE). The tools it references have their own
licenses.
