---
title: "Blog: SkyWalking Ruby Quick Start and Principle Introduction"
url: "/blog/2025-03-06-introduction-to-skywalking-ruby/"
date: "Thu, 06 Mar 2025 00:00:00 +0000"
author: ""
feed_url: "https://skywalking.apache.org/feed"
---
<h2 id="background">Background</h2>
<p>Ruby is a dynamic, object-oriented programming language with concise and elegant syntax, supporting multiple programming
paradigms, including object-oriented, functional, and metaprogramming. Leveraging its powerful metaprogramming
capabilities, Ruby allows modifying the behavior of classes and objects at runtime.
SkyWalking provides a <a href="https://rubygems.org/gems/skywalking">Ruby gem</a> to facilitate integration with Ruby projects, and
this gem supports many out-of-the-box frameworks and gems.</p>
<p>This article is based on skywalking-ruby-v0.1. We will guide you on how to quickly integrate the skywalking-ruby project
into Ruby projects and briefly introduce the implementation principle of SkyWalking Ruby&rsquo;s auto-instrumentation plugins using
redis-rb as an example.</p>
<p>The demonstration includes the following steps:</p>
<ol>
<li><strong>Deploy SkyWalking</strong>: This involves setting up the SkyWalking backend and UI programs to enable you to
see the final results.</li>
<li><strong>Integrate SkyWalking into Different Ruby Projects</strong>: This section explains how to integrate SkyWalking into
different Ruby projects.</li>
<li><strong>Application Deployment</strong>: You will export environment variables and deploy the application to facilitate
communication between your service and the SkyWalking backend.</li>
<li><strong>Visualization on SkyWalking UI</strong>: Finally, you will send requests and observe the results in the SkyWalking UI.</li>
</ol>
<h2 id="deploy-skywalking">Deploy SkyWalking</h2>
<p>Please download the SkyWalking APM program from the official SkyWalking website,
and then you can start all the required services using the <a href="https://skywalking.apache.org/docs/main/next/en/setup/backend/backend-docker/#start-the-storage-oap-and-booster-ui-with-docker-compose">quick start script</a>.</p>
<p>Next, you can access the address http://localhost:8080/. At this point, since no applications have been deployed, you
will not see any data.</p>
<p>Integrate SkyWalking into Different Ruby Projects
It is recommended to use <a href="https://bundler.io/">Bundler</a> to install and manage SkyWalking dependencies. Simply declare it in the Gemfile and run
bundle install to complete the installation.</p>
<div class="highlight"><pre tabindex="0"><code class="language-bash"><span style="display: flex;"><span><span style="color: #57606a;"># Gemfile</span>
</span></span><span style="display: flex;"><span><span style="color: #6639ba;">source</span> <span style="color: #0a3069;">"https://rubygems.org"</span>
</span></span><span style="display: flex;"><span>
</span></span><span style="display: flex;"><span>gem <span style="color: #0a3069;">"skywalking"</span>
</span></span></code></pre></div><h3 id="integration-in-rails-projects">Integration in Rails Projects</h3>
<p>For Rails projects, it is recommended to use the following command to automatically generate the configuration file:</p>
<div class="highlight"><pre tabindex="0"><code class="language-bash"><span style="display: flex;"><span>bundle <span style="color: #6639ba;">exec</span> rails generate skywalking:start
</span></span></code></pre></div><p>This command will automatically generate a <code>skywalking.rb</code> file in the <code>config/initializers</code> directory, where you can
configure the startup parameters.</p>
<h3 id="integration-in-sinatra-projects">Integration in Sinatra Projects</h3>
<p>For Sinatra projects, you need to manually call <code>Skywalking.start</code> when the application starts. For example:</p>
<div class="highlight"><pre tabindex="0"><code class="language-ruby"><span style="display: flex;"><span><span style="color: #6639ba;">require</span> <span style="color: #0a3069;">'sinatra'</span>
</span></span><span style="display: flex;"><span><span style="color: #6639ba;">require</span> <span style="color: #0a3069;">'skywalking'</span>
</span></span><span style="display: flex;"><span>
</span></span><span style="display: flex;"><span><span style="color: #0550ae;">Skywalking</span><span style="color: #0550ae;">.</span>start
</span></span><span style="display: flex;"><span>
</span></span><span style="display: flex;"><span>get <span style="color: #0a3069;">'/sw'</span> <span style="color: #cf222e;">do</span>
</span></span><span style="display: flex;"><span>  <span style="color: #0a3069;">"Hello SkyWalking!"</span>
</span></span><span style="display: flex;"><span><span style="color: #cf222e;">end</span>
</span></span></code></pre></div><p>In the Gemfile, place skywalking after sinatra and use <code>Bundler.require</code> during initialization, or call
<code>require 'skywalking'</code> after the sinatra gem is loaded. Note that the skywalking gem needs to be placed after
other gems (such as redis, elasticsearch).</p>
<h2 id="application-deployment">Application Deployment</h2>
<p>Before starting the application deployment, you can change the service name of the current application in SkyWalking
through environment variables. You can also modify its configuration, such as the server-side address. For more details,
please refer to the <a href="https://skywalking.apache.org/docs/skywalking-ruby/next/en/setup/quick-start/#configuration">documentation</a>.</p>
<p>Here, we will change the current service name to <code>sw-ruby</code>.</p>
<p>Next, you can start the application. Here is an example using <code>sinatra</code>:</p>
<div class="highlight"><pre tabindex="0"><code class="language-bash"><span style="display: flex;"><span><span style="color: #6639ba;">export</span> <span style="color: #953800;">SW_AGENT_SERVICE_NAME</span><span style="color: #0550ae;">=</span>sw-ruby
</span></span><span style="display: flex;"><span>
</span></span><span style="display: flex;"><span>ruby sinatra.rb
</span></span></code></pre></div><h2 id="visualization-on-skywalking-ui">Visualization on SkyWalking UI</h2>
<p>Now, send requests to the application and observe the results in the SkyWalking UI.</p>
<p>After a few seconds, revisit the SkyWalking UI at http://localhost:8080. You will be able to see the deployed <code>demo</code>
service on the homepage.</p>
<p><img alt="" src="service.png" /></p>
<p>Additionally, on the tracing page, you can see the request you just sent.</p>
<p><img alt="" src="trace.png" /></p>
<h2 id="plugin-implementation-mechanism">Plugin Implementation Mechanism</h2>
<p>To understand the implementation mechanism of Ruby Agent&rsquo;s auto-instrumentation plugins, it is essential to understand the concept
of the ancestor chain in Ruby. The ancestor chain is an ordered list, and in Ruby, each class or module has an ancestor
chain that includes all its parent classes and mixin modules (modules mixed in via include, prepend, or extend).
When Ruby looks up a method, it searches in the order of the ancestor chain until it finds the target method or throws a
<code>NoMethodError</code>.</p>
<div class="highlight"><pre tabindex="0"><code class="language-ruby"><span style="display: flex;"><span><span style="color: #cf222e;">class</span> <span style="color: #1f2328;">User</span>
</span></span><span style="display: flex;"><span><span style="color: #cf222e;">end</span>
</span></span></code></pre></div><p>We have defined a User class, and its ancestor chain is as shown in the following figure:</p>
<p><img alt="" src="p1.png" /></p>
<p>Next, mix in a module using the <code>prepend</code> method:</p>
<div class="highlight"><pre tabindex="0"><code class="language-ruby"><span style="display: flex;"><span><span style="color: #cf222e;">module</span> <span style="color: #24292e;">Dapper</span>
</span></span><span style="display: flex;"><span>  <span style="color: #cf222e;">def</span> <span style="color: #6639ba;">brave</span>
</span></span><span style="display: flex;"><span>    <span style="color: #0a3069;">"Hello from brave"</span>
</span></span><span style="display: flex;"><span>  <span style="color: #cf222e;">end</span>
</span></span><span style="display: flex;"><span><span style="color: #cf222e;">end</span>
</span></span><span style="display: flex;"><span>
</span></span><span style="display: flex;"><span><span style="color: #cf222e;">class</span> <span style="color: #1f2328;">User</span>
</span></span><span style="display: flex;"><span>  prepend <span style="color: #0550ae;">Dapper</span>
</span></span><span style="display: flex;"><span><span style="color: #cf222e;">end</span>
</span></span><span style="display: flex;"><span>
</span></span><span style="display: flex;"><span><span style="color: #6639ba;">p</span> <span style="color: #0550ae;">User</span><span style="color: #0550ae;">.</span>new<span style="color: #0550ae;">.</span>brave <span style="color: #57606a;"># =&gt; "Hello from brave"</span>
</span></span></code></pre></div><p><code>prepend</code> will insert at position 1 in the above figure. Ruby first looks for the brave method in the Dapper module, finds
it, and calls it. If the brave method is not found in Dapper, Ruby continues to search in the User class. If it is not
found in the User class, Ruby continues to search in Object, and so on.</p>
<p>Based on this mechanism, let&rsquo;s briefly introduce how we instrument the <a href="https://github.com/redis-rb/redis-client">redis-rb</a> method.
The following code is the target method to be instrumented:</p>
<div class="highlight"><pre tabindex="0"><code class="language-ruby"><span style="display: flex;"><span><span style="color: #57606a;"># lib/redis/client.rb</span>
</span></span><span style="display: flex;"><span><span style="color: #cf222e;">class</span> <span style="color: #1f2328;">Redis</span>
</span></span><span style="display: flex;"><span>  <span style="color: #cf222e;">class</span> <span style="color: #1f2328;">Client</span> <span style="color: #0550ae;">&lt;</span> <span style="color: #0550ae;">::</span><span style="color: #0550ae;">RedisClient</span>
</span></span><span style="display: flex;"><span>    <span style="color: #cf222e;">def</span> <span style="color: #6639ba;">call_v</span><span style="color: #1f2328;">(</span>command<span style="color: #1f2328;">,</span> <span style="color: #0550ae;">&amp;</span>block<span style="color: #1f2328;">)</span>
</span></span><span style="display: flex;"><span>      <span style="color: #cf222e;">super</span><span style="color: #1f2328;">(</span>command<span style="color: #1f2328;">,</span> <span style="color: #0550ae;">&amp;</span>block<span style="color: #1f2328;">)</span>
</span></span><span style="display: flex;"><span>    <span style="color: #cf222e;">rescue</span> <span style="color: #0550ae;">::</span><span style="color: #0550ae;">RedisClient</span><span style="color: #0550ae;">::</span><span style="color: #0550ae;">Error</span> <span style="color: #0550ae;">=&gt;</span> error
</span></span><span style="display: flex;"><span>      <span style="color: #0550ae;">Client</span><span style="color: #0550ae;">.</span>translate_error!<span style="color: #1f2328;">(</span>error<span style="color: #1f2328;">)</span>
</span></span><span style="display: flex;"><span>    <span style="color: #cf222e;">end</span>
</span></span><span style="display: flex;"><span>  <span style="color: #cf222e;">end</span>
</span></span><span style="display: flex;"><span><span style="color: #cf222e;">end</span>
</span></span></code></pre></div><p>Below is the core code for instrumentation:</p>
<div class="highlight"><pre tabindex="0"><code class="language-ruby"><span style="display: flex;"><span><span style="color: #cf222e;">module</span> <span style="color: #24292e;">Skywalking</span>
</span></span><span style="display: flex;"><span>  <span style="color: #cf222e;">module</span> <span style="color: #24292e;">Plugins</span>
</span></span><span style="display: flex;"><span>    <span style="color: #cf222e;">class</span> <span style="color: #1f2328;">Redis5</span> <span style="color: #0550ae;">&lt;</span> <span style="color: #0550ae;">PluginsManager</span><span style="color: #0550ae;">::</span><span style="color: #0550ae;">SWPlugin</span>
</span></span><span style="display: flex;"><span>      <span style="color: #cf222e;">module</span> <span style="color: #24292e;">Redis5Intercept</span>
</span></span><span style="display: flex;"><span>        <span style="color: #cf222e;">def</span> <span style="color: #6639ba;">call_v</span><span style="color: #1f2328;">(</span>args<span style="color: #1f2328;">,</span> <span style="color: #0550ae;">&amp;</span>block<span style="color: #1f2328;">)</span>
</span></span><span style="display: flex;"><span>          operation <span style="color: #0550ae;">=</span> args<span style="color: #0550ae;">[</span><span style="color: #0550ae;">0</span><span style="color: #0550ae;">]</span> <span style="color: #cf222e;">rescue</span> <span style="color: #0a3069;">"UNKNOWN"</span>
</span></span><span style="display: flex;"><span>          <span style="color: #cf222e;">return</span> <span style="color: #cf222e;">super</span> <span style="color: #cf222e;">if</span> operation <span style="color: #0550ae;">==</span> <span style="color: #032f62;">:auth</span>
</span></span><span style="display: flex;"><span>
</span></span><span style="display: flex;"><span>          <span style="color: #0550ae;">Tracing</span><span style="color: #0550ae;">::</span><span style="color: #0550ae;">ContextManager</span><span style="color: #0550ae;">.</span>new_exit_span<span style="color: #1f2328;">(</span>
</span></span><span style="display: flex;"><span>            <span style="color: #032f62;">operation</span><span style="color: #1f2328;">:</span> <span style="color: #0a3069;">"Redis/</span><span style="color: #0a3069;">#{</span>operation<span style="color: #0550ae;">.</span>upcase<span style="color: #0a3069;">}</span><span style="color: #0a3069;">"</span>
</span></span><span style="display: flex;"><span>          <span style="color: #1f2328;">)</span> <span style="color: #cf222e;">do</span> <span style="color: #0550ae;">|</span>span<span style="color: #0550ae;">|</span>
</span></span><span style="display: flex;"><span>            <span style="color: #57606a;"># Omitted handling of span </span>
</span></span><span style="display: flex;"><span>            <span style="color: #cf222e;">super</span><span style="color: #1f2328;">(</span>args<span style="color: #1f2328;">,</span> <span style="color: #0550ae;">&amp;</span>block<span style="color: #1f2328;">)</span> <span style="color: #57606a;"># Call the original method</span>
</span></span><span style="display: flex;"><span>          <span style="color: #cf222e;">end</span>
</span></span><span style="display: flex;"><span>        <span style="color: #cf222e;">end</span>
</span></span><span style="display: flex;"><span>      <span style="color: #cf222e;">end</span>
</span></span><span style="display: flex;"><span>
</span></span><span style="display: flex;"><span>      <span style="color: #cf222e;">def</span> <span style="color: #6639ba;">install</span>
</span></span><span style="display: flex;"><span>        <span style="color: #0550ae;">::</span><span style="color: #0550ae;">Redis</span><span style="color: #0550ae;">::</span><span style="color: #0550ae;">Client</span><span style="color: #0550ae;">.</span>prepend <span style="color: #0550ae;">Redis5Intercept</span>
</span></span><span style="display: flex;"><span>      <span style="color: #cf222e;">end</span>
</span></span><span style="display: flex;"><span>    <span style="color: #cf222e;">end</span>
</span></span><span style="display: flex;"><span>  <span style="color: #cf222e;">end</span>
</span></span><span style="display: flex;"><span><span style="color: #cf222e;">end</span>
</span></span></code></pre></div><p>Here, we define a Redis5Intercept module and prepend it to <code>::Redis::Client</code>. According to Ruby&rsquo;s method lookup mechanism,
when the <code>call_v</code> method of <code>Redis::Client</code> is called, Ruby will first execute the <code>call_v</code> method in <code>Redis5Intercept</code>. The
order of the ancestor chain is as follows:</p>
<div class="highlight"><pre tabindex="0"><code class="language-markdown"><span style="display: flex;"><span>Redis5Intercept -&gt; Redis::Client -&gt; ... (other parent classes and modules)
</span></span></code></pre></div><p>At the same time, in the <code>call_v</code> method of <code>Redis5Intercept</code>, <code>super(args, &amp;block)</code> will find the next method with the same
name along the ancestor chain, which in this case is the original <code>call_v</code> method in <code>Redis::Client</code>, while passing the
original arguments and block.</p>
<h2 id="conclusion">Conclusion</h2>
<p>This article explained the integration methods of SkyWalking Ruby in Ruby projects and briefly introduced the
implementation mechanism of SkyWalking Ruby&rsquo;s auto-instrumentation plugins.</p>
<p>Currently, the Ruby auto-instrumentation is in the early stages of development. In the future, we will continue to expand the
functionality of SkyWalking Ruby and add support for more plugins. So, stay tuned!</p>
