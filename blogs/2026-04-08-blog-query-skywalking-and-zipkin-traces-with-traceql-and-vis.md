---
title: "Blog: Query SkyWalking and Zipkin Traces with TraceQL and Visualize in Grafana"
url: "/blog/2026-04-08-traceql/"
date: "Wed, 08 Apr 2026 00:00:00 +0000"
author: ""
feed_url: "https://skywalking.apache.org/feed"
---
<h1 id="query-skywalking-and-zipkin-traces-with-traceql-and-visualize-in-grafana">Query SkyWalking and Zipkin Traces with TraceQL and Visualize in Grafana</h1>
<p>Apache SkyWalking introduced <strong>TraceQL</strong> support in version <strong>10.4.0</strong>, implementing
<a href="https://grafana.com/docs/tempo/v2.10.x/api_docs/">Grafana Tempo&rsquo;s HTTP query APIs</a> so that
Grafana can query and visualize traces stored in SkyWalking without any additional plugins.
This means you can now use the familiar Grafana Tempo data source to search, filter, and
drill into both <strong>SkyWalking native traces</strong> and <strong>Zipkin-compatible traces</strong> — all served
by your existing SkyWalking OAP server.</p>
<h2 id="architecture-overview">Architecture Overview</h2>
<pre tabindex="0"><code>┌────────────────────┐         Tempo HTTP API           ┌─────────────────────────────┐
│                    │  ──── /skywalking/api/search ──► │  SkyWalking Native Backend  │
│      Grafana       │                                  │  (Query Traces V2 API)      │
│  (Tempo Data Src)  │                                  ├─────────────────────────────┤
│                    │  ──── /zipkin/api/search ──────► │  Zipkin-Compatible Backend  │
└────────────────────┘                                  └──────────┬──────────────────┘
                                                                   │
                                                        ┌──────────▼──────────────────┐
                                                        │    SkyWalking OAP Server    │
                                                        │  ┌───────────────────────┐  │
                                                        │  │   TraceQL Service     │  │
                                                        │  │  (port 3200)          │  │
                                                        │  └───────────────────────┘  │
                                                        │  ┌───────────────────────┐  │
                                                        │  │  Storage (BanyanDB /  │  │
                                                        │  │  Elasticsearch / …)   │  │
                                                        │  └───────────────────────┘  │
                                                        └─────────────────────────────┘
</code></pre><p>The TraceQL Service sits inside the OAP server and exposes the Tempo-compatible HTTP API on
port <code>3200</code> (default). It converts traces from their native format into
<a href="https://github.com/grafana/tempo/blob/main/pkg/tempopb/tempo.proto">Tempo&rsquo;s format</a>,
where the trace detail part (<code>Trace</code> message) reuses OTLP <code>Trace</code> definitions.</p>
<h2 id="limitations-and-supported-traceql-features">Limitations and Supported TraceQL Features</h2>
<p>TraceQL is a rich query language, but SkyWalking currently implements a practical subset.
The following features are <strong>supported</strong>:</p>
<table>
  <thead>
      <tr>
          <th>Feature</th>
          <th>Examples</th>
      </tr>
  </thead>
  <tbody>
      <tr>
          <td>Spanset filter</td>
          <td><code>{resource.service.name=&quot;frontend&quot;}</code></td>
      </tr>
      <tr>
          <td>Resource attributes</td>
          <td><code>resource.service.name</code></td>
      </tr>
      <tr>
          <td>Span attributes</td>
          <td><code>span.http.method</code>, <code>span.http.status_code</code></td>
      </tr>
      <tr>
          <td>Intrinsic fields</td>
          <td><code>duration</code>, <code>name</code>, <code>status</code></td>
      </tr>
      <tr>
          <td>Comparison operators</td>
          <td><code>=</code>, <code>&gt;</code>, <code>&gt;=</code>, <code>&lt;</code>, <code>&lt;=</code></td>
      </tr>
      <tr>
          <td>Compound conditions</td>
          <td><code>{resource.service.name=&quot;frontend&quot; &amp;&amp; duration&gt;100ms}</code></td>
      </tr>
      <tr>
          <td>Duration units</td>
          <td><code>us</code>/<code>µs</code>, <code>ms</code>, <code>s</code>, <code>m</code>, <code>h</code></td>
      </tr>
  </tbody>
