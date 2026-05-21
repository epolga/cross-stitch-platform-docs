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

- **`plan/`** — planning, roadmap, milestone, and operational-strategy documents **only**. Anything that describes _what we intend to do_ or _when_.
- **`docs/`** — everything else: templates, helper notes, reference material, automation scripts, build-output documentation. Anything that describes _how things work_ or _what exists_.

Both directories use a parallel subfolder structure organized by topic/component (`cross-stitch/`, `uploader/`, `automation/`, plus topical folders under `plan/` like `aws/`, `etsy/`, `paypal/`, `integration-*/`, etc.). When adding a new document, choose the subfolder by topic and the top-level directory by plan-vs-operational. Do not put non-planning files under `plan/` — the previous restructure existed specifically to clean that up.

## Cross-project context

Documents under `plan/cross-stitch/` and `docs/cross-stitch/` describe the cross-stitch web app that lives at `../cross-stitch/`. Documents under `plan/uploader/` and `docs/uploader/` describe the WPF app at `../Uploader/`. Files like the Pinterest AI Agent series under `plan/cross-stitch/` span both projects (WPF integration is documented there).

When a planning doc references a file path, treat unprefixed paths as relative to the relevant sibling project, not this repo.

## AI workflow and documentation usage

This repository is the orchestration and knowledge hub for the entire Cross-Stitch platform ecosystem.

Before proposing plans, modifying files, or making architectural assumptions:

1. Read relevant documents under `docs/`
2. Read relevant planning documents under `plan/`
3. Summarize discovered assumptions and constraints
4. Only then propose changes or implementation steps

Do not invent or assume:

- DynamoDB schemas
- AlbumID/DesignID formats
- S3 path structures
- PDF generation conventions
- uploader ↔ website contracts
- Pinterest metadata conventions
- AWS deployment assumptions

The documentation directories are considered the source of truth unless explicitly overridden by newer instructions.

## Workspace awareness

The VS Code workspace includes:

- `cross-stitch-platform-docs/`
- `../cross-stitch/`
- `../Uploader/`

The documentation repository coordinates work across both sibling repositories.

When analyzing integrations or workflows:

- consider both repositories together
- identify cross-repo dependencies
- identify shared contracts
- identify synchronization risks

## Preferred behavior

Prefer:

- planning before coding
- explicit architecture summaries
- identifying missing documentation
- proposing incremental safe changes

Avoid:

- large speculative refactors
- inventing undocumented contracts
- changing multiple repositories without documenting assumptions first
