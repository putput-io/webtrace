# webtrace

See every click, API call, error, and page navigation in your browser — from your AI agent's perspective.

One script tag. One endpoint. Works with any framework.

## Setup

Paste this to your AI agent (Claude Code, Cursor, Codex, etc.):

```
Set up webtrace in this project: curl -sL https://raw.githubusercontent.com/putput-io/webtrace/main/AGENT.md
```

The agent will:
1. Write a `.webtrace.js` file to your static assets directory
2. Add `<script src="/.webtrace.js"></script>` to your HTML
3. Create a small server endpoint to store events in memory
4. Add `*webtrace*` to `.gitignore` — all webtrace files are automatically excluded from commits and deploys, so you never have to worry about cleaning up

Same port, same workflow. No proxy, no dependencies.

## Debugging

After setup, the agent will say: "All done. Go ahead and test the site — when something breaks, tell me and I'll be able to tell you exactly what you did."

Use the site. When something breaks:

> **Something broke. Check the logs.**

The agent reads the event stream and tells you what went wrong.

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

## How it works

A ~2KB script monkey-patches browser APIs (`fetch`, `history.pushState`, `Storage.setItem`) and uses event delegation to capture everything without touching application code. Events are batched (300ms debounce) and POSTed to a server endpoint that stores them in a capped in-memory array (max 500). The agent reads events via GET and clears via DELETE.

## Removing it

1. Delete the `.webtrace.js` file from static assets
2. Remove the `<script>` tag from your HTML
3. Delete the server endpoint

Or tell your agent: **"Remove webtrace from this project."**

## License

MIT
