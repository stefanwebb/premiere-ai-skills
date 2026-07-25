---
name: synchronize-clips
description: Use when asked to synchronize, align, or sync a camera recording with a high-quality external mic recording of the same take, or to compute, check, or verify the audio-sync offset between them, e.g. "sync the mic audio to the camera footage" or "check if project007's audio needs realigning". Also use to sanity-check for clock drift between the two recording devices.
---

# /synchronize-clips

Computes the offset (and a clock-drift sanity check) needed to align a
high-quality mic recording with a camera recording of the same speech, by
combining word-level ASR timestamps with local cross-correlation. Reports
the result and, when applicable, applies it in Premiere by building a
synced sequence (see Applying the offset).

## Running the command

Run in any environment with `premiere-ai` installed (Conda, uv, venv, or
otherwise — only the package needs to be available, not a specific tool):

    sync-audio "<camera_file>" "<mic_file>"

If the calling project has a `CLAUDE.md`, check it for conventions on
where the camera recording and mic recording live by default (e.g. a
fixed directory each is imported into) and use those as the default file
paths. If no such convention is documented, or it doesn't resolve to
exactly one file per role, ask which files to use rather than guessing.

## Cap the analysis at the first 3 minutes

`sync-audio` transcribes both inputs end to end, so its runtime scales
with take length — a 20-minute take costs two full 20-minute ASR passes.
**When either input is longer than 3 minutes, analyse only the first 3
minutes.** Both recordings start at their own t=0, so an offset measured
over the head of the take applies to the whole take.

`sync-audio` has no duration flag, so cut 3-minute heads with `ffmpeg`
first and run against those (audio-only is enough — it never looks at the
video stream):

    ffmpeg -y -v error -t 180 -i "<camera_file>" -vn -ac 1 -ar 48000 "$TMP/cam-3min.wav"
    ffmpeg -y -v error -t 180 -i "<mic_file>"          -ac 1 -ar 48000 "$TMP/mic-3min.wav"
    sync-audio "$TMP/cam-3min.wav" "$TMP/mic-3min.wav"

Put the cuts in a scratch directory, not the project. Apply the resulting
offset to the **full-length** clips.

The one thing this costs is drift coverage: `driftMsPerMinute` is then
fitted over 3 minutes and extrapolated across the take. Multiply it by
the full take length and report the projected total — if that comes out
to more than a frame, say so and offer a full-length run instead of
silently trusting the extrapolation.

## Syncing tracks already in a Premiere sequence

If the user asks to sync a video track and an audio track that already
exist in a sequence (rather than handing you file paths directly), don't
export the tracks to compute the offset. Use `premiere-cli` (see the
`premiere-cli` skill) to locate the on-disk media files backing each
clip — `get-full-clip-info` / `get-project-item-info` on the two clips
will give you their source file paths — then run `sync-audio` on those
paths directly.

## Useful flags

    sync-audio camera.mp4 mic.wav --anchor-spacing-seconds 60   # fewer anchors, faster on long takes
    sync-audio camera.mp4 mic.wav --high-fidelity                # decode at native sample rate, not 16kHz
    sync-audio camera.mp4 mic.wav -o report.json                 # write JSON instead of printing it

Full flag reference: `sync-audio --help`.

## Reading the result

    {
      "anchors": [...],
      "droppedLowConfidenceCount": 0,
      "driftFit": {"slopeSecondsPerSecond": ..., "driftMsPerMinute": ...},
      "recommendedOffsetSeconds": 0.238,
      "warnings": []
    }

- **`recommendedOffsetSeconds`** — `micSeconds - cameraSeconds` for the
  same spoken word: how far *into* the mic file a moment sits relative to
  where it sits in the camera file. **Positive means the mic started
  recording first**, so the mic clip must be shifted *earlier* (or,
  equivalently, its in-point moved that far into the source) to align
  with the camera. Negative means the camera started first and the mic
  clip must be shifted *later*.

  Read the sign off the anchors rather than trusting memory: each anchor
  reports its own `cameraSeconds` and `micSeconds`, and `micSeconds`
  being the larger of the two is what a positive offset means. Getting
  this backwards doubles the error instead of removing it.
- **`driftFit`** — `null` if fewer than 3 confident anchors were found
  (check `warnings` for why); otherwise `driftMsPerMinute` is the
  sanity-check number — near zero means the two devices' clocks agree
  well enough that a single constant offset is safe for the whole take.
- **`warnings`** — read these, e.g. anchors skipped for being too close
  to the recording's start/end, or a fallback to median offset when too
  few confident anchors exist.

