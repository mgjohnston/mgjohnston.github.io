---
layout: post
title: Mapping all my runs in Cambridge to celebrate a 70-week streak
categories: [running]
tags: [running, cambridge, coding]
---
Strava proudly told me today I had achieved a 70-week activity streak; meaning I've recorded something every calendar week back to January 2025.

This gladly gave me an excuse to play with the Strava API and build a small app I've been thinking about. It started when I saw a post a long time ago about someone who ran every road in their area[^hn-original]. Since then, I've seen [promotional pizza drawings](https://www.linkedin.com/posts/marcel-khan-a2579362_january-does-what-january-does-dry-jan-share-7414654516324872192-w4BH/) and there are loads of websites that allow you to [get a poster of your half marathon]({% post_url 2026-03-08-CambridgeHalf2026 %}). Similar to running every road, I wanted to see everywhere I've been in the last 70 weeks. 

I built a simple UI over the Strava API to pull the last N weeks, and draw them out on a map. 

![Cambridge Runs UI]({{ site.baseurl }}/images/cambridge-runs-ui.jpeg)

You can find the code [here](https://github.com/mgjohnston/strava-map/tree/main/cambridge-runs)

It's a fun map! You can see my local parkrun, my favourite route along the river and the route to squash. I've covered just over 700 km in and around Cambridge, out of 1,090 km of total distance in the 70 weeks!

![Heatmap of 121 runs around Cambridge]({{ site.baseurl }}/images/cambridge-runs-2025-01-01_2026-05-27.png)

[^hn-original]: Long since forgotten the original - similar to [this](https://news.ycombinator.com/item?id=34168284) - but I remember some code to find the missing roads, yet unrun.
