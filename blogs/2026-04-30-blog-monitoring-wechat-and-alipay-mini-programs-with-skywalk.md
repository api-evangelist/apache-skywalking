---
title: "Blog: Monitoring WeChat and Alipay Mini Programs with SkyWalking"
url: "/blog/2026-04-30-mini-program-monitoring-with-skywalking/"
date: "Thu, 30 Apr 2026 00:00:00 +0000"
author: ""
feed_url: "https://skywalking.apache.org/feed"
---
<p>Mini programs are a major part of the mobile experience in China, but the open-source observability ecosystem has long focused on web browsers and native apps. SkyWalking already covers browser (client-js), iOS, and the server side; mini programs and Android were the remaining gaps. With <a href="https://github.com/SkyAPM/mini-program-monitor">SkyAPM/mini-program-monitor</a> joining the SkyWalking ecosystem, the mini-program half of that gap is closed — one SDK supports both WeChat and Alipay, and the matching OAP-side component IDs, MAL rules, and UI templates are merged on <code>main</code> and will ship with 10.5.0.</p>
<p>This post is for teams that already run a SkyWalking backend and want to bring their mini programs into the same observability stack. The interesting parts aren&rsquo;t <em>that</em> the project exists — they are how the data flows from a mini program to a SkyWalking dashboard, how the two platforms coexist, and what design trade-offs you should know about before rolling this out.</p>
<h2 id="data-path">Data path</h2>
<p>The SDK uses two protocols:</p>
<ul>
<li><strong>OTLP HTTP</strong> (error logs, performance metrics, request metrics) → OAP <code>/v1/logs</code>, <code>/v1/metrics</code></li>
<li><strong>SkyWalking native</strong> (distributed tracing segments, optional) → OAP <code>/v3/segments</code></li>
</ul>
<p>Why not a single protocol? OTLP already covers logs and metrics, so there&rsquo;s no point reinventing native endpoints for those. But for tracing, OAP&rsquo;s native <code>SegmentObject</code> maps more cleanly onto SkyWalking&rsquo;s trace model, and <code>sw8</code> header propagation to the backend works without any conversion. So traces go native, everything else goes OTLP, and neither side has to translate.</p>
<p>OTLP defaults to protobuf; JSON is available for debugging. The SDK has zero runtime dependencies.</p>
<h2 id="two-platforms-two-independent-layers-and-dashboards">Two platforms, two independent Layers and dashboards</h2>
<p>Many teams maintain a WeChat mini program and an Alipay mini program against a shared backend. Rather than collapsing them into a single tagged service, the design promotes each platform to its own Layer — <code>WECHAT_MINI_PROGRAM</code> and <code>ALIPAY_MINI_PROGRAM</code> — with its own dashboard set. The SDK tags every signal with a resource attribute <code>miniprogram.platform = wechat | alipay</code> and assigns each platform its own component ID (WeChat = 10002, Alipay = 10003).</p>
<p>On the OAP side, the MAL rule&rsquo;s <code>filter</code> routes data into the right Layer at ingest:</p>
<div class="highlight"><pre tabindex="0"><code class="language-yaml"><span style="display: flex;"><span><span style="color: #0550ae;">metricPrefix</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span>meter_wechat_mp<span style="color: #fff;">
</span></span></span><span style="display: flex;"><span><span style="color: #0550ae;">filter</span><span style="color: #1f2328;">:</span><span style="color: #fff;"> </span><span style="color: #0a3069;">"{ tags -&gt; tags.miniprogram_platform == 'wechat' }"</span><span style="color: #fff;">
</span></span></span></code></pre></div><p>The Alipay rule mirrors this with <code>'alipay'</code>. The two rules are mutually exclusive — no double counting — and produce distinct metric prefixes (<code>meter_wechat_mp_*</code> vs <code>meter_alipay_mp_*</code>) that feed each Layer&rsquo;s dashboards. Even when both platforms use the same <code>service.name</code> (e.g. <code>mini-program-demo</code>), the UI exposes two completely separate entry points.</p>
<h2 id="asymmetric-metric-semantics">Asymmetric metric semantics</h2>
<p>This is the design choice I want to highlight. WeChat&rsquo;s base library exposes <code>PerformanceObserver</code>, which gives you renderer-authoritative timings: app launch, first render, route navigation, script execution, sub-package load — all real measurements. Alipay&rsquo;s base library doesn&rsquo;t offer an equivalent, so the SDK falls back to lifecycle hooks: the <code>App.onLaunch → App.onShow</code> delta is used as an approximation of launch time, and renderer-level timings simply aren&rsquo;t available.</p>
<p>So the two MAL rule sets are deliberately not the same:</p>
<ul>
<li><strong>WeChat</strong>: <code>app_launch_duration</code>, <code>first_render_duration</code>, <code>route_duration</code>, <code>script_duration</code>, <code>package_load_duration</code>, <code>request_duration_percentile</code>, <code>request_cpm</code></li>
<li><strong>Alipay</strong>: <code>app_launch_duration</code>, <code>first_render_duration</code>, <code>request_duration_percentile</code>, <code>request_cpm</code></li>
</ul>
<p>The Alipay <code>app_launch_duration</code> is a lifecycle approximation and is not directly comparable to WeChat&rsquo;s renderer timing — the dashboard tooltip says so explicitly. Putting the two numbers side by side is comparing two different measurement definitions.</p>
<h2 id="what-the-sdk-does">What the SDK does</h2>
<p>Four signals:</p>
<ul>
<li><strong>Errors</strong> — JS exceptions, unhandled promise rejections, and <code>pageNotFound</code> go out as OTLP logs, following the OTel <code>exception.*</code> semantic conventions (<code>exception.type</code>, <code>exception.stacktrace</code>). Anything downstream that speaks OTLP — SkyWalking, OTel Collector, Grafana — recognizes them.</li>
<li><strong>Performance</strong> — the metrics listed above. OTLP gauge.</li>
<li><strong>Requests</strong> — <code>wx.request</code> / <code>my.request</code> / <code>downloadFile</code> / <code>uploadFile</code> are reported as OTLP delta histograms, one batch per <code>flushInterval</code> (default 5s). The <code>le</code> bucket labels are already in milliseconds, and the MAL rule explicitly declares <code>MILLISECONDS</code> to disable the default SECONDS→MS rescale. Failed requests (4xx / 5xx / timeout) additionally emit an error log so you can pivot from a dashboard to a concrete failure.</li>
<li><strong>Tracing (opt-in)</strong> — when enabled, outbound requests get <code>sw8</code> header injection, and the resulting segments stitch together with backend traces into one end-to-end view. Trace data goes out as SkyWalking <code>SegmentObject</code>, not OTLP traces.</li>
</ul>
<p>Two reliability and cardinality details worth calling out:</p>
<p><strong>Persisting events on app hide.</strong> Mini programs get killed by the framework after some time in background, and weak networks make in-flight events easy to lose. The SDK writes unsent events to <code>wx.setStorage</code> / <code>my.setStorage</code> on <code>onAppHide</code> and restores them on the next launch.</p>
<p><strong>Avoiding cardinality explosions.</strong> Set <code>serviceInstance</code> to the app version (e.g. <code>1.4.2</code>), not a device ID — at a million DAU the device-ID dimension blows up the OAP instance index. For request paths, the SDK exposes <code>urlGroupRules</code> regex patterns to fold parameterized URLs like <code>/api/user/12345</code> into <code>/api/user/{id}</code> so the endpoint dimension doesn&rsquo;t blow up either.</p>
<h2 id="what-oap-needs">What OAP needs</h2>
<p>If you&rsquo;re on <code>main</code> or a release ≥ 10.5.0, the following are already shipped:</p>
<ul>
<li><code>config/component-libraries.yml</code> registers <code>WeChat-MiniProgram: 10002</code> and <code>AliPay-MiniProgram: 10003</code></li>
<li><code>config/otel-rules/miniprogram/</code> holds four MAL rules — service-scoped and instance-scoped for each platform</li>
<li><code>config/ui-initialized-templates/wechat_mini_program/</code> and <code>alipay_mini_program/</code> carry root / service / instance / endpoint dashboards</li>
<li><code>config/ui-initialized-templates/menu.yaml</code> registers both layers under the Mobile menu group</li>
</ul>
<p>The only thing left is enabling the OTel receiver and giving the SDK an OTLP HTTP port it can reach. SkyWalking OAP binds its OTLP HTTP handler onto the receiver-sharing-server port, and that port defaults to <code>0</code> — meaning it&rsquo;s folded into the core REST port (12800). If you want the SDK to use the standard OTLP HTTP port 4318, set the sharing port to 4318:</p>
<div class="highlight"><pre tabindex="0"><code class="language-bash"><span style="display: flex;"><span>docker run -d --name sw-oap <span style="color: #0a3069;">\
</span></span></span><span style="display: flex;"><span>  -p 11800:11800 -p 12800:12800 -p 4318:4318 <span style="color: #0a3069;">\
</span></span></span><span style="display: flex;"><span>  -e <span style="color: #953800;">SW_STORAGE</span><span style="color: #0550ae;">=</span>banyandb <span style="color: #0a3069;">\
</span></span></span><span style="display: flex;"><span>  -e <span style="color: #953800;">SW_STORAGE_BANYANDB_TARGETS</span><span style="color: #0550ae;">=</span>banyandb:17912 <span style="color: #0a3069;">\
</span></span></span><span style="display: flex;"><span>  -e <span style="color: #953800;">SW_OTEL_RECEIVER</span><span style="color: #0550ae;">=</span>default <span style="color: #0a3069;">\
</span></span></span><span style="display: flex;"><span>  -e <span style="color: #953800;">SW_RECEIVER_SHARING_REST_PORT</span><span style="color: #0550ae;">=</span><span style="color: #0550ae;">4318</span> <span style="color: #0a3069;">\
</span></span></span><span style="display: flex;"><span>  apache/skywalking-oap-server:latest
</span></span></code></pre></div><p>All receivers (OTLP, native segment, browser perf, log report) move to 4318 together, while GraphQL stays on 12800 for the UI.</p>
<p>Minimal SDK config:</p>
<div class="highlight"><pre tabindex="0"><code class="language-js"><span style="display: flex;"><span><span style="color: #cf222e;">import</span> <span style="color: #1f2328;">MiniProgramMonitor</span> <span style="color: #1f2328;">from</span> <span style="color: #0a3069;">'mini-program-monitor'</span><span style="color: #1f2328;">;</span>
</span></span><span style="display: flex;"><span>
</span></span><span style="display: flex;"><span><span style="color: #1f2328;">MiniProgramMonitor</span><span style="color: #1f2328;">.</span><span style="color: #1f2328;">init</span><span style="color: #1f2328;">({</span>
</span></span><span style="display: flex;"><span>  <span style="color: #1f2328;">service</span><span style="color: #0550ae;">:</span> <span style="color: #0a3069;">'mini-program-demo'</span><span style="color: #1f2328;">,</span>
</span></span><span style="display: flex;"><span>  <span style="color: #1f2328;">serviceInstance</span><span style="color: #0550ae;">:</span> <span style="color: #0a3069;">'1.4.2'</span><span style="color: #1f2328;">,</span>          <span style="color: #57606a;">// Recommended: app version
</span></span></span><span style="display: flex;"><span>  <span style="color: #1f2328;">collector</span><span style="color: #0550ae;">:</span> <span style="color: #0a3069;">'http://your-oap:4318'</span><span style="color: #1f2328;">,</span>
</span></span><span style="display: flex;"><span>  <span style="color: #1f2328;">enable</span><span style="color: #0550ae;">:</span> <span style="color: #1f2328;">{</span>
</span></span><span style="display: flex;"><span>    <span style="color: #1f2328;">error</span><span style="color: #0550ae;">:</span> <span style="color: #cf222e;">true</span><span style="color: #1f2328;">,</span>
</span></span><span style="display: flex;"><span>    <span style="color: #1f2328;">perf</span><span style="color: #0550ae;">:</span> <span style="color: #cf222e;">true</span><span style="color: #1f2328;">,</span>
</span></span><span style="display: flex;"><span>    <span style="color: #1f2328;">request</span><span style="color: #0550ae;">:</span> <span style="color: #cf222e;">true</span><span style="color: #1f2328;">,</span>
</span></span><span style="display: flex;"><span>    <span style="color: #1f2328;">tracing</span><span style="color: #0550ae;">:</span> <span style="color: #cf222e;">false</span><span style="color: #1f2328;">,</span>                  <span style="color: #57606a;">// Off by default; enable as needed
</span></span></span><span style="display: flex;"><span>  <span style="color: #1f2328;">},</span>
</span></span><span style="display: flex;"><span><span style="color: #1f2328;">});</span>
</span></span></code></pre></div><p>WeChat and Alipay use the same config — the SDK detects the platform at runtime and tags the data accordingly.</p>
<h2 id="compatibility">Compatibility</h2>
<ul>
<li>WeChat base library ≥ 2.11</li>
<li>Alipay base library ≥ 2.0</li>
<li>Apache SkyWalking OAP <code>main</code> or ≥ 10.5.0, with the OTLP HTTP receiver enabled</li>
<li>Any other OTLP-compatible backend (OpenTelemetry Collector, Grafana, etc.) also works, but you won&rsquo;t get the SkyWalking-specific cross-platform dashboards</li>
</ul>
<h2 id="whats-next">What&rsquo;s next</h2>
<p>To get involved, head over to <a href="https://github.com/SkyAPM/mini-program-monitor">SkyAPM/mini-program-monitor</a> and open an issue or PR. The repo also ships a <code>make preview</code> target that boots OAP, the UI, and both platform simulators locally — handy if you want to play with it end-to-end.</p>
<p>Android end-user experience monitoring is still a gap in the SkyWalking ecosystem; contributors interested in closing that one are very welcome.</p>
