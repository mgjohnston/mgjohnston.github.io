---
layout: post
title: Download all of your tagged photos from Facebook
categories: [coding]
tags: [python, hobby, facebook]
---

With friends leaving Facebook everyday, I thought it was high time I archived my Facebook data. Facebook now allows you to download all of the data you've provided to them - photos, likes, posts, contacts and more - but they do not give you an option to download your tagged photos from friends.

I found an easy solution, edited from [Nathan Merrit's post](https://gnmerritt.net/deletefacebook/2018/04/03/fb-photos-of-me/)  [accessed 12/01/2020], javascript from [Novack](https://github.com/Novack), and a Python 2 script from [Xavier Ripoll](https://github.com/xaviripo).

1. Download this python script: <https://github.com/mgjohnston/fmpd/tree/patch-1>

2. Scroll to the bottom of your "Photos of you" page [<https://www.facebook.com/me/photos>] and run:

  `for (link of document.getElementsByTagName('a')) { if (!link.href.includes("?fbid=")) continue; console.log(new URL(link.href).searchParams.get("fbid")); }`

  Save the console output as "list.txt" in the same folder as the script and edit just to have the FBIDs of the photos. 

3. Obtain your cookies.txt for Facebook.

  I used [cookies.txt for Chrome](https://chrome.google.com/webstore/detail/cookiestxt/njabckikapfpffapmjgojcnbfjonfjfg) and name this "cookies.txt" in the same folder.

4. Use Python 2 to download the photos.

  `python download.py`

