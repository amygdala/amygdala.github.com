<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN"
   "http://www.w3.org/TR/html4/loose.dtd">

<html lang="en">
<head>
	<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
	<title>Instructions for configuring and running the (alpha) Twitter Realtime Plugin</title>
	<meta name="generator" content="TextMate http://macromates.com/">
	<meta name="author" content="Amy Jo">
	<link rel="stylesheet" href="./css/bright.css" type="text/css" charset="utf-8" media="screen">
	<!-- Date: 2011-01-27 -->
</head>

<body style="margin:30px;" class="bright">
    
# Instructions for configuring and running the (alpha) Twitter Realtime Plugin #

## Introduction ##

This document describes how to set up and run the currently-in-**alpha** Twitter Realtime plugin. To set it up in its current state, you'll need to be comfortable with the command line. These instructions assume that you are already familiar with how to configure and use ThinkUp.

The Twitter Realtime plugin sets up a persistent connection to the Twitter [User Streams API](http://dev.twitter.com/pages/user_streams "User Streams | dev.twitter.com"), for each active Twitter account (instance) in your ThinkUp installation, using a modified version of the [Phirehose](http://code.google.com/p/phirehose/ "phirehose - Project Hosting on Google Code") libs (included). In future, when the Twitter Site Streams API is out of beta, Site Streams will be supported by this plugin as well.

In order to absorb spikes in the data stream, the plugin uses a 'message queue' to decouple the process(es) that pull items off the stream(s) from the process that parses the items and adds information to the ThinkUp database.  
The plugin supports two different implementations of this message queue.  The first uses a new database table (`tu_stream_data`).  The second uses the [Redis](http://redis.io/ "Redis") persistent key-value store to support the message queue. (Gory details supplied on request, especially if you are interested in Redis' capabilities.)  

Redis is lightweight, robust, and efficient, and reduces database load; and so its use should be preferable in general to the db-based implementation.  However, the plugin can only use the Redis implementation if you are running a version of PHP >= 5.3. (Details: this is due to the Redis client libs we're using, [Predis](https://github.com/nrk/predis); and in particular the specific version of those libs that we are using.  If you really want to use Redis with PHP 5.2, it is possible with a bit of extra code modification.)

### Alpha Testing ###

For alpha testing, it will be useful to test this plugin using a variety of platforms and configurations.  It's been primarily tested thus far on OS X with PHP 5.3.3, and there should be no issues running on Linux.  We are looking for testers who run PHP 5.2 (to check that the plugin will properly fall back to using the db-based message queue instead of Redis, regardless of configuration settings).  It will be 'interesting' to see how the plugin runs on Windows+Cygwin. (On Windows, Cygwin will be required). It would be helpful also to have the Redis integration tested on different platforms.

## Setup and Config ##

Use this branch: [https://github.com/amygdala/thinktank/tree/streaming2](https://github.com/amygdala/thinktank/tree/streaming2) .

- Optional: install Redis, if you are willing to test that aspect of the plugin: [http://redis.io/](http://redis.io/).  It would be useful to us if you are willing to try it. You may need to compile it, which should be straightforward.

- back up/copy your database.  [There are no known database issues with this code, but do this to be on the safe side.]

- run the db migration `<thinkup>/webapp/install/sql/mysql_migrations/2011-01-28_streaming.sql`. [This assumes that your copy of the database is already otherwise current-- if you are starting with an older version of the ThinkUp database you may first need to run additional ThinkUp migrations as you would normally do to update to a new release of the master, as described in the ThinkUp documentation.]

- when you go to 'Settings' in your ThinkUp installation in the browser, you should now see a "**Twitter Realtime**" plugin.  Activate it, then go into its configuration page.  For the Twitter Realtime plugin to work properly, the Twitter Plugin must already be activated and configured with a Twitter app 'consumer key' and 'consumer secret'.  Additionally, you should have already set up one or more Twitter accounts via the Twitter Plugin.

- in the Twitter Realtime plugin config, enter the path to your php interpreter (e.g. `/usr/bin/php`). This is required.  Double check that you've typed it correctly.

- to use Redis, go into the advanced plugin options and enter 'true'.  If that is set, the plugin will expect to find a Redis server running (as described below). Otherwise, if it is left blank or set to something other than 'true', the db-based queue will be used. Also, as discussed above, if you are not running PHP >= 5.3, then the database queue will be used even if you configure the plugin to use Redis.

- I don't think this should be necessary, but if anyone had to do this, let me know.  Double check that your path to 'touch' is `/usr/bin/touch`.  If it is not, edit the 'touch' path in `<thinkup>webapp/plugins/twitter_realtime/model/class.StreamMasterCollect.php` [search for 'touch']

- The Twitter Realtime plugin should list all the Twitter accounts (instances) in your ThinkUp installation. This is the same list you see in the Twitter Plugin config page. Pause/start the Twitter instances as you prefer.  Any instances that you pause will be paused in the crawler as well. 
A Twitter User Stream will be opened for each active (unpaused) Twitter account in separately-running processes, up to 5 accounts max (to avoid running afoul of Twitter guidelines).  

- set `slog_location` in the `config.inc.php` file to point to where a new streaming log will be, e.g.:

    `$THINKUP_CFG['slog_location']   = $THINKUP_CFG['source_root_path'].'logs/streaming.log';`

  Then, you will also need to create that file, `streaming.log`. E.g., 

    <code><pre>
    % cd &lt;thinkup&gt;/logs
    % touch streaming.log
    </pre></code>

## Running the scripts ##

If you are using Redis, first start up the Redis server in its own terminal window if you want to use it: 
cd to `<redis_installation>/src` and start the server:

    % ./redis-server
    
To start up the instance streams, in another terminal window, cd to `<thinkup>webapp/crawler`.  Then do:

    % php stream.php stream <admin_user_email> <pwd>
    
(E.g., use the same admin email and pwd that you use to run the crawler). You should see some output to STDOUT along the lines of the following, which shows starting up three instances.  In this example, IDs 1 and 3 belong to one user, and ID 2 belongs to a different user.  A PID file for each instance process gets written to `<thinkup>/webapp/plugins/twitter_realtime/streaming`.  

        have streamer method: stream
        in TwitterRealtimePlugin->stream_all()
        shell_exec of /usr/bin/touch <thinkup>/webapp/plugins/twitter_realtime/streaming/57141.pid
        started pid 57141 for amy@infosleuth.net and instance id 1
        shell_exec of /usr/bin/touch <thinkup>/webapp/plugins/twitter_realtime/streaming/57144.pid
        started pid 57144 for amy@infosleuth.net and instance id 3
        shell_exec of /usr/bin/touch <thinkup>/webapp/plugins/twitter_realtime/streaming/57147.pid
        started pid 57147 for info+bob@infosleuth.net and instance id 2

The startup script will then exit.  A child script, `<thinkup>/webapp/plugins/twitter_realtime/streaming/stream2.php`, is used to launch the individual instance streams.  These are the processes which write to the message queue.  
[`shell_exec` is used to launch the child processes-- please let me know if this acts up for anyone].

At this point you might check that all the intended streams are running, via:

        ps auxw | grep -i stream2 

You should see a process for each instance.  You will also find some temp log files, for the output of each of the above processes, in the `<thinkup>/logs` dir. They will have the format `<user_email>_<inst_id>.log`.  If there were to be any trouble opening up the individual streams, these logs are where the problems would be reported.


Next, start up the stream processor. This is the process that reads from the message queue and processes the data. In another terminal window, again cd to `<thinkup>webapp/crawler`.
Then run:  

        % php stream.php stream_process <admin_user_email> <pwd>
        
Once this process is running, you will see output generated in the `<thinkup>/logs/streaming.log` file (or whatever streaming log location you specified in the config file).
  
Once all the scripts are up and running, you can see the new realtime content displayed right away in the web app.

There is no problem in running the crawler at the same time as the streaming scripts are running. (One thing the crawler will do is expand the URLs collected by the streaming processes, if you have the `Expand URLs` plugin activated).
  
        
### Shutting down the scripts ###

To shut down the stream handling processes for the Twitter instances, do from `<thinkup>webapp/crawler`:

    % php stream.php shutdown_streams <admin_user_email> <pwd>
    
You should see some output along these lines:

    have streamer method: shutdown_streams
    in TwitterRealtimePlugin->shutdown_streams()
    killing all running streaming processes
    found pid 57141
    found pid 57144
    found pid 57147
    killed: 57141
    deleting 57141.pid
    killed: 57144
    deleting 57144.pid
    killed: 57147
    deleting 57147.pid
    
If you should run `php stream.php stream <admin_user_email> <pwd>` while stream processes are already opened, it will kill them based on their PIDs and then open new streams.

To shut down the 'stream processor' script (the one you started via `php stream.php stream_process`), just ctl-C in its terminal window.   
You can use ctl-C to shut down the Redis server also.

</body>
</html>
