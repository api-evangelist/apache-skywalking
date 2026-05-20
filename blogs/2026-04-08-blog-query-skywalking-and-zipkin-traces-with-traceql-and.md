---
title: "Blog: Query SkyWalking and Zipkin Traces with TraceQL and Visualize in Grafana"
url: "/blog/2026-04-08-traceql/"
date: "Wed, 08 Apr 2026 00:00:00 +0000"
author: ""
feed_url: "https://skywalking.apache.org/feed"
---
Query SkyWalking and Zipkin Traces with TraceQL and Visualize in Grafana Apache SkyWalking introduced TraceQL support in version 10.4.0 , implementing Grafana Tempo’s HTTP query APIs so that Grafana can query and visualize traces stored in SkyWalking without any additional plugins. This means you can now use the familiar Grafana Tempo data source to search, filter, and drill into both SkyWalking native traces and Zipkin-compatible traces — all served by your existing SkyWalking OAP server. Architecture Overview ┌────────────────────┐ Tempo HTTP API ┌─────────────────────────────┐ │ │ ────…
