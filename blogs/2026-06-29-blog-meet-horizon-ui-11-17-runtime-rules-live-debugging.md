---
title: "Blog: Meet Horizon UI · 11/17: Runtime Rules & Live Debugging"
url: "https://skywalking.apache.org/blog/2026-06-29-horizon-ui-runtime-rules-and-live-debugging/"
date: "2026-06-29"
author: ""
feed_url: "https://skywalking.apache.org/feed.xml"
---
This is the eleventh post in the Meet Horizon UI series, and it stays in Act 3 — operate it . The previous post was about reading what the backend already decided; this one is about changing how it decides — and then proving the change does what you meant. Almost everything OAP computes runs through a small family of DSLs: OAL turns traces into service and endpoint metrics, MAL turns meters (OpenTelemetry, Telegraf) into metrics, LAL turns logs into tags and metrics.
