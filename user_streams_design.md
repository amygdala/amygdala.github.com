<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN"
   "http://www.w3.org/TR/html4/loose.dtd">

<html lang="en">
<head>
	<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
	<title>Twitter Realtime plugin</title>
	<meta name="generator" content="TextMate http://macromates.com/">
	<meta name="author" content="Amy Jo">
	<link rel="stylesheet" href="./css/bright.css" type="text/css" charset="utf-8" media="screen">
	<!-- Date: 2011-01-27 -->
</head>
<body style="margin:30px;" class="bright">

# The Twitter Realtime Plugin #

## Introduction ##

This page has some information about the design and structure of the Twitter Realtime (User Streams) Plugin. More information as well as an editable to-do list for both planned near-term work and longer-term features, will be added shortly. We're interested in any comments and feedback on the design.  

Information about setup, configuration, and running the plugin scripts is [here](./setup_and_config.html).

This plugin adds new types of twitter data to the database, as described below.  
See the db migration `<thinkup>/webapp/install/sql/mysql_migrations/2011-01-28_streaming.sql` for the specifics of the new table schemas.
Currently, these new tables are relatively normalized.  For very large data sets, sometimes denormalized tables can be more workable.  We're interested in any feedback from people who've spent time thinking about similar database design issues, as now is the time make any schema modifications.

### JSON Stream items and ThinkUp data ###

The User Stream we're parsing contains JSON items, and a JSON parser/item handler has been added to ThinkUp.  You can look at the output of the new scripts, e.g. the script set up to process the items it pulls off the 'message queue', to see the structure of these messages.  If you do this, you'll notice that there is information we're not yet processing (though this plugin processes and stores more data than the crawler does).  It may be that we should be storing all of the data from each stream item, even if we don't yet make use of it.

## geo data handling ##

Where available, I'm storing both 'place' information and 'coordinates' information for a post. [If a user posts via twitter's web UI, 'place' but not 'coordinates' may be defined.  If a user posts via a mobile app, often the coordinate information is first specified and then an associated 'place' is included in the status for that location via Twitter's geo services.  I've also occasionally seen just the 'coordinates' set but not the 'place'].  

I'm storing this information using mysql spatial data types, specifically Polygon and Point. This representation is useful because it will allow us to perform locational queries on our data, e.g. 'contains' or 'intersects' queries.  The two new tables for this data are `tu_places` (holding place info) and `tu_post_locations` (storing the coordinates, if defined, for a post).  In addition, `tu_posts` has a new `place_id` field, storing the place (if defined) for a post.

A 'place' definition includes a bounding box. This polygon is stored in the 'bbox' field of the `tu_places` table. The 'longlat' field in the `tu_places` table is the centroid of this bounding box.  
To see readable data in these tables, do something like this:

      select id, place_id, place_type, name, full_name, ccode, country, AsText(longlat), AsText(bbox) from tu_places;
      select id, AsText(longlat), post_id, place_id from tu_post_locations;

It turns out that both 'place' and 'coordinates' data are long/lat, not lat/long as with the 'geo' field, which as I understand it is there just for backwards compatibility. Unfortunately, the existing geoencoder plugin uses the geo field (lat/long), and stores a point as a string, not a Point.  Clearly at some point we need a data migration to reconcile the location info. (I'd suggest converting the plugin's lat/long strings into long/lat Points).  For now, I have made no changes to the geo plugin, which continues to store user location and/or post 'geo' as lat/long strings. I figured this migration was best separated from the first iteration of the streaming implementation.

## 'entity' information-- hashtags, mentions, and urls ##

A post includes an 'entity' object, containing parsed-out information on the hashtags, mentions, and urls of the post.

For mentions and hashtags, there are two new tables for each.  One table can be viewed as storing mention or hashtag 'objects' (including a running count of how many times that object has been used), and the other is a join table indicating the posts that contain that mention or hashtag.

E.g., to see hashtags or mentions used more than 10 times, you can do something like this:

      select * from tu_mentions where count_cache > 10 order by count_cache desc;
or

      select * from tu_hashtags where count_cache > 10 order by count_cache desc;

[This count_cache value is only for the data collected by via the streaming API].

The 'mentions' information includes user id as well as user screenname.

The mentions list, particularly the first item in the list, is also used to ID 'old-style' retweets, by searching for a `"RT @first_mention"` string in the post text.  If that string is found, the `in_rt_of_user_id` field is set in tu_posts with the user id for that mention, regardless of whether the retweeted post itself can be located.

For tweet urls, an expanded url is sometimes included in the entity info. For now, expansions are only given for the 't.co' links.  I don't know if Twitter has plans to include the expansion for other shorteners in future or not.  For such urls, I store the expansion at the time the link record is created.

## stream 'events' ##

In addition to posts, the stream data  contains various other 'events'.  The only one I'm truly processing right now is the favorite event.  From the stream data, we now store both the favs of the authorized user, and others' favs of that user's posts.

For deletion events, I don't actually delete from the database, but do print out the deleted post text to STDOUT just for fun, if it is available.

For the other events, e.g. (un)follows, list additions, unfavorites, etc., I don't do any handling of them yet.
Also, you may notice that when you start up the stream, the first thing it sends is the user's list of friends.  I don't do anything with this list yet.  The friends information is of course updated by the crawler currently.

## 'message queue' implementation ##


As described [here](./setup_and_config.html), there are two implementation options for the 'message queue' that decouples the scripts that pull items off the stream, from the script that processes those items. One implementation uses Redis, and the other uses a database table as a queue, deleting each item from the database table as it is consumed.  

The Redis implementation should be faster and lighter-weight and has the advantage of removing load from the database.  Redis has a blocking list pop, and so the consumer script just uses that to wait for new data. (The blocking wait time is currently set to 5 minutes-- if that is exceeded without getting data, the script sleeps for a configurable period, and then reconnects to Redis and continues.  The stream sends a heartbeat/keep-alive, so normally there is no timeout issue).

The database implementation resets the auto increment count on script startup (and also if the id reaches a given max), just so it doesn't grow too large. 
If it does not find anything in the database queue, it sleeps for a configurable period and then tests the queue again.  

## Stream process management ##

Stream process management has changed from previous versions, which used PID flat files to track running streams.    

The app now stores stream process PIDs in the database-- which should be more robust-- and the individual UserStream processes (one for each 'owner') now generate an activity 'heartbeat' as they run.  This allows dead or wedged stream processes to be identified, killed, and restarted, while live stream processes can be left to run.  This lets a regularly-run script (e.g., a cron job) check for stream 'liveness' and restart the non-live processes as necessary.


</body>
</html>
