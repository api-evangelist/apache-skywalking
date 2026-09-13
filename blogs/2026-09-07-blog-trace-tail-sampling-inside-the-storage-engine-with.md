---
title: "Blog: Trace Tail Sampling Inside the Storage Engine with SkyWalking 11 and BanyanDB 0.11"
url: "https://skywalking.apache.org/blog/2026-09-07-banyandb-trace-tail-sampling/"
date: "2026-09-07"
author: "Kai Wan"
feed_url: "https://skywalking.apache.org/feed.xml"
---
SkyWalking 11.0.0 and BanyanDB 0.11.0 add trace tail sampling that runs inside the BanyanDB data node during compaction: each trace is judged as a whole after it is stored, and a dropped trace reclaims space that was already written.