</table>
<p>The following features are <strong>not yet supported</strong>:</p>
<ul>
<li>Spanset logical operations (<code>{...} AND {...}</code>, <code>{...} OR {...}</code>)</li>
<li>Pipeline operations (<code>|</code> operator)</li>
<li>Aggregate functions (<code>count()</code>, <code>avg()</code>, <code>max()</code>, <code>min()</code>, <code>sum()</code>)</li>
<li>Regular expression matching (<code>=~</code>, <code>!~</code>)</li>
<li><code>event</code> and <code>link</code> scopes</li>
<li><code>kind</code> intrinsic field</li>
<li>Streaming mode (must be disabled in the Grafana Tempo data source settings)</li>
</ul>
<blockquote>
<p><strong>Important</strong>: SkyWalking native trace support in TraceQL is based on the
<a href="https://skywalking.apache.org/docs/main/next/en/api/query-protocol/#trace-v2">Query Traces V2 API</a>.
Currently, only <strong>BanyanDB</strong> storage implements this API. Other storage backends
(e.g., Elasticsearch, MySQL, PostgreSQL) do not support SkyWalking native trace queries via TraceQL.
Zipkin-compatible traces are <strong>not</strong> subject to this restriction.</p>
</blockquote>
<h2 id="trace-format-conversion">Trace Format Conversion</h2>
<p>Since the trace detail part of Tempo&rsquo;s format reuses
<a href="https://opentelemetry.io/docs/reference/specification/protocol/">OTLP Trace</a> definitions,
the conversion descriptions below refer to OTLP field names (e.g., span kind, status code).</p>
<h3 id="skywalking-native-trace">SkyWalking Native Trace</h3>
<h4 id="trace-id-encoding">Trace ID Encoding</h4>
<p>SkyWalking native trace IDs are arbitrary strings (e.g.,
<code>2a2e04e8d1114b14925c04a6321ca26c.38.17739924187687539</code>), while Grafana Tempo requires
pure hex-encoded trace IDs. The TraceQL Service encodes each UTF-8 byte of the original trace
ID as two lowercase hex characters:</p>
<pre tabindex="0"><code>Original:  2a2e04e8d1114b14925c04a6321ca26c.38.17739924187687539
Encoded:   32613265303465386431313134623134393235633034613633323163613236632e33382e3137373339393234313837363837353339
</code></pre><p>This encoded hex trace ID is what appears in all API responses and in Grafana. When you click a
trace ID in Grafana, the TraceQL Service automatically decodes it back to the original SkyWalking
trace ID for the internal query.</p>
<h4 id="span-kind-mapping">Span Kind Mapping</h4>
<table>
  <thead>
      <tr>
          <th>SkyWalking Span Type</th>
          <th>OTLP Span Kind</th>
      </tr>
  </thead>
  <tbody>
      <tr>
          <td><code>Entry</code></td>
          <td><code>SPAN_KIND_SERVER</code></td>
      </tr>
      <tr>
          <td><code>Exit</code></td>
          <td><code>SPAN_KIND_CLIENT</code></td>
      </tr>
      <tr>
          <td><code>Local</code></td>
          <td><code>SPAN_KIND_INTERNAL</code></td>
      </tr>
  </tbody>
</table>
<h4 id="status-mapping">Status Mapping</h4>
<table>
  <thead>
      <tr>
          <th>SkyWalking <code>isError</code></th>
          <th>OTLP Status Code</th>
      </tr>
  </thead>
  <tbody>
      <tr>
          <td><code>true</code></td>
          <td><code>STATUS_CODE_ERROR</code></td>
      </tr>
      <tr>
          <td><code>false</code></td>
          <td><code>STATUS_CODE_OK</code></td>
      </tr>
  </tbody>
</table>
<h4 id="spanattachedevents">SpanAttachedEvents</h4>
<p>SkyWalking <a href="https://skywalking.apache.org/docs/main/next/en/concepts-and-designs/event/">SpanAttachedEvents</a> are converted to OTLP span events,
with <code>tags</code> mapped as string attributes and <code>summary</code> mapped as numeric attributes (serialized as strings).</p>
<h3 id="zipkin-trace">Zipkin Trace</h3>
<h4 id="span-kind-mapping-1">Span Kind Mapping</h4>
<table>
  <thead>
      <tr>
          <th>Zipkin Span Kind</th>
          <th>OTLP Span Kind</th>
      </tr>
  </thead>
  <tbody>
      <tr>
          <td><code>CLIENT</code></td>
          <td><code>SPAN_KIND_CLIENT</code></td>
      </tr>
      <tr>
          <td><code>SERVER</code></td>
          <td><code>SPAN_KIND_SERVER</code></td>
      </tr>
      <tr>
          <td><code>PRODUCER</code></td>
          <td><code>SPAN_KIND_PRODUCER</code></td>
      </tr>
      <tr>
          <td><code>CONSUMER</code></td>
          <td><code>SPAN_KIND_CONSUMER</code></td>
      </tr>
  </tbody>
