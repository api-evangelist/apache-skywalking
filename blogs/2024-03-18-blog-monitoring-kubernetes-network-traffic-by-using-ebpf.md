---
title: "Blog: Monitoring Kubernetes network traffic by using eBPF"
url: "/blog/2024-03-18-monitor-kubernetes-network-by-ebpf/"
date: "Mon, 18 Mar 2024 00:00:00 +0000"
author: ""
feed_url: "https://skywalking.apache.org/feed"
---
Background Apache SkyWalking is an open-source Application Performance Management system that helps users gather logs, traces, metrics, and events from various platforms and display them on the UI. With version 9.7.0, SkyWalking can collect access logs from probes in multiple languages and from Service Mesh, generating corresponding topologies, tracing, and other data. However, it could not initially collect and map access logs from applications in Kubernetes environments.
