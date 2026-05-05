---
title: "Blog: SkyWalking GraalVM Distro: Design and Benchmarks"
url: "/blog/2026-03-13-skywalking-graalvm-distro-design-and-benchmarks/"
date: "Fri, 13 Mar 2026 00:00:00 +0000"
author: ""
feed_url: "https://skywalking.apache.org/feed"
---
<p><em>A technical deep-dive into how we migrated Apache SkyWalking OAP to GraalVM Native Image — not as a one-off port, but as a repeatable pipeline that stays aligned with upstream.</em></p>
<p><img alt="graph.jpg" src="./graph.jpg" /></p>
<p>For the broader story of how AI engineering made this project economically viable, see <a href="/blog/2026-03-13-how-ai-changed-the-economics-of-architecture/">How AI Changed the Economics of Architecture</a>.</p>
<h2 id="why-graalvm-is-not-optional">Why GraalVM Is Not Optional</h2>
<p>GraalVM Native Image compiles Java applications Ahead-of-Time (AOT) into standalone executables. For an observability backend like SkyWalking OAP, this is not a performance optimization — it is an operational necessity.</p>
<p>An observability platform must be the most reliable component in the infrastructure. It has to survive the failures it is supposed to observe. In cloud-native environments where workloads scale, migrate, and restart constantly, the backend that watches everything cannot itself be the slow, heavy process that takes seconds to recover and gigabytes to idle.</p>
<p>Our benchmarks make the case concrete:</p>
<ul>
<li><strong>Startup:</strong> ~5 ms vs ~635 ms. In a Kubernetes cluster where an OAP pod gets evicted or rescheduled, a 635 ms gap means lost telemetry — traces, metrics, and logs that arrive during that window are simply dropped. At 5 ms, the new pod is receiving data before most clients even notice the disruption.</li>
<li><strong>Idle memory:</strong> ~41 MiB vs ~1.2 GiB. Observability backends run 24/7. In a multi-tenant or edge deployment, a 97% reduction in baseline RSS is the difference between fitting the observability stack on a small node and needing a dedicated one.</li>
<li><strong>Memory under load:</strong> ~629 MiB vs ~2.0 GiB at 20 RPS. A 70% reduction at production-like traffic means fewer nodes, lower cloud bills, and more headroom before the backend itself becomes a scaling bottleneck.</li>
<li><strong>No warm-up penalty:</strong> Peak throughput is available from the first request. The JVM&rsquo;s JIT compiler needs minutes of traffic before it optimizes hot paths — during that window, tail latency is worse and data processing lags behind. A native binary has no such phase.</li>
<li><strong>Smaller attack surface:</strong> No JDK runtime means fewer CVEs to track and patch. For a component that ingests data from every service in the cluster, that matters.</li>
</ul>
<p>These are not incremental improvements. They change what deployment topologies are practical. Serverless observability backends, sidecar-model collectors, edge nodes with tight memory budgets — all become realistic when the backend is this light and this fast.</p>
<h2 id="the-challenge-a-mature-dynamic-java-platform">The Challenge: A Mature, Dynamic Java Platform</h2>
<p>SkyWalking OAP carries all the realities of a large Java platform: runtime bytecode generation, reflection-heavy initialization, classpath scanning, SPI-based module wiring, and dynamic DSL execution. These patterns are friendly to extensibility but hostile to GraalVM native image.</p>
<p>The documented GraalVM limitations are only the beginning. In a mature OSS platform, those limitations are deeply entangled with years of runtime design decisions. Standard GraalVM native images struggle with runtime class generation, reflection, dynamic discovery, and script execution — all of which had deep roots in SkyWalking OAP.</p>
<p>There was also a very concrete mountain in the early history of this distro. Upstream SkyWalking relied heavily on Groovy for LAL, MAL, and Hierarchy scripts. In theory, that was just one more unsupported runtime-heavy component. In practice, Groovy was the biggest obstacle in the whole path. It represented not only script execution, but a whole dynamic model that was deeply convenient on the JVM side and deeply unfriendly to native image.</p>
<h2 id="the-design-goal-make-migration-repeatable">The Design Goal: Make Migration Repeatable</h2>
<p>The final design is not just &ldquo;run native-image successfully.&rdquo; It is a system that keeps migration work repeatable:</p>
<ol>
<li><strong>Pre-compile runtime-generated assets at build time.</strong> OAL, MAL, LAL, Hierarchy rules, and meter-related generated classes are compiled during the build and packaged as artifacts instead of being generated at startup.</li>
<li><strong>Replace dynamic discovery with deterministic loading.</strong> Classpath scanning and runtime registration paths are converted into manifest-driven loading.</li>
<li><strong>Reduce runtime reflection and generate native metadata from the build.</strong> Reflection configuration is produced from actual manifests and scanned classes instead of being maintained as a hand-written guess list.</li>
<li><strong>Keep the upstream sync boundary explicit.</strong> Same-FQCN replacements are intentionally packaged, inventoried, and guarded with staleness checks.</li>
<li><strong>Make drift visible immediately.</strong> If upstream providers, rule files, or replaced source files change, tests fail and force explicit review.</li>
</ol>
<p>That is the architectural shift that matters most. Reusable abstraction and foresight did not become less important in the AI era. They became more important, because they determine whether AI speed produces a maintainable system or just a fast-growing pile of code.</p>
<h2 id="turning-runtime-dynamism-into-build-time-assets">Turning Runtime Dynamism into Build-Time Assets</h2>
<p>SkyWalking OAP has several dynamic subsystems that are natural in a JVM world but problematic for native image:</p>
<ul>
<li>OAL generates classes at runtime.</li>
<li>LAL, MAL, and Hierarchy were historically tied to Groovy-heavy runtime behavior, which became one of the biggest practical blockers in the early distro work.</li>
<li>MAL, LAL, and Hierarchy rules depend on runtime compilation behavior.</li>
<li>Guava-based classpath scanning discovers annotations, dispatchers, decorators, and meter functions.</li>
<li>SPI-based module/provider discovery expects a more dynamic runtime environment.</li>
<li>YAML/config initialization and framework integrations depend on reflective access.</li>
</ul>
<p>In SkyWalking GraalVM Distro, these are not solved one by one as isolated patches. They are pulled into a build-time pipeline.</p>
<p>The precompiler runs the DSL engines during the build, exports generated classes, writes manifests, serializes config data, and generates native-image metadata. That means startup becomes class loading and registration, not runtime code generation. The runtime path is simpler because the build path became richer.</p>
<p>This is also why the project is more than a performance exercise. The design goal was to move complexity into a place where it is easier to verify, easier to automate, and easier to repeat.</p>
<h2 id="same-fqcn-replacements-as-a-controlled-boundary">Same-FQCN Replacements as a Controlled Boundary</h2>
<p>One of the most practical design choices in this distro is the use of same-FQCN replacement classes. We do not rely on vague startup tricks or undocumented ordering assumptions. Instead, the GraalVM-specific jars are repackaged so the original upstream classes are excluded and the replacement classes occupy the exact same fully-qualified names.</p>
<p>This matters for maintainability. It creates a very clear boundary:</p>
<ul>
<li>the upstream class still defines the behavior contract,</li>
<li>the GraalVM replacement provides a compatible implementation strategy,</li>
<li>and the packaging makes that swap explicit.</li>
</ul>
<p>For example, OAL loading changes from runtime compilation into manifest-driven loading of precompiled classes. Similar replacements handle MAL and LAL DSL loading, module wiring, config initialization, and several reflection-sensitive paths. The goal is not to fork everything. The goal is to replace only the places where the runtime model is fundamentally unfriendly to native image.</p>
<p>That boundary is then guarded by tests that hash the upstream source files corresponding to the replacements. When upstream changes one of those files, the build fails and tells us exactly which replacement needs review. This is what turns &ldquo;keeping up with upstream&rdquo; from an anxiety problem into a visible engineering task.</p>
<h2 id="reflection-config-is-generated-not-guessed">Reflection Config Is Generated, Not Guessed</h2>
<p>In many GraalVM migrations, <code>reflect-config.json</code> becomes a manually accumulated artifact. It grows over time, gets stale, and nobody is fully sure whether it is complete or why each entry exists. That approach does not scale well for a large, evolving OSS platform.</p>
<p>In this distro, reflection metadata is generated from the build outputs and scanned classes:</p>
<ul>
<li>manifests for OAL, MAL, LAL, Hierarchy, and meter-generated classes,</li>
<li>annotation-scanned classes,</li>
<li>Armeria HTTP handlers,</li>
<li>GraphQL resolvers and schema-mapped types,</li>
<li>and accepted <code>ModuleConfig</code> classes.</li>
</ul>
<p>This is a much healthier model. Instead of asking people to remember every reflective access path, the system derives reflection metadata from the actual migration pipeline. The build becomes the source of truth.</p>
<h2 id="keeping-upstream-sync-practical">Keeping Upstream Sync Practical</h2>
<p>If this distro were only a one-time engineering sprint, it would be much less interesting. The real challenge is keeping it alive while upstream SkyWalking continues to evolve.</p>
<p>That is why the repo includes explicit inventories and drift detectors:</p>
<ul>
<li>provider inventories that force new upstream providers to be categorized,</li>
<li>rule-file inventories that force new DSL inputs to be acknowledged,</li>
<li>SHA watchers for precompiled YAML inputs,</li>
<li>and SHA watchers for upstream source files with GraalVM-specific replacements.</li>
</ul>
<p>Good abstraction is not only about elegant code structure. It is about choosing a migration design that can survive contact with future change.</p>
<h2 id="benchmark-results">Benchmark Results</h2>
<p>We benchmarked the standard JVM OAP against the GraalVM Distro on an Apple M3 Max (macOS, Docker Desktop, 10 CPUs / 62.7 GB), both connecting to BanyanDB.</p>
<h3 id="boot-test-docker-compose-no-traffic-median-of-3-runs">Boot Test (Docker Compose, no traffic, median of 3 runs)</h3>
<table>
  <thead>
      <tr>
          <th>Metric</th>
          <th>JVM OAP</th>
          <th>GraalVM OAP</th>
          <th>Delta</th>
      </tr>
  </thead>
  <tbody>
      <tr>
          <td>Cold boot startup</td>
          <td>635 ms</td>
          <td>5 ms</td>
          <td>~127x faster</td>
      </tr>
      <tr>
          <td>Warm boot startup</td>
          <td>630 ms</td>
          <td>5 ms</td>
          <td>~126x faster</td>
      </tr>
      <tr>
          <td>Idle RSS</td>
          <td>~1.2 GiB</td>
          <td>~41 MiB</td>
          <td>~97% reduction</td>
      </tr>
  </tbody>