</table>
<h4 id="status-mapping-1">Status Mapping</h4>
<ol>
<li>If the <code>otel.status_code</code> tag is present, it is used directly.</li>
<li>Otherwise, if the <code>error</code> tag equals <code>true</code>, the status is <code>STATUS_CODE_ERROR</code>.</li>
<li>If neither tag is present, the status defaults to <code>STATUS_CODE_UNSET</code>.</li>
</ol>
<h4 id="endpoint-and-annotation-mapping">Endpoint and Annotation Mapping</h4>
<p>Zipkin endpoint fields are mapped to OTLP attributes (e.g., <code>localEndpoint.ipv4</code> → <code>net.host.ip</code>),
and Zipkin annotations are converted to OTLP span events.</p>
<p>For the full conversion details, see the <a href="https://skywalking.apache.org/docs/main/next/en/api/traceql-service/">TraceQL Service documentation</a>.</p>
<h2 id="how-to-enable-traceql">How to Enable TraceQL</h2>
<h3 id="step-1-enable-the-traceql-module">Step 1: Enable the TraceQL Module</h3>
<p>By default, the TraceQL module is <strong>disabled</strong> (<code>selector: ${SW_TRACEQL:-}</code>). To enable it, set
the selector to <code>default</code>:</p>
<div class="highlight"><pre tabindex="0"><code class="language-yaml"><span style="display: flex;"><span><span style="color: #57606a;"># In application.yml</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #0550ae;">traceQL</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">selector</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>${SW_TRACEQL:default}<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">default</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">enableDatasourceSkywalking</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>${SW_TRACEQL_ENABLE_DATASOURCE_SKYWALKING:true}<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">enableDatasourceZipkin</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>${SW_TRACEQL_ENABLE_DATASOURCE_ZIPKIN:true}<span style="color: #fff;">
</span></span></span></code></pre></div><p>Or via environment variables:</p>
<div class="highlight"><pre tabindex="0"><code class="language-bash"><span style="display: flex;"><span><span style="color: #6639ba;">export</span> <span style="color: #953800;">SW_TRACEQL</span><span style="color: #0550ae;">=</span>default
</span></span><span style="display: flex;"><span><span style="color: #6639ba;">export</span> <span style="color: #953800;">SW_TRACEQL_ENABLE_DATASOURCE_SKYWALKING</span><span style="color: #0550ae;">=</span><span style="color: #6639ba;">true</span>
</span></span><span style="display: flex;"><span><span style="color: #6639ba;">export</span> <span style="color: #953800;">SW_TRACEQL_ENABLE_DATASOURCE_ZIPKIN</span><span style="color: #0550ae;">=</span><span style="color: #6639ba;">true</span>
</span></span></code></pre></div><h3 id="step-2-enable-the-zipkin-receiver-for-zipkin-traces-only">Step 2: Enable the Zipkin Receiver (for Zipkin traces only)</h3>
<p>If you want to query Zipkin traces, you also need to enable the Zipkin receiver so that
SkyWalking can ingest Zipkin trace data:</p>
<div class="highlight"><pre tabindex="0"><code class="language-yaml"><span style="display: flex;"><span><span style="color: #57606a;"># In application.yml</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #0550ae;">receiver-zipkin</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">selector</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>${SW_RECEIVER_ZIPKIN:default}<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">  </span><span style="color: #0550ae;">default</span><span style="color: #1f2328;">:</span><span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">searchableTracesTags</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>${SW_ZIPKIN_SEARCHABLE_TAG_KEYS:http.method}<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">sampleRate</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>${SW_ZIPKIN_SAMPLE_RATE:10000}<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">restHost</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>${SW_RECEIVER_ZIPKIN_REST_HOST:0.0.0.0}<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #fff;">    </span><span style="color: #0550ae;">restPort</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>${SW_RECEIVER_ZIPKIN_REST_PORT:9411}<span style="color: #fff;">
</span></span></span></code></pre></div><p>Or via environment variable:</p>
<div class="highlight"><pre tabindex="0"><code class="language-bash"><span style="display: flex;"><span><span style="color: #6639ba;">export</span> <span style="color: #953800;">SW_RECEIVER_ZIPKIN</span><span style="color: #0550ae;">=</span>default
</span></span></code></pre></div><h3 id="full-configuration-reference">Full Configuration Reference</h3>
<p>For the complete list of all configuration options and their default values, see the
<a href="https://skywalking.apache.org/docs/main/next/en/api/traceql-service/#configuration">Configuration section of the TraceQL Service documentation</a>.</p>
<h2 id="configuring-grafana-tempo-data-source">Configuring Grafana Tempo Data Source</h2>
<blockquote>
<p><strong>Prerequisite</strong>: Grafana <strong>12 or later</strong> is required.</p>
</blockquote>
<p>Each trace backend (SkyWalking native / Zipkin) needs its own Tempo data source in Grafana,
because each is served under a different context path.</p>
<h3 id="context-paths">Context Paths</h3>
<p>The two backends are served under separate context paths on the same port:</p>
<table>
  <thead>
      <tr>
          <th>Backend</th>
          <th>Default Context Path</th>
          <th>Env Variable</th>
          <th>Full Default URL</th>
      </tr>
  </thead>
  <tbody>
      <tr>
          <td>SkyWalking native</td>
          <td><code>/skywalking</code></td>
          <td><code>SW_TRACEQL_REST_CONTEXT_PATH_SKYWALKING</code></td>
          <td><code>http://&lt;oap-host&gt;:3200/skywalking</code></td>
      </tr>
      <tr>
          <td>Zipkin</td>
          <td><code>/zipkin</code></td>
          <td><code>SW_TRACEQL_REST_CONTEXT_PATH_ZIPKIN</code></td>
          <td><code>http://&lt;oap-host&gt;:3200/zipkin</code></td>
      </tr>
  </tbody>
