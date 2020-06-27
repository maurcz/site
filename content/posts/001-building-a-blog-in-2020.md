---
title: "Building a personal dev blog in 2020"
date: 2020-06-26T22:26:46-04:00
categories: ["Web Development", "Blogging"]
summary: Where I explain how I leveraged hugo to finally get into the modern blogging era. 
---


In what now looks like another life, I had a mildly famous tech blog. It all started 10 years (yikes) ago, when I still worked with ABAP. I partnered with a friend to create an ABAP-themed blog called [ABAPZombie](https://www.abapzombie.com/), using [WordPress](https://wordpress.com/). If you think the name is weird, you should take a look at [one of our first page designs](https://web.archive.org/web/20120202221336/http://www.abapzombie.com/).

SAP is viewed as a tedious platform by most programmers [^1] , and back then we believe a lot of this had to deal with how people perceived the information being shared about the ABAP language. Our blog was an attempt to share dev knowledge without coorporate strings attached to the content. And, of course, a platform to vent when we needed.

Long story short, the site started to get popular, starte to give talks in Brazilian SAP events, and even wrote a book. But since I decided to leave the SAP world for good, I never went back into writing. But this ends today!

## Static Site Generators

I had many issues trying to keep a WordPress website up for over 10 years. Plugins, themes, databases, "core" updates - you name it. So this time, [CMS](https://en.wikipedia.org/wiki/Content_management_system)s were out of question.

My goal was to create a personal site that could be as simple as possible, so using a static site generator (SSG) was an obvious choice. It's basically a tool that estipulates a certain workflow pattern, which in the end will result in a bunch of generated html/js/css files, ready to be served in the web. 

Three main things drew me towards a SSG over a CMS: markdown, speed and maintainability. Just thinking about all the caching solutions for WordPress gives me the chills. I really like markdown. And with static files, I don't have to care about my website being down just because an syntax-highlight plugin got updated [^2].

[Jekyll](https://jekyllrb.com/) has been around since forever, and was the tool I first intended to use for this website, but influenced by a co-worker I ended up deciding to go with [Hugo](https://gohugo.io/).

## Hugo

I'm still amazed by how easy it was to setup everything for this site with Hugo. The [quickstart](https://gohugo.io/getting-started/quick-start/) docs are really easy to follow if you're in Linux or OSX (`brew install` and off you go).

There are [plenty of themes](https://themes.gohugo.io/) available. A piece of advice if your testing out lots of free themes: some of them require very specific things in your site's `config.toml` file. Make sure to double check if your're following the theme dev's guidelines for configuration before moving on to a theme "that works".

However, most themes I tried still felt like too much for what I wanted, so I decided to go and create my own. [This guide](https://www.pakstech.com/blog/create-hugo-theme/) by Janne Kemppainen was really helpful, but it still requires you to know a bit of html/js/css to play with his examples.

For my custom theme, I decided to take _all_ the clutter out of the way. 










[^1]: I have no data to back up this claim, but I encourage you to try and ask any non-SAP dev you know that had to touch anything ABAP-related.

[^2]: This happened to me once. And again with other plugins.

