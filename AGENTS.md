# AGENTS.md

## What this is

Route Overlay is a single-page, client-only web app (PWA) for orienteering. A user:

1. Loads a route file exported from Garmin Connect (`.gpx`, `.tcx`, or `.kml`).
2. Loads a photo of a paper orienteering map.
3. Picks 2+ matching point pairs (a point on the route vs. the same physical spot on the map photo) to compute a similarity transform (rotation + uniform scale + translation) that maps route coordinates onto the photo.
4. Fine-tunes the alignment with sliders or by dragging the overlay directly, then exports a PNG of the map photo with the route drawn on top, at full photo resolution.

Everything runs in the browser — no server, no backend, no network calls, no data upload. This is a deliberate privacy/simplicity feature; do not add a backend or external API calls unless the user explicitly asks for it.

## Structure

- [index.html](index.html) — the entire app: markup, CSS, and JS in one file (vanilla JS, no build step, no dependencies). This is also the GitHub Pages entry point, so it must stay named `index.html` at the repo root.
- [manifest.webmanifest](manifest.webmanifest) — PWA manifest (name, icons, theme colors, standalone display).
- [sw.js](sw.js) — service worker that caches the app shell for offline use.
- [icons/icon.svg](icons/icon.svg) — app icon (single scalable SVG, used for manifest + favicon + apple-touch-icon).
- [sample/](sample/) — sample orienteering map photo(s) for manual testing of alignment and control-circle detection. Not part of the deployed app.

## Conventions

- Keep it a single static HTML file with no build tooling or package manager — that's the whole point of the project. Don't introduce bundlers, frameworks, or npm dependencies unless explicitly requested.
- Vanilla ES5/ES6-ish JS, no modules, no external libraries.
- Route parsing supports GPX (`trkpt`/`rtept`), TCX (`Trackpoint`/`Position`), and KML (`gx:coord` or `coordinates`) — see `parseRouteFile` in `index.html`. If adding a new format, follow the same pattern: parse to a flat array of `{lat, lon}` points.
- The alignment math (`computeTransform`/`projectPoint`) is a complex-number similarity transform (rotation + uniform scale, no shear) fit by least squares from control point pairs. Keep any alignment changes consistent with that model unless asked to change the fit method.
- `detectControlMarks` (see `index.html`) finds printed control circles and the start triangle so taps can snap to their centers. It masks magenta/maroon/purple-ish pixels (`isControlColor`, tuned to reject brown contour lines and black symbols), then: circles are found via a circular Hough transform (`detectCirclesByHough`) so they're detected even when touching the same-colored course lines; the triangle is found via flood-fill blob shape analysis (`detectTriangleBlobs`), since it's normally isolated from the course lines. If detection misses marks on a given map, tune `isControlColor`'s thresholds or the Hough vote/radius parameters rather than replacing the approach outright.
- Adding a matching point is a tap-then-confirm flow: tapping the route diagram then the map creates a draggable, unconfirmed `pendingPair` (drawn in the accent color); the user can drag either marker to fine-tune before pressing "Confirm point", which is the only point `controlPoints`/the alignment transform are updated. Keep this confirm step when changing the add-point UI.
- `mapView`/`miniView` plus `makeZoomController` implement pinch-to-zoom/pan for the mini route diagram and map canvas during alignment (step 3). Any pointer math added to those canvases must go through `toContent()` to convert canvas-pixel taps into the same content-space coordinates used elsewhere (control points, route projection, etc.), otherwise taps will be misplaced when zoomed/panned. `sizeMapCanvas` only resets `mapView` when the canvas's pixel dimensions actually change, to avoid discarding zoom/pan on spurious `resize` events (e.g. mobile browser chrome show/hide).
- If you touch `sw.js`, bump `CACHE_NAME` (e.g. `route-overlay-v4`) so returning users get the updated app shell instead of a stale cache.

## Hosting

Deployed via GitHub Pages from this repo (serves `index.html` at the repo root, no `/docs` folder or build step). Pushing to the default branch updates the live site.

## Planned but not yet implemented

These were explicitly requested for later — do not implement unless the user asks:

- A custom domain for the GitHub Pages site (would need a `CNAME` file plus DNS configuration).
- A small, unobtrusive ad shown only on the initial screen (before any files are loaded), for light monetization.
