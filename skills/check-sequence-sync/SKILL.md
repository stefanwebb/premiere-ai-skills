---
name: check-sequence-sync
description: Use when asked to check, verify, or audit audio/video sync in a Premiere Pro sequence, to confirm a cut left the camera and mic clips aligned, or to find clips that repeat/duplicate footage at a cut — e.g. "is the final cut still in sync?", "verify the audio and video after removing pauses", "the end of a clip repeats at the next cut, find every place that happens". Also use after any timeline surgery on synced clips (silence removal, ripple deletes, retimes, manual in-point repairs) before reporting it as done.
---

# /check-sequence-sync

Runs the two checks that actually detect a slipped clip, over every clip on
one video track and its paired audio track:

1. **Sync offset invariant** — for every clip pair (same timeline start),
   `audio.inPointSeconds − video.inPointSeconds` equals the constant sync
   offset. Any pair off by more than the tolerance is *desynced*.
2. **Source overlap** — consecutive clips cut from the same media must not
   overlap in source time. When the incoming clip's in-point precedes the
   outgoing clip's out-point, the last N frames of one clip play again at
   the head of the next (heard and seen as a stutter at the cut).

Both are read-only. Duration, clip count and `coveragePercent` **cannot**
find either problem — a slip moves a clip's source in-point while its
timeline start, end and duration stay put — so never report a sequence as
in sync on the strength of those.

The tool is `check-sequence-sync` from `premiere-ai`.

## Before running

Identify from context (or ask if ambiguous):
- The sequence name (default: the active sequence).
- The video track and audio track holding the synced pair (0-based;
  default V1/A1 = `0`/`0`).
- The expected sync offset, if it is known — `recommendedOffsetSeconds`
  from `/synchronize-clips`, or whatever the project's notes record. If it
  is not known, the tool infers it per media file as the value most pairs
  agree on, which is right whenever most of the sequence is still in sync;
  say in the report that the offset was inferred.

## Running

    check-sequence-sync --sequence-name "<name>" \
      [--video-track-index 0] [--audio-track-index 0] \
      [--offset <seconds>] [--tolerance 0.001] [--json]

Exit status 0 means every check passed; 1 means at least one problem was
found. `--json` prints the full report (every drifted pair, every
overlap, and a `suggestedFix` per drifted pair) for scripting a repair.
`--from-json <path>` reads a saved `premiere-cli get-full-sequence-info`
dump instead of the live project, useful for comparing a backup sequence
with the current one.

The report lists, per problem, the timeline position as `MM:SS:FF` at the
sequence's frame rate and the 0-based clip index on the track, which is
what `premiere-cli trim-clip --clip-index` takes.

## Reading the result

- **Desynced pairs, with overlaps on one track only.** The track whose
  clips do NOT overlap their neighbours is the correct one; the other was
  slipped. The report names it per media file ("keep audio for
  main-camera-1.mp4") and gives the in-point that puts the slipped clip
  back. Do not assume the picture is the reference — on 2026-09-11 a
  repair that "fixed" sync by moving audio to match the video copied 37
  video overlaps into the audio, because it was the video that had slipped.
- **Desynced pairs, no overlaps.** No evidence which track slipped; the
  report defaults to keeping video. Check the footage before the first
  desynced pair against the raw recording (e.g. compare the words at that
  point in the transcript) before choosing.
- **Overlaps on both tracks, no drift.** The pairs agree with each other
  but both repeat the previous clip's tail. Both in-points must move
  forward by the overlap amount (or more — the overlap is the minimum
  correction, the true in-point may be later still). If a backup sequence
  from before the damage exists, take the in-points from there via
  `--from-json` on its dump.
- **Unpaired clips / duration mismatches.** The two tracks no longer have
  the same cut structure; a ripple delete hit one track and not the other.
  Fix the structure first — the sync checks assume it.

## Repairing

Slip in-points with `trim-clip --in-point-seconds`, which sets a source
in-point without moving the clip on the timeline:

    premiere-cli trim-clip --sequence-name "<name>" --track-type video \
      --track-index 0 --clip-index <i> --in-point-seconds <value>

After every call assert the returned `newValue.startSeconds`/`endSeconds`
are unchanged and `inPointSeconds` is the requested value; it worked on
linked pairs without dragging the other track on the run above, but
verify rather than assume. Duplicate the sequence first
(`premiere-cli duplicate-sequence --new-name "<name> before <what>"`),
then re-run `check-sequence-sync` and `save-project` when it passes.

Derive a clip's source out-point as `inPointSeconds + durationSeconds`;
the `outPointSeconds` Premiere reports goes stale after edits.

## Report

State PASS/FAIL, the number of pairs, the offset used and whether it was
given or inferred, and for a FAIL the count of desynced pairs (worst
drift, in seconds and frames) and the count of overlapping clips per
track. Never describe a sequence as verified in sync without this tool's
PASS.
