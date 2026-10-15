---
title: "Soma"
template: "content.html"
date: "2026-02-05"
tags: ["odin", "http", "cli"]
categories: ["web"]
desc: "A lightweight SSG (builds this site)"
github_url: "https://github.com/floeckdev/soma"
sub_title: "Rewriting a Static Site Generator in Odin"
---

Soma is a lightweight static site generator (SSG) that I originally wrote in Python and later rewrote in Odin. It also happens to be the software responsible for building this website.

The rewrite started as an opportunity to explore a language that was unfamiliar to me, particularly its approach to memory management, data structures and systems programming.

Along the way, I found myself reconsidering some of my assumptions about software design, learning more about how web servers operate and discovering that seemingly simple features can involve surprising amounts of work.

In the following, I talk about some of the decisions I found interesting, what I learned, and what I'd approach differently.

## 1. Why Odin?

I ended up choosing Odin for the rewrite for a few reasons. Mainly, I wanted to become more familiar with lower-level programming while still enjoying some modern language features. Odin appealed to me for its clean syntax, type inference, built-in string views and sensible approach to memory management through explicit `allocators` and `defer`. Rewriting Soma rather than starting something new also meant I could focus on learning the language while working with a problem I already understood.

Coming from Python, one of the biggest adjustments was having to think more carefully about memory allocation, ownership and lifetime. Although I don't handle this well currently in Soma, it was something at the very least I needed to consider.

I also wanted to understand more of the machinery involved in serving a website. Writing my own HTTP server, rather than relying entirely on an existing framework, gave me an opportunity to explore what it actually means to serve content. In Python I only needed to write a line:

```python
python3 -m http.serve
```

And this was all you really needed. Whereas in Odin I needed to handle a lot more; sockets, accepting clients and handling headers. Everything was there for you to play around with! Very cool but also daunting.

## 2. Working with the filesystem and memory

### 2.1 Walking directories

One small discovery that stuck with me involved Odin's filesystem walker.

While reading the documentation and exploring the language server's autocomplete suggestions, I came across a useful function:

```
os.walker_skip_dir(&w)
```

During a build, Soma traverses the site's source directory to discover content. However, the generated build directory may also exist inside that directory. Rather than entering the build directory and checking every file to determine whether it should be ignored, I could skip the entire directory before descending into it.

And look, this is probably not that significant to most of you. It's a small optimisation, but one that made the intent of the code clearer and eliminated unnecessary filesystem traversal. But I still found it really interesting; the fact that people that built these tools and functions thought about many use cases users would need!

Interestingly, this **wasn't something any of the LLM-generated suggestions I'd encountered had proposed.** This is likely something to do with Odin being relatively sparse part of their training data. In any case, it was a useful reminder of the value of exploring a language's standard library myself

### 2.2 Memory allocation and lifetime

Memory management was one of the biggest differences between the original Python version and the Odin rewrite.

Throughout development, I frequently relied on `context.allocator`. This was convenient, particularly while I was still becoming familiar with the language, but it also meant I wasn't always being deliberate about allocation lifetimes. In hindsight, I definitely should have thought more about allocation strategy earlier in the project.

One possible improvement would be using arenas for particular operations, such as a complete build or intermediate rendering stages, where groups of allocations could be discarded together. The next time I work on this project, this is one of the first things I will be addressing.

## 3. Building the templating system

### 3.1 Rebuilding rather than modifying

One interesting decision came up while implementing template inheritance.

When resolving a child template, I needed to replace blocks in the parent template's abstract syntax tree (AST) with the corresponding content from the child.

One option was to modify the existing array of nodes directly, inserting and removing elements as necessary.

For example, replacing part of:

```
[block_1, block_4, block_5]
```

with:

```
[block_1, block_2, block_3, block_4, block_5]
```

Instead, I opted to construct a new sequence of nodes during traversal. This avoided writing additional helpers for shifting and inserting elements into existing arrays. More importantly, I found the resulting code easier to follow and maintain.

It was a useful example of how modifying an existing structure isn't necessarily simpler than rebuilding it, even if rebuilding initially sounds less efficient.

### 3.2 Temporary data versus the registry

Another decision arose while implementing loop rendering in `_render_for`.

Soma maintains a registry of page-level information that can be referenced during rendering. However, loops also introduce temporary variables representing the current item.

```
render_for(node, page, scope):
    create child scope
    parse loop variable and collection
    resolve collection

    if collection is undefined:
        report error

    for each item in collection:
        bind item to loop variable in child scope
        render loop body using child scope
```

Initially, I wondered whether these values should be stored in the registry as well. After some time, I realised they served a different purpose. Registry entries represent persistent page-level information, whereas loop variables are temporary/ephemeral and only meaningful within their rendering scope.

