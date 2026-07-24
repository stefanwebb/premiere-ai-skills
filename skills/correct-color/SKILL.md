---
name: correct-color
description: Use when asked to color-correct or color-calibrate a Premiere Pro clip against a ColorChecker chart already sitting in the timeline, e.g. "correct color for this clip" or "calibrate against the chart at 00:01:30". Extracts the chart frame, builds a LUT, selects the target clip, and drives Premiere's native UI (screenshot + desktop-* primitives) to load the LUT onto Lumetri Color's Input LUT. See calibrate-lut for the LUT-building step if only that's needed.
---

# /correct-color

Color calibration against a ColorChecker chart that's already been shot
into a Premiere Pro sequence: extracts the frame containing the chart,
fits a correction LUT from it (`calibrate-lut`), selects the target clip,
and loads the fitted `.cube` onto that clip's Lumetri Color Input LUT by
driving the native Premiere UI. Use this when the chart is on the
timeline already; if you just have a standalone chart photo, go straight
to the `calibrate-lut` skill instead.

**Why UI-driving instead of an API call:** `premiere-cli apply-lut`
cannot set Input LUT directly — it's a registry-backed dropdown index on
this Premiere build, not a settable path string (see `premiere-cli`
skill's `apply-lut` entry and its `BUILD_FINDINGS.md`). Loading a
brand-new `.cube` Premiere hasn't seen before requires going through
Input LUT's own "Browse…" native file panel, which only the real UI
exposes. There is no AX geometry for Premiere's custom-drawn Lumetri
controls (see step 6), so this skill locates them visually from
`desktop-take-screenshot` output rather than by coordinate or by AX
query.

## Inputs

All three are optional, but **ask the user explicitly** before assuming
any of them — do not silently default:

- **Sequence** — ask if it should be the currently active sequence.
- **Video track** — ask if it should be `V1` (video track index 0).
- **Playhead position** (timecode `MM:SS:FF`) — ask if it should be the
  sequence's current playhead position.

If the user confirms all three defaults (or says nothing more specific),
resolve them via `premiere-cli` (see the `premiere-cli` skill for exact
flags) before doing anything else:

    premiere-cli get-premiere-state

reads the active sequence name and current `playheadSeconds` in one call
— use these as the confirmed defaults for sequence/position. If a video
track name was given instead of an index, resolve it against
`get-active-sequence`'s `videoTracks[].name` list.

If the user gave an explicit playhead position instead, move there first
and re-derive the frame's exact timecode from the read-back:

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

Step 6's UI-driving applies to whatever clip is selected in Premiere, so
there must be **exactly one** clip selected — clear any existing
selection before selecting the target clip found in step 1:

    premiere-cli deselect-all-clips --sequence-name "..."
    premiere-cli set-clip-selection --track-type video --track-index N --clip-index N \
      --select --sequence-name "..."

Confirm `actualSelected: true` in the result before continuing.

## 5. Ensure Lumetri Color is on the clip

Before touching the UI, make sure the selected clip actually has a
Lumetri Color component — cheaper to guarantee this via the API than to
discover it's missing three screenshots into step 6:

    premiere-cli apply-lut --track-type video --track-index N --clip-index N \
      --lut-path "<project-dir>/color-calibration/corrected_<timecode>.cube" --sequence-name "..."

This call is expected to report failure on the actual LUT-path step (see
the caveat above) — that's fine, ignore it. What matters is its side
effect: it ensures a Lumetri Color component exists on the clip
(`apply-effect --effect-name "Lumetri Color"` is the equivalent if you'd
rather not rely on a call whose primary purpose fails). Only proceed past
this if the clip ends up with Lumetri Color present — if you can't tell
from the response, confirm visually in step 6's first screenshot.

## 6. Drive the UI to load the LUT

This is the part with no AX shortcut: Premiere's Lumetri panel and its
Input LUT dropdown are custom-drawn (no accessibility geometry, no
pressable elements), so navigate it the way a person would — by looking
at screenshots and clicking what you see. Narrate progress with
`desktop-notify --state driving "..."` so the user knows synthetic input
is in flight, and don't touch the mouse/keyboard yourself while these
commands run (real input competes with the synthetic events for
whichever app is frontmost).

**Keep screenshots cheap — crop before reading, not after:** the slow
part of this whole step is round trips (screenshot → read → reason →
click), not upload time, but a full 4K screenshot still costs more
vision processing per round trip than the ~40px control you're actually
looking for needs. Take exactly **one** full-frame screenshot per step-6
run — right at sub-step 1, to confirm the right panel is frontmost and
Lumetri Color is present — and read that one at full size. For every
screenshot after that, crop it to the small region you expect the target
control in (with enough margin to still confirm a click landed) *before*
handing it to yourself to read, and read only the crop — never read the
full frame and then separately read a crop of it, that pays the vision
cost twice for the same information:

    premiere-cli desktop-take-screenshot /tmp/shot.png
    python3 -c "
    from PIL import Image
    im = Image.open('/tmp/shot.png')
    im.crop((x0, y0, x1, y1)).save('/tmp/shot_crop.png')
    "
    # then read only /tmp/shot_crop.png, not /tmp/shot.png

Pick `(x0, y0, x1, y1)` generously around where you expect the control —
a crop that's too tight to show a state change is worse than a full
frame, since it forces another round trip to widen it. Reuse the same
crop bounds for the "did my click land" follow-up screenshot in the same
sub-step rather than recomputing them from a fresh full frame.

**Coordinate caveat, load-bearing:** `desktop-move-mouse` /
`desktop-click-mouse` take coordinates in AX/CGEvent *point* space, not
raw screenshot pixels — on a Retina display a screenshot is 2x that
space. Take the pixel coordinate you read off the screenshot and **divide
by the display's backing scale factor (2 on Retina, 1 otherwise)** before
passing it to `--x`/`--y`. Verify with a follow-up screenshot after the
first click of a session — if it landed a row off, the scale factor is
wrong, not your reading of the image.

**No-op click caveat:** a click can silently do nothing even with
correct coordinates and confirmed frontmost state — occasionally the
first click after opening a panel has no visible effect, and a second,
identical click succeeds. If a screenshot shows no change after a click
that should have produced one, retry the same click once before
concluding the coordinates are wrong. Lumetri Color's six section headers
(Basic Correction, Creative, Curves, Color Wheels & Match, HSL Secondary,
Vignette) also carry no visible label of their own to read off the
screenshot — identify "Basic Correction" by position (the first of six
evenly-spaced rows immediately under the "Lumetri Color" header) rather
than by text.

1. **Open Effect Controls (not the dedicated "Lumetri Color" panel tab,
   if Premiere's layout has one docked separately) on the target clip and
   confirm Lumetri Color is there.**

       premiere-cli desktop-take-screenshot /tmp/step6-1.png

   Read this one at full size — it's the one full-frame read for this
   whole step. If the Effect Controls panel isn't showing the target
   clip, or Lumetri Color isn't in its effect stack, stop and tell the
   user — something upstream (selection, or the apply-lut side effect)
   didn't take. Both Effect Controls and a separately-docked Lumetri Color
   tab can show what looks like the same Input LUT row — if both are
   visible in the screenshot, drive the one inside Effect Controls; a
   click meant for one can land on the other's identical-looking control.
   Note the pixel bounds of the effect-stack list (roughly the left third
   of the screen, from the "Lumetri Color" row down) — reuse that same
   crop box for sub-steps 2–3 instead of recomputing it.

2. **Expand Lumetri Color, then Basic Correction, if collapsed.**
   Identify the disclosure triangle next to "Lumetri Color" (and, once
   that's open, "Basic Correction") in the crop from sub-step 1, convert
   its pixel coordinates to point space, and click it. Take the follow-up
   screenshot, crop it to the same box (see "Keep screenshots cheap"
   above), and read only the crop:

       premiere-cli desktop-click-mouse --x <px_x/scale> --y <px_y/scale>
       premiere-cli desktop-take-screenshot /tmp/step6-2.png
       python3 -c "from PIL import Image; Image.open('/tmp/step6-2.png').crop((x0,y0,x1,y1)).save('/tmp/step6-2_crop.png')"

   Confirm from the crop that Basic Correction's controls (including the
   "Input LUT" row) are now visible before continuing.

3. **Open the Input LUT dropdown.** Click on the dropdown control itself
   (not its label) using the same pixel→point conversion, then
   screenshot+crop (same pattern as sub-step 2) to confirm the popup list
   opened:

       premiere-cli desktop-click-mouse --x ... --y ...
       premiere-cli desktop-take-screenshot /tmp/step6-3.png
       python3 -c "from PIL import Image; Image.open('/tmp/step6-3.png').crop((x0,y0,x1,y1)).save('/tmp/step6-3_crop.png')"

4. **Click "Browse…" in the open list**, then wait briefly and
   screenshot+crop to confirm the native "Select a LUT" file panel opened
   — this crop box will be different (the panel opens roughly centered,
   not inside Effect Controls), so recompute it once from this
   screenshot and reuse it for sub-steps 5–7:

       premiere-cli desktop-click-mouse --x ... --y ...
       premiere-cli desktop-take-screenshot /tmp/step6-4.png

   If the panel didn't open, the click likely missed the row — re-read
   the coordinates from sub-step 3's crop rather than guessing an offset.

5. **Use Cmd+Shift+G to jump straight to the file** rather than
   navigating folders or relying on type-ahead — type-ahead cannot work
   here (synthetic keystrokes carry no attached keycode Premiere's
   file-list can match against), but Cmd+Shift+G's "Go to Folder" sheet
   accepts a **full file path** (directory + filename together) and
   selects that exact file outright:

       premiere-cli desktop-press-key --key g --modifier cmd --modifier shift
       premiere-cli desktop-take-screenshot /tmp/step6-5.png

   Crop to the file panel's bounds from sub-step 4 and confirm the "Go to
   Folder" sheet is open before typing into it. Skip this whole sub-step
   if the target file is already visible in the panel from sub-step 4's
   screenshot (e.g. because Premiere remembered the last-used folder) —
   click the visible file directly instead of summoning the sheet.

6. **Type the full absolute path to the `.cube` file and confirm.** This
   sheet's text field is a genuine native AX-compliant control (unlike
   Premiere's own panels), so `desktop-enter-text-with-validate` is safe
   and preferred here — it reads the typed value back and retries on a
   mismatch, which matters because dropped characters send the panel
   somewhere unrelated with no obvious error:

       premiere-cli desktop-enter-text-with-validate "<project-dir>/color-calibration/corrected_<timecode>.cube"
       premiere-cli desktop-press-key --key return
       premiere-cli desktop-take-screenshot /tmp/step6-6.png
       python3 -c "from PIL import Image; Image.open('/tmp/step6-6.png').crop((x0,y0,x1,y1)).save('/tmp/step6-6_crop.png')"

   The crop should now show the target `.cube` selected/highlighted in
   the file panel with its Open/Choose button enabled — `desktop-enter-
   text-with-validate`'s own read-back already confirms the typed path,
   so this crop only needs to confirm the RIGHT file got highlighted, not
   re-verify the text. If the file isn't the one selected, do not proceed
   — report what happened instead of retrying blindly.

7. **Confirm the selection.** Pressing Return again normally activates
   the panel's default (enabled) button once a file is selected — skip
   the extra screenshot here and go straight to sub-step 8's verification
   unless something in sub-step 6 looked off, since a dedicated Effect
   Controls check happens next either way:

       premiere-cli desktop-press-key --key return

   If sub-step 8 shows the panel still open, come back here, locate the
   "Open" / "Choose" / "Import" button visually instead (crop the file
   panel's bottom-right corner) and click it directly (same
   pixel→point conversion as above).

8. **Verify the result.** Take one more screenshot, crop it to the Input
   LUT row (the same box from sub-step 2), and confirm it now shows the
   `.cube` file's name (not "Browse…" or "None"), then `desktop-notify
   --state done "LUT applied"` and dismiss with
   `desktop-dismiss-notifications`.

Report the calibration diagnostics from step 3 alongside the final
confirmation — a LUT built from a poor fit is still worth flagging to the
user even after it's been applied.
