# Apache SkyWalking (apache-skywalking)

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Apache SkyWalking is an open-source APM (Application Performance Monitoring) system that provides monitoring, tracing, and diagnosing capabilities for distributed systems in cloud native architectures. It supports auto-instrumentation for Java, .NET, Python, Go, Node.js, PHP, and Ruby, offering distributed tracing, metrics collection, log aggregation, and continuous profiling through a unified observability platform governed by the Apache Software Foundation.

**URL:** [https://skywalking.apache.org/](https://skywalking.apache.org/)

**Run:** [Capabilities Using Naftiko](https://github.com/naftiko/fleet?utm_source=api-evangelist&utm_medium=readme&utm_campaign=company-api-evangelist&utm_content=repo)

## Tags:

 - APM, Application Performance Monitoring, Cloud Native, Distributed Tracing, Monitoring, Observability, Open Source, Tracing

## Timestamps

- **Created:** 2026-03-16
- **Modified:** 2026-04-19

## APIs

### Apache SkyWalking GraphQL Query API
The SkyWalking GraphQL Query API provides a comprehensive query interface for retrieving observability data including traces, metrics, logs, alarms, topology maps, and profiling results. It supports metadata queries (services, instances, endpoints), topology queries, metrics via SkyWalking Metrics Query Expression (MQE), log queries, trace queries, alarm queries, and profiling queries. The API is served on port 12800 and consumed by the native UI and CLI tools.

**Human URL:** [https://github.com/apache/skywalking-query-protocol](https://github.com/apache/skywalking-query-protocol)

#### Tags:

 - GraphQL, Metrics, Observability, Tracing, Logs, Alarms, Profiling

#### Properties

- [Documentation](https://skywalking.apache.org/docs/main/next/en/api/query-protocol/)
- [GitHubRepository](https://github.com/apache/skywalking-query-protocol)

### Apache SkyWalking REST API
The SkyWalking HTTP REST API exposes endpoints on port 12800 for health checks, PromQL-compatible metrics queries (Prometheus Query Language), LogQL log queries, and dynamic configuration management. PromQL support enables integration with Grafana and other Prometheus-compatible visualization tools.

**Human URL:** [https://skywalking.apache.org/docs/main/next/en/api/promql-service/](https://skywalking.apache.org/docs/main/next/en/api/promql-service/)

#### Tags:

 - REST, HTTP, PromQL, LogQL, Health Check, Metrics

#### Properties

- [Documentation](https://skywalking.apache.org/docs/main/next/en/api/promql-service/)
- [Documentation](https://skywalking.apache.org/docs/main/next/en/api/logql-service/)

### Apache SkyWalking gRPC Data Collect Protocol
The SkyWalking data collection protocol defines gRPC service definitions for telemetry data ingestion from language agents and service mesh proxies. It covers trace data (v3), JVM metrics, meter protocol, event reporting, browser performance data, instance properties, continuous profiling, eBPF profiling, and log data protocols. Agents report data to OAP server on port 11800.

**Human URL:** [https://github.com/apache/skywalking-data-collect-protocol](https://github.com/apache/skywalking-data-collect-protocol)

#### Tags:

 - gRPC, Protocol Buffers, Telemetry, Agents, Tracing, Metrics

#### Properties

- [Documentation](https://skywalking.apache.org/docs/main/next/en/api/trace-data-protocol-v3/)
- [GitHubRepository](https://github.com/apache/skywalking-data-collect-protocol)

## Common Properties

- [GitHubOrganization](https://github.com/apache?q=skywalking)
- [GitHubRepository](https://github.com/apache/skywalking)
- [Documentation](https://skywalking.apache.org/docs/)
- [Portal](https://skywalking.apache.org/)
- [Blog](https://skywalking.apache.org/blog/)
- [ReleaseNotes](https://github.com/apache/skywalking/releases)
- [Support](https://skywalking.apache.org/community/)
- [TermsOfService](https://www.apache.org/licenses/)
- [Java Agent SDK](https://github.com/apache/skywalking-java)
- [Python Agent SDK](https://github.com/apache/skywalking-python)
- [Go Agent SDK](https://github.com/apache/skywalking-go)
- [Node.js Agent SDK](https://github.com/apache/skywalking-nodejs)
- [PHP Agent SDK](https://github.com/apache/skywalking-php)
- [Ruby Agent SDK](https://github.com/apache/skywalking-ruby)
- [Rust Agent SDK](https://github.com/apache/skywalking-rust)
- [JavaScript Browser Agent SDK](https://github.com/apache/skywalking-client-js)

## Features

| Name | Description |
|------|-------------|
| Distributed Tracing | Auto-instrumented distributed tracing across 10+ languages with trace correlation and cross-service propagation. |
| Metrics Collection | Service, instance, and endpoint metrics with SkyWalking Metrics Query Expression (MQE) engine. |
| Log Aggregation | Centralized log collection and search with LAL (Log Analysis Language) rules. |
| Service Topology | Automatic service dependency mapping and topology visualization. |
| Alarm System | Rule-based alerting on metrics thresholds with webhook and notification integrations. |
| Continuous Profiling | CPU, memory, and network profiling via async-profiler, pprof, and eBPF. |
| eBPF Network Profiling | Out-of-process network performance profiling using eBPF without code instrumentation. |
| PromQL Compatibility | Prometheus Query Language API for Grafana and other Prometheus-compatible tools. |
| BanyanDB Storage | Native observability database optimized for time-series and trace data storage. |
| Multi-Layer Service Model | Hierarchical service model supporting mesh, Kubernetes, APISIX gateway, and custom layers. |

## Use Cases

| Name | Description |
|------|-------------|
| Microservices Observability | End-to-end monitoring and tracing for microservices architectures in Kubernetes. |
| Service Mesh Monitoring | Integration with Istio and other service meshes for traffic and performance monitoring. |
| Root Cause Analysis | Trace-based root cause analysis for distributed system failures and latency issues. |
| SLA Monitoring | Service level agreement monitoring with metrics dashboards and alerting. |
| Continuous Profiling | Always-on profiling for performance optimization without overhead in production. |

## Integrations

| Name | Description |
|------|-------------|
| Kubernetes | Native Kubernetes monitoring via skywalking-kubernetes Helm charts and event integration. |
| Grafana | PromQL-compatible metrics API enables native Grafana dashboard integration. |
| Istio | Service mesh telemetry collection from Istio-managed service traffic. |
| APISIX | API Gateway integration for monitoring API traffic through Apache APISIX. |
| Elasticsearch | Elasticsearch and OpenSearch backend storage for trace and log data. |
| BanyanDB | Native high-performance observability database built for SkyWalking. |
| Kafka | Kafka-based data pipeline for high-throughput telemetry ingestion. |
| OpenTelemetry | OpenTelemetry receiver for ingesting OTLP traces, metrics, and logs. |

## Maintainers

**FN:** Kin Lane

**Email:** info@apievangelist.com
