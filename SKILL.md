---
name: setup-cv-generator
description: Set up or restore a local, editable, deterministic CV generator in any website repository. Use when a printable CV or PDF needs measured semantic blocks, an explicit page plan, self-contained output, a chosen paper size, the existing site's design language, and separation from the default public build.
---

# Set up CV generator

Create a **local-first** CV. Put its editable source and supporting assets in
`curriculum/`; reuse the host website's visual language; and keep it out of
the default public build unless the user explicitly asks to publish it.

## Explicit skill update

Run this branch only when the user specifically asks to check for or install an
update to this skill. Ordinary CV setup, restoration, editing, and verification
skip it.

Identify the directory containing this `SKILL.md`. If its path is unavailable,
use `/Users/<username>/.agents/skills/setup-cv-generator` as the default on
macOS and `$HOME/.agents/skills/setup-cv-generator` elsewhere. Then compare its
installed Git revision with GitHub's `origin/main`:

1. Run `git -C <skill-directory> rev-parse HEAD`.
2. Run `git -C <skill-directory> ls-remote origin refs/heads/main`.
3. If the revisions differ, tell the user that a potentially newer upstream
   version is available at `https://github.com/janbsc4/setup-cv-generator` and
   ask whether they want to update it. Pull or replace the installed skill only
   after that explicit approval.

Treat a missing Git remote or an unavailable network as advisory: state that
the update check could not be completed, then continue with the requested CV
work.

Read [system.md](references/system.md) before creating, restoring, or changing
the system. It defines the portable file contract and the framework adapters.

## Workflow

1. Inspect the repository's framework, build and preview commands, route or
   page conventions, existing design tokens, fonts, `git status`, and every
   plausible legacy CV source. Identify the authoritative source before
   migrating content, and record which configured locales have equivalent
   source material. Preserve summary pages as summaries; do not expand them or
   invent translations from a richer source in another locale.
2. Ask whether A4 is the correct paper target or whether the user needs another
   size or orientation, unless they already specified it. Treat the answer as
   configuration shared by print CSS, PDF generation, verification, and the
   update guide. Let content flow onto as many pages as it needs.
3. Establish the `curriculum/` file contract from the reference. Use simple
   structured content files, a private-data template, local assets, one
   explicit page plan per locale, a general update guide, and a short
   repository-local pagination guide. Keep generator-specific code in the
   smallest appropriate adapter for the host stack.
4. Reuse the site's colour tokens, visual rhythm, and component conventions.
   Vendor permitted font files locally and embed them in generated output; do
   not depend on remote font loading for measurement or PDF generation. Add
   only the semantic CV markup, configured paper layout, and optional
   progressive enhancement the document needs.
5. Keep contact details and portrait local. Ignore the real private-data file,
   provide an example file, validate it before build, and confirm it is not
   staged. Require email. Treat phone as optional: its display and `tel:` href
   are either both blank or both present, and omit telephone markup when blank.
6. Choose the smallest PDF adapter justified by the repository's operating
   environment. For a deliberately macOS-local workflow, prefer the native
   Swift/WebKit/PDFKit adapter in the reference when Apple developer tools are
   already present. For cross-platform or CI workflows, use the portable
   Playwright/Poppler adapter. Obtain approval before installing system
   packages or downloading browser runtimes. Apply the native adapter's cache,
   bridge, AppKit lifetime, and runaway-output safeguards as one contract.
7. Add one dedicated CV command. It validates data before generation and
   exposes `build`, `check`, `plan`, `verify`, and locale-aware `preview`
   operations without changing the repository's default build or navigation.
   `check` measures every stable pageable block, solves candidate boundaries,
   and emits both concise prose and machine-readable JSON. `plan` persists one
   explicitly selected candidate. `verify` generates and tests the actual PDF;
   `preview` runs the same pipeline and opens only a verified PDF. Warnings may
   remain advisory, but invalid data, a stale or incomplete plan, planned-page
   overflow, generation failure, and PDF verification failure are hard stops.
8. As soon as real content and baseline print CSS render, run the deterministic
   paginator for every locale, before visual polishing. Review its measured
   candidate boundaries, choose the best semantic result, and persist an
   explicit page plan. Every pageable block must have a stable ID shared by
   data, measurement output, page plan, and final markup. Re-run after material
   content or layout changes; update the plan from measured candidates rather
   than searching for CSS break rules by trial and error.
9. Preserve one print geometry contract. Define usable page height exactly once
   as paper height minus top and bottom content margins, and derive measurement,
   explicit page containers, overflow checks, PDF generation, verification,
   and documentation from that configuration. Generate self-contained HTML
   only after the plan is accepted: place planned blocks into exact paper-sized
   page containers, embed CSS, images, and fonts, and add a print break between
   containers. Fail with the block ID and overflow in millimetres when any
   planned page exceeds usable height. Keep Print / Save as PDF available.
10. Run `verify` for every configured locale, the host project's relevant
   build, and `git diff --check`. Render every PDF page to PNG and generate a
   contact sheet containing all pages for one-image pagination inspection.
   Inspect it, then report generated-PDF checks separately from native
   Safari/browser print inspection. Treat font-size and global spacing changes
   as deliberate design work, never as pagination-fitting mechanisms.

## Completion

The system is complete when the dedicated CV command validates private and
content data, assigns stable IDs to every pageable block, emits reproducible
measurements and candidate boundaries, persists an explicit page plan, and
generates self-contained paged HTML from that plan. It produces and verifies a
complete PDF and contact sheet for every configured locale, opens only a
successfully verified PDF from `preview`, and leaves the host site's default
build unchanged. Verification fails on planned-page overflow, a page-size or
planned page-count mismatch, an unintentionally blank page, missing expected
text, clipped print content, or a missing rendered page.
A person can replace career content, contact details, and portrait by editing
documented files under `curriculum/`, without editing templates or application
code.