</table>
<p>Boot time is measured from OAP&rsquo;s first application log timestamp to the <code>listening on 11800</code> log line (gRPC server ready).</p>
<h3 id="under-sustained-load-kind--istio-1252--bookinfo-at-20-rps-2-oap-replicas">Under Sustained Load (Kind + Istio 1.25.2 + Bookinfo at ~20 RPS, 2 OAP replicas)</h3>
<p>30 samples at 10s intervals after 60s warmup.</p>
<table>
  <thead>
      <tr>
          <th>Metric</th>
          <th>JVM OAP</th>
          <th>GraalVM OAP</th>
          <th>Delta</th>
      </tr>
  </thead>
  <tbody>
      <tr>
          <td>CPU median (millicores)</td>
          <td>101</td>
          <td>68</td>
          <td>-33%</td>
      </tr>
      <tr>
          <td>CPU avg (millicores)</td>
          <td>107</td>
          <td>67</td>
          <td>-37%</td>
      </tr>
      <tr>
          <td>Memory median (MiB)</td>
          <td>2068</td>
          <td>629</td>
          <td><strong>-70%</strong></td>
      </tr>
      <tr>
          <td>Memory avg (MiB)</td>
          <td>2082</td>
          <td>624</td>
          <td><strong>-70%</strong></td>
      </tr>
  </tbody>
