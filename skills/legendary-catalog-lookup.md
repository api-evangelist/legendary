---
name: Look up Legendary Entertainment titles and news
description: >-
  Query the public WordPress REST API on legendary.com to find Legendary Pictures
  films, Legendary Television series, Legendary Comics titles, trailers and news
  posts, and resolve their key art.
api: openapi/legendary-wp-rest-openapi.yml
operations:
  - getSearch
  - getPosts
  - getPostsById
  - getFilm
  - getFilmById
  - getTelevision
  - getComics
  - getComicsById
  - getTrailers
  - getMediaById
  - getTypes
---

# Look up Legendary Entertainment titles and news

Legendary Entertainment's marketing site exposes a public, read-only WordPress REST
API. Use it to answer questions about Legendary's slate (films, TV, comics) and its
press announcements.

**Base URL:** `https://www.legendary.com/wp-json/wp/v2`

## Authentication

None required for reads. Every operation in this skill is public. Do **not** attempt
write operations — they require WordPress Application Password credentials that
Legendary does not issue to third parties.

## Steps

1. **Pick the right content type.** Call `getTypes` if unsure. Legendary separates its
   slate into distinct types rather than one "titles" collection:
   - `getFilm` — Legendary Pictures films (~64 records)
   - `getTelevision` — Legendary Television series
   - `getComics` — Legendary Comics titles (~52 records)
   - `getTrailers` — trailer entries
   - `getPosts` — news / press posts (~295 records)

2. **Search when the user names a title.** Call `getSearch` with `search=<title>` for a
   cross-type lookup, then fetch the full record from the matching type-specific
   operation (`getFilmById`, `getComicsById`, `getPostsById`).

3. **Page correctly.** Collections default to 10 records. Set `per_page` (max 100) and
   `page`. Read `X-WP-Total` and `X-WP-TotalPages` from the response headers to know
   when to stop, or follow the `Link` header's `rel="next"`. Never assume one page is
   the whole slate.

4. **Trim the payload.** These records are large (they carry rendered HTML plus
   `aioseo_*` SEO blobs). Pass `_fields=id,slug,title,link,date,excerpt,featured_media`
   to keep responses small.

5. **Resolve artwork in one call.** Add `_embed` to a request instead of a second
   round-trip: the featured image appears under
   `_embedded["wp:featuredmedia"][0].source_url`. Otherwise call `getMediaById` with the
   record's `featured_media` id. A `featured_media` of `0` means no image.

6. **Render text carefully.** `title.rendered`, `excerpt.rendered` and
   `content.rendered` contain HTML with entities. Strip or unescape before displaying.

## Sorting and filtering

- Newest news: `getPosts` with `orderby=date&order=desc` (the default).
- By date window: `after` / `before` accept ISO 8601 datetimes.
- By taxonomy: `categories` / `tags` accept term ids from `getCategories` / `getTags`.

## Error handling

Errors are **not** RFC 9457. The envelope is `{"code", "message", "data": {"status"}}`:

- `404 rest_post_invalid_id` — the id does not exist or is not published. Re-resolve
  via `getSearch` rather than guessing ids.
- `404 rest_no_route` — wrong path or method; re-check against `/wp-json/wp/v2`.
- `400 rest_invalid_param` — a parameter failed validation (commonly `per_page` > 100).
- `401 rest_unauthorized` — you attempted an authenticated operation; stop.

## Cautions

- There is no idempotency contract and no rate-limit headers. The site sits behind WP
  Engine edge caching, so back off on 4xx/5xx bursts rather than retrying tightly.
- Record counts drift as Legendary publishes; never hard-code totals.
- This is the platform API behind a marketing site, not a supported developer product.
  Treat availability and shape as unstable.
