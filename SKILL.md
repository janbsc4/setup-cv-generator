---
name: setup-cv-generator
description: Set up or restore a local, editable A4 CV generator in any website repository. Use when a printable CV or PDF must follow an existing site's design language, keep source content simple to update, and remain separate from the default public build.
---

# Set up CV generator

Create a **local-first** CV. Put its editable source and supporting assets in
`curriculum/`; reuse the host website's visual language; and keep it out of
the default public build unless the user explicitly asks to publish it.

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
5. Add one dedicated CV command. Validate data first, then produce the CV
   pages or standalone HTML without changing the repository's default build or
   navigation.
6. Preserve the print contract: an exact 210 mm × 297 mm A4 sheet, zero print
   margins, colour preservation, and controls hidden in print. An advisory
   compact-density check may report overflow, while Print / Save as PDF always
   invokes native printing.
7. Validate the data and generated document for every configured locale, run
   the host project's relevant build and `git diff --check`, and inspect the
   output HTML. Report deterministic checks separately from native browser/PDF
   print inspection.

## Completion

The system is complete when the dedicated CV command validates private and
content data, produces one page per configured locale, and leaves the host
site's default build unchanged. A person can replace career content, contact
details, and portrait by editing documented files under `curriculum/`, without
editing templates or application code.
