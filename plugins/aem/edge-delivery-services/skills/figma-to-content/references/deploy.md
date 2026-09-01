# Deploy mechanics — DA Source API, Code Sync, preview, verification

Commands for SKILL.md **Phase 5**. The *decisions* — what must hold before you
write, when you may publish, and the pre-publish gate — stay in SKILL.md; this
file is the how. **If a DA MCP server is available, prefer its tools** (da-auth
and da-content both defer to it); if you use it instead of these calls, apply the
same rule the `req` helper enforces: **assert the returned status, never assume
success.**

---

## Contents
- 1. Identities, refs, and the checked-request helper
- 2. content+code path only — block code must be live before the page renders
- 3. Both paths — media, overwrite guard, write, preview
- 4. Stage A verification commands (server-side, no browser)
- 5. Non-obvious rules (da-content / EDS)
- 6. Many pages
- 7. Publish to the live host — final, gated

## 1. Identities, refs, and the checked-request helper

```bash
# Two identities — keep them separate. DA (Document Authoring) and GitHub are the
# same org/repo in the standard EDS setup, but nothing guarantees it, so never
# assume one from the other. DA endpoints (admin.da.live/source, content.da.live,
# da.live/edit) use the DA pair; the render host and admin.hlx.page (code, preview,
# live) use the GitHub pair.
DA_ORG=<da-org>        # Document Authoring org
DA_REPO=<da-repo>      # Document Authoring repo/site
GH_OWNER=<gh-owner>    # GitHub owner
GH_REPO=<gh-repo>      # GitHub repo
# In the standard setup all four match: DA_ORG=GH_OWNER=<owner>, DA_REPO=GH_REPO=<repo>.
BRANCH=<branch>        # git deploy ref (usually main). For content+code this MUST be
                       # the branch the new-block code was pushed to and Code Sync built.
BRANCH_HOST=${BRANCH:?BRANCH is not set}
BRANCH_HOST=${BRANCH_HOST//\//-}   # host label: slashes → dashes ('feature/x' → 'feature-x').
                              # Used BOTH for the aem.page/aem.live hostname AND as the ref
                              # segment in every admin.hlx.page path (code/preview/live): that
                              # ref is a SINGLE path segment, so a slashed branch ('figma/x')
                              # splits it and 404s — pass the dashed label ('figma-x'), which is
                              # what AEM actually resolves. Only git itself (push/checkout) uses
                              # the literal slashed $BRANCH. For a slash-free branch the two
                              # forms are identical, so $BRANCH_HOST is always the safe choice
                              # for admin.hlx.page.
P=<path-without-extension>
TOKEN="$DA_TOKEN"      # from da-auth; 401 w/ empty body ⇒ expired, re-auth

# Fail fast if the branch host would be unresolvable (>63 chars won't resolve).
host="$BRANCH_HOST--$GH_REPO--$GH_OWNER"
[ "${#host}" -le 63 ] || { echo "❌ branch host '$host' is ${#host} chars (>63) — won't resolve; use a shorter branch/repo/org"; exit 1; }

# --- checked-request helper: every call asserts its status; a bare `curl -sS`
#     exits 0 on 401/403/409/5xx, so an unchecked curl silently "succeeds" on a
#     failed write. req <expected-codes> <curl-args…>: prints the body, retries a
#     few times on network/429/5xx, and aborts (non-zero) on any other mismatch.
#     Use it for every PUT/POST below. ---
req() {
  local expect="$1"; shift
  local attempt out code body
  for attempt in 1 2 3 4 5; do
    if out=$(curl -sS -w $'\n%{http_code}' "$@"); then code="${out##*$'\n'}"; else code="000"; fi
    body="${out%$'\n'*}"
    case ",$expect," in *",$code,"*) printf '%s' "$body"; return 0;; esac
    case "$code" in
      000|429|5??) sleep $((attempt * 2)); continue;;   # transient — bounded retry
      401)         echo "❌ 401 (empty body ⇒ token expired) — re-auth (da-auth) and retry" >&2; return 1;;
      *)           echo "❌ HTTP $code (expected $expect) — $*" >&2; return 1;;   # 4xx: do not retry
    esac
  done
  echo "❌ giving up after retries (last status $code) — $*" >&2; return 1
}
```

