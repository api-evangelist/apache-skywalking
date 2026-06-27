---
title: "Blog: Meet Horizon UI · 7/17: The Log Explorer"
url: "/blog/2026-06-23-horizon-ui-log-explorer/"
date: "2026-06-23"
feed_url: "https://skywalking.apache.org/feed.xml"
---
This is the seventh post in the Meet Horizon UI series. Part 6 was one request’s spans; this one is the log lines around it. Horizon surfaces logs through two distinct tabs , because there are really two different questions: “what did this service log over the last half hour?” and “what is this pod printing to stdout right now?” The Logs tab queries the logs SkyWalking has already collected and stored — indexed, filterable, correlated with traces.
