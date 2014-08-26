Twitter Most Followed
=====================

Rationale
---------

**How to Find Out Who's Popular on Twitter.** And why there's no point in doing it

> That’s easy. You just look at the number of followers, and you’ll get @katyperry, @justinbieber, and @BarackObama — the top most followed accounts in the whole Twitterverse. No surprise there, right?

> But what if you want to focus on a particular group of Twitter users, for example the Hacker News community? Who’s in the top most followed accounts by the HNers? This is not a trivial exercise, we need a different approach, but if you’re a HNer, the result will be just as predictable.

Read the whole story here: https://medium.com/@ducu/d659884fd942


Approach
--------

Let's take the Hacker News community as our target group.

There's a Twitter account named [@newsyc20](http://twitter.com/newsyc20/) (Hacker News 20 - "Tweeting Hacker News stories as soon as they reach 20 points."). We consider this as our **source**, and the followers of @newsyc20 as the HNers, our **target group**. To find out the top most followed accounts by our target group members, we get the complete lists of who each of them is following (aka friends), and we aggregate all these lists, counting the occurences of each friend.

You can realize that the most followed account will be exactly @newsyc20, because all the members are following it. But who's on the 2nd and 3rd place? Who's in the top 100 most followed? This is what we're going to find out by running the routine below, a transcript from [main.py](https://github.com/ducu/twitter-most-followed/blob/master/main.py)


``` python
# Step 1: Specify the source
source_name = 'newsyc20' # target group source
source_id, source_data = load_user_data(screen_name=source_name)

# Step 2: Load target group members
followers = load_followers(source_id) # target group

# Step 3: Load friends of target group members
for follower_id in followers:
	load_friends(user_id=follower_id)

# Step 4: Aggregate friends into top most followed
aggregate_friends() # count friend occurences
top_most_followed(100) # display results
```


Requirements
------------

The Python script in this repo uses [Twitter REST API](https://dev.twitter.com/docs/api/1.1) to get the data, and [Redis](http://redis.io/) to store and aggregate it. To use Twitter API you need [an existing application](https://apps.twitter.com/), and [some access tokens](https://dev.twitter.com/docs/auth/obtaining-access-tokens).

There are a couple of performance issues though when dealing with big data sets.

**Twitter API Rate Limits**

We have 13.3K members in our HNers target group. In order to load the friends for each of these members (step 3), we're calling the Twitter API [friends/ids](https://dev.twitter.com/docs/api/1.1/get/friends/ids) method. This method is rate limited at 15 calls/15 minutes/token. We have to perform about 15.3K calls, since one call returns at most 5000 items. The problem is that with a single access token, it would take 10 days and 16 hours to get all this data.

[Tweepy](https://github.com/tweepy/tweepy/) is the preferred Twitter API client for Python, and the current release works with a single access token. But here's a fork I created especially to extend Tweepy so it works with several access tokens in a round robin fashion transparently - https://github.com/svven/tweepy. Using about four dozen tokens, the overall retrieval time was reduced to 5 hours (2 hours work time, 3 hours sleep). With about hundred tokens added to `RateLimitHandler`, you would get maximum efficiency out of a single Tweepy API object. See how it's done in `get_api()` from [twitter.py](https://github.com/ducu/twitter-most-followed/blob/master/twitter.py).

**Redis ZUNIONSTORE**

After storing all this data, we have about 12.3K simple sets of friend ids in Redis, one set for each of our target group members. We are short of 1K sets because there are that many protected Twitter accounts so we cannot get their friends from Twitter API. There's an average of 1.3K items per set, ranging from 1 to 8.6M maximum items, a total of 16.2M items.

Aggregating all these sets can be easily done using the ZUNIONSTORE command with the default weight of 1. See `RedisStorage.set_most_followed()` method in [storage.py](https://github.com/ducu/twitter-most-followed/blob/master/storage.py). The problem is that for this workload, ZUNIONSTORE took more than 1 hour to execute on my 4GB machine. That was surprisingly slow, having a recent stable release of Redis, ver 2.8.9.

It turned out that a [performance patch](https://github.com/antirez/redis/pull/1786) for this command has been recently added, but it is only available in the beta 8 release of Redis, ver 3.0.0. You can read about it in the [release notes](https://raw.githubusercontent.com/antirez/redis/3.0/00-RELEASENOTES). Having installed this, running the ZUNIONSTORE on the same data set took less than 2 minutes.

---

To conclude, in order to run this exercise for a big data set, make sure you have a bunch of access tokens that you can use in [config.py](https://github.com/ducu/twitter-most-followed/blob/master/config.py), and install [Redis 3.0.0 beta 8](https://github.com/antirez/redis/archive/3.0.0-beta8.tar.gz). Then `pip install -r requirements.txt` in your [virtualenv](http://virtualenv.readthedocs.org/en/latest/virtualenv.html) so you have following packages

```
hiredis==0.1.4
redis==2.10.1
git+https://github.com/svven/tweepy.git#egg=tweepy
```


Credits
-------

Thanks to Jeff Miller ([@JeffMiller](https://twitter.com/JeffMiller)) for @newsyc20. It's one of the best Hacker News Twitter bots. Jeff actually did a similar analysis on the Hacker News community, but with a slightly different [approach](http://talkfast.org/2010/07/28/twitter-users-most-followed-by-readers-of-hacker-news/).

Many thanks also to Josiah Carlson ([@dr_josiah](https://twitter.com/dr_josiah)) for his valuable support on Redis related issues.


Results
-------

Finally here's the top 100 most followed accounts by the Hacker News community.

