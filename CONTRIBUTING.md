# Contributing

The most useful thing you can add is **a game result** — whether DLSS 5 worked in a game,
and the log line proving it. Failures are just as valuable as successes: knowing a game
*can't* work saves the next person an evening.

## Adding a game result

### The easy way (no git required)

**[Open a game result issue](https://github.com/ThunderRuler/dlss5-installer/issues/new/choose)**
and fill in the form. It gets added to the results log for you. This is the preferred
route — you don't need a GitHub account beyond signing in, and you can't get the
formatting wrong.

### The direct way (a pull request)

If you'd rather edit the file yourself, open
[`references/game-results.md`](references/game-results.md), click the **pencil icon**, and
add a row to the *Community results* table. GitHub forks the repo for you automatically
and turns your edit into a pull request — you never touch a command line.

Keep to the existing table format, and include the proof line from your log.

## What gets declined

Two hard rules, and they're the reason this project can exist publicly at all:

- **No binaries, ever.** No `.dll`, `.exe`, or `.addon64` files in the repo — above all
  `nvngx_dlssnr.dll`, which is unreleased NVIDIA code that leaked from a game build.
  Hosting it would get the whole repo taken down.
- **No links to runtime downloads.** No mirrors, no file hosts, no "here's where I got
  mine". The skill deliberately only locates and hash-verifies copies already on the
  user's own machine. Linking an upstream project's own releases page (RenoDX, ReShade,
  the Feeder) is fine.

Also declined: results from anti-cheat or multiplayer games. The skill refuses those
because DLL injection gets people banned, and the results log shouldn't imply otherwise.

## Improving the skill or references

Corrections to `SKILL.md` or anything in `references/` are welcome, particularly:

- a troubleshooting symptom → cause pair that isn't documented yet
- a config key that behaves differently than described
- an error code with a known fix

Say how you verified it. "The log said X and changing Y fixed it" is worth far more than
a plausible-sounding theory — the whole point of this project is evidence over guesswork.

## Reporting that the skill misbehaved

If your assistant did something wrong while following the skill — declared success when
it hadn't worked, missed an anti-cheat game, installed to the wrong folder — that's a bug
worth an issue. Include what it did and which step it was on. Those reports improve the
instructions for everyone.
