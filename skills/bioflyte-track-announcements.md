---
name: Track BioFlyte announcements
description: Pull BioFlyte's press releases and news from its WordPress content API, filter by
  category, and page through the full 36-item history without scraping HTML.
api: openapi/bioflyte-content-openapi.yml
operations: [getPosts, getPostsById, getCategories]
---

# Track BioFlyte announcements

BioFlyte publishes every press release and news item through the anonymously readable WordPress
REST API on its marketing site. Use it instead of scraping `/category/press-release/`.

**Base URL:** `https://www.bioflyte.com/wp-json/wp/v2`
**Auth:** none required for reads. Do not send credentials.

## Steps

1. **Resolve the taxonomy first** — call `getCategories`
   (`GET /categories?per_page=100&_fields=id,slug,name,count`). At harvest the terms were
   `15 News (18)`, `16 Press Release (18)`, `1 Uncategorized (0)`. Ids are site-specific; read
   them, never hardcode them.

2. **List announcements** — call `getPosts`
   (`GET /posts?per_page=100&orderby=date&order=desc&categories=16`).
   - `per_page` is capped at **100**; `page` starts at 1.
   - Read `X-WP-Total` and `X-WP-TotalPages` from the response headers to know how far to page,
     or follow the RFC 8288 `Link: <…>; rel="next"` header.
   - Use `_fields=id,date,modified,slug,link,title,excerpt,categories` to keep payloads small.
   - Use `after=<ISO8601>` (or `modified_after`) for incremental pulls — this is the correct way
     to poll, since there is no webhook or event feed.

3. **Fetch one item in full** — call `getPostsById` (`GET /posts/{id}`). `title.rendered`,
   `excerpt.rendered` and `content.rendered` are HTML strings, not plain text; the `link` field
   is the canonical public URL.

## Rules

- **Cache aggressively but expect staleness.** Responses carry
  `cache-control: max-age=2419200, must-revalidate` (28 days) from WP Engine behind Cloudflare,
  so a brand-new release can be served stale from the edge. If freshness matters, compare
  `X-WP-Total` between polls rather than trusting the cache window.
- **Respect `Crawl-delay: 10`** from `https://www.bioflyte.com/robots.txt`. There are no
  rate-limit headers, so back off on your own schedule.
- **Errors do not use RFC 9457.** Failures return
  `{"code": "...", "message": "...", "data": {"status": <int>}}`. `rest_invalid_param` (400)
  means you exceeded `per_page=100` or sent an out-of-enum `orderby`/`status`;
  `rest_post_invalid_id` (404) means the id is gone or unpublished. See
  `errors/bioflyte-problem-types.yml`.
- **Never call a write operation.** Every POST/PUT/PATCH/DELETE on this API needs a WordPress
  Application Password. There is no reason for an agent to hold one.
