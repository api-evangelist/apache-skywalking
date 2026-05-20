---
title: "Blog: Monitoring DynamoDB with SkyWalking"
url: "/blog/2023-03-13-skywalking-aws-dynamodb/"
date: "Mon, 13 Mar 2023 00:00:00 +0000"
author: ""
feed_url: "https://skywalking.apache.org/feed"
---
Background Apache SkyWalking is an open-source application performance management system that helps users collect and aggregate logs, traces, metrics, and events, and display them on the UI. Starting from OAP 9.4.0, SkyWalking has added AWS Firehose receiver , which is used to receive and calculate the data of CloudWatch metrics. In this article, we will take DynamoDB as an example to show how to use SkyWalking to receive and calculate CloudWatch metrics data for monitoring Amazon Web Services.
