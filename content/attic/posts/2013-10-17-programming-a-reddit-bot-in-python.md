---
layout: post
title: Programming A Simple Reddit Bot (in Python)
excerpt: "A tutorial via walking through the design of a simple bot."
tags: [python, praw, reddit]
date: 2013-10-17
comments: true
permalink: programming-a-reddit-bot-in-python
---

Almost a year ago I built a built a reddit bot called [VideoLinkBot](http://www.reddit.com/user/videolinkbot) that has been running on an old half-broken laptop in my room ever since. VLB was a great exercise in agile development that I started to write a post about awhile ago but abandoned. The gist of it is that the bot only satisfied a small portion of my vision when it first went live, but I didn't want to delay "bringing my product to market." This was a good decision because due mainly to time constraints, it took quite some time for me to incorporate most of the features I originally planned for the bot, many of which were deprioritized in my development path in favor of things my "end users" (redditors) specifically requested.

Courtesy of the [praw](https://praw.readthedocs.org/en/latest/) library, building a simple bot is really very easy. As an exercise, I wrote a clone of the now disabled "linkfixerbot" to illustrate a simple bot template to another redditor, since VLB was fairly complex. Instead of discussing VLB at length (I may do so at a later date) I'll describe a general strategy for programming a reddit bot via the LinkFixerBot example.

### LinkFixerClone

(Here's the [complete bot code](https://gist.github.com/dmarx/5550922))

On reddit, it's not uncommon for a comment to include internal links to either a subreddit or a user's overview page. To simplify building these links because they're so common, reddit markdown interprets the syntax "/u/username" as a shortlink to a user's history page, and "/r/subredditname" as a shortlink to a subreddit. Sometimes users either don't realize that this markdown feature exists (less an issue now than when the feature was first rolled out) or they simply neglect to add the first forward slash and instead of proper markdown, they write something like "r/subredditname." LinkFixerBot was a bot that scanned reddit for these mistakes and responded with repaired corrections.

We can break down the bot into three activities:

1. A component that scans the reddit comments feed.

2. A component that determines if a particular comment is a candidate for bot action.

3. A component that performs the bot's signature activity in the form of a comment response.

There are many types of reddit bots, but I think what I've outlined above describes the most common. Some other kinds of bots may be [moderator bots](https://github.com/Deimos/AutoModerator), or bots that mirror the contents of a specific subreddit.

The LinkFixerBot code should largely speak for itself: the while loop scans the reddit comment stream for comments, passing each comment to the "check_condition" function that determines if the bot should do anything. If the bot's services are required, the results of pre-processing the comment accomplished by the check_condition function are passed into the bot_action function, which causes the bot to respond with a comment. Rinse and repeat.

Some items worth note:

* It's a good idea to maintain a short cache of comments the bot has already looked at to prevent duplicated work. LinkFixerClone uses a fixed-length deque so when new comments come in, they push the old comments out. The deque length is set to twice the number of comments the bot can call down from reddit at a time.
* You're going to need to include exception handling. Some special cases to consider (that are probably less of a problem for most bots than for VLB):
  * A comment or submission of interest has been deleted
  * A comment or submission is sufficiently old that it has been archived
  * Reddit is down (consider adding a progressively increasing "back-off" time until your next attempt).
  * Your bot's username doesn't have enough karma to post comments as frequently as it wants to.

This last case can actually be remedied, at least to some extent, by just using the account and accruing karma.

The reddit API rules discuss the appropriate request rate, but this is actually not a concern for us because praw handles that.