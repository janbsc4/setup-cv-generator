# Portable CV generator reference

## File contract

Use `curriculum/` as the complete human-editable boundary. Keep files that the
host framework needs to discover elsewhere only when its conventions require
it; point to them from `curriculum/README.md`.

| Responsibility | Files |
| --- | --- |
| Paper size and orientation | `curriculum/config.yml` or equivalent host-conventional config |
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
`bin/cv`, `npm run cv`, or `make cv`). It must validate before generation,
provide a locale-aware preview when the environment supports it, and expose a
`verify` operation that generates and verifies the PDF.

## Verification dependencies

Use Playwright with Chromium as the deterministic PDF renderer. Add
`playwright` to the existing JavaScript development dependencies and commit the
package-manager lockfile. If the host has no JavaScript toolchain, isolate a
small package manifest and lockfile under the generator adapter. Install the
matching Chromium binary during setup with the package-manager equivalent of
`npx playwright install chromium`. The PDF script must wait for fonts and every
document image, emulate print media, and call `page.pdf()` with
`printBackground: true` and `preferCSSPageSize: true`.

Use Poppler as an independent verification layer. Require `pdfinfo`,
`pdftotext`, and `pdftoppm`; fail setup with a direct platform-specific remedy
when they are unavailable. On macOS the remedy is `brew install poppler`.
Installing a system package requires the user's approval. Do not silently
downgrade `verify` to HTML or browser measurements when a dependency is
missing.

## Document and print contract

Render a semantic document with a skip link, labelled controls, a heading
hierarchy, semantic lists for timeline content, and a meaningful portrait alt
text. The screen controls offer language switching, a live fit status, and a
Print / Save as PDF button.

Before implementation, ask whether A4 is the correct target or whether the user
needs another paper size or orientation, unless the request already says. Store
that choice once and use it for `@page`, Playwright generation, verification,
preview labels, and hand-off instructions. Support named sizes such as A4,
Letter, or Legal and explicit dimensions for a custom target.

Set `@page` to the configured size with zero margin. Give page content a small
internal safety margin, but do not constrain the complete CV to one page or
shrink it solely to reach a preset page count. Let the normal print flow create
additional pages when content needs them. Use `break-inside` and deliberate
breaks to keep headings with their content and avoid splitting short entries
awkwardly. Apply forced breaks only at meaningful document or locale
boundaries, never after the final content. Hide screen controls in print and
preserve background colour. Keep controls usable on a narrow screen even when
the paper preview itself scrolls horizontally.

The fit script measures after fonts and images resolve. It reports clipping and
awkward page breaks at the normal density. Apply a bounded compact density only
when the user wants a denser design, not automatically to force a target page
count. Its outcome is informational: call `window.print()` after measuring even
if the layout may be imperfect. Never turn an imperfect measurement into a
print block.

Label screen-only geometry honestly, for example “Fit looks safe; PDF not
verified.” The page may display “PDF verified” only when `verify` has tested an
actual generated PDF for the current build. If the UI persists that state,
write a generated verification manifest containing a digest of the verified
input/output and ignore stale or mismatched manifests. Printing remains
available in every status.

## PDF verification contract

`<cv-command> verify` is a hard development/handoff gate. It must:

1. validate data, build the current document, and generate a fresh PDF with
   Playwright;
2. use print-mode DOM geometry to assert that content is not clipped and to
   derive the pages required by the current content at the configured paper
   size, allowing only an explicitly documented sub-pixel tolerance;
3. use `pdfinfo` to assert the configured page dimensions and the page count
   derived from that print layout, never a fixed one-page-per-locale rule;
4. use `pdftotext` per page to reject nearly blank pages with a documented,
   conservative non-whitespace threshold and to find locale-specific identity,
   section, and final-entry text markers;
5. use `pdftoppm` to render every page into an ignored verification directory
   for visual inspection; and
6. exit non-zero for any failed assertion and print the PDF and rendered-image
   paths on success.

Page-size, page-count, and extracted-text checks inspect the PDF, while the
print-mode DOM check catches clipping within correctly sized and counted pages.
Keep all of them; one is not a substitute for the others. The expected text
markers must come from the current CV data rather than private contact details
or hard-coded sample copy. Report the resulting page count per locale; a
multi-page CV is valid when its content requires those pages.

## Update guide and hand-off

```zsh
cp curriculum/private.example.yml curriculum/private.yml
# Edit curriculum/data/, curriculum/private.yml, and curriculum/assets/.
<cv-command> build
<cv-command> verify
<cv-command> preview <locale>
<host-build-command>
git diff --check
```

Document the repository's actual commands and generated locations in
`curriculum/README.md`; do not leave placeholders in the delivered workflow.
Keep the real private file and output ignored. Confirm a normal host build does
not emit the CV unless publication was explicitly requested.

Advise the human to select the configured paper size and orientation, scale
100%, no margins, and background graphics in the native print dialog.
Playwright/Poppler verification is the repeatable handoff gate. Native Safari
or browser preview remains a separate renderer check, so report it only after
it was actually inspected.