</table>
<h3 id="setting-up-the-skywalking-data-source">Setting Up the SkyWalking Data Source</h3>
<ol>
<li>In Grafana, go to <strong>Configuration → Data Sources → Add data source</strong>.</li>
<li>Choose <strong>Tempo</strong>.</li>
<li>Set the URL to <code>http://&lt;oap-host&gt;:3200/skywalking</code>.</li>
<li><strong>Disable the Streaming option</strong> (SkyWalking does not support streaming mode).</li>
</ol>
<p><img alt="Disable Streaming" src="grafana-tempo-datasource-streaming.png" /></p>
<ol start="5">
<li>Save and test the data source.</li>
</ol>
<p><img alt="SkyWalking Data Source" src="grafana-tempo-skywalking-datasource.png" /></p>
<h3 id="setting-up-the-zipkin-data-source">Setting Up the Zipkin Data Source</h3>
<p>Same as above, but set the URL to <code>http://&lt;oap-host&gt;:3200/zipkin</code>.</p>
<p><img alt="Zipkin Data Source" src="grafana-tempo-zipkin-datasource.png" /></p>
<h2 id="configuring-trace-list-result-tags">Configuring Trace List Result Tags</h2>
<p>When you search for traces in Grafana, the trace list panel shows a summary of each trace.
The <code>tracesListResultTags</code> configuration controls <strong>which span tags are included in the search
result</strong> and displayed as columns in the trace list.</p>
<table>
  <thead>
      <tr>
          <th>Env Variable</th>
          <th>Default Value</th>
          <th>Purpose</th>
      </tr>
  </thead>
  <tbody>
      <tr>
          <td><code>SW_TRACEQL_ZIPKIN_TRACES_LIST_RESULT_TAGS</code></td>
          <td><code>http.method,error</code></td>
          <td>Tags shown for Zipkin traces</td>
      </tr>
      <tr>
          <td><code>SW_TRACEQL_SKYWALKING_TRACES_LIST_RESULT_TAGS</code></td>
          <td><code>http.method,http.status_code,rpc.status_code,db.type,db.instance,mq.queue,mq.topic,mq.broker</code></td>
          <td>Tags shown for SkyWalking traces</td>
      </tr>
  </tbody>
