---
type: "documentation"
title: "Images and equations in every kind of source v1 (Track T)"
description: "How an image becomes a source, how the images a document embeds are kept and read, how an equation already written in a source becomes an equation, and how the figure and equation routes work without docling. Plan..."
status: "stable"
reading_status: "read"
product: "second brain"
product_version: "0.1.0"
doc_kind: "contract"
audience: "people and agents using or changing second brain"
doc_path: "docs/images_v1.md"
doc_sha256: "2f189fecad6775b91aaf98499097816dbf8dcddf5b72a13c87359d04ba71663d"
---

# Images and equations in every kind of source v1 (Track T)

How an image becomes a source, how the images a document embeds are kept and
read, how an equation already written in a source becomes an equation, and
how the figure and equation routes work without docling. Plan:
`docs/knowledge_layer_improvement_plan_v1.md`, Track T. Builds on
`docs/document_conversion_v1.md` (Track M: figures, vision) and
`docs/equations_v1.md` (Track N: equation regions, reading on request).

## Why

Tracks M and N were built and tested on PDFs read by docling, and an audit of
main (2026-10-11) found that everything involving an image or an equation
worked only there. A standalone image reached a UTF-8 read and failed with an
opaque 400, after retention had already copied it into the workspace; an
image embedded in Markdown, HTML or a Word file was never catalogued (and the
one-call read deleted its link); LaTeX already written in a note gave no
Equation item, and a bullet that was mostly a formula was dropped from the
graph; `/sources/figures` told every non-PDF source to "re-read with
docling". A second brain for recipes, customer relationships, a lab notebook
or a blog archive needs the same abilities as one for papers: a photo of a
dish, an equation in a blog post, a whiteboard pasted into a meeting note.
Nothing below names a subject; every rule is about the shape of a file.

## The one catalog format

Everything here writes the catalogs Tracks M and N defined, beside the
converted copy, so every route and every answer that reads them works
unchanged:

```
sources/converted_markdown/<kind>/<stem>.md
sources/converted_markdown/<kind>/figures/<id>/fig-<n>.<png|jpg|gif|webp>
sources/converted_markdown/<kind>/figures/<id>/figures.json        located_by: standalone | embedded
sources/converted_markdown/<kind>/equations/<id>/eq-<n>.<ext>      (an equation that arrived as an image)
sources/converted_markdown/<kind>/equations/<id>/equations.json    located_by: text
```

A figure entry adds to Track M's fields `origin` (`standalone` or
`embedded`), `source_ref` (the path, URL or Word part the image came from),
`alt`, `media_type`, `caption_basis` (`given by the caller`, `figcaption`,
`caption paragraph`, `caption line`, `title`, `alt text`, `file metadata`),
`decoration` (with `decoration_reason`) for an image told to be a logo, an
icon or a running header by its size, its repetition or its place (Track AD3,
below), `heading_path` (the document's headings its block sits under, Track
AD1), and `not_kept` with the reason when the image could not be read. `page` is null: these sources have no
pages. An equation entry adds `from_text` (written in the source) or
`from_image` (with its crop), `display`, `delimiter` and, for a written one, a
`reading` with `method: text`.

A PDF's figures and equations stay docling's: a PDF converted by another
converter gets none of this, and a docling catalog left beside a copy made
again by another converter is neither overwritten nor written into notes
(its readings wait, by hash, for the next docling re-read, as before).

## T1. Images are sources

`POST /sources/process` (and folder intake, and URL intake) takes an image:
PNG, JPEG, GIF and WebP, the formats the vision backends here read, judged by
the file's own bytes (its magic number), never by its name. The image is kept
as the raw source and its converted copy is the image's title, its caption,
one figure block and a line saying what the file is (`image/jpeg, 192 x 144
px, 7 KB; taken …; file …`): what a vision reading is given as the document's
opening.

**The template.** `format-templates/image.md` (`type: image`): the mandatory
sections, then `## Caption` (the caller's words, else the file's own),
`## What it shows` (empty until vision reads the picture, then the reading,
marked `Interpretation (from the image): … [read from the image by …]`) and
`## Where it came from` (file, type and size, when it was taken and by whom
when the file says, the page it came from). Frontmatter `caption`,
`media_type`, `width`, `height`, `captured_at`, `creator`, `origin`. The note is
**built without an agent** (`deterministic: true` on the template): an agent
asked to extract claims from a caption would be paid to invent them. Its
empty sections stay empty (an honest gap), never `TODO`.
Its map (`scripts/build_image_maps.py`, `image-caption`) puts the caption at
the centre, where it came from on one side and what it shows on the other,
marked "read from the image: weaker than any text".

