# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Single-file static web app (`index.html`) — no build step, no dependencies, no backend. Deployable as-is to GitHub Pages, Cloudflare Pages, or any static host.

## How it works

Measures internet latency by timing `fetch()` calls to public DNS-over-HTTPS (DoH) endpoints:

| Endpoint  | IP        | URL pattern                                              |
|-----------|-----------|----------------------------------------------------------|
| Cloudflare| 1.1.1.1   | `https://1.1.1.1/dns-query?name=<random>.test.&type=A`  |
| Google    | 8.8.8.8   | same pattern                                             |
| Google    | 8.8.4.4   | same pattern                                             |
| Quad9     | 9.9.9.9   | same pattern                                             |

- Rotates endpoints round-robin via `epIdx`
- Random label per request (`rndLabel()`) busts caches; HTTP/2 reuse means RTT after the first request reflects pure network latency
- `AbortController` fires after 3 s → `'timeout'` (packet dropped)
- Immediate network `TypeError` → `'offline'` (no connection)
- `Accept: application/dns-json` header required; NXDOMAIN response is expected and fine — only timing matters

## Architecture

Everything lives in `index.html` in three sections:

**CSS** — design-token driven via CSS custom properties (`:root` dark, `[data-theme="light"]` light). Key tokens: `--bg`, `--bg-surface`, `--bg-raised`, `--border`, `--t1/t2/t3` (text), `--accent`, `--green/red/amber` + their `-dim`/`-edge` variants.

**HTML** — six sections inside `.app`: header → status panel → stats grid → controls → chart panel → footer. Tooltip (`#tooltip`) is outside `.app` for fixed positioning.

**JavaScript** — no framework, no modules. Key globals:
- `pings[]` — append-only array of `{ ts, status, lat, ep }`. Never truncated so block indices in the chart stay stable.
- `stats` — running min/max/sum/n/lost/total; updated per ping.
- `epIdx` — round-robin pointer into `ENDPOINTS`.
- `itvMs` — current interval in ms; changed by the `<select>`.
- `timer` — `setTimeout` handle for the ping scheduler.

**Ping loop**: `schedule()` → `doPing()` (async) → `.then(() => setTimeout(schedule, itvMs))`. Sequential — next ping only starts after the current one resolves or times out.

**Chart**: `initChart()` creates 60 `.block` divs once. `refreshChart()` updates each block by comparing `node.dataset.pi` (ping index) to avoid redundant DOM writes. The newest block gets a `just-born` CSS animation class. Latency bar height = `clamp(lat / 300ms, 4%, 100%)`.

**Status logic** (`computeStatus()`):
- `'offline'`  — last 5 pings all non-success and at least 3 samples
- `'degraded'` — >40% loss in last 10 pings (min 5 samples)
- `'online'`   — otherwise

## Serving locally

```bash
# Python 3
python3 -m http.server 8080

# Node (npx)
npx serve .
```

Open `http://localhost:8080`.

## Design notes

- Font: JetBrains Mono (Google Fonts CDN) — used for everything
- Aesthetic: dark precision-instrument / network terminal; sky-blue (`#38bdf8`) accent
- Theme is persisted to `localStorage` key `dns-mon-theme`; defaults to OS preference
- All colours go through CSS custom properties — add new themes by scoping overrides on `[data-theme="..."]`
