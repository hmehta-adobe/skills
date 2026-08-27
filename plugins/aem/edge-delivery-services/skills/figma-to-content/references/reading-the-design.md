# Reading the design — segmentation, placeholder content, site chrome

Detail for SKILL.md **Phase 1**. The Figma MCP capabilities and the call
budget/order stay in SKILL.md; this file holds the three judgement calls that
decide *what the section inventory actually contains*.

---

## 1. Segmentation heuristic

When the frame has no explicit grouping, derive the section list like this — **not**
from the raw child order:

1. Sort the frame's direct children by `y` (top to bottom).
2. **Descend into wrapper frames before classifying.** A top-level child is not
   necessarily one band: designers often wrap several bands in a single frame
   (commonly named `Landing page`, `Frame 123`, `Content`), and that wrapper may
   mix **chrome with content** — a nav bar sitting alongside the hero and a logo
   strip. Never classify or drop a whole wrapper on its own name; if a child
   contains multiple full-width bands, treat *its* children as the candidates. A
   frame that spans nearly the whole page height is a wrapper, not a section.
3. **Drop pure-decoration layers** from the section list — full-bleed background
   rectangles, gradients, blurs, absolutely-positioned shapes with no text or
   interactive child. Record each as the *background* of the content it sits behind
   (→ Phase 4 `section-metadata`); don't emit it as a section of its own.
4. **Attach short satellite bands to the band they serve — don't threshold on the
   gap.** Two very common conventions break a distance rule:
   - **A section heading kept in its own layer group.** A short text-only band
     (a heading, or heading + standfirst) immediately above a much taller content
     band belongs to it, *however large the gap*. Real designs space these on a
     layout grid — 80px is typical, comfortably more than a line-height — so a
     "within ~one line-height" test emits an orphan heading section **and** a
     headless content section, doubling the section count.
   - **A lone control between bands.** A single `instance`/button (a "See all",
     "View more" CTA) sitting between two bands is the trailing element of the one
     above it. It is neither decoration nor a section of its own.

   The reliable test is **role, not distance**: a band that cannot stand alone as a
   page section — a bare heading, a lone button — attaches to its nearest
   substantial neighbour. Merge siblings whose vertical extents overlap on the same
   basis (a background rect + a heading group + a button row are *one* section).
4. **Reconcile the count against the screenshot** before resolving: the eye sees
   the real sections; a mismatch means you over- or under-split — fix it before
   Phase 2. **On a very tall frame, one full-frame image can't do this.** A long
   landing page is routinely 8–10× its width, and a single render of that aspect
   ratio is too small to read — reconcile against a handful of per-band crops
   instead, and don't treat the unusable full-frame shot as the anchor Phase 1
   asks for.

Why it matters: every one of these mistakes lands on the section count the Phase 5
gate checks against the plan. A background rectangle emitted as its own section
produces an empty section; an orphaned heading produces two wrong sections where
there was one, and a headless content band that then reads as needing a block it
doesn't need.

## 2. Placeholder content is common — don't ship it

Designs routinely contain dummy copy (`Lorem ipsum`, a CTA literally labelled
"Button" or "Lorem Ipsum", the same card title repeated across every card) and
unfilled slots (empty or transparent image cells, blank stat boxes).

- Author from the **real text and media in the design context** — not from the
  placeholder, and not from invented filler.
- Where it's clearly placeholder, **flag it in the plan and confirm the real
  copy/media with the user** rather than publishing "Lorem Ipsum" to a live page.
- Distinct items (cards, tabs, news entries) need **distinct** copy and images —
  repeated-identical content is itself a placeholder smell.
- If the design *itself* carries only placeholder, you cannot manufacture the real
  content: **stop and get it from the user before publish.**

The Phase 5 pre-publish gate greps the deployed output for exactly these markers,
so a placeholder that survives Phase 1 fails the gate later — cheaper to flag now.

## 3. Site chrome is usually not page body

Site chrome (nav bar, footer) is usually **not** page body — in EDS it is sourced
from separate `/nav` and `/footer` documents via the header/footer blocks. Don't
author it into the page unless the user asks.

A frame that includes a rendered nav and footer is showing you the page *in
context*, not listing two sections to build.

**Chrome is not always a top-level sibling.** In some files the nav sits inside a
wrapper frame together with real content (§1 rule 2) — so "drop the nav" means
dropping *that child*, not the wrapper, or you lose the hero with it. Identify
chrome by what it contains (a logo + a link row; a copyright line + link columns),
never by its position in the child list.
