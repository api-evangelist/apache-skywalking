---
title: "Blog: Extending Apache SkyWalking with non-breaking breakpoints"
url: "/blog/2021-12-06-extend-skywalking-with-nbb/"
date: "Mon, 06 Dec 2021 00:00:00 +0000"
author: ""
feed_url: "https://skywalking.apache.org/feed"
---
Non-breaking breakpoints are breakpoints specifically designed for live production environments. With non-breaking breakpoints, reproducing production bugs locally or in staging is conveniently replaced with capturing them directly in production. Like regular breakpoints, non-breaking breakpoints can be: placed almost anywhere added and removed at will set to fire on specific conditions expose internal application state persist as long as desired (even between application reboots) The last feature is especially useful given non-breaking breakpoints can be left in production for days, weeks,…
