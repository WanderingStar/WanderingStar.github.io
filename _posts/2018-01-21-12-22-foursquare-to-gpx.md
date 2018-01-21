---
layout: post
title:  "Foursquare to GPX"
date:   2018-01-21 12:22:00
tags: python GPS gpx 4sq Foursquare
summary: >
  Getting Foursquare history as GPX

---

I just got back from a trip to the Maldives and Dubai. I'm an inveterate data hoarder,
particularly when it comes to travel, and I had a GPS tracker on pretty much the whole
time. But extracting and labeling the interesting waypoints from the tracks would be
tedious. I did it for the 20 SCUBA dives (by correlating the GPS tracks and the times
recorded by my dive computer), but no way was I going to bother for the many places I
visited on land.

Luckily, I'm also a habitual Foursquare/Swarm user, so I'd "checked in" at most of the
interesting places anyway. But how to get that data out of Foursquare and into my GPS
tracks?

The results: [Maldives](http://wander.ingstar.com/adventures/maldives-map.html)
[Dubai](http://wander.ingstar.com/adventures/dubai-map.html)

Happily, there are a couple of Python libraries that make it easy. I adapted the examples
from the documentation for [foursquare](https://github.com/mLewisLogic/foursquare) and
[gpxpy](https://github.com/tkrajina/gpxpy) to write the following scripts.

You'll need to sign up as a [Foursquare Developer](https://developer.foursquare.com/)
and register an App (it doesn't need to actually do anything) to get a client_id
and client_secret to access the API. Make sure you specify a redirect_uri for your
app that exactly matches what you enter below...

First, I grab all of my Foursquare data and store it in a file in one-JSON-per-line format:

{% highlight python %}

import foursquare
import json

client = foursquare.Foursquare(client_id='GET THIS FROM developer.foursquare.com',
                               client_secret='GET THIS FROM developer.foursquare.com ',
                               redirect_uri='http://example.com')

auth_uri = client.oauth.auth_url()

print("Open this URL in your browser and approve the permissions")
print(auth_uri)
print("Copy the code from the URL you were redirected to and paste it here")
token = input("code? ")

# Interrogate foursquare's servers to get the user's access_token
access_token = client.oauth.get_token(token)

# Apply the returned access token to the client
client.set_access_token(access_token)

# Retrieve all of your checkins
checkins = client.users.all_checkins()

# Write the checkins to a file, one JSON object per line
with open("/tmp/4sq.json", "w") as f:
    i = 0
    for checkin in checkins:
        i += 1
        f.write(json.dumps(checkin) + "\n")
        if i % 100 == 0:
            print(i)

print(i)
{% endhighlight %}

Now let's make a GPX file for each month with a waypoint for each venue 
(keeping only the first time it was visited):

{% highlight python %}

import json
from datetime import datetime

from gpxpy.gpx import GPX, GPXWaypoint, GPXTrack, GPXTrackSegment, GPXTrackPoint

tracks_from_waypoints = False  # if this is True, makes a single point Track instead of a Waypoint

seen_names = set()
last_yrmo = None
gpx = None
revisits = 0

with open("/tmp/4sq.json", "r") as f:
    for line in f.readlines():
        checkin = json.loads(line)
        try:
            name = checkin['venue']['name']
            # only keep the first visit to a named venue
            # note that this might not be the right thing if there are multiple venues
            # with the same name. we could dedupe based on venue id or similar, if that
            # matters
            if name in seen_names:
                revisits += 1
                continue
            seen_names.add(name)

            lat = checkin['venue']['location']['lat']
            lon = checkin['venue']['location']['lng']
            date = datetime.fromtimestamp(checkin['createdAt'])
            yrmo = f"{date.year}-{date.month:02}"
            if last_yrmo != yrmo:
                # if the month has changed, save the file we're working on and start a new one
                if gpx:
                    filename = f"/tmp/4sq-{last_yrmo}.gpx"
                    print(f"{filename}: {len(gpx.waypoints) + len(gpx.tracks)}")
                    with open(filename, "w") as f:
                        f.write(gpx.to_xml() + "\n")
                gpx = GPX()
                last_yrmo = yrmo
            if tracks_from_waypoints:
                # Create first track in our GPX:
                gpx_track = GPXTrack()
                gpx.tracks.append(gpx_track)

                # Create first segment in our GPX track:
                gpx_segment = GPXTrackSegment()
                gpx_track.segments.append(gpx_segment)

                # Create points:
                gpx_segment.points.append(GPXTrackPoint(latitude=lat, longitude=lon, time=date))
            else:
                gpx.waypoints.append(GPXWaypoint(latitude=lat, longitude=lon, time=date, name=name))
        except:
            # a few checkins in my history don't have all of the expected fields
            print(checkin)

# write out the leftovers
filename = f"/tmp/4sq-{last_yrmo}.gpx"
print(f"{filename}: {len(gpx.waypoints) + len(gpx.tracks)}")
with open(filename, "w") as f:
    f.write(gpx.to_xml() + "\n")

print(f"Revists: {revisits}")
{% endhighlight %}

In the process of figuring out how to do this, I discovered [Fog of World](https://fogofworld.com),
a "game" where you try to visit the entire world. Happily, it lets you import GPS traces, so now
I have yet another way to hoard data. h/t [philsturgeon](https://gist.github.com/philsturgeon/4431748)
