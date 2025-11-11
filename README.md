# AgentWebBotPlaywright (Windows + Docker)

This guide explains how this repo connects a Playwright container to your already-open desktop
browser via a WebSocket/DevTools endpoint, runs queued scripts in a watch folder, navigates to
websites, saves page HTML, and hands it off to an LLM agent for quick analysis.

## Overview
- You run Chrome/Edge locally with a remote debugging port (e.g., 9222).
- The container connects to that browser over a WebSocket (or CDP) endpoint.
- A watch folder (`playwright/scripts/`) on your host is mounted as `/scripts` in the container.
- Any new `.mjs` script dropped into `/scripts` runs once automatically.
- Scripts can navigate, click buttons, save HTML, and log to `playwright/logs/`.

## Prerequisites
- PowerShell Core (`pwsh`, v7+)
- Docker Desktop
- A Chromium-based browser (Chrome or Edge)

## 1) Build the connector image
Builds a lightweight Node image that contains only the Playwright Node bindings (`playwright-core`).

```powershell
# From the repo root, example is job scraping and application bot
docker build -t ailandedmyjob-playwright .\playwright
```

What’s inside
- Base: `node:20-alpine`
- Installs `playwright-core` (bindings only; no browsers downloaded)
- Entry script: `playwright/connect.mjs` (connects and runs watcher)

## 2) Start your desktop browser with a debugging port
Launch Chrome (or Edge) with `--remote-debugging-port`. Keep this session open while using the
workflow.

Chrome (Windows):
```powershell
& "C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe" `
  --remote-debugging-port=9222 `
  --user-data-dir="C:\\Temp\\chrome-devtools" `
  --no-first-run
```

Edge (Windows):
```powershell
& "C:\\Program Files (x86)\\Microsoft\\Edge\\Application\\msedge.exe" `
  --remote-debugging-port=9222 `
  --user-data-dir="C:\\Temp\\edge-devtools" `
  --no-first-run
```

Confirm the endpoint exists by visiting on the host:
- http://localhost:9222/json/version
- Copy `webSocketDebuggerUrl` from the JSON (it looks like `ws://localhost:9222/devtools/browser/<id>`)

Note: When used from inside a container, `localhost` must be replaced with `host.docker.internal`.

## 3) Run the connector container (watch + keep-alive)
Mount `playwright/scripts` and `playwright/logs` so your scripts run and logs/HTML persist to the
host. Use the WebSocket endpoint (recommended).

```powershell
$scripts = (Resolve-Path .\playwright\scripts).Path
$logs    = (Resolve-Path .\playwright\logs).Path
$ws      = "ws://host.docker.internal:9222/devtools/browser/<paste-id-here>"

# Keep-alive + watch mode: the connector stays up and executes new scripts once.
docker run -d --name ailmj-playwright `
  -e PW_WS_ENDPOINT=$ws `
  -e PW_MODE=watch `
  -e PW_KEEP_ALIVE=true `
  -e PW_CONNECT_TIMEOUT_MS=45000 `
  -v "${scripts}:/scripts" `
  -v "${logs}:/logs" `
  ailandedmyjob-playwright
```

Troubleshooting
- If connection times out, refresh the ID at http://localhost:9222/json/version and update `$ws`.
- If the DevTools HTTP endpoint doesn’t respond, use the WS endpoint (preferred) — the connector
  uses `connectOverCDP` to attach over WS reliably.

## 4) How the watch queue works
- The container polls `/scripts` every ~2 seconds.
- Each new `.mjs`/`.js` file executes exactly once for that container session.
- To re-run a script: rename it (e.g., add a suffix), or restart the container.
- Script shape: default export async function receives `{ browser, chromium }`.

Minimal script template:
```js
// /scripts/example.mjs
export default async ({ browser, chromium }) => {
  const context = browser.contexts()[0] ?? await browser.newContext();
  const page = context.pages()[0] ?? await context.newPage();
  await page.goto('https://example.com', { waitUntil: 'domcontentloaded' });
  return { url: page.url(), title: await page.title() };
};
```

## 5) Navigate and store HTML (examples in repo)
- Open a new tab in the existing window and dump HTML after DOM is ready:
  - `playwright/scripts/linkedin-jobs-save-dom-in-window-fast.mjs`
- Same flow but also wait for network idle:
  - `playwright/scripts/linkedin-jobs-save-dom-in-window.mjs`
- Where files go:
  - Logs: `playwright/logs/*.log`
  - HTML: `playwright/logs/html/*.html`

Filename convention used in scripts
- `<uuid>_<datetime>_<description>.html|.log` (e.g., `8a3..._2025-11-11T18-20-33.123Z_linkedin-jobs-dom.html`)

## 6) Inspecting saved HTML with an LLM agent
Goal: quickly understand which parts of a page to target (selectors/structure) for data retrieval.

Suggested flow
1) Save a fresh HTML snapshot to `playwright/logs/html/` using one of the scripts.
2) Load the HTML into your LLM agent with a prompt like:
   - “Given this HTML, identify the key sections and stable selectors for: job result cards, job
     titles, company names, locations, links to detail pages. Prefer robust selectors over brittle
     ones; avoid dynamic class hashes. Provide CSS selectors and XPaths.”
3) Use LLM output to write Playwright selectors (e.g., `page.locator('a[data-control-name="job_card"]')`).
4) Iterate: re-run a script that extracts those fields, log the data to JSON/CSV.

Notes
- SPAs may lazy-load content; consider waiting for specific selectors instead of full network idle.
- Some sites require auth; captured HTML will reflect the current signed-in state of your browser.
- Avoid scraping content behind strict ToS or protective measures.

## 7) Useful commands
- Tail connector logs:
  ```powershell
  docker logs -f ailmj-playwright
  ```
- List newest HTML/Log files:
  ```powershell
  Get-ChildItem -Recurse .\playwright\logs\* | Sort LastWriteTime -Descending | Select -First 5
  ```
- Restart to re-run all scripts:
  ```powershell
  docker restart ailmj-playwright
  ```
- Stop and clean up:
  ```powershell
  docker rm -f ailmj-playwright
  ```

## 8) Networking and ports (important)
- Your desktop browser listens on `localhost:9222`. From inside the container, use
  `host.docker.internal:9222` (not `localhost`).
- WebSocket endpoints change when the browser restarts; refresh via
  `http://localhost:9222/json/version` on the host.
- If you prefer the DevTools HTTP URL, the connector will still attach over CDP using the WS URL
  retrieved from the HTTP endpoint when available.
