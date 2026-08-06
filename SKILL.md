---
name: setup-cv-generator
description: Set up or restore a local, editable, paginated CV generator in any website repository. Use when a printable CV or PDF must target the user's chosen paper size, follow an existing site's design language, keep source content simple to update, and remain separate from the default public build.
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
2. Ask whether A4 is the correct paper target or whether the user needs another
   size or orientation, unless they already specified it. Treat the answer as
   configuration shared by print CSS, PDF generation, verification, and the
   update guide. Let content flow onto as many pages as it needs.
3. Establish the `curriculum/` file contract from the reference. Use simple
   structured content files, a private-data template, local assets, and an
   update guide. Keep generator-specific code in the smallest appropriate
   adapter for the host stack.
4. Reuse the site's font loading, colour tokens, visual rhythm, and component
   conventions. Add only the semantic CV markup, configured paper layout, and
   optional progressive enhancement the document needs.
5. Keep contact details and portrait local. Ignore the real private-data file,
   provide an example file, validate it before build, and confirm it is not
   staged.
6. Choose the smallest PDF adapter justified by the repository's operating
   environment. For a deliberately macOS-local workflow, prefer the native
   Swift/WebKit/PDFKit adapter in the reference when Apple developer tools are
   already present. For cross-platform or CI workflows, use the portable
   Playwright/Poppler adapter. Obtain approval before installing system
   packages or downloading browser runtimes.
7. Add one dedicated CV command. It validates data before generation and
   exposes `build`, locale-aware `preview`, and `verify` operations without
   changing the repository's default build or navigation. `verify` must
   generate and test the actual PDF; it is not an alias for browser geometry
   checks.
8. Preserve the print contract: a configured `@page` size, zero print margins,
   colour preservation, controls hidden in print, content-driven pagination,
   and a small safety margin inside each page. An advisory layout check may
   report clipping or awkward breaks, while Print / Save as PDF always invokes
   native printing.
9. Run `verify` for the combined document or every configured locale, the host
   project's relevant build, and `git diff --check`. Inspect the rendered page
   images. Report generated-PDF checks separately from native Safari/browser
   print inspection.

## Completion

The system is complete when the dedicated CV command validates private and
content data, produces and verifies a complete PDF for every configured locale
at the chosen paper size, and leaves the host site's default build unchanged.
Verification fails on a page-size or content-derived page-count mismatch, an
unintentionally blank page, missing expected text, or clipped print content.
A person can replace career content, contact details, and portrait by editing
documented files under `curriculum/`, without editing templates or application
code.
