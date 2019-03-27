---
title: "Migrating a blog to Hugo and Netlify"
date: 2019-03-27
keywords:
- gohugo
- hugo
- netlify
---

In 2016 I decided to start this website, at the time I landed on [Pelican](https://blog.getpelican.com/) as my static site generator. For hosting, I went with [DigitalOcean](https://www.digitalocean.com/). Static site generators essentially generate HTML files, this simplifies requirements greatly - no longer do we need databases or backend code to display simple sites. There is another benefit: **Performance**. Even on the most basic DigitalOcean plan, my website never faced any loading or timeout issues. This is the beauty of a VM running a webserver, which in turn serves simple HTML files - It takes a whole lot of traffic to dent performance (even when you get a small hug from the likes of [Hacker News](https://news.ycombinator.com/)).

I have always been happy with this setup, so why change? Well... Why not? Tech is an ever-changing world, trying new things is just part of our nature. What else is out there?

Queue in...

### Hugo

[Hugo](https://gohugo.io/) is a behemoth in the static site generator world, and after looking through the docs I could understand why:

- Easy to use CLI tool.
- One file configuration with support for various languages (toml, yaml, json).
- Well documented templates.
- Out of the box support for things like Disqus, Google Analytics, Syntax highlighting...

Migrating from Pelican was easy enough. Both Pelican and Hugo use Markdown, so that was mostly a `cp` job. One of the things I needed to ensure was keeping my old URLs in place, this was accomplished very easily via the following config options:

```toml
uglyurls = true

[permalinks]
  post = ":title"
```

`uglyurls` makes it possible to have the `.html` extension as part of the URL. The rest formats the URL using the title of the post. This allowed me to turn `/post/migrating-hugo-netlify/` into `migrating-a-blog-to-hugo-and-netlify.html`. An added bonus is that this kept my Disqus comments in place (Disqus uses the URL to determine what comments to load), this avoids me having to fiddle around within the Disqus settings and remapping URLs.

The overall experience with Hugo, so far, has been great.

### Netlify

I came across [Netlify](https://www.netlify.com/) in the Hugo [documentation](https://gohugo.io/hosting-and-deployment/hosting-on-netlify/):

> Netlify can host your Hugo site with CDN, continuous deployment, 1-click HTTPS, an admin GUI, and its own CLI.

Sounds really good. But is that enough to win me over a simple rsync deploy, like the one I was used to?... The answer is a big solid **yes**. First of all, Netlify has a **free** basic plan specifically designed for small personal websites (such as this one). That itself is enough to grab my attention.

Aside from hosting a website (for free) with your own domain and SSL (via Let's Encrypt), Netlify is a service that just _works_:

1. Select your Github/Gitlab/Bitbucket repo.
2. Netlify will auto-detect your Hugo site.
3. A "deploy" process starts, which automatically builds the site.
4. The site is now deployed and can be acessible via a the Netlify domain (in my case at https://elegant-cray-1a7195.netlify.com/)
5. You can now setup your own personal domain and SSL.

The result is the website you are currently looking at. A push to the Github [repo](https://github.com/joaodlf/joaodlf) automatically triggers a build and deploy.

Netlify appears to be a fantastic service, but it remains untested. How will it perform under load? And what about uptime?

### Conclusion

I now have a website that is easier to build and extend, easier to deploy, and hosted for free. Hugo is a clear winner, I don't see myself going back to Pelican. As for Netlify, it does seem to be a really great service - too good to be true? Maybe. Only time will tell.