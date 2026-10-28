# IRON PROXY v0.1 — independent browser-proxy alpha

**IRON CORE** (rewriting and URLs) + **IRON NETWORK** (Node transport and sessions) + **IRON BROWSER** (tabbed UI).

IRON is an original codebase, not an Ultraviolet/Rammerhead/Scramjet wrapper. It is a **browser application running inside a normal Chromium/Firefox/Safari browser**. It does **not** implement a native rendering engine. Its iframe contains isolated, proxied webpages; Chromium still does the rendering and JavaScript execution.

**Release state:** coherent alpha implementation, not a universal-web or production-security-certified proxy. All files are supplied; the source passes `node --check` in the authoring environment. **Dependencies could not be installed or integration tests run in that environment because outbound npm DNS was unavailable.** Run `npm test` before exposing a deployment; the test results are NOT claimed as passing.

## Architecture

```text
IRON BROWSER (public/index.html, app.css, app.js)
  tab UI ─ bookmarks/history/settings ─ Brave search results
      │ client-authenticated /api/*        │ iframe sandbox WITHOUT same-origin
      ▼                                    ▼
IRON NETWORK (src/server.js + network.js)  /p/<48-hex-capability>/<scheme>/<host>/<path>
  login sessions           └── proxy tab capability → cookie jar, throttling
  DNS/IP guard ─ pinned lookup ─ Undici HTTP client ─ upstream HTTPS/HTTP
                              │
                              ├── HTML → parse5 AST rewrite → bootstrap shim
                              ├── CSS  → CSSTree AST rewrite
                              └── other data → Node streaming pipeline
```

The **URL structure** preserves relative asset resolution: `/p/<cap>/https/example.com/path/file.css`. Each tab has a unique, unpredictable 192-bit capability and a separate server-side cookie jar. A sandboxed site cannot directly access the parent shell DOM, the shell's localStorage, or another tab's cookies. Proxied pages have an **opaque browser origin**. They can access their own proxy capability: therefore treat site code as untrusted and do not put credentials or keys in the page bootstrap.

**Security caveat:** a compromised/external page can copy its *own* bearer capability. These capabilities are scoped to a tab but permit public-web proxying until expiration/logout. Consequently **do not deploy this alpha as a public untrusted proxy service**. Exposing credentials, public-account logins, or confidential data is not recommended. A separate, more mature security architecture is needed before broad public release.

### Implemented in 0.1

- Password-protected app session with HttpOnly cookie and 12-hour expiration; no anonymous self-service account creation.
- Original `/p/` URL routing and path rewriting; GET, HEAD and POST to HTTP/HTTPS.
- Public-IP-only DNS/URL checks at connection time, rejection of nonstandard ports, redirect validation, resource throttling, a bounded 24-slot/100-waiter request scheduler, connection pooling, timeouts and maximum transform sizes.
- HTML parser-based rewrite for normal links, images, forms, script/style references, srcset (ordinary cases), `<base>`, inline CSS, style tags and meta refresh.
- CSS parser-based rewrite for `url()` and `@import` strings.
- Minimal in-page compatibility shim for string/URL `fetch()`, XMLHttpRequest `.open()`, History API navigation event reporting and common link navigation. **This is not full JS AST rewriting**.
- Streaming non-HTML/CSS resources using Node pipeline (with backpressure); bounded buffers for structural HTML/CSS transforms.
- Per-tab cookie jars, 2-hour inactive tab expiry, up to 12 tabs per login, explicit tab close cleanup, 2000-cookie-write ceiling per tab.
- Custom browser shell: tab switching, back/forward stack, reload, home, omnibox, bookmarks, local history, a private-history option, settings, diagnostics, desktop/mobile layout, automatic idle background-tab suspension and keyboard shortcuts.
- Real search using an optional Brave Search API key (no fabricated results); search links open via IRON CORE in the selected tab.
- Render configuration, health endpoint, secret-based authentication, graceful shutdown.

### Not implemented (do not treat as functional)

| Area | Alpha 0.1 capability | Missing / limitations |
|---|---|---|
| HTML | Structural URL rewrites | Full HTML5 edge cases; charset conversion; incremental parser streaming |
| CSS | Structural `url()` / `@import` | Complex data-srcset syntax; CSSOM rewriting in dynamically inserted sheets |
| JavaScript | Bootstrap shims + unmodified JS bytes | **AST rewrite**, runtime globals, absolute URL module imports, robust location interception |
| Modules | External `src` paths rewritten, relative paths often route | Absolute import specifiers, dynamic imports, import maps |
| Fetch / XHR | String/URL fetch and XHR.open shim | Request-object fetch, credentials emulation, edge CORS behavior |
| WebSocket | None | Native WebSocket interception and gateway / frame protocol |
| Workers | None | `Worker`, SharedWorker and Service Worker interception |
| Storage | Tab-isolated opaque iframe; server cookie jar | Native `document.cookie` APIs and localStorage often throw in opaque sandbox |
| History | Browser shell stacks + injected reports | SPA History API edge cases and origin-verified address bar updates |
| Forms | Simple GET/POST forms | multipart uploads, advanced auth/CSRF flows, unusual verbs |
| Media | Streaming static files | DRM, EME, protected streams, complex audio/video schemes |
| Downloads | Standard Content-Disposition and iframe allow-downloads | Popups, browser permission prompts, complex download managers |
| Third-party auth | Inconsistent | OAuth, anti-bot, CAPTCHA, SSO, CSP/auth constraints cannot be promised |
| Browser permissions | Very limited | Camera, mic, geolocation, hardware-backed APIs, privileged browser features |
| Transport | HTTP/HTTPS requests | HTTP/2 end-to-end tuning, WebSocket bridge, request coalescing, service workers |