Keeping them in local scope avoided unnecessarily expanding the registry and better reflected their intended lifetime. This also highlighted the importance of distinguishing data by its purpose rather than treating every accessible value as belonging to the same structure.

### 3.3. When is a helper worth writing?

A recurring question throughout the rewrite was whether small pieces of repeated logic justified their own helper functions. I often found myself choosing between creating another abstraction or leaving a small amount of duplication in place. I generally leaned towards keeping related logic together, particularly when introducing a helper would require jumping between functions to understand a relatively simple operation.

I don't think I've found the ideal balance yet. Repetition can make code harder to maintain, but abstraction also carries a cost, especially when it obscures the underlying behaviour.

## 4. Frontmatter, data models and errors

### 4.1 Keeping frontmatter simple

Soma uses a YAML-inspired frontmatter format rather than implementing the entire YAML specification.

It supports a small set of types:

```
Strings   "Hello, world."
Arrays    ["Hello", ", ", "world."]
Integers  0-9
Booleans  true, false
Dates     2026-12-12
```

This covers the information I currently need to describe pages, such as their titles, templates, dates and tags.

The main reason I decided against full YAML support was discovering just how complicated YAML parsing can be. I came across a [Twitter post by gingerBill](https://x.com/TheGingerBill/status/1861401720581922832), the creator of Odin, discussing the possibility of adding YAML support to the language. His comment *“its header file is over 9000 lines of code”* was enough to convince me that implementing a small subset of YAML was more than sufficient for Soma's needs.


### 4.2 Defining types too early

During development, I noticed a tendency to introduce structures before I fully understood how the information would be used. Dates were one example. I began implementing my own representation for parsed dates, only to discover later that Odin already provided useful date-related functionality. In retrospect, I could have spent more time understanding the available standard library before designing my own solution.

I also found myself tempted to reuse existing structures in places where they weren't quite appropriate. For example, during the final stages of building a site, I considered reusing my Page type because it already contained much of the information I needed. However, several fields would have been unused or meaningless.

I eventually introduced a separate Build_Context type. Although this meant having another structure, it more accurately represented the information required for that operation.

These experiences made me reconsider when to introduce a new type and when to keep a simpler representation until the requirements become clearer.

### 4.3 Providing useful error messages

One problem I encountered was maintaining enough context to produce helpful error messages.

When a failure occurred deep within frontmatter parsing or template processing, the function detecting the error didn't necessarily know which source file was being processed. I considered introducing shared state to track the current file, avoiding the need to pass diagnostic information through several layers of functions.

However, global state introduces its own problems, particularly when operations become more complex or need to run independently.

This is something I will likely revisit at a later date.

## 5. Markdown and the little surprises

### 5.1 Code blocks and syntax highlighting

Towards the end of the rewrite, I discovered that styling Markdown code blocks wasn't quite as straightforward as I'd expected.

Soma uses `md4c` to convert Markdown into HTML. For a fenced code block, the output looks roughly like this:

```html
<pre><code class="language-odin">main :: proc() {
    fmt.println(&quot;hi&quot;) // greet
}
</code></pre>
```

This is perfectly reasonable HTML, but it doesn't distinguish keywords, comments or other syntax elements. Initially, I explored post-processing these blocks myself. This involved traversing the generated HTML, identifying code blocks and inserting additional markup. However, syntax highlighting quickly becomes complicated once different programming languages and their individual syntax rules are involved.

Rather than maintaining my own highlighting implementation, I eventually decided to use Highlight.js. It handles the language-specific work, while Soma remains responsible for generating the page itself.

This was a good reminder that not every problem benefits from a custom implementation, particularly when the problem sits outside the project's main purpose.

### 5.2 Typography and quotation marks

Another small surprise involved quotation marks.

I wanted Markdown text to support typographic quotation marks, such as converting straight quotes into curly opening and closing quotes where appropriate. However, to my surprise, md4c doesn't automatically perform this transformation.

I considered adding an HTML post-processing step or switching to a Markdown implementation that supported this behaviour.. but for now, I've left it alone. It wasn't worth introducing additional processing complexity for what is largely a presentational detail.

Interestingly, both this and the code highlighting issue showed me how much work can sit behind seemingly trivial formatting features.

## 6. Reflections

Rewriting Soma in Odin has been a useful exercise in understanding what happens beneath the abstractions I was accustomed to working with in Python.

I've had to think more carefully about memory, data lifetime, filesystem operations and how to organise internal structures without introducing unnecessary complexity.

It's also challenged my tendency to design abstractions too early. Several times, the simpler solution only became obvious after I'd spent time exploring something more complicated.

Perhaps the biggest lesson has been that knowing what not to implement is just as important as knowing how to implement it.

Soma remains a relatively small project, and that's intentional. I want it to stay simple enough that I can understand and maintain it myself, while remaining useful for building this website.

