---
title: "Blog: Monitoring Nginx with SkyWalking"
url: "/blog/2023-12-23-monitoring-nginx-by-skywalking/"
date: "Sat, 23 Dec 2023 00:00:00 +0000"
author: ""
feed_url: "https://skywalking.apache.org/feed"
---
Background Apache SkyWalking is an open-source application performance management system that helps users collect and aggregate logs, traces, metrics, and events, and display them on the UI. In order to achieve monitoring capabilities for Nginx, we have introduced the Nginx monitoring dashboard in SkyWalking 9.7, and this article will demonstrate the use of this monitoring dashboard and introduce the meaning of related metrics. Setup Monitoring Dashboard Metric Define and Collection Since nginx-lua-prometheus is used to define and expose metrics, we need to install lua_nginx_module for…
