---
name: correct-color
description: Use when asked to color-correct or color-calibrate a Premiere Pro clip against a ColorChecker chart already sitting in the timeline, e.g. "correct color for this clip" or "calibrate against the chart at 00:01:30". End-to-end: extracts the chart frame, builds a LUT, and applies it to the clip — see calibrate-lut / desktop-set-input-lut for the individual steps if only one of them is needed.
---

# /correct-color

End-to-end color calibration against a ColorChecker chart that's already
been shot into a Premiere Pro sequence: extracts the frame containing the
chart, fits a correction LUT from it (`calibrate-lut`), and loads that LUT
onto the clip's Lumetri Input LUT (`desktop-set-input-lut`). Use this when
the chart is on the timeline already; if you just have a standalone chart
photo, go straight to the `calibrate-lut` skill instead.

## Inputs

All three are optional:

- **Sequence** — defaults to the currently active sequence.
- **Video track** — defaults to `V1` (video track index 0).
- **Playhead position** (timecode `MM:SS:FF` or seconds) — defaults to
  the sequence's current playhead position.

Whatever is not specified is resolved via `premiere-cli` (see the
`premiere-cli` skill for exact flags) before doing anything else:

    premiere-cli get-premiere-state

reads the active sequence name and current `playheadSeconds` in one call
— use these as the defaults for sequence/position. If a video track name
was given instead of an index, resolve it against `get-active-sequence`'s
`videoTracks[].name` list.

If a playhead position **was** explicitly requested and differs from the
current one, move there first and re-derive the frame's exact timecode
from the read-back:

    premiere-cli move-playhead --seconds N --sequence-name "..."

## 1. Find the clip under the playhead

The chart is assumed to be visible in whichever clip on the target video
track spans the resolved playhead time. Get the track's clip list and
find the one clip where `startSeconds <= t < endSeconds`:

    premiere-cli get-active-sequence --sequence-name "..."

(or `get-full-sequence-info` if you need it). Use that clip's
`trackIndex`/`clipIndex` for the selection step later. If no clip on the
target track covers that time, stop and tell the user — there's nothing
to extract a chart frame from.

## 2. Extract the chart frame

Resolve the project's own folder first (never a tmp directory — same
reasoning as `calibrate-lut`'s output-location guidance):

    premiere-cli get-project-info

`path` is the `.prproj` file; use its parent directory. Export the frame
into a `color-calibration/` subfolder there:

    premiere-cli export-frame --output "<project-dir>/color-calibration/chart_<timecode>.png" \
      --timecode <MM:SS:FF> --sequence-name "..."

Check `fileSizeBytes` isn't suspiciously small (see the `premiere-cli`
skill's `export-frame` caveats) — a blank frame means the playhead landed
somewhere without visible chart content.

## 3. Build the LUT

Ask which physical ColorChecker page is in frame if it isn't already
clear (classic 24-patch vs. the Calibrite video-production page) — see
the `calibrate-lut` skill's "Which one to use" section, this decision
isn't guessable from the image alone. Then run the matching command from
that skill, pointing `--output` at the same project folder (not the
extracted frame's tmp path, if it had been one):

    calibrate-lut-classic "<project-dir>/color-calibration/chart_<timecode>.png" \
      --matrix-method "Finlayson 2015" --matrix-degree 2 \
      --output "<project-dir>/color-calibration/corrected_<timecode>.cube"

    calibrate-lut "<project-dir>/color-calibration/chart_<timecode>.png" \
      --output "<project-dir>/color-calibration/corrected_<timecode>.cube"

Read and relay the fitting diagnostics (grid identity check, RMSE or
hue-residuals, and the caveats section) exactly as the `calibrate-lut`
skill describes before treating the `.cube` as good.

## 4. Select the clip

`desktop-set-input-lut` refuses to run unless **exactly one** clip is
selected, and applies to whatever is currently selected — clear any
existing selection before selecting the target clip found in step 1:

    premiere-cli deselect-all-clips --sequence-name "..."
    premiere-cli set-clip-selection --track-type video --track-index N --clip-index N \
      --select --sequence-name "..."

Confirm `actualSelected: true` in the result before continuing.

## 5. Apply the LUT

Follow the `desktop-set-input-lut` skill as-is, including its Safety
section (Premiere must be frontmost, hands off keyboard/mouse):

    premiere-cli desktop-set-input-lut "<project-dir>/color-calibration/corrected_<timecode>.cube"

Check `actualInputLut` in the result matches `expectedInputLut` before
declaring success, and report the calibration diagnostics from step 3
alongside it — an applied LUT built from a poor fit is still worth
flagging to the user.
