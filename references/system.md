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
| Accepted pagination | `curriculum/page-plan/<locale>.yml` or `.json` |
| Local contact details | `curriculum/private.yml` (ignored), `curriculum/private.example.yml` |
| Portrait, licensed fonts, and optional local assets | `curriculum/assets/` |
| Human update guide | `curriculum/README.md` |
| Agent pagination guide | `curriculum/PAGINATION.md` |
| Generated local output | `curriculum/output/` (ignored) or the host's ignored build output |
| Generator adapter | Host-conventional route, template, stylesheet, script, validator, and command |

Place all non-sensitive content in locale files. Use one locale when the CV is
single-language; add one file per language when it is translated. Give every
pageable block a stable, meaningful ID and keep equivalent IDs across locales.
This includes profile blocks, section introductions, timeline entries, skill
groups, language items, education entries, and any other unit the paginator
may place independently. The generator must reject missing or duplicate IDs.

## Content contract

Give each locale file a shallow, readable schema containing:

- `meta`, `controls`, `contact`, and section-label text;
- an identity block with a stable `id`;
- a profile block with a stable `id` and text;
- `experience` entries with stable `id`, `title`, `dates`, and `start` fields;
- skill groups with stable `id` values;
- languages with stable `id`, `language`, and `level` fields; and
- `education` entries with stable `id`, qualification, institution, dates,
  start, and end fields.

Validate parseability, required meaningful text, and stable cross-locale IDs.
Render each pageable unit with `data-cv-block-id="<id>"`; the measurement
report, accepted page plan, generated HTML, and error messages use that same
identifier. A block larger than usable page height is invalid: split it into
smaller semantic blocks with new stable IDs instead of allowing an implicit
fragment.
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
  output directory. Generate its CSS from the site's tokens and embed it with
  local assets in the output artifact.

Name the dedicated command for the repository's conventions (for example,
`bin/cv`, `npm run cv`, or `make cv`) and give it this operation contract:

- `build [locale]` validates data and the accepted page plan, then produces the
  self-contained paged HTML artifact;
- `check [locale]` measures blocks, evaluates the accepted plan, solves
  candidate plans, and emits matching prose and JSON without claiming PDF
  verification;
- `plan <locale> --candidate <candidate-id>` validates the current measurement
  digest and explicitly persists that candidate as the accepted plan;
- `verify [locale]` builds HTML, generates a fresh PDF, and independently tests
  that PDF; and
- `preview [locale]` validates and measures, checks the accepted plan, builds
  final HTML, generates and verifies the PDF, then opens it in the platform's
  native viewer.

Implement `preview` as one pipeline over one fresh input snapshot rather than
repeatedly invoking the other subcommands. Generate the measurement and final
HTML artifacts through the same renderer during that pipeline. Print advisory
`check` findings and continue; stop before opening for validation, missing or
stale plan, planned-page overflow, build, generation, or verification failures.
Open only the PDF from the current successful run.

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
locale arguments. Prove that advisory balance findings still open the verified
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

## Measurement and page-plan contract

Render content from editable YAML or JSON into an ignored, unpaginated
measurement artifact. Wait for embedded fonts and images, then measure the
border-box height of every `data-cv-block-id` in document order. Put vertical
spacing inside blocks or represent it as an explicit measured stack gap; do not
let collapsing margins create unmeasured height. Use the same render functions
and print CSS as the final document so measurement is not a second layout
implementation.

Store paper width, paper height, orientation, and the four content margins in
one configuration object. Define the only canonical vertical capacity as:

```text
usable_page_height_mm = paper_height_mm - margin_top_mm - margin_bottom_mm
```

Every component reads that computed value; no script, CSS template, verifier,
or guide may restate a second printable-height or safety-margin formula. Keep
`@page` margin at zero and implement the configured content margins inside each
explicit page container. Support named and custom paper dimensions, and use a
custom-dimension fixture to expose hard-coded A4 assumptions.