</table>
<p>Note that <code>service.name</code> and <code>span.kind</code> are <strong>always included</strong> regardless of this setting.</p>
<p>These tags appear as attribute columns in the Grafana Tempo trace search results, making it
easier to identify and group traces at a glance:</p>
<p><strong>SkyWalking native trace list:</strong></p>
<p><img alt="SkyWalking Trace List" src="grafana-tempo-skywalking-trace-list.png" /></p>
<p><strong>Zipkin trace list:</strong></p>
<p><img alt="Zipkin Trace List" src="grafana-tempo-zipkin-trace-list.png" /></p>
<p>You can customize these tags based on your application&rsquo;s instrumentation. For example, if your
services heavily use messaging, you might add <code>mq.destination</code> or <code>messaging.system</code> to the list.</p>
<h2 id="building-a-trace-dashboard-in-grafana">Building a Trace Dashboard in Grafana</h2>
<h3 id="skywalking-native-trace-dashboard">SkyWalking Native Trace Dashboard</h3>
<h4 id="step-1-explore-and-save">Step 1: Explore and Save</h4>
<ol>
<li>Go to the <strong>Explore</strong> page in Grafana.</li>
<li>Select the Tempo data source you configured for SkyWalking (e.g., <code>SkyWalkingTraceQL</code>).</li>
<li>Run a test query, then click <strong>Add to dashboard</strong> and save it as <code>SkyWalking Trace</code>.</li>
</ol>
<p><img alt="SkyWalking Explore" src="grafana-tempo-skywalking-explore.png" /></p>
<h4 id="step-2-configure-variables">Step 2: Configure Variables</h4>
<p>Add dashboard variables so users can filter traces dynamically (e.g., by service name):</p>
<p><img alt="SkyWalking Variables" src="grafana-tempo-skywalking-variables.png" /></p>
<h4 id="step-3-add-a-trace-panel">Step 3: Add a Trace Panel</h4>
<ol>
<li>Choose a <strong>Table</strong> chart (or edit the panel you saved).</li>
<li>Set <strong>Query type</strong> to <code>Search</code>.</li>
<li>Set the <strong>Service Name</strong> query condition to the variable <code>$Service</code>.</li>
<li>Add other query conditions as needed (e.g., duration, span name, tags).</li>
<li>Test and save.</li>
</ol>
<p><img alt="SkyWalking Panel" src="grafana-tempo-skywalking-panel.png" /></p>
<h4 id="step-4-view-trace-details">Step 4: View Trace Details</h4>
<p>Click any trace ID in the trace panel to jump to the Explore page showing the full trace
waterfall view with all spans, tags, and events:</p>
<p><img alt="SkyWalking Trace Detail" src="grafana-tempo-skywalking-trace-detail.png" /></p>
<h3 id="zipkin-trace-dashboard">Zipkin Trace Dashboard</h3>
<p>The setup for Zipkin traces is identical to SkyWalking native traces — just use the Zipkin
Tempo data source you configured (e.g., <code>ZipkinTraceQL</code>).</p>
<p><strong>Zipkin trace detail view:</strong></p>
<p><img alt="Zipkin Trace Detail" src="grafana-tempo-zipkin-trace-detail.png" /></p>
<h2 id="summary">Summary</h2>
<p>With TraceQL support in SkyWalking 10.4.0, you can now leverage Grafana&rsquo;s powerful Tempo
data source to query and visualize both SkyWalking native traces and Zipkin-compatible traces.
The key points to remember:</p>
<ol>
<li><strong>Enable the TraceQL module</strong> by setting <code>SW_TRACEQL=default</code> and enabling the desired backends.</li>
<li><strong>Configure separate Tempo data sources</strong> in Grafana for each backend (<code>/skywalking</code> and <code>/zipkin</code>).</li>
<li><strong>Disable the Streaming option</strong> in the Grafana Tempo data source settings.</li>
<li><strong>Customize result tags</strong> via <code>SW_TRACEQL_SKYWALKING_TRACES_LIST_RESULT_TAGS</code> and <code>SW_TRACEQL_ZIPKIN_TRACES_LIST_RESULT_TAGS</code> to control what&rsquo;s shown in search results.</li>
<li><strong>SkyWalking native trace queries require BanyanDB</strong> storage (Zipkin traces work with all storage backends).</li>
</ol>
<p>For the complete API reference and conversion details, see the
<a href="https://skywalking.apache.org/docs/main/next/en/api/traceql-service/">TraceQL Service documentation</a>.
For Grafana integration details, see
<a href="https://skywalking.apache.org/docs/main/next/en/setup/backend/ui-grafana/#use-grafana-as-the-ui">Use Grafana As The UI</a>.</p>
