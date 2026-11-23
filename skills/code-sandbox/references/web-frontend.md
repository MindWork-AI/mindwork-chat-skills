
# Web front-ends, single-file artifacts and browser tests

Read this when the deliverable runs in a browser, or a page needs a screenshot or proof that it works. Sessions and publishing are covered in `SKILL.md`.

The running example is a click counter in React, bundled into one `bundle.html` the user can open anywhere,
and clicked through by a real browser before anyone sees it.

## Two rules that shape everything

**Browsers need `chromium_sandbox=False`** (Python) or `chromiumSandbox: false` (Node) in `launch()`. The sandbox
is already isolated and Chromium's own sandbox cannot start inside it.

**A server lives only as long as the call that started it.** Processes are stopped after every call. Start the
server, test against it and finish in one program.

## A plain page needs no build

For a report or simple tool, write one HTML file with inline CSS and JavaScript. React, Tailwind or chart
libraries can come from a CDN `<script>` tag (internet is on) or be inlined. Check with a screenshot, then
publish `page.html`.

## React app as one file

A Vite + React + TypeScript project works with `pnpm`:

```bash
set -eo pipefail
pnpm create vite counter-app --template react-ts
cd counter-app && pnpm install 2>&1 | tail -3
pnpm exec vite build 2>&1 | tail -4
```

It downloads from npm (about 15 seconds) and persists in the session, so do it once. The build writes a
`dist/` folder; to hand over **one** self-contained HTML file, use the web-artifacts-builder scripts when the
user's setup provides them: upload `init-artifact.sh` and `bundle-artifact.sh` as `content_text` and
`shadcn-components.tar.gz` as `content_base64`, then `bash init-artifact.sh app` and, after editing
`src/App.tsx`, `bash bundle-artifact.sh` (result: `bundle.html`, about 300 KB, no external references).
Delete a `<link rel="icon" ...>` to `/favicon.svg` from `index.html` if the bundler cannot resolve it.
Without those scripts, a Vite single-file plugin may work (untested here); otherwise zip `dist/`.

Always `set -eo pipefail` when piping into `tail`/`head`, otherwise a failed build reports `status: ok`.

## Prove it works

Open the result in Chromium, use it like a person, collect page errors, take a screenshot. Use the roles,
labels and test ids of your own app:

```python
from playwright.sync_api import sync_playwright
with sync_playwright() as p:
    b = p.chromium.launch(chromium_sandbox=False)
    pg = b.new_page(viewport={"width": 640, "height": 360})
    errors = []; pg.on("pageerror", lambda e: errors.append(str(e)))
    pg.goto("file:///mnt/data/counter-app/bundle.html")
    pg.get_by_role("button", name="Add one").click()
    print("count =", pg.get_by_test_id("count").inner_text(), "errors =", errors)
    pg.screenshot(path="counter.png")
    b.close()
```

Then `sandbox_read_file("counter.png")` to see the page: layout, contrast, overflow. For mobile pages take a
second shot at 390 px width.

To test against a dev server, start it, wait, test and stop in one bash program:

```bash
set -eo pipefail
cd counter-app
pnpm exec vite --port 5173 --strictPort > /tmp/vite.log 2>&1 &
for i in $(seq 50); do curl -sf localhost:5173 >/dev/null && break; sleep 0.2; done
node /mnt/data/test.mjs      # chromium.launch({ chromiumSandbox: false }) ... page.goto("http://localhost:5173")
```

## Hand over

Publish `bundle.html` on its own; the link opens the app in the browser. For a multi-file build, zip it
first (`zip -r ../site.zip dist`) because folders cannot be published, and tell the user to unzip it before
opening. Add the screenshot as a preview if useful. Links expire (usually after 24 hours) and the page is
a static file, not a hosted site: say so if the user expects a permanent URL.
