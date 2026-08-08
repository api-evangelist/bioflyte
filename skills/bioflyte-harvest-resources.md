---
name: Harvest BioFlyte whitepapers and blog posts
description: Pull BioFlyte's Resources library — blog posts and downloadable whitepapers —
  through the `resource` custom post type, resolve the attached PDF from the media library, and
  keep the taxonomy split intact.
api: openapi/bioflyte-content-openapi.yml
operations: [getResourcesCategory, getResource, getResourceById, getMediaById, getMedia]
---

# Harvest BioFlyte whitepapers and blog posts

BioFlyte's long-form content lives in a custom post type called `resource`, not in `posts`. It
is the most substantive non-press dataset on the API — and the only place the biothreat
whitepapers are available as structured records.

**Base URL:** `https://www.bioflyte.com/wp-json/wp/v2`
**Auth:** none required for reads.

## Steps

1. **Resolve the split** — call `getResourcesCategory`
   (`GET /resources-category?per_page=100&_fields=id,slug,name,count`). At harvest:
   `30 Blog (4)`, `31 Whitepapers (2)`. Read the ids; they are site-specific.

2. **List one class of resource** — call `getResource` with the taxonomy filter:
   `GET /resource?resources-category=31&per_page=100&_fields=id,date,slug,link,title,featured_media,resources-category`.
   Omit the filter to get all six. Paging, `X-WP-Total` and the `Link` header behave exactly as
   on `getPosts`.

3. **Fetch the record** — call `getResourceById` (`GET /resource/{id}`). The public permalink in
   `link` follows `/resource/{category-slug}/{slug}/`, so the taxonomy term is part of the URL —
   changing a resource's category changes its public URL.

4. **Resolve the attached asset** — a resource's `featured_media` is an attachment id (for
   example `281058` on the anthrax airport whitepaper). Call `getMediaById`
   (`GET /media/{id}`) and read `source_url` for the direct file. `featured_media: 0` means no
   asset is attached.

5. **Cross-reference by search** — `getSearch` (`GET /search?search=<term>`) spans posts, pages
   and resources in one call and returns `subtype` plus `_links.self` pointing at the concrete
   collection route, so you can resolve a hit without knowing its type in advance.

## Rules

- **Do not assume `posts` contains the whitepapers.** `getPosts` returns press releases only.
  A harvest that reads `posts` and stops will silently miss every whitepaper.
- **`title.rendered` and `content.rendered` are HTML.** Strip or render; do not treat as text.
- **The media library is 374 items.** Page it with `per_page=100`; do not request it unfiltered
  if you only need one asset — resolve `featured_media` by id instead.
- **Errors** use the WordPress envelope, not RFC 9457 — see `errors/bioflyte-problem-types.yml`.
- **Reads only.** Every write on this API requires a WordPress Application Password.
