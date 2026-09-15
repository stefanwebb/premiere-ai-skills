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

The two tracks only stay in sync if their clips are **unlinked** before the
cuts are applied (step 3) and the sync is verified with the offset
invariant afterwards (step 5). On linked clips the underlying command
silently slips audio source in-points, and every metric except that
invariant reports the result as perfect.

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

   By default every VAD-confirmed silence is a cut candidate — no
   ANTHROPIC_API_KEY required. Add `--phrase-boundaries` to instead gate
   most cuts to word-gaps Claude judges to be phrase or sentence
   boundaries, so a mid-thought hesitation isn't clipped (this requires
   ANTHROPIC_API_KEY). Either way, **any silence of 1 s or longer is cut
   wherever it falls** — past a point a pause is not hesitation, it is
   dead air. Change that threshold with `--always-cut <seconds>`, or
   `--always-cut 0` to cut nothing except word-gaps.

3. **If the audio and video clips are LINKED, unlink them first.**

   `remove-track-intervals` silently desyncs linked pairs: it leaves the
   picture correct and slips the audio clips' *source in-points* by up to
   several seconds. Unlinked, both tracks receive identical cuts and the
   sync offset is preserved.

       premiere-cli select-all-clips --sequence-name "<name>"
       premiere-cli unlink-selection --sequence-name "<name>"
       premiere-cli deselect-all-clips --sequence-name "<name>"

   Linked is the normal state for footage that came through
   `/synchronize-clips`, so assume linked unless you have checked. Do not
   try to dodge this by pointing `--audio-track-index` at an empty track
   and letting the linked video pass do the work — that corrupts the
   ripple (observed: video coverage 100% -> 42.5%, audio 26 -> 58 clips).

4. **Apply the cuts**:

       premiere-cli remove-track-intervals --sequence-name "<name>" \
         --audio-track-index <N> [--video-track-index <M> ...] \
         --intervals-file /tmp/<name>-track<N>.cuts.txt

   A "could not reach the Premiere Bridge panel on port 47823 ... timed
   out" error here does not necessarily mean the cuts failed — applying a
   large batch of ripple deletes can leave the panel unresponsive to new
   requests until it finishes. Don't assume failure; move on to the checks
   below before retrying or reporting an error.

   **Every `N of M segment(s) in range could not be removed` warning is a
   defect to investigate, not noise.** On the run that exposed this bug, 26
   such warnings were dismissed as benign because the duration and coverage
   arithmetic worked out; 17 clips were desynced.

5. **Verify sync — coverage and duration CANNOT do this.**

   A slip changes a clip's source in-point while leaving its timeline
   start, end and duration untouched, so `get-timeline-summary` reports the
   correct duration and 100% coverage on a badly desynced sequence. Run
   `/check-sequence-sync`:

       check-sequence-sync --sequence-name "<name>" \
         --video-track-index <M> --audio-track-index <N> --offset <offset>

   `<offset>` is whatever `/synchronize-clips` reported as
   `recommendedOffsetSeconds` for this footage (0 if the audio is not a
   separately-recorded mic); omit it to infer from the majority of pairs.
   It checks the offset invariant on every pair AND that no clip repeats
   source already played by the previous clip. Report the worst drift.

   If pairs are desynced, do NOT rebuild, and do NOT assume the picture is
   the reference: the report says which track's clips still line up
   without overlapping their neighbours, and gives the in-point that slips
   the *other* track back. Apply those with `trim-clip --in-point-seconds`
   (sets a source in-point without moving the clip on the timeline), keep
   the clips unlinked while doing it, assert `startSeconds`/`endSeconds`
   are unchanged after each call, and re-run the check.

6. **Relink the pairs** (if you unlinked in step 3), one pair at a time —
   selecting everything and linking once would group all clips together
   rather than pairing them:

       for each clip index i:
         deselect-all-clips
         set-clip-selection --track-type video --track-index <V> --clip-index i --select
         set-clip-selection --track-type audio --track-index <N> --clip-index i --select
         link-selection

   Re-run `check-sequence-sync` afterwards: linking can move things.

7. **Check for dead air left behind.** Extract the resulting track and look
   for silence the pass should have removed:

       premiere-cli extract-audio-track --sequence-name "<name>" \
         --audio-track-index <N> --format wav --output /tmp/<name>-after.wav

   Then, in the `premiere-pro` env:

       from premiere_ai.remove_pauses import _detect_silence
       silence, duration = _detect_silence("/tmp/<name>-after.wav")
       print(sum(b - a for a, b in silence), "s of silence in", duration, "s")

   A little is expected — cuts are frame-quantised and every kept pause at a
   boundary counts. Seconds of it, or any single stretch over ~2 s, means
   something is being missed; report it rather than leaving it.

   This check finds *missed cuts*, not desync. It cannot substitute for
   step 5: a desynced sequence and a correct one both measured 0.2%
   residual silence on the run that exposed the linked-clip bug.

8. Report the result in plain language — how many intervals were
   applied, how many segments removed, the worst sync drift from step 5,
   and any `warnings` from the result (a track where a segment couldn't be
   removed is surfaced, not silently dropped — the rest of the cuts still
   succeed). Never describe a pass as verified on the strength of duration
   and coverage alone.

9. Clean up the temporary files (`.wav`, `.words.json`, `.txt`,
   `.cuts.txt`) created along the way.

## Notes

- `remove-pauses` only needs `ANTHROPIC_API_KEY` if `--phrase-boundaries` is
  passed (its Claude phrase/sentence-boundary detection step) — if it's
  requested but the key is missing, `remove-pauses` itself reports a clear
  error; nothing special needs to be done here.
- Full flag reference for either underlying command: `premiere-cli
  extract-audio-track --help`-equivalent is the `extract-audio-track`
  section of the `premiere-cli` skill; `remove-pauses --help` for the
  pause-detection flags; `remove-track-intervals` section of the
  `premiere-cli` skill for the apply step.
