---
title: "Blog: Monitoring Envoy AI Gateway with Apache SkyWalking"
url: "/blog/2026-04-02-envoy-ai-gateway-monitoring/"
date: "Thu, 02 Apr 2026 00:00:00 +0000"
author: ""
feed_url: "https://skywalking.apache.org/feed"
---
<h2 id="the-problem-flying-blind-with-llm-traffic">The Problem: Flying Blind with LLM Traffic</h2>
<p>LLM traffic is becoming a first-class citizen in production infrastructure. Teams are calling OpenAI, Anthropic,
AWS Bedrock, Azure OpenAI, Google Gemini — often multiple providers at once. But most organizations have
no unified visibility into this traffic:</p>
<ul>
<li><strong>Token costs spiral</strong> without knowing which teams, models, or providers drive the spend.
A single misconfigured prompt template can burn through thousands of dollars before anyone notices.</li>
<li><strong>Provider outages cause cascading failures.</strong> When OpenAI has a bad hour, your application goes down
with it — and you have no failover visibility to understand what happened or switch providers automatically.</li>
<li><strong>No unified metrics</strong> across heterogeneous LLM calls. Latency, Time to First Token (TTFT),
Time Per Output Token (TPOT), token usage, error rates — each provider reports these differently,
if at all. There is no single dashboard to compare them.</li>
</ul>
<p>This is the same observability gap that microservices faced a decade ago. The solution then was
service meshes and API gateways with built-in telemetry. For AI workloads, the answer is an AI gateway.</p>
<h2 id="why-an-ai-gateway">Why an AI Gateway</h2>
<p><a href="https://aigateway.envoyproxy.io/">Envoy AI Gateway</a> is an open-source AI gateway built on top of
<a href="https://www.envoyproxy.io/">Envoy Proxy</a> and <a href="https://gateway.envoyproxy.io/">Envoy Gateway</a>.
It is not a standalone SaaS product or a Python proxy — it is infrastructure-grade software built on
the same Envoy that already handles traffic for a large portion of cloud-native deployments.</p>
<p>Key capabilities:</p>
<ul>
<li><strong>Multi-provider routing</strong> — supports 16+ AI providers (OpenAI, Anthropic, AWS Bedrock, Azure OpenAI,
Google Gemini, Mistral, Cohere, DeepSeek, and more) behind a unified API.</li>
<li><strong>Token-based rate limiting</strong> — rate limit by token consumption, not just request count.</li>
<li><strong>Provider fallback</strong> — automatic failover when a provider is down or slow.</li>
<li><strong>Model virtualization</strong> — abstract model names so applications are decoupled from specific providers.</li>
<li><strong>Two-tier architecture</strong> — a reference architecture with a centralized entry gateway (Tier 1) for
auth and global routing, and per-cluster gateways (Tier 2) for inference optimization.</li>
<li><strong>CNCF ecosystem native</strong> — runs on Kubernetes, composes with existing Envoy filters, WASM plugins,
and standard Kubernetes Gateway API resources.</li>
</ul>
<p>Because Envoy AI Gateway natively emits GenAI metrics and access logs via OTLP following
<a href="https://opentelemetry.io/docs/specs/semconv/gen-ai/">OpenTelemetry GenAI Semantic Conventions</a>,
it plugs directly into any OpenTelemetry-compatible backend.</p>
<p>Starting from SkyWalking 10.4.0, the OAP server natively receives and analyzes Envoy AI Gateway&rsquo;s
OTLP metrics and access logs — no OpenTelemetry Collector needed in between.</p>
<h2 id="data-flow">Data Flow</h2>
<p>The AI Gateway pushes telemetry directly to SkyWalking via OTLP gRPC:</p>
<p><img alt="Data flow" src="workflow.jpg" /></p>
<ol>
<li><strong>Application</strong> sends LLM API requests through the Envoy AI Gateway.</li>
<li><strong>Envoy AI Gateway</strong> routes requests to AI providers (or local models like Ollama)
and records GenAI metrics (token usage, latency, TTFT, TPOT) and access logs.</li>
<li>The gateway pushes metrics and logs via <strong>OTLP gRPC</strong> directly to <strong>SkyWalking OAP</strong> on port 11800.</li>
<li>SkyWalking OAP parses metrics with MAL rules and access logs with LAL rules,
then stores everything in <strong>BanyanDB</strong>.</li>
</ol>
<p>No OpenTelemetry Collector is needed. SkyWalking OAP&rsquo;s built-in OTLP receiver handles everything.</p>
<h2 id="try-it-locally">Try It Locally</h2>
<p>This demo uses <a href="https://ollama.com/">Ollama</a> as a local LLM backend so you can try
everything without an API key. The <a href="https://github.com/envoyproxy/ai-gateway/tree/main/cmd/aigw">Envoy AI Gateway CLI</a>
(<code>aigw</code>) provides a standalone mode that runs outside Kubernetes — perfect for local testing.</p>
<h3 id="prerequisites">Prerequisites</h3>
<ul>
<li>Docker and Docker Compose</li>
<li><a href="https://ollama.com/">Ollama</a> installed on your host</li>
</ul>
<h3 id="step-1-start-ollama">Step 1: Start Ollama</h3>
<p>Start Ollama on all interfaces so Docker containers can reach it:</p>
<div class="highlight"><pre tabindex="0"><code class="language-bash"><span style="display: flex;"><span><span style="color: #953800;">OLLAMA_HOST</span><span style="color: #0550ae;">=</span>0.0.0.0 ollama serve
</span></span></code></pre></div><p>Pull a small model for testing:</p>
<div class="highlight"><pre tabindex="0"><code class="language-bash"><span style="display: flex;"><span>ollama pull llama3.2:1b
</span></span></code></pre></div><h3 id="step-2-start-the-stack">Step 2: Start the Stack</h3>
<p>Create a <code>docker-compose.yaml</code>:</p>
<div class="highlight"><pre tabindex="0"><code class="language-yaml"><span style="display: flex;"><span><span style="color: #0550ae;">services</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">banyandb</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">image</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>apache/skywalking-banyandb:0.10.0<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">container_name</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>banyandb<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">ports</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- <span style="color: #0a3069;">"17912:17912"</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">command</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>standalone --stream-root-path /tmp/stream-data --measure-root-path /tmp/measure-data<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">healthcheck</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">test</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #1f2328;">[</span><span style="color: #0a3069;">"CMD-SHELL"</span><span style="color: #1f2328;">,</span><span style="color: #fff;"> </span><span style="color: #0a3069;">"wget -qO- http://localhost:17913/api/healthz || exit 1"</span><span style="color: #1f2328;">]</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">interval</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>5s<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">timeout</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>3s<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">retries</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #0550ae;">10</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">oap</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">image</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>apache/skywalking-oap-server:10.4.0<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">container_name</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>oap<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">depends_on</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">banyandb</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">        </span><span style="color: #0550ae;">condition</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>service_healthy<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">ports</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- <span style="color: #0a3069;">"11800:11800"</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- <span style="color: #0a3069;">"12800:12800"</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">environment</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">SW_STORAGE</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>banyandb<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">SW_STORAGE_BANYANDB_TARGETS</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>banyandb:17912<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">healthcheck</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">test</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #1f2328;">[</span><span style="color: #0a3069;">"CMD-SHELL"</span><span style="color: #1f2328;">,</span><span style="color: #fff;"> </span><span style="color: #0a3069;">"bash -c 'echo &gt; /dev/tcp/localhost/12800' || exit 1"</span><span style="color: #1f2328;">]</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">interval</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>10s<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">timeout</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>5s<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">retries</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #0550ae;">30</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">start_period</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>60s<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">ui</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">image</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>apache/skywalking-ui:10.4.0<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">container_name</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>ui<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">depends_on</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">oap</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">        </span><span style="color: #0550ae;">condition</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>service_healthy<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">ports</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- <span style="color: #0a3069;">"8080:8080"</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">environment</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">SW_OAP_ADDRESS</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>http://oap:12800<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">aigw</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">image</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>envoyproxy/ai-gateway-cli:latest<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">container_name</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>aigw<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">depends_on</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">oap</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">        </span><span style="color: #0550ae;">condition</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>service_healthy<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">environment</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- OPENAI_BASE_URL=http://host.docker.internal:11434/v1<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- OPENAI_API_KEY=unused<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- OTEL_SERVICE_NAME=my-ai-gateway<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- OTEL_EXPORTER_OTLP_ENDPOINT=http://oap:11800<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- OTEL_EXPORTER_OTLP_PROTOCOL=grpc<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- OTEL_METRICS_EXPORTER=otlp<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- OTEL_LOGS_EXPORTER=otlp<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- OTEL_METRIC_EXPORT_INTERVAL=5000<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- OTEL_RESOURCE_ATTRIBUTES=job_name=envoy-ai-gateway,service.instance.id=aigw-1,service.layer=ENVOY_AI_GATEWAY<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">ports</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- <span style="color: #0a3069;">"1975:1975"</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">extra_hosts</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span>- <span style="color: #0a3069;">"host.docker.internal:host-gateway"</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">command</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #1f2328;">[</span><span style="color: #0a3069;">"run"</span><span style="color: #1f2328;">]</span><span style="color: #fff;">
</span></span></span></code></pre></div><p>Start everything:</p>
<div class="highlight"><pre tabindex="0"><code class="language-bash"><span style="display: flex;"><span>docker compose up -d
</span></span></code></pre></div><p>Wait for all services to become healthy (BanyanDB starts first, then OAP, then UI and AI Gateway):</p>
<div class="highlight"><pre tabindex="0"><code class="language-bash"><span style="display: flex;"><span>docker compose ps
</span></span></code></pre></div><p>The key OTLP configuration on the <code>aigw</code> service:</p>
<table>
  <thead>
      <tr>
          <th>Env Var</th>
          <th>Value</th>
          <th>Purpose</th>
      </tr>
  </thead>
  <tbody>
      <tr>
          <td><code>OTEL_SERVICE_NAME</code></td>
          <td><code>my-ai-gateway</code></td>
          <td>Service name in SkyWalking</td>
      </tr>
      <tr>
          <td><code>OTEL_EXPORTER_OTLP_ENDPOINT</code></td>
          <td><code>http://oap:11800</code></td>
          <td>SkyWalking OAP gRPC endpoint</td>
      </tr>
      <tr>
          <td><code>OTEL_EXPORTER_OTLP_PROTOCOL</code></td>
          <td><code>grpc</code></td>
          <td>OTLP transport</td>
      </tr>
      <tr>
          <td><code>OTEL_METRICS_EXPORTER</code></td>
          <td><code>otlp</code></td>
          <td>Enable metrics push</td>
      </tr>
      <tr>
          <td><code>OTEL_LOGS_EXPORTER</code></td>
          <td><code>otlp</code></td>
          <td>Enable access log push</td>
      </tr>
  </tbody>
