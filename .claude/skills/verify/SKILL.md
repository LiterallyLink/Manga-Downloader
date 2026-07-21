---
name: verify
description: Launch MangaShelf (Electron) and drive its UI over CDP to observe a change end-to-end.
---

# Verifying MangaShelf

Electron app, no test suite and no browser-driver deps. Verification means
launching the real app and driving the renderer over the Chrome DevTools
Protocol.

## Launch on isolated data

Never drive the app against the real profile: the UI mutates
`library.json` (reading progress, follows, downloads) as you click.

Substitute your real scratchpad path for `SP` below. It must be absolute and
outside the repo: a placeholder that fails to expand once left a whole Chromium
cache tree committed under `./undefined/`. The guard makes that failure loud
instead of silent, so keep it.

```bash
SP=<absolute scratchpad path>
case "$SP" in /*|[A-Za-z]:*) ;; *) echo "SP is not an absolute path; aborting"; exit 1;; esac
mkdir -p "$SP/udata"
cp "$APPDATA/mangashelf/library.json" "$APPDATA/mangashelf/settings.json" "$SP/udata/"
./node_modules/.bin/electron . --remote-debugging-port=9222 --user-data-dir="$SP/udata"
```

Electron honors `--user-data-dir`, so `app.getPath('userData')` resolves
into the copy. Seeding from the real `library.json` is what gives you
actual downloaded series and Continue Reading entries to drive; `settings.json`
carries the library path so `mangafile://` pages resolve. Real profile stays
untouched: confirm with a read of it afterwards.

Shut down with `taskkill //IM electron.exe //F`. The background Bash task
reports exit 1 when you do; that's the kill, not a crash.

## Drive it

Node 22 has a global `WebSocket`, so a ~60-line CDP client is enough. Grab
the page target from `http://127.0.0.1:9222/json/list` (match `index.html`),
then use `Runtime.evaluate` with `awaitPromise` + `returnByValue`,
`Input.dispatchMouseEvent` for real clicks, and `Page.captureScreenshot`.

Gotchas:

- `Runtime.evaluate` reuses one context, so top-level `const x` collides
  across calls. Wrap every expression in `(()=>{...})()` or `(async()=>{...})()`.
- Card hover actions (`.card-action`) only render on `:hover`. Send a
  `mouseMoved` to the cover first, or call `.click()` on the element directly.
- Views render asynchronously (MangaDex fetches). Resolve after a
  `setTimeout` of 1.5s for a view, 4-6s for the reader.
- `window.api.*` is reachable from `Runtime.evaluate`, which is the fastest
  way to assert persisted state (`getReadingAll`, `getLibrary`).

## Flows worth driving

- Home > Continue Reading: card click opens detail; hover play resumes;
  hover trash removes (section disappears when emptied).
- Library > Downloaded: card click opens the reader from local files;
  closing the overlay uncovers the same tab.
- The reader is an overlay over `#content`, not a route. "Close" hides it
  and reveals whatever view was underneath, so check
  `#content .view-title` to know where a close will land.

## Dev control server (simplest handle)

`electron . --dev` starts an HTTP control port on **127.0.0.1:9310**
([main.js](../../../electron/main.js), `startDevControlServer`). Prefer it over
raw CDP — no WebSocket plumbing:

```bash
curl -s -X POST http://127.0.0.1:9310/eval --data-binary '(async()=>{ ... })()'
curl -s "http://127.0.0.1:9310/shot?file=/abs/path.png"
curl -s http://127.0.0.1:9310/close     # same path as the titlebar X
```

**Renderer `window.close()` does NOT fire the BrowserWindow `close` event** —
it exits without running the quit handler. Any test touching quit behavior
must use `/close`, or it silently proves nothing.

Queueing real downloads for a test: point `settings.json` `libraryPath` at a
throwaway folder first. `download()` skips chapters already in the library, so
re-running a test with the same chapters queues nothing — filter against
`getLibraryManga(id).chapters` and take fresh ones, and queue 20+ if you need
the queue to still be busy a few seconds later.

## View lifecycle

`render(root, ...)` always receives `#content` itself, which never leaves the
DOM. `root.isConnected` is therefore **always true** and is useless as a
"am I still the active view?" guard. Views get a 4th `signal` argument that
`app.js` aborts on every navigation: hang window listeners off it
(`addEventListener(..., { signal })`) and check `signal.aborted` after awaits.

Regression to watch for: leaked listeners show up as *duplicated page
content*, not as a crash. Drive a view several times, then fire the event it
listens for, and count `.back-btn` / `.detail-hero` in `#content` — it should
be 1.

## Known pre-existing quirk

Resuming lands the page indicator ~2 pages before the saved page in
vertical/strip mode (saved 25 → shows 24/114), then saves that drifted
value. Both the Continue button and card-click paths do it. Not a
regression: don't chase it unless the change is about resume.
