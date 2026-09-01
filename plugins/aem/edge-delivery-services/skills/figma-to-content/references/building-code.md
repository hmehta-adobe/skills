# Building code — new blocks (3B) and additive variants (3D)

Detail for SKILL.md **Phase 3B and Phase 3D**, the two content+code paths. Load
this file as soon as the plan routes any section to 3B or 3D — at Phase 2.3
manifest time, not later when you start writing code. A content-only run (reuse
and default content) never needs it.

**Routing, the hard guardrails and the pre-publish gate stay in SKILL.md.** This
file is the build detail. Before writing any block code, read SKILL.md's
Guardrails: additive only, never change an existing block's rules, and the commit
is block code plus icons and nothing else.

---

## Contents
- 1. Phase 3B — create a NEW block (content + code)
- 2. Phase 3D — extend an EXISTING block with an ADDITIVE VARIANT
  (four boundary conditions, the fork test, naming, build route)

## 1. Phase 3B — create a NEW block (content + code)

Only for sections Phase 2 routed here — the project has no suitable block, or the
divergence is too deep for an additive variant to express — the **3B** case.
**Check 3D first:** on a site that already has blocks, a new block is the answer of
last resort, not the first. **Guardrails (strict):**

- Create **new, isolated block folders** only (`blocks/<new-name>/`).
- **Do NOT** skin this block by editing an existing block, `scripts.js`, or
  `head.html`, or by adding block-specific rules to global CSS — keep it
  self-contained under `blocks/<new-name>/`. *(Retargeting the project's
  global design tokens in `styles/styles.css` — the `:root` custom properties
  and base typography — is a separate, allowed project-theming step, not part
  of building this block; see Guardrails.)*
- New block **names and variant tokens** must obey EDS block-name rules
  (da-content html-content.md §3.3): lowercase alphanumeric + single hyphens,
  **no underscores, no double dashes, must not start with a digit**
  (`pricing-table` ✓, `pricing_table` / `2col` / `promo--wide` ✗). Names must
  be unique and not collide with existing blocks.

**Build route — invoke content-driven-development (don't hand-write the block).**
Build every new block by **invoking content-driven-development** — not by writing
block JS/CSS from scratch off this file's summary. It invokes **content-modeling**
(design the authoring model from the Figma structure/tokens) then
**building-blocks** and **testing-blocks**, and produces a self-contained
`blocks/<name>/` — no source URL, no installed substrate, no page chrome, and no
global styles. That is the route that honors the 3B guardrails above, and its
testing-blocks pass is the block's Stage B verification (Phase 5). Build a **bespoke, one-off** section
the same way — it is still an ordinary isolated block, and "one-off" changes
nothing about how it is generated.

> **Do not use snowflake here.** Snowflake converts an *already-rendered* page:
> it requires a reachable **Source URL**, **installs an overlay substrate** into
> the repo, and in block mode emits **header/footer fragments and global
> styles/tokens** — each of which violates this skill's constraints (isolated new
> block, don't touch globals, work from the **Figma frame**, not a live URL).
> Snowflake is the right tool for a *different* entry point — converting an
> existing static/rendered site — as noted under "When NOT to use".

Use the Figma design context/tokens from Phase 1 as the source of truth for
layout and styling. New-block **CSS must target structure, not authored
classes** — inline wrappers like `<span class="…">` are stripped inside block
cells at delivery (da-content html-content.md §3.9), so a class you emit in a
cell will not survive.

**Make the block responsive.** A Figma page frame is almost always a single
**desktop** width, but EDS pages are responsive. Author the block mobile-first
(or with explicit breakpoints) so a multi-column layout collapses to one column
on narrow viewports, and verify at mobile / tablet / desktop via
**testing-blocks** — don't ship a fixed desktop-width block. If the design
provides a **separate mobile frame**, use it to derive the breakpoint behavior
(what stacks, what hides, how type scales) — it's the *same page*, so it feeds
one responsive block, **not** a second page (see Inputs on frame variants).

The new block's code must be **committed and pushed to the deploy branch on
GitHub and built by Code Sync** before the page can render it — see Phase 5
(content+code).

---

---

## 2. Phase 3D — extend an EXISTING block with an ADDITIVE VARIANT

For sections Phase 2 routed here: the block's **authoring model fits**, the look
doesn't, and the gap is expressible as rules scoped under **one new class token**.
This is the path that keeps a mature site from accumulating five near-duplicate
card blocks. It is also the only path in this skill that writes into code **other
pages already depend on**, so it is fenced on both ends.

**It is never automatic.** The user confirmed *this specific token* in Phase 2.2,
at any confidence level. If they haven't, you are in 3B or you are still asking —
not here.

**The four boundary conditions — all four, or it's 3B:**

1. **CSS is additive and scoped.** Every new rule sits under the new token
   (`.cards.compact { … }`). **Zero** rules changed, removed, or added at the bare
   block level (`.cards`) or under any existing variant. The moment you edit an
   existing selector you are skinning a shared block, which is forbidden.
2. **JS is untouched, or gains one gated branch.** At most a new branch guarded on
   the token (`block.classList.contains('compact')`), leaving every existing path
   byte-identical — see **building-blocks**' variant-detection
   guidance. Reworking the decorate function is 3B.
3. **The authoring model is unchanged** — same rows, same cells, same order. A
   variant needing a different content model would break every page already using
   the block: 3B.
4. **The regression set renders unchanged.** The pages find-test-content listed for
   this block (2.0(a)) **are** the regression set — **published pages only**, since
   that list comes from `query-index.json`, so check the DA source listing
   ([existing-content-discovery.md](./references/existing-content-discovery.md) §2b)
   for unpublished siblings the proof would otherwise miss. Render each before and
   after via **testing-blocks** and confirm no visual change. **An un-rendered regression set
   is a failed check, not a passed one** — the same fail-closed rule as the rest of
   this skill; if you cannot render them all, say so and fail the box rather than
   silently sampling.
   **Read its size as a signal, not just a cost.** A generic block on a mature site
   is easily used on dozens of pages, and every one of them is blast radius. Past a
   handful, the honest question stops being "can I scope this variant?" and becomes
   "should this be a new block instead?" — **3B is self-contained and needs no
   regression proof at all.** A large regression set is the cheapest early warning
   that you are about to take on shared-code risk to save a hundred lines.

**Fork test.** If the variant's rules mostly *override* the block's styling rather
than *add* to it, it is a fork wearing a variant's name — the block now has two
personalities, the next reader cannot tell which rules serve which, and you have
taken on the shared-code risk without the benefit. Build 3B instead.

**Naming.** The token obeys the same EDS rules as a block name (3B): lowercase
alphanumeric + single hyphens, no underscores, no double dashes, not
digit-initial. It must not collide with a token the block already defines, or with
a section `Style` class (2.0(a) step 3 listed the ones in use).

**Build route — invoke content-driven-development**, which explicitly covers block
*modifications*, not only new blocks. Don't hand-write the variant CSS from this
file. Its testing-blocks pass serves both jobs: the new section's fidelity **and**
the regression set.

Push the variant under the same commit discipline as new-block code (Phase 5), and
report which shared block you extended and which pages you re-verified (Phase 6).

---
