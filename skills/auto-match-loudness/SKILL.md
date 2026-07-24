---
name: auto-match-loudness
description: Use when asked to auto-match loudness, normalize loudness, or apply Essential Sound's Auto-Match Loudness to a clip in Premiere Pro, e.g. "auto-match loudness on this clip" or "normalize the VO track to -23 LUFS". Opens/focuses the Essential Sound panel, sets Audio Type to Dialog, expands and enables the Loudness section, and clicks Auto-Match — all by driving the native UI via screenshot + desktop-* primitives, since Auto-Match has no ExtendScript/QE API.
---

# /auto-match-loudness

Applies Essential Sound's "Auto-Match Loudness" to a clip. There is no
ExtendScript or QE API for this feature at all — unlike `correct-color`,
where only the Input LUT *load* step lacked an API, here the entire
operation must be driven through the real Essential Sound panel by
looking at screenshots and clicking what's there.

## Inputs

Ask the user explicitly before assuming either of these — do not
silently default:

- **Currently selected clip**, or **another clip** specified by
  sequence, track, and playhead position (`MM:SS:FF`).

If they want another clip, resolve it via `premiere-cli` (see the
`premiere-cli` skill for exact flags) before touching the UI:

    premiere-cli get-premiere-state

reads the active sequence and current selection in one call. If a
different sequence/track/position was given instead, move the playhead
and re-derive the clip there:

    premiere-cli move-playhead --timecode <MM:SS:FF> --sequence-name "..."
    premiere-cli get-active-sequence --sequence-name "..."

Find the one clip on the target audio track where
`startSeconds <= t < endSeconds`, then select it explicitly (Essential
Sound always reflects the *current selection*, so there must be exactly
one clip selected):

    premiere-cli deselect-all-clips --sequence-name "..."
    premiere-cli set-clip-selection --track-type audio --track-index N --clip-index N \
      --select --sequence-name "..."

Confirm `actualSelected: true` before continuing. If the user meant the
currently selected clip, just confirm via `get-premiere-state` that
`selection` is non-empty and has exactly one entry — don't touch
selection at all in that case.

## Driving the UI

Same discipline as `correct-color`'s step 6 — no AX shortcut exists for
any of this, so narrate progress with `desktop-notify --state driving
"..."`, don't touch the mouse/keyboard yourself while these commands
run, and keep screenshots cheap: take one full-frame screenshot only
where noted below, crop every screenshot after that to the small region
you expect the target control in, and read only the crop.

**Coordinate caveat, load-bearing:** `desktop-move-mouse` /
`desktop-click-mouse` take coordinates in AX/CGEvent *point* space, not
raw screenshot pixels — on a Retina display a screenshot is 2x that
space. Divide the pixel coordinate you read off the screenshot by the
display's backing scale factor (2 on Retina, 1 otherwise) before passing
it to `--x`/`--y`. Verify with a follow-up screenshot after the first
click of a session — if it landed a row off, the scale factor is wrong,
not your reading of the image.

**No-op click caveat:** a click can silently do nothing even with
correct coordinates and confirmed frontmost state. If a screenshot shows
no change after a click that should have produced one, retry the same
click once before concluding the coordinates are wrong.

### 1. Open/focus the Essential Sound panel

Use the app menu bar, not a Premiere panel shortcut — this is a genuine
native AX-compliant menu, unlike Premiere's own custom-drawn panels, so
it's the most reliable way to reach this regardless of whether the panel
is already open, closed, or open-but-unfocused: selecting
**Window → Essential Sound** opens it and gives it focus if it wasn't
open, or just gives it focus if it already was.

    premiere-cli desktop-take-screenshot /tmp/step1-menubar.png

Read this one at full size to locate "Window" in the app menu bar (top
of screen). Click it, screenshot again to find "Essential Sound" in the
opened menu, and click that:

    premiere-cli desktop-click-mouse --x <window_x/scale> --y <window_y/scale>
    premiere-cli desktop-take-screenshot /tmp/step1-windowmenu.png
    # crop to the opened menu list, read, then:
    premiere-cli desktop-click-mouse --x <essential_sound_x/scale> --y <essential_sound_y/scale>

Take one more full-frame screenshot to confirm the Essential Sound panel
is now visible and frontmost before continuing — if it isn't, the second
click likely missed the menu item; re-open the Window menu and retry
rather than guessing an offset.

    premiere-cli desktop-take-screenshot /tmp/step1-confirm.png

### 2. Ensure Audio Type is "Dialog"

The Audio Type reads out just under the clip name near the top of the
panel. Crop to that region:

    premiere-cli desktop-take-screenshot /tmp/step2.png
    python3 -c "from PIL import Image; Image.open('/tmp/step2.png').crop((x0,y0,x1,y1)).save('/tmp/step2_crop.png')"

If it already reads "Dialog", skip to step 3. Otherwise click it to open
the Audio Type selector, screenshot+crop to confirm the options appeared,
click "Dialog", then screenshot+crop the same region again to confirm it
now reads "Dialog". Reuse this crop box for later steps' clip-name/type
sanity checks if useful.

### 3. Locate and expand "Loudness"

The "Loudness" section sits between "Enhance Speech" (above) and
"Repair" (below) in the panel's scrollable list, and may not be visible
without scrolling. Take a full-frame screenshot, crop to the panel's
scrollable list area, and check whether "Loudness" is visible:

    premiere-cli desktop-take-screenshot /tmp/step3-1.png
    python3 -c "from PIL import Image; Image.open('/tmp/step3-1.png').crop((x0,y0,x1,y1)).save('/tmp/step3-1_crop.png')"

If it isn't visible, scroll the panel (e.g. by scrolling the mouse wheel
over the panel, or dragging its scrollbar if visible) and re-screenshot
until "Loudness" appears between "Enhance Speech" and "Repair".

Once visible, check its disclosure triangle: pointing **right** means
collapsed, pointing **down** means expanded. If collapsed, click the
triangle and screenshot+crop the same region to confirm it now points
down and the section's contents (including the enable toggle and
Auto-Match/Reset buttons) are visible.

### 4. Ensure "Loudness" is enabled

Next to "Loudness"'s label is a toggle: left position = disabled
(contents greyed out), right position = enabled. If it's in the left
position, click it and screenshot+crop to confirm it's now in the right
position and "Auto-Match"/"Reset" are no longer greyed out.

### 5. Click "Auto-Match"

With Loudness expanded and enabled, click "Auto-Match", then
screenshot+crop the text underneath the button:

    premiere-cli desktop-click-mouse --x <automatch_x/scale> --y <automatch_y/scale>
    premiere-cli desktop-take-screenshot /tmp/step5.png
    python3 -c "from PIL import Image; Image.open('/tmp/step5.png').crop((x0,y0,x1,y1)).save('/tmp/step5_crop.png')"

Success reads **"Auto-Matched to target loudness of -23.00 LUFS"**. If it
doesn't read that (e.g. still blank, or shows something else), click
"Auto-Match" a second time and re-check — this button is known to
occasionally no-op on its first press (see the no-op click caveat
above). If it still hasn't succeeded after a second attempt, stop and
report what the text actually shows rather than retrying indefinitely.

Once confirmed, `desktop-notify --state done "Auto-matched loudness"` and
dismiss with `desktop-dismiss-notifications`, and report the final
Auto-Match text back to the user.
