---
title: "Blog: Scaling with Apache SkyWalking"
url: "/blog/2022-01-24-scaling-with-apache-skywalking/"
date: "Mon, 24 Jan 2022 00:00:00 +0000"
author: ""
feed_url: "https://skywalking.apache.org/feed"
---
Background In the Apache SkyWalking ecosystem, the OAP obtains metrics, traces, logs, and event data through SkyWalking Agent, Envoy, or other data sources. Under the gRPC protocol, it transmits data by communicating with a single server node. Only when the connection is broken, the reconnecting policy would be used based on DNS round-robin mode.