The caller's `kind` chooses the template as for any source; when that
template does not take images (its `raw_source_kinds` lack `image`), the image
template is used and the result says so in `template_for_image`
(`requested`, `used`, `why`). A template added later that takes images is
used as asked.

**The caption**, strongest first: `caption` in the request; the file's own
description (XMP `dc:description`, PNG `Description`, EXIF `ImageDescription`,
Windows `XPSubject`); its title (XMP `dc:title`, PNG `Title`, `XPTitle`); a
comment (JPEG `COM`, GIF comment, PNG `Comment`, EXIF `UserComment`). A
camera writes its own name as the description of every photo (`… DIGITAL
CAMERA`, `… DSC`): one that repeats the camera's make or model, or is a few
capitalised words with no lower case, is not a caption. **GPS coordinates are
never read** (`second_brain/figures/images.py`): where a photo was taken can
name a home, and nothing here needs it.

**Refused before anything is copied.** A file the pipeline cannot read is
refused with `415 {reason, message}` before retention copies it, on the
background path too: `unsupported_image` for an image format no vision
backend reads (BMP, TIFF, HEIC, AVIF, SVG, ICO: "convert it to PNG or JPEG")
or one past the size limit (25 MB), `unsupported_binary` for a binary that is
neither a document the pipeline converts nor text (an old `.doc`, an archive,
audio). Any other text file is read as before.

**URL intake** accepts a URL that serves an image (`image/png`, `image/jpeg`,
`image/gif`, `image/webp`): kept as served, never decoded as text (the
hidden-instruction scan reads text; an image's bytes are not text), refused
when its bytes are not the type it was served as or when it was cut short.

**Folder intake.** An image in the folder is a source of its own
(`action: process`, `image: true`, `estimated_tokens: 0`: the note costs no
extraction), unless a document of the same folder embeds it (`![…](…)`,
`<img>`, `![[…]]`): then `action: embedded`, "embedded in X; taken in with
that document". A Markdown, HTML or Word file reports how many images it holds
(`images`). The vision allowance of a dry run counts PDFs, image sources and
documents that hold images. An image format no backend reads is
`unsupported`, with the reason. With per-file kinds (Track U4,
`file_kinds.py`), an image file's kind is `image` by its file type, whatever the
folder's kind.

## T2. Images inside any document

After conversion (and for Markdown, after the direct copy), every image the
copy references is kept (`second_brain/figures/embedded.py`):

- **Markdown**: `![alt](path "title")`, `<img src alt title class>`, and
  Obsidian's `![[name|alt]]` (found by name under the note's folder or its
  parent, as a vault does). Not inside code, figure blocks or comments.
- **HTML**: the converter's Markdown, with each image's `<figcaption>`, alt,
  title and class read from the page itself (`page_images`), matched by path.
- **Word (DOCX)**: the pictures in the file (`second_brain/figures/docx_media.py`:
  `word/document.xml`, its relationships and `word/media/`), in reading order,
  with the alt text and title Word stores and the caption: a paragraph in the
  document's caption style just after or before the picture, the style found by
  its built-in name in `styles.xml`, never by its id, which Word translates.
  Matched to the converter's image references by order; when the converter
  dropped them, each picture is placed after the paragraph before it.

**Where the bytes come from.** A local path is resolved against the folder the
document came from (taken before retention copies the file away from its
images), and read only when its bytes are an image within the size limit: a
path in a document can point anywhere, and only an image is ever copied.
Inline data URIs are decoded (one the converter cut short is reported). A
**remote image** is fetched only through `conversion/web_fetch.py` (http and
https, no internal address before or after a redirect, no cookies, a size
limit, image types by served type and by bytes) and only when
`images.fetch_remote` in `config/ingestion.json` allows: `fetched_pages` (the
default: for a page taken in by URL, whose site the owner already chose to
contact, relative paths resolved against the page), `always`, or `never`. A
local file's remote images are not fetched by default, because fetching an
image can tell its server the document was opened. One not fetched is still
catalogued, by its caption, with the reason. An image the source no longer
reaches (a moved original, a page offline) is carried over from the earlier
conversion by its path. `images.max_per_source` (40) caps the images kept.

**The caption**, strongest first: a `<figcaption>` or caption paragraph; an
italic line right under the image (consumed into the block, not repeated); the
image's title; its alt text (not when it only repeats the file name); the
file's own metadata. A caption from a caption source gives the figure its
label (`Figure 1`), so a pass finds the text that refers to it.

