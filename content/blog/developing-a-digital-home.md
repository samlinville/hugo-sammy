---
title: "Developing a Digital Home"
subtitle: "Or, how to automate yourself into a habit"
date: 2020-09-10T22:06:16-07:00
draft: true
---
I'm so bad at blogging. I've tried to make it a habit several times over the past few years, and every attempt is exactly the same. I start, I write two or three pieces, and a year later the blog is stale and I'm embarrassed. I think I can boil this down to a handful of things that consistently hold me back.

- Finding a text editor that's easy to write in, easy to organize, and fits well into my personal workflow has been a years-long quest. Most text editors suck. All markdown editors suck.
- Most text editors and writing tools have terrible mobile experiences...or no mobile experience at all. This tied me to my laptop, which is not always closest at hand when inspiration strikes.
- Writing a post in one editor and manually exporting it to a CMS is a hassle, but writing a post directly in a CMS is always a terrible experience.
- Static site builders unilaterally require content to be structured in a YAML file (or some adjacent flavor of structured text). Manually transferring content into this structure is annoying as hell.
- Maintaining a VPS is a ton of recurring work with little reward: Backups, database maintenance, package updates, key rotations...*ugh.*

The best way to make a habit is to keep it simple. When I look at this list and think about all the complications of building and maintaining a website, it's no wonder I've struggled to consistently publish writing.

## What if it were different?

Over the past few months, I've been slowly and steadily working through these problems and testing solutions. I found that Notion is a really great way to write (and to organize my entire life), and that static sites are best hosted on a platform like Netlify, not a VPS. Ensuring that my writing is stored and published in an open, accessible format is really important to me, too: no matter what, I always want to be able to read it, edit it, and convert it to new formats.

My dream workflow looks like this:

1. Open Notion
2. Write a blog post
3. Label it as "published" when I'm ready for it to go live

No databases to maintain, no YAML files to write, no manual data manipulation. From the moment I'm ready to share a piece of writing with the world, an automated toolchain moves it into the right place, optimizes any media, creates a backup copy, and publishes it on the web. When I need to update a post, that's propagated automatically, too.

The best part? It's not a dream workflow. It's real.

*"Wait, what? Notion isn't a CMS!"*

Out of the box, no it is not. An unofficial API wrapper makes it possible to extricate data from Notion and do whatever I want with it. Thanks to the kind folks that created it, I've been able to build a completely automated publishing toolchain **for free.**
