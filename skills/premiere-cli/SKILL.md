---
name: premiere-cli
description: Use when Claude needs to query or drive Premiere Pro itself (not just log a message) via the Premiere Bridge CEP panel — e.g. reading the active project's sequences — or drive the native Premiere Pro UI directly via macOS Accessibility/synthetic input (screenshot, key press, text entry, mouse move/click, a persistent on-screen notification). Growing list of commands; check here first before assuming a capability doesn't exist.
---

# /premiere-cli

Sends a command to the Premiere Bridge CEP panel and prints its JSON
response. This is the general mechanism for Claude to read and act on
live state inside Premiere Pro — distinct from `premiere-log`, which only
displays text in the panel and never touches the project.

## Setup (once per machine)

`premiere-cli` comes from the `premiere-cli` Python package
(https://github.com/stefanwebb/premiere-cli):

    pipx install premiere-cli        # or: uv tool install premiere-cli
    premiere-cli install-panel          # installs the CEP panel + enables PlayerDebugMode
    # restart Premiere Pro, open Window > Extensions > Premiere Bridge
    premiere-cli doctor                 # verifies the whole chain

If `premiere-cli` is not on PATH or `doctor` reports failures, fix that
before attempting any other command. API behavior notes are calibrated
against Premiere Pro 26.3.0 (see the repo's docs/BUILD_FINDINGS.md) —
re-verify build-sensitive commands after a Premiere upgrade.

## Running a command

    premiere-cli <command>

The full response is a JSON object printed to stdout:
`{"ok": true, "result": ...}` on success, `{"ok": false, "error": "..."}`
on any failure (including the panel being closed). Exit code is 0 for
`ok: true`, 1 otherwise — but the JSON on stdout is always well-formed
either way, so parse it directly rather than branching only on exit code.

## Available commands

> **2026-07-17 correction — undo:** several entries below say "undo is
> non-functional on this build". That is now known to be only PARTLY
> true (see BUILD_FINDINGS.md): `qe.project.undo()`/`redo()` DO work for
> operations that enter the undo stack (timeline edits, speed changes,
> placements), verified via `qe.project.undoStackIndex()`. Marker adds
> and track renames never enter the stack, so they remain un-undoable —
> and calling `undo` after one of those pops the NEXT stack entry
> (possibly a user edit). The `previousValue` read-backs those entries
> recommend remain the safest restoration path.

### `create-sequence`

Creates a new sequence in the currently open project.

    premiere-cli create-sequence --name "My Sequence" --bin "B-Roll/Interviews"
    premiere-cli create-sequence --name "Vertical Cut" --bin "Shorts" --fps 30 --width 1080 --height 1920

`--name` and `--bin` are required; `--fps`/`--width`/`--height` default to
`25`/`3840`/`2160` (4K, 25fps). `--bin` is a `/`-separated path; missing
bin segments are created automatically. Fails (does not create anything)
if a sequence with that exact name already exists in that bin.

Runs fully autonomously — no dialog pops up. The sequence is created from
a bundled preset (4K/25fps/Rec.709) rather than Premiere's interactive
"New Sequence" dialog. **Only the default fps/width/height (25/3840/2160)
are reliable right now** — confirmed working, no dialog, correct settings.
Requesting non-default values goes through a `getSettings()`/`setSettings()`
override step (`host/index.jsx`) that is confirmed buggy as of 2026-07-12
(live-tested: the sequence is created, but the requested fps/resolution do
not reliably end up applied) — this is a known, open issue to revisit, not
a documented working feature yet. Until it's fixed, prefer the defaults or
verify the result's actual sequence settings in Premiere Pro after any
non-default `create-sequence` call.

Result shape:

    {
      "name": "My Sequence",
      "bin": "B-Roll/Interviews",
      "sequenceID": "<guid>",
      "settingsApplied": ["fps", "resolution"],
      "settingsFailed": {},
      "colorSpace": "rec709 (from bundled sequence preset)"
    }

`settingsFailed` (a `{field: error message}` object) being non-empty means
the sequence WAS created but one or more of bin placement / frame rate /
resolution could not be applied as requested — check it even when
`ok: true`, don't assume every field in the request took effect.

### `extract-audio-track`

Extracts part or all of one audio track to a local audio file.

    premiere-cli extract-audio-track --output /tmp/vo.wav --audio-track-index 0
    premiere-cli extract-audio-track --output /tmp/clip.mp3 --audio-track-index 1 \
      --start-seconds 12.0 --end-seconds 45.5 --sequence-name "Sequence 02" --format mp3

`--output` and `--audio-track-index` (0-based, e.g. `0` = Audio 1) are
required. `--sequence-name` defaults to the currently active sequence.
`--start-seconds`/`--end-seconds` must be given together or not at all —
omit both to extract the entire sequence duration. `--format` is
`wav` (default) / `mp3` / `aac`, each resolved to a preset bundled inside
the installed Premiere Pro app itself — no Adobe Media Encoder or external
preset file needed. `--preset-path` overrides `--format` with any other
`.epr` file (e.g. a specific bitrate/format not covered above).

Implementation: temporarily mutes every video track and every audio track
except the target one, sets the sequence's in/out points if a range was
given, calls the standard DOM's `seq.exportAsMediaDirect()` (synchronous,
no Adobe Media Encoder dependency), then restores all original mute
state and in/out points — this happens even if the export throws.

Result shape:

    {
      "sequenceName": "Sequence 02",
      "audioTrackIndex": 1,
      "outputPath": "/tmp/clip.mp3",
      "format": "mp3",
      "presetPath": "/Applications/Adobe Premiere Pro 2026/.../MP3 256kbps High Quality.epr",
      "range": { "startSeconds": 12.0, "endSeconds": 45.5 },
      "fileSizeBytes": 1234567
    }

All three formats (`wav`, `mp3`, `aac`) are live-tested and confirmed
working, including the ranged `--start-seconds`/`--end-seconds` path.

### `remove-track-intervals`

Ripple-deletes a list of time intervals from an audio track — and,
optionally, the same intervals from one or more linked video tracks, so
they stay in sync. Destructive: verify on a duplicate/throwaway sequence
before running against real footage.

    premiere-cli remove-track-intervals --audio-track-index 0 \
      --intervals-file /tmp/main-mic.cuts.txt
    premiere-cli remove-track-intervals --sequence-name "Sequence 01" \
      --audio-track-index 0 --video-track-index 0 \
      --intervals-file /tmp/main-mic.cuts.txt

`--audio-track-index` and `--intervals-file` are required. `--sequence-name`
defaults to the currently active sequence (switched to automatically if a
named sequence isn't already the active tab — the QE DOM this command
uses only ever operates on whichever sequence is frontmost). `--video-track-index`
is repeatable — pass one per video track that should get the same cuts
(e.g. the camera track linked to this audio track); omit entirely if
there's no linked video. `--intervals-file` is a `remove-pauses`-style
cuts file: one `"MM:SS:FF - MM:SS:FF"` line per interval (blank lines
ignored) — exactly what `remove-pauses <file> -o cuts.txt` writes.

Result shape:

    {
      "sequenceName": "Sequence 01",
      "audioTrackIndex": 0,
      "videoTrackIndices": [0],
      "intervalsApplied": 2,
      "totalSegmentsRemoved": 4,
      "warnings": []
    }

`warnings` lists any interval where a segment couldn't be removed on a
given track (rare — surfaced rather than silently dropped) but doesn't
fail the whole command; the intervals that did succeed still get applied.

### `export-frame`

Exports a single PNG frame from a sequence at a given timecode.

    premiere-cli export-frame --output /tmp/frame.png --timecode 00:12:05
    premiere-cli export-frame --output /tmp/frame.png --timecode 00:12:05 --sequence-name "Sequence 02"

`--output` and `--timecode` (as `"MM:SS:FF"`, same format as
`remove-track-intervals`'s interval boundaries) are required.
`--sequence-name` defaults to the currently active sequence (switched to
automatically if a named sequence isn't already the active tab — the QE
DOM this command uses only ever operates on whichever sequence is
frontmost).

Result shape:

    {
      "sequenceName": "Sequence 02",
      "timecode": "00:12:05",
      "timeSeconds": 12.2,
      "outputPath": "/tmp/frame.png",
      "width": 3840,
      "height": 2160,
      "method": "qe",
      "attempts": [{"args": ["/tmp/frame.png"], "success": false, "error": "..."},
                   {"args": ["/tmp/frame.png", "/tmp/frame.png"], "success": true}],
      "succeededWithArgs": ["/tmp/frame.png", "/tmp/frame.png"],
      "fileSizeBytes": 6408423
    }

Two export mechanisms are tried in order, reported via `method`:

1. **`"qe"`** — the QE DOM's `exportFramePNG`. Its argument order is
   version-dependent across Premiere builds and its return value is
   outright unreliable (live-tested returning `false` on calls that DID
   write a file), so the command tries a short list of plausible
   signatures and judges success only by checking the filesystem.
   `attempts` records every signature tried; `succeededWithArgs` is the
   one that worked. Live-tested 2026-07-17 on Premiere Pro 2026: the
   variant that works on this build is `exportFramePNG(outputPath,
   outputPath)` — the output path passed twice, an undocumented quirk
   (sensible-looking width/height string args ran without error but wrote
   nothing).
2. **`"ame"`** — if every QE guess fails, a blocking one-frame Media
   Encoder export via `seq.exportAsMediaDirect()` with the bundled
   `PNG Sequence (Match Source)` preset (path reported as `presetPath`
   in the result; `succeededWithArgs` is absent for this method). The
   sequence's in/out points are narrowed to exactly one frame and
   restored afterward. Also live-tested working 2026-07-17.

Premiere writes the file under a variant of the requested name — QE
appends `.png` even to an already-`.png` path (`frame.png` →
`frame.png.png`) and AME appends a frame number (`frame.png` →
`frame0.png`). The command detects these variants by checking their exact
paths directly and renames the file to the path you asked for, so callers
never have to care. (Direct path checks, not directory listing: macOS
returns an empty listing to the panel for some directories, e.g. `/tmp`,
even though the files themselves are readable — the same TCC/sandboxing
quirk previously hit with `/Applications` during preset discovery.)

If both mechanisms fail, the result is `{"ok": false, "error": "...",
"attempts": [...]}` — not a crash. The error string states whether the
AME fallback was skipped (no bundled preset found) or ran and produced
no file.

A frame exported from a time range with no video clips is a "success"
that yields a blank image (QE: fully transparent; AME: fully black) —
if you get a suspiciously small `fileSizeBytes`, check that the sequence
actually has visible content at that timecode.

**Playhead-move failure:** before attempting export, the command must move the
playhead to the target timecode using `seq.setPlayerPosition(...)`, trying two
argument forms: a ticks string and a Time object. If neither form succeeds, the
command returns `{"ok": false, "error": "could not move the playhead to the
requested timecode with any known argument form (ticks string, Time object)",
"attempts": [...]}`. Note that this failure's `attempts` items have a **different
shape** than the exportFramePNG-failure case — they are `{form, success, error}`
(where `form` is a string like `"ticksString"` or `"TimeObject"`) rather than
`{args, success, error}`, so the two failure modes can be distinguished by
inspecting the `attempts` array structure.

Restores the sequence's original playhead position afterward, regardless
of success or failure.

### `get-project-info`

Snapshot of the currently active project — `app.project` only.

    premiere-cli get-project-info

Result shape:

    {
      "name": "project0002",
      "path": "/Volumes/MediaDrive/.../project0002.prproj",
      "numSequences": 1,
      "sequences": [
        {
          "name": "Sequence 01",
          "sequenceID": "<guid>",
          "frameRate": 25.0,
          "durationSeconds": 143.2
        }
      ],
      "numRootItems": 4
    }

`numRootItems` is `app.project.rootItem.children.numItems` — the count of
top-level bins/items in the Project panel, not a recursive total.
`frameRate`/`durationSeconds` can be `null` per-sequence if that field
couldn't be read. Live smoke-tested 2026-07-17 against a real project (returned plausible, correct-looking data); individual fields have not been exhaustively validated.

### `list-project-items`

Recursively lists every item in the active project's bin tree.

    premiere-cli list-project-items

Result shape:

    {
      "items": [
        {
          "name": "B-Roll",
          "treePath": "/B-Roll",
          "nodeId": "<guid>",
          "type": "BIN",
          "isSequence": false,
          "mediaPath": null
        },
        {
          "name": "interview-01.mp4",
          "treePath": "/B-Roll/interview-01.mp4",
          "nodeId": "<guid>",
          "type": "CLIP",
          "isSequence": false,
          "mediaPath": "/Volumes/MediaDrive/.../interview-01.mp4"
        }
      ],
      "count": 2
    }

Depth-first walk of `app.project.rootItem.children`, descending into bins
(capped at depth 32 as a defensive limit against pathological structures).
`type` is `"BIN"`/`"CLIP"`/`"FILE"`/`"ROOT"` where the `ProjectItemType`
constant is resolvable, else the raw numeric type as a string. `isSequence`
and `mediaPath` are `null` if their underlying calls (`item.isSequence()`/
`item.getMediaPath()`) throw — bins in particular have no media path. This
Live smoke-tested 2026-07-17 against a real project (returned plausible, correct-looking data); individual fields have not been exhaustively validated.

### `get-full-project-overview`

Comprehensive snapshot of the active project: a nested bin tree, every
sequence, and a classification of every non-bin item by media type.

    premiere-cli get-full-project-overview

Result shape:

    {
      "project": { "name": "project0002", "path": "/tmp/project0002.prproj" },
      "binTree": {
        "name": "project0002",
        "bins": [
          {
            "name": "B-Roll",
            "bins": [],
            "items": [
              { "name": "interview-01.mp4", "type": "CLIP", "mediaPath": "/tmp/interview-01.mp4" }
            ]
          }
        ],
        "items": []
      },
      "sequences": [
        { "name": "Sequence 01", "sequenceID": "<guid>", "frameRate": 25.0, "durationSeconds": 143.2 }
      ],
      "mediaTypeCounts": { "video": 3, "audio": 1, "image": 0, "sequence": 1, "other": 0, "offline": 0 }
    }

`binTree` starts at the root bin and nests recursively (capped at depth
32). `mediaTypeCounts` classifies every non-bin item by its `mediaPath`
extension: `video` (mp4/mov/mxf/avi/m4v/mts), `audio`
(wav/mp3/aac/m4a/aiff), `image` (png/jpg/jpeg/tif/tiff/psd/svg/gif), and
`other` for anything else. A sequence item always counts as `"sequence"`
regardless of any extension on its media path. An item where
`item.isOffline()` is true additionally increments `"offline"`, on top of
its type bucket — the two counts aren't mutually exclusive. This command
Live smoke-tested 2026-07-17 against a real project (returned plausible, correct-looking data); individual fields have not been exhaustively validated.

### `search-project-items`

Searches the project's item tree by name substring, extension, offline
status, and/or color label — at least one filter is required.

    premiere-cli search-project-items --name-contains "interview"
    premiere-cli search-project-items --extension mov --offline-only
    premiere-cli search-project-items --color-label 3

Result shape:

    {
      "items": [
        {
          "name": "vo-01.wav",
          "treePath": "/Audio/vo-01.wav",
          "nodeId": "<guid>",
          "mediaPath": "/tmp/vo-01.wav",
          "isOffline": false,
          "colorLabel": 3
        }
      ],
      "count": 1
    }

`--name-contains` is a case-insensitive substring match against item
names. `--extension` is compared case-insensitively against the media
path's extension, with or without a leading dot. `--offline-only` matches
items where `item.isOffline()` is true. `--color-label` (an int 0-15)
matches items where `item.getColorLabel()` equals it. A matching item must
satisfy ALL filters given (AND, not OR). Walks the whole project tree the
same way `list-project-items` does, skipping bins themselves for matching
purposes but always recursing through them.

Fails client-side (`exit 2`) if no filter is given at all. If
`--color-label` is given but `getColorLabel()` isn't available on this
Premiere build, the whole command fails:
`{"ok": false, "error": "getColorLabel is not available on this Premiere build"}`.
Live smoke-tested 2026-07-17 against a real project (returned plausible, correct-looking data); individual fields have not been exhaustively validated.

### `get-active-sequence`

Reads the ACTIVE sequence's full track/clip structure — no arguments, no
sequence-name override (use `get-full-sequence-info` for that). Standard-DOM
read only, no QE DOM involved, no need to switch tabs. This command Live smoke-tested 2026-07-17 against a real project (returned plausible, correct-looking data); individual fields have not been exhaustively validated.

    premiere-cli get-active-sequence

Result shape:

    {
      "name": "Sequence 01",
      "sequenceID": "<guid>",
      "frameRate": 25.0,
      "durationSeconds": 143.2,
      "width": 3840,
      "height": 2160,
      "videoTracks": [
        {
          "index": 0,
          "name": "V1",
          "isMuted": false,
          "isLocked": false,
          "clipCount": 2,
          "clips": [
            {
              "name": "clip.mp4",
              "trackIndex": 0,
              "clipIndex": 0,
              "nodeId": "<id>",
              "startSeconds": 0.0,
              "endSeconds": 10.0,
              "inPointSeconds": 0.0,
              "outPointSeconds": 10.0,
              "durationSeconds": 10.0,
              "mediaPath": "/Volumes/.../clip.mp4",
              "disabled": false
            }
          ]
        }
      ],
      "audioTracks": [ /* same shape as videoTracks */ ]
    }

Fails with `{"ok": false, "error": "no active sequence"}` if nothing is
active. Every clip field is read individually — an unreadable field is
`null`, not a command failure.

### `get-full-sequence-info`

Everything `get-active-sequence` has, plus per-clip effect components and
the sequence's markers. Live smoke-tested 2026-07-17 against a real project (returned plausible, correct-looking data); individual fields have not been exhaustively validated.

    premiere-cli get-full-sequence-info
    premiere-cli get-full-sequence-info --sequence-name "Sequence 02"

`--sequence-name` defaults to the currently active sequence. Result shape
adds, per clip:

    "components": [
      {"displayName": "Motion", "matchName": "AE.ADBE Motion", "enabled": true, "numProperties": 8}
    ]

and, at the sequence level:

    "markers": [
      {"name": "Chapter 1", "comments": "...", "type": "Comment", "startSeconds": 5.0, "endSeconds": 5.0, "guid": "<guid>"}
    ],
    "markerCount": 1

Markers are iterated via `getFirstMarker()`/`getNextMarker()` (no reliable
indexing on this API), capped at 10000 iterations as a safety guard. Output
can be large for big sequences — there is no pagination.

### `get-full-clip-info`

Full detail on one specific clip, including every property of every
applied effect. Live smoke-tested 2026-07-17 against a real project (returned plausible, correct-looking data); individual fields have not been exhaustively validated.

    premiere-cli get-full-clip-info --track-type video --track-index 0 --clip-index 2
    premiere-cli get-full-clip-info --sequence-name "Sequence 01" --track-type audio --track-index 1 --clip-index 0

`--track-type` (`video`/`audio`), `--track-index`, and `--clip-index` are
required; `--sequence-name` defaults to the active sequence. Out-of-range
indices fail with a message naming the actual track/clip counts, e.g.
`"trackIndex 3 is out of range — sequence has 2 video track(s)"`.

Result shape (everything `get-active-sequence`'s clips have, plus):

    {
      "name": "clip.mp4",
      "trackIndex": 0,
      "clipIndex": 2,
      "nodeId": "<id>",
      "startSeconds": 20.0,
      "endSeconds": 30.0,
      "inPointSeconds": 0.0,
      "outPointSeconds": 10.0,
      "durationSeconds": 10.0,
      "mediaPath": "/Volumes/.../clip.mp4",
      "disabled": false,
      "type": "Clip",
      "speed": 100.0,
      "isSpeedReversed": false,
      "linkedItemsCount": 1,
      "projectItem": {"name": "clip.mp4", "treePath": "/Bin/clip.mp4", "nodeId": "<id>"},
      "components": [
        {
          "displayName": "Motion",
          "matchName": "AE.ADBE Motion",
          "enabled": true,
          "properties": [
            {"displayName": "Position", "value": "[1920,1080]", "isTimeVarying": false, "keyCount": null}
          ]
        }
      ]
    }

`value` is always a stringified, 200-char-truncated representation of
`prop.getValue()` (which can throw or return a non-primitive) — treat it as
diagnostic text, not a typed value. `keyCount` is only populated when
`isTimeVarying` is `true`. Components are capped at 50 per clip and
properties at 100 per component; a `truncated: true` field appears on the
clip and/or component if either cap was hit.

### `get-timeline-summary`

A compact, human-oriented overview of a sequence — per-track clip counts
and coverage, not per-clip detail. Live smoke-tested 2026-07-17 against a real project (returned plausible, correct-looking data); individual fields have not been exhaustively validated.

    premiere-cli get-timeline-summary
    premiere-cli get-timeline-summary --sequence-name "Sequence 02"

`--sequence-name` defaults to the currently active sequence. Result shape:

    {
      "sequenceName": "Sequence 01",
      "frameRate": 25.0,
      "durationSeconds": 143.2,
      "videoTracks": [
        {"index": 0, "name": "V1", "clipCount": 5, "coveragePercent": 82.4}
      ],
      "audioTracks": [
        {"index": 0, "name": "A1", "clipCount": 3, "coveragePercent": 60.0}
      ],
      "totalClips": 8,
      "clipsWithEffects": 2,
      "markerCount": 1
    }

`coveragePercent` is that track's total clip duration divided by the
sequence duration, as a percentage rounded to 1 decimal (`0` if the
sequence duration is unreadable or zero). `clipsWithEffects` counts clips
whose `components` collection exceeds the built-in baseline (Motion +
Opacity for video clips, one baseline component for audio clips) —
unreadable clips are skipped rather than counted.

### `get-premiere-state`

Full point-in-time snapshot of whatever Premiere Pro state currently
exists — app version, open project, active sequence, playhead position,
and current selection. Live smoke-tested 2026-07-17 against a real project
(returned correct app version, active sequence, playhead position, and a
real timeline selection); individual fields have not been exhaustively
validated.

    premiere-cli get-premiere-state

No flags. Unlike most other commands, having no project open (or no active
sequence, or nothing selected) is **not** an error here — the whole point
of this command is a truthful snapshot of whatever exists right now, so
`ok: true` with every nested field `null` is a normal, valid response.

Result shape:

    {
      "appVersion": "25.0.0",
      "project": { "name": "project0002", "path": "/Volumes/.../project0002.prproj" },
      "activeSequence": {
        "name": "Sequence 01",
        "sequenceID": "<guid>",
        "frameRate": 25.0,
        "durationSeconds": 143.2,
        "width": 3840,
        "height": 2160
      },
      "playheadSeconds": 12.2,
      "selection": [
        {
          "name": "clip1.mp4",
          "mediaType": "Video",
          "startSeconds": 10.0,
          "endSeconds": 15.0,
          "nodeId": "<node-id>"
        }
      ]
    }

`project`, `activeSequence`, `playheadSeconds`, and `selection` are each
`null` when there's no project open, no active sequence, the playhead
couldn't be read, or `getSelection()` itself is unavailable, respectively.
An empty array (not `null`) for `selection` means a sequence is active but
nothing is currently selected in it. Individual fields inside
`activeSequence`/each selection entry can also independently be `null` if
that one Premiere API call failed — this doesn't fail the whole command.

### `inspect-dom-object`

Interactive DOM explorer for API discovery: evaluates a property-path
expression against the live Premiere Pro DOM and reflects on whatever it
finds — properties, method names, and enumerable keys. This is a
discovery/debug tool, not a stable feature, and Live smoke-tested 2026-07-17 against a real project (returned plausible, correct-looking data); individual fields have not been exhaustively validated.

    premiere-cli inspect-dom-object --expression "app.project.activeSequence"
    premiere-cli inspect-dom-object --expression "qe.project"

`--expression` is required and must be a **bare property path** rooted at
`app`, `qe`, or `$.global` — dots and bracket-indexing are allowed
(`app.project.rootItem.children[0]`), but parentheses are not: a call like
`app.project.save()` is rejected outright, since this command is strictly
read-only and a method call could mutate the open project. If the
expression starts with `qe`, `app.enableQE()` is called first.

Result shape:

    {
      "expression": "app.project.activeSequence",
      "typeofValue": "object",
      "stringValue": "[object Sequence]",
      "isNull": false,
      "isUndefined": false,
      "reflectMethods": ["getSettings", "getPlayerPosition", "..."],
      "reflectProperties": ["name", "sequenceID", "..."],
      "forInKeys": ["name", "videoTracks", "..."],
      "commonProperties": { "name": "Sequence 01", "sequenceID": "<guid>" }
    }

`reflectMethods`/`reflectProperties` come from ExtendScript's own
`.reflect` mechanism when the object exposes it (authoritative when
present, `null` otherwise). `forInKeys` is a plain for-in enumeration,
capped at 200 entries. `commonProperties` is a fixed candidate list
(`name`, `numItems`, `numTracks`, `numSequences`, `length`, `path`, `type`,
`mediaType`, `seconds`, `ticks`, `sequenceID`, `nodeId`, `treePath`,
`guid`) probed defensively — only properties that exist, aren't functions,
and don't throw on read are included, each stringified. `stringValue` is
`String(value)` truncated to 500 characters. A malformed or non-property-path
expression, or an evaluation error (e.g. a typo'd path), returns
`{"ok": false, "error": "..."}` rather than throwing.

Related to, but more general than, `debug-qe-inspect`: that command is
hardcoded to one specific QE sequence/track/clip investigation, while this
one accepts an arbitrary path into either the standard or QE DOM.

### `get-open-projects`

Lightweight list of every currently open project in Premiere Pro — this
replaces the multi-project-listing role of `get-project-metadata`, which
has been removed. Live smoke-tested 2026-07-17 (correct single-project listing with isActive).

    premiere-cli get-open-projects

No flags. Result shape:

    {
      "projects": [
        {
          "name": "project0002",
          "path": "/Volumes/MediaDrive/.../project0002.prproj",
          "isActive": true
        }
      ],
      "count": 1
    }

`isActive` is `true` for whichever open project's path matches
`app.project.path`, `false` for the others, or `null` if either path
couldn't be read. Having no project open is **not** an error here —
`{"ok": true, "result": {"projects": [], "count": 0}}` is the normal,
truthful response. Deliberately shallow: no per-project sequence detail is
included — use `get-project-info` (or `get-full-project-overview`) for
that on whichever project is currently active.

### `set-active-project`

Switches which currently-open project is active.

    premiere-cli set-active-project --name "project0002"
    premiere-cli set-active-project --path "/Volumes/MediaDrive/.../project0002.prproj"

**Partially live-verified 2026-07-17** (target resolution, the
`alreadyActive` short-circuit, and the unknown-project error path are
confirmed working; the actual activation path still needs a second open
project to test). Unlike the rest of this file's commands, this
one has not been confirmed working against a real multi-project-open
Premiere session. The activation API is uncertain on this Premiere build,
so the ExtendScript implementation tries three candidate methods in order
and verifies success after each by checking that `app.project.path`
actually changed to the target (never by trusting a method's own return
value):

1. `targetProject.activate()`, if this build exposes it.
2. `app.openDocument(path)` — documented as an "open project" call, but a
   fallback guess here: its behavior when the target is **already open**
   is unconfirmed on this build, and it may simply foreground the
   project, pop a dialog, or (worst case) re-open a duplicate. Treat this
   command as **mutating** and verify manually before scripting it into an
   unattended workflow.
3. A bare `app.project = target` assignment — no supporting reference in
   any of the three MCP repos studied for `PREMIERE_API_NOTES.md`, kept as
   a last-resort guess.

At least one of `--name`/`--path` is required (client-side `exit 2` if
neither is given); if both are given they must resolve to the same open
project, or the command fails. Already-active target short-circuits
without trying any activation method.

Result shape on success:

    {
      "activated": { "name": "project0002", "path": "/Volumes/.../project0002.prproj" },
      "attempts": [
        {"method": "activate", "success": false, "error": "..."},
        {"method": "openDocument", "success": true}
      ]
    }

or, if already active:

    {
      "activated": { "name": "project0002", "path": "/Volumes/.../project0002.prproj" },
      "alreadyActive": true
    }

Failure shapes: `{"ok": false, "error": "at least one of name or path is
required"}` (client-side), `{"ok": false, "error": "no open project
matched ...", "availableProjects": [...]}` if neither `--name` nor
`--path` matches an open project, `{"ok": false, "error": "name and path
did not resolve to the same open project", "availableProjects": [...]}`
if both are given and disagree, and `{"ok": false, "error": "could not
activate project \"...\" with any known method", "attempts": [...]}` if
every method fails.
### `move-playhead`

Moves a sequence's playhead to a target time — either a `"MM:SS:FF"`
timecode or a raw seconds value (exactly one of the two is required).

    premiere-cli move-playhead --timecode 00:12:05
    premiere-cli move-playhead --seconds 12.5 --sequence-name "Sequence 02"

`--timecode` and `--seconds` are mutually exclusive — passing both, or
neither, fails client-side (`exit 2`) before the panel is even contacted.
`--sequence-name` defaults to the currently active sequence; if the
resolved sequence isn't already the active one, it's made active first
(moving the playhead implies the user wants to see it, same as
`remove-track-intervals`'s tab-switch behavior) — no QE DOM needed.

Result shape:

    {
      "sequenceName": "Sequence 02",
      "timecode": "00:12:05",
      "requestedSeconds": 12.2,
      "playheadSeconds": 12.2,
      "attempts": [{"form": "ticksString", "success": true}]
    }

`timecode` is only present in the result when you supplied one (omitted
for the `--seconds` form). `playheadSeconds` is a read-back of the actual
position via `seq.getPlayerPosition()` — it can differ slightly from
`requestedSeconds` due to frame quantization.

Implementation reuses the exact `seq.setPlayerPosition()` pattern
`export-frame` uses to move the playhead: try a ticks string first, fall
back to a `Time` object, recording each attempt as `{form, success,
error?}`. Live-tested 2026-07-17: both the `seconds` and `timecode`
forms moved the playhead to the exact requested position, with the
read-back `playheadSeconds` matching.
Unlike `export-frame`, there is no restore step afterward: moving the
playhead is the entire point of this command.

If neither argument form succeeds in moving the playhead:

    {"ok": false, "error": "could not move the playhead to the requested
    time with any known argument form (ticks string, Time object)",
    "attempts": [...]}

## Additional commands (compact reference)

### Work area / in-out points

- `premiere-cli get-work-area [--sequence-name NAME]` — read the work area bar bounds. Returns `{sequenceName, inSeconds, outSeconds}`. live smoke-tested 2026-07-17.
- `premiere-cli set-work-area --start-seconds N --end-seconds N [--sequence-name NAME]` — set the work area bar. Returns `{sequenceName, startSeconds, endSeconds}`. live smoke-tested 2026-07-17.
- `premiere-cli get-sequence-in-out [--sequence-name NAME]` — read sequence in/out points. Returns `{sequenceName, inSeconds, outSeconds}`. live smoke-tested 2026-07-17.
- `premiere-cli set-sequence-in-out --in-seconds N --out-seconds N [--sequence-name NAME]` — set sequence in/out points (export range etc.), via the shared multi-form range helper. Returns `{sequenceName, inSeconds, outSeconds, attempts}`. live smoke-tested 2026-07-17.
- `premiere-cli is-work-area-enabled [--sequence-name NAME]` — check if the work area bar is enabled. Returns `{sequenceName, workAreaEnabled}`. live-tested 2026-07-17: no work-area-enabled API exists on this Premiere 2026 build — returns the honest probe-failure error.
- `premiere-cli get-export-file-extension --preset-path PATH [--sequence-name NAME]` — get the file extension a given `.epr` preset would export. Returns `{sequenceName, presetPath, extension}`. live smoke-tested 2026-07-17.

### Workspace

- `premiere-cli get-workspaces` — list available workspace layouts. Returns `{workspaces, count}`. live smoke-tested 2026-07-17.
- `premiere-cli set-workspace --name NAME` — switch workspace (e.g. `Editing`, `Color`, `Audio`). Returns `{set: true, workspace}`. live smoke-tested 2026-07-17.

### Playback

- `premiere-cli play-timeline` — start QE playback of the frontmost sequence tab (active-sequence-only, no `--sequence-name`). Returns `{playing: true}`. live smoke-tested 2026-07-17.
- `premiere-cli stop-playback` — stop QE playback (same active-sequence-only scope). Returns `{stopped: true}`. live smoke-tested 2026-07-17.
- `premiere-cli play-source-monitor [--speed N]` — play the clip open in the Source Monitor; speed defaults to `1.0`. Returns `{playing: true, speed}`. live smoke-tested 2026-07-17.
- `premiere-cli get-source-monitor-position` — read the Source Monitor's time indicator. Returns `{seconds}`; fails if no clip is open. live smoke-tested 2026-07-17.

### Version info

- `premiere-cli get-version-info` — Premiere Pro version/build info. Returns `{version, buildNumber, isDocumentOpen, path}`, each field individually best-effort. live smoke-tested 2026-07-17.
### Project & media reports

`get-bin-contents --bin-path "B-Roll/Interviews"` — recursively lists a bin's
contents by `/`-separated path (same convention as `create-sequence`'s
`--bin`, never auto-created here). Key fields: `items[].{name,treePath,
nodeId,type,mediaPath,isOffline,colorLabel}`, `count`. live smoke-tested 2026-07-17.

`get-project-item-info --node-id <id>` (or `--tree-path <path>`) — full
detail on one project item: media path, footage interpretation, in/out
points, proxy status, project/XMP metadata, markers, `isSequence`/
`isMulticamClip`/`isMergedClip`. Merges the reference project's
`get_project_item_info` and `get_item_info` tools into one command — they
were near-duplicates, so a separate `get-media-item-info` was skipped
entirely (see `premiere-pro/plugin/README.md` for the full field list).
live smoke-tested 2026-07-17.

`get-timeline-gaps [--sequence-name NAME] [--track-type video|audio|both]
[--min-gap-seconds N]` — finds empty spaces between clips on a sequence's
tracks. Key fields: `gapCount`, `gaps[].{trackType,trackIndex,
gapStartSeconds,gapEndSeconds,gapDurationSeconds,beforeClip,afterClip}`.
live smoke-tested 2026-07-17.

`get-offline-media` — project-wide scan for offline/missing media. Key
fields: `offlineCount`, `items[].{nodeId,name,treePath,mediaPath}`. Not yet
live-tested.

`get-used-media-report [--sequence-name NAME]` — every distinct source file
a sequence references, with a use count. Key fields: `uniqueMediaCount`,
`media[].{name,mediaPath,offline,useCount,tracks}`. live smoke-tested 2026-07-17.

`get-all-project-paths` — every unique media file path in the project. Key
fields: `pathCount`, `paths[].{path,name,nodeId,offline}`. Not yet
live-tested.

`get-unused-media` — project items never referenced by a clip in any
sequence. Key fields: `unusedCount`, `items[].{nodeId,name,treePath,
mediaPath}`. live smoke-tested 2026-07-17.

`get-duplicate-media` — project items that share the same source media
file. Key fields: `duplicateGroupCount`, `duplicates[].{mediaPath,count,
items}`. live smoke-tested 2026-07-17.

`get-clip-links --track-type video --track-index 0 --clip-index 2
[--sequence-name NAME]` — clips linked to the addressed clip (same
addressing as `get-full-clip-info`), matched heuristically by shared source
media and identical start time — there's no direct "get linked items" API.
Key fields: `linkedCount`, `linkedClips[].{nodeId,name,trackType,
trackIndex,startSeconds,endSeconds}`. live smoke-tested 2026-07-17.

`get-insertion-bin` — the bin currently focused in the Project panel (where
a new import would land). Key fields: `name`, `nodeId`, `treePath`. Not yet
live-tested.

`get-project-panel-metadata` — the Project panel's column/metadata
configuration as an XML string. Key field: `metadata`. Not yet
live-tested.
### Catalogs, markers & metadata reads

- `list-available-effects` — `premiere-cli list-available-effects`. Lists
  every video effect this Premiere install exposes via QE
  (`qe.project.getVideoEffectList()`), e.g. for looking up an exact name
  before scripting an effect-apply command. Result: `{"effects": [{"name",
  "index"}], "count"}`. live smoke-tested 2026-07-17.
- `list-available-audio-effects` — `premiere-cli list-available-audio-effects`.
  Same as above for `qe.project.getAudioEffectList()`. Result:
  `{"effects": [{"name", "index"}], "count"}`. live smoke-tested 2026-07-17.
- `list-available-transitions` — `premiere-cli list-available-transitions`.
  Lists video transitions via `qe.project.getVideoTransitionList()`; per
  `PREMIERE_API_NOTES.md`, PPro 2026 is known to return this list EMPTY
  even though by-name lookup still works, so an empty list falls back to
  probing a fixed set of common transition names via
  `getVideoTransitionByName` (entries carry `source: "list"` or
  `"byName"`). Result: `{"transitions", "count", "listError", "probedByName"}`.
  live smoke-tested 2026-07-17.
- `list-available-audio-transitions` — `premiere-cli list-available-audio-transitions`.
  Lists audio transitions via `qe.project.getAudioTransitionList()`, no
  fallback probing. Result: `{"transitions": [{"name", "index"}], "count"}`.
  live smoke-tested 2026-07-17.
- `list-markers` — `premiere-cli list-markers [--sequence-name NAME]`.
  Lists every marker on a sequence (defaults to the active one), same
  `getFirstMarker`/`getNextMarker` iteration as `get-full-sequence-info`.
  Result: `{"sequenceName", "markers": [{"name", "comments", "type",
  "startSeconds", "endSeconds", "guid"}], "markerCount"}`. live smoke-tested 2026-07-17.
- `get-clip-markers` — `premiere-cli get-clip-markers --track-type video
  --track-index 0 --clip-index 2 [--sequence-name NAME]`. Lists markers on
  one TIMELINE clip (addressed the same way as `get-full-clip-info` —
  `sequenceName?`/`trackType`/`trackIndex`/`clipIndex` — not a project-item
  `item_id` as the reference tool used, since clip-scoped commands in this
  panel address the timeline consistently). Result: `{"sequenceName",
  "trackType", "trackIndex", "clipIndex", "clipName", "markers", "markerCount"}`.
  live smoke-tested 2026-07-17.
- `get-sequence-markers-by-type` — `premiere-cli get-sequence-markers-by-type
  --type Chapter [--sequence-name NAME]`. Lists sequence markers whose
  `type` matches exactly one of `Comment`/`Chapter`/`Segmentation`/
  `WebLink`/`FlashCuePoint`. Result: `{"sequenceName", "type", "markers", "count"}`.
  live smoke-tested 2026-07-17.
- `get-item-metadata` — `premiere-cli get-item-metadata --node-id ID` or
  `--name NAME`. Reads a project item's Premiere project metadata
  (`item.getProjectMetadata()`) — at least one of `--node-id`/`--name` is
  required (client-side `exit 2` otherwise); `--node-id` is matched
  exactly, `--name` matches the first item found with that exact name in
  a depth-first bin walk. Result: `{"name", "nodeId", "mediaPath",
  "projectMetadata"}`. live smoke-tested 2026-07-17.
- `get-color-label` — `premiere-cli get-color-label --node-id ID`. Reads
  `item.getColorLabel()` for a project item (same nodeId/name addressing
  as `get-item-metadata`). Result: `{"name", "nodeId", "colorLabel"}` —
  `colorLabel` is `null` (not a command failure) if unreadable on this
  Premiere build. live smoke-tested 2026-07-17.
- `get-footage-interpretation` — `premiere-cli get-footage-interpretation
  --node-id ID`. Reads `item.getFootageInterpretation()` (alpha usage,
  field type, frame rate, alpha flags, pixel aspect ratio). Fails with
  `{"ok": false, "error": "no footage interpretation available for this
  item"}` if the item has none (e.g. a bin or sequence item) — that's a
  command-level failure, not a per-field null. Result: `{"name", "nodeId",
  "alphaUsage", "fieldType", "frameRate", "ignoreAlpha", "invertAlpha",
  "pixelAspectRatio"}`. live smoke-tested 2026-07-17.
- `get-xmp-metadata` — `premiere-cli get-xmp-metadata --node-id ID`. Reads
  the raw XMP XML via `item.getXMPMetadata()`; truncated to 100KB
  (102400 chars) with `truncated: true` if it was cut. Result: `{"name",
  "nodeId", "xmpMetadata", "truncated"}`. live smoke-tested 2026-07-17.
- `get-color-space` — `premiere-cli get-color-space --node-id ID`. Reads
  `item.getColorSpace()`/`getOriginalColorSpace()`/`getEmbeddedLUTID()`/
  `getInputLUTID()`, each independently best-effort. Result: `{"name",
  "nodeId", "colorSpace", "originalColorSpace", "embeddedLUT", "inputLUT"}`.
  live smoke-tested 2026-07-17.
- `get-render-queue-status` — `premiere-cli get-render-queue-status`. Per
  `PREMIERE_API_NOTES.md`, Adobe Media Encoder exposes no real
  queue-introspection API from ExtendScript, so this reports only
  `app.encoder.isRunning()` (if that method exists on this build) rather
  than inventing job/progress detail; fails if `app.encoder` itself is
  unavailable. Result: `{"isRunning", "info"}`. live smoke-tested 2026-07-17.
Ported from leancoderkavy's premiere-pro-mcp reference project. All are
read-only, standard-DOM only unless noted, and none are live-tested yet.

### Item metadata writes

Write-side counterparts of the metadata reads above. Every setter returns
`{previousValue, requestedValue, newValue, verified}` (field names vary
slightly per command) — undo is NON-FUNCTIONAL on this build, so
`previousValue` is the only restoration path if a caller needs to revert.
live smoke-tested 2026-07-17.

- `set-item-metadata --node-id ID --field-path "Column.Intrinsic.Description" --value "..."` —
  sets one project metadata field via `item.setProjectMetadata(value,
  [fieldPath])`. No single-field getter exists, so `previousValue`/`newValue`
  are the full metadata blob before/after; `verified` is a substring check.
- `set-color-label --node-id ID --color-label 0-15` — `item.setColorLabel()`;
  degrades honestly (getter and setter both) if unavailable on this build,
  mirroring `get-color-label`.
- `set-footage-interpretation --node-id ID [--frame-rate N] [--pixel-aspect-ratio N] [--field-type N] [--alpha-usage N]` —
  at least one field required. `getFootageInterpretation()` → mutate given
  fields → `setFootageInterpretation(i)` → re-get to verify; returns full
  before/after interpretation objects.
- `set-xmp-metadata --node-id ID (--xmp XML | --xmp-file PATH)` —
  **REPLACES THE ITEM'S ENTIRE XMP BLOCK**, not a merge — run
  `get-xmp-metadata` first and modify. XMP is too large for a command
  line, so `--xmp-file` reads a local file in the Python CLI (never on
  the ExtendScript side) and sends its contents. Read-back truncated to
  100KB like the getter.

### Clip, track & edit-point reads

- `premiere-cli get-clip-at-position --seconds 12.5 [--track-type video --track-index 0] [--sequence-name "Seq 01"]` — finds every clip covering a given time; `--track-index` requires `--track-type`, and omitting both scans every video/audio track. Result: `{"sequenceName", "seconds", "clipCount", "clips": [...]}`. Test status: request-body test only.
- `premiere-cli get-clip-at-playhead [--track-type video|audio|both] [--sequence-name ...]` — clips covering the current playhead across one or both track types (default both). Result: `{"playheadSeconds", "clipCount", "clips": [...]}`. Test status: request-body test only.
- `premiere-cli get-next-edit-point [--direction next|previous] [--track-type ...] [--sequence-name ...]` — nearest clip start/end boundary before/after the playhead. Result: `{"found", "direction", "editPointSeconds"?, "playheadSeconds"}`. Test status: request-body test only.
- `premiere-cli get-sequence-count` — number of sequences in the active project. Result: `{"count"}`. Test status: request-body test only.
- `premiere-cli get-total-clip-count [--sequence-name ...]` — clip counts across all tracks of a sequence. Result: `{"videoClips", "audioClips", "total"}`. Test status: request-body test only.
- `premiere-cli get-target-tracks [--sequence-name ...]` — which tracks are currently targeted for editing. Result: `{"video": [{"index","name"}], "audio": [...]}`. Test status: request-body test only.
- `premiere-cli get-track-info --track-type video --track-index 1 [--sequence-name ...]` — one track's full clip/transition list plus mute/lock/target state. Result: `{"name", "clipCount", "isMuted", "isLocked", "isTargeted", "clips": [...], "transitions": [...]}`. Test status: request-body test only.
- `premiere-cli get-encoder-presets [--format "H.264"]` — probes `app.encoder.getExporters()`/`.getPresets()`, an API flagged unconfirmed in `PREMIERE_API_NOTES.md`; an unsupported build returns `{"available": false, "error": "..."}` rather than failing. Result: `{"available", "presets": [{"exporterName","name","path"}], "count"}`. Test status: request-body test only.
- `premiere-cli get-qe-clip-info --track-type video --track-index 0 --clip-index 2 [--sequence-name ...]` — QE-DOM clip detail beyond what the standard DOM exposes; activates the resolved sequence first (QE only reads the active sequence). Time fields are suffixed `Seconds` since QE uses `.secs`, not `.seconds`. Test status: request-body test only.
- `premiere-cli get-source-monitor-info` — info about the clip open in the Source Monitor; no clip loaded is a valid `{"loaded": false}`, not an error. Test status: request-body test only.
- `premiere-cli get-clip-adjustment-layer --track-type video --track-index 0 --clip-index 1 [--sequence-name ...]` — whether the addressed clip is an adjustment layer. Result: `{"clipName", "isAdjustmentLayer"}`. Test status: request-body test only.

### Markers & history

- `premiere-cli add-marker --seconds 5.0 [--name "Chapter 1"] [--comments ...] [--type Chapter] [--duration-seconds 2.0] [--color-index 3] [--sequence-name ...]` — adds a marker to a sequence via `markers.createMarker()`. Verified by re-finding the new marker by `guid` after creation (never trusting `createMarker()`'s own returned object). Result: `{"sequenceName", "marker": {"name","comments","type","startSeconds","endSeconds","guid"}}`. live smoke-tested 2026-07-17.
- `premiere-cli update-marker --guid GUID [--name ...] [--comments ...] [--type ...] [--color-index N] [--duration-seconds N | --end-seconds N] [--sequence-name ...]` — updates a marker identified by `guid` (not by time-matching, unlike the reference tool). `marker.start` is not mutable here. Verified by re-reading the marker fresh after mutating. Result adds `applied` (per-field success map). live smoke-tested 2026-07-17.
- `premiere-cli delete-marker --guid GUID [--sequence-name ...]` — deletes a marker identified by `guid` via `markers.deleteMarker()`. Verified by confirming the marker count dropped by exactly one and the guid is gone from a fresh walk. Result: `{"deleted": true, "markerCountBefore", "markerCountAfter"}`. live smoke-tested 2026-07-17.
- `premiere-cli add-marker-to-project-item (--node-id ID | --name NAME) --seconds 3.0 [--marker-name ...] [--comments ...] [--type ...] [--duration-seconds N] [--color-index N]` — adds a SOURCE marker to a project item via `item.getMarkers()`; the marker's label is `--marker-name` (not `--name`, which addresses the item, same convention as `get-item-metadata`). Verified the same way as `add-marker`. live smoke-tested 2026-07-17.
- `premiere-cli redo` — REWRITTEN 2026-07-17: `qe.project.redo()` WORKS on this build (the earlier "non-functional" finding was wrong — see the correction note at the top of this section). Verified via `qe.project.undoStackIndex()` incrementing. Result: `{"redone", "verified", "method", "undoStackIndexBefore", "undoStackIndexAfter", "note"}`. Underlying API live-verified 2026-07-17; the rewritten command itself takes effect on the next panel reload.
- `premiere-cli undo [--count N]` — REWRITTEN 2026-07-17: `qe.project.undo()` WORKS for operations that entered the undo stack (see the correction note at the top of this section); each iteration is verified via `qe.project.undoStackIndex()` decrementing, and the loop stops as soon as the index stops moving. **CAUTION**: marker adds and track renames never enter the stack — undoing right after one of those pops the next stack entry instead (possibly a user edit). Result: `{"undoneCount", "stackExhausted", "undoStackIndexBefore", "undoStackIndexAfter", ...}`. Underlying API live-verified 2026-07-17; the rewritten command itself takes effect on the next panel reload.
- `premiere-cli move-playhead-to-edit [--direction next|previous] [--sequence-name ...]` — finds the nearest clip boundary before/after the playhead (same search as `get-next-edit-point`) and seeks there using `move-playhead`'s ticks-string/Time-object pattern, verified via a `getPlayerPosition()` read-back. Result: `{"editPointSeconds", "playheadSeconds", "attempts"}`. live smoke-tested 2026-07-17.
- `premiere-cli set-poster-frame (--node-id ID | --name NAME) --seconds N` — **API uncertain**: no poster-frame method is documented anywhere in `PREMIERE_API_NOTES.md`. Probes several plausible setter names and, if one doesn't throw, probes plausible getters to attempt a read-back; `verified: false` means unconfirmed, not a real success. live smoke-tested 2026-07-17.
  LIVE FINDING 2026-07-17: no poster-frame API on this build — returns the probe-failure error.
- `premiere-cli select-project-item (--node-id ID | --name NAME)` — calls `item.select()` on a Project-panel item (renamed from the reference's `select_item`). No confirmed selection read-back API exists — the result honestly reports only that the call didn't throw. live smoke-tested 2026-07-17.
### Timeline selection

Ported from leancoderkavy's premiere-pro-mcp `selection.ts`/`advanced.ts`. All
are MUTATING (they change Premiere's clip selection) and standard-DOM only
(`clip.setSelected()`/`isSelected()`). Every command verifies the outcome via
a fresh `seq.getSelection()` read-back afterward — never the setter's own
return value — and returns `{"selectedCount", "selectedClips": [...up to 20],
"truncated"?}` alongside command-specific fields. live smoke-tested 2026-07-17.

- `premiere-cli select-clips-by-name --name-contains STR [--add-to-selection] [--sequence-name ...]` — selects every clip (video+audio) whose name contains `STR` (case-insensitive); replaces the existing selection unless `--add-to-selection` is given.
- `premiere-cli select-all-clips [--track-type video|audio|both] [--sequence-name ...]` — selects every clip on the given track type (default `both`).
- `premiere-cli deselect-all-clips [--sequence-name ...]` — deselects every clip across all video/audio tracks.
- `premiere-cli select-clips-in-range --start-seconds N --end-seconds N [--sequence-name ...]` — replaces the selection with every clip that OVERLAPS `[start, end)` (partial overlap qualifies, not just full containment).
- `premiere-cli select-clips-by-color --color-label N [--sequence-name ...]` — replaces the selection with clips whose source `projectItem.getColorLabel()` equals `N` (0-15); fails honestly with `"getColorLabel is not available on this Premiere build"` if that getter is absent, same degrade pattern as `search-project-items`.
- `premiere-cli invert-selection [--sequence-name ...]` — flips every clip's selection state; a clip whose `isSelected()` throws is skipped (left untouched), tracked in `skippedUnreadable`.
- `premiere-cli select-disabled-clips [--sequence-name ...]` — replaces the selection with clips where `clip.disabled === true`; unreadable clips are skipped (`skippedUnreadable`).
- `premiere-cli set-clip-selection --track-type video|audio --track-index N --clip-index N (--select|--deselect) [--sequence-name ...]` — selects/deselects one clip, addressed the same way as `get-full-clip-info`; result also includes `actualSelected` (a direct `clip.isSelected()` read-back on the addressed clip, independent of the sequence-wide `selectedClips` list).
### Track management

Mutation ports of leancoderkavy's premiere-pro-mcp track tools. Every one
verifies its own mutation by reading the affected state back afterward —
that read-back is the result's proof, not the mutation call's return
value. live smoke-tested 2026-07-17.

- `premiere-cli add-track --track-type video|audio [--index N] [--count N] [--sequence-name ...]` — adds track(s) one at a time (never a bulk call — see `PREMIERE_API_NOTES.md`'s CEP-bridge-wedging warning), `--count` capped at 8. Merges the reference's `add_track` and `add_tracks` tools. Tries standard-DOM insert, then `addTrack`, then two QE `addTracks` signatures. Verified by `numTracks` increasing. Result: `{"added", "totalTracks", "attempts"}`.
- `premiere-cli lock-track --track-type video|audio --track-index N --locked true|false [--sequence-name ...]` — locks/unlocks a track (`setLocked`, falling back to `setLock`). Verified via `isLocked()`.
- `premiere-cli set-track-visibility --track-index N --visible true|false [--sequence-name ...]` — video-only; renamed from the reference's toggle to an explicit set. **Caveat**: no distinct visibility flag exists — this is `track.setMute()` on a video track with the sense inverted. Verified via `isMuted()`.
- `premiere-cli set-track-mute --track-type video|audio --track-index N --muted true|false [--sequence-name ...]` — mutes/unmutes a track; renamed from the reference's toggle to an explicit set, generalized to both track types. Verified via `isMuted()`.
- `premiere-cli rename-track --track-type video|audio --track-index N --name "..." [--sequence-name ...]` — renames a track via `track.name` assignment. Verified by reading `track.name` back.
- `premiere-cli set-target-track --track-type video|audio --track-index N --targeted true|false [--sequence-name ...]` — targets/untargets a track for insert/overwrite edits. Verified via `isTargeted()`.
- `premiere-cli set-all-tracks-targeted --targeted true|false [--track-type video|audio|both] [--sequence-name ...]` — targets/untargets every track of the given type(s). Verified with a per-track `isTargeted()` read-back summary. Result includes `tracksAffected` and per-track `video`/`audio` arrays.

### Clip transform & opacity

Mutation ports of leancoderkavy's premiere-pro-mcp `track-targeting.ts`/
`clipboard.ts` property setters (wave 3), addressed the same way as
`get-full-clip-info` (`--track-type`/`--track-index`/`--clip-index`, not
node_id). All build on shared `host/index.jsx` helpers (`findClipComponent`,
`setComponentProperty`) that identify components by matchName (never index
alone) and properties by displayName (exact, then normalized-lowercase —
locale-dependent). Every setter reads the property before AND after
`setValue()`, returning `{"previousValue", "requestedValue", "newValue",
"verified"}` — **undo is non-functional on this build**, so `previousValue`
is the only restoration path if a mutation turns out unwanted. Design-only —
live smoke-tested 2026-07-17.

- `premiere-cli set-clip-position --track-type video|audio --track-index N --clip-index N --x N --y N [--sequence-name ...]` — sets Motion/Position (pixels).
- `premiere-cli set-clip-scale --track-type video|audio --track-index N --clip-index N --scale N [--sequence-name ...]` — sets Motion/Scale (100 = original size).
- `premiere-cli set-clip-rotation --track-type video|audio --track-index N --clip-index N --degrees N [--sequence-name ...]` — sets Motion/Rotation (degrees; can exceed 360 for multiple rotations).
- `premiere-cli set-clip-anchor-point --track-type video|audio --track-index N --clip-index N --x N --y N [--sequence-name ...]` — sets Motion/Anchor Point (pixels).
- `premiere-cli set-clip-opacity --track-type video|audio --track-index N --clip-index N --opacity N [--sequence-name ...]` — sets Opacity/Opacity (0-100).
- `premiere-cli set-uniform-scale --track-type video|audio --track-index N --clip-index N --uniform true|false [--sequence-name ...]` — sets Motion/Uniform Scale, which links (true) or unlinks (false) Scale Width/Scale Height.
- `premiere-cli set-scale-width-height --track-type video|audio --track-index N --clip-index N [--scale-width N] [--scale-height N] [--sequence-name ...]` — sets Scale Width and/or Scale Height independently; unlike the reference tool (which silently force-disables Uniform Scale first), this reads Uniform Scale and fails with an informative error if it's on rather than flipping it for the caller — call `set-uniform-scale --uniform false` first.
- `premiere-cli set-anti-alias-quality --track-type video|audio --track-index N --clip-index N --enabled true|false [--sequence-name ...]` — **API uncertain**: no anti-alias property is documented for the Motion component; probes a short list of plausible displayNames and returns an honest "not found on this build" error (naming every name tried) if none exist.
- `premiere-cli set-blend-mode --track-type video|audio --track-index N --clip-index N --blend-mode N [--sequence-name ...]` — sets Opacity/Blend Mode to a raw int. **API uncertain**: the int↔name enum mapping is version-dependent across Premiere builds (reference repos disagree on map size/order), so this command takes/returns the raw int only — no name-mapping table of our own.
### Clip audio & flags

Mutation ports of leancoderkavy's premiere-pro-mcp audio/timeline/advanced
tools (wave 3). Clip addressing matches `get-full-clip-info`
(`--track-type video|audio --track-index N --clip-index N [--sequence-name
...]`). live smoke-tested 2026-07-17 — undo is non-functional on this build, so
`previousValue` in each result is the only restoration path.

- `premiere-cli set-clip-volume --track-type ... --track-index N --clip-index N --db N [--sequence-name ...]` — sets the Volume/Level property. **dB calibration is UNVERIFIED**: Level is linear amplitude, not dB; converted via hetpatel's `linear = 10^((db-15)/20)`, which disagrees with the other two reference repos. Result includes raw linear `previousValue`/`requestedValue`/`newValue` plus `previousDbEstimate` so a caller can recalibrate.
- `premiere-cli set-clip-pan --track-type ... --track-index N --clip-index N --pan N [--sequence-name ...]` — sets the Panner Balance/Pan property, -100 (full left) to 100 (full right). No dB conversion involved.
- `premiere-cli adjust-audio-levels --track-type ... --track-index N --clip-index N --db N [--sequence-name ...]` — `--db` is a DELTA relative to the clip's current level, not an absolute value: reads current linear Level, estimates its dB via the same calibration as `set-clip-volume`, adds the delta, writes back. Same calibration uncertainty, compounded.
- `premiere-cli add-audio-keyframes --track-type ... --track-index N --clip-index N --keyframes '[{"seconds":0,"db":-60},{"seconds":1,"db":0}]' [--sequence-name ...]` — adds Level keyframes; `seconds` is clip-relative, offset internally by `clip.start` to sequence time. Calls `setTimeVarying(true)` first. **Uses a DIFFERENT, uncalibrated dB->linear formula** (`10^(db/20)`) from `set-clip-volume` — the two commands' dB values are not interchangeable. Reports per-keyframe `{addKeySuccess, setValueSuccess}`.
- `premiere-cli rename-clip --track-type ... --track-index N --clip-index N --name "..." [--sequence-name ...]` — renames a timeline clip; tries standard-DOM `clip.name = x` first, falls back to QE `setName()`. Verified via read-back.
- `premiere-cli batch-rename-clips --track-type ... --track-index N --new-name-template "Scene_{n}" [--name-contains ...] [--start-number N] [--sequence-name ...]` — renames clips on one track via QE `setName()`; template supports `{n}` (counter) and `{name}` (existing name). Capped at 200 renames per call.
- `premiere-cli set-clip-enabled --track-type ... --track-index N --clip-index N --enabled true|false [--sequence-name ...]` — enables/disables a clip (renamed from the reference's `enable_disable_clip`); probes `clip.disabled = !enabled` then `clip.setDisabled(!enabled)`. Verified via read-back.
- `premiere-cli batch-set-clips-enabled --enabled true|false [--name-contains ...] [--track-type video|audio|both] [--sequence-name ...]` — enables/disables every matching clip; capped at 200. No `target: "selected"` equivalent — combine with `select-clips-by-name`/`--name-contains` instead.
- `premiere-cli set-frame-blend --track-type ... --track-index N --clip-index N --enabled true|false [--sequence-name ...]` — QE `qeClip.setFrameBlend(bool)`. Activates the sequence first; maps the standard-DOM clip index to the Nth non-`"Empty"` QE item (QE track item lists interleave gap items).
- `premiere-cli set-time-interpolation --track-type ... --track-index N --clip-index N --type 0|1|2 [--sequence-name ...]` — QE `qeClip.setTimeInterpolationType()` (0=sampling, 1=blending, 2=optical flow); same Empty-item-skipping QE addressing as `set-frame-blend`.
- `premiere-cli set-clip-properties --track-type ... --track-index N --clip-index N [--opacity N] [--speed N] [--scale N] [--position-x N] [--position-y N] [--rotation N] [--sequence-name ...]` — bulk setter over Motion/Opacity properties; at least one property flag required. Position sets X/Y together, preserving whichever axis wasn't specified.

### Effects

Mutation ports of leancoderkavy's premiere-pro-mcp `effects.ts`/`clipboard.ts` (wave 4). Clip addressing matches `get-full-clip-info` (`--track-type --track-index N --clip-index N [--sequence-name ...]`); every mutation reads `clip.components` before/after and reports counts AND names, never trusting a QE call's own return value. Undo is non-functional on this build, so `remove-effect`/`remove-effect-by-name` are the primary cleanup path — Motion/Opacity are always refused. live smoke-tested 2026-07-17.

- `premiere-cli apply-effect --track-type video --track-index N --clip-index N --effect-name "Gaussian Blur" [--sequence-name ...]` — applies a video effect via the standard QE dance (`getVideoEffectByName` + `addVideoEffect`); verified by `components.numItems` increasing by 1. See `list-available-effects` for exact names.
- `premiere-cli apply-audio-effect --track-type audio --track-index N --clip-index N --effect-name "..." [--sequence-name ...]` — audio counterpart via `getAudioEffectByName`/`addAudioEffect`.
- `premiere-cli remove-effect --track-type ... --track-index N --clip-index N --component-index N [--sequence-name ...]` — removes one component by index. `component.remove()` is DISPUTED across reference repos (works vs. impossible) — judged only by `components.numItems` dropping, not by whether the call throws. Refuses Motion/Opacity.
- `premiere-cli remove-effect-by-name --track-type ... --track-index N --clip-index N --effect-name "..." [--sequence-name ...]` — removes every component matching a displayName/matchName (backwards iteration); Motion/Opacity still refused even by name.
- `premiere-cli remove-all-effects --track-type ... --track-index N --clip-index N [--sequence-name ...]` — QE `qeClip.removeEffects()`; strips all applied effects in one call, built-ins survive per QE semantics.
- `premiere-cli color-correct --track-type video --track-index N --clip-index N [--exposure N] [--contrast N] [--saturation N] [--temperature N] [--tint N] [--sequence-name ...]` — applies Lumetri Color if missing, then sets each given control by displayName; Lumetri repeats displayNames across sub-sections, so each control tries every matching property until one write succeeds ("first writable match wins"). `0` is a legitimate value, never falsy-checked.
- `premiere-cli apply-lut --track-type video --track-index N --clip-index N --lut-path /path/to/look.cube [--sequence-name ...]` — **CORRECTED 2026-07-23**: ensures Lumetri Color is applied, but the `--lut-path` step itself always fails on this build — "Input LUT" is a registry-backed dropdown INDEX, not a settable path string (see BUILD_FINDINGS.md). Only useful now for the Lumetri-Color-ensure side effect. To switch to a LUT **already loaded** in Premiere's dropdown, use `set-effect-property --component-name "Lumetri Color" --property-name "Input LUT" --value N --value-type number` instead. To load a **brand-new** `.cube` file Premiere hasn't seen before: driving the native UI directly is the only way to do this on this build, but there's currently no automated skill/command for it — the `desktop-set-input-lut` command that used to do this was removed 2026-07-23 pending a rebuild on `premiere-cli`'s generic `desktop-*` UI-automation primitives (`desktop-take-screenshot`, `desktop-press-key`, `desktop-enter-text`, `desktop-move-mouse`, `desktop-click-mouse`, etc.) plus AI vision-language screen understanding. Until that lands, load new LUTs manually via Effect Controls > Lumetri Color > Basic Correction > Input LUT > Browse….
- `premiere-cli stabilize-clip --track-type video --track-index N --clip-index N [--smoothness N] [--method "Subspace Warp"|"Position"|"Position, Scale, Rotation"] [--sequence-name ...]` — applies Warp Stabilizer; analysis itself runs asynchronously in Premiere afterward.
- `premiere-cli copy-effects-between-clips --source-track-type ... --source-track-index N --source-clip-index N --target-track-type ... --target-track-index N --target-clip-index N [--effect-name "..."] [--source-sequence-name ...] [--target-sequence-name ...]` — re-applies each non-intrinsic effect (or just `--effect-name`) from source to target via QE, then copies property values across. Source/target may be on different sequences.
- `premiere-cli copy-effect-values --source-track-type ... --source-track-index N --source-clip-index N --target-track-type ... --target-track-index N --target-clip-index N --effect-name "..." [--source-sequence-name ...] [--target-sequence-name ...]` — copies one effect's property VALUES only; the effect must already exist on BOTH clips.
- `premiere-cli batch-apply-effect --effect-name "..." [--name-contains ...] [--track-type video|audio|both] [--track-index N] [--sequence-name ...]` — applies an effect to every matching clip, capped at 100; no `target: "selected"` equivalent — combine with `select-clips-by-name` instead.
### Keyframes & effect properties

Wave-4 ports of leancoderkavy's premiere-pro-mcp `keyframes.ts`/`advanced.ts`
(`set_color_value`). Addressed by `--track-type/--track-index/--clip-index`
(same as `get-full-clip-info`) plus `--component-name`/`--property-name` —
`--component-name` is matched generically against BOTH matchName and
displayName (any component, not just Motion/Opacity). `--seconds`/
`--start-seconds`/`--end-seconds` are **clip-relative**, offset internally by
`clip.start` to sequence time. Key-time argument form is disputed across
Premiere builds — mutating commands try a ticksString, then a `Time` object,
then a raw seconds number, recording every attempt. live smoke-tested 2026-07-17;
undo is non-functional on this build, so read-back `previousValue`s are the
only restoration path.

- `premiere-cli get-effect-properties --track-type ... --track-index N --clip-index N --component-name "Motion" [--sequence-name ...]` — lists ALL properties of one named component (value, isTimeVarying, keyCount), uncapped.
- `premiere-cli set-effect-property --track-type ... --track-index N --clip-index N --component-name "Motion" --property-name "Scale" --value 150 --value-type number [--sequence-name ...]` — sets one property; `--value-type` (`number`/`string`/`boolean`) disambiguates how `--value` is coerced before sending.
- `premiere-cli get-keyframes --track-type ... --track-index N --clip-index N --component-name "Motion" --property-name "Position" [--sequence-name ...]` — lists every keyframe as `{sequenceSeconds, clipRelativeSeconds, value}`. Key-time representation is disputed across builds — normalized via an unverified heuristic.
- `premiere-cli add-keyframe --track-type ... --track-index N --clip-index N --component-name "Motion" --property-name "Scale" --seconds 1.5 --value 120 [--sequence-name ...]` — adds a keyframe, calling `setTimeVarying(true)` first if needed. `verified` is a keyframe-COUNT check only, not an exact-time match.
- `premiere-cli remove-keyframe --track-type ... --track-index N --clip-index N --component-name "Motion" --property-name "Scale" --seconds 1.5 [--sequence-name ...]` — removes a keyframe within ±half a frame; the cleanup path for `add-keyframe` mistakes.
- `premiere-cli remove-keyframe-range --track-type ... --track-index N --clip-index N --component-name "Motion" --property-name "Scale" --start-seconds 0 --end-seconds 2 [--sequence-name ...]` — removes all keyframes in a clip-relative range via `removeKeyRange()`.
- `premiere-cli set-keyframe-interpolation --track-type ... --track-index N --clip-index N --component-name "Motion" --property-name "Scale" --seconds 1.5 --interpolation-type 5 [--sequence-name ...]` — sets interpolation to a **raw int** whose enum meaning is version-dependent (0=Linear/4=Hold/5=Bezier vs. a 0/1/2 map, per two disagreeing reference repos) — no translation applied.
- `premiere-cli get-value-at-time --track-type ... --track-index N --clip-index N --component-name "Motion" --property-name "Scale" --seconds 1.5 [--sequence-name ...]` — reads a property's interpolated value at a clip-relative time.
- `premiere-cli set-color-value --track-type ... --track-index N --clip-index N --component-name "Lumetri Color" --property-name "Tint" --alpha 255 --red 128 --green 64 --blue 32 [--sequence-name ...]` — sets a color-typed property (e.g. Lumetri tint, title fill), each channel 0-255.
### Transitions

Mutation ports of leancoderkavy's premiere-pro-mcp `transitions.ts` (wave 4).
`add-transition` merges the reference's `add_transition`/`add_transition_to_clip`
into one command, addressed like `get-full-clip-info` (`--track-index
--clip-index`, not node_id/cut-point-seconds). Apply is QE-only
(`qe.project.getVideoTransitionByName()` then a disputed `addTransition()`
signature — arity 3-7, duration as seconds-string/ticks-string/number,
tried on both `qeClip` and `qeTrack`); every attempt is verified via a
`track.transitions.numItems` delta, never the call's return value. Not yet
live-tested.

- `premiere-cli add-transition --track-type video --track-index N --clip-index N --at-end true|false [--transition-name "Cross Dissolve"] [--duration-seconds N] [--sequence-name ...]` — adds a transition at one clip's start (`--at-end false`) or end (`--at-end true`); omit `--transition-name` for the default transition. Video only.
- `premiere-cli batch-add-transitions [--track-index N] [--at-end true|false] [--transition-name "..."] [--duration-seconds N] [--sequence-name ...]` — applies the same transition to every cut point on one video track (default track 0, `--at-end` default true); capped at 100 clips per call.
- `premiere-cli remove-transition --track-type video|audio --track-index N --transition-index N [--sequence-name ...]` — **new, not a reference port**: cleanup for the two commands above since undo is non-functional on this build. Standard-DOM `track.transitions[i].remove(...)`, arity probed via an attempts array, verified via a `numItems` drop of exactly one. An out-of-range `--transition-index` gets a transitions listing back (index + name) to help find the right one.

### Timeline editing

Mutation ports of leancoderkavy's premiere-pro-mcp `timeline.ts`/`advanced.ts`
(wave 5) — the highest-risk tier: these directly add, remove, move, trim,
split, duplicate, replace, and re-speed clips. Undo is NON-FUNCTIONAL on this
build, so every command re-reads the timeline afterward (clip counts,
start/end times) rather than trusting a call's own return value. Not yet
live-tested.

- `premiere-cli add-to-timeline --node-id ID|--name NAME --track-type video|audio --track-index N --start-seconds N --mode insert|overwrite [--sequence-name ...]` — UPDATED 2026-07-17: places a project item via the TRACK object's own `track.insertClip(item, TimeObject)`/`overwriteClip(item, TimeObject)`, which HONORS the track index (probe-verified — the sequence-level `seq.insertClip()`/`overwriteClip()` ignore their video-track index on this build and are kept only as a fallback; check `placedVia`/`trackHonored` in the result). For video placements, auto-scans every audio track afterward for the source's own linked-audio clip landing near the requested time and removes it (the auto-linked-audio trap in `PREMIERE_API_NOTES.md`) — reports `linkedAudioCleanup`. The updated placement path takes effect on the next panel reload.
- `premiere-cli remove-from-timeline --track-type ... --track-index N --clip-index N --ripple true|false [--sequence-name ...]` — **Destructive**: permanently removes a clip via `clip.remove(ripple, false)`; verified via a `numItems` drop of exactly one.
- `premiere-cli move-clip --track-type ... --track-index N --clip-index N --start-seconds N [--sequence-name ...]` — moves a clip to a new absolute start time on the SAME track via `clip.start` assignment (does not change track, per the documented API limitation); verified via a read-back.
- `premiere-cli trim-clip --track-type ... --track-index N --clip-index N [--in-point-seconds N] [--out-point-seconds N] [--sequence-name ...]` — trims in and/or out point (source-media-relative seconds); each field verified independently.
- `premiere-cli split-clip --track-type ... --track-index N --seconds N [--sequence-name ...]` — razors whichever clip covers a sequence-time position via QE `qeTrack.razor()`; verified via the track's clip count increasing by exactly one.
- `premiere-cli duplicate-clip --track-type ... --track-index N --clip-index N [--target-start-seconds N] [--sequence-name ...]` — inserts a new instance of the clip's own project item onto the SAME track (ripple insert, so never destructive), defaulting the target start to right after the original's own end.
- `premiere-cli replace-clip --track-type ... --track-index N --clip-index N --replacement-node-id ID|--replacement-name NAME [--sequence-name ...]` — **Destructive**: removes a clip and reinserts a different project item at the same start time; verified by re-finding a clip there whose media path matches the replacement.
- `premiere-cli set-clip-speed --track-type ... --track-index N --clip-index N --speed-percent N [--sequence-name ...]` — REWRITTEN + LIVE-VERIFIED 2026-07-17: sets playback speed via the calibrated 5-arg `qeClip.setSpeed(multiplier, durationTicksString, reverse, false, false)` (the only working form on this build — speeds are MULTIPLIERS internally, 1 = 100%). Negative `--speed-percent` = reversed. Speed-up shortens the clip (frame-rounded, so the applied speed can differ slightly); slow-motion does NOT extend the timeline item. Verified via `clip.getSpeed()`/`isSpeedReversed()` read-backs; result includes `newSpeedPercent` and `verified`. (`--ripple`/`--maintain-audio` are still accepted for compatibility but IGNORED — the working signature has no such knobs.)
- `premiere-cli get-clip-speed --track-type ... --track-index N --clip-index N [--sequence-name ...]` — read-only: `clip.getSpeed()`/`isSpeedReversed()`.
### Advanced timeline edits

Wave-5 mutation ports of leancoderkavy's premiere-pro-mcp `advanced.ts`. Clip
addressing matches `get-full-clip-info` (`--track-type --track-index N
--clip-index N [--sequence-name ...]`). Several of these QE methods
(`roll`/`slide`/`slip`/`moveToTrack`) have argument signatures that are
UNCONFIRMED or DISPUTED on this build — every plausible form is tried and
recorded in an `attempts` array, and success is judged by an actual
before/after state change, never by whether the call merely avoided
throwing. Undo is non-functional on this build, so there is no way back
from the destructive commands short of re-doing the edit by hand. Not yet
live-tested.

- `premiere-cli ripple-delete-clip --track-type ... --track-index N --clip-index N [--sequence-name ...]` — **destructive**: ripple-deletes one clip and closes the gap (single-clip counterpart of `remove-track-intervals`). Tries `qeClip.rippleDelete()`, falling back to `remove(bool,bool)` attempts, judged by an actual QE-track `numItems` drop.
- `premiere-cli roll-edit --track-type ... --track-index N --clip-index N --offset-seconds N [--sequence-name ...]` — rolls the edit point between the clip and its neighbor via `qeClip.roll(...)`; positive rolls later, negative rolls earlier.
- `premiere-cli slide-edit --track-type ... --track-index N --clip-index N --offset-seconds N [--sequence-name ...]` — slides the clip earlier/later without changing its own duration via `qeClip.slide(...)`; adjacent clips absorb the shift.
- `premiere-cli slip-edit --track-type ... --track-index N --clip-index N --offset-seconds N [--sequence-name ...]` — slips the clip's source in/out points without moving it on the timeline via `qeClip.slip(...)`.
- `premiere-cli move-clip-to-track --track-type ... --track-index N --clip-index N --target-track-index N [--sequence-name ...]` — REWRITTEN + LIVE-VERIFIED 2026-07-17: a TRUE move via the 4-arg `qeClip.moveToTrack(videoOffset, audioOffset, "0", false)` (RELATIVE track offsets; found in the antipaster repo). Per-clip effects/keyframes survive (verified with a Gaussian Blur) and the start time is preserved — the old lossy remove+overwrite path is gone. Verified by re-finding the clip on the target track; check `positionPreserved`. The audio-clip arg order is assumed by symmetry with a fallback (video form is the live-verified one).
- `premiere-cli reverse-clip --track-type ... --track-index N --clip-index N [--reverse true|false] [--sequence-name ...]` — REWRITTEN + LIVE-VERIFIED 2026-07-17: reverses (default) or un-reverses playback via the calibrated `qeClip.setSpeed(multiplier, ticksString, REVERSE, false, false)` — the third argument is the reverse flag; negative speeds do NOT reverse on this build. Preserves the current speed magnitude; short-circuits if already in the requested state. Verified via `clip.isSpeedReversed()`.
- `premiere-cli link-selection [--sequence-name ...]` — links the currently-SELECTED video/audio clips via `seq.linkSelection()` (QE has no link/unlink method at all). Select clips first (e.g. `set-clip-selection`) — operates on whatever is already selected. No confirmed "is linked" read-back exists; reports before/after selection instead.
- `premiere-cli unlink-selection [--sequence-name ...]` — unlinks the currently-selected linked clip(s) via `seq.unlinkSelection()`; same selection-based approach as `link-selection`.
- `premiere-cli overwrite-clip-at (--item-node-id ID | --item-name "...") --track-type ... --track-index N --start-seconds N [--sequence-name ...]` — **destructive**: overwrites a bin project item onto the timeline, replacing whatever's there. Tries `seq.overwriteClip(item, time, v, a)` (non-addressed side passed `-1`) across several time-argument forms, then a track-level fallback, then a QE last resort with no time control. Guards the **auto-linked-audio trap** (`PREMIERE_API_NOTES.md`): overwriting a video clip whose source has audio can silently place/destroy audio elsewhere on that track — this command snapshots audio tracks beforehand and removes any newly-added clip sharing the placed item's identity within 0.1s of the request (cannot recover audio the overwrite itself destroyed).
### Razor, selection ops & nesting

Mutation ports of leancoderkavy's premiere-pro-mcp `track-targeting.ts`,
`utility.ts`, and `sequence.ts` (wave 5). `set_clip_start_time` was
skipped (duplicate of an earlier wave's move-clip command). All are
destructive or semi-destructive — undo is non-functional on this build,
so verify on a duplicate/throwaway sequence before running against real
footage. None yet live-tested.

- `premiere-cli razor-all-tracks [--seconds N] [--sequence-name ...]` — razors every video+audio track at `--seconds` (default: the playhead) via QE `track.razor()`. Per-track verified by an item-count increase.
- `premiere-cli set-item-in-out (--node-id ID | --name NAME) [--in-seconds N] [--out-seconds N] [--media-type 1|2|4]` — sets in/out points on a PROJECT item (not a timeline clip) via `item.setInPoint()`/`setOutPoint()`. No confirmed getter exists for read-back on this build — see `note` in the result.
- `premiere-cli clear-item-in-out (--node-id ID | --name NAME) [--clear-in true|false] [--clear-out true|false]` — clears a project item's in/out points via `clearInPoint()`/`clearOutPoint()` (both default true).
- `premiere-cli clear-sequence-in-out [--sequence-name ...]` — resets the sequence's in/out points to Premiere's own -400000 "unset" sentinel (not a full-range set like the reference tool's own implementation), verified via a read-back reporting both points `null`.
- `premiere-cli remove-selected-clips [--ripple true|false] [--sequence-name ...]` — removes every currently-selected clip; `--ripple true` closes the gap (default false). Verified via a total-clip-count delta.
- `premiere-cli lift-selection [--sequence-name ...]` — lifts (removes WITHOUT closing the gap) content between the sequence's in/out points via QE `qeSeq.lift()`; requires in/out points set first (e.g. via `set-sequence-in-out`).
- `premiere-cli extract-selection [--sequence-name ...]` — extracts (removes AND closes the gap) content between the sequence's in/out points via QE `qeSeq.extract()`.
- `premiere-cli nest-clips --name "New Nest" [--sequence-name ...]` — nests the CURRENTLY SELECTED clips into a new nested sequence via `seq.createSubsequence(true)` (select clips first); renames the resulting sequence (and its Project-panel item) to `--name`.
- `premiere-cli freeze-frame --track-index N --clip-index N --output-path /path/to/freeze.png [--track-type video|audio] [--at-seconds N] [--sequence-name ...]` — mirrors the reference tool's ACTUAL behavior: exports a still and imports it as a project item. Does **not** place a frozen clip on the timeline via Time Remapping — the reference tool never does that either, despite the command's name.
- `premiere-cli match-frame [--track-type video|audio] [--track-index N] [--seconds N] [--sequence-name ...]` — finds the clip at a sequence position, computes the source time, and attempts `app.sourceMonitor.openProjectItem()` to load it into the Source Monitor (the reference tool itself never does this step, only computes the numbers). No confirmed API seeks the monitor to the exact offset afterward.
- `premiere-cli add-adjustment-layer [--track-index N] [--start-seconds N] [--duration-seconds N] [--sequence-name ...]` — creates an adjustment-layer project item via `qe.project.newAdjustmentLayer()` (PPro 2026's replacement for the removed `qeSeq.addAdjustmentLayer()`) and places it on a video track; item-creation and timeline-placement are verified independently. `--duration-seconds` is accepted but not yet applied.
- `premiere-cli unnest-sequence --node-id ID [--sequence-name ...]` — replaces a nested-sequence clip (addressed by its timeline nodeId) with copies of the nested sequence's own clips.

### Media & bins

Wave-6 mutation ports of leancoderkavy's premiere-pro-mcp `media.ts`/
`track-targeting.ts`/`utility.ts`/`project.ts`. Project-item addressing is
`--node-id ID` or `--name NAME` (one required); bin addressing is a
'/'-separated path, same convention as `create-sequence`'s `--bin`. Every
mutation is read-back-verified where a getter exists; where none is
documented, the result honestly reports that only the call's non-throw is
confirmed. Undo is non-functional on this build, so `previousValue` (where
present) is the only restoration path. None yet live-tested.

- `premiere-cli import-media (--file-path PATH | --file-paths JSON_ARRAY) [--target-bin-path PATH]` — imports file(s) via `app.project.importFiles()`. Pre-filtered to a safe extension allowlist (video/audio/image/`.prproj`) and REFUSED if any file's extension is unrecognized — an unsupported format can pop a BLOCKING dialog that freezes the bridge. Verified by diffing whole-project nodeId snapshots before/after, never by trusting `importFiles()`'s own return value. `--target-bin-path` must already exist.
- `premiere-cli import-folder --folder-path PATH [--target-bin-path PATH]` — imports every allowlisted file in a folder; non-matching files are skipped and reported, not attempted.
- `premiere-cli import-image-sequence --first-frame-path PATH [--target-bin-path PATH]` — imports a numbered image sequence as ONE clip (`asNumberedStills=true`); only the first frame's path is given.
- `premiere-cli create-bin --bin-path "B-Roll/Interviews"` — creates every missing segment of the bin path, reporting created vs. already-existing segments.
- `premiere-cli rename-bin --bin-path "B-Roll" --new-name "Archive"` — renames an EXISTING bin (never auto-created) via `bin.renameBin()`.
- `premiere-cli move-items-to-bin (--node-ids JSON_ARRAY | --node-id ID | --name NAME) --target-bin-path PATH` — **merges** the reference project's single-item and multi-item move tools into one command; target bin must already exist (`create-bin` first). Each move verified via the item's own `treePath` afterward.
- `premiere-cli relink-media (--node-id ID | --name NAME) --new-path PATH` — checks `canChangeMediaPath()` first, then `item.changeMediaPath(newPath, true)`; verified via `getMediaPath()`.
- `premiere-cli refresh-media (--node-id ID | --name NAME)` — `item.refreshMedia()`; no confirmed getter to detect a real content change.
- `premiere-cli set-item-offline (--node-id ID | --name NAME)` — **Destructive-ish**: unlinks the item's media via `item.setOffline()`; verified via `isOffline()`. `relink-media` is the manual recovery path.
- `premiere-cli detach-proxy (--node-id ID | --name NAME)` — checks `hasProxy()` first, then `item.detachProxy()`; verified via `hasProxy()`.
- `premiere-cli set-override-frame-rate (--node-id ID | --name NAME) --fps N` — sets `item.setOverrideFrameRate(fps)`; verified via `getFootageInterpretation().frameRate`.
- `premiere-cli set-override-pixel-aspect-ratio (--node-id ID | --name NAME) --numerator N --denominator N` — sets `item.setOverridePixelAspectRatio(num, den)`; verified via `getFootageInterpretation().pixelAspectRatio`.
- `premiere-cli set-scale-to-frame-size (--node-id ID | --name NAME)` — `item.setScaleToFrameSize()`; no confirmed getter to read the flag back.
- `premiere-cli set-item-start-time (--node-id ID | --name NAME) --seconds N` — converts to a ticks string and calls `item.setStartTime()` (source-media timecode offset). **API uncertain**: no confirmed getter exists on this build — a plausible getter name list is probed defensively.
- `premiere-cli rename-project-item (--node-id ID | --name NAME) --new-name NAME` — renames via `item.name = x`; verified via a name read-back.
### Project management

Ports of leancoderkavy's premiere-pro-mcp `project.ts` (wave 6). Mutations
are read-back-verified where a getter exists; undo is non-functional on
this build, so `previousValue`/`countBefore` fields are the only
restoration path. None yet live-tested.

- `premiere-cli save-project` — saves the open project via `app.project.save()`. Verified via `isDirty()` if that getter exists on this build, else call-level only.
- `premiere-cli save-project-as --path /path/to/new.prproj` — saves to a new path via `saveAs()`. **NOTE: switches the currently open project to the new file** — not a "save a copy."
- `premiere-cli open-project --path /path/to/project.prproj` — opens a project via `app.openDocument()` after a file-exists check. **WARN: may pop dialogs** that hang the bridge until dismissed in Premiere.
- `premiere-cli set-active-sequence --sequence-name "Sequence 01"` — makes a sequence active, trying assignment then `openSequence()`.
- `premiere-cli find-items-by-media-path --path-contains project0002` — READ; finds project items whose media path contains a substring, via `findItemsMatchingMediaPath()` with a manual bin-walk fallback.
- `premiere-cli create-smart-bin --name Offline --query "isOffline:true"` — creates a search bin via `rootItem.createSmartBin(name, query)`.
- `premiere-cli add-custom-metadata-field --name Shot --label "Shot Number" --type 2` — adds a **project-wide** metadata schema field (0=Integer, 1=Real, 2=String, 3=Boolean). Not removable from script.
- `premiere-cli import-sequences-from-project --project-path /path/to/source.prproj [--sequence-ids '["id1","id2"]']` — imports sequences from another project file; omit `--sequence-ids` for all.
- `premiere-cli import-fcp-xml --xml-path /path/to/cut.xml` — imports an FCP XML file, trying `importFiles()` then the legacy `openFCPXML()`. **WARN: may pop dialogs.**
- `premiere-cli import-ae-comps --aep-path /path/to/title.aep [--comp-names '["Comp 1"]'] [--target-bin-path "B-Roll/AE"]` — imports After Effects comps. **WARN: may pop dialogs.**
- `premiere-cli create-bars-and-tone [--width 1920] [--height 1080] [--name "..."]` — REWRITTEN 2026-07-17: the standard-DOM `app.project.newBarsAndTone()` throws "Illegal Parameter type" on this build, but the QE 2-arg `qe.project.newBarsAndTone(w, h)` WORKS (probe-verified; from the antipaster repo) — tried standard-DOM first, then QE. The QE form takes no name, so Premiere names the item itself (e.g. "HD Bars and Tone") and a requested `--name` is applied afterward via rename. Verified via a root-bin nodeId diff. Underlying API live-verified; the rewritten command takes effect on the next panel reload.
- `premiere-cli set-transcode-on-ingest --enabled true|false` — toggles transcode-on-ingest via `setEnableTranscodeOnIngest()` — an API not documented outside the reference tool; fails honestly if absent on this build.
- `premiere-cli set-project-panel-metadata --metadata '<xml/>'` — **REPLACES** the entire Project panel column configuration; not a merge — run `get-project-panel-metadata` first and modify.
- `premiere-cli get-graphics-white-luminance` — READ; the project's HDR white-luminance setting in nits.
- `premiere-cli set-graphics-white-luminance --value 100` — sets the HDR white-luminance value in nits.
### Sequence operations

Sequence-level ports from leancoderkavy's premiere-pro-mcp `sequence.ts`,
`utility.ts`, and `advanced.ts` (wave 6). Undo is non-functional on this
build. None yet live-tested.

- `premiere-cli duplicate-sequence --new-name "Copy" [--sequence-name ...]` — duplicates a sequence via `seq.clone()`, then renames the copy (and its Project-panel item, per the renaming gotcha) to `--new-name`.
- `premiere-cli set-sequence-settings [--frame-rate N] [--width N] [--height N] [--audio-sample-rate N] [--par-numerator N --par-denominator N] [--field-type 0|1|2] [--display-format N] [--sequence-name ...]` — merges 7 reference tools (generic settings + 6 individual setters) into one command; at least one field required. Each field is set and independently read-back-verified — `setSettings()` is documented buggy on this build, so a non-throwing call is never trusted as success (see per-field `applied`/`failed` in the result).
- `premiere-cli create-subsequence --ignore-track-targeting true|false [--sequence-name ...]` — a more direct port of `create_subsequence` than `nest-clips`: same underlying `seq.createSubsequence()` call, but doesn't enforce a selection or rename the result.
- `premiere-cli auto-reframe-sequence --numerator N --denominator N --motion-preset slower|default|faster --new-name "Vertical" --nest true|false [--sequence-name ...]` — reframes a sequence to a new aspect ratio via the 5-arg `seq.autoReframeSequence()`. **API uncertain**: numerator/denominator are treated as an aspect ratio per PREMIERE_API_NOTES.md, unconfirmed against this build (the reference tool's own implementation instead takes target width/height).
- `premiere-cli create-sequence-from-preset --name "My Seq" --preset-path /path/to/custom.sqpreset` — creates a sequence from a caller-supplied `.sqpreset` via the same no-dialog QE path `create-sequence` uses for its bundled preset.
- `premiere-cli create-sequence-from-clips --name "Assembled" --node-ids '["1","2"]' [--target-bin-path "B-Roll"]` — creates a sequence from an ordered list of project items via `app.project.createNewSequenceFromClips()`; `--target-bin-path` must already exist (not auto-created).
- `premiere-cli attach-custom-property --property-id myKey --value myValue [--sequence-name ...]` — attaches a custom key/value property via `seq.attachCustomProperty()`. **API uncertain**: no confirmed getter exists for read-back on any reference repo studied.
- `premiere-cli close-sequence [--sequence-name ...]` — closes a sequence's UI tab via `seq.close()` (does not delete the sequence from the project); reports whether it's still listed in `app.project.sequences` afterward.
- `premiere-cli export-sequence-as-project --output-path /path/to/out.prproj [--sequence-name ...]` — exports a sequence as a standalone `.prproj` via `seq.exportAsProject()`; verified via the filesystem.
- `premiere-cli scene-edit-detection --mode ApplyCuts|CreateMarkers --apply-to-linked-audio true|false --sensitivity Low|Medium|High [--sequence-name ...]` — runs scene edit detection via `seq.performSceneEditDetectionOnSelection()`. Operates on the current SELECTION — refuses if nothing is selected.
### Export & encoding

Mutation ports of leancoderkavy's premiere-pro-mcp `export.ts` (wave 6).
`export_frame`/`capture_frame` were skipped — this panel's own
`export-frame` command already covers that ground. None yet live-tested.
`add-to-render-queue`/`encode-project-item`/`encode-file` are all
**fire-and-forget**: Adobe Media Encoder exposes no progress API, so each
returns as soon as the job is queued, long before AME finishes writing the
file — none of them can verify the output the way the synchronous
`export-sequence`/`export-fcp-xml`/`export-aaf`/`export-omf` commands do.

- `premiere-cli export-sequence --output /path/to/out.mp4 --preset-path /path/to/preset.epr [--range entire|in-to-out|work-area] [--sequence-name ...]` — exports the sequence via the blocking `seq.exportAsMediaDirect()`, same synchronous mechanism as `extract-audio-track`/`export-frame`. `--preset-path` is required (no auto-discovered default). Verified via a filesystem check.
- `premiere-cli export-fcp-xml --output /path/to/out.xml [--sequence-name ...]` — exports the sequence as a Final Cut Pro XML file via `seq.exportAsFinalCutProXML()`. Verified via a filesystem check.
- `premiere-cli export-aaf --output /path/to/out.aaf [--mixdown true|false] [--mono true|false] [--rate N] [--bits N] [--sequence-name ...]` — exports as AAF via `seq.exportAsAAF()`; defaults (`mixdown=true`, `mono=false`, `rate=48000`, `bits=16`) mirror the reference tool. Verified via a filesystem check.
- `premiere-cli export-omf --output /path/to/out.omf [--title ...] [--rate N] [--bits N] [--audio-encapsulated true|false] [--audio-file-format 0|1] [--trim-audio-files true|false] [--handle-frames N] [--sequence-name ...]` — exports as OMF via `app.project.exportOMF()`, a 9-arg call whose exact signature is **UNCONFIRMED** against our own Premiere build. Verified via a filesystem check if the call succeeds.
- `premiere-cli add-to-render-queue --output /path/to/out.mp4 --preset-path /path/to/preset.epr [--range entire|in-to-out|work-area] [--start-batch true|false] [--sequence-name ...]` — queues the sequence in Adobe Media Encoder via `encodeSequence()`; `--start-batch` (default false) controls whether `startBatch()` fires immediately. **Fire-and-forget** — see above.
- `premiere-cli create-subclip (--node-id ID | --item-name NAME) --subclip-name NAME --in-seconds N --out-seconds N [--take-video true|false] [--take-audio true|false]` — creates a subclip via `item.createSubClip()`; trailing-arg order is **disputed** across reference repos, so several plausible orders are tried (`attempts` in the result), verified by a project-tree nodeId diff (never the call's own return value).
- `premiere-cli encode-project-item (--node-id ID | --name NAME) --output /path/to/out.mp4 --preset-path /path/to/preset.epr [--start-batch true|false]` — queues a PROJECT ITEM (not a sequence) via `app.encoder.encodeProjectItem()`. **Fire-and-forget** — see above.
- `premiere-cli encode-file --input /path/to/in.mov --output /path/to/out.mp4 --preset-path /path/to/preset.epr [--start-batch true|false]` — queues an EXTERNAL file (not necessarily in the project) via `app.encoder.encodeFile()`; encodes the whole file, no in/out trimming. **Fire-and-forget** — see above.
- `premiere-cli manage-proxies --action attach (--node-id ID | --name NAME) --proxy-path /path/to/proxy.mov [--is-hi-res true|false]` — attaches an already-rendered proxy file via `item.attachProxy()`, verified via `hasProxy()`/`getProxyPath()` where available.
- `premiere-cli manage-proxies --action enable|disable` — toggles proxy playback project-wide via `app.project.setProxyEnabled()`, verified via `isProxyEnabled()`. No item required. The reference tool's own `"create"` action (queue-a-proxy-encode) is deliberately dropped — compose `encode-project-item`/`add-to-render-queue` against a proxy preset instead.
### Source monitor, text & captions

Ports of leancoderkavy's premiere-pro-mcp `source-monitor.ts`, `text.ts`,
`captions.ts`, and `clipboard.ts`/`advanced.ts` (wave 6). No title-creation
API exists on this build (`createNewTitle` is gone, Essential Graphics
can't be scripted) — text/MOGRT/caption commands route around that per
`PREMIERE_API_NOTES.md`. None yet live-tested.

- `premiere-cli open-in-source (--node-id ID | --name NAME)` — opens a project item in the Source Monitor via `app.sourceMonitor.openProjectItem(item)`; verified via a `getProjectItem()` read-back.
- `premiere-cli close-source-monitor` — closes the single clip open in the Source Monitor via `app.sourceMonitor.closeClip()`.
- `premiere-cli close-all-source-clips` — closes every clip open across the Source Monitor's tabs via `app.sourceMonitor.closeAllClips()`.
- `premiere-cli set-source-in-out [--in-seconds N] [--out-seconds N]` — sets in/out points on whatever clip is open in the Source Monitor via `item.setInPoint()`/`setOutPoint()` (mediaType 4 = "all"); fails if no clip is loaded.
- `premiere-cli insert-from-source --track-type video|audio --track-index N [--at-seconds N] [--sequence-name ...]` — 3-point inserts the Source Monitor's clip (its own in/out points) at `--at-seconds` (default: the playhead) via `seq.insertClip()`. Same track-index-ignored caveat and auto-linked-audio cleanup as `add-to-timeline`.
- `premiere-cli overwrite-from-source --track-type video|audio --track-index N [--at-seconds N] [--sequence-name ...]` — **destructive**: same as `insert-from-source` but via `seq.overwriteClip()`, replacing whatever's at the target position.
- `premiere-cli add-text-overlay --text "..." [--start-seconds N] [--duration-seconds N] [--caption-format subtitle|608|708|teletext]` — mirrors leancoderkavy's `text.ts` `add_text_overlay` EXACTLY: a 1-arg `seq.createCaptionTrack(formatNum)` call + `addCaption()` + a `.text` assignment, since no title-creation API exists. **Documented, unresolved arity conflict**: this 1-arg form disagrees with the 3-arg signature `create-caption-track`/`PREMIERE_API_NOTES.md` document for the same call — unconfirmed on this build.
- `premiere-cli import-mogrt --mogrt-path /path/to/x.mogrt --start-seconds N --video-track-index N --audio-track-index N [--sequence-name ...]` — imports a `.mogrt` onto the timeline via `seq.importMGT(path, startTicks, v, a)`. Verified via the target video track's clip count, not the call's ambiguous return value (`text.ts` treats it as boolean; `PREMIERE_API_NOTES.md` documents a trackItem).
- `premiere-cli import-mogrt-from-library --library-name NAME --mogrt-name NAME --start-seconds N --video-track-index N --audio-track-index N [--sequence-name ...]` — imports via `seq.importMGTFromLibrary(libName, mogrtName, ticks, v, a)`. **Deliberate correction**: the reference tool's own call omits the library name (looks like a bug); this command requires it per `PREMIERE_API_NOTES.md`'s documented signature.
- `premiere-cli get-mogrt-component --track-type video|audio --track-index N --clip-index N [--sequence-name ...]` — read-only `clip.getMGTComponent()`, properties serialized like `get-effect-properties`. A MOGRT's Source Text value is a 4-byte-header+JSON binary blob per `PREMIERE_API_NOTES.md` — reported raw, never decoded or written.
- `premiere-cli create-caption-track (--node-id ID | --name NAME) --start-seconds N [--format subtitle|608|708|teletext|ebu|op42|op47]` — creates a caption track from an imported caption item (e.g. `.srt`) via `seq.createCaptionTrack(item, startSeconds, formatConstant)` — the 3-arg, integer-constant signature `PREMIERE_API_NOTES.md` documents. Resolves `Sequence.CAPTION_FORMAT_*` via a `typeof` guard, falling back to a known int for `subtitle`/`608`/`708`/`teletext` only. No caption READ API exists.
- `premiere-cli replace-clip-media (--node-id ID | --name NAME) --new-media-path /path/to/file` — **destructive-ish**: swaps a PROJECT ITEM's underlying media file via `item.canChangeMediaPath()` + `item.changeMediaPath(newPath, true)`, affecting every clip referencing it project-wide. Deliberately NOT a port of `clipboard.ts`'s `replace_clip_media` (that tool is actually a per-clip `overwriteClip()`, identical to this panel's existing `replace-clip`) — this does the item-level media swap `PREMIERE_API_NOTES.md` documents instead. Undo is non-functional — `previousMediaPath` in the result is the only restoration path.

## Desktop UI-automation commands (macOS only)

A separate mechanism from everything above: these `desktop-*` commands
never touch the Premiere Bridge panel's `/command` endpoint at all (except
`desktop-notify`/`desktop-dismiss-notifications`, which don't touch
Premiere either). Instead they drive the real, on-screen Premiere Pro
application directly via the macOS Accessibility API and synthetic
CGEvent key/mouse events — for actions ExtendScript/QE simply can't do
(there is currently no such gap these fill; they exist as raw primitives
for building new automation on, e.g. a future vision-language-driven
replacement for the old `desktop-set-input-lut` command).

**Setup (once per machine, macOS only):**

    pip install "premiere-cli[macos-desktop]"

Grant the **Accessibility** permission (System Settings > Privacy &
Security > Accessibility) to whichever process actually runs the
command — Terminal, iTerm, etc. — not to Premiere Pro itself. Without it
these commands fail with no clean error path; check this proactively if
one doesn't behave as expected.

**Safety — read before running:** every command below (except the notify
pair) posts REAL synthetic keyboard/mouse events, which macOS routes to
whichever app is frontmost. Each command confirms Premiere is frontmost
first and refuses to proceed if it can't (exit code 3) — but between that
confirmation and the actual input, the user's own actions can still steal
focus back, so tell the user to expect the command to run and to leave
the keyboard/mouse alone for the few seconds it takes. Every command shows
an on-screen overlay (amber while confirming frontmost, red while
driving, green when done) pinned top-right of the screen so this is
visible — call `desktop-notify` yourself first if you want a persistent,
more specific heads-up visible before/across several of these calls (see
below).

- `premiere-cli desktop-take-screenshot /path/to/frame.png` — confirms Premiere is frontmost, then saves a full-screen PNG via `screencapture -x`. Returns `{"ok": true, "path", "fileSizeBytes"}`. Exit codes: 0 ok, 1 Premiere not running, 3 couldn't confirm frontmost, 5 `screencapture` itself failed. Useful for visually verifying the effect of a prior `desktop-click-mouse`/`desktop-press-key` call — it re-confirms frontmost itself, so it correctly re-activates Premiere even if focus drifted away in between (live-observed: the terminal/IDE running these commands tends to regain focus between separate invocations).
- `premiere-cli desktop-press-key --key NAME [--modifier {command,cmd,shift,option,alt,control,ctrl}]` — one synthetic key press (down+up). `--key` is a name (letters, digits, `return`/`enter`, `escape`, `tab`, `space`, `delete`/`backspace`, `up`/`down`/`left`/`right`, `home`/`end`/`pageup`/`pagedown`, `f1`-`f12`, punctuation like `minus`/`equal`/`comma`/`period`, ...) or a decimal virtual keycode; `--modifier` is repeatable for chords (e.g. `--modifier cmd --modifier shift`). Returns `{"ok": true, "key", "keycode", "modifiers"}`. Exit codes: 0 ok, 1 Premiere not running, 2 unknown key/modifier name, 3 couldn't confirm frontmost. This posts a raw, unattributed key event — for literal unicode text (punctuation/characters that need an actual attached string, not a keycode) use `desktop-enter-text` instead.
- `premiere-cli desktop-enter-text "some text"` — types literal unicode text at the current insertion point/focus — no select-all, no clearing, no validation. Returns `{"ok": true, "text"}`. Exit codes: 0 ok, 1 Premiere not running, 3 couldn't confirm frontmost. Click into the target field first (`desktop-click-mouse`) — this doesn't set focus itself.
- `premiere-cli desktop-enter-text-with-validate "some text" [--attempts N]` — types text into whatever field currently has keyboard focus, then reads that field's value back to verify, retrying (select-all + retype) up to `--attempts` times (default 3) if it doesn't match. Returns `{"ok", "text", "actualValue", "error"?}`. Exit codes: 0 verified match, 1 Premiere not running, 2 no focused element to validate against, 3 couldn't confirm frontmost, 4 typed but the read-back never matched. **Load-bearing caveat, live-tested 2026-07-23**: Premiere's own custom-drawn panels do NOT reliably report focus via AX — clicking directly into the Project panel's search field and calling this immediately after still read back an unrelated, stale combobox as "focused," even though the text itself landed correctly (confirmed separately via `desktop-enter-text` + a screenshot). This command is only reliable against a genuinely native AX-compliant field addressed by reference elsewhere in code — for typing into Premiere's own UI, prefer plain `desktop-enter-text` and verify success visually with `desktop-take-screenshot` instead.
- `premiere-cli desktop-move-mouse --x N --y N` — moves the mouse cursor to screen coordinates (x, y) via a real CGEvent move — e.g. to hover something, or position the cursor ahead of a separate click call. Returns `{"ok": true, "x", "y"}`. Exit codes: 0 ok, 1 Premiere not running, 3 couldn't confirm frontmost. Coordinates are in AX/CGEvent point space (NOT raw screenshot pixels — on a Retina display a screenshot is 2x this space); get exact coordinates for a specific on-screen element via the Accessibility API rather than eyeballing a screenshot and doing the math by hand (live-tested 2026-07-23: hand-computed screenshot-pixel-to-point conversion landed one panel row off a target field).
- `premiere-cli desktop-click-mouse [--x N] [--y N] [--button {left,right}]` — clicks the mouse: at `(x, y)` if both given, else at the CURRENT cursor position (read live, not cached) — omit both to click wherever `desktop-move-mouse` last positioned the cursor, or after a real (non-synthetic) mouse move. `--button` defaults to `left`. Re-checks Premiere is still frontmost immediately before the down/up (not just at the start), since focus can slip in the gap. Returns `{"ok": true, "x", "y", "button"}`. Exit codes: 0 ok, 1 Premiere not running, 2 invalid button or only one of x/y given, 3 couldn't confirm frontmost (initially, or right before the click).
- `premiere-cli desktop-notify "message" [--state {waiting,driving,done,info}]` — shows (or updates, if one is already showing) an always-on-top notification window pinned top-right of the screen, independent of Premiere and independent of any other command's own process — it's backed by a separate, persistent daemon process that a LATER call updates in place rather than duplicating (live-verified: a second call updated the same window, confirmed via screenshot and exactly one daemon process via `pgrep`). `--state` sets the accent color: `waiting`=amber, `driving`=red, `done`=green, `info`=slate (default). Returns `{"ok": true, "message", "state"}`. Does NOT require or check Premiere's frontmost state. Use this to give the user a persistent, specific heads-up across a multi-step `desktop-*` sequence, on top of (not instead of) each command's own transient overlay.
- `premiere-cli desktop-dismiss-notifications` — closes the notification window from `desktop-notify` and terminates its daemon process, if one is running. Idempotent: calling it with nothing running returns `{"ok": true, "wasRunning": false}` rather than erroring (live-verified).

## Notes

- If the panel isn't open in Premiere Pro, `premiere-cli` still prints
  `{"ok": false, "error": "..."}` to stdout (exit 1) — this is a normal,
  parseable outcome, not a crash.
- Default port is 47823; only pass `--port` if the user has said the panel
  is running on a different one.
- When a new command is added to the panel, it gets documented here —
  check this file for the current command list rather than assuming.
