# Local CV generator skill

`setup-cv-generator` equips an existing website repository with a local,
editable CV workflow. It produces a print-ready A4 CV or PDF while preserving
the site's visual language and leaving its normal build and navigation alone.

It is intended for a CV that is maintained alongside a site but is not
automatically published. The person maintaining it should be able to update
career content, contact details, and a portrait without changing templates or
application code.

## What it creates

The skill first inspects the host repository, then chooses the smallest adapter
that fits its framework. It establishes this human-editable boundary:

```text
curriculum/
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
`bin/cv`, `npm run cv`, or `make cv`. That command validates the CV data before
generating the document and, where supported, provides a locale-aware preview.

## How it works

1. It checks the existing website's framework, build commands, design tokens,
   fonts, conventions, and working-tree state.
2. It creates a small CV-specific adapter: a route, template, stylesheet,
   validator, and command only where the host stack needs them.
3. It reuses the site's typography, colour palette, spacing, and component
   conventions, adding semantic CV markup and A4 print styling.
4. It separates private contact data and the portrait from the shared CV
   content. The real private-data file and generated output are ignored by
   Git; an example file explains the required shape.
5. It validates every locale, generates the local CV pages, runs the host
   project's relevant build, checks the diff, and reports what was verified.

Before doing CV work, the skill compares its installed revision with
`origin/main`. If a newer revision exists, it asks before updating; it never
updates itself automatically. An unavailable remote or network only makes that
check advisory, so CV work can continue.

## Print and accessibility contract

The generated document is semantic and selectable HTML with a skip link,
heading hierarchy, labelled controls, semantic timeline lists, and useful
portrait alt text. It is designed for a 210 mm × 297 mm A4 sheet with zero
print margins, colour preservation, and screen-only controls hidden on paper.

On screen, it offers language switching, a fit status, and a **Print / Save as
PDF** button. A compact-density fit check can warn about potential overflow,
but printing always continues through the browser's native print dialog. The
final check for one-page fit and clipping is therefore native print preview;
automated validation cannot replace it.

## Typical project workflow

After setup, the delivered `curriculum/README.md` contains the exact commands
for that repository. The workflow normally looks like this:

```zsh
cp curriculum/private.example.yml curriculum/private.yml
# Edit curriculum/data/, curriculum/private.yml, and curriculum/assets/.
<cv-command> build
<cv-command> preview <locale>
<host-build-command>
git diff --check
```

For native printing, select A4 paper, 100% scale, no margins, and background
graphics in the system print dialog.

## When to use it

Use this skill when a website needs a local, editable CV or PDF that follows
the site's design language, keeps sensitive data out of version control, and
does not affect the public site unless publishing is explicitly requested.

The complete portable file and content contract is in
[`references/system.md`](references/system.md). The agent-facing execution
workflow is in [`SKILL.md`](SKILL.md).
