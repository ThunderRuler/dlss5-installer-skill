# Motion vectors — the thing that decides Feeder image quality

Applies to **Scenario C/D only** (DLSS5-Feeder). Games with native DLSS get exact per-pixel
velocities from their own engine and need none of this.

The Feeder does not estimate motion itself. It reads the output of a motion-vector shader
**you** install. Get this wrong and DLSS runs with zero or wrong vectors, which looks like
heavy smearing, ghosting and distant shimmer — the single biggest source of "it works but
looks bad".

## Choosing a provider

Fixed **at compile time** by the `DLSS5_MV_PROVIDER` preprocessor definition
(ReShade overlay → select `DLSS5_Feed.fx` → *Preprocessor definitions* → reload effects).

| Value | Provider | Enable this technique | Notes |
|---|---|---|---|
| `0` *(default)* | anything writing shared **`texMotionVectors`** — qUINT, `dh_uber_motion` | that shader's own | The old convention. **DRME does not compile on ReShade 6.8.** |
| `1` | **iMMERSE Launchpad** | `Launchpad` | Full resolution, image-space. Warping around flames/transparents is worst here. |
| `2` | **VORT** | `vort_Motion` | MIT. |
| **`3`** | **LumeniteFX Kernel** ← **author's recommendation** | `Lumenite_Kernel` | 1/8-res flow **plus a confidence map** the feed uses. The config the beta was tuned on. |
| `4` | LumeniteFX QuantMotion | `LUMENITE: QuantMotion` | Same shape, different estimator. |

Rules that apply to all of them:

- The provider's technique must be **enabled and above `DLSS 5 Feed`** in the technique list.
- Only **one** provider enabled at a time.
- Compiling for one provider while a different one is enabled is the classic silent failure.
  The addon warns in the overlay and log.
- Nothing is bundled — install the provider yourself.

**LumeniteFX** (https://github.com/umar-afzaal/LumeniteFX) — note the default branch is
`mainline`, not `main`; raw URLs against `main` silently return a 14-byte error page.
Kernel needs `Shaders/lumenite_Kernel.fx`, the four `Shaders/include/lumenite_*.fxh`
headers, and `Textures/lumenite_bluenoise256.png`.

## Reading the diagnostics

v0.6+ prints the resolved provider and probes the actual vectors:

```
DLSS5_MV_PROVIDER=3 (LumeniteFX Kernel) -> Lumenite_Kernel (enabled)
MV probe (centre 64x64, frame 15000): mean |mv| 1.759 px, max 2.26 px, 99% non-zero
```

- `-> none (not installed)` — the shader compiled for a provider whose technique is absent
  or disabled.
- `MV provider none` + *"no known texMotionVectors provider found ... motion vectors will be
  zero (still images only)"* — the shader is on provider `0` and nothing writes that texture.
  **This is what a heavily smearing image usually means.**
- `0% non-zero` while the camera is moving — vectors are absent. If the camera was *still*,
  zero is correct; check before diagnosing.
- Implausibly small magnitudes (`mean 1.7 px` at 1440p while moving fast) — the provider is
  producing degenerate flow, often because ReShade's **depth buffer is unavailable**, which
  matters most for depth-based estimators like Kernel.

`DLSS5_Feed_Debug` renders motion as colour — static areas grey, motion coloured. Use it
rather than guessing.

## Quality knobs (v0.6+)

| Setting | What it does |
|---|---|
| `GEOM_ENABLE` | **Derives static-world vectors** by fitting camera motion from flow + depth each frame, instead of trusting the provider. Big win in scenes that are mostly static world with a moving camera. Off by default. |
| `GEOM_AGREE_PX` / `GEOM_OUTLIER_PX` | How closely the provider must agree with the fitted model before it is trusted. |
| `GEOM_DYNAMIC_MARGIN` | Margin around genuinely moving objects. |
| `MASK_STRENGTH`, `GEOM_MASK_REJECTED` | Writes `DLSS5_Mask`, telling DLSS to **favour the current frame** where the vector was rejected. Aimed at ghosting on rotating/occluded objects. |
| `VALIDATE_LUMA` | Luma test used to reject bad vectors (mask only). |
| `MV_SCALE`, `MV_SIGN` | Diagnostic multipliers. Flip `MV_SIGN` if the image doubles or smears *with* motion. |

## Where this approach runs out

Reconstructed motion vectors are an approximation. They hold up well in slower scenes with
distinctive features, and degrade badly with:

- **fast near-camera motion** that exceeds the estimator's search radius
- **repetitive texture** (asphalt, water, foliage) — the aperture problem
- **rotation** (spinning wheels) — not translational motion at all
- **screen-space UI** over a moving background — no real motion to estimate

**Driving sims at speed hit all four at once.** BeamNG.drive was tested end-to-end
(see game-results) and the honest conclusion is that no amount of provider tuning fixes
it: the game needs engine motion vectors. If a game in this category later ships native
DLSS, it becomes a clean Scenario A/B install and this whole problem disappears.

## UI smearing

Separate problem, separate fix. The neural pass reprojects screen-space UI using world
motion vectors, which ghosts it. Two levers, **both off by default**, in the
`renodx-dlss5` panel:

- `NRUICorrection` — the UI correction pass
- `NRAutoMask` — automatic masking

Set them in the addon's overlay panel, **not** by editing `reshade.ini` mid-session:
ReShade rewrites that file on shutdown and will discard your edit.

Deterministic fallback: `UIMask.fx` ships with ReShade's standard shader set. You paint a
mask over the screen regions holding UI and those pixels are excluded outright. Manual, but
exact — and fine for games whose HUD sits in fixed positions.