The paginator consumes ordered block measurements plus semantic constraints
such as `keep_with_next`, `break_allowed_after`, and optional preferred section
boundaries. It must be deterministic for identical inputs. Produce several
ranked candidate plans when more than one valid boundary set exists; score
unused space and semantic penalties explicitly rather than mutating typography.
The accepted plan is a tracked, human-readable file containing a schema
version, locale, input digest, ordered pages, and ordered block IDs. Compute the
digest from every measurement input: normalized content and private render
inputs, page geometry, renderer and CSS, font bytes, asset dimensions, and the
measurement schema. Hash private values without exposing them in the plan. For
example:

```yaml
schema_version: 1
locale: en
input_digest: "sha256:..."
pages:
  - number: 1
    blocks: [identity, profile, experience-2026]
  - number: 2
    blocks: [experience-2023, education, languages]
```

Give each candidate a stable ID within its measurement report. Require the plan
to contain every current block exactly once and in source order. A missing
block, duplicate block, unknown block, changed input digest,
invalid boundary, or oversized block is a hard failure. `check` may still emit
candidate repairs before exiting non-zero so an agent can choose and persist a
replacement plan through the explicit `plan` operation. Never silently replace
an accepted plan.

Emit concise prose to stdout and write the equivalent structured report to an
ignored stable path such as `curriculum/output/check/<locale>.json`. Include at
least:

- schema version, locale, input digest, and all paper/margin dimensions in mm;
- the canonical usable page height;
- each block's ID, measured height in mm, source order, and constraints;
- the accepted plan's pages, used height, remaining height, and overflow;
- ranked candidate boundaries with page membership, score, and reasons; and
- hard violations and advisory balance warnings.

Use millimetres as the report and error interface, retaining enough precision
to diagnose the boundary while applying only one documented sub-pixel
tolerance internally. More than 20% unused height on a non-final page is an
advisory balance warning. Planned-page overflow is always a hard failure:
identify the page, the first offending block, and overflow in millimetres.

Choose a candidate by semantic quality: keep headings with their first block,
keep short timeline entries intact, and prefer boundaries between entries or
major sections. Accept whitespace when moving a block would harm readability.
Treat font size, line height, and global spacing as design inputs. Change them
only through a separately stated design decision, then remeasure and replace
the stale plan; never use global typography or spacing changes as fitting
mechanisms.

Exercise measurement and planning with fixtures for duplicate and missing IDs,
a block taller than usable height, a stale plan digest, a forced semantic
boundary, multiple ranked candidates, a page over capacity, retained
whitespace, and custom paper dimensions.

## Generated document and print contract

After a plan is accepted, generate the final HTML from structured content and
the plan. The HTML is an ignored build artifact, not the human-maintained
source. Make it self-contained: inline its CSS and encode or embed all images
and font files. Vendor only fonts whose licence permits repository use; when a
site relies on Google Fonts or another remote provider, obtain and record a
permitted local copy or choose a user-approved local fallback. Measurement,
HTML generation, and PDF creation must make no network requests.

Render one exact paper-sized `.cv-page` container per planned page and one
content box with the configured margins and canonical usable height. Place
blocks by ID, add a print break after every container except the last, and
assert page overflow in the rendered DOM before PDF generation. Set `@page` to
the configured size with zero margin, preserve background colour, and hide
screen controls in print. Keep the preview usable on a narrow screen without
letting screen width or scaling affect print geometry.

Render a semantic document with a skip link, labelled controls, heading
hierarchy, semantic lists, meaningful portrait alt text, and a Print / Save as
PDF button. Label screen geometry honestly; display “PDF verified” only for a
verification manifest whose input and output digests match the current build.
Printing remains available even when advisory balance warnings exist.

## PDF verification contract

`<cv-command> verify` is a hard development/handoff gate. It must:

1. validate data, build the current document, and generate a fresh PDF with the
   selected adapter;
