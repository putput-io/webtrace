# Webtrace — Agent Setup

Browser event interceptor. Captures clicks, API calls, errors, navigation. You read the logs via curl.

## Setup (2 changes)

### 1. Detect framework and find the HTML shell

| Marker file | Framework | HTML file |
|---|---|---|
| `svelte.config.js` | SvelteKit | `src/app.html` |
| `next.config.*` | Next.js | `app/layout.tsx` |
| `nuxt.config.ts` | Nuxt | `app.html` |
| `vite.config.*` (no above) | Vite | `index.html` |

### 2. Add the snippet to `<head>` (before app scripts)

```html
<!-- webtrace -->
<script>
(function(){var q=[],fl=!1,oF=window.fetch;function L(t,d){d.page=location.pathname+location.search;q.push({ts:Date.now(),type:t,detail:d});S()}var ft;function S(){ft||(ft=setTimeout(F,300))}function F(){ft=null;if(!q.length||fl)return;fl=!0;var b=q.splice(0);oF("/api/v1/debug",{method:"POST",headers:{"Content-Type":"application/json"},body:JSON.stringify(b)}).catch(function(){}).finally(function(){fl=!1})}function P(el){var p=[];while(el&&el!==document.body&&p.length<4){var s=el.tagName.toLowerCase();if(el.id)s+="#"+el.id;p.unshift(s);el=el.parentElement}return p.join(" > ")}window.fetch=function(i,o){var u=typeof i==="string"?i:i instanceof Request?i.url:String(i);if(u.indexOf("/api/v1/debug")!==-1)return oF.apply(this,arguments);var m=(o&&o.method)||"GET",bd=o&&o.body,ps=null;try{if(bd)ps=JSON.parse(bd)}catch(e){}L("fetch_req",{method:m,url:u,body:ps});return oF.apply(this,arguments).then(function(r){var c=r.clone();c.json().then(function(d){L("fetch_res",{status:r.status,method:m,url:u,data:d})}).catch(function(){L("fetch_res",{status:r.status,method:m,url:u,data:"(non-json)"})});return r}).catch(function(e){L("fetch_err",{method:m,url:u,error:String(e)});throw e})};document.addEventListener("click",function(e){var el=e.target,t=(el.textContent||"").trim().substring(0,80),h=null,a=el.closest?el.closest("a"):null;if(a)h=a.getAttribute("href");L("click",{el:P(el),href:h,text:t,isButton:!!(el.closest?el.closest("button"):null)})},!0);document.addEventListener("change",function(e){var el=e.target;if(!el.tagName)return;var tg=el.tagName.toLowerCase();if(tg==="input"||tg==="select"||tg==="textarea"){var v=el.value;if(el.type==="password")v="***";else if(v&&v.length>80)v=v.substring(0,80)+"...";L("input",{el:P(el),name:el.name||el.id||"",type:el.type||"",value:v})}},!0);document.addEventListener("submit",function(e){var f=e.target,d={};try{new FormData(f).forEach(function(v,k){d[k]=typeof v==="string"?v.substring(0,80):"(file)"})}catch(x){}L("submit",{action:f.action||"",method:f.method||"GET",el:P(f),data:d})},!0);var oSI=Storage.prototype.setItem;Storage.prototype.setItem=function(k,v){L("ls_set",{key:k,value:typeof v==="string"&&v.length>60?v.substring(0,60)+"...":v});return oSI.apply(this,arguments)};var oRI=Storage.prototype.removeItem;Storage.prototype.removeItem=function(k){L("ls_remove",{key:k});return oRI.apply(this,arguments)};window.addEventListener("popstate",function(){L("nav",{type:"popstate",path:location.pathname+location.search})});var oPS=history.pushState;history.pushState=function(){oPS.apply(this,arguments);L("nav",{type:"pushState",path:location.pathname+location.search})};var oRS=history.replaceState;history.replaceState=function(){oRS.apply(this,arguments);L("nav",{type:"replaceState",path:location.pathname+location.search})};window.addEventListener("error",function(e){L("error",{message:e.message,filename:e.filename,line:e.lineno,col:e.colno})});window.addEventListener("unhandledrejection",function(e){L("error",{message:String(e.reason),type:"unhandledrejection"})});document.addEventListener("visibilitychange",function(){L("visibility",{hidden:document.hidden})});window.addEventListener("beforeunload",function(){F()});L("init",{url:location.href,referrer:document.referrer})})();
</script>
```

