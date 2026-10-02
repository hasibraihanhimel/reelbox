# Reelstream project handoff summary

Use this document as context when continuing the project in a new chat.

## 1. Project identity

- **Project name:** Reelstream
- **Project directory:** `/home/ubuntu/reelstream`
- **Stack:** FastAPI/Python backend + vanilla HTML/CSS/JavaScript frontend
- **Main backend file:** `app.py`
- **Frontend files:** `static/index.html`, `static/styles.css`, `static/app.js`
- **Container file:** `Dockerfile`
- **Dependencies:** `requirements.txt`
- **Uploaded source foundation:** Moviebox API project extracted from `Moviebox-API-main.zip`
- **Current canonical Git branch:** `main`
- **Latest project commit at the time of this summary:** `0f499e7 Fix media timeout request construction`

## 2. Product purpose

Reelstream is a dark cinematic movie and series catalog. It reads catalog, title-detail, playback, and subtitle information from the supplied Moviebox upstream API foundation. It does not invent movies, ratings, streams, or availability. If the upstream service has no stream for a title, the UI explains that the source is unavailable.

The app is intended to provide:

- Movie browsing
- Series browsing
- Animation browsing
- Search and autocomplete
- Movie and series detail pages
- Movie playback
- Series episode playback
- Quality selection
- Subtitle language selection
- Watchlist stored in browser local storage
- New-release/update feed
- Catalog refresh support

## 3. Frontend features currently present

### Navigation

The header includes:

- Reelstream brand/logo
- Home
- Movies
- Series
- Animation
- Rankings
- My list/watchlist
- Global search field
- Search button
- Catalog refresh button

### Catalog browsing

The app has separate category routes:

- `/browse/movies`
- `/browse/series`
- `/browse/animation`
- `/rankings`
- `/watchlist`

Catalog cards display upstream-provided metadata such as:

- Poster
- Title
- Year
- Rating when available
- Genre
- Release information
- Category/type information

Catalog lists support pagination where the upstream response exposes continuation information.

### Category separation

The category bug was fixed. The backend now uses typed filtering/fallback logic so that:

- Movies use `subjectType == 1`.
- Series use `subjectType == 2`.
- Animation uses anime/animation classification from the upstream taxonomy rather than simply showing generic movie results.

The frontend has category tabs/cards and responsive catalog-grid styling.

### Search

Search routes:

- `/api/search?q=...&page=1`
- `/api/search/suggest?q=...`

The frontend provides:

- Debounced search behavior
- Search result rendering
- Search pagination
- Empty-result state
- Search suggestions/autocomplete
- Navigation from suggestions to title pages

The backend caches search responses briefly to reduce upstream latency and repeated requests.

### Detail pages

Detail route:

- `/title/{slug}`

A detail page includes:

- Poster and title
- Type label such as FEATURE or SERIES
- Year
- Rating
- Duration when available
- Country when available
- Synopsis/description
- Genres/tags
- Add-to-watchlist button
- Movie Play button for feature films
- Series season tabs and episode buttons
- Playback area
- Quality selector when multiple sources exist
- Subtitle selector when caption tracks exist

### Watchlist

The watchlist is a browser-side feature using local storage. It does not currently require login or a database.

## 4. Playback implementation

### Stream lookup

Backend route:

```text
GET /api/stream/{subject_id}?detail_path=...&se=...&ep=...
```

The backend calls the upstream player API and normalizes sources into objects containing fields such as:

- `id`
- `resolution`
- `format`
- `url`
- `size`
- `duration`
- `codec`

The response also preserves:

- `has_resource`
- `hls`
- `dash`
- `free_episodes`
- `limited`
- `note`

### Critical movie parameter behavior

This was an important bug fix:

- **Movies require:** `se=0&ep=0`
- **Series episodes normally use:** `se>=1&ep>=1`

The movie detail page now calls:

```javascript
loadStream(0, 0)
```

Previously it incorrectly called `loadStream(0, 1)`, which made the upstream return “No stream found for this episode” even when the movie had valid sources.

### Movie playback

Movies now:

- Show a `Play movie` button.
- Automatically call the movie stream lookup with `se=0&ep=0`.
- Render the video when a valid source is returned.
- Attempt autoplay.
- Fall back to muted autoplay if browser audio autoplay is blocked.
- Display a loading spinner while connecting/buffering.
- Display a useful unavailable message when no source is returned.

### Series playback

Series pages:

- Show seasons.
- Show episode buttons.
- Call stream lookup with the selected season and episode.
- Automatically attempt playback after the episode source loads.
- Show episode number in the player label.

### Quality selection

Sources are sorted by resolution. The app retains every returned source.

