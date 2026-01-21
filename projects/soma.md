---
title: "soma"
template: "content"
date: "2026-01-19"
tags: ["python", "ssg"]
categories: ["web"]
desc: "A lightweight SSG (built this site)"
github_url: "https://github.com/floeckdev/soma"
rank: 1
---

## Introduction

A static site generator (SSG) is a tool that transforms structured content (such as Markdown) into HTML. This greatly reduces the time and effort required to maintain a static site. For more information, see [this article.](https://www.cloudflare.com/en-au/learning/performance/static-site-generator/)

The motivation for this project really stemmed from my disdain in using others. Previously, I had used [Jekyll](https://jekyllrb.com/), and then [Zola](https://www.getzola.org/), but I found the features overkill for my needs. This lead me to creating my own.

## How Soma Works

***Soma*** is discovery based and category agnostic. The only frontmatter required in each page is the title and template. The tool generates a few basic templates and site content to get you started, showcasing the basic layout expected so you can begin adjusting how you like. Any additional frontmatter (such as date) is accessible in the templates. Date being tricky as it is is supported natively and the default templates show how they can be used.

Below I've included a basic visualisation of what site structure is required for ***Soma*** to work. As you can see, it follows a fairly simple convention of root pages existing in the root dir and categories having their own index.md.

```python
new-soma/                   
    index.md            # Required
    contact.md          
    blog/               
        index.md        # Required    
        first-post.md   
    projects/           
        index.md        # Required  
        first-proj.md
```

An important feature of the SSG was the ability for templates to be able to automatically access items for each category. For instance, the blog index page should be able to show all blog posts (slices in future). ***Soma*** handles this for you, with each category template injected with `items`. The code block below showcases how to use this in the category template.

> `{% for item in items | sort(attribute="rank") %}`

## 1. Clean URLs: From `.html` to Folders

Shortly after I got the basic functionality working, the first thing I wanted to do was adjust the url structure of the category items.

The initial implementation had links looking like follows.

> `my-site.com/blog/first-post.html`

To be clear, there is nothing wrong with this. I just thought it'd look nicer if the items were accessible without the .html extension so I could access items for each category at links like follows.

> `my-site.com/blog/first-post`

Doing this was not too difficult. Most traditional static file web servers (such as Apache or Nginx) when processing a request at '/' look for a index.html file or something rather internally. i.e. `my-site.com/` resolves to the `index.html` sitting at the root. 

You can take a look at [ngix's implementation here](https://nginx.org/en/docs/http/ngx_http_index_module.html) to see what I mean.

This is also the case for folders within the build contents. So all we needed to do was adjust the build process to ensure each item is instead created as a directory, and within that the built html as `index.html`.

## 2. The Typing Dilemma

Initially when I created a few basic interfaces of the markdown pages I had setup rather strict requirements.

```python
class MarkdownPageBase(TypedDict):
    title: str
    template: str
    date: str
    content: str
```

It later dawned on me while using the tool for my site that I don't really care what the frontmatter of each markdown pages is. All I really needed was the `template` and `title`. This could have been further simplified to just `template`.

However as I began to flesh out more of the tools functionality I got lost in the weeds a little with union returns and such (due to certain pages requiring some fields and others not) which overall made the program unnecessarily messy.

This led me to simplify the interface. The tool only requires that each markdown page contains in its frontmatter the `template` and the `title`. 

I am sure there are probably better ways of doing this. But given I wanted the tool to only be as 'complex as needed' for each use case, I decided to make it very minimal.

This is my second time working with typed items and its interesting. I really like the linting it provides and how I can be sure of somewhat expected behaviour. But I don't like how in a small project like this one I would need to likely create a separate class for a markdown page where date is required and I do not see the point in spending time to do this for this scale of project. Something to think about.

Something else I was wondering. How much time I have spent thinking about type safety? Is this really the best use of my time? Sure, it may be more loosey goosey but why does it feel like I have contracted TypeScript.

## 3. Live Reload

Initially I had simply used the livereload module. But after some thought I felt there was a simpler solution. I also felt this was an opportunity to learn. 

After a little thought I decided to implement it like follows. Each time I build, I write to a file `.build-info.json` that contains the build time and hash. I also inject a script in dev mode into each page that contains a build hash. 

```javascript
const currentHash = document.querySelector('meta[name="build-hash"]').content;
    setInterval(async () => {
        const response = await fetch('/.build-info.json');
        const buildInfo = await response.json();
        
        if (buildInfo.hash !== currentHash) {
            location.reload();
        }
    }, 1000);
```

As you can see above, it is fairly straightforward. Every second I check to see if the build hash on the page matches that of the build hash in the written file. If we detect a difference, we refresh the page.

There's obviously a lot of room for improvement here. The next logical step would be to build out a data structure to manage the state of each page and its dependencies to optimise the live reload but that would require a lot more work and I think I got out of it what I wanted in avoiding using livereload and a websocket in the first place.

## 4. File Paths

During implementation, an interesting technical challenge I faced was how to ensure I copy assets over to new instances in different environments. If I used standard filesystem paths such as `~/Lab/soma/fonts`, the Python package might be distributed in a variety of formats where the files may not exist as regular filesystem objects at runtime. The code below demonstrates a little more clearly.

```python
def copy_default_fonts(font_path: Path):
    """Copy default fonts from package"""
    try:
        with resources.as_file(resources.files('soma.fonts')) as font_dir:
            for font in font_dir.iterdir():
                if font.is_file():
                    shutil.copy(font, font_path / font.name)
    except Exception as e:
        print(f"soma (error: {e}")
```

If you take a look at line 4 you will see `resources.as_file` being called. By using this approach instead of a filesystem path we ensure that the resources can be reliably accessed regardless of how the package is installed or executed.

## 5. Development Process w/ LLM's

During the initial development of this tool I consulted with a few LLM's to get an idea of what the project should look like. This helped greatly. I did have a look at a few existing repo's but most were extremely extensive and complicated (to me) pieces of software. Using the LLM's I was able to extract the common denominator operations my SSG should perform and from there I was able to implement each part as I liked.

Something I did notice that bugged me however was the LLM's tendency to couple everything really tightly. I can see how this is quite efficient, but to me it didn't seem very readable/understandable. I started to instead when encountering rather troublesome bugs to just ask it to provide a text response to the problem areas so I could make better sense of the problem myself. I do not want to waste time learning how the LLM wrote a complicated solution to something that in reality was pretty simple to fix.

## 6. Project Structure

Recently I've noticed a trend towards keeping all your code within as few files as possible. I was pretty against this early days in my programming journey but the more I try to adhere to this the more sense it makes. Whenever I need to find a function I simply CTRL-F and I know its somewhere in the file. It's also allegedly faster although I am yet to test this for myself. I think I used to over modularise. I still split out things as needed, but only when I feel it absolutely necessary.

## Summary

This project was an excellent exercise for myself to get more familiar with web technologies, tooling and python. Some other unexpected discoveries along the way such as how I interact with LLM's and what my development process is like were welcome.

In future, I will port this to C for speed and to get more familiar with C. From here I will look to optimising build and providing slices to items for categories for larger, more complex sites.