---
layout: post
title: The Best of The Best on CrossValidated
excerpt: 'Using the CrossValidated data explorer to find interesting users and posts.'
tags: [crossvalidated, stackexchange, data explorer, sql, t-sql]
modified:
comments: true
---

[CrossValidated](http://stats.stackexchange.com/) is the StackExchange Q&A site focused on statistics. I mainly participate on CV answering questions ([http://stats.stackexchange.com/users/8451/david-marx](http://stats.stackexchange.com/users/8451/david-marx)). One somewhat alternative use of SE sites that I like is to "stalk" interesting users. I started this practice on [StackOverflow](http://stackoverflow.com/) (the very popular programming QA site), where I'd occasionally stumble across very high reputation users answering topics with tags I was interested in, so I'd check their answer history to see what other interesting things they'd posted. This was interesting, but because users on SO have such varied skills, you never know what you're going to get: maybe I visited a users page because they were answering a Python question, but it turns out most of their activity is talking about some other technology I'm not interested in.

On the other hand, because the scope of CV is so much narrower, an analogous investigation on CV almost always produces very interesting results (i.e. in general everyone on CV's background and interests intersect with my own at least a little). SE has a very interesting tool available called the [data explorer](http://data.stackexchange.com/), a T-SQL interface to their database. As I happen to be a database jockey for my day job, I decided to make my CV user investigation more efficient by leveraging this tool.

[Here's a query I built that returns a list of CV users who have authored answers that scored >=25 points](http://data.stackexchange.com/stats/query/129374/cv-users-with-most-interesting-answers). The approach I chose satisfied me for several reasons:

1. I'm not just interested in reputation. Repuatation is largely a function of activity on the site: if you answer a lot of question but never achieve a very high score, the shear volume of your answers will give you high reputation. Similarly, asking a lot of questions will boost your reputation, and I'm specifically interestd in people with good answers.
2. I'm not just interested in the highest scoring responses on the site. I want reading material: theoretically, I would like to read the highest scoring answers on the site, but I'm lazy:: I'd like to be able to visit a users profile and have several interesting answers to read, not just one.

In particular, to satisfy the second criterion, my query sorts users in a fashion that is not entirely obvious when glancing at the results: instead of sorting by the user's repuation or their answer reputation or their max answer score or the number of high scoring answers, I used a combiend index: the results are sorted by the highest answer score they achieved multiplied by the number of answers above the score threshhold (24). I like this approach for the following reasons:

* People with lots of good answers should bubble up to the top.
* As the number of good answers goes down, the max score that user has achieved becomes increasingly relevant.
* Users with just one good answer should sink to the bottom unless their answer was really good.

{% highlight sql %}
SELECT MAX(score) max_score, 
       count(p.id) good_answers,
       (SELECT count(*) 
        FROM posts p2
        WHERE p2.posttypeid=2 
          AND p2.owneruserid = u.id) total_answers,
       u.reputation,
       'site://users/' + CAST(u.Id AS nvarchar) + '?tab=answers&sort=votes|'
         + u.displayname  cv_user, -- link to profile, will display as username
       u.age, 
       u.location, 
       u.websiteurl,
       lastaccessdate
FROM   users u, posts p
WHERE  u.id = p.owneruserid
  AND  p.score >= 25
  AND  p.posttypeid = 2   -- answers only
  AND  p.parentid <> 1337 -- ignore jokes
  AND  p.parentid <> 726  -- ignore quotes
GROUP BY reputation, u.displayname, u.age, u.location, u.websiteurl, 
       u.lastaccessdate, u.id
ORDER BY max(score) * count(p.id) desc
{% endhighlight %}      