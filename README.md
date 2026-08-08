# Local CV generator skill

`setup-cv-generator` equips an existing website repository with a local,
editable CV workflow. It produces a paginated CV or PDF for the chosen paper
size while preserving the site's visual language and leaving its normal build
and navigation alone.

It is intended for a CV that is maintained alongside a site but is not
automatically published. The person maintaining it should be able to update
career content, contact details, and a portrait without changing templates or
application code.

> **Important — install the skill outside the website repository.** Keep this
> Git checkout in your agent's skills directory. On macOS, use
> `/Users/<username>/.agents/skills/setup-cv-generator` by default; on other
> systems, use `$HOME/.agents/skills/setup-cv-generator`. Run it against the
> website repository. Cloning it inside that repository creates an embedded
> Git repository, which can be staged as a Gitlink instead of normal source
> files.

## What it creates

The skill first inspects the host repository, then chooses the smallest adapter
that fits its framework. It establishes this human-editable boundary:

```text
curriculum/
├── config.yml                  # chosen paper size and orientation
├── data/                       # one YAML or JSON file per locale
├── assets/                     # portrait and other local assets
├── private.example.yml          # copyable template for contact details
├── private.yml                  # real private details; ignored by Git
├── output/                      # generated local output; ignored by Git
└── README.md                    # project-specific editing and build guide
```

Some frameworks require generator code or locale data in conventional
locations outside `curriculum/`. In that case, the project-specific
`curriculum/README.md` points to them; `curriculum/` remains the place a person
edits day to day.

It also adds one dedicated command appropriate to the repository, such as
`bin/cv`, `npm run cv`, or `make cv`. That command validates the CV data,
checks projected pagination and major-section splits early, generates and
verifies the PDF, and provides a locale-aware `preview` operation that opens
the verified PDF in the platform's native viewer.

## How it works

1. It checks the existing website's framework, build commands, design tokens,
   fonts, conventions, and working-tree state.
2. It asks whether A4 or another paper size and orientation is the right target,
   unless those requirements were already provided.
3. It creates a small CV-specific adapter: a route, template, stylesheet,
   validator, and command only where the host stack needs them.
4. It reuses the site's typography, colour palette, spacing, and component
   conventions, adding semantic CV markup and print styling for the configured
   paper target.
5. It separates private contact data and the portrait from the shared CV
   content. The real private-data file and generated output are ignored by
   Git; an example file explains the required shape.
6. It chooses either a macOS-native WebKit/PDFKit adapter or a portable
   Playwright/Poppler adapter from the repository's real operating
   environment, asking before downloads or system-package installs.
7. Once real content first renders, it runs a fast print-layout check for every
   locale. The check reports projected page count and names any major section
   that crosses a page boundary, before visual polishing makes rework costly.
8. Its `preview` operation validates and builds the HTML, reports advisory
   layout findings, generates and independently checks the actual PDF, and
   opens that PDF only after verification succeeds.

The skill checks `origin/main` only when the user specifically asks to update
the skill. If a newer revision exists, it asks before changing the installed
copy. Ordinary CV work does not contact the remote or run an update check.

## Print and accessibility contract

The generated document is semantic and selectable HTML with a skip link,
heading hierarchy, labelled controls, semantic timeline lists, and useful
portrait alt text. It uses the configured PDF paper size with zero print
margins, colour preservation, screen-only controls hidden on paper, and a small
internal safety margin to avoid fractional blank-page overflow in native
renderers. The paper size and orientation come from the setup choice. Content
flows onto additional pages when needed; the generator does not compress every
CV into one page.

On screen, the HTML offers language switching, a fit status, and a **Print /
Save as PDF** escape hatch. Routine use does not require manual browser
printing: `preview` generates, verifies, and opens the PDF directly. The same
measurement powers an early command-line check that reports total projected
pages and major sections split across pages. The status distinguishes advisory
browser geometry from a PDF actually verified by the command.

The selected adapter generates a reproducible print-CSS PDF. PDFKit on the
native path or Poppler on the portable path then checks its page dimensions,
content-derived page count, and extracted text and renders every page for
inspection. Verification fails for a size or page-count mismatch,
unintentionally blank pages, missing expected text, or clipped content. Native
Safari/browser preview remains a useful separate renderer check.

## Typical project workflow

After setup, the delivered `curriculum/README.md` contains the exact commands
for that repository. The workflow normally looks like this:

```zsh
cp curriculum/private.example.yml curriculum/private.yml
# Edit curriculum/data/, curriculum/private.yml, and curriculum/assets/.
<cv-command> preview <locale>
<host-build-command>
git diff --check
```

Use `build`, `check`, and `verify` separately for focused debugging or
automation. With one configured locale, `preview` may omit the locale. With
multiple locales, it requires one and lists the valid choices when omitted.

For optional browser-print inspection, select the configured paper size and
orientation, 100% scale, no margins, and background graphics in the system
print dialog.

## When to use it

Use this skill when a website needs a local, editable CV or PDF that follows
the site's design language, keeps sensitive data out of version control, and
does not affect the public site unless publishing is explicitly requested.

The complete portable file and content contract is in
[`references/system.md`](references/system.md). The agent-facing execution
workflow is in [`SKILL.md`](SKILL.md).
