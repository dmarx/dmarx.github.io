---
layout: post
title: Scraping your twitter home timeline with python and mongodb
#excerpt: "Using the twitter API and NoSQL to construct a dataset of your friends tweets."
tags: [text processing, natural language processing, nlp, python, mongo, mongodb, nosql, twython, twitter]
modified:
comments: true
---

### Background

About a year and a half ago I was hanging out with two colleagues, John and Jane. John and I were discussing various new happenings we'd heard about recently. Jane was very impressed with how current we were and wondered how we did it. I described how I subscribe to several blogs and that suits me fine, but John insisted that we both needed to try Twitter.

I buckled down and finally created a twitter account. I didn't really know who to follow, so picked a prominent local data scientist and let him "vet" users for me: I skimmed his "following" list and decided to also follow anyone who's description made them sound reasonably interesting (from a data science stand point). The problem with this method is that I ended up following a bunch of his random friends who don't actually talk about data science. Right now, there's just too much happening on me twitter feed to keep up. If I don't check it every now and then, I'll quickly amass hundreds of backlogged tweets, so I have strong motivation to "trim the fat" of my following list.

<!--more-->

###Setup

To get started, I'm going to explain how to scrape your twitter homepage. But first things first, we're going to need a few things:

####Twitter API wrapper

There are several python twitter API wrappers available right now. I did some research back when I first started tinkering with twitter and landed on the Twython package. I don't remember what led me to it, but I think the main thing is that it has a strong community and so there's a lot of good documentation and tutorials describing how to use it.

To install Twython, just use pip like you would for most anything else:

<pre style="background-color:#efefef;">
pip install twython
</pre>
    
No surprises here.

#### Twitter API authentication

We're going to need to do two things to get our scraper working with twitter. First, we need to register a new app at [http://apps.twitter.com](http://apps.twitter.com). If your desired app name is taken, just add your username to make it unique. It's not mentioned anywhere on the page, but you can't have the '@' symbol in your app name (or at least, it can't be preceded by the '@' symbol).

Next, register an access token for your account. It only needs to have read-only permissions, and keeping it this way ensures we won't do any real damage with our experiment.

Finally, store the authentication information in a config file (I called mine "scraper.cfg") like so:

<pre style="background-color:#efefef;">
[credentials]
app_key:XXXXXXXXXXXXXX
app_secret:XXXXXXXXXXXXXX
oath_token:XXXXXXXXXXXXXX
oath_token_secret:XXXXXXXXXXXXXX
</pre>

 
#### MongoDB

Finally, we're going to need to set up a repository to persist the content we're scraping. My MO is usually to just use SQLite and to maybe define the data model using SQLAlchemy's ORM (which is totally overkill but I still do it anyway for some reason). The thing here though is:

1. There's a lot of information on tweets

2. I'm not entirely sure which information I'm going to find important just yet

3. The datamodel for a particular tweet is very flexible and certain fields may appear on one tweet but not another.

I figured for this project, it would be unnecessarily complicated to do it the old fashioned way and, more importantly, I'd probably be constantly adding new fields to my datamodel as the project developed, rendering my older scrapes less valuable because they'd be missing data. So to capture all the data we might want, we're going to just drop the tweets in toto in a NoSQL document store. I chose mongo because I'd heard a lot about it and found it suited my needs perfectly and is very easy to use, although querying it uses a paradigm that I'm still getting used to (we'll get to that later).

Download and install MongoDB from [http://docs.mongodb.org/manual/installation/](http://docs.mongodb.org/manual/installation/).
I set the data directory to be on a different (larger) disk than my C drive, so I start mongo
like this:

<pre style="background-color:#efefef;">
C:\mongodb\bin\mongod --dbpath E:\mongodata\db
</pre>

We will need to run this command to start a mongo listener before running our scraper. Alternatively, you could just drop a system call in the scraper to startup mongo, but you should check to make sure it's not running first. I found just spinning up mongo separately to be simple enough for my purposes.

Since we've already got a config file started, let's add our database name and collection (NoSQL analog for a relational table) to the config file, so our full config file will look like:

<pre style="background-color:#efefef;">
[credentials]
app_key:XXXXXXXXXXXXXX
app_secret:XXXXXXXXXXXXXX
oath_token:XXXXXXXXXXXXXX
oath_token_secret:XXXXXXXXXXXXXX

[database]
name:twitter_test
collection:home_timeline
</pre>

Take note: all we have to do to define the collection is give it a name. We don't need to describe the schema at all (which, as described earlier, is part of the reason I'm using mongo for this project).

