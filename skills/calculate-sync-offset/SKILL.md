---
name: calculate-sync-offset
description: Use when asked to compute, check, or verify the audio-sync offset between a camera recording and a high-quality external mic recording of the same take, e.g. "sync the mic audio to the camera footage" or "check if project007's audio needs realigning". Also use to sanity-check for clock drift between the two recording devices.
---

# /calculate-sync-offset

Computes the offset (and a clock-drift sanity check) needed to align a
high-quality mic recording with a camera recording of the same speech, by
combining word-level ASR timestamps with local cross-correlation. Reports
the result and, when applicable, offers to apply it in Premiere (see
Applying the offset).

## Running the command

Run in any environment with `premiere-ai` installed (Conda, uv, venv, or
otherwise — only the package needs to be available, not a specific tool):

    sync-audio "<camera_file>" "<mic_file>"

If the calling project has a `CLAUDE.md`, check it for conventions on
where the camera recording and mic recording live by default (e.g. a
fixed directory each is imported into) and use those as the default file
paths. If no such convention is documented, or it doesn't resolve to
exactly one file per role, ask which files to use rather than guessing.

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

- **`recommendedOffsetSeconds`** — the amount to shift the mic clip
  *later* (positive) or *earlier* (negative) to align it with the camera
  recording.
- **`driftFit`** — `null` if fewer than 3 confident anchors were found
  (check `warnings` for why); otherwise `driftMsPerMinute` is the
  sanity-check number — near zero means the two devices' clocks agree
  well enough that a single constant offset is safe for the whole take.
- **`warnings`** — read these, e.g. anchors skipped for being too close
  to the recording's start/end, or a fallback to median offset when too
  few confident anchors exist.

Report the recommended offset and drift rate back to the user in plain
language, e.g. "shift the mic clip 238ms later; drift is negligible at
0.7ms/min."

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
6. `trim-clip` — trim the video clip and the mic audio clip, applying
   `recommendedOffsetSeconds`, so both start and end at the same
   sequence time.
7. `link-selection` — re-link the trimmed video and mic audio clips.

Confirm the plan with the user before running it — it mutates the
sequence — and prefer computing exact trim points from the reported
offset rather than eyeballing them.
