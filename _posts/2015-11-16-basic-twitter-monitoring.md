---
layout: post
title: Using python to monitor twitter for trending content
excerpt: "Sipping from the firehose"
tags: [python, twython, twitter, OOP]
modified:
comments: true
---

There are many reasons why you might want to know what is trending on twitter. 
Maybe you're a news company and you want to catch breaking news as it's 
happening, or you're a stock broker who wants to be able to trade on information
before thier competitors. There are several companies for whom this kind of 
trend detection is essentially their entire business model. I'm not going to get
very deep in the weeds on the topic here, but we'll get our feet wet.

### The twitter API

I discussed the twitter API a little bit in my earlier post on scraping the
home timeline, but we're going to focus on the 
[public stream endpoints](https://dev.twitter.com/streaming/public) endpoints:

#### [GET statuses / firehose](https://dev.twitter.com/streaming/reference/get/statuses/firehose) 

The firehose is a feed of all tweets (and a few other twitter object) as they
arrive. In terms of both volume and velocity, the firehose is *massive*. As of 
2013, Twitter reported users generated approximately [500 million tweets per
day](http://www.internetlivestats.com/twitter-statistics/, around 6,000 tweets
per second.

The firehose is prohibitively expensive for a hobby project. We're going to play
with the other two endpoints.

#### [GET statuses / sample](https://dev.twitter.com/streaming/reference/get/statuses/sample)

The `sample` endpoint is often described as "using a straw to sip from the 
firehose." We'll start here

#### [POST statuses / filter](https://dev.twitter.com/streaming/reference/post/statuses/filter)

The `filter` API let's you filter the firehose for specific terms, creating a 
stream of just those tweets relevant to your search.

Before we play with twitter, we need to deal with an important preliminary: 
constructing a cache that will let us constrain our attention to just the recent 
past.

### Building a cache

One of my favorite out-of-the-box tools I use for caching python webscraping 
projects is the [`collections.deque`](https://docs.python.org/2/library/collections.html#collections.deque) 
class. The deque is extremely convenient because it is a LIFO (last in: first 
out) queue which right out of the box has an optional `maxlen` parameter 
defining the max number of entries it will store at once before new entries 
force the old ones out. 

The maxlen parameter is suitable for most of my caching needs, but for this 
particular project I don't have any fixed size of information I'm interested in 
(although it might be useful to use maxlen to at least set an upper bound). 
Instead, I want entries to expire after some fixed amount of time. To accomplish
this, we'll extend the `deque` class to build an "expiring deque" or "date 
limited deque" for lack of a better term.

Not surprisingly, Python has several robust libraries for caching, most notably 
[Beaker](https://beaker.readthedocs.org/en/latest/). But just for fun, we're
going to roll our own, which will have the added benefit of being extremely
lightweight and reducing external dependencies.

#### Requirements

I'm going to define a new class that will extend the deque by inheriting from 
it to get the base behavior, but then I'll add a few new features and modify
several existing methods to get the behavior I need. Here's how the new class 
will work:

1. Define a time-to-live (TTL) for items in the container.
2. Associate values with a timestamp.
3. Define a "flush" function that repeatedly checks if the oldest item in the container
has expired, removing items until all expired items have been dropped.
4. Modify base methods to call the flush function upon data access or 
modification.

#### Inheritance

The normal python class recipe is to inherit from the "object" class. To inherit
from the deque class, we simple specify this as the parent class in the class
definition, like so:

{% highlight python %}
from collections import deque

class DateDeque(deque): 
    """
    A deque whose entries expire after a set amount of time. Motivating
    use case is webscraping for trend detection.
    """
    pass
{% endhighlight %}

If we stopped here, the `DateDeque` class would be completely identical to 
`collections.deque`.

#### Overriding __init__

Next, we need a way to define the TTL for a given instance. Let's update the
`__init__` method so this can be passed in as a parameter when the instance is
created. A very convenient way to specify TTL is using the 
[`datetime.timedelta`](https://docs.python.org/2/library/datetime.html#datetime.timedelta)
class, which can take a variety of "time words" as input arguments.

Additionally, if we use `datetime.datetime` objects for our timestamps, we can 
compare them to our timedelta directly to calculate TTL. I want to attach the 
timestamp function to the class, but there's nothing class or instance specific
about this function, so we'll attach it as a static method to make this 
explicit.

{% highlight python %}
from collections import deque
from datetime import timedelta
import time

class DateDeque(deque): 
    """
    A deque whose entries expire after a set amount of time. Motivating
    use case is webscraping for trend detection.
    """
    def __init__(self, **kargs):
        self.delta = timedelta(**kargs)
        
    @staticmethod
    def timestamp():
        st = time.gmtime(time.time())
        return datetime(*st[:7])
{% endhighlight %}

#### Associating timestamps with values

Every value that gets inserted needs to have a timestamp associated with it so 
we can calculate TTL. We have two ways we could do this:

1. We could track entries and timestamps separately with a mapping between them
2. We can augment the entries with timestamps and track them together.

Option (1) is a bit fancier, but (2) is way easier and less prone to issues 
(i.e. if the mapping breaks).

As new values are added to the collection, we'll intercept them and replace the
original value with a tuple of the form `(timestamp, value)`. We'll need to add
a "date augment" function. This is essentially a private function, but python
doesn't have private functions. by convention, we proceed the function name with
and underscore to denote to anyone who might use the code that this function is
not meant to be called outside of the internal API.

{% highlight python %}
from collections import deque
from datetime import timedelta
import time

class DateDeque(deque): 
    """
    A deque whose entries expire after a set amount of time. Motivating
    use case is webscraping for trend detection.
    """
    def __init__(self, **kargs):
        self.delta = timedelta(**kargs)
    @staticmethod
    def timestamp():
        st = time.gmtime(time.time())
        return datetime(*st[:7])
        
    def _date_augment(self, x):
        return (self.timestamp(), x)
{% endhighlight %}

Now we need to actually intercept value insertions. For 
now, pretend we've defined the `flush` function. For a little added fanciness,
I'm going to override the `__len__` function so I can easily test the "true"
length of the object (i.e. by preceding the calculation with a flush event).

{% highlight python %}
from collections import deque
from datetime import timedelta
import time

class DateDeque(deque): 
    """
    A deque whose entries expire after a set amount of time. Motivating
    use case is webscraping for trend detection.
    """
    def __init__(self, **kargs):
        self.delta = timedelta(**kargs)
    @staticmethod
    def timestamp():
        st = time.gmtime(time.time())
        return datetime(*st[:7])
    def _date_augment(self, x):
        return (self.timestamp(), x)
        
    def append(self, X):
        super(DateDeque, self).append(self._date_augment(X))
        self.flush()
    def appendleft(self, X):
        super(DateDeque, self).appendleft(self._date_augment(X))
        self.flush()
    def extend(self, X):
        super(DateDeque, self).extend([self._date_augment(x) for x in X])
        self.flush()
    def extendleft(self, X):
        super(DateDeque, self).extendleft([self._date_augment(x) for x in X])
        self.flush()
    def pop(self, X):
        self.flush()
        d, v = super(DateDeque, self).pop([self._date_augment(x) for x in X])
        return v
    def popleft(self):
        self.flush()        
        d, v = super(DateDeque, self).popleft()
        return v
    def __len__(self):
        self.flush()
        return super(DateDeque, self).__len__()
{% endhighlight %}

#### Flush

With all the other pieces in place, the last thing we need to do is define the 
flush mechanism.

{% highlight python %}
from collections import deque
from datetime import timedelta
import time

class DateDeque(deque): 
    """
    A deque whose entries expire after a set amount of time. Motivating
    use case is webscraping for trend detection.
    """
    def __init__(self, **kargs):
        self.delta = timedelta(**kargs)
    @staticmethod
    def timestamp():
        st = time.gmtime(time.time())
        return datetime(*st[:7])
    def _date_augment(self, x):
        return (self.timestamp(), x)
    def append(self, X):
        super(DateDeque, self).append(self._date_augment(X))
        self.flush()
    def appendleft(self, X):
        super(DateDeque, self).appendleft(self._date_augment(X))
        self.flush()
    def extend(self, X):
        super(DateDeque, self).extend([self._date_augment(x) for x in X])
        self.flush()
    def extendleft(self, X):
        super(DateDeque, self).extendleft([self._date_augment(x) for x in X])
        self.flush()
    def pop(self, X):
        self.flush()
        d, v = super(DateDeque, self).pop([self._date_augment(x) for x in X])
        return v
    def popleft(self):
        self.flush()        
        d, v = super(DateDeque, self).popleft()
        return v
    def __len__(self):
        self.flush()
        return super(DateDeque, self).__len__()
        
    def flush(self):
        now = self.timestamp()
        while True:
            #if len(self) == 0: # This is circular
            if super(DateDeque, self).__len__() == 0:
                break
            d = now - self[0][0]
            if d > self.delta:
                super(DateDeque, self).popleft()
            else:
                break
{% endhighlight %}

And there we have it: a deque augmented to support time-based expiration of 
entries! This class is going to form the basic building block of our cache.

### Customizing the cache