The default selection was changed from highest quality to a faster quality preference:

- Prefer 480p when available.
- Otherwise use the lowest available source.
- 720p and 1080p remain available in the quality selector.

This reduces initial bandwidth and buffering for large MP4 files.

When switching quality:

- The current mute state is preserved.
- The current playback time is saved.
- The selected source is loaded.
- The player seeks back to the previous time after metadata loads.
- Playback resumes automatically if it was playing.
- The label shows that the new quality is resuming.

### Loading and buffering UI

The player includes a loading overlay/spinner and responds to:

- `loadstart`
- `waiting`
- `canplay`
- `playing`
- `error`

The overlay is intended to disappear once the browser reaches a playable state.

### Media proxy

Backend route:

```text
GET /api/media-proxy?url=...&detail_path=...&subject_id=...&se=...&ep=...
```

Purpose:

- Fetch signed upstream media using required player headers.
- Preserve browser `Range` headers.
- Return `206 Partial Content` when the upstream responds with a range.
- Preserve `Content-Range`, `Content-Length`, `Accept-Ranges`, `ETag`, and media type.
- Remove the upstream download behavior by serving the response as video media.
- Stream bytes instead of loading the whole movie into memory.
- Add cache and `nosniff` response headers.

The current implementation uses a shared `httpx.AsyncClient` and builds a per-media-request timeout with an unlimited read timeout so that long-lived video streams are not interrupted by the short API timeout.

The last media timeout bug was corrected in commit `0f499e7`: `timeout` must be set while building the `httpx` request, not passed to `AsyncClient.send()`.

### Known playback limitation

The upstream service returns large signed MP4 files and no HLS/DASH source for the tested movie. The local/preview backend has successfully returned valid `206 video/mp4` ranges and the browser has played the movie.

The published Manus production gateway has repeatedly returned `502 Bad Gateway` specifically on the large `/api/media-proxy` response. Catalog, detail, and stream lookup work in production, but production video-byte proxying has not been reliable. This is the primary unresolved issue.

Do not assume that changing the movie source parameters will fix this production-only problem; the source lookup is already returning valid MP4 URLs.

## 5. Subtitle system

### Subtitle lookup

Backend route:

```text
GET /api/stream/{subject_id}/captions?detail_path=...&se=...&ep=...
```

The backend:

1. Looks up the stream.
2. Selects a source with an upstream media ID.
3. Calls the upstream caption endpoint.
4. Normalizes caption language, label, and URL.

### Subtitle proxy

Backend route:

```text
GET /api/subtitle-proxy?url=...
```

The route only allows approved upstream subtitle hosts and returns `text/vtt`.

If the upstream returns SRT, the backend converts it to WebVTT by:

- Adding the `WEBVTT` header.
- Converting comma millisecond separators to periods.
- Removing numeric cue indexes where necessary.

### Frontend behavior

Subtitle lookup was made asynchronous so that video creation does not wait for subtitle metadata. When captions arrive:

- A subtitle selector is inserted.
- Caption tracks are appended to the video.
- The user can select Off or an available language.

## 6. Backend/API routes

### Health and static routes

```text
GET /health
GET /manus-routes.json
GET /styles.css
GET /app.js
GET /assets/{asset}
```

`/health` returns a small service/cache status object.

### Catalog routes

```text
GET /api/home
GET /api/movies?page=1&sort=...
GET /api/tv-series?page=1&sort=...
GET /api/animation?page=1&sort=...
GET /api/ranking
GET /api/new-releases
```

### Search/detail routes

```text
GET /api/search?q=...&page=1
GET /api/search/suggest?q=...
GET /api/detail/{slug}
```

### Refresh routes

```text
GET /api/refresh
POST /api/refresh
POST /api/scheduled/catalog-refresh
```

`/api/refresh` was changed to accept both GET and POST for simple cron services that do not expose an HTTP method selector.

The normal refresh response was deliberately reduced to a compact object:

```json
{
  "status": "success",
  "updated_at": "...",
  "sections": 5
}
```

It no longer returns the full homepage catalog, avoiding cron-job.org response-size limits.

`/api/scheduled/catalog-refresh` remains the authenticated Manus scheduler endpoint and should not be used by ordinary external cron services.

## 7. Automatic updates

The app includes:

- Cached homepage snapshot support.
- Cached new-release snapshot support.
- A frontend new-release feed with an `AUTO-UPDATED` label.
- A refresh button that triggers the normal refresh route.
- A scheduled refresh route for the Manus heartbeat integration.

The scheduled refresh updates:

- Homepage data.
- Movies new-release page.
- Series new-release page.
- Animation new-release page.

