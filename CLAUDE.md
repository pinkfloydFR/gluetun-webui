# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A minimal Node/Express web UI that proxies and displays status from one or two [Gluetun](https://github.com/qdm12/gluetun) VPN containers' HTTP control servers. No frontend framework or build step — plain HTML/CSS/vanilla JS served as static files. This is a fork customized from the upstream `Sir-Scuzza/gluetun-webui` project.

## Commands

- `npm start` — run the server (`node src/server.js`), listens on `PORT` (default 3000)
- `npm run dev` — run with `nodemon` for auto-restart on change
- No test suite, linter, or build step exists in this repo (see `FINDINGS.md` C-04 — adding tests is a known gap, not an oversight to silently "fix" by inventing a framework choice without asking).
- Docker: `docker compose -f docker-compose.example.yml up -d --build` — the project ships no pre-built image; it must always be built locally so local modifications take effect.

## Architecture

**Backend (`src/server.js`)** — a single-file Express app that:
- Reads VPN instance config from env vars into an `INSTANCES` map keyed `'1'`/`'2'`. Each instance has `name`, `url` (Gluetun control server base URL), `apiKey`. Instance 1 falls back to legacy `GLUETUN_CONTROL_URL`/`GLUETUN_API_KEY` if `GLUETUN_1_*` isn't set — preserve this fallback when touching instance resolution.
- Every `/api/*` route (except `/api/instances`) requires `?instance=1` or `?instance=2` in the query string, resolved via `resolveInstance()`. There is no concept of a "default" instance for data routes.
- `/api/*` routes are thin proxies to the Gluetun control API (`gluetunFetch`), never expose upstream error bodies to the client — errors are logged server-side (`console.error('[upstream]', ...)`) and a generic message is returned (`handleUpstreamError`). Keep this pattern for any new proxy route: don't leak upstream response bodies to the browser.
- Two endpoints have version-compatibility fallback logic that must be preserved when editing: `fetchVpnSettings()` (tries `/v1/vpn/settings`, falls back to `/v1/openvpn/settings` on 401/404) and `fetchPortForwarded()` (tries `/v1/portforward`, falls back to `/v1/openvpn/portforwarded` on 404). These exist because different Gluetun versions/ACL roles expose different endpoint names.
- `GET /api/health` is dual-purpose: with no `?instance=` param it's a plain liveness probe returning `{ok:true}` (used by the Docker `HEALTHCHECK` and must stay unauthenticated/param-free for that to work); with `?instance=N` it fans out `Promise.allSettled` across five upstream calls and returns an aggregate snapshot. Only a 401 from the *primary* `vpn/status` call sets `authError` — secondary endpoint 401s (e.g. an ACL role that omits `portforward`) must not trip the global auth-error banner.
- Three rate limiters exist for different route classes: `readLimiter` (GET `/api/*`, 120/min), `uiLimiter` (static file serving only, 500/15min), `vpnActionLimiter` (PUT `/api/vpn/:action`, 10/min). Do not apply `uiLimiter` broadly to `/api/*` — it was previously misapplied there and caused the dashboard to 429 within minutes under normal auto-refresh polling (see FINDINGS.md N-01); keep it scoped to `express.static`.
- The `/api/*` 404 handler (`app.use('/api/', ...)`) must stay registered *before* the SPA catch-all (`app.get('*', ...)`) so unknown API paths return JSON 404s instead of `index.html`.
- Security headers (CSP, X-Content-Type-Options, X-Frame-Options, Referrer-Policy, Permissions-Policy) are set via middleware near the top of the file — extend this block rather than adding headers ad hoc in individual routes.
- `startAlertMonitor()` runs a server-side `setInterval` (independent of any browser tab) that checks each *configured* instance's `/v1/vpn/status` every `ALERT_CHECK_INTERVAL_MS` and calls `sendAppriseNotification()` on an Apprise-compatible endpoint (`APPRISE_URL`) only on up/down **transitions**, tracked per-instance in `alertState` — never on every check, to avoid spam. This is the only background polling in the app; the frontend's own polling (`app.js`) is separate and only runs while a tab is open, so it cannot be relied on for alerting. `APPRISE_URL` typically embeds a secret token in its path — it must stay in env vars / `docker-compose.yml` (untracked), never hardcoded or committed.

**Frontend (`src/public/`)** — no build step; served directly by `express.static`.
- `app.js` polls `GET /api/health?instance=N` for both instances in parallel on an interval (`scheduleNextPoll`, user-configurable 5s–60s, persisted only in-memory via the `<select>`), and renders into DOM elements namespaced per instance via the `$i(n, id)` helper, which expects element IDs of the form `i{n}-{id}` (e.g. `i1-vpn-status`, `i2-badge-vpn`). When adding a new displayed field, follow this `i{n}-` prefix convention in both `index.html` and the corresponding `render*(n, ...)` function.
- Poll history (last 30 states) is kept in `sessionStorage` per instance (`gluetun_history_{n}`) and validated against `VALID_STATES` on load before use — untrusted storage content should never be rendered without this kind of validation.
- Instance display names/configured-state come from `GET /api/instances` at init, not hardcoded — `instanceConfig` starts with placeholder defaults and is overwritten once that call resolves.
- VPN start/stop (`vpnAction`) has no UI-level authentication; the README documents fronting the app with a reverse proxy (Caddy/Nginx/Traefik) doing HTTP Basic auth for any non-localhost exposure. Don't add auth logic to the app itself without discussing — this is a documented, intentional boundary.

## Deployment / Docker conventions

- `Dockerfile` is a two-stage build; both `FROM` lines pin the same image by digest (not just tag) — when bumping the Node/Alpine version, update the digest on both stages together, and re-verify `package-lock.json` is committed (`npm ci` requires it, and this repo has previously regressed to `npm install`/missing lockfile — see FINDINGS.md D-03).
- Runs as non-root (`appuser`), read-only root filesystem, all capabilities dropped — `docker-compose.example.yml` mirrors these hardening flags (`read_only`, `cap_drop: [ALL]`, `no-new-privileges`). Preserve these when editing compose examples.
- The container binds to `127.0.0.1` in the example compose, not `0.0.0.0` on the host — do not change this default to a network-wide bind without explicit instruction, since exposing VPN start/stop controls network-wide with no built-in auth was a previously-fixed finding (FINDINGS.md F-05).

## Project-tracked findings

`FINDINGS.md` is a running, dated code-review log (bugs/security/quality/infra) with severity tags and a "Recommended Next Steps" priority list, maintained across review passes. If asked to do a code review or fix "known issues" in this repo, check this file first — it likely already documents the issue, its severity, and whether it was previously fixed and regressed.