### Getting Started

So we're all set up with twython and mongo: time to start talking to twitter.

We start by calling in the relevant configuration values and spinning up a Twython instance:

{% highlight python %}
import ConfigParser
from twython import Twython

config = ConfigParser.ConfigParser()
config.read('scraper.cfg')

# spin up twitter api
APP_KEY    = config.get('credentials','app_key')
APP_SECRET = config.get('credentials','app_secret')
OAUTH_TOKEN        = config.get('credentials','oath_token')
OAUTH_TOKEN_SECRET = config.get('credentials','oath_token_secret')

twitter = Twython(APP_KEY, APP_SECRET, OAUTH_TOKEN, OAUTH_TOKEN_SECRET)
twitter.verify_credentials()
{% endhighlight %}

To get the most recent tweets from our timeline, we hit the "/statuses/home_timeline" API endpoint. We can get a maximum of 200 tweets per call to the endpoint, so let's do that. Also, I'm a little data greedy, so I'm also going to ask for "contributor details:"

{% highlight python %}
params = {'count':200, 'contributor_details':True}
home = twitter.get_home_timeline(**params)
{% endhighlight %}

Now, if we want to do persistent scraping of our home feed, obviously we can't just wrap this call in a while loop: we need to make sure twitter knows what we've already seen so we only get the newest tweets. To do this, we will use the "since_id" parameter to set a limit on how far back in the timeline the tweets in our response will go.

### Paging and Cursoring

