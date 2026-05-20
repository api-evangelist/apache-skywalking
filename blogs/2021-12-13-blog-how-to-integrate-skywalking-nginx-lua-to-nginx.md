---
title: "Blog: How to integrate skywalking-nginx-lua to Nginx?"
url: "/blog/2021-12-13-skywalking-nginx-agent-integration/"
date: "Mon, 13 Dec 2021 00:00:00 +0000"
author: ""
feed_url: "https://skywalking.apache.org/feed"
---
We Can integrate Skywalking to Java Application by Java Agent TEC.， In typical application, the system runs Java Web applications at the backend of the load balancer, and the most commonly used load balancer is nginx. What should we do if we want to bring it under surveillance? Fortunately, skywalking has provided Nginx agent 。 During the integration process, it is found that the examples on the official website only support openresty.
