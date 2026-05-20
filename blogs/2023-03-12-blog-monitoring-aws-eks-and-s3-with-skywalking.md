---
title: "Blog: Monitoring AWS EKS and S3 with SkyWalking"
url: "/blog/2023-03-12-skywalking-aws-s3-eks/"
date: "Sun, 12 Mar 2023 00:00:00 +0000"
author: ""
feed_url: "https://skywalking.apache.org/feed"
---
SKyWalking OAP’s existing OpenTelemetry receiver can receive metrics through the OTLP protocol, and use MAL to analyze related metrics in real time. Starting from OAP 9.4.0, SkyWalking has added an AWS Firehose receiver to receive and analyze CloudWatch metrics data. This article will take EKS and S3 as examples to introduce the process of SkyWalking OAP receiving and analyzing the indicator data of AWS services.
