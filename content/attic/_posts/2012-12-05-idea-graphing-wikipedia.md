---
layout: post
title: Idea - Graphing Wikipedia
excerpt: "Project idea"
tags: [idea, network graph, wikipedia]
date: 2012-12-05
comments: true
---

I'm almost certain something like this has been done before. Anyway, here's the idea:

1. Download the wikipedia database dump.
2. Ingest article texts into a database
3. Scrape wikipedia links out of the first paragraph of each article.
4. Create a directed graph of articles where two articles share an edge if they are linked as described in (3). Treat article categories as node attributes.
5. Investigate community structure of wikipedia articles, particularly which categories cluster together
6. Extra challenge: Try to find articles that won't ["get you to philosophy"](http://en.wikipedia.org/wiki/Wikipedia:Getting_to_Philosophy)

There are currently over 4M articles in the english wikipedia, so for this to be feasible I will probably need to invent some criterion for including articles in the project, probably minimum length, minimum age, or minimum edits. Alternatively, I might just focus on certain categories/subcategories.