# Local CV generator skill

`setup-cv-generator` is a skill for a coding agent. Initial setup requires an
agent; afterward, the CV content can be updated manually and its explicit page
plan can be maintained through a short repository-local guide.

This skill equips an existing website repository with a local,
editable CV workflow. It produces a paginated CV PDF for the chosen paper
size while preserving the site's visual language and leaving its normal build
and navigation alone.

It is intended for a CV that is maintained alongside a site but is not
automatically published. A person can update career content, contact details,
and a portrait without requiring an active AI agent subscription.

> [!CAUTION]
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
├── config.yml                  # paper dimensions and content margins
├── data/                       # one YAML or JSON file per locale
├── page-plan/                  # accepted block-to-page plan per locale
├── assets/                     # portrait, vendored fonts, and local assets
├── private.example.yml          # copyable template for contact details
├── private.yml                  # real private details; ignored by Git
├── output/                      # generated local output; ignored by Git
├── README.md                    # project-specific editing and build guide
└── PAGINATION.md                # short guide for later maintenance agents
```

Some frameworks require generator code or locale data in conventional
locations outside `curriculum/`. In that case, the project-specific
`curriculum/README.md` points to them; `curriculum/` remains the place a person
edits day to day.

It also adds one dedicated command appropriate to the repository, such as
`bin/cv`, `npm run cv`, or `make cv`. That command validates the CV data,
measures stable semantic blocks, proposes candidate page boundaries, checks the
accepted page plan, explicitly persists a selected candidate, generates and
verifies the PDF, and provides a locale-aware `preview` operation that opens the
verified PDF in the platform's native viewer.

## How it works

1. It checks the existing website's framework, build commands, design tokens,
   fonts, conventions, and working-tree state.
2. It asks whether A4 or another paper size and orientation is the right target,
   unless those requirements were already provided.
3. It creates a small CV-specific adapter: a route, template, stylesheet,
   validator, and command only where the host stack needs them.
4. It reuses the site's typography, colour palette, spacing, and component
   conventions, while vendoring permitted font files so measurements never
   depend on a remote font service.
5. It separates private contact data and the portrait from the shared CV
   content. The real private-data file and generated output are ignored by
   Git; an example file explains the required shape.
6. It chooses either a macOS-native WebKit/PDFKit adapter or a portable
   Playwright/Poppler adapter from the repository's real operating
   environment, asking before downloads or system-package installs.
7. Every pageable content block receives a stable ID. Once real content first
   renders, a deterministic paginator measures each block and emits prose plus
   JSON with block heights, ranked candidate boundaries, and any violations.
8. An agent chooses a semantically good candidate and persists the explicit
   block-to-page plan. The generator then builds exact paper-sized page
   containers and fails if a planned page overflows, naming the offending block
   and overflow in millimetres.
9. Its `preview` operation validates and builds self-contained HTML, reports
   advisory balance findings, generates and independently checks the actual
   PDF, renders every page, creates an all-pages contact sheet, and opens that
   PDF only after verification succeeds.

The skill checks `origin/main` only when the user specifically asks to update
the skill. If a newer revision exists, it asks before changing the installed
copy. Ordinary CV work does not contact the remote or run an update check.

## Print and accessibility contract

The generated document is semantic and selectable HTML with a skip link,
heading hierarchy, labelled controls, semantic timeline lists, and useful
portrait alt text. It is a generated, self-contained artifact with embedded
CSS, images, and locally vendored fonts; editable YAML or JSON remains the
human-maintained source. It uses the configured PDF paper size with zero print
margins and explicit page containers. One canonical value controls vertical
capacity: paper height minus the configured top and bottom content margins.

On screen, the HTML offers language switching, status, and a **Print / Save as
PDF** escape hatch. Routine use does not require manual browser printing:
`preview` generates, verifies, and opens the PDF directly. `check` writes both a
human summary and JSON containing measured block heights, accepted-page usage,
ranked candidate boundaries, warnings, and hard failures. Typography and global
spacing are design choices, never automatic fitting controls.

The selected adapter generates a reproducible print-CSS PDF. PDFKit on the
native path or Poppler on the portable path then checks its page dimensions,
page count against the accepted plan, and extracted text, renders every page,
and composes the PNGs into one contact sheet. Verification fails for planned
overflow, a size or page-count mismatch, unintentionally blank pages, missing
expected text, clipped content, or missing rendered pages. Native
Safari/browser preview remains a useful separate renderer check.

## Typical project workflow

After setup, the delivered `curriculum/README.md` contains the exact commands
for that repository. The workflow normally looks like this:

```zsh
cp curriculum/private.example.yml curriculum/private.yml
# Edit curriculum/data/, curriculum/private.yml, and curriculum/assets/.
<cv-command> check <locale>
# Review the candidate IDs, then persist one explicitly:
<cv-command> plan <locale> --candidate <candidate-id>
<cv-command> preview <locale>
<host-build-command>
git diff --check
```

Use `build`, `check`, `plan`, and `verify` separately for focused maintenance,
debugging, or automation. With one configured locale, `preview` may omit the
locale. With multiple locales, it requires one and lists the valid choices
when omitted.

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
