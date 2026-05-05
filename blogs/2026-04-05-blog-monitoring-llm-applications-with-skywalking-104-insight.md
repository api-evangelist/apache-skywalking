---
title: "Blog: Monitoring LLM Applications with SkyWalking 10.4: Insights into Performance and Cost"
url: "/blog/2026-04-05-virtual-genai-monitoring/"
date: "Sun, 05 Apr 2026 00:00:00 +0000"
author: ""
feed_url: "https://skywalking.apache.org/feed"
---
<h1 id="the-problem-as-applications-consume-llms-monitoring-leaves-a-blind-spot">The Problem: As Applications &ldquo;Consume&rdquo; LLMs, Monitoring Leaves a Blind Spot</h1>
<p>With the deep penetration of Generative AI (GenAI) into enterprise workflows, developers face a challenging paradox: while powerful LLM capabilities are easily integrated via <code>Spring AI</code> or <code>OpenAI SDKs</code>, the actual performance and reliability of these calls remain largely invisible.</p>
<h3 id="1-the-black-box-of-cost-and-performance-is-the-expensive-model-worth-it">1. The &ldquo;Black Box&rdquo; of Cost and Performance: Is the Expensive Model Worth It?</h3>
<p>Facing high LLM bills, organizations often only see a total sum paid to a provider, but cannot calculate the &ldquo;ROI&rdquo; within the application.</p>
<ul>
<li><strong>Blind Upgrades:</strong> You might switch to a premium flagship model for a better experience. But in your specific business scenario, does paying several times more per token actually yield lower latency or a faster <strong>TTFT (Time to First Token)</strong>?</li>
<li><strong>Lack of Real-World Benchmarks:</strong> Official benchmarks mean little without your real-world business requests. You need to know which model achieves the perfect balance between &ldquo;Token/Cost Consumption&rdquo; and &ldquo;Response Speed&rdquo; under your actual prompt lengths and concurrency levels.</li>
</ul>
<h3 id="2-the-vanishing-golden-timeout">2. The Vanishing &ldquo;Golden Timeout&rdquo;</h3>
<p>Many teams set timeouts for LLM calls arbitrarily (e.g., 30s or 60s).</p>
<ul>
<li><strong>Too Short:</strong> During peak periods or long-text generation, requests are frequently interrupted, causing business failure rates to soar.</li>
<li><strong>Too Long:</strong> If a provider hangs, requests pile up in memory, blocking execution threads and potentially leading to the collapse of the entire Java application or microservice cluster.
Only by mastering the <strong>P99/P95 Latency</strong> can you set rational timeout policies based on data rather than intuition.</li>
</ul>
<h3 id="3-the-overlooked-experience-killer-ttft">3. The Overlooked Experience Killer: TTFT</h3>
<p>In GenAI scenarios, a user&rsquo;s perception of speed depends less on the total duration of the conversation and more on <strong>&ldquo;when the first word appears.&rdquo;</strong> * A streaming response with a 10s total duration but a <strong>500ms TTFT</strong> feels instantaneous.</p>
<ul>
<li>A non-streaming response with a 5s total duration but a <strong>4s TTFT</strong> feels &ldquo;frozen.&rdquo;
If your observability system only tracks total latency, you miss the core UX metric that explains why users complain about &ldquo;AI slowness.&rdquo;</li>
</ul>
<hr />
<p><strong>SkyWalking 10.4: A &ldquo;Digital Dashboard&rdquo;</strong><br />
From the Application Perspective The <strong>Virtual GenAI</strong> capability introduced in Apache SkyWalking 10.4 fills this &ldquo;observability vacuum.&rdquo; It avoids reliance on external gateways by using application-side probes (like the Java Agent) to collect the most authentic data from the client&rsquo;s perspective.</p>
<ul>
<li><strong>Precise Latency Distribution:</strong> Multi-dimensional metrics (P50, P90, P99) help visualize LLM fluctuations to inform dynamic timeout strategies.</li>
<li><strong>Core UX Metric — TTFT Monitoring:</strong> Native support for first-token latency in streaming calls.</li>
<li><strong>Multi-dimensional Model Profiling:</strong> Aligns token usage, estimated cost, and performance across Providers and Models, helping you choose the most cost-effective solution for your specific needs.</li>
</ul>
<hr />
<h1 id="virtual-genai-observability">Virtual GenAI Observability</h1>
<blockquote>
<p><strong>Virtual GenAI</strong> represents Generative AI service nodes detected by probe plugins. All performance metrics are based on the <strong>GenAI Client Perspective</strong>.</p>
</blockquote>
<p>For instance, the <strong>Spring AI plugin</strong> in the Java Agent detects the response latency of a Chat Completion request. SkyWalking then visualizes these in the dashboard:</p>
<ul>
<li><strong>Traffic &amp; Success Rate</strong> (CPM &amp; SLA)</li>
<li><strong>Latency &amp; TTFT</strong></li>
<li><strong>Token Usage</strong> (Input/Output)</li>
<li><strong>Estimated Cost</strong></li>
</ul>
<p><strong>Screenshots:</strong>
<img alt="provider-dashboard-1.png" src="provider-dashboard-1.png" />
<img alt="provider-dashboard-2.png" src="provider-dashboard-2.png" />
<img alt="provider-dashboard-3.png" src="provider-dashboard-3.png" />
<img alt="model-dashboard-1.png" src="model-dashboard-1.png" />
<img alt="model-dashboard-2.png" src="model-dashboard-2.png" />
<img alt="model-dashboard-3.png" src="model-dashboard-3.png" /></p>
<h1 id="how-it-works">How It Works</h1>
<p>When the SkyWalking Java Agent or OTLP probes intercept calls to mainstream AI frameworks (e.g., Spring AI, OpenAI SDK), they report Trace data to the SkyWalking OAP.
The OAP aggregates and computes this data to generate performance metrics for both <strong>Providers</strong> and <strong>Models</strong>, which are then rendered in the built-in Virtual-GenAI dashboards.</p>
<h1 id="installation--configuration">Installation &amp; Configuration</h1>
<h2 id="requirements">Requirements</h2>
<ul>
<li><strong>SkyWalking Java Agent:</strong> &gt;= 9.7</li>
<li><strong>SkyWalking OAP:</strong> &gt;= 10.4</li>
</ul>
<h2 id="semantic-conventions--compatibility">Semantic Conventions &amp; Compatibility</h2>
<p>SkyWalking Virtual GenAI follows <strong>OpenTelemetry GenAI Semantic Conventions</strong>. OAP identifies GenAI-related Spans based on:</p>
<h3 id="skywalking-java-agent">SkyWalking Java Agent</h3>
<ul>
<li>Spans must be of type Exit, have the SpanLayer attribute set to GENAI, and contain the gen_ai.response.model tag.</li>
</ul>
<h3 id="otlp--zipkin-probes">OTLP / Zipkin Probes</h3>
<ul>
<li>Spans must contain the <code>gen_ai.response.model</code> tag.</li>
</ul>
<p>For details, refer to the E2E configurations:</p>
<ul>
<li><a href="https://github.com/apache/skywalking/blob/master/test/e2e-v2/cases/virtual-genai/docker-compose.yml">SkyWalking Java Agent Reporting</a></li>
<li><a href="https://github.com/apache/skywalking/blob/master/test/e2e-v2/cases/otlp-virtual-genai/docker-compose.yml">Probe Reporting OTLP Data</a></li>
<li><a href="https://github.com/apache/skywalking/blob/master/test/e2e-v2/cases/zipkin-virtual-genai/docker-compose.yml">Probe Reporting Zipkin Data</a></li>
</ul>
<hr />
<h1 id="genai-estimated-cost-configuration">GenAI Estimated Cost Configuration</h1>
<h2 id="overview">Overview</h2>
<p>SkyWalking provides a built-in <a href="https://github.com/apache/skywalking/blob/master/oap-server/server-starter/src/main/resources/gen-ai-config.yml">GenAI Billing Configuration File</a>.</p>
<p>This file defines how SkyWalking maps model names from Trace data to their corresponding providers and estimates the token cost for each LLM call. The estimated cost is displayed in the SkyWalking UI alongside trace and metric data, helping users intuitively understand the financial impact of their GenAI usage.</p>
<blockquote>
<p><strong>Important:</strong> The pricing in this file is intended for cost estimation only and must not be treated as actual billing or invoice amounts. Users are advised to regularly verify the latest rates on the providers&rsquo; official pricing pages.</p>
</blockquote>
<h2 id="configuration-structure">Configuration Structure</h2>
<h3 id="top-level-fields">Top-level Fields</h3>
<table>
  <thead>
      <tr>
          <th style="text-align: left;">Field</th>
          <th style="text-align: left;">Type</th>
          <th style="text-align: left;">Description</th>
      </tr>
  </thead>
  <tbody>
      <tr>
          <td style="text-align: left;"><code>last-updated</code></td>
          <td style="text-align: left;"><code>date</code></td>
          <td style="text-align: left;">The last update date of the pricing data. All prices are based on public billing standards announced by providers prior to this date.</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>providers</code></td>
          <td style="text-align: left;"><code>list</code></td>
          <td style="text-align: left;">List of GenAI provider definitions. Each entry contains matching rules and specific model pricing information.</td>
      </tr>
  </tbody>