2. run the adapter's geometry checks for clipped elements and planned-page
   overflow, allowing only an explicitly documented sub-pixel tolerance and
   reporting the first offending block plus overflow in millimetres;
3. assert configured PDF page dimensions and an exact page-count match with the
   accepted plan;
4. inspect extracted text per page to reject nearly blank pages with a
   documented, conservative threshold and find locale-specific identity,
   section, and final-entry markers;
5. render every page into an ignored verification directory and compose the
   page PNGs, in order with visible page labels, into one contact-sheet image;
   and
6. exit non-zero for any failed assertion and print the PDF, page-image, contact
   sheet, measurement JSON, and accepted-plan paths on success.

Exercise the adapter with fixtures for one page, ordinary multi-page plans, a
nearly blank trailing page, a missing expected marker, clipped content, plan
overflow, missing rendered output, and custom paper dimensions. Assert contact
sheet page order and prove that every failure fixture exits non-zero for the
intended reason.

For the portable adapter, use print-mode DOM geometry to validate every
explicit page, then use `pdfinfo`, `pdftotext`, and `pdftoppm` for steps 3-5.
For the macOS-native adapter, use HTML overflow checks before printing, then use
PDFKit plus CoreGraphics/AppKit. Page size, plan/page-count equality, extracted
text, rendered pages, and the contact sheet inspect different failure modes;
keep all of them. Expected text markers come from current CV data rather than
private contact details or hard-coded sample copy. Report the resulting page
count per locale; a multi-page CV is valid when the accepted plan requires it.

## Native adapter troubleshooting

| Symptom | Classification | Required response |
| --- | --- | --- |
| Swift cannot write its module cache | Environment | Set both Swift and Clang module-cache paths to the dedicated temporary cache. |
| WebKit load or probe stalls with service denials | Environment | Retry with approved native GUI access. |
| JavaScript reports an unsupported result type | Bridge | Use `callAsyncJavaScript`, return `JSON.stringify(...)`, and decode the string. |
| Modal print API does not compile | Swift API | Use `runModal(for:delegate:didRun:contextInfo:)`. |
| AppKit crashes during callback cleanup | Lifetime | Retain print objects and defer main-queue window cleanup. |
| PDF grows rapidly or printing never completes | Runaway print | Keep the default print worker; enforce time and size limits. |
| A planned page overflows after a content edit | Stale plan | Report the first offending block and overflow in mm, emit new candidates, and persist a reviewed replacement plan. |

## Update guide and hand-off

```zsh
cp curriculum/private.example.yml curriculum/private.yml
# Edit curriculum/data/, curriculum/private.yml, and curriculum/assets/.
<cv-command> check <locale>
<cv-command> plan <locale> --candidate <candidate-id>
<cv-command> preview <locale>
<host-build-command>
git diff --check
```

Document `build`, `check`, `plan`, and `verify` as focused maintenance,
diagnostic, and automation operations alongside the primary `preview`
workflow.

Document the repository's actual commands and generated locations in
`curriculum/README.md`; do not leave placeholders in the delivered workflow.
Create a short `curriculum/PAGINATION.md` for later agents. Name the stable-ID
field, page-plan path, prose and JSON check outputs, exact `check` and `plan`
commands for remeasuring and accepting a candidate, overflow failure format,
contact-sheet path, and the rule that typography or global spacing changes
require a separate design decision. Keep architecture and setup rationale in
this skill; keep only repository-specific maintenance facts in that guide.
Keep the real private file and output ignored. Confirm a normal host build does
not emit the CV unless publication was explicitly requested.

For optional browser-print inspection, advise the human to select the
configured paper size and orientation, scale 100%, no margins, and background
graphics in the native print dialog. Refresh the built HTML before printing and
inspect every edge of every page, especially later pages, for shrink-to-fit or
exposed-background strips. Generated-PDF verification through the selected
adapter is the repeatable handoff gate. Native Safari or browser preview
remains a separate renderer check, so report it only after it was actually
inspected.