Report the recommended offset and drift rate back to the user in plain
language, and express the offset as **whole seconds plus a whole number
of audio samples** at the mic's sample rate — not as a decimal fraction
of a second. At 48kHz, `samples = round(offsetSeconds * 48000)`, then
split off the whole seconds:

    6.031709952881989 s  ->  6s + 1522 samples   (289,522 samples total)

e.g. "the mic started 6s + 1522 samples before the camera, so shift the
mic clip that much earlier; drift is negligible at 0.36ms/min." Samples
are the unit the offset actually gets applied in (see the `premiere-cli`
skill's Time precision section), so reporting them avoids a second
rounding step when it's time to place the clip.

## Applying the offset

`sync-audio` itself only **computes** the offset — it does not move
anything in Premiere. When the two inputs are files on disk (not yet in
a sequence), after reporting the offset ask whether the user wants a
synced sequence built from them. If so, use `premiere-cli` (see the
`premiere-cli` skill for exact flags) to:

1. `create-sequence` — a new sequence to hold the synced clips.
2. `add-to-timeline` — insert the camera clip (video + its native audio)
   and the mic clip (audio only).
3. `unlink-selection` — unlink the camera clip's video from its native
   audio.
4. `remove-from-timeline` — delete the camera clip's native audio track
   clip.
5. `move-clip-to-track` — move the mic audio clip onto the now-empty
   audio track.
6. `trim-clip` — set **both the in-point and the out-point** of the video
   clip and of the mic audio clip, per the arithmetic below.
7. `link-selection` — re-link the trimmed video and mic audio clips.

Confirm the plan with the user before running it — it mutates the
sequence — and prefer computing exact trim points from the reported
offset rather than eyeballing them.

### Both ends, not just the head

Aligning the heads is only half the job. The two devices also *stopped*
at different times, so after the head trim one clip still overruns the
other and the sequence ends with video over silence (or audio over
black). **Trim the tails too, so the two clips are exactly the same
length.** Get the source durations from `ffprobe` (or
`get-project-item-info`) and compute all four points up front:

    K  = recommendedOffsetSeconds        # + means the mic started first
    Dc = camera source duration
    Dm = mic source duration

    cameraIn  = max(0, -K)               # camera skips its head if it started first
    micIn     = max(0,  K)               # mic skips its head if it started first
    L         = min(Dc - cameraIn, Dm - micIn)     # common overlap length

    cameraOut = cameraIn + L
    micOut    = micIn    + L

Set each head with `trim-clip --in-point-seconds`. **The tails need a
razor, not a trim** — `trim-clip --out-point-seconds` writes the source
out-point and reports `verified: true` but leaves the clip its original
length on the timeline (see the `premiere-cli` skill's Time precision
section). Cut each clip at `L` and delete the remainder:

    premiere-cli split-clip --track-type video --track-index 0 --seconds <L>
    premiere-cli remove-from-timeline --track-type video --track-index 0 \
      --clip-index 1 --ripple false

Pick `L` on a whole video frame (`floor(L * fps) / fps`) so the video cut
lands cleanly; the audio head keeps its sample-exact in-point either way.
Both clips then run `[0, L]` on the timeline. Worked example from a real
take —
Dc = 1233.600 s, Dm = 1236.753 s, K = +6.032 s (6s + 1522 samples):

    cameraIn  = 0          micIn  = 6.032
    L         = min(1233.600, 1230.722) = 1230.722
              -> 1230.72 after flooring to a 25fps frame (30,768 frames)
    cameraOut = 1230.72    micOut = 1236.752

i.e. the mic loses 6.032 s off its head and the camera loses 2.880 s off
its tail. Note which clip gets trimmed at which end is not fixed — it
depends on which device stopped last, so compute it, don't assume.

**Verify before declaring done:** re-read both clips and check they
report the same `startSeconds` (0) and the same `endSeconds`, and that
`get-timeline-summary` shows `coveragePercent` 100 on both tracks and a
`durationSeconds` equal to `L`. Coverage below 100, or a sequence
duration still equal to the untrimmed length, means a tail survived.
Re-run `link-selection` afterwards — the razor breaks the A/V link.

**Do `trim-clip` last, and re-check the in-point after any later edit.**
`add-to-timeline` and `move-clip-to-track` snap a clip's in-point to a
video frame boundary — at 25fps that silently rounds the offset by up to
20ms, which is the same order as the misalignment being corrected. Only
`trim-clip --in-point-seconds` writes the exact value (it sets ticks
directly and verifies the read-back), so apply it after the clip is on
its final track and confirm `inPointSeconds` in the response matches
`recommendedOffsetSeconds` before declaring the sync done.
