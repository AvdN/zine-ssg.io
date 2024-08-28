---
.title = "SuperHTML Basics",
.description = "",
.author = "Loris Cro",
.layout = "page.shtml",
.date = @date("2023-06-16T00:00:00"),
.draft = false,
---

## Intro

**Make sure to read [Scripty Basics]($link.page('docs/scripty')) first.**

SuperHTML is a templating language for HTML that uses Scripty to express templating logic.

There is [a](https://gohugo.io/templates/introduction/) [borderline](https://mustache.github.io/) [infinite](https://handlebarsjs.com/) [number](https://jinja.palletsprojects.com/) [of](https://github.com/ruby/erb) HTML templating languages, but the majority of those used by static site generators are a form of macro system that preprocesses the HTML code as unstructured text.

This "macro" approach has the upside of making it possible to use the same templating language for other file formats (you *can* use Jinja to template CSV files, for example), but it also has a variety of downsides.

The most important downside is that it makes it unnecessarily easy to generate HTML in an unstructured manner:

- making it harder for homans to understand what the templating logic is doing
- making it easier for humans to output malformed html
- making it prohibitively complex for tooling to catch mistakes statically (i.e. in your editor)

The following is an extremely tame example of what is possible with curly braced templating languages:
```html
<a href="baz">
{{ if .Foo }}
  </a><a href="foo">
{{ end }}
</a>
```

This example can't be recreated faithfully in SuperHTML because it operates within the limits of well-formed HTML.

### Design Goals
SuperHTML is designed to help you write correct HTML by catching mistakes early.

It is [my](https://github.com/kristoff-it) belief that people resort to using JSX for creating static websites even when they don't really have any frontend SPA (Single Page Application) need because the experience of editing vanilla HTML is objectively worse.

To this point: as part of my work to create SuperHTML, I wrote an HTML parser from scratch that then I used to implement a [language server for HTML](https://github.com/kristoff-it/superhtml). 

While doing so I discovered that by spring of 2024 no other HTML language server that reports syntax errors exists. Some proprietary editors do have support for reporting HTML syntax errors, and one HTML language server does exist; the one that comes with VSCode (also used by other editors), but it doesn't report errors for malformed HTML.

## File Extension
SuperHTML files have a `.shtml` file extension.


## Scripting Attributes
The main way of expressing logic in SuperHTML is by giving Scripty expressions to HTML attributes.

Any HTML attribute can be scripted (with some restrictions explained later in this page).

***template.shtml***
```superhtml
<div class="$page.title.len().gt(25).then('long-title')">...</div>
```

***output***
```html
<div class="long-title">...</div>
```

## [Logic Attributes]($heading.id('attributes'))

SuperHTML defines a list of special attributes that drive templating logic.

### `:text` 
The `:text` attribute sets the content of an element to the result of its Scripty expression. The value is HTML-escaped before printing.

The expression must evaluate to either 
[`String`]($link.page('docs/superhtml/scripty').ref('String')) 
or
[`Int`]($link.page('docs/superhtml/scripty').ref('Int')).


Requires the element to have an empty body.

***template.shtml***
```superhtml
<div :text="$page.title"></div>
```

***output***
```html
<div>Page Title</div>
```

### `:html`
Same as `:text`, but it doesn't escape HTML. Avoid using it with untrusted data.

***template.shtml***
```superhtml
<div :html="$page.content()"></div>
```

***output***
```html
<div>
  <h1>Page Title</h1>
  <p>Lorem Ipsum</p>
</div>
```


### `:if`
Toggles the body of an element based on the value of its Scripty expression. 

The expression must evaluate to either 
[`Bool`]($link.page('docs/superhtml/scripty').ref('Bool')) 
or
[`?any`]($link.page('docs/superhtml/scripty').ref('any')).

The second kind of value (`?any`) is an *optional* value. You can get an optional value for example by trying to access a custom field in a page frontmatter, or when trying to access the `next` page.

Optional values can either be present or missing. 

When the value is missing, the `:if` attribute evaluates to `false` and the body of the element is elided.

When the value is present, it becomes available as the 
[`$if`]($link.page('docs/superhtml/scripty')) 
Scripty global variable.

***template.shtml***
```superhtml
<div :if="$page.draft.not()">
  <span>won't be printed</span>
</div>
<div :if="$page.next?()">
  Next: <span :text="$if.title"></span>
</div>
```

***output***
```html
<div></div>
<div>
  Next: <span>Other Page Title</span>
</div>
```

### `:loop`
Evaluates the body of an element once for every element in the value of its Scripty expression.

The expression must evaluate to `[any]`.

The iterator becomes available as the 
[`$loop`]($link.page('docs/superhtml/scripty')) 
Scripty global variable.

***template.shtml***
```superhtml
<ul :loop="$page.subpages()">
  <li>
    <a href="$loop.it.link()" :text="$loop.it.title"></a>
  </li>
</div>
```

***output***
```html
<ul>
  <li> <a href="page1/">Page 1</a> </li>
  <li> <a href="page2/">Page 2</a> </li>
  <li> <a href="page2/">Page 3</a> </li>
</ul>
```

In nested loops the inner iterator will shadow the outer `$loop`, but you can use 
[`$loop.up()`]($link.page('docs/superhtml/scripty').ref('Iterator'))
to access it.

## Elements
SuperHTML also defines a small number of custom elements.

### `<ctx>`
This element supports all the logic attributes but has the property that its start/end tags will not be rendered.

One use of this property is to put text at a specific location of an element:

***template.shtml***
```superhtml
<div>
  Created by: <ctx :text="$page.author"></ctx>
</div>
```

***output***
```html
<div>
  Created by: Loris Cro
</div>
```

A second important property of `<ctx>` is that any attribute defined on it will become available as a field of the 
[`$ctx`]($link.page('docs/superhtml/scripty').ref('Ctx'))
Scripty global variable.


***template.shtml***
```superhtml
<ctx about="$site.page('about')">
  <a href="$ctx.about.link()" :text="$ctx.about.title"></a>
</ctx>
```

***output***
```html
<a href="about/">About</a>
```

Nested instances of `<ctx>` will have all their attributes merged in `$ctx` (i.e. there is no `up` function). 

Shadowing is not allowed for `<ctx>` attributes and will be reported as an error if two definitions of the same attribute are active in the same scope.

### `<super>`
Used to define template extension hierarchies, see the next section for more information.

### `<extend>`
Used to define template extension hierarchies, see the next section for more information.

## Extending templates

Let's continue the example from the previous section where we try to collect all common boilerplate from `homepage.html` and `page.html`.

***`layouts/templates/base.shtml`***
```superhtml
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8">
    <title id="title"><super></title>
  </head>
  <body id="main">
    <super>
  </body>
</html>
```

***`layouts/homepage.shtml`***
```superhtml
<extend template="base.html">

<title id="title" var="$site.title"></title>

<body id="main">
  <h1 var="$page.title"></h1>
  <div var="$page.content"></div>
</body>
```

***`layouts/post.shtml`***
```superhtml
<extend template="base.html">

<title id="title" var="$site.title"></title>

<body id="main">
  <h1>Blog</h1>
  <h2 var="$page.title"></h2>
  <h3>by <span var="$page.author"></span></h3>
  <h4>Posted on: <span var="$page.date.format('January 02, 2006')"></span></h4>
  <div var="$page.content"></div>
</body>
```

Let's analyze what we just saw:

- `layouts/templates/base.shtml` now contains all the main HTML boilerplate and has two `<super>` tags: one inside `<title>` and one inside `<body>` (both of which have also gained an `id` attribute).

- both layouts now start with an `<extend>` tag and have lost their original structure (since it was collected in `base.html`), keeping only the parts that each defines differently than the other.

### `<extend>`
When a layout wants to extend a template, it must declare at the very top the template name using the extend tag, like so: 

```superhtml
<extend template="foo.html">
```
A template that extends another won't have a normal HTML structure, but rather it will be a list of HTML elements that will replace a correspoding super tag from the base template they're extending.


### `<super>`

The super tag defines an extension point in a template. The direct parent of a super tag must have an `id` attribute.

**Each top level element in a template that extends another must correspond to a super tag in the template being extended.**

Let's call the "template that extends another" the **super template** (imagine that placing `<super>` in the code is like calling a *super template* for help).

Since super tags don't have any id of their own, **the *super template* uses the same tag and `id` of the parent element of each super tag to match its content with the correct super tag**.

***template***
```superhtml
<title id="title"><super> - Sample Site</title>
```

***layout***
```superhtml
<title id="title">Home</title>
```

***evaluates to***
```html
<title id="title">Home - Sample Site</title>
```

#### Why not just give `id`s to super tags?

Matching via parent elements is admittedly an uncommon choice, but it has some very real upsides over the alternatives.

Let's define a block in a curly brace templating language (mustache, jinja, hugo, etc):

***hugo***
```html
{{ define "main" }}
  <p> Hello World </p>
{{ end }}
```

This is the equivalent of a top-level element in a Zine template that extends another template, but it has a critical disadvantage: **you know nothing about where the content will be put**.

Unless you have perfect recollection of what "main" is, you won't know if your content should be framed in a container element or not. In fact, all the following declarations in a hugo base template are possible:

***ok***
```html
<!DOCTYPE html>
<html>
  <head></head>
  <body>
    {{ block "main" . }}{{ end }}
  </body>
</html>
```
***oops, needed a `<body>` wrapper***
```html
<!DOCTYPE html>
<html>
    <head></head>
    {{ block "main" . }}{{ end }}
</html>
```
***oops, too many wrappers***
```html
<!DOCTYPE html>
<html>
  <head></head>
  <body>
    <p>{{ block "main" . }}{{ end }}</p>
  </body>
</html>
```

Contrast this with our previous example:

***`layouts/homepage.html`***
```html
<extend template="base.html"/>

<title id="title" var="$site.title"></title>

<body id="main">
  <h1 var="$page.title"></h1>
  <div var="$page.content"></div>
</body>
```
By looking at the *super template* we know that we are putting content directly into the `body` element of the template we are extending.

In fact if we were to make a mistake and define "main" as a `div` element in our layout, we would get a compile error:

***shell***
```
$ zig build

---------- MISMATCHED BLOCK TAG ----------
The super template defines a block that has the wrong tag.
Both tags and ids must match in order to avoid confusion
about where the block contents are going to be placed in
the extended template.

note: super template block tag:
(post.html) layouts/post.html:19:2:
    <div id="main">
     ^^^

note: extended template block defined here:
(base.html) layouts/templates/base.html:16:4:
    <body id="main">
     ^^^^
trace:
    layout `post.html`,
    content `about.md`.
```

So, if we were to give `id`s to super tags, we would have to introduce another
fake html tag and we would end up with the same problem that curly brace templating 
languages have today.

***where would this go?***
```html
<content id="main">
  <p> Hello World </p>
</content>
```

Repeating the parent element is a form of ***typing*** for your templates, in 
a sense.

## Extension chains

In the previous section we saw how a template can extend another. We are now 
going to see how longer extension chains look like.

At this point it's also useful to think of templates as documents that can do two things:

- Define an interface by using `<super>`.
- Fulfill an interface by extending another template and definig top-level elements that match corresponding blocks in the template they extend.

Let's see an example: 

***`layouts/templates/base.shtml`***
```superhtml
<!DOCTYPE html>
<html>
  <head id="head">
    <meta charset="UTF-8">
    <title id="title"><super> - My Blog</title>
    <super>
  </head>
  <body id="body">
    <super>
  </body>
</html>
```
***`layouts/templates/with-menu.shtml`***
```superhtml
<extend template="base.html">

<title id="title"><super></title>

<head id="head">
  <script>console.log("Hello World!");</script>
</head>

<!-- at the top level only comments 
     and block definitions are allowed -->
<body id="body">
    <nav>
        <a>Home</a>
        <a>About</a>
    </nav>
    <div id="content">
        <super>
    </div>
</body>
```
***`layouts/page.shtml`***
```superhtml
<extend template="with-menu.html">

<title id="title" var="$page.title"></title>

<div id="content" var="$page.content"></div>
```

Note how `with-menu.shtml` is both fulfilling the *interface* (ie the extension points) of `base.shtml` and at the same time it's creating new ones for another *super template* to fulfill in turn.


Let's see now how attributes like `var` work.

## SuperHTML Template Logic

SuperHTML templates implement logic via special attributes that have semantic meaning for the templating language. The values given to those attributes are [Scripty](https://github.com/kristoff-it/scripty) expressions.

Example markdown file:

***`content/foo/index.md`***
```ziggy
---
.title = "My Post",
.date = @date("2020-07-06T00:00:00"),
.author = "Sample Author",
.draft = false,
.layout = "post.html",
.tags = ["tag1", "tag2", "tag3"],
--- 
The content
```


#### `var`
Prints the contents of a Scripty variable.

***`layouts/post.html`***
```html
<span var="$page.title"></span>  
```

***`output`***
```html
<span>My Post</span>
```


#### `if`
Toggles the body of an element based on a condition.


***`layouts/post.html`***
```html
<div if="$page.title.len().eq(1)" id="foo">
    <b>Won't be rendered</b>
</div>  
```

***`output`***
```html
<div id="foo"></div>
```

#### `loop`
Repeats the body of an element based on a condition. Inside an element with a `loop` attribute, `$loop` becomes available.


***`layouts/post.html`***
```html
<ul loop="$page.tags" id="tags">
   <li var="$loop.it"></li>
</ul>  
```

***`output`***
```html
<ul id="tags">
   <li>tag1</li>
   <li>tag2</li>
   <li>tag3</li>
</ul>  
```

#### `inline-loop`
Repeats the **entire** element based on a condition. Inside an element with a `inline-loop` attribute, `$loop` becomes available.

***`layouts/post.html`***
```html
<div inline-loop="$page.tags" var="$loop.it"></div>
```

***`output`***
```html
<div>tag1</div>
<div>tag2</div>
<div>tag3</div>
```


## Assets & Scripty
You can also use Scripty to add logic to normal attributes.

One notable example is linking to assets.

```html
<img src="$site.asset('logo.png').link()">
<link rel="stylesheet" type="text/css" href="$site.asset('style.css').link()">
```


## Next Steps

- [Learn more about Zine's Asset System](assets/)
- [Read the Scripty Reference Documentation](scripty/)



## Layout vs Template
Throughout this document (and in error messages) you will see a distinction being made between 'layouts' and 'templates'.

We'll see in a bit how templates extend each other by defining "placeholders" that can be then filled out by other templates. Layouts are basically the final link of a "chain" of templates that extend one another.

In Zine a layout is a template that can be fully evaluated to a complete HTML file
(i.e. it has no "extension placeholders" left).

The special name for those templates is used because, unike other templates, they define a final, complete *layout* for a set of pages of the same kind.
