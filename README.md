# RPN Scientific Calculator (installable web app)

A single-page, stack-based (RPN) scientific calculator that installs as an app on
Android and iOS, runs full screen, and works offline once installed.

## Files
| file | purpose |
|---|---|
| `index.html` | the whole app (markup, styles, logic) |
| `manifest.webmanifest` | app name, icons, display mode: installed = `standalone` (browser controls gone, standard Android bars stay); the in-page Full button can go deeper via the Fullscreen API |
| `sw.js` | service worker: caches the app shell so it launches offline. **Bump `CACHE = 'rpn-calc-v1'` on every release.** |
| `icons/` | 192/512 (`any`), 512 `maskable`, `apple-touch-icon` (180), favicon |
| `.nojekyll` | skips Jekyll processing on GitHub Pages |

## Deploy (GitHub Pages)
```bash
git init && git add . && git commit -m "RPN calculator"
git remote add origin https://github.com/<you>/rpn-calc.git && git push -u origin main
```
Then **Settings -> Pages -> Build and deployment: Deploy from a branch -> `main` / `/ (root)` -> Save**.
App URL: `https://<you>.github.io/rpn-calc/`.

Confirm both answers after deploying (either one wrong = install refused):

```bash
curl -sI https://<you>.github.io/rpn-calc/manifest.webmanifest | grep -i content-type   # want application/manifest+json
curl -sI https://<you>.github.io/rpn-calc/sw.js                | head -1                # want HTTP/2 200
```

Everything is addressed relatively (`start_url: "./"`, `scope: "./"`), so the same
files work at a domain root or under a project subdirectory.

## Install
* **Android (Chrome / Edge)** - open the URL, then use the in-app *Install* chip, the
  address-bar install icon, or browser menu -> **Install app**. It then launches from
  the launcher with no browser UI, full screen, offline.
* **iOS / iPadOS Safari** - Share -> **Add to Home Screen**. (Safari exposes no
  install API; `apple-mobile-web-app-capable` + `apple-touch-icon` drive this path.)
* **Desktop (Chrome / Edge)** - address-bar install icon, or menu -> *Install*.

## Local testing
```bash
python3 -m http.server 8080     # then open http://localhost:8080
```
`localhost` is a secure context, so the manifest and service worker work there.
Opening `index.html` from disk (`file://`) does **not** install and cannot go offline.

DevTools -> Application -> Manifest lists any unmet installability criterion verbatim;
the Lighthouse "PWA" audit checks the same set.

If a host ever serves `.webmanifest` with a non-JSON content type, rename it to
`manifest.json` and update the `<link rel="manifest">` href - `application/json`
is accepted too. (GitHub Pages serves `.webmanifest` correctly out of the box.)

## Layout (portrait and landscape)
One column normally; **two columns as soon as the viewport is much wider than it is
tall** (a phone turned sideways, a short wide window). The rule lives in
`isWideLayout()`: ratio `width/height >= 1.3`, at least 560px wide, and no taller
than `PHONEISH_H` (620px - the same figure as the `(max-height: ...)` mobile media
query). It sets `html.wide`, and CSS turns `.calculator` into a two-column grid:

| left column | right column |
|---|---|
| stack history + input line, then the four scientific / stack rows (AC..EE) | the number pad down from row 5: digits, operators, `Inv`/`Fn`/cursor keys, Deg-Rad and ENTER |

No markup is duplicated to do this: the keypad's two `<div class="keys">` groups are
`display: contents` (invisible to layout) in one column, and only become separate
blocks - while `.keypad` itself goes `display: contents` - in the wide layout.
Everything else is unchanged: keys still never overlap or clip, the history window
eats the leftover height on the left, and the digit rows stretch to fill the right.
Rotating back re-runs the same auto-detection (resize / orientationchange /
visualViewport events plus a 500ms poll), so no reload is needed. Long-press the
display and the readout now ends with `layout: one-column|two-column` plus the size
and ratio it decided from.

## Display modes (what the app auto-detects)
| mode | Full button | top padding | bottom guard |
|---|---|---|---|
| browser tab (windowed) | *(none - removed; install for full screen)* | `env(safe-area-inset-top)` as reported | honoured |
| Fullscreen API (dormant: no button, code kept) | *(restore via comment in Row 9)* | **guard ON** - this mode really draws under the status bar/cutout (`viewport-fit=cover` -> Android SHORT_EDGES while fullscreen); uses live `env()` with the recorded top inset as fallback | live insets only (gesture bar is hidden here, remembered floors would push keys too far up) |
| installed app (standalone) | hidden (`html.app-mode`) - this is THE full-screen path | none: the viewport already starts below the status bar, so the reported top inset is ignored (padding it would double-count and show a blank strip) | honoured (curved corners / gesture area) |
| iOS home-screen app | hidden (no Fullscreen API in WebKit) | as above | as above |

Long-press (or right-click) the **display window** for a readout of the detected
mode, the insets the browser reports, and the padding actually applied. If a device
ever draws content under its status bar, set `--min-top-guard` (in `html.immersive`)
to the height needed - no other change required.
