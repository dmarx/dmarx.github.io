---
layout: post
title: Building a better map of reddit
excerpt: "Heuristics for social media graph extraction, validation, and visualization"
tags: [python, twython, twitter, OOP]
modified:
comments: true
---

To start with, let me say this is not the first time someone has built a map of reddit. In fact, it's not even the first time
*I've* built a map of reddit (oh man... has it really been a year since my last blog post? Yikes! I guess I've been busy). But I had a 
few ideas kicking around for improving on the existing methods that had been tried before. The different mapping (graphing) methodologies
hinge on three things: 

1. What defines a relationship, and the strength of that relationship?
2. How do we choose which relationships to ignore?
3. How do we visualize the graph?

## Prior Work

### The Subreddit Mentions Graph

Let's start with criticizing my own prior work. I've played with a couple of different approaches, but the only one I've published so far
was my "Subreddit Mentions Graph" a posted about a year ago (almost to the day). In that project, I inferred the existence and strength of 
an A->B relationship between two subreddits by counting the number of users who had linked to subreddit B from inside subreddit A. This 
approach is powerful for several reasons: first and foremost, the choice to count users instead of comments was a really important decision.
Taking a raw count of comments has the effect that bot activities overwhelm the data. Using raw comment counts is feasible and could potentially
produce very good results, but requires that you come up with a mechanism for identifying and filtering out bot accounts. By only counting the
unique users who had made relevant comments, we are essentially letting each user "cast a vote" in favor of the existence of a particular relationship.

Another thing that was powerful about "subreddit mention" approach was that each event suppring the existence of a relationship has a timestamp
attached to it. If we treat these events as a point process, we can calculate the strength of an edge at any arbitrary point in time. I didn't 
actually leverage this in my project, but it's something I've wanted to revisit for studying the dynamics of reddit over time. Instead, I chose
to aggregate edges in discrete windows of time. I have repeated the windowing approach in this new project. The main reason I did this is because
the dataset I'm using is segregated by month, so it's just easier this way.

Another strength of this approach was that the dataset was small and extremely information dense. Other approaches (including the approach used
for this new project) require looking at all comments in a given period at once to determine which subreddits had users in common. The "mentions" approach let's us constrain attention to a small subset of comments: if we monitor the comment feed for relevant comments over the period we're
interested in, we never need to operate on a large dataset. This makes calculations much faster and more tractable on conventional hardware. The
results I achieved closely approximated the results achieved by others, so this approach demonstrates that we don't need to look at all comments
simultaneously to get good results in this kind of endeavour. 

A problem with the "counting unique users" approach to determining edge strength is that it doesn't contextualize the numbers we get. Consider, 
for instance, two subreddits -- /r/subredditA and /r/subredditB -- with a 1M users each. If 1k users from subredditA made "/r/subredditB" comments, 
we'd calculate an edge strength of 1k, even though only 0.1% of the users of either subreddit made such comments. We could scale this strength by
the number of users or the number of subscribers, but this figure isn't reliable since certain subreddits (e.g. /r/IAmA, /r/AskReddit) have a 
tendency to attract single-use novelty or throwaway accounts. A better approach would be to scale by some notion of "active users," but this still 
doesn't address the fact that some subreddits are probably much more likely to make /r/<subreddit> type comments than others, for various reasons. It
would be best if the edge strength were inferred from an activity metric, and then rescaled based on a similar activity metric. The problem here is 
that it's difficult to come up with a rescaling metric that is appropriately similar to the metric used to infer the edge strength to begin with, so
we're left with the raw number.

### The "Reddit World Map"

Probably the best known map of reddit was made by Randall Hiever (/u/rhiever), resulting in a publication as part of his doctoral research. His
approach had its pros and cons. Randall inferred edges by counting mutual users between two subreddits over a given period. This approach is similar
to my "mentions" approach of counting unique users in that it can be interpreted as letting each user cast a vote in favor of a relationship
between two subreddits. By reviewing mutual users over all public comments in a period, his edges were based on much more data than mine and
therefore strong edges had much more support. But, this approach has the side effect of producing many extremely weak edges that may need to be 
filtered out. Additionally, this  approach is still subject to the problem that edge strengths are not contextualized by the activity/size of the
communities they're connected to. To address this, Randall employed an approach for extracting the "core" subgraph by calculating an "edge 
significance" for each edge. I think the general philosophy of this approach is great, but I'm not a fan of the method he employed for several
reasons.

#### Edge significance

Without getting too deep into the math, the general technique Randall employed for identifying "significant" edges goes like this (I haven't read  
his article or the edge significance article in some time so I might have some of the minor details fuzzed):

1. Calculate the "strength" of a node (i.e. subreddit) by summing up the weights of all of its edges. This is done separately for incoming and outgoing edges.
2. Calculate each edges normalized weight as (edge weight)/(node strength).
3. There's a special distribution we can reference to determine what we would expect a "random" edge's normalized weight to be as function of the degree (number of edges) of the node whose strength we're scaling from. 
4. This leads us to a traditional frequent statistics test: we choose a significance threshold (somewhat arbitrarily) to get a cut-off normalized edge weight, which can be interpreted as "100*(1-threshold)% of edges with normalized strength weaker than the cut off are extremely likely to be 'random' edges that we can ignore."
5. Drop all nodes that aren't connected to any edges after applying this procedure.

This approach has the effect of preserving the "multi-scale" structure of the network, which essentially means that we can use this technique to 
extract a smaller version of the network that will exhibit statistical properties very similar to the larger network. If we're mainly interested in
calculating statistics on our network, applying this approach will make our lives (and calculations) a lot easier. 

But, if our main goal is to create a "map" that members of the network can use to navigate the network, and in particular identify esoteric
sub-communities that may be relevant to them, than this approach is actually deceptively aggressive. The application of a statistically inferred
threshold is very appealing because, frankly, people like p-values (despite the fact that most people don't interpret them correclty, which is a whole other issue). Consider a two nodes with degree one that are connected by a single edge with high edge weight: no matter what threshold we set,
this approach will always eliminate that edge, dropping both nodes from the network. Similarly, imagine an arbitrarily large clique of nodes that
are all connected by equally (high) weighted edges to each other and no one else: this technique will again cause us to drop all edges in the clique 
and as a result to drop the clique entirely.

The edge significance approach is fine if we just want to extract a statistically similar subgraph, or in the absence of better information for determining edge significance. But, we're interested in much more than just finding the "core" subgraph, and we can actually construct much better
approaches to identify "significant" edges here.

## The new way