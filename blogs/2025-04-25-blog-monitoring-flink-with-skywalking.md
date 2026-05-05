---
title: "Blog: Monitoring Flink with SkyWalking"
url: "/blog/2024-04-19-flink-monitoring-by-skywalking/"
date: "Fri, 25 Apr 2025 00:00:00 +0000"
author: ""
feed_url: "https://skywalking.apache.org/feed"
---
<h1 id="background">Background</h1>
<p><a href="https://flink.apache.org/">Apache Flink</a> is a framework and distributed processing engine for stateful computations over unbounded and bounded data streams. Flink has been designed to run in all common cluster environments, perform computations at in-memory speed and at any scale.</p>
<p><a href="https://skywalking.apache.org/">Apache SkyWalking</a> is an application performance monitor tool for distributed systems, especially designed for microservices, cloud native and container-based (Kubernetes) architectures.</p>
<p><a href="https://opentelemetry.io/">OpenTelemetry</a> is a collection of APIs, SDKs, and tools. Use it to instrument, generate, collect, and export telemetry data (metrics, logs, and traces) to help you analyze your software’s performance and behavior.</p>
<p>Since <code>SkyWalking</code> 10.3, a new out-of-the-box feature has been introduced that enables Flink monitoring data to be visualized on the SkyWalking UI via the OpenTelemetry Collector, which gathers metrics from Flink endpoints.</p>
<h1 id="development">Development</h1>
<h2 id="preparation">Preparation</h2>
<ol>
<li><a href="https://github.com/apache/skywalking">SkyWalking OAP,v10.3 +</a></li>
<li><a href="https://github.com/apache/flink">Flink v2.0-preview1 +</a></li>
<li><a href="https://github.com/open-telemetry/opentelemetry-collector-contrib">OpenTelemetry-collector v0.87+</a></li>
</ol>
<h2 id="process">Process</h2>
<ol>
<li>Set up <code>SkyWalking</code> oap and UI.</li>
<li>Set up the <code>Flink</code> cluster By configuring <code>jobmanager</code> and <code>taskmanager</code> to expose prometheus http endpoints.</li>
<li>Set up <code>OpenTelemetry-collector</code>.</li>
<li>Run your job.</li>
</ol>
<h2 id="data-flow">Data flow</h2>
<p><img alt="" src="data-flow.png" /></p>
<h2 id="configuration">Configuration</h2>
<h3 id="docker-compose">docker-compose</h3>
<pre tabindex="0"><code>version: "3"

services:
  oap:
    extends:
      file: ../../script/docker-compose/base-compose.yml
      service: oap
    ports:
      - "12800:12800"
    networks:
      - e2e

  banyandb:
    extends:
      file: ../../script/docker-compose/base-compose.yml
      service: banyandb
    ports:
      - 17912

  jobmanager:
    image: flink:2.0-preview1
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        metrics.reporter.prom.factory.class: org.apache.flink.metrics.prometheus.PrometheusReporterFactory
        metrics.reporter.prom.port: 9260
    ports:
      - "8081:8081"
      - "9260:9260"
    command: jobmanager
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - e2e

  taskmanager:
    image: flink:2.0-preview1
    environment:
      - |
        FLINK_PROPERTIES=
        jobmanager.rpc.address: jobmanager
        metrics.reporter.prom.factory.class: org.apache.flink.metrics.prometheus.PrometheusReporterFactory
        metrics.reporter.prom.port: 9261
    depends_on:
      jobmanager:
        condition: service_healthy
    ports:
      - "9261:9261"
    command: taskmanager
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9261/metrics"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - e2e

  executeJob:
    image: flink:2.0-preview1
    depends_on:
      taskmanager:
        condition: service_healthy
    command: &gt;
      bash -c "
      ./bin/flink run -m jobmanager:8081 examples/streaming/WindowJoin.jar"
    networks:
      - e2e

  otel-collector:
    image: otel/opentelemetry-collector:${OTEL_COLLECTOR_VERSION}
    networks:
      - e2e
    command: [ "--config=/etc/otel-collector-config.yaml" ]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    expose:
      - 55678
    depends_on:
      oap:
        condition: service_healthy

networks:
  e2e:
