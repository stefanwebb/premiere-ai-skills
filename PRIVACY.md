# Privacy Policy

_Last updated: 2026-07-19_

This repository contains **Claude Code skill definitions only** — Markdown
files that tell Claude how to invoke the `premiere-cli` and `premiere-ai`
command-line tools. It contains no application code, no servers, and no
telemetry. It does not itself collect, store, or transmit any data.

## What this plugin does

Installing this plugin gives Claude a set of instructions for driving
Adobe Premiere Pro and related video-production workflows on your own
machine. When you invoke a skill, Claude runs local commands
(`premiere-cli`, `premiere-log`, `sync-audio`, `remove-pauses`, etc.)
against:

- your local Premiere Pro project files, and
- the Premiere Bridge CEP panel, a local HTTP server bound to
  `127.0.0.1` (port 47823 by default) that only accepts connections from
  your own machine.

None of this traffic leaves your computer as a result of anything in
this repository.

## Third-party processing

This plugin wraps two separate open-source packages, which you install
and run yourself:

- **[premiere-cli](https://github.com/stefanwebb/premiere-cli)** — drives
  Premiere Pro entirely locally; makes no network calls.
- **[premiere-ai](https://github.com/stefanwebb/premiere-ai)** — runs
  transcription and forced alignment locally, but its `remove-pauses`
  command sends the transcript text (word-level timestamps, not audio)
  to the Anthropic API to detect natural sentence/phrase boundaries. This
  only happens if you run that command and have set `ANTHROPIC_API_KEY`.
  See [Anthropic's privacy policy](https://www.anthropic.com/legal/privacy)
  for how that data is handled.

Neither package is maintained in this repository; see their own
repositories for further detail on their behavior.

## Data storage

This plugin stores nothing. Any files created by the underlying tools
(transcripts, cut lists, exported audio, Premiere project files) are
written to locations on your own filesystem that you control, and are
never uploaded anywhere by this plugin.

## Your control

Since everything runs locally under your own commands, you control what
runs and when. Uninstalling the plugin (`/plugin uninstall
premiere-ai-skills`) removes the skill definitions; no residual data or
service registration is left behind.

## Changes to this policy

If this plugin's scope changes in a way that affects data handling, this
file will be updated accordingly.

## Contact

Questions about this policy: [info@stefanwebb.me](mailto:info@stefanwebb.me).
