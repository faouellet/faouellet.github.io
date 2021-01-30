---
title: Of source control and databases - Part 0 - Overview
permalink: /dvcs-intro/
tags: [Source Control, Database]
categories: [Of source control and databases]
excerpt: What I cannot create, I do not understand.
---

---

All the code for the articles in this series can be found [here](https://github.com/faouellet/DVCSUS).

---

[Once](https://faouellet.github.io/refactoring/) [again](https://faouellet.github.io/codereview/), I offer you an adaptation of some content I developed in my previous life as a teaching assistant. This time, I'll be walking you through an alternative solution to what was the first homework I usually gave in a course on software development.

## Motivation

{%
  include figure.html
  src="/assets/img/feynman.webp"
  caption=""
%}

Like Feynman, I am a firm believer that you cannot fully grasp a technology if you have no practical knowledge of it. To truly grok something, you have to be able to build it yourself. From deriving mathematical equations to building your own drones, you have to go that extra mile to fundamentally understand something.

It's with this mindset that I set out to fix one problem that most of my students suffered from: they had a hard time collaborating using source control systems. Since pontificating about source control best practices seemed to only go so far, I challenged them to build a minimalist source control software, named DVCSUS (**D**istributed **V**ersion **C**ontrol **S**ystem of **U**niversité de **S**herbrooke) and based on [Git](https://git-scm.com/). If that didn't help them understand how source control softwares worked, I don't know what will!

## Scope

Of course, building a fully-featured source control software goes way beyond the scope of a single university course. Expecting students to come up with a rival to [Git](https://git-scm.com/) or [Mercurial](https://www.mercurial-scm.org/) within (probably less than) four months is setting them up for failure. Instead, it's wiser to concentrate their efforts on the most basic functionalities of source control softwares: creating a repository, adding files to it and commiting modifications.

Moreover, while it was not in the original homework assignment, I'll also be showing how to add branches and interacting with a remote repository. What can I say? I had too much fun while working on these articles!

## Databases?

Since my homework is still being used (at least as far as I know), I would be remiss to offer a straight up solution here on my blog. Thus, I'll be showing you an alternative design and implementation that uses databases to store a repository instead of the filesystem as is done by the most common source control softwares. In a sense, rather than showing you how to implement a '[Git](https://git-scm.com/)-lite', I'll be showing you how to implement a '[Fossil](https://fossil-scm.org/home/doc/trunk/www/index.wiki)-lite'.
