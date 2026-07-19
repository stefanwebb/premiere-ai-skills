---
name: premiere-log
description: Use when running automation against a Premiere Pro project (transcribing, cutting pauses, building/rendering HyperFrames compositions, creating projects) to surface progress inside the Premiere Bridge CEP panel, if it's open.
---

# /premiere-log

Sends a status line to the Premiere Bridge CEP panel (from the
[premiere-cli](https://github.com/stefanwebb/premiere-cli) package), if
it's open, so the user sees progress inside Premiere Pro while
automation runs.

## When to use

Call it at meaningful milestones during automation that touches a Premiere
Pro project — not for every internal step. Good moments:

- Starting and finishing a long-running task (`transcribe`, `remove-pauses`,
  rendering a HyperFrames composition, `create-empty-premiere-project`)
- A task fails partway through
- A multi-step workflow moves to its next stage

Skip it for routine internal steps that wouldn't mean anything to someone
watching Premiere Pro.

## Running the command

Run in any environment with `premiere-cli` installed (Conda, uv, venv, or
otherwise — only the package needs to be available, not a specific
tool):

    premiere-log "<message>" [--level info|warn|error] [--source <name>]

Examples:

    premiere-log "Transcribing..." --source transcribe
    premiere-log "Transcription complete" --source transcribe
    premiere-log "Pause detection failed: no VAD model available" --level error --source remove-pauses

## Notes

- If the panel isn't open in Premiere Pro, `premiere-log` exits with a
  non-zero status and an error on stderr — treat this as informational,
  not a reason to fail the overall task, unless the user specifically
  needs the panel to be watched.
- Default port is 47823; only pass `--port` if the user has said the panel
  is running on a different one.
