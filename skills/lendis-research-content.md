---
name: Research Lendis content
description: Search and read Lendis' published content - Magazin blog posts, Ratgeber buyer's
  guides, Wiki glossary entries, case studies and catalogs - through the public WordPress REST
  API at www.lendis.io/wp-json.
api: openapi/lendis-content-openapi.yml
operations:
  - searchContent
  - listPosts
  - getPostsItem
  - listRatgeber
  - getRatgeberItem
  - listWiki
  - getWikiItem
  - listCaseStudy
  - getCaseStudyItem
generated: '2026-07-19'
method: generated
---

# Research Lendis content

Lendis GmbH rents IT hardware to German SMBs under a Device-as-a-Service model. Its marketing
site runs on WordPress and exposes a **public, read-only** REST API. Use it to answer questions
about Lendis' offering, its published guidance, and its customer stories.

**Base URL:** `https://www.lendis.io/wp-json`
**Auth:** none. Every operation below is an anonymous GET.
**Language:** all content and all error messages are German.

## When to use this skill

- A question about what Lendis offers, how DaaS works, or IT-procurement terminology.
- You need Lendis' own words rather than a summary — quotes, guide text, case-study detail.

Do **not** use this for anything about a customer's own devices, orders or contracts. That lives
in LendisOS behind `api.lendis.io`, which is a private AWS API Gateway with no public routes and
returns `403 Missing Authentication Token` to everyone.

## Steps

### 1. Start with search, not with a listing

`searchContent` is the cheapest entry point.

```
GET /wp/v2/search?search=Device%20as%20a%20Service&per_page=20
```

Each result is a stub: `id`, `title`, `url`, `type`, `subtype`. **`subtype` tells you which
collection to fetch the full object from** — `post` → `/wp/v2/posts/{id}`, `ratgeber` →
`/wp/v2/ratgeber/{id}`, `wiki` → `/wp/v2/wiki/{id}`, `case-study` → `/wp/v2/case-study/{id}`.

Search terms must be German. Try `Mietrate`, `Lifecycle`, `Beschaffung`, `Offboarding`,
`Leasing`, `Onboarding`.

### 2. Fetch the full object

Use the matching `get…Item` operation with the numeric `id`:

- `getPostsItem` — a Magazin blog post
- `getRatgeberItem` — a Ratgeber buyer's guide
- `getWikiItem` — a Wiki glossary entry
- `getCaseStudyItem` — a customer case study

Body text arrives as rendered HTML under `content.rendered`. Strip the tags before quoting.

### 3. Or browse a collection directly

When you want breadth rather than a specific hit:

- `listWiki` — the glossary. The best surface for defining a term.
- `listRatgeber` — the buyer's guides. Longest-form practical guidance.
- `listPosts` — the Magazin, newest first.
- `listCaseStudy` — named customers (KoRo, Bilthouse Gruppe and others).

### 4. Trim the payload

Full objects are large. Ask only for what you need:

```
GET /wp/v2/wiki?per_page=100&_fields=id,slug,title,link
```

`_fields` works on collections and single items. Use `_embed` only when you actually need the
author, featured image or taxonomy terms inlined.

## Rules

**Pagination.** `page` (1-based) and `per_page` (default 10, **max 100**). Read `X-WP-Total` and
`X-WP-TotalPages` from the response headers, or follow the `Link` header's `rel="next"` until it
is absent. Requesting a page past the end returns `400 rest_post_invalid_page_number`.

**Errors.** The envelope is `{code, message, data.status}` — **not** RFC 9457 problem+json.
Branch on `code`, never on `message`, which is German prose. The two you will actually hit:
`rest_post_invalid_id` (404, the ID does not exist or is not published) and `rest_forbidden`
(401, you touched a privileged route). Full list: `errors/lendis-problem-types.yml`.

**Stay on the read surface.** `/wp/v2/settings`, `/wp/v2/plugins` and every write method return
`401 rest_forbidden` to anonymous callers. Do not attempt writes — you have no credentials, and
this is a live production site.

**Rate limits.** None are documented and no `RateLimit` headers are returned, but the site sits
behind Cloudflare. Self-throttle, keep concurrency low, and back off on `429` or `5xx`.

**No idempotency contract.** Irrelevant here since every operation is a safe GET, but worth
knowing: the WordPress REST API defines no `Idempotency-Key` mechanism.

**Attribution.** This content is Lendis' copyrighted marketing material. Quote sparingly and
link back to the object's `link` field.
