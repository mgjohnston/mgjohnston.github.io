---
layout: post
title: Petrol has never been so cheap
categories: [blog, coding]
tags: [petrol, R]
---
I remember as a child, that petrol was always under £1/litre. During the current pandemic, it hit the news that petrol, once again, was nearing the £1/litre mark. It made me wonder, how expensive is petrol today?

The [RAC](https://www.racfoundation.org/data/uk-pump-prices-over-time) generously provides this data, plotted below.

![Petrol Prices]({{ site.baseurl }}/images/PetrolPriceUnadj.png)

However, over time the value of the pound has changed, with year-on-year inflation. Thus, even though the current price of petrol may be over a pound, it may be under a pound in uninflated money. The Government provides [information on inflation](https://www.ons.gov.uk/economy/inflationandpriceindices/timeseries/cdko/mm23). I used this data to make a constant UK 2000 pound.

![Pound Inflation]({{ site.baseurl }}/images/AdjPound.png)

Using this data to correct the price of petrol to a 2000 £, we can see petrol prices are remarkably static around 70p/litre (2000 £). Most noteably, since 2000 petrol has never been so cheap!

![Adjusted Petrol Prices]({{ site.baseurl }}/images/PetrolPriceAdj.png)