</table>
<h3 id="provider-definition">Provider Definition</h3>
<p>Each entry under <code>providers</code> defines a GenAI provider:</p>
<div class="highlight"><pre tabindex="0"><code class="language-yaml"><span style="display: flex;"><span><span style="color: #0550ae;">providers</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span>- <span style="color: #0550ae;">provider</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>&lt;provider-name&gt;<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">prefix-match</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span>- &lt;prefix-1&gt;<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span>- &lt;prefix-2&gt;<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">models</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span>- <span style="color: #0550ae;">name</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>&lt;model-name&gt;<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">aliases</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #1f2328;">[</span>&lt;alias-1&gt;, &lt;alias-2&gt;]<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">input-estimated-cost-per-m</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>&lt;cost&gt;<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">output-estimated-cost-per-m</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>&lt;cost&gt;<span style="color: #fff;">
</span></span></span></code></pre></div><table>
  <thead>
      <tr>
          <th style="text-align: left;">Field</th>
          <th style="text-align: left;">Type</th>
          <th style="text-align: left;">Required</th>
          <th style="text-align: left;">Description</th>
      </tr>
  </thead>
  <tbody>
      <tr>
          <td style="text-align: left;"><code>provider</code></td>
          <td style="text-align: left;"><code>string</code></td>
          <td style="text-align: left;">Yes</td>
          <td style="text-align: left;">The provider identifier (e.g., <code>openai</code>, <code>anthropic</code>, <code>gemini</code>). It is displayed as the Virtual GenAI service name in SkyWalking.</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>prefix-match</code></td>
          <td style="text-align: left;"><code>list[string]</code></td>
          <td style="text-align: left;">Yes</td>
          <td style="text-align: left;">A list of prefixes used to match model names to this provider. If a model name in the Trace data starts with any of these prefixes, it will be mapped to this provider.</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>models</code></td>
          <td style="text-align: left;"><code>list[model]</code></td>
          <td style="text-align: left;">No</td>
          <td style="text-align: left;">A list of model definitions containing pricing information. If omitted, the system can still identify the provider but will not perform cost estimation.</td>
      </tr>
  </tbody>
