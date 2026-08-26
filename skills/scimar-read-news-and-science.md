---
name: Read the Scimar news archive and science pages
description: Page through Scimar's published news stories and its Science & Products pages as JSON, resolving categories, authors and featured images in one request.
api: openapi/scimar-content-openapi.yml
base_url: https://scimar.ca/wp-json/wp/v2
auth: none (anonymous read)
operations: [getPosts, getPostsById, getPages, getPagesById, getCategories, getCategoriesById, getTags, getMedia, getMediaById]
generated: '2026-08-26'
method: generated
---

# Read the Scimar news archive and science pages

Scimar keeps its writing in two standard WordPress collections — `posts` (46 news stories, the
hepatalin explainers, investor updates and team profiles) and `pages` (19 marketing and science
pages: About, Science & Products, The Science, Product Pipeline, Team, Community, Videos, Media,
Testimonials, Raise History, Legal, Privacy, Accessibility, Contact).

There are **no custom post types** on this site. `GET /wp/v2/types` returns only WordPress core
and block-editor internals. The "Team" and "Member Category" sections that appear in
`sitemap.xml` and in `llms.txt` are **not** exposed through `wp/v2` — if you need those, read the
`/team/` page or the sitemap, not the REST API. Do not go looking for a `team` endpoint; it is
not registered.

Everything here is anonymous. Do not send credentials.

## 1. List news stories, newest first

```
GET https://scimar.ca/wp-json/wp/v2/posts?per_page=20&page=1&orderby=date&order=desc
```

`getPosts`. Read pagination from the response headers, never from the body:

- `X-WP-Total` — total stories (46 at harvest)
- `X-WP-TotalPages` — pages at the current `per_page`
- `Link: <...>; rel="next"` — the RFC 8288 next page

`per_page` is capped at **100**; exceeding it returns `400 rest_invalid_param`.

## 2. Trim the payload

Post bodies are full HTML and this site injects a large All in One SEO block on every record
(`aioseo_head`, `aioseo_head_json`, `aioseo_meta_data`, `aioseo_breadcrumb`,
`aioseo_breadcrumb_json`, `aioseo_notices`). Ask for only what you need:

```
GET /wp/v2/posts?per_page=20&_fields=id,date,slug,link,title,excerpt,categories,tags
```

Titles and excerpts come back as `{"rendered": "<html string>"}` and are HTML-escaped
(`Scimar&#8217;s`). Unescape before displaying.

## 3. Resolve relations in one request

```
GET /wp/v2/posts?per_page=10&_embed
```

`_embed` inlines the relations the record advertises in `_links` — `author`,
`wp:featuredmedia`, `wp:term` (categories and tags) and `replies` — into `_embedded`. Without
it you would need a follow-up call per relation.

## 4. Read the science and pipeline pages

```
GET /wp/v2/pages?per_page=100&_fields=id,slug,link,title
GET /wp/v2/pages/{id}?_fields=title,content,link
```

`getPages` / `getPagesById`. The pipeline content lives at slug `product-pipeline`, the science
explainer at `the-science`, and the parent at `science-products`.

## 5. Read taxonomies

```
GET /wp/v2/categories?per_page=100
GET /wp/v2/tags?per_page=100
```

`getCategories` / `getTags`. Both accept `search`, `slug`, `include`, `exclude`, `order` and
`orderby`, and both are paginated the same way as posts.

## Errors you will actually hit

See `errors/scimar-problem-types.yml`. The envelope is
`{"code":..., "message":..., "data":{"status":...}}` — **not** RFC 9457 problem+json.

- `400 rest_invalid_param` — `per_page` over 100, `page` under 1, or a bad `orderby`/`status` enum.
- `404 rest_post_invalid_id` — the id does not exist or is not public.
- `404 rest_no_route` — the path is not registered. Enumerate from `https://scimar.ca/wp-json/`.
- A `404` on a non-API path returns an **HTML** body, not the error envelope. Check `Content-Type`.

## What this content is, and is not

This is a corporate website's CMS API. It returns Scimar's public writing about hepatalin, its
product pipeline and its community programs. It does **not** return clinical data, trial results,
diagnostic measurements or anything from the NuPa products. No such API exists.

The corpus is also static: the newest post sitemap `lastmod` is 2023-11-02 and the newest page
`lastmod` is 2024-06-19. Treat it as an archive, not a feed.