---

## 2. content+code path ONLY — block code must be LIVE before the page renders

Skip this whole section for content-only; the code is already deployed.

**Step 1 — commit and push.** The commit-scope rules are in SKILL.md Phase 5 and
the Guardrails (public web root; additive; disclose anything beyond block code +
icons). The sequence:

```bash
git add blocks/<new-block> icons/<name>.svg   # never `git add .` / `git add -A`
git status --short          # confirm NOTHING else is staged
git commit -m "feat: <new-block> block" || { echo "❌ commit failed — nothing pushed"; exit 1; }
git show --stat HEAD        # ← inspect the file list BEFORE it leaves the machine
git push origin "$BRANCH"
```

**Check the file list before pushing, not after.** If a commit turns out to be
over-broad and it has **not** been pushed, `git commit --amend` (or
`git rm --cached <file>` then amend) removes it cleanly. Once pushed, the only
options are live-with-it or rewriting the history of a shared branch — and a
follow-up "drop those files" commit **does not** remove them: they remain
retrievable at the earlier ref, and were served for the whole window in between.
That window is the cost of checking after the push instead of before.

Open a PR instead if the project protects `$BRANCH`; the branch that renders the
page must contain the block code. For a **3D variant**, the same discipline applies
and the diff must be additive-only (SKILL.md Phase 3D condition 1).

**Step 2 — Code Sync** builds automatically on push. Optionally force it; a non-2xx
here isn't fatal if the push already synced, so don't abort on it:

```bash
req 200,202 -X POST -H "Authorization: Bearer $TOKEN" \
  "https://admin.hlx.page/code/$GH_OWNER/$GH_REPO/$BRANCH_HOST/*" >/dev/null || true
```

**Step 3 — poll until the block's JS is live** on the branch host (bounded — don't
hang), then confirm its CSS too: a block whose JS loads but whose CSS 404s renders
unstyled.

```bash
BH="https://$BRANCH_HOST--$GH_REPO--$GH_OWNER.aem.page"
for i in $(seq 1 24); do
  code=$(curl -s -o /dev/null -w '%{http_code}' --compressed "$BH/blocks/<new-block>/<new-block>.js")
  [ "$code" = "200" ] && break
  [ "$i" = "24" ] && { echo "❌ block JS not live after ~2min — check push/branch/Code Sync"; exit 1; }
  sleep 5
done
# CSS gets the SAME bounded poll as the JS above — it can lag behind JS, and a
# single-shot check would abort a deploy that was seconds from being fine.
for i in $(seq 1 24); do
  csscode=$(curl -s -o /dev/null -m 15 -w '%{http_code}' --compressed "$BH/blocks/<new-block>/<new-block>.css")
  [ "$csscode" = "200" ] && break
  [ "$i" = "24" ] && { echo "❌ block CSS not live after ~2min ($csscode) — block renders unstyled; the Stage A gate box requires JS *and* CSS at 200, so failing here is the same verdict, reached sooner"; exit 1; }
  sleep 5
done
```

---

## 3. Both paths — media, overwrite guard, write, preview

**1) Upload referenced media FIRST** — every authored `<img>` must resolve at
**preview** time. Field name MUST be `data`. Detect the format from the **bytes**
and keep the `type=` MIME, the `<media-path>` extension, and the authored
`content.da.live` URL extension in agreement — detect the format from the image's
magic bytes, since the filename suffix is not authoritative (SKILL.md Phase 4). Skip images that use a stable external URL the preview can sideload.

