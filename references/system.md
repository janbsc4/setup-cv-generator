# Portable CV generator reference

## Source discovery

Before creating the file contract, inventory existing CV sources: public HTML
pages, structured data, source documents, and tracked PDFs. Name one
authoritative source for each configured locale. A public page may be an
intentional summary while a PDF contains the full history; preserve that
distinction instead of merging by assumption.

For a legacy PDF, extract selectable text in this order:

1. use Poppler (`pdftotext`) or PDFKit when already available;
2. otherwise use Ghostscript's `txtwrite` device when available; and
3. ask the user for source content when extraction is incomplete, garbled, or
   otherwise untrustworthy.

Compare the extracted text with rendered pages before treating it as
authoritative. Record locale coverage explicitly. Configure only the locales
with equivalent source material unless the user supplies or approves a
translation.

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
Require email and validate its `mailto:` link. Phone is optional: require its
display and href fields to be both blank or both present, validate `tel:` only
when present, and render no telephone link when blank. Confirm that the
portrait resolves inside the repository. Make every schema change in the
renderer, every locale file, example private file when applicable, and
validator together.

Keep the private file ignored and provide this editable shape:

```yaml
phone:
  display: "" # Optional; fill both phone fields or leave both blank.
  href: ""
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
`bin/cv`, `npm run cv`, or `make cv`) and give it this operation contract:

- `build [locale]` validates data and produces the HTML source;
- `check [locale]` builds as needed and measures print layout without claiming
  PDF verification;
- `verify [locale]` builds HTML, generates a fresh PDF, and independently tests
  that PDF; and
- `preview [locale]` performs the build, advisory check, PDF generation, and
  verification stages, then opens the verified PDF in the platform's native
  viewer.

Implement `preview` as one pipeline over a single fresh HTML artifact rather
than repeatedly invoking the other subcommands. Print advisory `check`
findings and continue; stop before opening for validation, build, generation,
or verification failures. Open only the PDF from the current successful run.

When exactly one locale is configured, let `preview` omit the locale. When
several are configured, require one and list the valid values for a missing or
unknown locale before building. Open exactly that locale's PDF. Use the
platform opener (`open` on macOS, `xdg-open` on Linux, or `cmd /c start ""` on
Windows) when the environment supports desktop launching. Request approval
when a managed environment requires native GUI access. If launching fails,
preserve and print the verified PDF path while returning a preview-stage
failure.

Keep the generated HTML available as an intermediate artifact for design and
renderer debugging. Browser Print / Save as PDF remains an optional escape
hatch and a separate interactive-renderer check, not a routine generation
step.

Test `preview` with a successful single-locale run without an argument, an
explicit locale in a multilingual setup, and missing and unknown multilingual
locale arguments. Prove that advisory layout findings still open the verified
PDF, each hard pipeline failure prevents viewer launch, and an opener failure
reports the preserved verified artifact.

## PDF adapter choice

Choose the adapter from the workflow's real portability requirement, not from
an assumption that every local tool must be cross-platform.

### macOS-native adapter

Use a small Swift command backed by WebKit, PDFKit, AppKit, and CoreGraphics
when the generator is intentionally local to a current macOS system and
`xcrun swift --version` succeeds. This is the better fit for repositories whose
documented maintenance workflow already targets macOS/zsh: it avoids adding a
JavaScript toolchain, a downloaded Chromium runtime, and a separate PDF suite
solely for one local document command.

Generate the PDF with `WKWebView.printOperation(with:)`, attached to an
off-screen `NSWindow`, and run the print operation asynchronously so WebKit can
paginate on the AppKit event loop. Configure `NSPrintInfo` from the shared paper
configuration, save without showing panels, and keep native Print / Save as PDF
available in the HTML.

Run Swift with both module caches directed to a dedicated temporary directory,
especially in a managed environment:

```zsh
cv_cache_dir="${TMPDIR:-/tmp}/cv-swift-module-cache"
mkdir -p "$cv_cache_dir"
SWIFT_MODULECACHE_PATH="$cv_cache_dir" \
  CLANG_MODULE_CACHE_PATH="$cv_cache_dir" \
  xcrun swift script/cv_pdf.swift
