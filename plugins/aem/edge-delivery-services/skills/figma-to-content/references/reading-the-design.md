# Reading the design — segmentation, placeholder content, site chrome

Detail for SKILL.md **Phase 1**. The Figma MCP capabilities and the call
budget/order stay in SKILL.md; this file holds the three judgement calls that
decide *what the section inventory actually contains*.

---

## 1. Segmentation heuristic

When the frame has no explicit grouping, derive the section list like this — **not**
from the raw child order:

1. Sort the frame's direct children by `y` (top to bottom).
2. **Drop pure-decoration layers** from the section list — full-bleed background
   rectangles, gradients, blurs, absolutely-positioned shapes with no text or
   interactive child. Record each as the *background* of the content it sits behind
   (→ Phase 4 `section-metadata`); don't emit it as a section of its own.
3. **Merge siblings that form one visual band** — nodes whose vertical extents
   overlap or sit within ~one line-height of each other (a background rect + a
   heading group + a button row are *one* section, not three).
4. **Reconcile the count against the screenshot** before resolving: the eye sees
   the real sections; a mismatch means you over- or under-split — fix it before
   Phase 2. **On a very tall frame, one full-frame image can't do this.** A long
   landing page is routinely 8–10× its width, and a single render of that aspect
   ratio is too small to read — reconcile against a handful of per-band crops
   instead, and don't treat the unusable full-frame shot as the anchor Phase 1
   asks for.

Why it matters: a full-bleed background rectangle emitted as its own "section"
produces an empty section in the output and throws off the section count the Phase
5 gate checks against the plan.

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
