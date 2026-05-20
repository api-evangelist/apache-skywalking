---
title: "Blog: Profiling Java application with SkyWalking bundled async-profiler"
url: "/blog/2024-12-09-skywalking-async-profiler/"
date: "Mon, 09 Dec 2024 00:00:00 +0000"
author: ""
feed_url: "https://skywalking.apache.org/feed"
---
Background Apache SkyWalking is an open-source Application Performance Management system that helps users gather logs, traces, metrics, and events from various platforms and display them on the UI. In version 10.1.0, Apache SkyWalking can perform CPU analysis through eBPF, which supports multiple languages, but not Java. This article discusses how Apache SkyWalking 10.2.0 uses async-profiler to collect CPU, memory allocation, and locks for analysis, solving this limitation, and also provides memory allocation and occupancy analysis.