**In the copy** the reference becomes the figure block Track M writes
(`<!-- figure:n sha256=… -->`, `### Figure n`, the image, `**Caption:**`); an
image inside a sentence leaves `[Figure n]` there and its block after the
paragraph. A reading of the same image (by hash) from an earlier conversion is
carried over. The note gets `## Figures`, ingestion a Figure node per image,
the answer's `unread_regions` lists the unread ones with `recover_figures`,
`/sources/figures` reads them; all unchanged.

**Whether an uncaptioned image is read** is the kind of source's to decide,
not one global switch (`vision.interpret_uncaptioned` in `config/ingestion.json`):

| setting | effect |
|---|---|
| `null` (the default) | each template's own default: a document skips an image it gives no caption for (mostly logos and decoration); an image-centred template (`image`) reads it |
| `true` / `false` | every template |
| `{"<template_id>": true \| false}` | the templates named; the rest keep their default |

Also: a **standalone image** is never skipped for want of a caption (the
picture is the source); a note, page or Word file whose prose is short beside
its pictures (600 characters per picture, up to three) is **image-centred**
and reads them unless its template is set otherwise (a PDF is always a
document); a **decoration** is `decoration_skipped` (Track AD3,
`figures/decorations.py`), told by the document's shape for every source,
never by a file name or alt text:

| reason | when |
|---|---|
| `too small to carry content` | 48 px on its long side, or 4,096 pixels: an icon, a badge, a tracking pixel (whatever its caption) |
| `repeated across the document` | the same image on more than one page (the same bytes, or, for a converter that crops each page afresh, the same size within 3 px at the same place within 2% of the page), or the same bytes used three times in a document without pages; an image the document captions, or one repeating a captioned figure, never counts |
| `in the page header or footer band` | wholly inside the top or bottom 10% of its page and no taller than 12% of it; not for a captioned image |

A decoration stays in the catalog but is not listed in the note's
`## Figures` and is never offered in an answer.
Any image is still read when asked for by number. The plan reports
`uncaptioned_policy` (`reads_uncaptioned`, `template_id`, `image_centred`).

**Vision** is the same opt-in path as any figure: during processing only
when `vision.enabled`, after it through `POST /sources/figures` (a dry run
first). The `interpret_figure` job keys on the image's SHA-256, so the same
picture is never paid for twice. For an image note, the reading also fills
`## What it shows`.

**The one-call read** (`POST /agent/knowledge/read`) keeps an image as a
reference instead of deleting it: a figure block reads `[Figure n (image:
sources/converted_markdown/<kind>/figures/<id>/fig-n.jpg) **Caption:** …
**Interpretation (from the image):** …]`, a bare image `[image: <alt>
(<path>)]`, paths workspace-relative.

## T3. Equations already written are equations

`second_brain/equations/text.py` reads, by shape and in any language:

- display LaTeX: `$$…$$`, `\[…\]` holding mathematics (so a Markdown-escaped
  `\[1\]` is not one), and `\begin{equation|align|gather|multline|eqnarray|
  displaymath}` environments;
- inline LaTeX (`$…$`, `\(…\)`) only where unambiguous: mathematics that
  states a relation (`=`, `<`, `>`, `\le`, `\approx`, `\in`, …), never a price
  (`$5 and $10`, `US$5`); a lone symbol (`$x_i$`) is the sentence's, not an
  equation;
- never inside code, figure blocks, comments or a reading the pipeline wrote;
- **MathML** in a web page, made LaTeX before the page is converted
  (`second_brain/equations/mathml.py`): the author's own
  `<annotation encoding="application/x-tex">` or `alttext` when given, else
  the presentation elements converted (identifiers, numbers, operators with
  named symbols, rows, fractions, roots, scripts, limits, fences, tables),
  written as `$$…$$` for `display="block"` and `$…$` inline. A converter's
  Markdown escapes inside the LaTeX (`x\_i`) are taken out of the copy.

