# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository nature

This is a **documentation-only** repository — no source code, no build system, no tests. It is the central docs and planning hub for two sibling projects that live next to it on disk (not in this repo):

- `../cross-stitch/` — the cross-stitch web application
- `../Uploader/` — the WPF uploader application

The VS Code workspace at `cross-stitch.code-workspace` opens all three folders together so the docs and both codebases can be edited side-by-side.

There are no build, lint, test, or run commands. Work in this repo is editing Markdown, text, spreadsheets, and PDFs.

## The plan/ vs docs/ split (load-bearing)

The most recent restructuring commit enforces a strict separation between two top-level directories. New files must go in the correct one:

- **`plan/`** — planning, roadmap, milestone, and operational-strategy documents **only**. Anything that describes *what we intend to do* or *when*.
- **`docs/`** — everything else: templates, helper notes, reference material, automation scripts, build-output documentation. Anything that describes *how things work* or *what exists*.

Both directories use a parallel subfolder structure organized by topic/component (`cross-stitch/`, `uploader/`, `automation/`, plus topical folders under `plan/` like `aws/`, `etsy/`, `paypal/`, `integration-*/`, etc.). When adding a new document, choose the subfolder by topic and the top-level directory by plan-vs-operational. Do not put non-planning files under `plan/` — the previous restructure existed specifically to clean that up.

## Cross-project context

Documents under `plan/cross-stitch/` and `docs/cross-stitch/` describe the cross-stitch web app that lives at `../cross-stitch/`. Documents under `plan/uploader/` and `docs/uploader/` describe the WPF app at `../Uploader/`. Files like the Pinterest AI Agent series under `plan/cross-stitch/` span both projects (WPF integration is documented there).

When a planning doc references a file path, treat unprefixed paths as relative to the relevant sibling project, not this repo.
