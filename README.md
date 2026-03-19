# DNS Uptime Monitor

A single-page internet connectivity monitor that measures real-time latency and packet loss by timing DNS-over-HTTPS requests. No backend, no build step, no dependencies — just one `index.html` file.

![dark theme screenshot placeholder](https://placehold.co/800x400/06080d/38bdf8?text=DNS+Uptime+Monitor)

## What it does

Every second (configurable), the app fires a `fetch()` to one of four public DNS-over-HTTPS endpoints and measures the round-trip time:

| Provider    | IP        |
|-------------|-----------|
| Cloudflare  | 1.1.1.1   |
| Google      | 8.8.8.8   |
| Google      | 8.8.4.4   |
| Quad9       | 9.9.9.9   |

Each request queries a random hostname (`<random>.test.`) so responses are never cached. After the first request, HTTP/2 connection reuse means the measured latency reflects pure network RTT rather than TLS/TCP setup overhead.

Requests time out after 3 seconds. A timeout is counted as packet loss; an immediate network error is counted as offline.

## Status detection

| Status     | Condition                                            |
|------------|------------------------------------------------------|
| 🟢 Online   | Default — recent pings succeeding normally           |
| 🟡 Degraded | >40% packet loss in the last 10 pings (min 5 samples)|
| 🔴 Offline  | All of the last 5 pings failed (min 3 samples)       |

## Chart

The chart shows one row per calendar minute, newest at the top. Each row contains 60 blocks — one per second. Block colour encodes latency from bright green (fast, <50 ms) through dark green to red (slow/timeout). Empty slots are transparent gaps.

The current in-progress minute is highlighted with a blue left edge.

## Features

- Dark / light theme toggle, persisted to `localStorage`
- Adjustable ping interval (1 s / 2 s / 5 s / 10 s / 30 s)
- Running stats: current latency, average, min, max, packet loss %
- Per-block tooltip showing timestamp, endpoint, and latency
- Up to 20 minutes of history kept in memory
- Fully client-side — works from `file://` or any static host

## Running locally

```bash
# Python 3
python3 -m http.server 8080

# Node
npx serve .
```

Then open `http://localhost:8080`.

## Deploying

Drop `index.html` onto any static host — GitHub Pages, Cloudflare Pages, Netlify, etc. No build step or server-side logic required.

## How latency is measured

```
const t0 = performance.now();   // recorded before fetch()
await fetch(dohUrl, { signal }); // DNS query over HTTPS
const lat = performance.now() - t0;
```

`performance.now()` gives sub-millisecond precision. The timestamp for each ping is also captured before `fetch()` so that timeout entries (which resolve 3 s later) are plotted in the correct second-slot on the chart rather than drifting forward.