This is going to be a very brief overview of the motivation behind cursoring and how it works. For a more in depth explanation, check the twitter docs here: [https://dev.twitter.com/docs/working-with-timelines](https://dev.twitter.com/docs/working-with-timelines)

Consider a situation in which, since the last call to the timeline, so many new tweets have been written that we can't get them all in a single call. Twitter has a "paging" option, but if we use this, it's possible that the tweets on the bottom of one page will overlap with the tweets on the top of the next page (if new tweets are still coming into the timeline). So instead of "paging" we'll use "cursoring:" in addition to giving twitter a limit for how far back we can go, we'll also give a limit for the most recent tweet in any particular call. We'll do this using a "max_id" parameter. The API will still return the tweet with this ID though, so we want to set the max_id value just lower than the last tweet we saw. If you're in a 64bit environment, you can do this by subtracting '1' from the id.

Putting this all together, here's what our persistent scraper looks like so far:

{% highlight python %}
latest = None   # most recent id we've seen
while True:
    try:
        newest = None # this is just a flag to let us know if we should update the value of "latest"
        params = {'count':200, 'contributor_details':True, 'since_id':latest}
        home = twitter.get_home_timeline(**params)        
        if home:
            while home:
                store_tweets(home) # I'll define this function in a bit
                
                # Only update "latest" if we're inside the first pass through the inner while loop
                if newest is None:
                    newest = True
                    latest = home[0]['id']
                    
                params['max_id'] = home[-1]['id'] - 1
                home = twitter.get_home_timeline(**params)
{% endhighlight %}

### Rate limiting

As with pretty much any web API, twitter doesn't take too kindly to people slamming their servers. You can read more about the [rate limits for different API endpoints here](https://dev.twitter.com/docs/rate-limiting/1.1). Here's what concerns us:

* The rate limiting windows are 15 minutes long. Every 15 minutes, the window resets.  
* We can make 15 calls to the statuses/home_timeline endpoint within a given window.  
* If we exceed this threshold, our GET request to the API will return a 429 ("Too many requests") code that Twython will feed to us as a twython.TwythonRateLimitError exception  
* Twitter provides an API endpoint to query the rate limiting status of your application at application/rate_limit_status.  
* The application/rate_limit_status endpoint is itself rate limited to 180 requests per window.

If we don't pass in any parameters, the `application/rate_limit_status` endpoint will return the rate limit statuses for every single API endpoint which is much more data than we need, so we'll limit the data we get back by constraining the response to "statuses" endpoints:

{% highlight python %}
status = twitter.get_application_rate_limit_status(resources = ['statuses'])
{% endhighlight %}

This returns a JSON response wihch we only want a particular set of values from, so let's select that bit out:

{% highlight python %}
status = twitter.get_application_rate_limit_status(resources = ['statuses'])
home_status = status['resources']['statuses']['/statuses/home_timeline']        
{% endhighlight %}

Finally, we'll test how many API calls are remaining in the current window, and if we've run out set the application to sleep until the window resets, double check that we're ok, and then resume scraping. I've wrapped this procedure in a function to make it simple to perform this test:

{% highlight python %}
def handle_rate_limiting():
    while True:
        status = twitter.get_application_rate_limit_status(resources = ['statuses'])
        home_status = status['resources']['statuses']['/statuses/home_timeline']        
        if home_status['remaining'] == 0:                
            wait = max(home_status['reset'] - time.time(), 0) + 1 # addding 1 second pad
            time.sleep(wait)
        else:
            return
{% endhighlight %}

We're only testing one of the API endpoints we're hitting though: we're hitting the application/rate_limit_status endpoint as well, so we should include that in our test just to be safe although realistically, there's no reason to believe we'll ever hit the limitation for that endpoint.

{% highlight python %}
def handle_rate_limiting():
    app_status = {'remaining':1} # prepopulating this to make the first 'if' check fail
    while True:
        if app_status['remaining'] > 0:
            status = twitter.get_application_rate_limit_status(resources = ['statuses', 'application'])
            app_status = status['resources']['application']['/application/rate_limit_status']        
            home_status = status['resources']['statuses']['/statuses/home_timeline']        
            if home_status['remaining'] == 0:                
                wait = max(home_status['reset'] - time.time(), 0) + 1 # addding 1 second pad
                time.sleep(wait)
            else:
                return
        else :
            wait = max(app_status['reset'] - time.time(), 0) + 10
            time.sleep(wait)
{% endhighlight %}

Now that we have this, we can insert it into the while loop that performs the home timeline scraping function. While we're at it, we'll throw in some exception handling just in case this rate limiting function doesn't work the way it's supposed to.

{% highlight python %}
while True:
    try:
        newest = None
        params = {'count':200, 'contributor_details':True, 'since_id':latest}
        handle_rate_limiting()
        home = twitter.get_home_timeline(**params)        
        if home:
            while home:
                store_tweets(home)
                
                if newest is None:
                    newest = True
                    latest = home[0]['id']
                    
                params['max_id'] = home[-1]['id'] - 1
                handle_rate_limiting()
                home = twitter.get_home_timeline(**params)
        else:            
            time.sleep(60)
    
    except TwythonRateLimitError, e:
        print "[Exception Raised] Rate limit exceeded"
        reset = int(twitter.get_lastfunction_header('x-rate-limit-reset'))
        wait = max(reset - time.time(), 0) + 10 # addding 10 second pad
        time.sleep(wait)
    except Exception, e:
        print e
        print "Non rate-limit exception encountered. Sleeping for 15 min before retrying"
        time.sleep(60*15)
{% endhighlight %}

### Storing Tweets in Mongo

First, we need to spin up the database/collection we defined in the config file.

{% highlight python %}
from pymongo import Connection

DBNAME = config.get('database', 'name')
COLLECTION = config.get('database', 'collection')
conn = Connection()
db = conn[DBNAME]
tweets = db[COLLECTION]
{% endhighlight %}

I've been calling a placeholder function `store_tweets()` above, let's actually define it:

{% highlight python %}
def store_tweets(tweets_to_save, collection=tweets):
    collection.insert(tweets_to_save)
{% endhighlight %}

Told you using mongo was easy! In fact, we could actually just replace every single call to `store_tweets(home)` with `tweets.insert(home)`. It's really that simple to use mongo.

The reason I wrapped this in a separate function is because I actually want to process the tweets I'm downloading a little bit for my own purposes. A component of my project is going to involve calculating some simple statistics on tweets based on when they were authored, so before storing them I'm going to convert the time stamp on each tweet to a python datetime object. Mongo plays miraculously well with python, so we can actually store that datetime object without serializing it.

{% highlight python %}
import datetime

def store_tweets(tweets_to_save, collection=tweets):
    for tw in tweets_to_save:
        tw['created_at'] = datetime.datetime.strptime(tw['created_at'], '%a %b %d %H:%M:%S +0000 %Y')
    collection.insert(tweets_to_save)
{% endhighlight %}

### Picking up where we left off

The first time we run this script, it will scrape from the newest tweet back as far in our timeline as it can (approximately 800 tweets back). Then it will monitor new tweets and drop them in the database. But this behavior is completely contingent on the persistence of the "latest" variable. If the script dies for any reason, we're in trouble: restarting the script will do a complete scrape on our timeline from scratch, going back as far as it can through historical tweets again. To manage this, we can query the "latest" variable from the database instead of just blindly setting it to "None" when we call the script:

{% highlight python %}
latest = None   # most recent id scraped
try:
    last_tweet = tweets.find(limit=1, sort=[('id',-1)])[0] # sort: 1 = ascending, -1 = descending
    if last_tweet:
        latest = last_tweet['id']
except:
    print "Error retrieving tweets. Database probably needs to be populated before it can be queried."
{% endhighlight %}

And we're done! The finished script looks like this:

{% highlight python %}
import ConfigParser
import datetime
from pymongo import Connection
import time
from twython import Twython, TwythonRateLimitError

config = ConfigParser.ConfigParser()
config.read('scraper.cfg')

# spin up twitter api
APP_KEY    = config.get('credentials','app_key')
APP_SECRET = config.get('credentials','app_secret')
OAUTH_TOKEN        = config.get('credentials','oath_token')
OAUTH_TOKEN_SECRET = config.get('credentials','oath_token_secret')

twitter = Twython(APP_KEY, APP_SECRET, OAUTH_TOKEN, OAUTH_TOKEN_SECRET)
twitter.verify_credentials()

# spin up database
DBNAME = config.get('database', 'name')
COLLECTION = config.get('database', 'collection')
conn = Connection()
db = conn[DBNAME]
tweets = db[COLLECTION]

def store_tweets(tweets_to_save, collection=tweets):
    """
    Simple wrapper to facilitate persisting tweets. Right now, the only
    pre-processing accomplished is coercing 'created_at' attribute to datetime.
    """
    for tw in tweets_to_save:
        tw['created_at'] = datetime.datetime.strptime(tw['created_at'], '%a %b %d %H:%M:%S +0000 %Y')
    collection.insert(tweets_to_save)

def handle_rate_limiting():
    app_status = {'remaining':1} # prepopulating this to make the first 'if' check fail
    while True:
        wait = 0
        if app_status['remaining'] > 0:
            status = twitter.get_application_rate_limit_status(resources = ['statuses', 'application'])
            app_status = status['resources']['application']['/application/rate_limit_status']
            home_status = status['resources']['statuses']['/statuses/home_timeline']
            if home_status['remaining'] == 0:
                wait = max(home_status['reset'] - time.time(), 0) + 1 # addding 1 second pad
                time.sleep(wait)
            else:
                return
        else :
            wait = max(app_status['reset'] - time.time(), 0) + 10
            time.sleep(wait)

latest = None   # most recent id scraped
try:
    last_tweet = tweets.find(limit=1, sort=[('id',-1)])[0] # sort: 1 = ascending, -1 = descending
    if last_tweet:
        latest = last_tweet['id']
except:
    print "Error retrieving tweets. Database probably needs to be populated before it can be queried."

no_tweets_sleep = 1
while True:
    try:
        newest = None # this is just a flag to let us know if we should update the "latest" value
        params = {'count':200, 'contributor_details':True, 'since_id':latest}
        handle_rate_limiting()
        home = twitter.get_home_timeline(**params)
        if home:
            while home:
                store_tweets(home)

                # Only update "latest" if we're inside the first pass through the inner while loop
                if newest is None:
                    newest = True
                    latest = home[0]['id']

                params['max_id'] = home[-1]['id'] - 1
                handle_rate_limiting()
                home = twitter.get_home_timeline(**params)
        else:
            time.sleep(60*no_tweets_sleep)

    except TwythonRateLimitError, e:
        reset = int(twitter.get_lastfunction_header('x-rate-limit-reset'))
        wait = max(reset - time.time(), 0) + 10 # addding 10 second pad
        time.sleep(wait)
    except Exception, e:
        print e
        print "Non rate-limit exception encountered. Sleeping for 15 min before retrying"
        time.sleep(60*15)
{% endhighlight %}