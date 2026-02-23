# Code Review Findings

> Last reviewed: 2026-02-23  
> Scope: security, correctness, reliability, code quality  
> Status key: 🔴 High · 🟡 Medium · 🔵 Low · ✅ Fixed

---

## Open Findings

### Security

| # | Severity | File | Finding |
|---|---|---|---|
| S-01 | 🟡 Medium | `src/server.js` | **`express.json()` has no explicit body size limit.** Express defaults to 100 KB. No route needs more than a few bytes. Tighten to prevent body-flood attacks. |
| S-02 | 🟡 Medium | Deployment | **No UI-layer authentication.** `PUT /api/vpn/:action` (start/stop VPN) is accessible to any process that can reach the server. A reverse proxy with HTTP Basic auth (Nginx, Caddy, Traefik) is the recommended mitigation. |
| S-03 | 🔵 Low | `src/server.js` | **`GLUETUN_CONTROL_URL` is not validated at startup.** The value is used verbatim as a fetch target. A malformed or attacker-controlled value could target arbitrary internal addresses. Validate with `new URL(GLUETUN_URL)` at boot and exit on failure. |
| S-04 | 🔵 Low | `src/server.js` | **Error handler logs raw upstream error messages.** `console.error('[error]', err.message)` may include truncated Gluetun response bodies in container logs. Consider structured logging with level filtering. |
| S-05 | 🔵 Low | `src/server.js` | **No `Strict-Transport-Security` (HSTS) header.** Intentionally omitted for plain-HTTP local use. Must be added if the app is ever placed behind an HTTPS reverse proxy. |
| S-06 | 🔵 Low | `src/server.js` | **Rate limiter uses in-memory store.** Counters reset on every container restart. Acceptable for single-instance home use; note for any production or shared deployment. |
| S-07 | 🟡 Medium | `src/server.js` | **Upstream error messages forwarded to browser.** `gluetunFetch` throws errors containing the upstream status code and up to 200 chars of the response body. Route handlers pass `err.message` directly into the JSON response (`res.json({ error: err.message })`), potentially leaking internal API paths, version strings, or debug info from Gluetun. Return a generic message to the client and log the detail server-side only. |
| S-08 | 🔵 Low | `src/server.js` | **No graceful shutdown handler.** The process does not handle `SIGTERM`/`SIGINT`. Docker sends `SIGTERM` on `docker stop`; without a handler, in-flight requests are dropped and the process falls back to `SIGKILL` after the timeout. Add `process.on('SIGTERM', () => server.close())`. |

### Code Quality / Correctness

| # | Severity | File | Finding |
|---|---|---|---|
| C-01 | 🔵 Low | `src/public/app.js` | **`running` is a dead destructured variable.** `renderVpnStatus` returns `{ state, running }` but `running` is never read in `poll()`. Remove from the destructuring assignment. |
| C-02 | 🔵 Low | `src/public/app.js` | **Total server failure does not reset card fields.** When `fetchHealth()` throws (Node server unreachable), the catch block only calls `renderBanner`. The four data cards retain stale values from the last successful poll. Call `renderPublicIp`, `renderPortForwarded`, `renderDns`, and reset the VPN card fields in the catch path. |
| C-03 | 🔵 Low | `package.json` | **Express 4 used; Express 5 is stable.** Express 5 (released Oct 2024) adds native async error propagation, deprecating the manual 4-argument error handler. Non-urgent upgrade candidate. |
| C-04 | 🔵 Low | All | **No tests.** No unit or integration test suite exists. The highest-value targets are `gluetunFetch` error handling, the `renderVpnStatus` state machine, and `renderBanner` output for each state. |
| C-05 | 🔵 Low | `src/public/app.js` | **`innerHTML` used for spinner markup.** `refreshBtn.innerHTML = '<span class="spin">…</span> Refresh'` is safe (hardcoded string) but inconsistent with the `textContent`-only approach used everywhere else. Use `document.createElement` for consistency. |
| C-06 | 🔵 Low | `src/server.js` | **`express.json()` runs on every request.** The body parser is registered globally but only the `PUT /api/vpn/:action` route consumes a body. Scope it to that route or to `/api/vpn` to skip unnecessary parsing on GETs. |