</table>
<h3 id="model-definition">Model Definition</h3>
<p>Each entry under <code>models</code> defines the pricing for a specific model:</p>
<table>
  <thead>
      <tr>
          <th style="text-align: left;">Field</th>
          <th style="text-align: left;">Type</th>
          <th style="text-align: left;">Required</th>
          <th style="text-align: left;">Description</th>
      </tr>
  </thead>
  <tbody>
      <tr>
          <td style="text-align: left;"><code>name</code></td>
          <td style="text-align: left;"><code>string</code></td>
          <td style="text-align: left;">Yes</td>
          <td style="text-align: left;">The standard model name used for matching.</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>aliases</code></td>
          <td style="text-align: left;"><code>list[string]</code></td>
          <td style="text-align: left;">No</td>
          <td style="text-align: left;">Alternative names that should resolve to the same billing entry. This is useful when providers use different naming conventions (see the &ldquo;Model Aliases&rdquo; section).</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>input-estimated-cost-per-m</code></td>
          <td style="text-align: left;"><code>float</code></td>
          <td style="text-align: left;">No</td>
          <td style="text-align: left;">Estimated cost per 1,000,000 (one million) input (Prompt) tokens. The default unit is USD.</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>output-estimated-cost-per-m</code></td>
          <td style="text-align: left;"><code>float</code></td>
          <td style="text-align: left;">No</td>
          <td style="text-align: left;">Estimated cost per 1,000,000 (one million) output (Completion) tokens. The default unit is USD.</td>
      </tr>
  </tbody>
