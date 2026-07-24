---
name: calibrate-lut
description: Use when asked to build, fit, or calibrate a color-correction LUT from a photo of a ColorChecker calibration chart, e.g. "build a LUT from this ColorChecker photo" or "color-calibrate this camera against the chart shot". Covers both the traditional 24-patch ("classic") chart and the Calibrite ColorChecker Passport Video 2's video-production page.
---

# /calibrate-lut

Builds a 3D `.cube` correction LUT from a single photo of a ColorChecker
calibration chart, via one of two `premiere-ai` commands (from the
[premiere-ai](https://github.com/stefanwebb/premiere-ai) package) — chart
detection (SAM3) and patch measurement are shared, but the two commands
fit fundamentally different corrections and are **not interchangeable**:

| Command | Chart page | Fits |
|---------|-----------|------|
| `calibrate-lut` | Video-production page (Calibrite ColorChecker Passport Video 2) | Per-channel (R/G/B) tone curves for exposure/white-balance + a single global hue rotation for chromatic/skin-tone chips |
| `calibrate-lut-classic` | Traditional 24-patch page (X-Rite/Calibrite Classic) | A color-correction matrix (3x3 + offset, or an exposure-invariant root-polynomial) fit against published reference sRGB values |

## Which one to use

Ask which physical chart page was photographed if it isn't already clear
— these are two different pages of the same physical target (or two
different products entirely) requiring two different photos; one
command's output is never a substitute for the other's. Do **not** try to
combine measurements from two separate photos of the two different pages
into one correction — that's only physically valid when both pages are
photographed together, in the same frame, under identical lighting (a
single clapper-board shot showing the whole chart), which neither command
here assumes.

## Setup (once per machine)

Needs `premiere-ai` installed in whatever environment the command runs
in, and the SAM3 segmentation model downloaded before first use:

    hf download mlx-community/sam3-4bit

## Running the command

Video page:

    calibrate-lut chart_photo.png
    calibrate-lut chart_photo.png --output corrected.cube --lut-size 33
    calibrate-lut chart_photo.png --highlight-shadow-ire 100,97,94,6,3,0

Classic page:

    calibrate-lut-classic chart_photo.png --matrix-method "Finlayson 2015" --matrix-degree 2
    calibrate-lut-classic chart_photo.png --matrix-method "Finlayson 2015" --matrix-degree 2 --output corrected.cube --lut-size 33

Default `calibrate-lut-classic` to `--matrix-method "Finlayson 2015"
--matrix-degree 2` unless the user asks for a different matrix method or
degree. Full flag reference: `calibrate-lut --help` /
`calibrate-lut-classic --help`.

Both commands default `--output` to `<image>.cube` next to the input
photo if `--output`/`-o` isn't given — fine when the chart photo already
lives in the Premiere Pro project's own folder, but **if the chart photo
is a frame extracted to a scratch/tmp location** (e.g. pulled from the
sequence via a temporary export), that default would save the `.cube`
into the tmp directory too. In that case always pass `--output`
explicitly, pointing at the Premiere Pro project's own folder (e.g.
`assets/luts/corrected.cube` or wherever the project keeps LUTs) —
never leave the calibrated `.cube` sitting in a tmp directory.

## Reading the output

Both commands print their fitting diagnostics to stdout as they go, then
save the `.cube`. Read these before declaring success:

- **Grid identity check** (both) — reports how many of the 24 patches were
  consistent with the recovered chart orientation, e.g. `24/24 patches
  consistent with orientation 'rotate-ccw+flip-cols'`. Anything less than
  the full patch count means some patches were excluded from the fit —
  relay this to the user as a caveat, not a silent success.
- **`calibrate-lut-classic`'s per-patch RMSE** — printed before/after
  correction (`raw=... -> corrected=...`); lower is better. Large
  per-patch `err` values on individual rows are worth flagging even when
  the overall RMSE improves.
- **`calibrate-lut`'s post-rotation residuals** — degrees of hue error
  remaining per chromatic/skin patch after the global rotation; large
  residuals on multiple patches suggest the chart wasn't lit evenly.

## Caveats to relay to the user

- `calibrate-lut` (video page): the chromatic hue targets are exact
  (computed analytically from Rec.709), but the skin-tone target and the
  default `--highlight-shadow-ire` values are reasoned estimates, not
  vendor-confirmed figures — Calibrite doesn't publish them. Suggest
  verifying visually against Premiere's own skin-tone-line vectorscope
  toggle before trusting the result as final.
- `calibrate-lut-classic`: reference values are BabelColor/Danny Pascale's
  published *nominal* sRGB measurements for the Classic chart, not the
  physical unit's own individually-calibrated values — good enough to
  verify grid orientation and fit a real correction against, but not a
  factory-exact reference.

## Applying the result

The `.cube` these commands produce is exactly what Premiere's Lumetri
Color "Input LUT" consumes. There is currently no automated skill to load
it onto a clip — the `desktop-set-input-lut` command/skill that used to do
this was removed 2026-07-23 pending a rebuild on generic `desktop-*`
UI-automation primitives plus AI vision-language screen understanding.
Until that lands, load it manually: select the clip, open Effect Controls
> Lumetri Color > Basic Correction, and set Input LUT > Browse… to the
produced `.cube` file.
