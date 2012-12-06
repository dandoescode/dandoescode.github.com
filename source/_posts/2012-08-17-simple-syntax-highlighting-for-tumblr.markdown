---
layout: post
title: "Simple Syntax Highlighting for Tumblr"
date: 2012-08-17 00:00
comments: true
categories: [JS]
---

I was looking to add syntax highlighting to my blog here, and figured I would use the ever-handy (and simple) [Google Code Prettifier](http://code.google.com/p/google-code-prettify/),

It ended up being really simple, and I added a bit of JavaScript so I don't have to manually write the `<pre>` tags and add the `prettyprint` class to code blocks.  This is helpful when you are writing your posts in Markdown.

Two simple steps:

1.  Either upload the `prettify.css` and the `prettify.js` files to Tumblr and link to them, or you can link to them directly in their repository:

``` html
    <link rel="stylesheet" type="text/css" href="http://google-code-prettify.googlecode.com/svn/trunk/src/prettify.css" />
    <script src="http://google-code-prettify.googlecode.com/svn/trunk/src/prettify.js"></script>
```

2.  Insert the following code somewhere in your theme:


``` html
    <script type="text/javascript">
        $(function() {
            var run = false;
            $("pre code").parent().each(function() {
                run = true;
                $(this).addClass('prettyprint');
            });
            if (run) {
                prettyPrint();
            }
        });
    </script>
```

_Note: Make sure you put the stylesheet link above your Theme's CSS._

_Note 2: This assumes you have jQuery already loaded._
