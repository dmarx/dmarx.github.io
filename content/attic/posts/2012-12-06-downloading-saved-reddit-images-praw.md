---
layout: post
title: Downloading images from saved reddit links
excerpt: "A concise demonstration of the power of the python praw library"
tags: [praw, python, reddit, webscraping]
date: 2012-12-06
comments: true
---

Reddit user aesptux posted to [/r/learnprogramming](http://www.reddit.com/r/learnprogramming/comments/14dojd/pythoncode_review_script_to_download_saved_images/) requesting a code review of [their script](https://github.com/aesptux/download-reddit-saved-images/blob/master/script.py) to download images from saved reddit links. This user made the same mistake I originally made and tried to work with the raw reddit JSON data. I hacked together a little script to show them how much more quickly they could accomplish their task using praw. It took almost no time at all, and the meat of the code is only about 20 lines. What a great API.

Here’s my code: 

{% gist 4225456 %}
