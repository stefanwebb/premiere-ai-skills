---
name: create-empty-premiere-project
description: Use when asked to create a new/fresh/empty Premiere Pro project, e.g. "/create-empty-premiere-project project0002" or "/create-empty-premiere-project myseries 2".
---

# /create-empty-premiere-project

Creates a fresh, empty Premiere Pro project from `premiere-cli`'s bundled
empty-project template, via `premiere-cli init-project`.

## Parsing the invocation

The invocation gives a project name and, optionally, a series name. Parse
whatever form the user supplies into `project_name` and (if present)
`series_name`:

- A single name → `project_name` only, no series, e.g. `project0002`.
- Two names (space-separated, or written as `<series>/<project>`) → the
  first is `series_name`, the second is `project_name`, e.g. `myseries 0002`
  or `myseries/episode 2` → series `myseries`, project `episode 2` (or `0002`).

If the project name is genuinely ambiguous or missing entirely, ask the user
rather than guessing.

## Running the command

Run in any environment with `premiere-cli` installed (Conda, uv, venv, or
otherwise — only the package needs to be available; no Premiere Pro
session or panel connection is required for this command):

```
premiere-cli init-project "<project_name>" --base-dir "<base_dir>"
premiere-cli init-project "<project_name>" --series "<series_name>" --base-dir "<base_dir>"
```

`--base-dir` is required. If the calling project has a `CLAUDE.md`
documenting a default base directory for new Premiere Pro projects, use
that. Otherwise ask the user where to create the project rather than
guessing.

## What it does

1. Copies `premiere-cli`'s bundled empty-project template (an empty
   `Untitled.prproj` plus an `Adobe Premiere Pro Auto-Save` folder) to
   `<base-dir>/[<series_name>/]<project_name>`.
2. Renames the copied `.prproj` file to `<project_name>.prproj`.

It refuses to run (and makes no changes) if the destination directory
already exists — report this to the user rather than deleting or
overwriting anything there.

## Notes

- Requires the `premiere-cli` package to be installed in whatever
  environment the command runs in; remind the user if it isn't available.
- The template ships inside the `premiere-cli` package itself, so this
  works without any shared/external drive mounted — only the destination
  (`--base-dir`) needs to be reachable.
- This only creates the project shell (folder + empty `.prproj`); it does
  not import footage, create bins, or otherwise touch the project's
  contents.
