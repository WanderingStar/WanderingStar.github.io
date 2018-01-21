---
layout: post
title:  "Bulk Download Garmin Connect activities as GPX"
date:   2018-01-21 13:08:00
tags: javascript GPS gpx Garmin
summary: >
  Bulk Download Garmin Connect activities as GPX

---

I mentioned in my [last post]({{page.previous.url}}) that I'd discovered a new way
to hoard location data... [Fog of World](https://fogofworld.com).

Of course, I started out by importing all of the GPX files from previous adventures
that I had lying around (unfortunately, some of my adventures didn't have tracks,
particularly the [Rally](http://wander.ingstar.com/adventures/rally.html), :sigh:),
which got me thinking about what other data I have lying around.

I've had a succession of bike GPSes, and have tracked their data on Garmin Connect.
Garmin Connect is nice enough to let you download GPX files of your activities...
one at a time. I have more than 700 of these (mostly very repetitive), so doing
that by hand is a non-starter.

As a mostly-backend coder, my first thought was to write a scraper to download them
all, but that would mean the usual hassle with logins, but a search turned up an
easier way: run javascript in the browser to do it for you...

[Pastebin]()https://pastebin.com/YN6Gex5R)

In case that goes away, here's the code:

{% highlight javascript %}

/*
Here's javascript that can be run in any modern browser fairly simply in javascript. Might be easier to set up than Python.

You'll want to pre-set a download location in your browser settings to some folder, name it gpx or something, and tell your browser to auto-download there, or else you'll get a ton of popup save dialogs.

First Navigate to the last (most recent) activity you have in Garmin Connect (as in https://connect.garmin.com/modern/activity/5555555555 ), then hit F12 (should work in chrome/IE) to open dev tools to get to the Javascript Console. Then paste the below code and hit enter to run it. Can change ttl from 100 to whatever # of activities you want to download.
If you want a different format, change the "gpx" part of the URL to the appropriate format acronym if garmin supports it.
If your connection is too slow to do a full download in less than 3 seconds every time, change the downloadTimeoutLength from 3 * 1000 to whatever number you want (it's 3*1000 because that's 3000 milliseconds = 3 seconds).

[CODE]*/
var a = window.location.pathname.split("/");
var id = a[a.length-1];
var gpxUrl = "https://connect.garmin.com/modern/proxy/download-service/export/gpx/activity/";
var cnt = 1, ttl = 1000; /*Change ttl from 1000 to whatever # of activities you want to download*/
var downloadTimeoutLength = 3 * 1000;
var downloadUrl = gpxUrl + id;
window.location.href = downloadUrl;

setTimeout(
   (getMore = function(){
	jQuery.getJSON("https://connect.garmin.com/modern/proxy/activity-service/activity/"+id+"/relative?_="+Math.random()
		,function(d){
			id = d.previousActivityId;
			downloadUrl = gpxUrl + id;
			window.location.href = downloadUrl;
			if(++cnt<ttl){
				setTimeout(getMore,downloadTimeoutLength );
			}
		}
	);
   })
   ,downloadTimeoutLength 
);
/*[/CODE]

It goes from most recent back downloading each one. If you don't put the right total # to download them all, just navigate to the last one it got and re-run from there.*/
{% endhighlight %}
