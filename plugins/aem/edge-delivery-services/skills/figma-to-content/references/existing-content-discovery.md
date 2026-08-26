# Existing-content discovery — read a real page before you build

> **When this applies.** Any target project that **already has content**. On a
> fresh or empty site there is nothing to read and building is correct — see §7.

On a mature EDS site, **an existing page in the same family is the highest-value
artifact available**, and finding one comes *before* any new-block decision.

Block source tells you what a block **can** do in isolation. A real page tells
you what the design system **does**: which variant *combinations* it composes,
which section `Style` classes it relies on, and — critically — **which sections
use no block at all**. When the two disagree, the page wins.

The failure mode this prevents: resolving several sections to new blocks, then
discovering an already-authored page in the same family that solved every one of
them with existing blocks plus section styling. Every line of that new block
code is redundant, and the plan the user confirmed was missing its correct
option.

---

## 1. Read channels — try them in order, stop at the first `200`

Existing content is reachable by several routes with different auth. Establish
which one works in **preflight** (SKILL.md Phase 0) and record it as the run's
read channel, so discovery is never blocked mid-run by a guess about reachability.

| Order | Channel | Notes |
|---|---|---|
| 1 | **Local dev server** — `http://<dev-host>/<path>.plain.html` | The `aem up` dev server **proxies authored content** from the configured origin, so it serves *their* pages, not just your local files. Usually already running for your own render. Cheapest and typically unauthenticated. Port is per project (`aem up` default is `3000`, but projects reassign it) — confirm the actual port, don't assume. |
| 2 | **Preview / live host** — `https://<branch-host>--<repo>--<owner>.aem.page/<path>.plain.html` (or `.aem.live`) | May require auth on access-protected sites. |
| 3 | **DA Source API** — `GET https://admin.da.live/source/{daOrg}/{daRepo}/<path>.html` with `$DA_TOKEN` | The authored source, pre-pipeline (see §5 for what that means). Also the route for **listing** (§2). |

```bash
# Probe the ladder; record the first channel that answers 200.
# Confirm the project's actual dev port first — 3000 is only the `aem up` default.
for base in "http://localhost:3000" \
            "https://$BRANCH_HOST--$GH_REPO--$GH_OWNER.aem.page"; do
  code=$(curl -s -o /dev/null -w '%{http_code}' "$base/index.plain.html")
  echo "$base → $code"
done
```

**A `401`/`403` on one host is an auth fact about that host — never proof the
content does not exist.** Fall through to the next channel. Concluding "existing
content is unreachable" from a single `401` is the error that makes every
downstream discovery step get skipped. (SKILL.md draws this same line for Figma
`403`-vs-transport-cap in Phase 0, for `429`-is-not-no-data in Phase 1, and for
`401`-is-not-a-new-page in Phase 5.)

---

## 2. Enumerate the pages, then prioritize the same family

```bash
# a) the published set
curl -s "$BASE/query-index.json" | head -c 2000     # paths + titles (if the project ships one)
curl -s "$BASE/sitemap.xml" | grep -oE '<loc>[^<]+'  # fallback

# b) the authored source tree (DA) — also lists unpublished pages
curl -s -H "Authorization: Bearer $DA_TOKEN" \
  "https://admin.da.live/source/$DA_ORG/$DA_REPO/"          # root
curl -s -H "Authorization: Bearer $DA_TOKEN" \
  "https://admin.da.live/source/$DA_ORG/$DA_REPO/<parent>/"  # siblings of the target
```

**Prioritize a page in the same family as the target** — in this order:

1. **The target path itself.** If it already exists, that *is* a same-family
   page: read it first. It may already solve every section, and it also decides
   the overwrite question (SKILL.md Phase 2.2).
2. **Path-prefix siblings** — the parent directory listing (e.g. everything
   under `/solutions/`, `/products/`, `/compare/`).
3. **Same template** — pages whose `metadata` block declares the same
   `template` or `theme`.
4. **Any page whose design shares sections** with the Figma frame.

**Read the listing for everything it says, not just the one question you asked
it.** A directory listing fetched to answer "is my target path free?" also
answers "what else lives here?" — same call, two purposes. Discarding the rest
of that response is how an already-authored sibling goes unnoticed.

---

## 3. Invoke find-test-content per reuse candidate

**find-test-content** (same plugin) already does block-keyed page discovery:
it queries the site's query-index, searches each page for a block, and reports
**every variant class found on that block element**. That variant list is the
composed token set §4 needs — it is the whole point of running it.

```bash
node .claude/skills/find-test-content/scripts/find-block-content.js <block-name> <host>
```

Invoke it for **each block you are considering reusing**, before deciding that
block doesn't fit. What it gives you: which pages use the block, how many
instances, and the real variant combinations.

**What it cannot tell you — cover these by reading the page directly (§4):**

- **Sections that use no block.** It searches *by block name*, so a section
  authored as default content in a styled section is invisible to it. This is
  exactly the case most likely to be over-engineered into a new block (§6, and
  SKILL.md Phase 2.1 rule 1).
- **Same-family pages** as such — it keys on a block, not a path prefix. Use §2
  for that.
- **Section `Style` classes**, which are not block classes at all (§5).

