# Webtrace — Agent Setup Instructions

You are setting up webtrace, a browser-side event interceptor. It captures every user interaction (clicks, API calls, errors, navigation) and sends them to a server endpoint you can read.

There are exactly **two changes** to make:
1. Add the client snippet to the HTML `<head>`
2. Create a server endpoint to receive and serve events

## Step 1: Detect the framework

Look at the project's root files to determine the framework:

| File | Framework |
|------|-----------|
| `svelte.config.js` | SvelteKit |
| `next.config.js` or `next.config.ts` | Next.js |
| `nuxt.config.ts` | Nuxt |
| Express/Fastify/Hono in `package.json` | Node server |

## Step 2: Generate a secret

Generate a random secret string (16+ characters, alphanumeric). You will use this in the server endpoint. **Remember this secret** — you need it to read logs later.

## Step 3: Add the client snippet

Find the project's HTML shell and add this `<script>` tag inside `<head>`, before any application scripts:

- **SvelteKit**: `src/app.html`
- **Next.js**: `app/layout.tsx` (use `<Script strategy="beforeInteractive">`) or `pages/_document.tsx`
- **Nuxt**: `nuxt.config.ts` `app.head.script` or `app.html`
- **Vite (React/Vue)**: `index.html`
- **Express**: wherever HTML is served/templated

### The snippet

```html
<script>
(function(){
var q=[],fl=false,oF=window.fetch;
function log(t,d){d.page=location.pathname+location.search;q.push({ts:Date.now(),type:t,detail:d});sf();}
var ft;function sf(){if(!ft)ft=setTimeout(flush,300);}
function flush(){ft=null;if(!q.length||fl)return;fl=true;var b=q.splice(0);oF('/api/v1/debug',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(b)}).catch(function(){}).finally(function(){fl=false;});}
function ep(el){var p=[];while(el&&el!==document.body&&p.length<4){var s=el.tagName.toLowerCase();if(el.id)s+='#'+el.id;p.unshift(s);el=el.parentElement;}return p.join(' > ');}
window.fetch=function(input,init){var url=typeof input==='string'?input:(input instanceof Request?input.url:String(input));if(url.indexOf('/api/v1/debug')!==-1)return oF.apply(this,arguments);var method=(init&&init.method)||'GET';var body=init&&init.body;var parsed=null;try{if(body)parsed=JSON.parse(body);}catch(e){}log('fetch_req',{method:method,url:url,body:parsed});return oF.apply(this,arguments).then(function(res){var clone=res.clone();clone.json().then(function(data){log('fetch_res',{status:res.status,method:method,url:url,data:data});}).catch(function(){log('fetch_res',{status:res.status,method:method,url:url,data:'(non-json)'});});return res;}).catch(function(err){log('fetch_err',{method:method,url:url,error:String(err)});throw err;});};
document.addEventListener('click',function(e){var el=e.target;var text=(el.textContent||'').trim().substring(0,80);var href=null;var a=el.closest?el.closest('a'):null;if(a)href=a.getAttribute('href');var btn=el.closest?el.closest('button'):null;log('click',{el:ep(el),href:href,text:text,isButton:!!btn});},true);
document.addEventListener('change',function(e){var el=e.target;if(!el.tagName)return;var tag=el.tagName.toLowerCase();if(tag==='input'||tag==='select'||tag==='textarea'){var val=el.value;if(el.type==='password')val='***';else if(val&&val.length>80)val=val.substring(0,80)+'...';log('input',{el:ep(el),name:el.name||el.id||'',type:el.type||'',value:val});}},true);
document.addEventListener('submit',function(e){var form=e.target;var data={};try{var fd=new FormData(form);fd.forEach(function(v,k){data[k]=typeof v==='string'?v.substring(0,80):'(file)';});}catch(err){}log('submit',{action:form.action||'',method:form.method||'GET',el:ep(form),data:data});},true);
var oSI=Storage.prototype.setItem;Storage.prototype.setItem=function(k,v){log('ls_set',{key:k,value:typeof v==='string'&&v.length>60?v.substring(0,60)+'...':v});return oSI.apply(this,arguments);};
var oRI=Storage.prototype.removeItem;Storage.prototype.removeItem=function(k){log('ls_remove',{key:k});return oRI.apply(this,arguments);};
window.addEventListener('popstate',function(){log('nav',{type:'popstate',path:location.pathname+location.search});});
var oPS=history.pushState;history.pushState=function(){oPS.apply(this,arguments);log('nav',{type:'pushState',path:location.pathname+location.search});};
var oRS=history.replaceState;history.replaceState=function(){oRS.apply(this,arguments);log('nav',{type:'replaceState',path:location.pathname+location.search});};
window.addEventListener('error',function(e){log('error',{message:e.message,filename:e.filename,line:e.lineno,col:e.colno});});
window.addEventListener('unhandledrejection',function(e){log('error',{message:String(e.reason),type:'unhandledrejection'});});
document.addEventListener('visibilitychange',function(){log('visibility',{hidden:document.hidden});});
window.addEventListener('beforeunload',function(){flush();});
log('init',{url:location.href,referrer:document.referrer});
})();
</script>
```