```bash
req 200,201 -X PUT -H "Authorization: Bearer $TOKEN" \
  -F "data=@<local-image>;type=<image/mime>" \
  "https://admin.da.live/source/$DA_ORG/$DA_REPO/<media-path>" >/dev/null
# then reference it in the HTML as https://content.da.live/$DA_ORG/$DA_REPO/<media-path>
```

**2) Overwrite guard.** The Source-API PUT clobbers an existing page and still
returns 200, so the abort MUST be bound to an explicit decision, not to a warning.
Two facts come from the Phase 2.2 plan, recorded per path:

- `PLANNED_STATE` = `new` | `exists` — what the 2.2 existence check saw
- `OVERWRITE_OK` = `yes` — **only** when the user confirmed overwriting an existing
  page; unset/no otherwise

Re-GET now (auth is checked **before** existence, so a 401 empty body = expired
token, **not** "new"). Refuse to write if the state changed since planning or the
overwrite was never confirmed — a page can be created in the gap between planning
and this write.

```bash
exists=$(curl -sS -o /dev/null -w '%{http_code}' -H "Authorization: Bearer $TOKEN" \
  "https://admin.da.live/source/$DA_ORG/$DA_REPO/$P.html")
case "$exists" in
  404)  # absent now
    [ "$PLANNED_STATE" = "exists" ] && { echo "❌ plan expected an existing page but it's gone (404) — STOP and reconfirm"; exit 1; }
    ;;   # planned-new and still absent → proceed
  200)  # present now
    if [ "$OVERWRITE_OK" != "yes" ]; then
      echo "❌ $P.html exists in DA but overwrite was NOT confirmed in the plan — STOP and reconfirm with the user"; exit 1
    fi
    if [ "$PLANNED_STATE" != "exists" ]; then
      echo "❌ plan saw a NEW path (404) but it exists now (200) — a page was created since planning;"
      echo "   do NOT overwrite on the stale confirmation — STOP and reconfirm"; exit 1
    fi
    echo "→ overwriting $P.html (existing page, confirmed in the plan)"
    ;;
  401) echo "❌ 401 on existence check — token expired; re-auth (da-auth) and retry"; exit 1;;
  *)   echo "❌ unexpected $exists on existence check — resolve before writing"; exit 1;;
esac
```

**3) Write content** — multipart, field name MUST be `data`, type `text/html`:

```bash
req 200,201 -X PUT -H "Authorization: Bearer $TOKEN" \
  -F "data=@content/$P.html;type=text/html" \
  "https://admin.da.live/source/$DA_ORG/$DA_REPO/$P.html" >/dev/null   # 201 new / 200 update
```

**4) Preview** — separate and required. Path **without** `.html`; ref =
`$BRANCH_HOST` (the dashed label, not the slashed git branch).

```bash
req 200 -X POST -H "Authorization: Bearer $TOKEN" \
  "https://admin.hlx.page/preview/$GH_OWNER/$GH_REPO/$BRANCH_HOST/$P" >/dev/null
```

**The deploy sequence STOPS here.** Publishing to the live host is a separate,
final step that runs only after every pre-publish gate box passes **and** only if
the user asked to publish — never inline here, before verification (SKILL.md).

---

## 4. Stage A verification commands (server-side, no browser)

**Fetch once and assert the status — never pipe a bare `curl` into `grep` here.**
On a failed fetch `grep -c about:error` returns `0` and `grep -o '<img' | wc -l`
returns `0`, so a gate box that means "no broken images" **passes on a read that
never happened**. A false PASS on the pre-publish gate is worse than no check at
all: fail the box instead.