**Address-bar warning:** the site iframe cannot be read by the privileged shell due sandbox isolation. The optional navigation message is page-controlled and is not cryptographically verified. Therefore the address bar is NOT a trusted origin / TLS security indicator, even though it displays the URL reported by the page. Do not type sensitive credentials into this alpha.

## Deploy on Render (no Python, works from a Chromebook)

1. Push the contents of this directory (not just a single HTML file) to a GitHub repository.
2. In Render, choose **New → Web Service**, connect that repo and select the Node runtime. Alternatively deploy from `render.yaml` as a Blueprint.
3. **Build command:** `npm install --omit=dev`; **Start command:** `npm start`. For a private or later reproducible deploy, generate and commit `package-lock.json` after your first successful `npm install`, then change to `npm ci --omit=dev`.
4. Set `IRON_ACCESS_KEY` to your **own long, unique, unpredictable passphrase** of at least 16 characters. Never commit it to GitHub. **A public Render URL is still a public endpoint; the password is essential.**
5. Optional: set `BRAVE_SEARCH_API_KEY` from the official Brave Search API dashboard to enable genuine in-app web searches. Without it, direct URL browsing remains available; text search shows an honest configuration error.
6. Optional: `IRON_PUBLIC_ORIGIN=https://your-service.onrender.com` (with no trailing slash). Render sets `PORT`; the server binds `0.0.0.0`.
7. Deploy, verify `/healthz` responds with `{ "ok": true }`, open the service URL and enter your access key.

Render's Node hosting requires the service to listen on `0.0.0.0:$PORT` and forwards incoming HTTPS through its platform infrastructure; this project follows that pattern.

For local Node (22.19.0 or newer):

```bash
npm install
export IRON_ACCESS_KEY='replace-with-your-own-strong-long-passphrase'
npm start
# open http://localhost:10000
```

Windows PowerShell uses `$env:IRON_ACCESS_KEY='...'` rather than `export`. Never configure `IRON_TEST_ALLOW_LOOPBACK` in production.

## Tests and benchmarking

```bash
npm run check
npm test
IRON_ACCESS_KEY='your-secret' npm run bench -- --base http://localhost:10000 --target https://example.com --runs 20
```

Tests include URL/SSRF unit cases, HTML/CSS rewriting assertions and an integration test using a local fixture (loopback is allowed **only** inside a child process with `NODE_ENV=test` and `IRON_TEST_ALLOW_LOOPBACK=1`). Test-mode exceptions are never accepted in production. In your own environment, **run the tests before deploying**. The benchmark measures end-to-end HTTP-proxy fetch completion, **not full in-iframe website rendering** or JavaScript success, and it does not itself establish competition leadership.

Reproducible competition plan: pin Node, hardware, RAM, regional network, selected public sites and versions of all engines. Use 30 cold and 100 warm navigations per website under 1, 4 and 8 concurrent tabs. Collect median/p95 page-load latency (DOMContentLoaded and load), navigation success, console errors, JS tests, redirects, WebSocket echo tests, memory, CPU, and startup overhead. Repeat randomized engine ordering to reduce time/network bias. Publish failures and confidence intervals. **No comparisons or superiority claims exist yet.**

## Future roadmap

1. **Phase 1–2 (first pass):** original proxy routing, session-bound transport and streaming. Implemented but not runtime-verified here.
2. **Phase 3 (partial):** HTML + CSS syntax-tree transforms are present. JS AST transformation, full `srcset`, charset and edge HTML remain.
3. **Phase 4–5 (partial):** shell navigation, tabs and UI are implemented. Full site compatibility remains.
4. **Phase 6 (configurable):** Brave Search results via real API key. No key = search unavailable, not fake.
5. **Phase 7 (partial):** tabs, history, bookmarks and server-side cookie jars. Stable app/session restoration and robust storage virtualization remain.
6. **Phase 8–10 (future):** performance tuning after measurements, complete benchmark suite, production-hardening audits, WebSocket bridging, CSP/storage/worker compatibility and secure-origin architecture.

Design decisions: Node streams for large assets, bounded HTML/CSS materialization where a full syntax tree is necessary; Undici keep-alive and a connection-time DNS guard; no WebAssembly workers until benchmarks show an actual win. IRON does not claim to outperform UV, Rammerhead or Scramjet without comparable results.

## Operational notes

- Ephemeral Render disks and process restarts clear login sessions and cookie jars. Bookmarks/history persist **locally in your browser only**, not on Render.
- Password sign-out destroys all server-side tab jars; closing a tab attempts to dispose its frame and revoke capability.
- This proxy cannot change network restrictions or browser policy and is not designed to defeat access controls.
- Only operate it for sites and services you have permission to access, and respect their terms.
- Private browsing mode in this alpha **only disables recording to this application's local history**; it does not hide requests from the proxy, hosting platform, network or destination website.