## Step 4: Create the server endpoint

Create the appropriate endpoint file for the framework. Replace `YOUR_SECRET_HERE` with the secret you generated in Step 2.

### SvelteKit

Create `src/routes/api/v1/debug/+server.ts`:

```typescript
import { json, type RequestHandler } from "@sveltejs/kit";

const SECRET = "YOUR_SECRET_HERE";
const MAX_EVENTS = 500;
const logs: { ts: number; type: string; detail: unknown }[] = [];

export const POST: RequestHandler = async ({ request }) => {
  const entries = await request.json();
  if (Array.isArray(entries)) {
    logs.push(...entries);
    while (logs.length > MAX_EVENTS) logs.shift();
  }
  return json({ ok: true });
};

export const GET: RequestHandler = async ({ url }) => {
  if (url.searchParams.get("secret") !== SECRET) {
    return new Response("", { status: 404 });
  }
  const since = Number(url.searchParams.get("since") || 0);
  const filtered = since ? logs.filter((l) => l.ts > since) : logs;
  return json(filtered);
};

export const DELETE: RequestHandler = async ({ url }) => {
  if (url.searchParams.get("secret") !== SECRET) {
    return new Response("", { status: 404 });
  }
  logs.length = 0;
  return json({ ok: true });
};
```

### Next.js (App Router)

Create `app/api/debug/route.ts`:

```typescript
import { NextRequest, NextResponse } from "next/server";

const SECRET = "YOUR_SECRET_HERE";
const MAX_EVENTS = 500;
const logs: { ts: number; type: string; detail: unknown }[] = [];

export async function POST(request: NextRequest) {
  const entries = await request.json();
  if (Array.isArray(entries)) {
    logs.push(...entries);
    while (logs.length > MAX_EVENTS) logs.shift();
  }
  return NextResponse.json({ ok: true });
}

export async function GET(request: NextRequest) {
  const secret = request.nextUrl.searchParams.get("secret");
  if (secret !== SECRET) return new Response("", { status: 404 });
  const since = Number(request.nextUrl.searchParams.get("since") || 0);
  const filtered = since ? logs.filter((l) => l.ts > since) : logs;
  return NextResponse.json(filtered);
}

export async function DELETE(request: NextRequest) {
  const secret = request.nextUrl.searchParams.get("secret");
  if (secret !== SECRET) return new Response("", { status: 404 });
  logs.length = 0;
  return NextResponse.json({ ok: true });
}
```

### Express

Add to your server file:

