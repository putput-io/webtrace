# webtrace

See every click, API call, error, and page navigation in your browser — from your AI agent's perspective.

Zero project file changes. Zero dependencies. Works with any framework.

## Setup

Paste this to your AI agent (Claude Code, Cursor, Codex, etc.):

> **Add webtrace to this project so I can reproduce bugs in my browser and you can see exactly what happened. Read the setup instructions at: https://raw.githubusercontent.com/putput-io/webtrace/main/AGENT.md**

The agent will create a `.webtrace/` folder (gitignored) containing a small proxy server that sits in front of your dev server. No project files are modified.

## Debugging

After setup, the agent will say something like: "All done. Go ahead and test the site at `localhost:<port>` — when something breaks, tell me and I'll be able to tell you exactly what you did."

Use the site. When something breaks:

> **Something broke. Check the logs.**

The agent reads the event stream — every click, every API call and response, every JS error, every navigation — and tells you what went wrong.

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

The agent creates a Node.js proxy server in `.webtrace/` (using only built-in modules, no dependencies). The proxy:

1. Forwards all requests to your real dev server
2. Injects a ~2KB `<script>` snippet into HTML responses
3. Receives browser events at `/api/v1/debug` and stores them in memory (capped at 500)
4. Serves events to the agent via a secret-gated GET endpoint

```
You browse at                        Proxy                          Dev server
localhost:9877  ──── requests ────>  .webtrace/server.js  ────────> localhost:5173
                <─── HTML + snippet   (injects snippet,              (your app,
                                       stores events)                 unchanged)
```

No files in your project are touched. The `.webtrace/` folder is excluded via `.git/info/exclude` (a local-only gitignore that's never committed).

## Removing it

```
rm -rf .webtrace
```

That's it. No files to revert, no config to undo.

## License

MIT