</table>
<p>The <code>OTEL_RESOURCE_ATTRIBUTES</code> must include:</p>
<ul>
<li><code>job_name=envoy-ai-gateway</code> — routing tag for MAL/LAL rules</li>
<li><code>service.instance.id=&lt;id&gt;</code> — instance identity</li>
<li><code>service.layer=ENVOY_AI_GATEWAY</code> — routes logs to AI Gateway LAL rules</li>
</ul>
<p>The MAL and LAL rules are enabled by default in SkyWalking OAP. No OAP-side configuration is needed.</p>
<h3 id="step-3-run-the-demo-app">Step 3: Run the Demo App</h3>
<p>Create a simple Python application that sends requests through the AI Gateway (<code>app.py</code>).
It mixes normal requests, streaming requests (for TTFT/TPOT metrics), and error requests
(non-existent model → HTTP 404, always captured by the LAL sampling policy):</p>
<div class="highlight"><pre tabindex="0"><code class="language-python"><span style="display: flex;"><span><span style="color: #cf222e;">import</span> <span style="color: #24292e;">time</span><span style="color: #0550ae;">,</span> <span style="color: #24292e;">random</span><span style="color: #0550ae;">,</span> <span style="color: #24292e;">requests</span>
</span></span><span style="display: flex;"><span>
</span></span><span style="display: flex;"><span>GATEWAY <span style="color: #0550ae;">=</span> <span style="color: #0a3069;">"http://localhost:1975"</span>
</span></span><span style="display: flex;"><span>HEADERS <span style="color: #0550ae;">=</span> <span style="color: #1f2328;">{</span><span style="color: #0a3069;">"Authorization"</span><span style="color: #1f2328;">:</span> <span style="color: #0a3069;">"Bearer unused"</span><span style="color: #1f2328;">,</span> <span style="color: #0a3069;">"Content-Type"</span><span style="color: #1f2328;">:</span> <span style="color: #0a3069;">"application/json"</span><span style="color: #1f2328;">}</span>
</span></span><span style="display: flex;"><span>
</span></span><span style="display: flex;"><span>questions <span style="color: #0550ae;">=</span> <span style="color: #1f2328;">[</span>
</span></span><span style="display: flex;"><span>    <span style="color: #0a3069;">"What is Apache SkyWalking? Answer in one sentence."</span><span style="color: #1f2328;">,</span>
</span></span><span style="display: flex;"><span>    <span style="color: #0a3069;">"What is Envoy Proxy used for? Answer in one sentence."</span><span style="color: #1f2328;">,</span>
</span></span><span style="display: flex;"><span>    <span style="color: #0a3069;">"What are the benefits of an AI gateway? Answer in two sentences."</span><span style="color: #1f2328;">,</span>
</span></span><span style="display: flex;"><span>    <span style="color: #0a3069;">"Explain observability in three sentences."</span><span style="color: #1f2328;">,</span>
</span></span><span style="display: flex;"><span><span style="color: #1f2328;">]</span>
</span></span><span style="display: flex;"><span>
</span></span><span style="display: flex;"><span><span style="color: #cf222e;">def</span> <span style="color: #6639ba;">chat</span><span style="color: #1f2328;">(</span>model<span style="color: #1f2328;">,</span> question<span style="color: #1f2328;">,</span> stream<span style="color: #0550ae;">=</span><span style="color: #cf222e;">False</span><span style="color: #1f2328;">):</span>
</span></span><span style="display: flex;"><span>    resp <span style="color: #0550ae;">=</span> requests<span style="color: #0550ae;">.</span>post<span style="color: #1f2328;">(</span>
</span></span><span style="display: flex;"><span>        <span style="color: #0a3069;">f</span><span style="color: #0a3069;">"</span><span style="color: #0a3069;">{</span>GATEWAY<span style="color: #0a3069;">}</span><span style="color: #0a3069;">/v1/chat/completions"</span><span style="color: #1f2328;">,</span>
</span></span><span style="display: flex;"><span>        json<span style="color: #0550ae;">=</span><span style="color: #1f2328;">{</span><span style="color: #0a3069;">"model"</span><span style="color: #1f2328;">:</span> model<span style="color: #1f2328;">,</span> <span style="color: #0a3069;">"messages"</span><span style="color: #1f2328;">:</span> <span style="color: #1f2328;">[{</span><span style="color: #0a3069;">"role"</span><span style="color: #1f2328;">:</span> <span style="color: #0a3069;">"user"</span><span style="color: #1f2328;">,</span> <span style="color: #0a3069;">"content"</span><span style="color: #1f2328;">:</span> question<span style="color: #1f2328;">}],</span> <span style="color: #0a3069;">"stream"</span><span style="color: #1f2328;">:</span> stream<span style="color: #1f2328;">},</span>
</span></span><span style="display: flex;"><span>        headers<span style="color: #0550ae;">=</span>HEADERS<span style="color: #1f2328;">,</span> timeout<span style="color: #0550ae;">=</span><span style="color: #0550ae;">60</span><span style="color: #1f2328;">,</span> stream<span style="color: #0550ae;">=</span>stream<span style="color: #1f2328;">,</span>
</span></span><span style="display: flex;"><span>    <span style="color: #1f2328;">)</span>
</span></span><span style="display: flex;"><span>    <span style="color: #cf222e;">if</span> stream<span style="color: #1f2328;">:</span>
</span></span><span style="display: flex;"><span>        chunks <span style="color: #0550ae;">=</span> <span style="color: #1f2328;">[]</span>
</span></span><span style="display: flex;"><span>        <span style="color: #cf222e;">for</span> line <span style="color: #0550ae;">in</span> resp<span style="color: #0550ae;">.</span>iter_lines<span style="color: #1f2328;">():</span>
</span></span><span style="display: flex;"><span>            <span style="color: #cf222e;">if</span> line<span style="color: #1f2328;">:</span>
</span></span><span style="display: flex;"><span>                chunks<span style="color: #0550ae;">.</span>append<span style="color: #1f2328;">(</span>line<span style="color: #0550ae;">.</span>decode<span style="color: #1f2328;">())</span>
</span></span><span style="display: flex;"><span>        <span style="color: #cf222e;">return</span> resp<span style="color: #0550ae;">.</span>status_code<span style="color: #1f2328;">,</span> <span style="color: #0a3069;">f</span><span style="color: #0a3069;">"[streamed </span><span style="color: #0a3069;">{</span><span style="color: #6639ba;">len</span><span style="color: #1f2328;">(</span>chunks<span style="color: #1f2328;">)</span><span style="color: #0a3069;">}</span><span style="color: #0a3069;"> chunks]"</span>
</span></span><span style="display: flex;"><span>    <span style="color: #cf222e;">return</span> resp<span style="color: #0550ae;">.</span>status_code<span style="color: #1f2328;">,</span> resp<span style="color: #0550ae;">.</span>json<span style="color: #1f2328;">()</span>
</span></span><span style="display: flex;"><span>
</span></span><span style="display: flex;"><span><span style="color: #cf222e;">while</span> <span style="color: #cf222e;">True</span><span style="color: #1f2328;">:</span>
</span></span><span style="display: flex;"><span>    r <span style="color: #0550ae;">=</span> random<span style="color: #0550ae;">.</span>random<span style="color: #1f2328;">()</span>
</span></span><span style="display: flex;"><span>    <span style="color: #cf222e;">if</span> r <span style="color: #0550ae;">&lt;</span> <span style="color: #0550ae;">0.2</span><span style="color: #1f2328;">:</span>
</span></span><span style="display: flex;"><span>        <span style="color: #57606a;"># Error request: non-existent model triggers 404</span>
</span></span><span style="display: flex;"><span>        status<span style="color: #1f2328;">,</span> body <span style="color: #0550ae;">=</span> chat<span style="color: #1f2328;">(</span><span style="color: #0a3069;">"non-existent-model"</span><span style="color: #1f2328;">,</span> <span style="color: #0a3069;">"hello"</span><span style="color: #1f2328;">)</span>
</span></span><span style="display: flex;"><span>        <span style="color: #6639ba;">print</span><span style="color: #1f2328;">(</span><span style="color: #0a3069;">f</span><span style="color: #0a3069;">"[error] model=non-existent-model status=</span><span style="color: #0a3069;">{</span>status<span style="color: #0a3069;">}</span><span style="color: #0a3069;">"</span><span style="color: #1f2328;">)</span>
</span></span><span style="display: flex;"><span>    <span style="color: #cf222e;">elif</span> r <span style="color: #0550ae;">&lt;</span> <span style="color: #0550ae;">0.5</span><span style="color: #1f2328;">:</span>
</span></span><span style="display: flex;"><span>        <span style="color: #57606a;"># Streaming request — generates TTFT and TPOT metrics</span>
</span></span><span style="display: flex;"><span>        q <span style="color: #0550ae;">=</span> random<span style="color: #0550ae;">.</span>choice<span style="color: #1f2328;">(</span>questions<span style="color: #1f2328;">)</span>
</span></span><span style="display: flex;"><span>        status<span style="color: #1f2328;">,</span> info <span style="color: #0550ae;">=</span> chat<span style="color: #1f2328;">(</span><span style="color: #0a3069;">"llama3.2:1b"</span><span style="color: #1f2328;">,</span> q<span style="color: #1f2328;">,</span> stream<span style="color: #0550ae;">=</span><span style="color: #cf222e;">True</span><span style="color: #1f2328;">)</span>
</span></span><span style="display: flex;"><span>        <span style="color: #6639ba;">print</span><span style="color: #1f2328;">(</span><span style="color: #0a3069;">f</span><span style="color: #0a3069;">"[stream] status=</span><span style="color: #0a3069;">{</span>status<span style="color: #0a3069;">}</span><span style="color: #0a3069;"> </span><span style="color: #0a3069;">{</span>info<span style="color: #0a3069;">}</span><span style="color: #0a3069;">"</span><span style="color: #1f2328;">)</span>
</span></span><span style="display: flex;"><span>    <span style="color: #cf222e;">else</span><span style="color: #1f2328;">:</span>
</span></span><span style="display: flex;"><span>        <span style="color: #57606a;"># Normal non-streaming request</span>
</span></span><span style="display: flex;"><span>        q <span style="color: #0550ae;">=</span> random<span style="color: #0550ae;">.</span>choice<span style="color: #1f2328;">(</span>questions<span style="color: #1f2328;">)</span>
</span></span><span style="display: flex;"><span>        status<span style="color: #1f2328;">,</span> body <span style="color: #0550ae;">=</span> chat<span style="color: #1f2328;">(</span><span style="color: #0a3069;">"llama3.2:1b"</span><span style="color: #1f2328;">,</span> q<span style="color: #1f2328;">)</span>
</span></span><span style="display: flex;"><span>        answer <span style="color: #0550ae;">=</span> body<span style="color: #0550ae;">.</span>get<span style="color: #1f2328;">(</span><span style="color: #0a3069;">"choices"</span><span style="color: #1f2328;">,</span> <span style="color: #1f2328;">[{}])[</span><span style="color: #0550ae;">0</span><span style="color: #1f2328;">]</span><span style="color: #0550ae;">.</span>get<span style="color: #1f2328;">(</span><span style="color: #0a3069;">"message"</span><span style="color: #1f2328;">,</span> <span style="color: #1f2328;">{})</span><span style="color: #0550ae;">.</span>get<span style="color: #1f2328;">(</span><span style="color: #0a3069;">"content"</span><span style="color: #1f2328;">,</span> <span style="color: #0a3069;">""</span><span style="color: #1f2328;">)[:</span><span style="color: #0550ae;">80</span><span style="color: #1f2328;">]</span>
</span></span><span style="display: flex;"><span>        tokens <span style="color: #0550ae;">=</span> body<span style="color: #0550ae;">.</span>get<span style="color: #1f2328;">(</span><span style="color: #0a3069;">"usage"</span><span style="color: #1f2328;">,</span> <span style="color: #1f2328;">{})</span>
</span></span><span style="display: flex;"><span>        <span style="color: #6639ba;">print</span><span style="color: #1f2328;">(</span><span style="color: #0a3069;">f</span><span style="color: #0a3069;">"[ok] status=</span><span style="color: #0a3069;">{</span>status<span style="color: #0a3069;">}</span><span style="color: #0a3069;"> tokens=</span><span style="color: #0a3069;">{</span>tokens<span style="color: #0a3069;">}</span><span style="color: #0a3069;"> answer=</span><span style="color: #0a3069;">{</span>answer<span style="color: #0a3069;">}</span><span style="color: #0a3069;">..."</span><span style="color: #1f2328;">)</span>
</span></span><span style="display: flex;"><span>    time<span style="color: #0550ae;">.</span>sleep<span style="color: #1f2328;">(</span>random<span style="color: #0550ae;">.</span>randint<span style="color: #1f2328;">(</span><span style="color: #0550ae;">20</span><span style="color: #1f2328;">,</span> <span style="color: #0550ae;">30</span><span style="color: #1f2328;">))</span>
</span></span></code></pre></div><p>Run it:</p>
<div class="highlight"><pre tabindex="0"><code class="language-bash"><span style="display: flex;"><span>pip install requests
</span></span><span style="display: flex;"><span>python app.py
</span></span></code></pre></div><p>The application talks to the AI Gateway on port 1975, which routes to Ollama.
Each request generates GenAI metrics (token usage, latency, TTFT, TPOT) and access logs
that the gateway pushes to SkyWalking via OTLP.</p>
<p>The error requests (non-existent model → HTTP 404) are always captured by the access log
sampling policy, so you will see them in the SkyWalking log view.</p>
<h3 id="step-4-view-in-skywalking-ui">Step 4: View in SkyWalking UI</h3>
<p>Open <a href="http://localhost:8080">http://localhost:8080</a> and select the <strong>GenAI &gt; Envoy AI Gateway</strong> menu.</p>
<p>The service list shows <code>my-ai-gateway</code> with CPM, latency, and token rates at a glance:</p>
<p><img alt="Service list" src="screen-1.png" /></p>
<p>Click into the service to see the full dashboard — Request CPM, Latency (average + percentiles),
Input/Output Token Rates, TTFT, and TPOT:</p>
<p><img alt="Service dashboard" src="screen-2.png" /></p>
<p>The <strong>Providers</strong> tab breaks down metrics by AI provider:</p>
<p><img alt="Provider breakdown" src="screen-3.png" /></p>
<p>The <strong>Models</strong> tab shows per-model metrics including TTFT and TPOT (streaming only).
Note the <code>unknown</code> model entries — these are the error requests with non-existent models:</p>
<p><img alt="Model breakdown" src="screen-4.png" /></p>
<p>The <strong>Log</strong> tab shows access logs. The sampling policy drops normal successful responses
but always captures errors (HTTP 404) and high-token requests:</p>
<p><img alt="Access logs" src="screen-5.png" /></p>
<h3 id="cleanup">Cleanup</h3>
<div class="highlight"><pre tabindex="0"><code class="language-bash"><span style="display: flex;"><span>docker compose down
</span></span></code></pre></div><h2 id="deploying-on-kubernetes">Deploying on Kubernetes</h2>
<p>For production deployments, Envoy AI Gateway runs as a full Kubernetes controller with
Envoy Gateway as the control plane. See the
<a href="https://aigateway.envoyproxy.io/docs/getting-started/">Envoy AI Gateway getting started guide</a>
for Kubernetes installation.</p>
<p>The OTLP configuration is the same — set the <code>OTEL_*</code> environment variables on the
AI Gateway&rsquo;s external processor to point at SkyWalking OAP&rsquo;s gRPC port (11800).
See the <a href="https://skywalking.apache.org/docs/main/next/en/setup/backend/backend-envoy-ai-gateway-monitoring/">SkyWalking Envoy AI Gateway Monitoring</a>
documentation for details.</p>
<h2 id="genai-observability-without-an-ai-gateway">GenAI Observability Without an AI Gateway</h2>
<p>Not every deployment uses an AI gateway. If your applications call LLM providers directly,
SkyWalking 10.4.0 also provides GenAI observability through the
<a href="https://skywalking.apache.org/docs/main/next/en/setup/service-agent/virtual-genai/">Virtual GenAI</a> layer.</p>
<p>This works with any SkyWalking-instrumented, OpenTelemetry-instrumented, or Zipkin-instrumented application.
When traces carry <code>gen_ai.*</code> tags (following
<a href="https://opentelemetry.io/docs/specs/semconv/gen-ai/">OpenTelemetry GenAI Semantic Conventions</a>),
SkyWalking derives per-provider and per-model metrics from the client side:
latency, token usage, success rate, and estimated cost.</p>
<p>For Java applications, the SkyWalking Java Agent (9.7+) includes a Spring AI plugin that automatically
instruments calls to 13+ providers (OpenAI, Anthropic, AWS Bedrock, Google GenAI, DeepSeek, Mistral, etc.)
with the correct <code>gen_ai.*</code> span tags — no code changes needed.</p>
<p>This is a different use case from the Envoy AI Gateway monitoring covered above:</p>
<ul>
<li><strong>Envoy AI Gateway layer</strong>: infrastructure-level observability — what the gateway sees across all traffic.
Best for platform teams managing centralized AI routing.</li>
<li><strong>Virtual GenAI layer</strong>: application-level observability — what each instrumented app sees for its own LLM calls.
Best for teams without a centralized gateway, or for per-application cost tracking.</li>
</ul>
<h2 id="references">References</h2>
<ul>
<li><a href="https://aigateway.envoyproxy.io/">Envoy AI Gateway</a> — project site and documentation</li>
<li><a href="https://github.com/envoyproxy/ai-gateway/tree/main/cmd/aigw">Envoy AI Gateway CLI</a> — standalone mode for local development</li>
<li><a href="https://skywalking.apache.org/docs/main/next/en/setup/backend/backend-envoy-ai-gateway-monitoring/">SkyWalking Envoy AI Gateway Monitoring</a> — OAP setup doc</li>
<li><a href="https://skywalking.apache.org/docs/main/next/en/setup/service-agent/virtual-genai/">SkyWalking Virtual GenAI</a> — client-side GenAI observability</li>
<li><a href="https://opentelemetry.io/docs/specs/semconv/gen-ai/">OpenTelemetry GenAI Semantic Conventions</a> — the metric/attribute standard both projects follow</li>
</ul>
