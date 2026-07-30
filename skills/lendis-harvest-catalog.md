---
name: Harvest the Lendis site catalog
description: Enumerate everything the Lendis WordPress REST API exposes - content types,
  taxonomies, terms, media and downloadable product catalogs - by walking the discovery
  endpoints first, so a crawl never guesses at routes.
api: openapi/lendis-content-openapi.yml
operations:
  - getNamespaceIndex
  - listTypes
  - listTaxonomies
  - listCategories
  - listTags
  - listLetter
  - listRatgeberKategorien
  - listKataloge
  - getKatalogeItem
  - listMedia
generated: '2026-07-19'
method: generated
---

# Harvest the Lendis site catalog

Enumerate the full public content surface of `www.lendis.io` without guessing routes. The API
describes itself; start there.

**Base URL:** `https://www.lendis.io/wp-json`
**Auth:** none.

## Steps

### 1. Read the route index

```
GET /wp/v2
```

`getNamespaceIndex` returns every route in the `wp/v2` namespace with its supported methods and
full argument schemas. This is the authoritative list — the repo's OpenAPI was derived from it.
If a route you want is not in here, it does not exist.

`GET /wp-json/` (one level up) lists all namespaces. Besides `wp/v2` this deployment registers
plugin namespaces (`jet-engine/v1`, `rankmath/v1`, `weglot/v2` and others). Those are third-party
plugin surfaces, not Lendis content — ignore them unless you have a specific reason.

### 2. Discover the content types

```
GET /wp/v2/types
```

`listTypes` returns a map keyed by type name. The field that matters is **`rest_base`** — that
is the path segment for the collection. Five of these are Lendis-specific rather than WordPress
core:

| type | rest_base | what it is |
|---|---|---|
| `wiki` | `wiki` | glossary entries, grouped by the `letter` taxonomy |
| `ratgeber` | `ratgeber` | buyer's guides, grouped by `ratgeber-kategorien` |
| `case-study` | `case-study` | customer case studies |
| `kataloge` | `kataloge` | downloadable product catalogs |
| `testimonial` | `testimonial` | short customer quotes |

Each type also carries its `taxonomies` array — that is how you know which term filters apply.

### 3. Discover the taxonomies, then their terms

```
GET /wp/v2/taxonomies
```

Then walk each one: `listCategories`, `listTags`, `listLetter`, `listRatgeberKategorien`. Terms
carry a `count` field — use it to skip empty terms and to sanity-check your crawl totals.

### 4. Walk each collection to exhaustion

For each `rest_base`, page through with the maximum page size and only the fields you need:

```
GET /wp/v2/{rest_base}?per_page=100&page=1&_fields=id,slug,title,link,modified
```

Stop when the `Link` header no longer carries `rel="next"`, or when `page` exceeds the
`X-WP-TotalPages` you read from the first response. Do not compute the page count yourself from
a guessed page size — read the header.

Keep `modified` in your harvest. On a re-run it lets you fetch only what changed.

### 5. Resolve the catalogs and media

`listKataloge` gives you the catalog objects; `getKatalogeItem` gives you one in full. The actual
PDF is a media item — follow `featured_media` to `/wp/v2/media/{id}` (`listMedia` for the full
library) and read **`source_url`** for the direct file URL, plus `mime_type` and
`media_details` for size.

## Rules

**Order matters.** Discovery (`/wp/v2` → `/types` → `/taxonomies`) before enumeration. Routes and
custom post types on this site come from plugins and can change; a hardcoded route list will rot.

**Pagination bounds.** `per_page` is capped at **100**. Exceeding it returns `400
rest_invalid_param`. Paging past the end returns `400 rest_post_invalid_page_number`.

**`_fields` is not optional at scale.** Full objects embed rendered HTML for every entry. A
harvest without `_fields` will pull megabytes you then discard.

**Errors.** `{code, message, data.status}`, not RFC 9457. Branch on `code`. See
`errors/lendis-problem-types.yml`.

**Do not crawl the privileged routes.** `/wp/v2/settings` and `/wp/v2/plugins` return `401
rest_forbidden` to anonymous callers. Repeatedly probing them looks like an attack against a
Cloudflare-fronted production site.

**Throttle.** No rate limits are published and no `RateLimit` headers come back. Keep concurrency
to a small number of requests in flight, pause between pages, and back off on `429` or `5xx`.
