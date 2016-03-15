---
author: Sourav Dey
author_twitter: "@resdntalien"
categories:
- data science
- machine learning
- algorithms
layout: post
title: "Data Science at Cortex: Bringing Intelligence to Towers"
---

Cortex's vision is to use data-science to make commercial building operations more efficient.  Over the next few posts we want to give readers a "look under the hood" of our data-science operations.  To that end, we'll discuss how we solved one of the foundational problems at Cortex: figuring out when a building's heating, ventilation, and air conditioning (aka HVAC) systems were turned on.

<!--break-->

## Motivation

This problem is important because HVAC is the biggest consumer of energy (i.e. money and emissions) in day to day building operations.  Most buildings have a lease obligations that require the inside air to be within a certain temperature range by a certain time, e.g. between 72-76 degrees by 8am during the week.  The key to efficiency is to turn on the HVAC systems just in time so that the building reaches the right temperature by the lease obligation time.  You don't want it late -- because then tenants (who pay a lot in rent) will complain it's too cold/too hot.  But you don't want it too early either -- because then the building ends up wasting energy and money.  Optimizing the start time to be just right can create hundreds of thousands of dollars of savings over the course of the year for a single building.  It's no joke. 

## Defining the Problem

But before we can optimize the building start time, we have to know the historical start time for a building.  When I first heard this problem I thought it was super simple.  Someone must just log when the building was turned on. Right?  Wrong. Turns out the world is not that simple. In fact, what it means to "turn on a building" is kind of fuzzy. There's no master "on" switch. HVAC in buildings consists of many moving pieces that are turned on at different times -- chillers, fans, etc. This really clouds the meaning of turning "on" a building. 

This is where most data science starts; without even a clear definition of the problem.  So before anything, let's look at some examples from buildings.  Because the one thing I've learned from doing data science for many years is: 

> "Do human learning before any machine learning."

In fact I think most data scientists don't do enough data exploration.  Even now, I often jump the gun and try to throw the data into a machine learning algorithm way too early in the process. Looking at few plots of example data is not enough.  I think you need to look at hundreds of examples to really understand what is going on.  

## Data Exploration

Anyways, now I'm ranting.  Let's look at some data instead.  Here's a plot of all the sensor data from a large commercial building on a weeknight.  The time range shown is from 8:00 PM the day before to 8:00 AM in the morning, which is the lease obligation start time.  There are four types of sensors in this particular building that we're focused on when identifying when the HVAC systems started up: (1) supply air temperature (SAT), (2) static pressure (in the air handler units), (3) electric demand, and (4) steam demand.  Each type is shown in a separate subplot.  There are multiple SAT and static pressure sensors in different parts of the HVAC system -- that's why those subplots have multiple traces.  By contrast, we have a single sensor for total building electric and steam demand.  The sensors are sampled every 15 minutes.   

<img src="https://s3.amazonaws.com/cortex-blog-content/images/easy_example.png" width="100%">

Looking at this sensor data, we observe a clear edge transition in most sensors when the building turns on around 12:00 AM: 


1.  Supply air temperature falls to 50 degrees.
2.  Static pressure goes up to well above 0.0.
3.  Electric demand goes up from its baseline of 2000 kW.
4.  Steam demand spikes significantly above 0.

This is great.  This shows that there is something learnable.  If I just looked at this example, and others like it, I would say the building start time is the first edge across all the relevant HVAC signals. In this case, that is 12:00 AM. Simple. Right?

Of course not. We didn't look at enough data.  This example is the best case scenario.  Often, some small portion of the HVAC system turns on early, before a human would actually say "the building HVAC systems started."   The plot below shows an example of that. 

<img src="https://s3.amazonaws.com/cortex-blog-content/images/hard_example.png" width="100%">

A couple of static pressure and SAT signals show an edge around 12:00 AM, but most of the other edges are around 5:00 AM.  In this case, as humans that understand building operations, we would say the building's HVAC began its startup sequence at 5:00 AM.  Just a few fans turned on at 12:00 AM, so that wasn't really the building's HVAC systems starting in any meaningful way.

In this case, the simple rule from above would fail because it would say 12:00 AM is the start time.  But we can amend the rule to still work: If there are too few edges near the earliest edge, then it's not the start time. The start time is the first edge when some critical mass of HVAC signals have an edge.  

This is good.  We're doing the human learning that is the precursor to machine learning.  But we need to look at more data.  After a more thorough look, we found (to the surprise of no one) that the story is more complicated.  Some nights the building is on all night, e.g. it's so cold that the HVAC systems ran all night.  Other days the HVAC system never turns on.  Here's an example of one such day when the building ran all night. 

<img src="https://s3.amazonaws.com/cortex-blog-content/images/on_all_night.png" width="100%">

Inspecting the data, it seems that if there are very few edges, and the static pressure is high all night, then the building was on all night.  Alternatively, if there are very few edges and the static pressure is low all night, then the building was off all night.  

Further complicating matters is that different buildings have different sensors.  Some have steam, some don't. Some have just a few static pressure sensors, others have eighty or more.  How do we weigh the evidence across different sensors of different types?  Looking at the data, it seems that we can treat all sensors equally.  But this assumption may be wrong.  It's something to keep in mind as we're evaluating different algorithms. 

## Solution Skeleton

In the end, we looked at hundreds of days across all of our buildings at various times of the year -- spring, summer, fall, winter, weekdays, weekends, holidays, buildings where there are lots of sensors, buildings where there are few sensors. After a while, a "solution skeleton" started to come together.  We thought we could build an automatic start time finder using two algorithms: 


1.  **An edge finder algorithm.**  This would be an algorithm that, given a single time series, would identify sharp transitions (edges) in it.  
2.  **A start time estimator from edges.** This would be an algorithm that, given a set of edges from all the sensors, would figure out the start time -- if there was one.  If not, it would tell us if the building was on or off all night. 

Across the next few posts, we'll put some meat on this skeleton, discussing the development of this algorithm.
