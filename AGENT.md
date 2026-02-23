# Webtrace — Agent Setup

Webtrace captures every browser interaction (clicks, API calls, errors, navigation) so you can see exactly what the user did. It runs as a proxy in front of the dev server — zero project file changes.

## What you're building

A Node.js proxy server inside `.webtrace/` that:

1. **Proxies all requests** to the real dev server (auto-detect port from the project's dev script or config)
2. **Injects the snippet below** into every HTML response (before `</head>`)
3. **Handles `/api/v1/debug`** — POST to receive events, GET to read them (requires secret), DELETE to clear (requires secret)
4. **Caps storage** at 500 events in memory. No dependencies — use only Node.js built-ins (`http`, `https`).

Generate a random secret for reading logs. Store it in the proxy code.

## Setup steps

1. Create `.webtrace/` directory
2. Add `.webtrace/` to `.git/info/exclude` (local gitignore, never committed)
3. Write the proxy server to `.webtrace/server.js`
4. Test it: start the project's dev server, then start the proxy, then `curl` the proxy port to verify HTML includes the snippet

## The snippet to inject

Inject this exact HTML before `</head>` in every HTML response from the upstream dev server:

```html
<!-- webtrace -->
<script>
(function(){var q=[],fl=!1,oF=window.fetch;function L(t,d){d.page=location.pathname+location.search;q.push({ts:Date.now(),type:t,detail:d});S()}var ft;function S(){ft||(ft=setTimeout(F,300))}function F(){ft=null;if(!q.length||fl)return;fl=!0;var b=q.splice(0);oF("/api/v1/debug",{method:"POST",headers:{"Content-Type":"application/json"},body:JSON.stringify(b)}).catch(function(){}).finally(function(){fl=!1})}function P(el){var p=[];while(el&&el!==document.body&&p.length<4){var s=el.tagName.toLowerCase();if(el.id)s+="#"+el.id;p.unshift(s);el=el.parentElement}return p.join(" > ")}window.fetch=function(i,o){var u=typeof i==="string"?i:i instanceof Request?i.url:String(i);if(u.indexOf("/api/v1/debug")!==-1)return oF.apply(this,arguments);var m=(o&&o.method)||"GET",bd=o&&o.body,ps=null;try{if(bd)ps=JSON.parse(bd)}catch(e){}L("fetch_req",{method:m,url:u,body:ps});return oF.apply(this,arguments).then(function(r){var c=r.clone();c.json().then(function(d){L("fetch_res",{status:r.status,method:m,url:u,data:d})}).catch(function(){L("fetch_res",{status:r.status,method:m,url:u,data:"(non-json)"})});return r}).catch(function(e){L("fetch_err",{method:m,url:u,error:String(e)});throw e})};document.addEventListener("click",function(e){var el=e.target,t=(el.textContent||"").trim().substring(0,80),h=null,a=el.closest?el.closest("a"):null;if(a)h=a.getAttribute("href");L("click",{el:P(el),href:h,text:t,isButton:!!(el.closest?el.closest("button"):null)})},!0);document.addEventListener("change",function(e){var el=e.target;if(!el.tagName)return;var tg=el.tagName.toLowerCase();if(tg==="input"||tg==="select"||tg==="textarea"){var v=el.value;if(el.type==="password")v="***";else if(v&&v.length>80)v=v.substring(0,80)+"...";L("input",{el:P(el),name:el.name||el.id||"",type:el.type||"",value:v})}},!0);document.addEventListener("submit",function(e){var f=e.target,d={};try{new FormData(f).forEach(function(v,k){d[k]=typeof v==="string"?v.substring(0,80):"(file)"})}catch(x){}L("submit",{action:f.action||"",method:f.method||"GET",el:P(f),data:d})},!0);var oSI=Storage.prototype.setItem;Storage.prototype.setItem=function(k,v){L("ls_set",{key:k,value:typeof v==="string"&&v.length>60?v.substring(0,60)+"...":v});return oSI.apply(this,arguments)};var oRI=Storage.prototype.removeItem;Storage.prototype.removeItem=function(k){L("ls_remove",{key:k});return oRI.apply(this,arguments)};window.addEventListener("popstate",function(){L("nav",{type:"popstate",path:location.pathname+location.search})});var oPS=history.pushState;history.pushState=function(){oPS.apply(this,arguments);L("nav",{type:"pushState",path:location.pathname+location.search})};var oRS=history.replaceState;history.replaceState=function(){oRS.apply(this,arguments);L("nav",{type:"replaceState",path:location.pathname+location.search})};window.addEventListener("error",function(e){L("error",{message:e.message,filename:e.filename,line:e.lineno,col:e.colno})});window.addEventListener("unhandledrejection",function(e){L("error",{message:String(e.reason),type:"unhandledrejection"})});document.addEventListener("visibilitychange",function(){L("visibility",{hidden:document.hidden})});window.addEventListener("beforeunload",function(){F()});L("init",{url:location.href,referrer:document.referrer})})();
</script>
```

## Agent behavior

**After setup, tell the user:** "All done. Go ahead and test the site at `localhost:<PROXY_PORT>` — when something breaks, tell me and I'll be able to tell you exactly what you did."

**Do NOT:**
- Modify any project files outside `.webtrace/`
- Commit anything
- Show curl commands or the secret to the user

**When the user says "check the logs", "what happened", "something broke", or similar:**
1. Read the secret from `.webtrace/server.js`
2. Run: `curl -s "localhost:<PROXY_PORT>/api/v1/debug?secret=<SECRET>"`
3. Analyze the events and explain what happened
4. Clear after: `curl -s -X DELETE "localhost:<PROXY_PORT>/api/v1/debug?secret=<SECRET>"`