</table>
<h2 id="model-matching-mechanism">Model Matching Mechanism</h2>
<h3 id="provider-level-prefix-matching">Provider-Level Prefix Matching</h3>
<p>When SkyWalking receives a Trace containing a GenAI call, it determines the <strong>Provider</strong> based on the following priority order:</p>
<ol>
<li><strong><code>gen_ai.provider.name</code> tag</strong>: This tag is retrieved first. It follows the latest <code>OpenTelemetry</code> GenAI semantic conventions.</li>
<li><strong><code>gen_ai.system</code> tag</strong>: If the above tag is missing, the system falls back to this legacy tag. Note: This tag is only parsed when processing OTLP or Zipkin format data, primarily for compatibility with older versions of libraries like the Python auto-instrumentation.</li>
<li><strong>Prefix Matching</strong>: If neither of the above tags exists, <code>SkyWalking</code> reads the <code>prefix-match</code> rules defined in <code>gen-ai-config.yml</code> and attempts to identify the provider by matching the <strong>Model Name</strong>.</li>
</ol>
<div class="highlight"><pre tabindex="0"><code class="language-yaml"><span style="display: flex;"><span>- <span style="color: #0550ae;">provider</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>openai<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">prefix-match</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span>- gpt<span style="color: #fff;">
</span></span></span></code></pre></div><p>Any model name starting with <strong>gpt</strong> (such as <strong>gpt-4o</strong>, <strong>gpt-4.1-mini</strong>, or <strong>gpt-5-nano</strong>) will be mapped to the <strong>openai</strong> provider.
A single provider can have multiple prefixes:</p>
<div class="highlight"><pre tabindex="0"><code class="language-yaml"><span style="display: flex;"><span>- <span style="color: #0550ae;">provider</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>tencent<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">prefix-match</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span>- hunyuan<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span>- Tencent<span style="color: #fff;">
</span></span></span></code></pre></div><h3 id="model-level-longest-prefix-matching">Model-level Longest-Prefix Matching</h3>
<p>Once the provider is determined, SkyWalking uses a Trie-based longest-prefix matching algorithm to find the best billing entry. This is crucial because model names returned in provider API responses often include version numbers or timestamps, differing from the base model name in the config.
Example OpenAI config:</p>
<div class="highlight"><pre tabindex="0"><code class="language-yaml"><span style="display: flex;"><span><span style="color: #0550ae;">models</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span>- <span style="color: #0550ae;">name</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>gpt-4o<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">input-estimated-cost-per-m</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #0550ae;">2.5</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">output-estimated-cost-per-m</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #0550ae;">10.0</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span>- <span style="color: #0550ae;">name</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>gpt-4o-mini<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">input-estimated-cost-per-m</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #0550ae;">0.15</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">output-estimated-cost-per-m</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #0550ae;">0.6</span><span style="color: #fff;">
</span></span></span></code></pre></div><p>Matching behavior:</p>
<table>
  <thead>
      <tr>
          <th style="text-align: left;">Model Name in Trace</th>
          <th style="text-align: left;">Matched Configuration Entry</th>
          <th style="text-align: left;">Reason</th>
      </tr>
  </thead>
  <tbody>
      <tr>
          <td style="text-align: left;"><code>gpt-4o</code></td>
          <td style="text-align: left;"><code>gpt-4o</code></td>
          <td style="text-align: left;">Exact match</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>gpt-4o-2024-08-06</code></td>
          <td style="text-align: left;"><code>gpt-4o</code></td>
          <td style="text-align: left;">Longest prefix is <code>gpt-4o</code></td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>gpt-4o-mini</code></td>
          <td style="text-align: left;"><code>gpt-4o-mini</code></td>
          <td style="text-align: left;">Exact match (Longer prefix <code>gpt-4o-mini</code> takes priority over <code>gpt-4o</code>)</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>gpt-4o-mini-2024-07-18</code></td>
          <td style="text-align: left;"><code>gpt-4o-mini</code></td>
          <td style="text-align: left;">Longest prefix is <code>gpt-4o-mini</code></td>
      </tr>
  </tbody>
