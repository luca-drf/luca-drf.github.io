---
title: "Building this site with HUGO"
date: 2022-04-18T12:38:24+01:00
tags: [web development, hugo, github]
draft: false
featured: true
image: "https://i.ibb.co/8cSRNxf/resized-hugo.png"
alt: "hugo logo"
summary: "How I went from knowing nothing about HUGO framework to publishing my first custom website in just few days, all thanks to HUGO great ecosystem."
---

HUGO is really **fast** when it comes to building static assets, but its real
speed is its linear learning curve.


Background
----------

I was looking to build a personal website that I could easily update and deploy.
I've used [WordPress](https://wordpress.com/) in the past, but this time I
wanted to try something simpler (didn't really need any server-side
capabilities). As I'm not really a Web front-end developer, and I'm not
massively interested in diving into UI/UX too deep, my main focus is on content
editing and deployment automation.

Basically I just wanted to be able to write a bunch of Markdown (or similar
markup language), compile it into nice HTML (with maybe a little client-side JS
code to improve the UX), test it locally (my development platform is an Apple
Silicon Mac) then version and deploy it with as little hassle as possible.

Luckily for me, all the above requirements are covered by a framework called
[HUGO](https://gohugo.io/) which is a static website generator written in
[Go](https://go.dev/) (having basic knowledge of Go helps but isn't really
required for effectively use HUGO).

The main advantage of maintaining/deploying a static website is that it only
requires a CDN to be served (with obvious advantages also in terms of
security/privacy and testing), so in this case I'm hosting this site using
[GitHub Pages](https://pages.github.com/) which provides the CDN and all the
CI/CD tools I need in one place.


Development
-----------

First I've [installed HUGO](https://gohugo.io/getting-started/installing/)
I'm developing on an Apple Silicon Mac, and HUGO can be easily installed via
[Mac Homebrew](https://brew.sh/) with:

```bash
brew install hugo
```

Then I created a new site:

```bash
hugo new site lucadrf.dev
```

At this point I needed a base theme, as I didn't want to build the whole site
from scratch (and had zero experience with HUGO). There are hundreds of themes
to chose from that span several use cases. In my case I wanted mainly a "resume
style" home page with maybe a blog space. I've found just what I was looking for
with [this theme](https://themes.gohugo.io/themes/hugo-resume/) which code was
conveniently hosted on [GitHub](https://github.com/eddiewebb/hugo-resume) as
well (thank you [eddiewebb](https://github.com/eddiewebb)).

As I wanted to add several changes to the theme I figured it would have been
cleaner to fork it and maintain my own (simplified) version. So I did it and
added my own fork to the project as a submodule:

```bash
cd lucadrf.dev
git init
git submodule add https://github.com/luca-drf/hugo-resume.git themes/hugo-resume
```

Using [exampleSite](https://github.com/eddiewebb/hugo-resume/tree/master/exampleSite)
as reference I've added my:
- [data/skills.json](https://github.com/luca-drf/hugo-resume/blob/master/exampleSite/data/skills.json)
- [data/experience.json](https://github.com/luca-drf/hugo-resume/blob/master/exampleSite/data/experience.json)
- [data/education.json](https://github.com/luca-drf/hugo-resume/blob/master/exampleSite/data/education.json)

Then updated `config.toml` and finally built the site with:

```bash
hugo
```

The entire website will be compiled and stored in `./public` (default) as static
assets.


A really nice feature is the development web server which serves the site
locally and can be configured to rebuild/reload upon changes. To run the server
I've simply:

```bash
hugo server
```

More info about setting up HUGO and the theme at:
- [HUGO Quickstart](https://gohugo.io/getting-started/quick-start/)
- [Hugo Resume](https://github.com/luca-drf/hugo-resume#hugo-resume)


Deployment
----------

After several iterations of coding and local testing, and having obtained a
decent site, I've set up automated deploy on GitHub Pages, taking advantage of
[GitHub Actions](https://github.com/features/actions) and its community. In fact
there are already two repositories with a comprehensive set of
Actions for HUGO in order to [build](https://github.com/peaceiris/actions-hugo)
and [deploy](https://github.com/peaceiris/actions-gh-pages) on GitHub Pages.

The workflow consists in having a GitHub Pages repository for the project with a
dedicated branch (`gh-pages`) working as a "deployment" space for the built
assets (i.e. the content of `./public`), then configure GitHub Pages to serve
the assets on `gh-pages` rather than `main`.

Here's the complete workflow:

```yaml
name: GitHub Pages

on:
  push:
    branches:
      - main  # Set a branch name to trigger deployment
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-20.04
    concurrency:
      group: ${{ github.workflow }}-${{ github.ref }}
    steps:
      - uses: actions/checkout@v3
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.97.2'

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        if: ${{ github.ref == 'refs/heads/main' }}
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

So every push onto `main` will trigger an Action that will build the site and
then push the resulting assets onto `gh-pages`.

I'm keeping `main` as a "release" branch and have `dev` as the main development
branch.

In order to create a new blog post then I'll simply...

```bash
hugo new blog/new-blog-title.md
```

...and start adding content. Once the post is done, I can quickly check it locally,
then merge/push onto `main` and voilà 💁🏻‍♂️ 😃

More info on building and hosting on GitHub:
- [Hosting HUGO on GitHub](https://gohugo.io/hosting-and-deployment/hosting-on-github/)


Tips And Tricks
---------------

- When building/serving locally, HUGO uses a cache directory (`$TMP_DIR/hugo_cache`
by default). In some cases you might want to ignore the cache and rebuild
everything (e.g. changing certain file names might break the build if the cache
is not invalidated) so you can pass `--ignoreCache` to `hugo` or `hugo server`
commands.


Conclusion
----------

I think the main strengths of HUGO are its modularity (both in terms of project
layout and functionalities) and its community. I also appreciated its good
balance between *conventions* and *configurations* make it overall extremely
flexible yet easy to pick up while using it or, in other terms, very Agile.
Also, the complexity doesn't seem to grow dramatically when adding new features
and/or diverging from what the theme was initially designed to do.

So, I've had good fun in building this site so far, and I'll definitely keep
diving into HUGO.

