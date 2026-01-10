---
layout: post
title: Navigating the reddit graph
excerpt: "Building a map of reddit"
tags: [link analysis, networkx, gephi, python, network analysis]
modified:
comments: true
---

*(This is a project I've been sitting on for a while and have just never gotten around to writing a post on. I hope to fill this in with more details in the future, but for now I just want to get it on the blog)*

Let's start off with the results. If you're interested in reading how I built this thing, keep reading after the jump.

https://dmarx.github.io/SubredditMentionsGraph/network/

<iframe 
    src="https://dmarx.github.io/SubredditMentionsGraph/network/" 
    marginwidth="0" 
    marginheight="0"
    width="900" 
    height="1000"    
    scrolling="no"
    >
</iframe>



Back in February 2015, I was interested in the effect that promoting a subreddit could have on its traffic. I built a tool which you can play with at http://www.subredditmentions.com that lets you search for a subreddit and then generates a plot of the internet traffic for that subreddit (if the moderators made it public) overlaid with the times when people made comments outside of the searched subreddit linking to it. This was a really fun project with a lot of moving parts (praw scraper, sqlite datastore, flask backend, d3 + bootstrap frontend) and it deserves its own write up, but I'm not going to get into that yet

As a consequence of building this tool, I accumulated a lot of data (basically) in the form of:

    (timestamp, username, subreddit of comment, linked subredit)
    
Read this as: "on date/time <<timestamp>> user <<username>> made a comment in <<subreddit of comment>> that included a link to <<linked subreddit>>."
    
I tend to see graphs everywhere and, not surprisingly, saw one here:

    (timestamp, username, source, target)
    
I'm currently playing with leveraging the time component of the graph for another project, but for this project I just flattened the graph by ignoring the time component. This basically gives us a directed bipartite graph whose edges look like this:

        Subreddit(source) --> Redditor(username) --> Subreddit(target)
        
I wanted to build a weighted subreddit->subreddit graph. The first projection I tried was to just count the number of times a given (source, target) pair occurred in the data. This gave reasonable results, but necessitated a lot of additional processing because of bots. For those unfamiliar with reddit bots, the problem is basically that there are a lot of automated user accounts patrolling reddit generating formulaic comments, which can include specific subreddit names. For instance, /r/Askreddit has an automoderator rule that recommends that users post certain kinds of questions to /r/answers. Failing to ignore this heavily inflates the /r/Askreddit -> /r/Answers edge weight. Similarly, there are many bots that post a link to a single subreddit whenever they comment (usually the bot's personal subreddit just to centralize discussion about the bot, not to spam a subreddit). There are also users who are trying to promote new subreddits that behave similarly, but they are much less of a problem in the data than bots, although I would like to handle them as well.

I began buy pruning out the most problematic bot accounts one by one. This worked very well, but ultimately this approach is clearly not scalable. 

I ultimately went with a different projection: calculating edge weight between two subreddits as the count of unique users associated with that edge. This approach has the added benefit of making edge weights interpretable as "the number of users who voted in support of this subreddit->subreddit relationship."

Using this alternative projection, I found I didn't need to account for bots at all. I still could, but each bot now only contributes a single "vote" so their effect is sufficiently mitigated to no longer being problematic and arguably being as useful as any other user promoting a given edge. Instead of targetting bots directly, I found I got good results by just dropping all edges that didn't have a weight of at least 2. 

I had been storing data for subredditmentions.com in a sqlite database, so building the edge list reduced to a simple sql query:

    SELECT source, target, COUNT(DISTINCT users) weight 
    FROM mentions 
    GROUP BY source, target
    HAVING weight >= 2;
    
To build the visualization, I spooled the edgelist to a csv, read it into Gephi, calculated the layout using the ForceAtlas 2 algorithm in LinLog mode with the "prevent overlap" feature, and then exported the result toa sigma.js webapp using a plugin provided by the Oxford Internet Institute.