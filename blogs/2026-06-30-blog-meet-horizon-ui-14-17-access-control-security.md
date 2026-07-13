---
title: "Blog: Meet Horizon UI · 14/17: Access Control & Security"
url: "https://skywalking.apache.org/blog/2026-06-30-horizon-ui-access-control-and-security/"
date: "2026-06-30"
author: ""
feed_url: "https://skywalking.apache.org/feed.xml"
---
This is the fourteenth post in the Meet Horizon UI series, and it opens Act 4 — govern &amp; secure it . Everything so far was about what Horizon shows and does ; this post is about who gets to see and do it . And one architectural point matters up front: all of this — RBAC, authentication, the audit log, break-glass, themes — is Horizon&rsquo;s own governance, enforced in its BFF , and it does not touch the OAP admin host.
