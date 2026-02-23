# webtrace

See every click, API call, error, and page navigation in your browser — from your AI agent's perspective.

One `<script>` tag. No SDK. No signup. Production-safe.

## Setup

Paste this to your AI agent (Claude Code, Cursor, Codex, etc.):

> **Add webtrace to this project so I can reproduce bugs in my browser and you can see exactly what happened. Read the setup instructions at: https://raw.githubusercontent.com/putput-io/webtrace/main/AGENT.md**

The agent will:
1. Detect your framework (SvelteKit, Next.js, Express, etc.)
2. Add a `<script>` snippet to your HTML `<head>`
3. Add a small server endpoint to receive events
4. Generate a secret so only your agent can read the logs

No npm install. No config files. Two small code changes, fully automated.

## Debugging

After setup, the agent will say: "All done. Go ahead and test the site — when something breaks, tell me and I'll be able to tell you exactly what you did."

Reproduce the bug, then tell your agent:

> **Something broke. Check the logs.**

The agent silently reads the event stream — every click, every API request and response, every JS error, every navigation — and tells you what went wrong. No curl commands. No console pasting. It just knows.

## What it captures

| Event | How |
|-------|-----|
| All fetch requests + responses | Wraps `window.fetch` |
| All clicks | `document` click listener (capture phase) |
| All input changes | `document` change listener (passwords masked) |
| All form submissions | `document` submit listener |
| All page navigations | Wraps `history.pushState` / `replaceState` + `popstate` |
| All JS errors | `window.error` + `unhandledrejection` |
| localStorage writes | Wraps `Storage.prototype.setItem` / `removeItem` |
| Tab focus/blur | `visibilitychange` |

## Production-safe

Webtrace is designed to stay deployed:

- **Auth-gated reads** — logs require a secret to access. Without it, the endpoint returns 404.
- **Capped storage** — events auto-evict at 500 entries. No memory growth.
- **No external calls** — events POST to your own server. No third-party services.
- **~2KB client footprint** — negligible impact on page load.

## How it works

The client snippet monkey-patches browser APIs (`fetch`, `history.pushState`, `Storage.setItem`) and uses event delegation (`document.addEventListener`) to capture everything without touching application code. Events are batched (300ms debounce) and POSTed as JSON to your server endpoint.

The server stores events in a capped in-memory array. The agent reads them via a GET request with the secret.

```
Browser                              Your Server
┌──────────────────┐    POST         ┌──────────────────┐
│ fetch wrapper    │ ──────────────> │ In-memory array   │
│ click listener   │   (no auth)    │ (max 500 events)  │
│ input listener   │                │                    │
│ nav wrapper      │ <───────────── │ GET (secret req'd) │
│ error listener   │   Agent reads  │                    │
└──────────────────┘                └──────────────────┘
```

## Removing it

Tell your agent:

> **Remove webtrace from this project.**

The agent reverts the two changes it made: the `<script>` tag and the server endpoint.

## License

MIT
