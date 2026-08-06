---
name: setup-cv-generator
description: Set up or restore a local, editable A4 CV generator in any website repository. Use when a printable CV or PDF must follow an existing site's design language, keep source content simple to update, and remain separate from the default public build.
---

# Set up CV generator

Create a **local-first** CV. Put its editable source and supporting assets in
`curriculum/`; reuse the host website's visual language; and keep it out of
the default public build unless the user explicitly asks to publish it.

## Update check

Before any other work, identify the directory containing this `SKILL.md` and
compare its installed Git revision with GitHub's `origin/main`:

1. Run `git -C <skill-directory> rev-parse HEAD`.
2. Run `git -C <skill-directory> ls-remote origin refs/heads/main`.
3. If the revisions differ, tell the user that a potentially newer upstream
   version is available at `https://github.com/janbsc4/setup-cv-generator` and
   ask whether they want to update it. Do not update the skill automatically.

Treat a missing Git remote or an unavailable network as advisory: state that
the update check could not be completed, then continue with the requested CV
work.

Read [system.md](references/system.md) before creating, restoring, or changing
the system. It defines the portable file contract and the framework adapters.

## Workflow

1. Inspect the repository's framework, build and preview commands, route or
   page conventions, existing design tokens, fonts, and `git status`.
2. Establish the `curriculum/` file contract from the reference. Use simple
   structured content files, a private-data template, local assets, and an
   update guide. Keep generator-specific code in the smallest appropriate
   adapter for the host stack.
3. Reuse the site's font loading, colour tokens, visual rhythm, and component
   conventions. Add only the semantic CV markup, A4 layout, and optional
   progressive enhancement the document needs.
4. Keep contact details and portrait local. Ignore the real private-data file,
   provide an example file, validate it before build, and confirm it is not
   staged.
5. Install the PDF verification dependencies from the reference. Keep
   Playwright and its Chromium browser in the project's development toolchain;
   check for Poppler's command-line tools and install the platform package with
   the user's approval when it is absent.
6. Add one dedicated CV command. It validates data before generation and
   exposes `build`, locale-aware `preview`, and `verify` operations without
   changing the repository's default build or navigation. `verify` must
   generate and test the actual PDF; it is not an alias for browser geometry
   checks.
7. Preserve the print contract: an A4 `@page`, zero print margins, colour
   preservation, controls hidden in print, and a small pagination safety
   margin inside each sheet. An advisory compact-density check may report
   overflow, while Print / Save as PDF always invokes native printing.
8. Run `verify` for the combined document or every configured locale, the host
   project's relevant build, and `git diff --check`. Inspect the rendered page
   images. Report generated-PDF checks separately from native Safari/browser
   print inspection.

## Completion

The system is complete when the dedicated CV command validates private and
content data, produces and verifies one PDF sheet per configured locale, and
leaves the host site's default build unchanged. Verification fails on a wrong
page count, a nearly blank page, missing expected text, or print-layout
overflow. A person can replace career content, contact details, and portrait
by editing documented files under `curriculum/`, without editing templates or
application code.
