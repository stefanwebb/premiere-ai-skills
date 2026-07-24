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

Two skills need extra setup beyond the above — see each skill's own file
for details: `desktop-set-input-lut` needs `premiere-cli`'s optional
`macos-desktop` extra plus a macOS Accessibility permission grant;
`calibrate-lut` needs the `mlx-community/sam3-4bit` model downloaded via
`hf download`.

## Skills

| Skill | Wraps | Purpose |
|-------|-------|---------|
| `premiere-cli` | `premiere-cli` | Query/drive the open Premiere Pro project via the bridge panel |
| `premiere-log` | `premiere-log` | Surface automation progress inside the bridge panel |
| `create-empty-premiere-project` | `premiere-cli` | Scaffold a fresh Premiere Pro project from the bundled template |
| `remove-pauses-from-track` | `premiere-ai` + `premiere-cli` | Ripple-delete silences from one track, keeping a linked video track in sync |
| `calculate-sync-offset` | `premiere-ai` | Compute the audio/video sync offset for a camera + external-mic take |
| `desktop-set-input-lut` | `premiere-cli` (macOS only) | Load a new `.cube` file onto a clip's Lumetri Input LUT by driving Premiere's native UI |
| `calibrate-lut` | `premiere-ai` | Build a `.cube` correction LUT from a photo of a ColorChecker chart (video or classic page) |

## Scope

This plugin only carries Claude-facing skill definitions. Changes to the
underlying CLI behavior belong in
[premiere-cli](https://github.com/stefanwebb/premiere-cli) or
[premiere-ai](https://github.com/stefanwebb/premiere-ai), not here.

## License

[CC-BY-SA-4.0](LICENSE).