### Infrastructure / Docker

| # | Severity | File | Finding |
|---|---|---|---|
| D-01 | 🔵 Low | `docker-compose.yml` | **No resource limits.** No `mem_limit`, `cpus`, or `pids_limit` defined. Add `deploy.resources.limits` or compose v2 resource keys to prevent resource exhaustion. |

---

## Fixed Findings (resolved in this review cycle)

<details>
<summary>Click to expand — 22 issues resolved</summary>

| # | Severity | Finding |
|---|---|---|
| F-01 | 🔴 High | `favicon.svg` missing — every page load 404'd and fell through to the SPA handler |
| F-02 | 🔴 High | No rate limiting on read endpoints — `/api/health` (5 parallel upstream fetches) had no protection |
| F-03 | 🔴 High | `npm install` instead of `npm ci` — non-deterministic builds |
| F-04 | 🔴 High | `--no-audit` suppressed npm vulnerability scanning in the Docker build |
| F-05 | 🔴 High | Port bound to `0.0.0.0` — UI exposed to entire local network |
| F-06 | 🟡 Medium | `NODE_ENV=production` not set in Dockerfile |
| F-07 | 🟡 Medium | `node-fetch` dependency unnecessary — Node 20 ships native `fetch` |
| F-08 | 🟡 Medium | `docker-compose` healthcheck missing `start_period` |
| F-09 | 🟡 Medium | `X-Powered-By: Express` header leaked server fingerprint |
| F-10 | 🟡 Medium | `redirect: 'error'` missing on upstream fetch — SSRF redirect amplification risk |
| F-11 | 🟡 Medium | No `Permissions-Policy` header |
| F-12 | 🟡 Medium | Docker base image not pinned to digest (mutable tag) |
| F-13 | 🟡 Medium | `sessionStorage` history not validated on restore — CSS class injection via tampered storage |
| F-14 | 🔵 Low | Duplicate `Content-Security-Policy` (meta tag + HTTP header) |
| F-15 | 🔵 Low | Unknown `/api/*` GET paths returned `index.html` instead of a JSON 404 |
| F-16 | 🔵 Low | `readLimiter` applied to all HTTP methods — `PUT` action requests double-counted |
| F-17 | 🔵 Low | `express.json()` body parser registered without size limit (later noted — see S-01) |
| F-18 | 🔵 Low | `badge.warn` state displayed text "Unknown" — semantically incorrect |
| F-19 | 🔵 Low | Stale IP fields displayed with error badge after failed `publicIp` poll |
| F-20 | 🔵 Low | Toast element missing `role="status"` / `aria-live="polite"` |
| F-21 | 🔵 Low | `no-new-privileges`, `cap_drop: ALL`, `read_only` filesystem not set in compose |
| F-22 | 🔵 Low | `redundant PORT=3000` env var in docker-compose |

</details>

---

## Recommended Next Steps (priority order)

1. **S-01** — Restrict `express.json({ limit: '2kb' })` (one-line change)
2. **S-07** — Stop forwarding upstream error details to the browser; return generic message
3. **C-02** — Reset all card fields in `poll()` catch block
4. **S-03** — Validate `GLUETUN_CONTROL_URL` at startup with `new URL()`
5. **S-08** — Add graceful shutdown handler (`SIGTERM` / `SIGINT`)
6. **S-02** — Document reverse-proxy auth setup in README; add example Caddy/Nginx snippet
7. **C-01** — Remove unused `running` from destructuring in `poll()`
8. **C-06** — Scope `express.json()` to PUT routes only
9. **C-04** — Add tests for `gluetunFetch`, `renderVpnStatus`, and `renderBanner`
10. **C-05** — Replace `innerHTML` spinner with `createElement`
11. **D-01** — Add container resource limits to `docker-compose.yml`
12. **C-03** — Plan Express 5 migration (review changelog for breaking changes first)