Important implementation detail:

- Manus’s internal scheduler can authenticate to `/api/scheduled/catalog-refresh`.
- A simple external cron service should call `/api/refresh` with GET.

## 8. Caching and performance

The backend currently has:

- Shared `httpx.AsyncClient` connection pooling.
- Up to 256 total upstream connections.
- Up to 64 keep-alive connections.
- Short TTL in-memory caching for API results.
- Longer detail and subtitle cache lifetimes.
- Cached category/search responses.
- Player-domain reuse for approximately 10 minutes.
- Token reuse and token locking.
- Cache-Control headers for API and media responses.

The frontend currently has:

- Debounced search.
- Fast 480p default playback.
- Asynchronous subtitle lookup.
- Metadata preload instead of eagerly downloading the entire file.
- Playback-position preservation during quality switching.
- Loading/buffering overlay.

This is not yet a true 20,000-user architecture. It lacks a dedicated video CDN, distributed cache, persistent database, and horizontally scaled media service.

## 9. Visual/design system

The UI uses a dark cinematic style:

- Near-black background.
- Deep navy cards and panels.
- Amber/gold accent color.
- Large poster cards.
- Rounded panels.
- Cinematic hero treatment.
- Responsive catalog grid.
- High-contrast playback controls.

The project includes a Reelstream logo at:

```text
static/assets/reelstream-logo.png
```

## 10. Files and responsibilities

```text
app.py
  FastAPI app, upstream adapter, cache, catalog routes, stream lookup,
  subtitles, media proxy, static serving, refresh routes.

static/index.html
  HTML shell, metadata, asset references, route shell.

static/app.js
  Client routing, catalog rendering, search, detail pages, watchlist,
  movie/series playback, quality switching, subtitle controls.

static/styles.css
  Complete dark cinematic visual system, responsive layout, cards,
  player loading state, category controls, quality/subtitle controls.

static/manus-routes.json
  Route manifest for the web project.

static/assets/reelstream-logo.png
  Project/favicon logo.

Dockerfile
  Python 3.12-slim container; runs Uvicorn on the platform PORT.

requirements.txt
  FastAPI, Uvicorn, httpx.

README.md
  Basic local-run and route notes.

FREE_DEPLOYMENT_GUIDE.md
  Separate deployment instructions; not part of this handoff scope.
```

## 11. Current known issues

1. **Published video playback:** The published Manus site can return `502 Bad Gateway` for `/api/media-proxy` while catalog and stream lookup remain healthy. The last local backend test returned valid `206 video/mp4` bytes, but the published gateway still needs further investigation or a separate media backend/CDN.
2. **Upstream availability:** Some titles genuinely have no upstream stream. The app correctly displays an unavailable message rather than fabricating a source.
3. **Large audience scaling:** The current single FastAPI service and proxy are not designed to guarantee 20,000 concurrent viewers.
4. **Signed URLs:** Upstream media URLs are signed and time-sensitive. Playback always needs a fresh stream lookup.
5. **Quality:** Available qualities depend entirely on the upstream response. The app cannot create 1080p if the upstream only returns 480p.
6. **Persistence:** Watchlist is browser-local; cache files are local filesystem snapshots and are not a distributed database.

## 12. Recommended next debugging plan

When continuing in a new chat, focus on the published media path in this order:

1. Verify `/api/stream/...` returns a fresh signed 480p URL.
2. Verify `/api/media-proxy` with `Range: bytes=0-1023`.
3. Compare the production proxy response with a local FastAPI response.
4. Check whether the hosting gateway permits long-lived `StreamingResponse` and large range responses.
5. If the production gateway cannot proxy media, move only the media/backend portion to a host that supports range streaming, or use a dedicated media CDN/edge proxy.
6. Keep the frontend source-selection and movie `se=0&ep=0` logic unchanged unless a new upstream contract is observed.

## 13. Useful exact test commands

Local health:

```bash
curl http://127.0.0.1:3000/health
```

Movie stream lookup:

```bash
curl 'http://127.0.0.1:3000/api/stream/1979394220376980688?detail_path=modha-rathiri-hindi-i7Dq0g5Edm2&se=0&ep=0'
```

Published stream lookup:

```bash
curl 'https://reelstream-qfcxrzaz.manus.space/api/stream/1979394220376980688?detail_path=modha-rathiri-hindi-i7Dq0g5Edm2&se=0&ep=0'
```

Refresh endpoint:

```bash
curl -X GET 'https://reelstream-qfcxrzaz.manus.space/api/refresh'
```

Expected compact refresh response:

```json
{
  "status": "success",
  "updated_at": "timestamp",
  "sections": 5
}
```
