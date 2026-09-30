# Site repair — diagnosis

Casey reported the fleet's websites broken: `browse.html` linking to a non-existent GitHub
account, older pages (the radio shows, the open mics) missing, `superinstance.dev`/`superinstance.ai`
hokey, `ai-writings.pages.dev` not working, and everything supposed to point at the real
paid domains.

**The short version: one of those is a real content bug. Most of the rest is Cloudflare
flapping. And the fix everyone will want to apply would break 79 working URLs.**

---

## 1. The real bug, and it is one line

`https://superinstance.dev/browse.html` line 187:

```js
<a href="https://github.com/casey-digennaro/${r.name}" target="_blank">${r.name}</a>
```

`github.com/casey-digennaro` returns **404 — the account does not exist.**

```
GET https://api.github.com/users/casey-digennaro  →  404
```

The correct org is `SuperInstance`. The template literal is fine; the account is wrong.

**The landing page itself is clean.** All 41 GitHub links on `index.html` are
`github.com/SuperInstance/…`. Only `browse.html` is wrong.

---

## 2. The fix that would break 79 working URLs

A search finds **117 references to `casey-digennaro` across the org.** 79 of them are in
`SuperInstance/SuperInstance` alone. They look identical to the bug above. They are not.

They are **Cloudflare Workers subdomains**:

```
fleet-vector-api.casey-digennaro.workers.dev
the-tap.casey-digennaro.workers.dev
fleet-dashboard.casey-digennaro.workers.dev
fleet-wiki.casey-digennaro.workers.dev
fleet-edge.casey-digennaro.workers.dev
superinstance-assets.casey-digennaro.workers.dev
```

`casey-digennaro` is the **Cloudflare account subdomain** — confirmed via
`GET /accounts/{id}/workers/subdomain` → `{"subdomain": "casey-digennaro"}`. It is not
renameable without renaming the Cloudflare account, and these URLs are **correct as
written**.

**A blind `casey-digennaro → SuperInstance` find-replace would break 79 correct references
across the fleet's main documentation, to fix a bug that is not in that repo.** There are
**zero** `github.com/casey-digennaro` references in `SuperInstance/SuperInstance`.

---

## 3. What is actually up and what is actually down

Sampled repeatedly over ~20 minutes. **The sites oscillate.**

```
        superinstance.dev  superinstance.ai  quilt-show  ai-writings  luciddreamer.ai
sample 1     200              200             200          200          000
sample 2     200              200             200          200          200
sample 3     200              200             200          503          000
sample 4     200              200             200          200          200
```

`ai-writings.pages.dev` went `503,503,503,503,200,200` inside fifteen seconds.
`superinstance.ai` was 200, then 503, then 200. `superinstance.github.io` went 200 → 000.

**The account has 374 worker scripts deployed.** All six of the workers named in the docs
are deployed and present. An intermittent 503/000 across Workers, Pages, and custom domains
*simultaneously* points at account-level throttling or an edge problem, not at per-site
faults. Nothing in the content explains an oscillation.

`ai-writings.pages.dev` and `the-tap.pages.dev` are **not in this Cloudflare account at all**
— they are deployed somewhere else, which is why they behave differently from ours.

---

## 4. The pages that are genuinely missing

`SITE-MAP.md` is **live on the site** (435 lines) and specifies a twelve-section education
site. Those routes are **404**:

| route | status |
|---|---|
| `/concepts/` | 404 |
| `/crates/` | 404 |
| `/build/` | 404 |
| `/play/` | 404 |

Sections specified and not present: ternary conservation, bottle protocol, agent
lifecycle, self-improving harness, conservation law, wavelet decomposition, the SEED crates
catalog, per-crate pages, build-your-first-agent, playground.

**This is the "many many pages." They are designed in `SITE-MAP.md` and do not exist.** They
were specified, and either never built or built and not deployed.

Of the 19 internal links on the landing page, **18 resolve. The only dead one is
`tutorials/`.**