### 3. Create the server endpoint

Generate a random secret (16+ hex chars). Create the endpoint file for the detected framework:

**SvelteKit** — create `src/routes/api/v1/debug/+server.ts`:
```typescript
import { json, type RequestHandler } from "@sveltejs/kit";
const SECRET = "GENERATED_SECRET";
const MAX = 500;
const logs: { ts: number; type: string; detail: unknown }[] = [];
export const POST: RequestHandler = async ({ request }) => {
  const e = await request.json();
  if (Array.isArray(e)) { logs.push(...e); while (logs.length > MAX) logs.shift(); }
  return json({ ok: true });
};
export const GET: RequestHandler = async ({ url }) => {
  if (url.searchParams.get("secret") !== SECRET) return new Response("", { status: 404 });
  const since = Number(url.searchParams.get("since") || 0);
  return json(since ? logs.filter(l => l.ts > since) : logs);
};
export const DELETE: RequestHandler = async ({ url }) => {
  if (url.searchParams.get("secret") !== SECRET) return new Response("", { status: 404 });
  logs.length = 0;
  return json({ ok: true });
};
```

**Next.js** — create `app/api/debug/route.ts`:
```typescript
import { NextRequest, NextResponse } from "next/server";
const SECRET = "GENERATED_SECRET";
const MAX = 500;
const logs: { ts: number; type: string; detail: unknown }[] = [];
export async function POST(req: NextRequest) {
  const e = await req.json();
  if (Array.isArray(e)) { logs.push(...e); while (logs.length > MAX) logs.shift(); }
  return NextResponse.json({ ok: true });
}
export async function GET(req: NextRequest) {
  if (req.nextUrl.searchParams.get("secret") !== SECRET) return new Response("", { status: 404 });
  const since = Number(req.nextUrl.searchParams.get("since") || 0);
  return NextResponse.json(since ? logs.filter(l => l.ts > since) : logs);
}
export async function DELETE(req: NextRequest) {
  if (req.nextUrl.searchParams.get("secret") !== SECRET) return new Response("", { status: 404 });
  logs.length = 0;
  return NextResponse.json({ ok: true });
}
```

**Express** — add to server file:
```javascript
const WEBTRACE_SECRET = "GENERATED_SECRET";
const wtLogs = [];
app.post("/api/v1/debug", express.json(), (req, res) => {
  if (Array.isArray(req.body)) { wtLogs.push(...req.body); while (wtLogs.length > 500) wtLogs.shift(); }
  res.json({ ok: true });
});
app.get("/api/v1/debug", (req, res) => {
  if (req.query.secret !== WEBTRACE_SECRET) return res.status(404).send("");
  const since = Number(req.query.since || 0);
  res.json(since ? wtLogs.filter(l => l.ts > since) : wtLogs);
});
app.delete("/api/v1/debug", (req, res) => {
  if (req.query.secret !== WEBTRACE_SECRET) return res.status(404).send("");
  wtLogs.length = 0; res.json({ ok: true });
});
```

### 4. Verify

Run: `curl -s "localhost:<PORT>/api/v1/debug?secret=<SECRET>"` — should return `[]`.

## Agent behavior

**After setup, tell the user:** "All done. Go ahead and test the site — when something breaks, tell me and I'll be able to tell you exactly what you did."

**Do NOT:**
- Show curl commands to the user
- Commit the changes (let the user decide)
- Show the secret to the user (you manage it)

**When the user says "check the webtrace logs" or similar:**
1. Read the secret from the endpoint file you created
2. Run: `curl -s "localhost:<PORT>/api/v1/debug?secret=<SECRET>"`
3. Analyze the JSON events and explain what happened
4. If needed, filter: `| jq '[.[] | select(.type == "error")]'`
5. Clear after: `curl -s -X DELETE "localhost:<PORT>/api/v1/debug?secret=<SECRET>"`

**To find the secret later:** read the server endpoint file and look for the `SECRET` constant.
