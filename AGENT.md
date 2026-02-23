# Webtrace — Agent Setup

Webtrace captures every browser interaction (clicks, API calls, errors, navigation) so you can see exactly what the user did.

## Steps

**All created files and directories must include `.webtrace` in the name** so a single gitignore pattern covers everything.

1. **Add `*webtrace*` to `.gitignore`**

2. **Write `.webtrace.js`** to the project's static/public asset directory so it's served directly by the dev server. Contents are in "The JavaScript file" section below.

3. **Add a script tag** to the project's HTML shell: `<script src="/.webtrace.js"></script>` inside `<head>`, before app scripts.

4. **Create a server endpoint** at the URL path `/.webtrace` (the route directory should be named `.webtrace`). It handles:
   - `POST` — receives a JSON array of events, appends to an in-memory array, caps at 500 entries (FIFO)
   - `GET` — returns the events array as JSON. Supports `?since=<timestamp>` query param to return only events with `ts` greater than the given value (milliseconds since epoch)
   - `DELETE` — clears the array

5. **REQUIRED — Update the project's agent instructions file** so future agents know webtrace is installed. Find the project's instruction file (`CLAUDE.md`, `.cursorrules`, or `.github/copilot-instructions.md` — whichever exists) and append the following block. Replace `<DEV_PORT>` with the project's actual dev server port:

```
## Webtrace (browser session logging)

This project has webtrace installed. When the user says "check the logs", "what happened", or "something broke", read browser events:
- Read all: curl -s localhost:<DEV_PORT>/.webtrace
- Read recent (last 5 min): curl -s "localhost:<DEV_PORT>/.webtrace?since=$(($(date +%s)*1000 - 300000))"
- Clear: curl -s -X DELETE localhost:<DEV_PORT>/.webtrace
Each event has a `ts` field (ms since epoch). Use `?since=` to skip old events. Do not show curl commands to the user. Analyze the events and explain what happened.
```

**Do not skip step 5.** Without it, future agents won't know webtrace exists.

## The JavaScript file

Write this exact content to the static `.webtrace.js` file:

```javascript
(function(){var q=[],fl=!1,oF=window.fetch;function L(t,d){d.page=location.pathname+location.search;q.push({ts:Date.now(),type:t,detail:d});S()}var ft;function S(){ft||(ft=setTimeout(F,300))}function F(){ft=null;if(!q.length||fl)return;fl=!0;var b=q.splice(0);oF("/.webtrace",{method:"POST",headers:{"Content-Type":"application/json"},body:JSON.stringify(b)}).catch(function(){}).finally(function(){fl=!1})}function P(el){var p=[];while(el&&el!==document.body&&p.length<4){var s=el.tagName.toLowerCase();if(el.id)s+="#"+el.id;p.unshift(s);el=el.parentElement}return p.join(" > ")}window.fetch=function(i,o){var u=typeof i==="string"?i:i instanceof Request?i.url:String(i);if(u.indexOf("/.webtrace")!==-1)return oF.apply(this,arguments);var m=(o&&o.method)||"GET",bd=o&&o.body,ps=null;try{if(bd)ps=JSON.parse(bd)}catch(e){}L("fetch_req",{method:m,url:u,body:ps});return oF.apply(this,arguments).then(function(r){var c=r.clone();c.json().then(function(d){L("fetch_res",{status:r.status,method:m,url:u,data:d})}).catch(function(){L("fetch_res",{status:r.status,method:m,url:u,data:"(non-json)"})});return r}).catch(function(e){L("fetch_err",{method:m,url:u,error:String(e)});throw e})};document.addEventListener("click",function(e){var el=e.target,t=(el.textContent||"").trim().substring(0,80),h=null,a=el.closest?el.closest("a"):null;if(a)h=a.getAttribute("href");L("click",{el:P(el),href:h,text:t,isButton:!!(el.closest?el.closest("button"):null)})},!0);document.addEventListener("change",function(e){var el=e.target;if(!el.tagName)return;var tg=el.tagName.toLowerCase();if(tg==="input"||tg==="select"||tg==="textarea"){var v=el.value;if(el.type==="password")v="***";else if(v&&v.length>80)v=v.substring(0,80)+"...";L("input",{el:P(el),name:el.name||el.id||"",type:el.type||"",value:v})}},!0);document.addEventListener("submit",function(e){var f=e.target,d={};try{new FormData(f).forEach(function(v,k){d[k]=typeof v==="string"?v.substring(0,80):"(file)"})}catch(x){}L("submit",{action:f.action||"",method:f.method||"GET",el:P(f),data:d})},!0);var oSI=Storage.prototype.setItem;Storage.prototype.setItem=function(k,v){L("ls_set",{key:k,value:typeof v==="string"&&v.length>60?v.substring(0,60)+"...":v});return oSI.apply(this,arguments)};var oRI=Storage.prototype.removeItem;Storage.prototype.removeItem=function(k){L("ls_remove",{key:k});return oRI.apply(this,arguments)};window.addEventListener("popstate",function(){L("nav",{type:"popstate",path:location.pathname+location.search})});var oPS=history.pushState;history.pushState=function(){oPS.apply(this,arguments);L("nav",{type:"pushState",path:location.pathname+location.search})};var oRS=history.replaceState;history.replaceState=function(){oRS.apply(this,arguments);L("nav",{type:"replaceState",path:location.pathname+location.search})};window.addEventListener("error",function(e){L("error",{message:e.message,filename:e.filename,line:e.lineno,col:e.colno})});window.addEventListener("unhandledrejection",function(e){L("error",{message:String(e.reason),type:"unhandledrejection"})});document.addEventListener("visibilitychange",function(){L("visibility",{hidden:document.hidden})});window.addEventListener("beforeunload",function(){F()});L("init",{url:location.href,referrer:document.referrer})})();
```

## Agent behavior

**After setup, tell the user:** "All done. Go ahead and test the site — when something breaks, tell me and I'll be able to tell you exactly what you did."

**Do NOT:**
- Commit anything
- Show curl commands to the user

**When the user says "check the logs", "what happened", "something broke", or similar:**
1. Fetch recent events (last 5 minutes): `curl -s "localhost:<DEV_PORT>/.webtrace?since=$(($(date +%s)*1000 - 300000))"`
2. If that's empty or you need more context, fetch all: `curl -s localhost:<DEV_PORT>/.webtrace`
3. Analyze the events and explain what happened
4. Clear after if needed: `curl -s -X DELETE localhost:<DEV_PORT>/.webtrace`
