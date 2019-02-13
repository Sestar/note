# Highlight.js

[![Build Status](https://travis-ci.org/highlightjs/highlight.js.svg?branch=master)](https://travis-ci.org/highlightjs/highlight.js)

Highlight.js is a syntax highlighter written in JavaScript. It works in
the browser as well as on the server. It works with pretty much any
markup, doesn’t depend on any framework, and has automatic language
detection.

## Getting Started

The bare minimum for using highlight.js on a web page is linking to the
library along with one of the styles and calling
[`initHighlightingOnLoad`][1]:

```html
<link rel="stylesheet" href="/path/to/styles/default.css">
<script src="/path/to/highlight.pack.js"></script>
<script>hljs.initHighlightingOnLoad();</script>
```

&nbsp;
|    a    |       b       |      c     |
|:-------:|:------------- | ----------:|
|   居中  |     左对齐    |   右对齐   |
|=========|===============|============|
&nbsp;

&nbsp;

$$ x \href{why-equal.html}{=} y^2 + 1 $$

$$ x = {-b \pm \sqrt{b^2-4ac} \over 2a}. $$
&nbsp;

&nbsp;

```flow
st=>start: Start
op=>operation: Your Operation
cond=>condition: Yes or No?
e=>end
st->op->cond
cond(yes)->e
cond(no)->op
```
&nbsp;
