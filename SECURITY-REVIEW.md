# Security review

- **Reviewed:** upstream `jasmucrai/truelayer2actual` at `af2363c` (2026-11-05)
- **Date:** 2026-11-08
- **Scope:** `src/` (web server, OAuth flow, token storage, TrueLayer and Actual
  clients, notifications), `Dockerfile`, GitHub workflows, production
  dependencies.
- **Verdict:** OK to deploy **behind an authenticating reverse proxy**, on an
  internal network with Actual.

## Findings

| # | Severity | Finding | Recommendation / status |
|---|----------|---------|----------------------------------|
| 1 | High | The dashboard has **no authentication**. Anyone who reaches it can start bank connections, delete connections, change pairings, add budgets and trigger syncs. Upstream assumes a LAN or a proxy with its own auth. | Only expose the dashboard through an authenticating reverse proxy; don't publish the port. |
| 2 | Medium | Bank OAuth tokens (`data/tokens.json`), config (`data/config.json`, may hold an Actual encryption password) and a full cached copy of the Actual budget (`data/actual-cache/`) are stored **unencrypted** at rest. Files are written atomically with mode `0600`. | Keep `data/` on a private volume. The fork's image creates it `0700`, owned by a non-root user. |
| 3 | Medium | Container ran as **root** (no `USER`). | Fixed: runs as `node` (uid 1000); CI asserts non-root. |
| 4 | Medium | Uses the Actual **server password**, which grants full API access to Actual. | Reach Actual over an internal network; supply the password as a runtime secret, never in source control. |
| 5 | Medium | OAuth `state` is not bound to the browser session, so anyone with dashboard access could complete a flow and link *their* bank into the budget. | Mitigated by #1 (only trusted users can reach the dashboard). |
| 6 | Low | Base image tag not digest-pinned; upstream publishes `:latest`. | Pin the published image by digest where you deploy it; Dependabot bumps the base image weekly; publish now waits for CI. |
| 7 | Low | `/healthz` is unauthenticated and returns consent-expiry dates and Actual init error text. | Covered by #1. |
| 8 | Info | Host allow-list accepts any IP literal or single-label host (by design for LAN use). | Irrelevant behind a proxy; set `DASHBOARD_URL`. |

## Positives

- No secrets in logs: token exchange and refresh log only URLs, client id and
  OAuth error codes; `describeErrorBody` avoids echoing upstream payloads.
- OAuth `state` is 128-bit random, single-use, 15-minute TTL; pairing sessions
  likewise.
- All HTML output goes through `escapeHtml`; strict CSP
  (`default-src 'none'`, no scripts), `X-Frame-Options: DENY`,
  `Cache-Control: no-store`.
- CSRF defence on POSTs via `Origin` and `Sec-Fetch-Site`; rejects `Origin: null`.
- DNS-rebinding defence via Host allow-list.
- Outbound calls limited to TrueLayer (`auth`/`api`), the configured Actual
  server and optional ntfy / Home Assistant webhooks (env-configured only).
- No `child_process`, `eval`, `new Function` or install scripts.
- `npm audit --omit=dev`: 0 vulnerabilities (axios 1.20.0, express 4.22.3,
  @actual-app/api 26.11.0, better-sqlite3 13.0.3).
- Atomic file writes and a state lock around token/config mutation.

## Fork changes

- `Dockerfile`: non-root `node` user, `/app/data` owned `0700`.
- `ci.yml`: reusable; adds `npm audit` and a Docker build that asserts non-root.
- `publish.yml`: publishes only after CI passes; adds `sha-<short>` tags;
  can be dispatched by the auto-merge workflow.
- Dependabot for npm (daily), Docker base image and GitHub Actions, with
  auto-merge of minor/patch updates after CI; majors labelled `needs-review`.

## Pulling upstream changes

Upstream is not tracked automatically. Use GitHub's **Sync fork** deliberately,
review the diff (especially `src/web/`, `src/auth/`, `src/clients/`), and
update this file's "Reviewed" commit.
