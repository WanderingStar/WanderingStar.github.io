---
layout: post
title:  "Zoom to fit in ipyleaflet"
date:   2020-02-02 22:22:22
tags: javascript GPS gpx python leaflet jupyter
summary: >
  Zoom to fit in ipyleaflet

---

I've been messing around with JupyterLab and the
[ipyleaflet](https://github.com/jupyter-widgets/ipyleaflet) widget for displaying
mapping data.

I wanted to set up an interactive widget that allowed selection of a set of tracks
from a list, but ipyleaflet doesn't have a hook for Leaflet's
[flyToBounds](https://leafletjs.com/reference-1.6.0.html#map-flytobounds)
method, so I was having trouble making it focus the map on the right area.

Here's a quick and dirty scheme for displaying a particular bounding rectangle
in the map: make a map centered at the center of the bounds at the highest zoom
level, then zoom out repeatedly until the bounds are within the frame of the map.

One slightly tricky thing is that the map doesn't know its bounds until it's
drawn, so this has to be done reactively with a change observer.

{% highlight python %}
from traitlets import Tuple

m = Map(center=center(b), zoom=18, close_popup_on_click=False)

# define a handler that will get called each time the bounds change
def zoom_out_to_target_bounds(change):
    # the change owner is the widget triggering the handler, in this case a Map
    m = change.owner

    # if we're not zoomed all the way out already, and we have a target...
    if m.zoom > 1 and m.target_bounds:
        b = m.target_bounds
        n = change.new
        if (n[0][0] < b[0][0] and n[0][1] < b[0][1] and
            n[1][0] > b[1][0] and n[1][1] > b[1][1]):
            # bounds are already large enough, so remove the target
            m.target_bounds = None
        else:
            # zoom out
            m.zoom = m.zoom - 1

# Store the target bounds in the Map object itself
m.target_bounds = Tuple()
m.target_bounds = ((33.66832279243364, 135.8861750364304), (33.670108611729475, 135.89357256889346))

# use observe to make the handler get called when the bounds change
m.observe(zoom_out_to_target_bounds, 'bounds')
{% endhighlight %}
