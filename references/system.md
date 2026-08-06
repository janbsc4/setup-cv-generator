# Portable CV generator reference

## File contract

Use `curriculum/` as the complete human-editable boundary. Keep files that the
host framework needs to discover elsewhere only when its conventions require
it; point to them from `curriculum/README.md`.

| Responsibility | Files |
| --- | --- |
| Editable CV content | `curriculum/data/<locale>.yml` or `.json` |
| Local contact details | `curriculum/private.yml` (ignored), `curriculum/private.example.yml` |
| Portrait and optional local assets | `curriculum/assets/` |
| Human update guide | `curriculum/README.md` |
| Generated local output | `curriculum/output/` (ignored) or the host's ignored build output |
| Generator adapter | Host-conventional route, template, stylesheet, script, validator, and command |

Place all non-sensitive content in locale files. Use one locale when the CV is
single-language; add one file per language when it is translated. Keep the
same stable IDs and ordering for equivalent timeline entries across locales.

## Content contract

Give each locale file a shallow, readable schema containing:

- `meta`, `identity`, `controls`, `contact`, and `sections` text;
- a profile paragraph;
- `experience` entries with stable `id`, `title`, `dates`, and `start` fields;
- `skills` strings;
- `languages` with `language` and `level` fields; and
- `education` entries with stable `id`, qualification, institution, dates,
  start, and end fields.

Validate parseability, required meaningful text, and stable cross-locale IDs.
Validate that telephone and email links use `tel:` and `mailto:` and that the
portrait resolves inside the repository. Make every schema change in the
renderer, every locale file, example private file when applicable, and
validator together.

Keep the private file ignored and provide this editable shape:

```yaml
phone:
  display: "+34 600 000 000"
  href: "tel:+34600000000"
email:
  display: "name@example.com"
  href: "mailto:name@example.com"
portrait_path: curriculum/assets/portrait.jpg
```

## Host adapter

Choose one adapter after inspecting the repository; do not assume a generator,
package manager, or language.

- **Static-site generator:** add local CV pages and a CV-only configuration
  overlay or build flag. Keep `curriculum/` excluded from the normal build and
  render it only through the dedicated command.
- **Application framework:** add a static or server-rendered CV route only to
  the dedicated local build. Keep content parsing, print CSS, and optional fit
  script isolated from normal application bundles where the framework allows.
- **Plain static website:** generate standalone semantic HTML in the ignored
  output directory. Link the site's existing CSS or a small CV stylesheet that
  uses its tokens and locally available fonts.

Name the dedicated command for the repository's conventions (for example,
`bin/cv`, `npm run cv`, or `make cv`). It must validate before generation and
provide a locale-aware preview when the environment supports it.

## Document and print contract

Render a semantic document with a skip link, labelled controls, a heading
hierarchy, semantic lists for timeline content, and a meaningful portrait alt
text. The screen controls offer language switching, a live fit status, and a
Print / Save as PDF button.

Set the physical sheet to exactly 210 mm × 297 mm, with content inside a
bounded frame. Use `@page` A4 with zero margin; hide screen controls in print;
and preserve background colour. Keep controls usable on a narrow screen even
when the A4 sheet itself scrolls horizontally.

The fit script measures after fonts and images resolve. It first tests the
normal density, then a deliberately bounded compact density. Its outcome is
informational: call `window.print()` after measuring even if content may
overflow. Never turn an imperfect measurement into a print block.

## Update guide and hand-off

```zsh
cp curriculum/private.example.yml curriculum/private.yml
# Edit curriculum/data/, curriculum/private.yml, and curriculum/assets/.
<cv-command> build
<cv-command> preview <locale>
<host-build-command>
git diff --check
```

Document the repository's actual commands and generated locations in
`curriculum/README.md`; do not leave placeholders in the delivered workflow.
Keep the real private file and output ignored. Confirm a normal host build does
not emit the CV unless publication was explicitly requested.

Advise the human to select A4, scale 100%, no margins, and background graphics
in the native print dialog. Native preview is the final authority on one-page
fit and clipping, so report it only after it was actually inspected.