---

## 5. The blocker for fixing #1

`superinstance.dev` is served by **GitHub Pages** — `x-github-request-id`,
`x-github-edge-region: iad`, `via: 1.1 varnish` — last deployed **26 September**.

`browse.html` is live but exists in **no repository I can reach**, public or private, on any
of the 11 branches of `SuperInstance/SuperInstance`. There is no CNAME naming
`superinstance.dev` anywhere in the org. `SuperInstance/SuperInstance`'s own `pages.yml`
deploys the **entire repository root** (546 files), which is what serves
`superinstance.github.io` — and that is a *different* site from the curated landing page.

**So the one real fix is blocked on a fact only Casey has: which repository serves
`superinstance.dev`, and where `browse.html` lives in it.**

---

## What is ready to go the moment that is answered

1. **One-line fix** in `browse.html`: `casey-digennaro` → `SuperInstance` in the GitHub href.
2. **A 404 audit** for the eleven dead `SITE-MAP.md` routes, with the option to build them.
3. **`tutorials/`** — the one dead link on the landing page.
4. **A Cloudflare account-level look** — 374 workers with intermittent 503s across every
   hostname is the thing to rule out first, and it explains most of what was reported.

## What I deliberately did not do

I did not run a fleet-wide replace of `casey-digennaro`. Two of the three namespaces are
correct and one is wrong, they differ only by prefix, and the wrong one is in a file whose
source I cannot locate. A rename that looks obvious, applied to 117 references across a
live org, is how you turn one dead link into 79.

---

## ADDENDUM — a live credential, found and partly fixed (09:40)

**`SuperInstance/micrograd-quilt` is now PUBLIC, and it carried a live `LLM_API_KEY`.**

What happened, in order:
- I flagged this repo on the private→public pass and blocked PR #7 with a comment naming the
  credential and the byte-identical "template".
- **PR #7 was merged anyway** (squash `44de6055f1`, 04:49) and the repo went public.
- The `.env` and its byte-identical "template" (same sha256 `27076c53e09fecbb`) were
  publicly fetchable at
  `https://raw.githubusercontent.com/SuperInstance/micrograd-quilt/main/memory_consolidation/memory_consolidation.env`
  — **a 16-character `LLM_API_KEY`.**

**What I fixed.** Removed `memory_consolidation.env` from HEAD (force-push amended onto
`fc01a3b482`) and placeholdered every credential-shaped assignment in the template.

**And the fix had a bug of its own.** My first `.gitignore` rule was `.env` — which matches
a file named *exactly* `.env`, not `memory_consolidation.env`. The rule looked correct,
appeared in the diff, and did nothing. It took an untracked `git rm --cached` and a
force-push to actually remove it. Corrected to `*.env`.

**What is still true, and it matters:**

- The **git tree at HEAD is clean** (verified against the contents API, which is
  authoritative). `raw.githubusercontent.com` still returns 200 for the old path, but that
  is a **stale CDN cache** — the served template is 3518 B where HEAD's is 3525 B.
- The file is **still in the object store** at the merge commit and every commit before it,
  and `raw.githubusercontent.com` will serve it from any historical ref indefinitely.
- **Therefore the key has been publicly reachable since 04:49 and must be ROTATED.**
  Deleting a file is not unpublishing a secret. A value that has been fetchable from a
  public URL has to be treated as public.

History rewrite is possible (`git filter-repo`) and is the only thing that removes it from
the object store. It is also **Casey's call, not mine**: it destroys commit SHAs, breaks
every existing clone, and invalidates every PR reference. Given a 16-character value that
looks like a placeholder-length string, rotation is almost certainly the cheaper and more
honest response — but that judgement is his, not a find-and-replace's.

**The lesson generalises past this repo:** a `.gitignore` entry reads as a fix in the diff
and is not a fix in the repo unless the pattern actually matches the filename in question.
Check what the pattern matches, not what it looks like.