</table>
<p>This mechanism ensures versioned API model names map to the correct pricing tier without requiring exact full names in the configuration file.</p>
<h3 id="model-aliases">Model Aliases</h3>
<p>Some providers use different naming conventions across API responses and documentation. For example, Anthropic&rsquo;s model might appear as <code>claude-4-sonnet</code> or <code>claude-sonnet-4</code>. The aliases field supports both formats under a single billing entry:</p>
<div class="highlight"><pre tabindex="0"><code class="language-yaml"><span style="display: flex;"><span>- <span style="color: #0550ae;">name</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>claude-4-sonnet<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">aliases</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #1f2328;">[</span>claude-sonnet-4]<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">input-estimated-cost-per-m</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #0550ae;">3.0</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">output-estimated-cost-per-m</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #0550ae;">15.0</span><span style="color: #fff;">
</span></span></span></code></pre></div><p>Under this configuration, <code>claude-4-sonnet</code> and <code>claude-sonnet-4</code> (as well as any versioned variants, such as <code>claude-sonnet-4-20250514</code>) will resolve to the same <strong>billing entry</strong>.<br />
<strong>Note:</strong> Aliases also participate in <strong>longest prefix matching</strong>. Therefore, <code>claude-sonnet-4-20250514</code> will match the alias <code>claude-sonnet-4</code>, which in turn resolves to the pricing information for <code>claude-4-sonnet</code>.</p>
<h2 id="custom-configuration">Custom Configuration</h2>
<h3 id="adding-a-new-provider">Adding a New Provider</h3>
<p>To add a provider that is not included in the default configuration:</p>
<div class="highlight"><pre tabindex="0"><code class="language-yaml"><span style="display: flex;"><span><span style="color: #0550ae;">providers</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #57606a;"># ... Existing providers ...</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span>- <span style="color: #0550ae;">provider</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>ollama<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">prefix-match</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span>- mymodel<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">models</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span>- <span style="color: #0550ae;">name</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>mymodel-large<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">input-estimated-cost-per-m</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #0550ae;">1.0</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">output-estimated-cost-per-m</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #0550ae;">5.0</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span>- <span style="color: #0550ae;">name</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>mymodel-small<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">input-estimated-cost-per-m</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #0550ae;">0.1</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">      </span><span style="color: #0550ae;">output-estimated-cost-per-m</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #0550ae;">0.5</span><span style="color: #fff;">
</span></span></span></code></pre></div><p>For OTLP/Zipkin data, a dedicated estimated tag has been added. You can now view the cost of each GenAI call directly on the UI.
<img alt="otlp-estimated-tag" src="otlp-estimated-tag.png" /></p>
<h1 id="main-metrics">Main Metrics</h1>
<h2 id="1provider-level">1.Provider Level</h2>
<table>
  <thead>
      <tr>
          <th style="text-align: left;">Metric ID</th>
          <th style="text-align: left;">Description</th>
          <th style="text-align: left;">Meaning</th>
      </tr>
  </thead>
  <tbody>
      <tr>
          <td style="text-align: left;"><code>gen_ai_provider_cpm</code></td>
          <td style="text-align: left;">Calls Per Minute</td>
          <td style="text-align: left;">Requests per minute (Throughput)</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>gen_ai_provider_sla</code></td>
          <td style="text-align: left;">Success Rate</td>
          <td style="text-align: left;">Request success rate</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>gen_ai_provider_resp_time</code></td>
          <td style="text-align: left;">Avg Response Time</td>
          <td style="text-align: left;">Average response time</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>gen_ai_provider_latency_percentile</code></td>
          <td style="text-align: left;">Latency Percentiles</td>
          <td style="text-align: left;">Response time percentiles (P50, P75, P90, P95, P99)</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>gen_ai_provider_input_tokens_sum/avg</code></td>
          <td style="text-align: left;">Input Token Usage</td>
          <td style="text-align: left;">Total and average input token usage</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>gen_ai_provider_output_tokens_sum/avg</code></td>
          <td style="text-align: left;">Output Token Usage</td>
          <td style="text-align: left;">Total and average output token usage</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>gen_ai_provider_total_estimated_cost/avg</code></td>
          <td style="text-align: left;">Estimated Cost</td>
          <td style="text-align: left;">Total estimated cost and average cost per call</td>
      </tr>
  </tbody>