```

A `WKWebView` still needs AppKit, font, LaunchServices, Web Inspector, and GPU
services. If loading or measurement stalls under a filesystem sandbox, request
approval and retry the same command with native GUI access. Treat the stall as
an environment failure, not evidence of a layout defect.

Await fonts and images with `callAsyncJavaScript`, return a JSON string, and
decode that string in Swift. Returning a Promise or nested JavaScript object
directly across the bridge is unreliable across SDK versions:

```swift
webView.callAsyncJavaScript(
    """
    await document.fonts.ready;
    await Promise.all([...document.images].map(image => image.complete
      ? Promise.resolve()
      : new Promise(resolve => image.addEventListener("load", resolve,
          { once: true }))));
    return JSON.stringify(window.cvPrintLayout());
    """,
    arguments: [:],
    in: nil,
    in: .page
) { result in
    // Decode the returned String with JSONSerialization.
}
```

Start printing with Swift's imported
`runModal(for:delegate:didRun:contextInfo:)`; the Objective-C selector
`runOperationModalForWindow:...` does not import as `runOperationModal(...)`.
Keep the web view, off-screen window, print operation, and delegate alive until
the completion callback finishes. Capture verification state on the main
queue, then close the window on the main queue after a short asynchronous
delay; closing it synchronously from AppKit's completion stack can crash.

Leave `NSPrintOperation.canSpawnSeparateThread` at its default. Forcing it off
can hang printing and grow an intermediate PDF without bound. Apply both an
elapsed-time limit and an output-size limit; on breach, stop generation, remove
only the known generated artifact, and report the guard that fired.

Verify the resulting PDF with PDFKit: assert media-box dimensions and page
content, normalize compatibility characters before matching CJK text, reject
interior blank pages, and use CoreGraphics/AppKit to render every page to PNG.
WebKit can emit a conclusively blank trailing sheet for some fragmented CSS
layouts; the adapter may remove only that trailing page after both extracted
text and rendered-pixel checks prove it blank, then rerun all assertions.

This path trades portability for a smaller local dependency surface. Do not use
it when Linux/Windows support, container execution, or CI generation is a real
requirement. Native WebKit generation is deterministic for the configured Mac,
but Safari's interactive print preview remains a separate renderer check.

### portable adapter

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
paper width, height, orientation, and internal safety margin once. Generate or
inject the corresponding CSS custom properties and `@page` rule, and use the
same parsed values in the layout probe, PDF adapter, verification, preview
labels, and hand-off instructions. Support named sizes such as A4, Letter, or
Legal and explicit dimensions for a custom target. Add a fixture using custom
dimensions so an A4 literal in any adapter causes verification to fail.

Set `@page` to the configured size with zero margin. Give page content a small
internal safety margin, but do not constrain the complete CV to one page or
shrink it solely to reach a preset page count. Let the normal print flow create
additional pages when content needs them. Use `break-inside` and deliberate
breaks to keep headings with their content and avoid splitting short entries
awkwardly. Apply forced breaks only at meaningful document or locale
boundaries, never after the final content. Hide screen controls in print and
preserve background colour. Keep controls usable on a narrow screen even when
the paper preview itself scrolls horizontally.

The fit script measures after fonts and images resolve. Mark each major section
with a stable selector such as `data-cv-section`. For every locale, calculate
the projected page count from the configured printable page height and report
each major section whose top and bottom fall on different pages. Run this
`<cv-command> check` probe as soon as real content and baseline print CSS exist,
before typography and spacing are polished, and repeat it after material
content or layout changes. Its output must distinguish a normal multi-page flow
from a major-section split so the user can judge whether a deliberate break is
better.

Ordinary DOM coordinates do not reflect forced print breaks. Put deliberate
break metadata in markup, for example `data-cv-break-before="page"`; let print
CSS select that attribute and make the probe add the corresponding virtual page
offset. After major sections no longer split, inspect the rendered pages for
balance. A valid boundary that leaves most of a page blank should move to a
better semantic boundary when one exists.

The fit script also reports clipping and awkward page breaks at the normal
density. Apply a bounded compact density only when the user wants a denser
design, not automatically to force a target page count. Its outcome is
informational: call `window.print()` after measuring even if the layout may be
imperfect. Never turn an imperfect measurement into a print block.

Label screen-only geometry honestly, for example “Fit looks safe; PDF not
verified.” The page may display “PDF verified” only when `verify` has tested an
actual generated PDF for the current build. If the UI persists that state,
write a generated verification manifest containing a digest of the verified
input/output and ignore stale or mismatched manifests. Printing remains
available in every status.

## PDF verification contract

`<cv-command> verify` is a hard development/handoff gate. It must:

1. validate data, build the current document, and generate a fresh PDF with the
   selected adapter;
2. run the adapter's geometry checks for clipped elements, allowing only an
   explicitly documented sub-pixel tolerance;
3. assert configured PDF page dimensions and content-driven pagination, never
   a fixed one-page-per-locale rule;
4. inspect extracted text per page to reject nearly blank pages with a
   documented, conservative threshold and find locale-specific identity,
   section, and final-entry markers;
5. render every page into an ignored verification directory for visual
   inspection; and
6. exit non-zero for any failed assertion and print the PDF and rendered-image
   paths on success.

Exercise the adapter with fixtures for one page, ordinary multi-page flow, an
intentional page break, a nearly blank trailing page, a missing expected
marker, clipped content, and custom paper dimensions. The failure fixtures must
prove that `verify` exits non-zero for the intended reason.

For the portable adapter, use print-mode DOM geometry to derive the expected
page count, then use `pdfinfo`, `pdftotext`, and `pdftoppm` for steps 3-5. For
the macOS-native adapter, use HTML overflow checks before printing, let WebKit's
print operation determine pagination, then use PDFKit plus
CoreGraphics/AppKit. Page size, page count, extracted text, and rendered pages
inspect different failure modes; keep all of them. Expected text markers come
from current CV data rather than private contact details or hard-coded sample
copy. Report the resulting page count per locale; a multi-page CV is valid when
its content requires it.

## Native adapter troubleshooting

| Symptom | Classification | Required response |
| --- | --- | --- |
| Swift cannot write its module cache | Environment | Set both Swift and Clang module-cache paths to the dedicated temporary cache. |
| WebKit load or probe stalls with service denials | Environment | Retry with approved native GUI access. |
| JavaScript reports an unsupported result type | Bridge | Use `callAsyncJavaScript`, return `JSON.stringify(...)`, and decode the string. |
| Modal print API does not compile | Swift API | Use `runModal(for:delegate:didRun:contextInfo:)`. |
| AppKit crashes during callback cleanup | Lifetime | Retain print objects and defer main-queue window cleanup. |
| PDF grows rapidly or printing never completes | Runaway print | Keep the default print worker; enforce time and size limits. |
| `check` disagrees after a forced CSS break | Model drift | Drive CSS and virtual probe offsets from shared break metadata. |

## Update guide and hand-off

```zsh
cp curriculum/private.example.yml curriculum/private.yml
# Edit curriculum/data/, curriculum/private.yml, and curriculum/assets/.
<cv-command> preview <locale>
<host-build-command>
git diff --check
```

Document `build`, `check`, and `verify` as focused diagnostic and automation
operations alongside the primary `preview` workflow.

Document the repository's actual commands and generated locations in
`curriculum/README.md`; do not leave placeholders in the delivered workflow.
Keep the real private file and output ignored. Confirm a normal host build does
not emit the CV unless publication was explicitly requested.

For optional browser-print inspection, advise the human to select the
configured paper size and orientation, scale 100%, no margins, and background
graphics in the native print dialog. Generated-PDF verification through the
selected adapter is the repeatable handoff gate. Native Safari or browser
preview remains a separate renderer check, so report it only after it was
actually inspected.
