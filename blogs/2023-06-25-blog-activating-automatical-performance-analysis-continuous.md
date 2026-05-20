---
title: "Blog: Activating Automatical Performance Analysis -- Continuous Profiling"
url: "/blog/2023-06-25-intruducing-continuous-profiling-skywalking-with-ebpf/"
date: "Sun, 25 Jun 2023 00:00:00 +0000"
author: ""
feed_url: "https://skywalking.apache.org/feed"
---
Background In previous articles, We have discussed how to use SkyWalking and eBPF for performance problem detection within processes and networks . They are good methods to locate issues, but still there are some challenges: The timing of the task initiation : It’s always challenging to address the processes that require performance monitoring when problems occur. Typically, manual engagement is required to identify processes and the types of performance analysis necessary, which cause extra time during the crash recovery.