Each becomes a region **read from the text**, numbered with every equation of
the copy in order, with its heading path and the words before and after it.
The note's `## Source equations` lists it as
`- Equation 1: $$…$$ _(eq. 1, read from the text)_ — section: … — before: … — after: …`;
ingestion makes an `Equation` node (`derived_from: text`, `read: true`,
confidence 0.9: the source's own text), found by the words of its document,
section and context; `/sources/equations` lists it as `read_from_text` and
never plans it. The copy is not rewritten for it.

**An equation that arrives as an image**: an embedded image marked as one by
its markup (a class or role naming math, latex, tex, equation, formula, eqn,
katex or mathjax; Pandoc's `{.math}`), or whose alt text is LaTeX. Its crop is
kept (`eq-<n>.<ext>`), the copy carries `[equation not recovered (eq. N); see
the source page]` in its place (the marker's form without a page; the older
forms still parse), `unread_regions` lists it, and `POST /sources/equations`
reads it by vision (`read_equations`) exactly as a docling crop. Alt text that
is LaTeX is the author's own reading: the copy gets the LaTeX, the region a
reading from the text, and the crop stays for a vision re-read. A vision
reading is kept by the crop's hash.

**Math kept, not dropped.** A line of a note that is mostly a formula (a
bullet `- $$h = \frac{w}{f+w}$$`, a `$$` block, mathematics in Unicode
symbols) used to be dropped from the graph (a formula is not a claim). It is
now an `Equation` node (`equation:<doc>-text-<hash>`, `derived_from: text`,
labelled `Equation (as written) in <document>, under <section>: … $$…$$`);
one the note also lists in `## Source equations` is that equation, mentioned
again. It is still never a Claim.

## Hand-written notes (with Track U2)

A native note (one written in the wiki by hand, `notes/native.py`) has no
converted copy, so nothing is catalogued or copied for it, and it is never
rewritten. At ingest, a line of it that is one image reference
(`![alt](path)`, `<img>`, `![[name|alt]]`) becomes a Figure node
(`figure:<note>-img-<n>`, labelled `Image n: <alt or title> (<path>)`,
`derived_from: caption`, `native: true`), so a question about the picture
reaches the note; its formulas become Equation nodes as above. To have such an
image read by vision, process the note as a source (`POST /sources/process`).

## T4. The routes work without docling

- `POST /sources/figures` plans and reads any source with a figure catalog;
  a source with none says why in its own terms (a PDF: re-read with docling;
  anything else: it holds no image the brain could keep).
- `POST /sources/equations`: written equations are `read_from_text`; an
  equation image is `would_read`; `methods.local` is unavailable for a source
  that is not a PDF, with `offer: "vision"` and the reason ("the local formula
  model reads PDF pages only"), and `auto` chooses vision. `pages` on a source
  without pages is a 422 `pages_not_applicable` with the `known_ids`;
  `section` also finds regions by the headings they sit under. A source with no
  equation says so (`how_to_get`), with no docling advice.
- Rollback (`/sources/equations/rollback`, `/sources/reread/rollback`)
  restores the copy, its catalogs and crops, and the notes, for any source.
- `POST /sources/reread` on a source that is not a PDF converts it as before
  and catalogues its images and equations again; its dry run says there is no
  preview.
- The answer's `recover` and `recover_figures` bodies work as given.

## Settings

`config/ingestion.json` (workspace override as usual):

```json
{"vision": {"interpret_uncaptioned": null},
 "images": {"fetch_remote": "fetched_pages", "max_per_source": 40}}
```

A malformed value is replaced by its default and reported.

## Fixtures

`tests/fixtures/make_images_everywhere.py` (standard library only: a PNG
writer, a small baseline JPEG encoder, a 5x7 bitmap font, a DOCX built from
its XML parts) writes four unrelated synthetic sources: a recipe box (a dish
photo with its caption in a PNG text chunk; a recipe embedding a JPEG photo
with an EXIF description), a blog post with LaTeX in three forms and one
equation drawn as an image, a weekly meeting note in Markdown and as a Word
file with an embedded diagram and a caption paragraph, and a web page with a
`<figure>`/`<figcaption>` and MathML. `tests/fixtures/markitdown_stand_in.py`
reproduces the shapes markitdown writes for HTML and DOCX, for the suite.

## Limitations

- HTML and Word files need markitdown (`oc-markitdown` or the `markitdown`
  Python package) to be converted at all; without it they are refused as
  before. A Word file's own equations (OMML) are catalogued only when the
  converter writes them as LaTeX.
- EPUB, PowerPoint and spreadsheets: their images are not read yet.
- A hand-written note's images are Figure nodes by their words only: not
  copied, not in `unread_regions`, not readable by vision until the note is
  processed as a source.
- An equation image is recognised by its markup or LaTeX alt text; an
  unmarked one is a figure (vision still reads it, as a figure).
- Matching a question to an equation reads the words around it and its
  document, not its mathematics (as Track N).
- Image-centred is judged by length alone; a long photo essay with captioned
  images is a document, which is right only because its images have captions.
- The caption rules read metadata fields by their standard names; a tool that
  writes a caption under another name is not read.