```javascript
const DEBUG_SECRET = "YOUR_SECRET_HERE";
const MAX_EVENTS = 500;
const debugLogs = [];

app.post("/api/v1/debug", express.json(), (req, res) => {
  if (Array.isArray(req.body)) {
    debugLogs.push(...req.body);
    while (debugLogs.length > MAX_EVENTS) debugLogs.shift();
  }
  res.json({ ok: true });
});

app.get("/api/v1/debug", (req, res) => {
  if (req.query.secret !== DEBUG_SECRET) return res.status(404).send("");
  const since = Number(req.query.since || 0);
  res.json(since ? debugLogs.filter((l) => l.ts > since) : debugLogs);
});

app.delete("/api/v1/debug", (req, res) => {
  if (req.query.secret !== DEBUG_SECRET) return res.status(404).send("");
  debugLogs.length = 0;
  res.json({ ok: true });
});
```

### Fastify

```javascript
const DEBUG_SECRET = "YOUR_SECRET_HERE";
const MAX_EVENTS = 500;
const debugLogs = [];

fastify.post("/api/v1/debug", async (request) => {
  if (Array.isArray(request.body)) {
    debugLogs.push(...request.body);
    while (debugLogs.length > MAX_EVENTS) debugLogs.shift();
  }
  return { ok: true };
});

fastify.get("/api/v1/debug", async (request, reply) => {
  if (request.query.secret !== DEBUG_SECRET) return reply.status(404).send("");
  const since = Number(request.query.since || 0);
  return since ? debugLogs.filter((l) => l.ts > since) : debugLogs;
});

fastify.delete("/api/v1/debug", async (request, reply) => {
  if (request.query.secret !== DEBUG_SECRET) return reply.status(404).send("");
  debugLogs.length = 0;
  return { ok: true };
});
```

### Hono

```typescript
const DEBUG_SECRET = "YOUR_SECRET_HERE";
const MAX_EVENTS = 500;
const debugLogs: { ts: number; type: string; detail: unknown }[] = [];

app.post("/api/v1/debug", async (c) => {
  const entries = await c.req.json();
  if (Array.isArray(entries)) {
    debugLogs.push(...entries);
    while (debugLogs.length > MAX_EVENTS) debugLogs.shift();
  }
  return c.json({ ok: true });
});

app.get("/api/v1/debug", (c) => {
  if (c.req.query("secret") !== DEBUG_SECRET) return c.body("", 404);
  const since = Number(c.req.query("since") || "0");
  return c.json(since ? debugLogs.filter((l) => l.ts > since) : debugLogs);
});

app.delete("/api/v1/debug", (c) => {
  if (c.req.query("secret") !== DEBUG_SECRET) return c.body("", 404);
  debugLogs.length = 0;
  return c.json({ ok: true });
});
```

## Step 5: Verify

Start the dev server and run:

```bash
curl -s "localhost:<PORT>/api/v1/debug?secret=YOUR_SECRET_HERE"
```

It should return `[]`. Open the app in a browser, click around, then run the curl again — you should see events.

## Reading logs (use these commands when debugging)

```bash
# All events
curl -s "localhost:<PORT>/api/v1/debug?secret=YOUR_SECRET_HERE" | jq .

# Only API calls
curl -s "localhost:<PORT>/api/v1/debug?secret=YOUR_SECRET_HERE" | jq '[.[] | select(.type | startswith("fetch"))]'

# Only errors
curl -s "localhost:<PORT>/api/v1/debug?secret=YOUR_SECRET_HERE" | jq '[.[] | select(.type == "error")]'

# Only clicks
curl -s "localhost:<PORT>/api/v1/debug?secret=YOUR_SECRET_HERE" | jq '[.[] | select(.type == "click")]'

# User journey timeline
curl -s "localhost:<PORT>/api/v1/debug?secret=YOUR_SECRET_HERE" | jq '.[] | "\(.ts) [\(.type)] \(.detail.url // .detail.path // .detail.text // .detail.key // "")"'

# Clear logs before a new debugging session
curl -s -X DELETE "localhost:<PORT>/api/v1/debug?secret=YOUR_SECRET_HERE"
```

## Removing webtrace

1. Remove the `<script>` tag from the HTML `<head>`
2. Delete the server endpoint file (or the route lines if added to an existing file)

That's it. Two changes to undo.
