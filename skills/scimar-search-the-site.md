---
name: Search scimar.ca and resolve results
description: Run a cross-content-type search against scimar.ca and turn each lightweight result pointer into the full object by branching on subtype.
api: openapi/scimar-content-openapi.yml
base_url: https://scimar.ca/wp-json/wp/v2
auth: none (anonymous read)
operations: [getSearch, getPostsById, getPagesById, getTypes, getTypesByType, getStatuses, getTaxonomies, getMediaById]
generated: '2026-08-26'
method: generated
---

# Search scimar.ca and resolve results

The site exposes one search endpoint that spans every searchable post type, and it returns
**pointers**, not full records. Resolving them is a two-step flow.

Everything here is anonymous. Do not send credentials.

## 1. Search

```
GET https://scimar.ca/wp-json/wp/v2/search?search=hepatalin&per_page=20&page=1
```

`getSearch`. Each result is a thin object:

```json
{"id": 123, "title": "What is hepatalin?", "url": "https://scimar.ca/news-stories/what-is-hepatalin/",
 "type": "post", "subtype": "post", "_links": {"self": [{"embeddable": true, "href": "..."}]}}
```

Pagination is the same as everywhere else: `X-WP-Total`, `X-WP-TotalPages` and the `Link` header.
`per_page` caps at 100. `context` here accepts only `view` and `embed` — not `edit`.

Narrow it with `type` (`post`) and `subtype` (`post`, `page`, or `any`).

## 2. Resolve each pointer

Branch on `subtype`, then fetch the real record:

| `subtype` | operation | path |
|---|---|---|
| `post` | `getPostsById` | `/wp/v2/posts/{id}` |
| `page` | `getPagesById` | `/wp/v2/pages/{id}` |

Or follow `_links.self[0].href` directly — it is already the correct absolute URL, which avoids
hard-coding the mapping.

Add `?_fields=...` on the resolve call; these records carry the full HTML body plus the All in
One SEO block and are large.

## 3. Know what is searchable before you search

```
GET /wp/v2/types
GET /wp/v2/taxonomies
GET /wp/v2/statuses
```

`getTypes` / `getTaxonomies` / `getStatuses`. On scimar.ca `types` returns 11 entries, all of
them WordPress core or block-editor internals — `post`, `page`, `attachment`, `nav_menu_item`,
`wp_block`, `wp_template`, `wp_template_part`, `wp_global_styles`, `wp_navigation`,
`wp_font_family`, `wp_font_face`. Only `post` and `page` are meaningful content.

Call this first rather than assuming. It is the cheapest way to learn that a site has no custom
types, which is exactly the trap on this deployment: the sitemap advertises `team` and
`member-category` sections that have no REST route.

## Two routes that will not behave as you expect

- `GET /wp/v2/settings` — returns **401** `rest_forbidden` anonymously. Do not retry it.
- `GET /wp/v2/users` — does **not** return a JSON error. It **302-redirects**, so a client that
  follows redirects gets HTML. There is no anonymous author directory on this site; read the
  `/team/` page instead.

## Errors

See `errors/scimar-problem-types.yml`. `400 rest_invalid_param` on a bad `per_page`/`context`;
`404 rest_no_route` on an unregistered path; HTML bodies on non-API 404s.
