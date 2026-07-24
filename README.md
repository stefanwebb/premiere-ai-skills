# premiere-ai-skills

Claude Code plugin bundling the skills for driving Adobe Premiere Pro and
running AI-assisted video-production workflows, on top of:

- [premiere-cli](https://github.com/stefanwebb/premiere-cli) — the CLI
  (`premiere-cli`, `premiere-log`) and CEP bridge panel that expose Premiere
  Pro's scripting APIs
- [premiere-ai](https://github.com/stefanwebb/premiere-ai) — AI-assisted
  workflows (transcription, pause removal, sync-offset detection, project
  setup) built on top of `premiere-cli`

This repo contains **only** the Claude Code skills — install the two
packages above separately to get the underlying CLIs and panel.

## Install

```bash
# 1. The underlying tools
pipx install premiere-cli    # or: uv tool install premiere-cli
premiere-cli install-panel
pip install premiere-ai      # into whichever env you run automation from

# 2. The Claude Code plugin
/plugin marketplace add stefanwebb/premiere-ai-skills
/plugin install premiere-ai-skills@premiere-ai-skills
```

One skill needs extra setup beyond the above — see its own file for
details: `calibrate-lut` needs the `mlx-community/sam3-4bit` model
downloaded via `hf download`.

## Skills

| Skill | Wraps | Purpose |
|-------|-------|---------|
| `premiere-cli` | `premiere-cli` | Query/drive the open Premiere Pro project via the bridge panel |
| `premiere-log` | `premiere-log` | Surface automation progress inside the bridge panel |
| `create-empty-premiere-project` | `premiere-cli` | Scaffold a fresh Premiere Pro project from the bundled template |
| `remove-pauses-from-track` | `premiere-ai` + `premiere-cli` | Ripple-delete silences from one track, keeping a linked video track in sync |
| `calculate-sync-offset` | `premiere-ai` | Compute the audio/video sync offset for a camera + external-mic take |
| `calibrate-lut` | `premiere-ai` | Build a `.cube` correction LUT from a photo of a ColorChecker chart (video or classic page) |
| `correct-color` | `premiere-cli` + `premiere-ai` | End-to-end: extract a ColorChecker frame from a sequence, build a LUT (applying it is currently a manual step — see the skill's own note) |

> `desktop-set-input-lut` (which drove Premiere's native UI to set a clip's
> Lumetri Input LUT) was removed 2026-07-23 pending a rebuild on top of
> `premiere-cli`'s generic `desktop-*` UI-automation primitives
> (`desktop-take-screenshot`, `desktop-press-key`, `desktop-enter-text`,
> `desktop-move-mouse`, `desktop-click-mouse`, etc.) plus AI vision-language
> screen understanding, rather than hand-written Accessibility-tree
> navigation specific to one control.

## Scope

This plugin only carries Claude-facing skill definitions. Changes to the
underlying CLI behavior belong in
[premiere-cli](https://github.com/stefanwebb/premiere-cli) or
[premiere-ai](https://github.com/stefanwebb/premiere-ai), not here.

## License

[CC-BY-SA-4.0](LICENSE).