</code></pre><p>If you plan to expose metrics data using the pushGateway pattern,
please refer to the <a href="https://nightlies.apache.org/flink/flink-docs-release-2.0-preview1/docs/deployment/metric_reporters/#prometheuspushgateway">documentation</a>.</p>
<h3 id="opentelemetry-collector">OpenTelemetry-collector</h3>
<pre tabindex="0"><code>receivers:
  prometheus:
    config:
      scrape_configs:
        - job_name: "flink-jobManager-monitoring"
          scrape_interval: 30s
          static_configs:
            - targets: ['jobmanager:9260']
              labels:
                cluster: flink-cluster
          relabel_configs:
            - source_labels: [ __address__ ]
              target_label: jobManager_node
              replacement: $$1
          metric_relabel_configs:
            - source_labels: [ job_name ]
              action: replace
              target_label: flink_job_name
              replacement: $$1
            - source_labels: [ ]
              target_label: job_name
              replacement: flink-jobManager-monitoring

        - job_name: "flink-taskManager-monitoring"
          scrape_interval: 30s
          static_configs:
            - targets: [ "taskmanager:9261" ]
              labels:
                cluster: flink-cluster
          relabel_configs:
            - source_labels: [ __address__ ]
              regex: (.+)
              target_label: taskManager_node
              replacement: $$1
          metric_relabel_configs:
            - source_labels: [ job_name ]
              action: replace
              target_label: flink_job_name
              replacement: $$1
            - source_labels: [ ]
              target_label: job_name
              replacement: flink-taskManager-monitoring

exporters:
  otlp:
    endpoint: oap:11800
    tls:
      insecure: true

processors:
  batch:
service:
  pipelines:
    metrics:
      receivers:
        - prometheus
      processors:
        - batch
      exporters:
        - otlp
</code></pre><p>Warning:<br />
Please do not edit the value of the <code>job_name</code> configuration, otherwise <code>SkyWalking</code> will not handle these data.<br />
<code>oap</code> means the address of your <code>SkyWalking oap</code> address,please replace it accordingly.<br />
Since the original <code>Flink metrics</code> contain the  <code>job_name</code> labels, and SkyWalking relies on the <code>job_name</code> label to handle OpenTelemetry data,
to avoid conflicts, we use <code>metric_relabel_configs</code> to rename the original <code>job_name</code> label to <code>flink_job_name</code>.</p>
<h1 id="metrics-definition">Metrics Definition</h1>
<p>Monitoring metrics involve in <code>Cluster Metrics</code>, <code>TaskManager Metrics</code>, and <code>Job Metrics</code>.</p>
<h2 id="cluster-metrics">Cluster Metrics</h2>
<p><img alt="" src="cluster-dashboard-1.png" />
<img alt="" src="cluster-dashboard-2.png" />
<img alt="" src="cluster-dashboard-3.png" /></p>
<p><code>Cluster Metrics</code> mainly focuses on statistics from the perspective of the entire cluster, as well as displaying JVM-related metrics of the JobManager, such as:</p>
<ul>
<li><code>Running Jobs</code>：The number of currently running jobs.</li>
<li><code>TaskManagers</code>：The number of TaskManagers.</li>
<li><code>Task Managers Slots Total</code>：The total number of TaskManager slots.</li>
<li><code>Task Managers Slots Available</code>：The number of available TaskManager slots.</li>
<li><code>JVM CPU Load</code>：The CPU load of the JobManager&rsquo;s JVM.</li>
</ul>
<h2 id="taskmanager-metrics">TaskManager Metrics</h2>
<p><img alt="" src="broker-dashboard-1.png" />
<img alt="" src="broker-dashboard-2.png" />
<img alt="" src="broker-dashboard-3.png" /></p>
<p><code>TaskManager Metrics</code> mainly focuses on statistics from the perspective of individual TaskManager nodes, such as:</p>
<ul>
<li><code>JVM Memory Heap Used</code>：The amount of JVM heap memory used on the TaskManager node.</li>
<li><code>JVM Memory Heap Available</code>：The amount of JVM heap memory available on the TaskManager node.</li>
<li><code>NumRecordsIn</code>：The number of records received per minute by the TaskManager.</li>
<li><code>NumBytesInPerSecond</code>：The number of bytes received per second by the TaskManager.</li>
<li><code>IsBackPressured</code>：Indicates whether the TaskManager node is under backpressure.</li>
<li><code>IdleTimeMsPerSecond</code>：The idle time per second of the TaskManager node.</li>
</ul>
<h2 id="job-metrics">Job Metrics</h2>
<p><img alt="" src="topic-dashboard-1.png" />
<img alt="" src="topic-dashboard-2.png" /></p>
<p><code>Job Metrics</code>mainly focuses on statistics from the perspective of running jobs, such as:</p>
<ul>
<li><code>Job RunningTime</code>：The duration for which the job has been running.</li>
<li><code>Job Restart Number</code>：The number of times the job has been restarted.</li>
<li><code>Checkpoints Failed</code>：The number of failed checkpoints.</li>
<li><code>NumBytesInPerSecond</code>：The number of bytes received per second by the job.</li>
</ul>
<p>You can find explanations for each metric in the tip of the corresponding chart.<br />
<img alt="" src="tip.png" /></p>
<h1 id="references">References</h1>
<ul>
<li><a href="https://nightlies.apache.org/flink/flink-docs-release-2.0-preview1/docs/deployment/metric_reporters/#prometheus">Flink Prometheus</a></li>
<li><a href="https://skywalking.apache.org/docs/main/next/en/setup/backend/backend-flink-monitoring/">SkyWalking Flink Monitoring</a></li>
</ul>