```bash
BASE="https://$BRANCH_HOST--$GH_REPO--$GH_OWNER.aem.page/$P.plain.html"
frag=$(mktemp); trap 'rm -f "$frag" "$page"' EXIT
code=$(curl -s --compressed -m 20 -o "$frag" -w '%{http_code}' "$BASE")
[ "$code" = "200" ] && [ -s "$frag" ] || {
  echo "❌ Stage A fetch failed ($code, $(wc -c <"$frag") bytes) — gate boxes FAIL; do not read as 'clean'"; exit 1; }
# A 200 with a NON-EMPTY body is still not proof you got the page: an auth wall or
# maintenance page is 200 and non-empty, and every grep below then reports a clean
# result for content you never fetched. Require a positive marker.
grep -q 'class="[a-z]' "$frag" || {
  echo "❌ fetched 200 but body carries no authored classes — likely an auth/error page; gate boxes FAIL"; exit 1; }

grep -c about:error "$frag"                              # expect 0 (no broken images)
grep -o '<img' "$frag" | wc -l                           # expect = authored image count
# digits are legal in EDS block names (just not first), so the token class must
# include 0-9 — matching the pattern used in existing-content-discovery.md §4:
grep -oE 'class="[a-z][a-z0-9 -]*"' "$frag" | sort -u    # every authored block class present
```

**Section count.** Count top-level sections, not every `<div>` (blocks and rows are
divs too, so a raw count runs several times high). **Do not count it on
`.plain.html`** — the fragment has no `<main>` (nor `<section>`) wrappers, so a
`<main> > div` count there is always 0. Count it on an artifact that has them:

```bash
# rendered (post-preview): EDS wraps each top-level section as <div class="section">
page=$(mktemp)
code=$(curl -s --compressed -m 20 -o "$page" -w '%{http_code}' \
  "https://$BRANCH_HOST--$GH_REPO--$GH_OWNER.aem.page/$P")
[ "$code" = "200" ] && [ -s "$page" ] || { echo "❌ render fetch failed ($code) — section-count box FAILS"; exit 1; }
grep -oE 'class="section[ "]' "$page" | wc -l    # expect = planned section count
```

or count the top-level `<main> > div` in the **local source** `content/$P.html`
before upload — it carries the Phase 4 `<body>/<main>` skeleton, so its direct
`<main>` children *are* the sections. Rich default content (heading / list / link
counts) *does* survive in `.plain.html`, so spot-check that there.

---

## 5. Non-obvious rules *(da-content / EDS)*

- multipart field name is exactly **`data`** — other names silently 200 with
  nothing written.
- verify media on the **render host** (`…aem.page/$P.plain.html` or the page),
  **not** by GETting `content.da.live/…` directly — a direct GET returns `401` by
  design even when the upload succeeded and the pipeline internalizes it.
- payload is a **body fragment**, not a full document.
- upload only **stages** the doc; the page is not reachable until **preview**.
  Referenced binaries/external image URLs must be reachable at **preview** time.
- branch host `<branch-host>--<gh-repo>--<gh-owner>` must be **≤ 63 chars** or it
  won't resolve (asserted by the `host` length check in §1).

---

## 6. Many pages

Drive `PUT → preview` with a concurrency pool + retry (`429`/`5xx`) rather than a
hand-rolled loop; publish stays a gated, post-verify step. An unattended multi-page
run can outlast a single ~1h token, so **refresh the token before long batches and
on any `401`-with-empty-body**, then resume — don't abort the whole run. Each page
still clears the pre-publish gate before it is eligible to publish; a gate failure
on one page blocks that page's publish, not the batch.

---

## 7. Publish to the live host — final, gated

Runs **only** when every pre-publish gate box positively passed **and** the user
asked to publish (SKILL.md "Publish to the live host").

```bash
# ref = $BRANCH_HOST (dashed label, not the slashed git branch).
req 200 -X POST -H "Authorization: Bearer $TOKEN" \
  "https://admin.hlx.page/live/$GH_OWNER/$GH_REPO/$BRANCH_HOST/$P" >/dev/null \
  || { echo "❌ publish failed — page stays preview-only"; exit 1; }
```