</table>
<h2 id="2-model-level">2. Model Level</h2>
<table>
  <thead>
      <tr>
          <th style="text-align: left;">Metric ID</th>
          <th style="text-align: left;">Description</th>
          <th style="text-align: left;">Meaning</th>
      </tr>
  </thead>
  <tbody>
      <tr>
          <td style="text-align: left;"><code>gen_ai_model_call_cpm</code></td>
          <td style="text-align: left;">Calls Per Minute</td>
          <td style="text-align: left;">Requests per minute for this specific model</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>gen_ai_model_sla</code></td>
          <td style="text-align: left;">Success Rate</td>
          <td style="text-align: left;">Model-specific request success rate</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>gen_ai_model_latency_avg/percentile</code></td>
          <td style="text-align: left;">Latency</td>
          <td style="text-align: left;">Average and percentiles of model response duration</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>gen_ai_model_ttft_avg/percentile</code></td>
          <td style="text-align: left;">TTFT</td>
          <td style="text-align: left;">Time to First Token (Streaming only)</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>gen_ai_model_input_tokens_sum/avg</code></td>
          <td style="text-align: left;">Input Token Usage</td>
          <td style="text-align: left;">Detailed input token consumption for the model</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>gen_ai_model_output_tokens_sum/avg</code></td>
          <td style="text-align: left;">Output Token Usage</td>
          <td style="text-align: left;">Detailed output token consumption for the model</td>
      </tr>
      <tr>
          <td style="text-align: left;"><code>gen_ai_model_total_estimated_cost/avg</code></td>
          <td style="text-align: left;">Estimated Cost</td>
          <td style="text-align: left;">Estimated total cost and average cost for the model</td>
      </tr>
  </tbody>
</table>
<h2 id="recommended-usage-scenarios">Recommended Usage Scenarios</h2>
<ul>
<li>Performance Evaluation: Use Latency and Time to First Token (TTFT) metrics to analyze model inference efficiency and the end-user interaction experience.</li>
<li>Token Monitoring: Real-time monitoring of Input and Output token consumption to analyze resource utilization across different business scenarios.</li>
<li>Cost Alerting: Set alert thresholds based on Estimated Cost or token consumption to promptly detect abnormal calls and prevent budget overruns.</li>
</ul>
