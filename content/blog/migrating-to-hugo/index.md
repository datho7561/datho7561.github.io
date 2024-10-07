---
title: "Migrating to Hugo"
date: 2024-10-07
math: true
comments: true
---

![Jekyll logo (a test tube with red liquid) with an arrow pointing towards the Hugo logo (the letter H in a pink hexagon)](jekyll-to-hugo.png)

In this blog post I'll document migrating my blog to Hugo.

I've been picking away at this migration throughout various days Red Hat has encouraged us to learn new subjects.
The suggested subject for these days was LLMs, which I'm not too fussed on.

## Why

Here are the main appeals of Hugo to me:

### It doesn't use Ruby

Jekyll has been a pain to set up.
Sometimes some required packages are missing after installing the `jekyll` binary,
or the wrong versions of the libraries are shiped with the OS.
To get around this, I developed my blog in a carefully crafted pet container using [toolbx](https://containertoolbx.org/).
I think `toolbx` is cool, but I would really like to avoid this.
Working with Jekyll becomes even more of a mess if I want to do anything on Windows,
where Ruby must be installed through `msys2`.
In contrast, Hugo ships in Fedora, my main OS for getting things done,
and at first glance seems to work.

### I found a good theme

I'm using [the hextra theme](https://github.com/imfing/hextra),
which GitHub suggested to me.
I was immediately struck by how good it looked, while not being too fancy.
The Jekyll theme I was using,
[minima](https://github.com/jekyll/minima) (the default one),
looks okay.
I like that it's minimal,
but I don't like the formatting of the landing page,
nor the lack of support for header images in blog posts.
Also, [it seems minima has been abandoned](https://github.com/jekyll/minima/issues/411)
(although reading through a few bugs the truth is more complex).
As a result,
there are a few bugs in the codebase and some new Jekyll features that it doesn't incorporate.
In order to overcome these limitations,
you must either override the broken portions of the theme in your site,
or maintain a fork of the theme and use that.

## Expected Issues

### I want to preserve my permalinks

My [VS Code extensions with esbuild blog post](https://datho7561.github.io/permalink/vscode-webpack-to-esbuild) is linked in a few places,
and it would be nice if those links still worked.

### I am hosting a p2 repository through my Jekyll blog

The .p2 repository contains an Eclipse plugin.
It's a bunch of files that I would like to statically serve.
There are better ways to do this,
but for now I'd like to keep it as is.

### CI/CD configuration

I have no idea how to configure the CI/CD pipelines for GitHub Pages.
Hopefully it's similar to GitHub Actions,
but GitHub Pages existed long before GitHub Actions, so I'm not sure.

### It would be nice to have some commenting solution

I was using [utteranc.es](https://utteranc.es) on Jekyll in order to allow folks to comment on my blog.
I will need to see if this or some other solution will work for Hugo.

## What Happened

### Good Stuff
1. The page hot reloading is really helpful

2. Deploying to GitHub Actions is easy; the theme I'm using, Hextra, provides a GitHub Action in order to do it.

3. $\LaTeX$ support, although I have no immediate use for it.

4. Adding comments through utteranc.es wasn't too bad. I think the difficulty varies from theme to theme.

5. Hosting static files was as breeze, so my p2 repo should be intact

### Bad Stuff

#### Configuration

The configuration caused me headaches.
The root config file can be in YAML, TOML, or JSON.
When you read documentation for Hugo, forum posts on the Hugo forum, or documentation for the theme,
the author often provides code for one of these,
and it can take a while to realize which one it is and translate it to the one you are using.

A contributing factor to my confusion was that I chose to use TOML,
which I've had limited exposure to in the past.

Also, some configuration can be done at a per-page level,
and some can/must be done in the root config file.
I got confused a few times which place I should put the config,
such as when setting up some page redirects.

#### Theme Modification

Inevidably, you'll want to change some aspects of your theme.
For me, this was the fact that, at the time of writing, Hextra doesn't support preview images for blog posts.

There are mechanisms to override parts of a theme,
but I found them confusing and not powerful enough to accomplish what I was hoping them to.
The only workaround is copying some or all of the theme into your repo and modifying it.

Some parts of the Hugo documentation instruct you to vendor your theme from the get go using git submodules.
I tried this at the start, but immediately regretted it, because git submodules are annoying to work with.
Switching over to using Golang's dependency system wasn't bad,
but as a result I gave up on most of the control I have over the theme.

I gave up on showing blog post images on the landing page, because it would be very painful.

#### Hot Reloading

Sometimes the page hot reloading doesn't work.
For example, if you break your hugo config,
hugo will serve an error page,
but after you fix the error,
the page won't reload.
Also, sometimes hotloading CSS changes doesn't work properly.
All of these issues involve restarting the CLI tool that serves the pages.
It's not bad to deal with, but over time it got to be annoying.

#### Creating a new post

The CLI to create a new blog post is bad.
I don't think it's necessary in order for Hugo to work.
I don't know why the documentation instructs you to use it.
If you are using Hugo, ignore it and just create a Markdown file.
Better yet, create an `index.md` file in a named directory.
Hugo will know what to do.

### Conclusions

Overall, this process was pretty painless, despite my gripes above.
It may have taken almost a full calendar year,
but that was because I was fussing over minutae.
I really like how the blog turned out after the changes,
and I hope this makes it easy to maintain.

