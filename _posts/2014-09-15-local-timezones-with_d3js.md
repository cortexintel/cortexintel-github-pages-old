---
author: Geoff Harcourt
author_twitter: "@geoffharcourt"
categories: javascript d3
layout: post
title: 'Plotting data with non-local timezones in D3.js'
---

[D3](http://d3js.org) is a versatile JavaScript library for building data-driven
visualizations in HTML and SVG. At Cortex, we use D3 on every page in our
dashboards to display metrics and recommendations to our users, often layering
two different but related pieces of data in the same chart to provide insight to
users. D3 is an incredibly powerful and flexible library, but that power comes
with a steeper learning curve relative to some other charting solutions.

<!--break-->

Soon after Cortex's launch, we noticed a problem when we viewed data about
buildings in New York from computers in California that had never surfaced in
our automated testing or manual QA efforts: all of our charts for New York
buildings displayed times on data points or X-axis tick marks as Pacific
Standard Time (PST) rather than New York's Eastern Standard Time (EST). While
one could argue that it's nice to see data translated into your own time zone
wherever you are, it's not convenient to think about building operations
remotely from a time zone that differs from the building's own.

I was initially confused about this behavior, as we output times from our chart
data API in [Coordinated Universal Time
(UTC)](http://en.wikipedia.org/wiki/Coordinated_Universal_Time) and specify the
local time zone of the building. Attempting to shift the chart data into the
building's local time in JavaScript with [Moment.js](http://momentjs.com) had no
effect, as D3 still translated the times into PST for the charts. After some
research, I found that D3's linear scale for time can support times in the
user's local time zone or UTC, but does not support fixing a linear time scale
to a different time zone, even if the data is marked with a different time zone.

My first attempt at a solution was to get the user's timezone from the browser
and submit it with the API request for chart data, compute the offset with in
Ruby, and then feed data back to the charts with "fake" times that would make
the chart's output appear to be on the correct times, essentially changing the
listed timezone for our data from EST to PST without changing the times
themselves. This behavior would probably work in most cases, but it felt
inelegant and was likely to produce problems whenever time zones didn't follow
the same daylight savings rules. This approach also produced undesired locations
for tick marks for the hours in the day, as D3's "intelligent" tick values
builder would dutifully mark off times every four hours, but rather than
starting at midnight, it would begin at 3am.

Our next attempt at solving the problem was to build our own D3 time scale that
would run alongside D3's `time.scale()` and `time.scale.utc()`. I still believe
this approach may be in our future, but after an initial code spike, it appeared
to be overkill for what we needed to accomplish immediately. D3's scales are
complicated functions with many moving parts, and writing and testing all the
parts of a new time scale type was going to be arduous.

The solution we arrived was a small Coffeescript formatter class that could
translate the displayed time for axis ticks or line points from the user's local
time to the building's specified time zone. We took advantage of this formatter
to override D3's (usually great) automatic axis tick creation to force ticks
at midnight, 4am, 8am, noon, 4pm, 8pm and the following midnight on all days,
even those lengthened or shortened by daylight savings changes.

{% highlight coffee %}
class Cortex.Graph.FixedLocalTimeFormatter
  constructor: (@presentedTimeZone) ->

  format: (momentFormattedTime) =>
    (d) =>
      moment.
        tz(d, Cortex.Utilities.browser_current_time_zone()).
        tz(@presentedTimeZone).
        format(momentFormattedTime)

  localMidnight: (data) =>
    earliestTimeInSet = d3.min data, (d) ->
      moment(Date.parse(d.measured_at))

    (earliestTimeInSet || moment()).
      tz(@presentedTimeZone).
      hours(0).
      minutes(0).
      seconds(0).
      milliseconds(0)

  ticksForDataSet: (data) =>
    xTickValues = for hour in [0, 4, 8, 12, 16, 20]
      @localMidnight(data).clone().hours(hour)

    # Add the following midnight to end the day
    xTickValues.push(@localMidnight(data).clone().add(1, "days"))
    xTickValues
{% endhighlight %}

`format()` takes the time which has been forced into the browser's local time by
D3 and turns it into a Moment time object. With Moment's excellent time zone
support, we assign it the browser's timezone (calculated with
[js-timezonedetect](http://pellepim.bitbucket.org/jstz/)) and then transform it
to the building's assigned time zone. `localMidnight()` allows us to always make
the beginning of a daily chart start at midnight in the building's time zone,
even if the data for the chart does not begin at midnight. `ticksForDataSet()`
produces fixed ticks at our desired times and doesn't get spoofed by DST
changes. Note that we're using doing something that looks laborious in our list
comprehension with Moment's `clone()` method because Moment time objects are
mutable, so adding a value or changing a characteristic of the object changes it
everywhere that the object is referenced.

We call this code in our chart classes when we perform tasks like generate
labels, tooltips, or ticks. Here's our x-axis generation code:

{% highlight coffee %}

  # Base time-series chart
  _fixedLocalTimeZoneFormatter: =>
    new Cortex.Graph.FixedLocalTimeFormatter(@_timeZone())

  _drawXAxis: =>
    xAxis = d3.svg.axis()
    .scale(xScale)
    .orient('bottom')
    .tickValues(@_fixedLocalTimeZoneFormatter().ticksForDataSet(@_ticksForDataSet()))
    .tickFormat(@_fixedLocalTimeZoneFormatter().format("h A"))

    # draw the axis and apply to the chart...

{% endhighlight %}

Building a small class of our own rather than bolting a huge new function on to
D3 resulted in much less code to write, no monkey-patching or other
hard-to-maintain changes to D3's existing scale functions, and much easier
testing. We can mock time zone detection in our JS command line test-runner to
simulate time zones around the world, and the entire class has three methods
that produce understandable, predictable results.

If you're not already using D3, check out creator and maintainer Mike Bostock's
excellent [visualization examples with code](http://bl.ocks.org/mbostock). If
you do anything with times or time zones in JavaScript, take a look at
[Moment.js](http://momentjs.com) and [Moment
Timezone](http://momentjs.com/timezone).