So: §2 finds the page, §3 finds the block usages, §4 reads the composition. All
three, not one.

---

## 4. Extract the real composition from a page

```bash
P=<path-without-extension>

# Every class token the page actually carries — block names, variants,
# and (on a pipeline-served host) section Style classes:
curl -s --compressed "$BASE/$P.plain.html" | grep -oE 'class="[a-z][a-z0-9 -]*"' | sort -u

# Rendered page: the section wrappers, with their applied Style classes
curl -s --compressed "$BASE/$P" | grep -oE '<div class="section[^"]*"' | sort -u
```

Read the output as three separate findings:

1. **The full class token list per block** — e.g. `cards icons icons-sm editorial
   carousel`, not `cards icons`. **Variant tokens compose, and a later token can
   override an earlier one's layout** — a `carousel` token that switches the
   block to a scrolling `display: block` viewport makes the `.cards.icons` 3-up
   grid rule irrelevant. Judging fitness from one variant's CSS in isolation,
   when the project ships four tokens together, reaches a conclusion about a
   rendering that never happens. **The authoritative rendered example of a block
   is that block on this page, with these tokens** — preferable to a generic
   `liveExampleUrl` whenever a real usage exists.
2. **Which sections carry no block** — a section whose only classes are section
   `Style` classes is default content in a styled section. Note it: the
   equivalent section in your design needs no block either.
3. **The section `Style` vocabulary the project actually uses** — e.g.
   `contained`, `dark`, `center`, `narrow`. These are reusable on your page for
   free, and they are frequently what makes a "designed-looking" band look
   designed (§5, §6).

Also worth reading from the page: the authoring model in practice (which cell
holds what), and the `metadata` block's `template`/`theme` keys.

---

## 5. Section `Style` classes are applied by the pipeline — not visible in a
hand-written fragment

`section-metadata` `Style` values become **classes on the section wrapper**. That
transform happens when the document is served **through the pipeline** (preview /
live). Consequences:

- A **real, previewed page** shows the applied classes — its `.plain.html`
  carries e.g. `<div class="dark spacer-top-xxl">` and **no** `section-metadata`
  block, because the block was consumed.
- A **hand-written local fragment** has never been through that transform, so it
  still contains the literal `section-metadata` table and shows `Style dark` as
  visible text.

**Never conclude a section class is inert or dead CSS from any of these:**

| Observation | Why it proves nothing |
|---|---|
| `Style dark` renders as visible text locally | Your fragment hasn't been through the pipeline |
| No `section-metadata` handler in `decorateSections` | The transform is pipeline-side; a client-side handler is not required |
| No such handler anywhere in git history | Same — there was never anything to commit |

Each observation can be individually accurate and the inference still wrong.
**Verify against a real previewed page** (§4, second command) before deciding a
section class does nothing.

Getting this backwards has two costs: purely section-styled bands look like they
need bespoke blocks, and genuine fidelity gaps get reported as unfixable when
they were always fixable with an existing class.

---

## 6. Worked example

Design frame has three bands you'd reach for a block for: a **logo + label
strip**, a **4-up icon feature row**, and a **closing CTA panel** (heading,
paragraph, two buttons, in a dark rounded centered container).

Blocks-only survey (the wrong route) concludes: `cards` exists but
`.cards.icons` is `grid-template-columns: repeat(3, 1fr)` and the design is 4-up
⇒ new block; the logo strip has a bespoke badge treatment ⇒ new block; the CTA
panel looks like a designed component ⇒ new block. **Three new blocks.**

Pages-first (§2 → §3 → §4) finds `/solutions/<sibling>` in the same family and
reads it:

| Band | What the real page does | Resolution |
|---|---|---|
| Logo strip | `cards minimal divider`; eyebrow is a default-content `<p>` above the block; badge cell + label cell | **reuse** — content model is identical |
| Icon feature row | `cards icons icons-sm editorial carousel`; icons as 36×36 SVG `<picture>` | **reuse** — `carousel` overrides the 3-up grid to a scrolling viewport, so the 4-up objection was against a rule that never applies |
| CTA panel | **no block** — `h2` + `p` + two buttons as default content in a section styled `contained dark center narrow` | **default content (3C)** — the panel look is three existing section classes |

Result: zero new blocks. Note the third row is the one no block-keyed search
could have found (§3) — it took reading the page.

---

## 7. When there is genuinely nothing to find

Fresh site, empty content tree, or no page in any related family: discovery
returns nothing and **building new blocks is the correct outcome**. Say so
explicitly in the plan — "enumerated `query-index.json` / the DA source tree, no
same-family page exists" — so the reader can tell *discovery ran and found
nothing* from *discovery never ran*. The two justify very different plans.

---

## 8. Fingerprints are leads, not confirmation

Design-system traces that name a page type — a block variant documented
`(compare)`, a `.cmp` variant meaning "compare page", icons named after a
specific page — are evidence that **such a page exists**. The correct inference
runs toward discovery: *go find and read that page*.

Using them the other way — as reassurance that your new-block choice matches the
design system's conventions — inverts the signal, and converts the strongest
available hint that you're duplicating existing work into confidence that you
aren't.
