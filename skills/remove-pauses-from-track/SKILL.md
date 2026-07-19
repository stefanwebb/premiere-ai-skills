---
name: remove-pauses-from-track
description: Use when asked to remove pauses, silences, or dead air from a specific track in a Premiere Pro sequence, e.g. "remove the pauses from the narration track" or "cut the silences out of audio track 2 (and keep the camera video in sync)". Also use when a linked video track must stay frame-aligned with the audio track being cleaned up.
---

# /remove-pauses-from-track

Detects and ripple-deletes silent pauses from one audio track in a
Premiere Pro sequence — and, if that audio is paired with a video track
(e.g. narration synced to camera footage), applies the exact same cuts to
that video track too, so the two stay in sync. **Destructive**: verify on
a duplicate/throwaway sequence before running on real footage — there is
no automated undo-grouping for the batch of cuts (each is its own undo
step in Premiere).

## Before running

Identify from context (or ask if ambiguous):
- The sequence name (or confirm the active sequence is the right one).
- The audio track index (0-based, e.g. `0` = Audio 1) to clean up.
- Whether that audio is linked to a video track, and if so, its index
  (0-based, e.g. `0` = Video 1). There's no auto-detection of "linked"
  tracks — state it explicitly, or omit if the audio track has no
  corresponding video (e.g. a pure narration/VO track).

Call `premiere-cli get-project-info` to confirm the sequence exists (in
the currently active project) and read its `frameRate` — needed for step
2 below.

## Running the pipeline

1. **Export the track's audio** (always the *entire* sequence — no
   `--start-seconds`/`--end-seconds` — so the exported file's time 0 is
   exactly the sequence's time 0, with no offset math needed later):

       premiere-cli extract-audio-track --sequence-name "<name>" \
         --audio-track-index <N> --format wav --output /tmp/<name>-track<N>.wav

2. **Detect pauses**, passing the sequence's *actual* fps (from
   `get-project-info` above) so the cut list's frame quantization
   matches what `remove-track-intervals` will independently re-derive
   from the sequence itself — these must agree for the frame numbers to
   mean the same thing on both sides:

       remove-pauses /tmp/<name>-track<N>.wav --aggressiveness 1.0 \
         --min-pause 10 --fps <actual fps> -o /tmp/<name>-track<N>.cuts.txt

   (`--aggressiveness 1.0` cuts tight to both edges of each pause;
   `--min-pause 10` cuts pauses as short as 10ms — these are this skill's
   defaults, not `remove-pauses`'s own CLI defaults, which are more
   conservative.)

3. **Apply the cuts**:

       premiere-cli remove-track-intervals --sequence-name "<name>" \
         --audio-track-index <N> [--video-track-index <M> ...] \
         --intervals-file /tmp/<name>-track<N>.cuts.txt

4. Report the result in plain language — how many intervals were
   applied, how many segments removed, and any `warnings` from the
   result (a track where a segment couldn't be removed is surfaced, not
   silently dropped — the rest of the cuts still succeed).

5. Clean up the temporary files (`.wav`, `.words.json`, `.txt`,
   `.cuts.txt`) created along the way.

## Notes

- `remove-pauses` requires `ANTHROPIC_API_KEY` (used for its Claude
  phrase/sentence-boundary detection step) — if it's missing, `remove-pauses`
  itself reports a clear error; nothing special needs to be done here.
- Full flag reference for either underlying command: `premiere-cli
  extract-audio-track --help`-equivalent is the `extract-audio-track`
  section of the `premiere-cli` skill; `remove-pauses --help` for the
  pause-detection flags; `remove-track-intervals` section of the
  `premiere-cli` skill for the apply step.
