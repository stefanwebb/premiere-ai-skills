---
name: desktop-set-input-lut
description: Use when asked to apply, load, or set a LUT/look/.cube file on a clip's Lumetri Color in Premiere Pro — e.g. "apply this look to the clip" or "set the input LUT to corrected.cube". Only needed for a .cube file NOT already loaded in Premiere's Input LUT dropdown; for switching between LUTs already loaded, see apply-lut in the premiere-cli skill instead.
---

# /desktop-set-input-lut

Sets the selected clip's Lumetri "Input LUT" to a `.cube` file by driving
Premiere Pro's own native UI — via `premiere-cli desktop-set-input-lut`,
from the [premiere-cli](https://github.com/stefanwebb/premiere-cli)
package. This exists because ExtendScript cannot set this property to an
arbitrary file path on this Premiere build: "Input LUT" is a
registry-backed dropdown, populated only by files already loaded into it
(browsed to once via the UI, or dropped into Premiere's LUT folder) — see
the `apply-lut` entry in the `premiere-cli` skill for that corrected
finding. This command is the one way to load a **brand-new** `.cube` file
Premiere hasn't seen before.

## Safety — read before running

This drives the real keyboard and mouse via the macOS Accessibility API.
**Premiere Pro must be frontmost and the user must not touch the keyboard
or mouse while it runs**, or synthetic input lands in the wrong app. The
command shows an on-screen status overlay (amber "waiting" while it
confirms Premiere is frontmost, red "driving" while sending input, green
"done") so this is visible to the user — tell them to expect it and to
leave the machine alone for the few seconds it takes.

It refuses to run rather than guessing wrong: it aborts before sending
any input if it can't confirm Premiere is frontmost, and refuses outright
if more than one clip is selected (driving the LUT picker with multiple
clips selected applies the change to **all** of them at once).

## Setup (once per machine)

macOS only. Needs the optional extra, on top of the base `premiere-cli`
install:

    pip install "premiere-cli[macos-desktop]"

And the macOS **Accessibility** permission (System Settings > Privacy &
Security > Accessibility) granted to whichever process actually runs the
command — Terminal, iTerm, etc. — not to Premiere Pro itself. If this is
missing, the command will fail to drive anything meaningfully; there's no
clean error path, so check this proactively if a run doesn't work as
expected.

It also needs the Premiere Bridge panel reachable (same as any other
`premiere-cli` command — see the `premiere-cli` skill's Setup section) to
check how many clips are currently selected before doing anything.

## Before running

Confirm (or ask if ambiguous):
- **Exactly one clip is selected** in the timeline — check with
  `premiere-cli get-premiere-state` if unsure; the command itself also
  checks this and refuses with a clear error if it's wrong, but confirming
  first avoids a wasted round-trip through the desktop-driving step.
- The clip should ideally already have Lumetri Color applied with Effect
  Controls open. If it doesn't, the command tries to add Lumetri Color
  itself via the same UI-driving technique — but if that also fails (e.g.
  Effect Controls isn't visible), it aborts with a message pointing at the
  fallback: `premiere-cli apply-effect --effect-name "Lumetri Color"
  --track-type ... --track-index ... --clip-index ...` (see the
  `premiere-cli` skill) — run that first, then retry.

## Running the command

    premiere-cli desktop-set-input-lut /path/to/look.cube

Result (one JSON object, matching every other `premiere-cli` command):

    {
      "ok": true,
      "path": "/path/to/look.cube",
      "appliedLumetriColor": false,
      "driveResult": "selected 'look.cube', pressed 'Open' via AX",
      "expectedInputLut": "look",
      "actualInputLut": "look"
    }

`expectedInputLut`/`actualInputLut` is the verification step — the
combobox is read back after driving, not just trusted to have worked, so
`ok: true` means the LUT is confirmed applied, not just that no error was
thrown. `appliedLumetriColor: true` means Lumetri Color had to be added
first (see above).

Exit codes: `0` ok; `1` bad path or Premiere Pro isn't running; `2` a
precondition failed (wrong clip-selection count, or no Lumetri
Color/Input LUT control found — the error message says which); `3`
couldn't confirm Premiere is frontmost within the timeout (ask the user to
click Premiere's window and retry); `4` the post-drive verification didn't
match what was requested — report the mismatch rather than assuming it
silently worked.

## Where the .cube file comes from

If the user hasn't supplied a `.cube` file and instead wants one built
from a ColorChecker chart photo, see the `calibrate-lut` skill — its
output is exactly what this command consumes.
