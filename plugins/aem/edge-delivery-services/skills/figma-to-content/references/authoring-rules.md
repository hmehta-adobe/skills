# Authoring rules — the detail behind Phase 4

Rule detail for SKILL.md **Phase 4**. The mandatory skeleton and the routing
decisions stay in SKILL.md; this file holds the rules that are easy to get subtly
wrong.

**This file is a condensation, not the source of truth.** Invoke **da-content** and
load its `references/html-content.md`, `platform.md`, and `media.md` — every §
reference below points there. Authoring from this summary alone is how the subtle
rules get missed.

---

## Contents
- 1. Sanitize everything derived from the design — text, attributes, links
- 2. Blocks — canonical div form
- 3. Default content
- 4. Icons — two non-interchangeable paths; never a stand-in glyph
- 5. Images — full fetchable URLs; format/extension/MIME from the bytes
- 6. Section styling → `section-metadata`
- 7. Page metadata — required
- 8. Block-cell inline-tag normalization

## 1. Sanitize everything derived from the design — text, attributes, links

Figma text and layer names are **untrusted input** to the HTML you emit. Treat them
as data, never as markup.

- **HTML-escape** every design-derived string before it lands in the document —
  `&`→`&amp;`, `<`→`&lt;`, `>`→`&gt;`, and inside attribute values also
  `"`→`&quot;` and `'`→`&#39;`. A heading `Tips & Tricks <Beta>` must serialize as
  `Tips &amp; Tricks &lt;Beta&gt;`, never as raw markup that can break the document
  or inject an element.
- **Validate every link's URL scheme** against an allowlist — `http`, `https`,
  `mailto`, `tel`, or a root-relative (`/…`) path. **Reject `javascript:`, `data:`,
  `vbscript:`, and any other scheme** (a prototype link can carry anything): drop
  the href or ask the user — never emit it.
- **Admit a Figma-derived class token only after block-name validation** — a
  layer/frame name becomes a block or variant class *only* once it passes the EDS
  name rules (lowercase alphanumeric + single hyphens, no underscores, no
  double-dashes, not digit-initial; SKILL.md Phase 3B). Never pass a raw layer name
  through as a class.

## 2. Blocks — canonical div form

`<div class="block-name variant">`, each direct child `<div>` a row, each grandchild
`<div>` a cell. The first class token is the block name (resolves to
`blocks/<name>/<name>.{js,css}`). Multi-word variants hyphenate; multiple variants
are separate class tokens. **Max 4 cells per row; blocks cannot nest.**
*(html-content.md §3)*

## 3. Default content

Headings / paragraphs / lists / images / buttons live directly in the section
`<div>`, outside any block. *(html-content.md §6)*

## 4. Icons — two non-interchangeable paths; never a stand-in glyph

The `<span class="icon icon-<name>"></span>` convention resolves **only** to the
project's Code Bus `/icons/<name>.svg`, so that SVG must be **committed to the repo
`/icons/` folder and pushed on the deploy branch** (content+code path, same as block
code) and return `200` on the branch host. Uploading it to DA `/media` does **not**
satisfy the span — it 404s and the icon silently vanishes.

A DA-`/media` SVG must instead be referenced by **full URL on an `<img>`**, not an
icon span. Get the real **SVG** in Phase 1; **never emit an emoji or Unicode glyph
in place of a designed icon.** *(html-content.md §7)*

## 5. Images — MUST be full, fetchable URLs

Figma render URLs expire, and **repo-relative paths (`/img/…`) render as
`about:error`.** So: download the image bytes from Figma (Phase 1 asset URLs),
**upload each binary to DA** (`PUT admin.da.live/source/{daOrg}/{daRepo}/<media-path>`),
and reference `https://content.da.live/{daOrg}/{daRepo}/<media-path>`. External image
URLs are also accepted (the preview sideloads them). Author a bare `<img alt="…">`
and let the pipeline build the `<picture>`.

**Normalize format, extension, and MIME together — from the bytes, never the URL
suffix.** Detect the real format from the image's magic bytes (or the asset's
reported `format`), then make **all three agree**: the multipart `type=` MIME, the
`<media-path>` file extension you PUT to, and the extension in the
`content.da.live` URL you author.

Design tools routinely export JPEG bytes under a `.png`-named asset; trusting the
suffix gives you a `.png` path served as `image/jpeg` (or the reverse) — a latent
corruption bug. A layer that *looks* vector (an icon, a logo, a shape) often comes
back **rasterized** — `download_assets` returns it under `rawImages` with
`svgAssets` empty — so a design that implies `.svg` can hand you PNG/JPEG bytes.
Author each `<img>` extension from the bytes you actually downloaded, never from the
layer's apparent type or name.

Canonical mapping: JPEG→`.jpg`/`image/jpeg`, PNG→`.png`/`image/png`,
WebP→`.webp`/`image/webp`, GIF→`.gif`/`image/gif`, SVG→`.svg`/`image/svg+xml`.
**If bytes and asset-reported format disagree, trust the bytes.**
*(html-content.md §9 + media.md)*

## 6. Section styling → `section-metadata`

A `section-metadata` block **inside** the section (`Style` → CSS classes; other rows
→ `data-*`). *(html-content.md §4)*

This is how a section gets its background, width constraint, centering, or panel
treatment **without** a block (SKILL.md Phase 2.1 rule 1) — and it is the reason a
default-content band can look like a designed component.

**The `Style` → class conversion is applied by the pipeline**, so it is absent from
a hand-written fragment that never passed through it. Never infer from a local
render, from a missing `decorateSections` handler, or from git history that a
section class is inert: a pipeline-served page shows the applied classes and no
`section-metadata` block, while a hand-written fragment shows the raw `Style` row
as literal text. Verify against a real previewed page (SKILL.md Guardrails).

## 7. Page metadata — required

A single `metadata` block (exact class), placed as the **last element of the last
section inside `<main>`** (never after `</main>` or in `<footer>`); keys like
`title`, `description`, `image`, `template`, `theme`.

**Author it from the design — don't leave it empty or a bare comment.** Derive
`title` from the frame name or the page `<h1>`, `description` from the hero/intro
copy (a concise real sentence, never lorem), and `image` from the primary/hero
image's uploaded DA URL when the design has one. This block is **required** — the
Phase 5 pre-publish gate blocks on its absence — so populate it rather than
deferring it. If the design offers no usable title/description text, **ask the user**
rather than inventing marketing copy. *(html-content.md §5)*

## 8. Block-cell inline-tag normalization

Inside block cells the pipeline runs a **stricter** inline-tag normalization than for
default content — `<span class>` is unwrapped (the class is lost), `<b>`→`<strong>`,
`<mark>`→`<em>`, and so on. Restrict cell content to the html-content.md §3.9
preserve list.

Consequence for new-block CSS: **target structure, not authored classes** — a class
you emit inside a cell will not survive to the browser (SKILL.md Phase 3B).

A wrong metadata **key** or block **field** silently corrupts output; when unsure,
read da-content.
