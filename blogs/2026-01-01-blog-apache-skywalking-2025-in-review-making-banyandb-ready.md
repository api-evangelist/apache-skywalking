---
title: "Blog: Apache SkyWalking 2025 in Review: Making BanyanDB Ready for Production"
url: "/blog/2026-01-01-skywalking-2025-year-in-review/"
date: "Thu, 01 Jan 2026 00:00:00 +0000"
author: ""
feed_url: "https://skywalking.apache.org/feed"
---
<p>2025 was a very focused year for the Apache SkyWalking community: <strong>moving BanyanDB from “native storage” to a “production-ready default”</strong>, and making SkyWalking APM fully benefit from that foundation.</p>
<p>This post summarizes the key milestones, with an emphasis on BanyanDB.</p>
<h2 id="storage-strategy-saying-goodbye-to-h2">Storage strategy: saying goodbye to H2</h2>
<p>We started 2025 with a clear direction: the <strong>H2 storage option is permanently removed</strong>.
This change reduced long-term maintenance burden and removed a storage choice that was not aligned with production and cloud-native deployments.</p>
<h2 id="banyandb-from-080-foundations-to-090-production-features">BanyanDB: from 0.8.0 foundations to 0.9.0 production features</h2>
<p><strong>BanyanDB 0.8.0</strong> delivered the “day-2 operations” foundation that a default storage backend needs. The community put a lot of effort into making queries faster and more predictable (for example <code>index_mode</code>, numeric index types, and multiple query-path optimizations), while also making the system safer under real production pressure. Disk-usage thresholds and a query <strong>memory protector</strong> were added as guardrails, and the operational toolbox matured with snapshot/backup/restore utilities and improved metadata synchronization.</p>
<p>Just as importantly, 0.8.0 started filling in the missing pieces of a full platform: native property storage and lifecycle-related capabilities that later enabled stronger HA and stage-based deployment patterns.</p>
<p><strong>BanyanDB 0.9.0</strong> was the “production features” milestone. It introduced the <strong>Trace</strong> data model as a first-class citizen, which unlocked much deeper trace storage and query capabilities. On the reliability and scaling side, the release brought configurable replicas, liaison-side improvements (including load balancing and moving some TopN flow), and broader correctness work such as migrations, version compatibility checks, and access logs.</p>
<p>It also made long-term operations more cloud-friendly with backup/restore support for AWS S3, GCS, and Azure Blob Storage, and added authentication primitives needed in shared environments. In short, 0.9.0 is where BanyanDB clearly moved beyond a “fast storage engine” into a “production platform”.</p>
<h2 id="skywalking-apm-banyandb-becomes-the-default-path">SkyWalking APM: BanyanDB becomes the default path</h2>
<p>With <strong>APM 10.2.0</strong>, the project made the strategic shift official: H2 was removed permanently, and BanyanDB 0.8.0 became the default path that SkyWalking invests in. A lot of the work here was not flashy, but essential — refining OAP behavior (group settings, index model changes, Progressive TTL, query limits, and more) so running BanyanDB in production felt stable and predictable.</p>
<p>With <strong>APM 10.3.0</strong>, SkyWalking and BanyanDB moved forward together: BanyanDB 0.9.0’s new trace model was adopted end-to-end, reducing inefficient query round-trips and enabling new query views that significantly lowered page latency. The integration also expanded into lifecycle-aware operations with hot/warm/cold stage configuration (including TTL and query support), and added BanyanDB <strong>self-monitoring</strong> through OAP and the UI — the kind of end-to-end polish that turns a storage backend into a truly native solution.</p>
<p>If you’d like this review to cover <strong>APM 10.4.x</strong> as well, please point me to the corresponding release content in this repo (I didn’t find an APM 10.4.0 release announcement in the current checkout).</p>
<h2 id="packaging-and-deployment-ecosystem-helm">Packaging and deployment ecosystem (Helm)</h2>
<p>BanyanDB’s production readiness is not only server features — it also depends on deployment maturity.</p>
<ul>
<li>Helm charts:
<ul>
<li>SkyWalking Kubernetes Helm Chart 4.8.0 improved BanyanDB deployment defaults by updating the bundled BanyanDB Helm dependency, fixing an init-job volume-mount mismatch, and aligning OAP/UI images with the APM 10.3.0 line.</li>
<li>BanyanDB Helm 0.4.0 added backup/restore sidecars and a default volume for property storage.</li>
<li>BanyanDB Helm 0.5.0 introduced stage-aware patterns (hot/warm/cold), improved lifecycle-sidecar scheduling, moved liaison to StatefulSet, refined internal networking, and expanded configuration options.</li>
<li>BanyanDB Helm 0.5.1 refined liaison configuration and fixed restore-init environment issues.</li>
<li>BanyanDB Helm 0.5.3 fixed a liaison/data-node port issue.</li>
</ul>
</li>
</ul>
<h2 id="the-rest-of-the-community-agents-and-tooling-kept-moving">The rest of the community: agents and tooling kept moving</h2>
<p>While storage was the “main storyline”, the community shipped releases across agents, clients, and surrounding components throughout 2025.</p>
<p>Below is a consolidated view of the other releases, grouped by project, with the most important notes.</p>
<ul>
<li>
<p><strong>SkyWalking Java Agent</strong></p>
<ul>
<li><strong>9.4.0</strong>: agent self-observability; async-profiler support; broader plugin improvements.</li>
<li><strong>9.5.0</strong>: virtual thread executor plugin; compatibility and stability fixes; dependency upgrades.</li>
</ul>
</li>
<li>
<p><strong>SkyWalking Go</strong></p>
<ul>
<li><strong>0.6.0</strong>: richer manual APIs (events/logs/metrics, set span error); goframev2 plugin; bug fixes including Redis cluster mode.</li>
</ul>
</li>
<li>
<p><strong>SkyWalking for NodeJS</strong></p>
<ul>
<li><strong>0.8.0</strong>: Express 4/5 compatibility, keep-alive HTTP trace fix, and test/dependency maintenance.</li>
</ul>
</li>
<li>
<p><strong>SkyWalking Python</strong></p>
<ul>
<li><strong>1.2.0</strong>: sampling service, <code>sw_grpc</code> plugin, async/profiling stability fixes, Python 3.13 support, and dropping Python 3.7.</li>
</ul>
</li>
<li>
<p><strong>SkyWalking PHP</strong></p>
<ul>
<li><strong>1.0.0</strong>: reach 1.0; add PSR-3 log reporting; upgrade toolchain/dependencies.</li>
</ul>
</li>
<li>
<p><strong>SkyWalking Rust</strong></p>
<ul>
<li><strong>0.9.0</strong>: migrate to Rust edition 2024 and upgrade dependencies.</li>
<li><strong>0.10.0</strong>: Kafka client configuration refactor, <code>rdkafka</code> upgrade, CI maintenance.</li>
</ul>
</li>
<li>
<p><strong>SkyWalking Ruby</strong></p>
<ul>
<li><strong>0.1.0</strong>: initialize agent core and e2e tests; add plugins for Sinatra, redis-rb, net-http, memcached, and Elasticsearch.</li>
</ul>
</li>
<li>
<p><strong>SkyWalking Client JS</strong></p>
<ul>
<li><strong>1.0.0</strong>: add Core Web Vitals and static resource metrics; fix fetch/resource error handling; dependency and e2e/test improvements.</li>
</ul>
</li>
<li>
<p><strong>SkyWalking Satellite</strong></p>
<ul>
<li><strong>1.3.0</strong>: support native eBPF Access Log protocol and async-profiler protocol; upgrade Go toolchain.</li>
</ul>
</li>
<li>
<p><strong>SkyWalking Eyes</strong></p>
<ul>
<li><strong>0.7.0</strong>: improve installation/docs, respect gitignore behavior, upgrade Go, and simplify release steps.</li>
<li><strong>0.8.0</strong>: add Elixir support and stronger dependency-license scanning (notably Ruby via Gemfile.lock), plus stability fixes.</li>
</ul>
</li>
</ul>
<h2 id="looking-ahead-possible-directions-in-2026">Looking ahead: possible directions in 2026</h2>
<p>2025 was about making BanyanDB ready for production. In 2026, the community is exploring the next set of improvements that could make the whole stack simpler to operate, more stable under stress, and easier to integrate into broader observability ecosystems.</p>
<p>Possible areas include:</p>
<ul>
<li><strong>BanyanDB: remove the etcd dependency</strong>: the direction under discussion is to move away from etcd (given ecosystem activity and maintenance concerns) and rely more on DNS-based discovery plus BanyanDB’s native property capabilities.</li>
<li><strong>BanyanDB: stronger stability testing</strong>: more systematic testing, including chaos testing, to validate behavior under failures and noisy conditions.</li>
<li><strong>BanyanDB: better observability export</strong>: introducing First Occurrence Data Collection (FODC) as a sidecar and proxy server to provide a unified stream of observability data to third-party systems.</li>
<li><strong>SkyWalking APM: broader runtime and query capabilities</strong>: cold-stage data query support, a newer Java runtime (Java 25), and consideration of TraceQL protocol (Temper) support.</li>
</ul>
<h2 id="closing">Closing</h2>
<p>Thanks to everyone who contributed to SkyWalking in 2025. Every contribution is high-value — code, documentation, reviews, testing, issue triage, and operational experience — and each of them helped move the project forward.</p>
<p>We also want to say a special thank you to the countless end users across global companies. Many of the most valuable improvements don’t start from a pull request: they start from real-world use cases, performance investigations, production feedback, bug reports, and the patience to help us reproduce and validate fixes.</p>
<p>As another milestone, SkyWalking reached <strong>968 GitHub contributors</strong> globally, and we expect the <strong>1000th</strong> contributor milestone to arrive soon in 2026. But the community is much larger than the number suggests, and SkyWalking’s progress has always been driven by collaboration between contributors, adopters, and maintainers.</p>
<p>Apache SkyWalking was originally created by Sheng Wu as a personal project in May 2015. It would never have grown into what it is today without the whole community — and it will keep moving forward because of the community.</p>
