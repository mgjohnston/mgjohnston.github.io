---
layout: post
title: 538Riddler - Can you stay in your lane?
categories: [ coding]
tags: [538riddler, R]
---
Answer to the [Riddler Express](https://fivethirtyeight.com/features/can-you-stay-in-your-lane/) (03/07) 

37/15 (2.47)

There are three categories of lane: edge (2/5), middle (1/5), and one from edge (2/5).

Following each of the possibilities of the first choice.

Edge (2/5). Then either a two person solution (1/3) or a three person solution (2/3).
Middle (1/5). Only has one three person solution (1)
One from edge (2/5). Has only two person solutions (1).

Summing these together: 1/5(2*2*1/3 + 2*3*2/3 + 1*3 + 2*2) = 37/15


##Riddler Express

It’s summertime and my local swimming pool, which has exactly five swimming lanes (and no general swim area), may be opening in the coming weeks. It remains unclear what social distancing practices will be required, but it’s quite possible that swimmers will not be allowed to occupy adjacent lanes.

Under these guidelines, the pool could accommodate at most three swimmers — one each in the first, third and fifth lanes.

Suppose a queue of swimmers arrives at the pool when it opens at 9 a.m. One at a time, each person randomly picks a lane from among the lanes that are available (i.e., the lane has no swimmer already and is not adjacent to any lanes with swimmers), until no more lanes are available.

At this point, what is the expected number of swimmers in the pool?