</table>
<p>Both variants reported identical entry-service CPM, confirming equivalent traffic processing capability.</p>
<p>Service metrics collected every 30s via swctl for all discovered services:
<code>service_cpm</code>, <code>service_resp_time</code>, <code>service_sla</code>, <code>service_apdex</code>, <code>service_percentile</code>.</p>
<p>Full benchmark scripts and raw data are in the <a href="https://github.com/apache/skywalking-graalvm-distro/tree/main/benchmark">benchmark/</a> directory of the distro repository.</p>
<h2 id="current-status">Current Status</h2>
<p>The project is a runnable experimental distribution, hosted in its own repository: <a href="https://github.com/apache/skywalking-graalvm-distro">apache/skywalking-graalvm-distro</a>.</p>
<p>The current distro intentionally focuses on a modern, high-performance operating model:</p>
<ul>
<li><strong>Storage:</strong> BanyanDB</li>
<li><strong>Cluster modes:</strong> Standalone and Kubernetes</li>
<li><strong>Configuration:</strong> none or Kubernetes ConfigMap</li>
<li><strong>Runtime model:</strong> fixed module set, precompiled assets, and AOT-friendly wiring</li>
</ul>
<p>This focus is deliberate. A repeatable migration system starts by making a clear scope runnable, then expanding without losing control.</p>
<h2 id="getting-started">Getting Started</h2>
<p>Because the SkyWalking GraalVM Distro is designed for peak performance, it is optimized to work with <strong>BanyanDB</strong> as its storage backend. The current published image is available on Docker Hub, and you can boot the stack using the following <code>docker-compose.yml</code>.</p>
<div class="highlight"><pre tabindex="0"><code class="language-yaml"><span style="display: flex;"><span><span style="color: #0550ae;">version</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #0a3069;">'3.8'</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #0550ae;">services</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">banyandb</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">image</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>ghcr.io/apache/skywalking-banyandb:e1ba421bd624727760c7a69c84c6fe55878fb526<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">container_name</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>banyandb<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">restart</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>always<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">ports</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- <span style="color: #0a3069;">"17912:17912"</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- <span style="color: #0a3069;">"17913:17913"</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">command</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>standalone --stream-root-path /tmp/stream-data --measure-root-path /tmp/measure-data --measure-metadata-cache-wait-duration 1m --stream-metadata-cache-wait-duration 1m<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">healthcheck</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">test</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #1f2328;">[</span><span style="color: #0a3069;">"CMD"</span><span style="color: #1f2328;">,</span><span style="color: #fff;"> </span><span style="color: #0a3069;">"sh"</span><span style="color: #1f2328;">,</span><span style="color: #fff;"> </span><span style="color: #0a3069;">"-c"</span><span style="color: #1f2328;">,</span><span style="color: #fff;"> </span><span style="color: #0a3069;">"nc -nz 127.0.0.1 17912"</span><span style="color: #1f2328;">]</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">interval</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>5s<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">timeout</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>10s<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">retries</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #0550ae;">120</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">oap</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">image</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>apache/skywalking-graalvm-distro:0.1.1<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">container_name</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>oap<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">depends_on</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">banyandb</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">        </span><span style="color: #0550ae;">condition</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>service_healthy<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">restart</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>always<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">ports</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- <span style="color: #0a3069;">"11800:11800"</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- <span style="color: #0a3069;">"12800:12800"</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">environment</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">SW_STORAGE</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>banyandb<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">SW_STORAGE_BANYANDB_TARGETS</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>banyandb:17912<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">SW_HEALTH_CHECKER</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>default<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">healthcheck</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">test</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #1f2328;">[</span><span style="color: #0a3069;">"CMD-SHELL"</span><span style="color: #1f2328;">,</span><span style="color: #fff;"> </span><span style="color: #0a3069;">"nc -nz 127.0.0.1 11800 || exit 1"</span><span style="color: #1f2328;">]</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">interval</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>5s<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">timeout</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>10s<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">retries</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #0550ae;">120</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">ui</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">image</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>ghcr.io/apache/skywalking/ui:10.3.0<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">container_name</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>ui<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">depends_on</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">oap</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">        </span><span style="color: #0550ae;">condition</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>service_healthy<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">restart</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>always<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">ports</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- <span style="color: #0a3069;">"8080:8080"</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">environment</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">SW_OAP_ADDRESS</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>http://oap:12800<span style="color: #fff;">
</span></span></span></code></pre></div><p>Simply run:</p>
<div class="highlight"><pre tabindex="0"><code class="language-bash"><span style="display: flex;"><span>docker compose up -d
</span></span></code></pre></div><p>We invite the community to test this new distribution, report issues, and help us move it toward a production-ready state.</p>
<p><em>Special thanks to the GraalVM team for the technology foundation.</em></p>
