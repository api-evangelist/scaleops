---
title: "HPA’s Three Architectural Flaws (And Why Your Autoscaling Keeps Failing)"
url: "https://scaleops.com/blog/hpas-three-architectural-flaws-and-why-your-autoscaling-keeps-failing/"
date: "Tue, 03 Mar 2026 16:25:56 +0000"
author: "Nic Vermandé"
feed_url: "https://scaleops.com/feed/"
---
<section class="afb-take-aways wp-block-take-aways">
  <div class="afb-take-aways__inner">

<h4 class="wp-block-heading has-text-color has-large-font-size" style="color: #5551FF; font-style: normal; font-weight: 600;">Main takeaways</h4>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>Kubernetes HPA scales based on CPU or memory utilization after traffic spikes, not before. This reactive model causes latency spikes during predictable traffic patterns (daily peaks, jobs, etc.). </li>



<li>HPA and VPA conflict when resource requests change, creating unstable replica counts and the Kubernetes “death spiral”. </li>



<li>ScaleOps eliminates this instability with coordinated rightsizing and proactive replica optimization that pre-warms capacity and stabilizes scaling behavior</li>
</ul>
</div>
</div>
</section>



<h2 class="wp-block-heading">The Promise vs. Reality of HPA</h2>



<p>HPA is the most deployed autoscaler in Kubernetes. It&#8217;s also architecturally limited in ways that matter for production workloads.</p>



<p>The design is straightforward: HPA monitors resource utilization, compares it against a target threshold, and adjusts replica counts accordingly. Set <code>averageUtilization: 70</code>, and HPA scales your deployment when CPU usage crosses that line.</p>



<pre class="wp-block-code"><code>apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70</code></pre>



<p>This works well for static workloads with predictable traffic and unchanging resource requests. The problems emerge when you move beyond that baseline, which every production cluster eventually does.</p>



<p>HPA has three architectural limitations that compound in production environments:</p>



<p><strong>First, HPA is purely reactive.</strong> </p>



<p>It scales based on observed symptoms (elevated CPU) rather than anticipated demand. Traffic arrives, CPU rises, metrics-server collects (up to 60 seconds later), HPA detects the threshold breach, then initiates scaling. By the time new pods pass readiness probes, users have already experienced degraded latency. Predictable patterns — Monday morning spikes, lunch rush traffic, end-of-month processing — trigger the same reactive scramble every time, even though the pattern is entirely foreseeable from historical data.</p>



<p><strong>Second, HPA&#8217;s percentage math breaks when requests change.</strong> </p>



<p>The <code>averageUtilization</code> target calculates against <code>resources.requests.cpu</code>, not actual node capacity. When you <a href="https://scaleops.com/blog/kubernetes-workload-rightsizing/">rightsize</a>, whether manually, via VPA, or through optimization tooling, the denominator in HPA&#8217;s calculation shifts. A pod using 150m CPU with 200m requests shows 75% utilization. Drop requests to 100m (a reasonable optimization), and the same 150m usage becomes 150% utilization. HPA interprets this as an emergency and scales out aggressively, even though actual resource consumption hasn&#8217;t changed.</p>



<p><strong>Third, VPA&#8217;s histograms get polluted when HPA scales.</strong> </p>



<p>VPA maintains <a href="https://scaleops.com/blog/why-pod-rightsizing-fails-in-production-a-deep-dive-into-vpa-and-what-actually-works/">per-container usage histograms</a> to inform rightsizing recommendations. When HPA adds replicas, load spreads across more pods, and per-pod utilization drops. VPA&#8217;s histogram sees &#8220;lower usage&#8221; and recommends smaller requests. Smaller requests trigger the percentage math problem. The loop accelerates: this is the documented VPA/HPA &#8220;death spiral&#8221; that Kubernetes upstream explicitly warns against.</p>



<p>These three issues don&#8217;t exist in isolation. They feed each other: reactive scaling amplifies the percentage math problem during traffic spikes, histogram pollution degrades recommendations over time, and the combination creates oscillating, unpredictable autoscaler behavior.</p>



<p>Throughout this article, we&#8217;ll use <strong>TaxiMetrics</strong> as our demonstration workload—a representative microservices application that exhibits these scaling pathologies under realistic traffic patterns. We&#8217;ll examine each flaw in detail, then show how ScaleOps&#8217; Replica Optimization addresses all three as a unified system.</p>



<h2 class="wp-block-heading">Flaw #1: HPA is Always Late</h2>



<p>HPA operates on a fundamental architectural constraint: it scales based on observed resource utilization, not anticipated demand. This reactive model means scaling decisions always lag behind the events that trigger them.</p>



<h3 class="wp-block-heading">The Metrics Pipeline</h3>



<p>Understanding why HPA is late requires tracing the metrics pipeline:</p>



<figure class="wp-block-image size-large"><img alt="" class="wp-image-8336" height="904" src="https://scaleops.com/content/uploads/2026/03/image-20260223-185904-1024x904.png" width="1024" /></figure>



<p>The cumulative delay from traffic spike to available capacity ranges from 30 seconds (best case: images cached, fast startup) to 3+ minutes (cold nodes, large images, slow readiness probes). During this window, existing pods absorb the full load increase, often resulting in elevated latency, throttling, or dropped requests.</p>



<h3 class="wp-block-heading">Symptom-Based Scaling</h3>



<p>The core issue is that HPA scales on <em>symptoms</em> rather than <em>causes</em>:</p>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>Signal Type</td><td>Example</td><td>When HPA Sees It</td></tr><tr><td>Cause</td><td>Traffic increases 3x</td><td>Never (HPA doesn&#8217;t monitor traffic)</td></tr><tr><td>Symptom</td><td>CPU utilization hits 85%<br /></td><td>30-60 seconds after traffic spike</td></tr></tbody></table></figure>
</div>


<p>By the time CPU rises enough to trigger scaling, the traffic spike has already impacted user experience.</p>



<h3 class="wp-block-heading" id="Predictable-Patterns,-Repeated-Panic">Predictable Patterns, Repeated Panic</h3>



<p>Most production traffic patterns are predictable. Business applications exhibit clear seasonality:</p>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>Pattern</td><td>Frequency</td><td>Data Available</td></tr><tr><td>Monday morning spike</td><td>Weekly</td><td>Months of historical data</td></tr><tr><td>Lunch rush</td><td>Daily</td><td>Repeats every 24 hurs</td></tr><tr><td>End-of-month processing</td><td>Monthly</td><td>Predictable to the day</td></tr><tr><td>Seasonal peaks (Black Friday, Holidays, etc.) </td><td>Annually</td><td>Years of historical data</td></tr></tbody></table></figure>
</div>


<p>HPA ignores all of this. It has no mechanism to learn from historical patterns or <a href="https://scaleops.com/blog/kubernetes-capacity-planning-pros-cons-best-practices/">pre-warm capacity</a> before predictable demand. Every Monday morning is treated as a novel event, triggering the same reactive scaling, even when the pattern has repeated for years.</p>



<h2 class="wp-block-heading">The Over-Provisioning Tax</h2>



<p>Teams recognize that HPA&#8217;s reactive nature creates reliability risk, which leads them to compensate with one of these mechanisms:</p>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>Strategy</td><td>Trade-off</td></tr><tr><td>Set <code>minReplicas</code> artificially high</td><td>Paying for idle capacity 24/7</td></tr><tr><td>Over-provision resource requests</td><td>Wasted compute, poor bin-packing efficiency</td></tr><tr><td>Disable HPA, use fixed replica counts</td><td>No elasticity, always provisioned for peak</td></tr><tr><td>Accept latency degradation during scaling</td><td>SLA impact, degraded user experience</td></tr></tbody></table></figure>
</div>


<p>Each approach trades cost for reliability (or accepts reliability degradation). None addresses the underlying architectural limitation.</p>



<p>With TaxiMetrics, we observe this pattern consistently: traffic spikes that are entirely predictable from historical data still trigger reactive scaling, with P99 latency spiking during the 30-90 second window before new pods become available.</p>



<h2 class="wp-block-heading">Flaw #2: The Percentage Math Problem</h2>



<p>HPA&#8217;s <code>targetAverageUtilization</code> setting appears straightforward: set 70%, and HPA maintains utilization around that level. The implementation details matter.</p>



<h3 class="wp-block-heading" id="How-HPA-Calculates-Utilization">How HPA Calculates Utilization</h3>



<p>The <code>averageUtilization</code> metric calculates against <code>resources.requests.cpu</code>, not node capacity or container limits:</p>



<p><code>utilization = (current CPU usage) / (CPU requests) × 100</code></p>



<p>This means the denominator in HPA&#8217;s calculation is whatever value exists in your pod&#8217;s resource requests at that moment.</p>



<pre class="wp-block-code"><code>apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: taximetrics-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: taximetrics-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Calculated against requests, not capacity</code></pre>



<h3 class="wp-block-heading">The Math Breakdown</h3>



<p>Consider a pod with stable resource consumption:</p>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>Metric</td><td>Before Rightsizing</td><td>After rightsizing</td></tr><tr><td>CPU Requests</td><td>200m</td><td>100m</td></tr><tr><td>Actual Usage</td><td>150m</td><td>150m (unchanged)</td></tr><tr><td>Utilization</td><td>150 ÷ 200 = <strong>75%</strong></td><td>150 ÷ 100 = <strong>150%</strong></td></tr><tr><td>HPA Response</td><td>Stable (below 80% target)</td><td>Scale out immediately</td></tr></tbody></table></figure>
</div>


<p>The workload hasn&#8217;t changed: the amount of traffic is identical and the actual CPU consumption is the same 150 millicores. But HPA&#8217;s percentage calculation shifted because the denominator changed.</p>



<p>With a 70% target, HPA calculates desired replicas using:</p>



<p><code>desiredReplicas = ceil(currentReplicas × (currentUtilization / targetUtilization))</code></p>



<p>Before rightsizing: <code>ceil(2 × (75 / 70))</code> = <code>ceil(2.14)</code> = <strong>3 replicas</strong></p>



<p>After rightsizing: <code>ceil(2 × (150 / 70))</code> = <code>ceil(4.28)</code> = <strong>5 replicas</strong></p>



<p>Same workload, same traffic, but HPA now wants 5 replicas instead of 3.</p>



<h3 class="wp-block-heading">When This Happens</h3>



<p>This behavior triggers whenever resource requests change:</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li><strong>VPA in Auto mode</strong>: <a href="https://scaleops.com/blog/kubernetes-vpa/">VPA</a> applies new requests, pods restart, HPA recalculates against new denominator</li>



<li><strong>Manual rightsizing</strong>: Engineering adjusts requests based on observed usage</li>



<li><strong>Optimization tooling</strong>: Any system that modifies <code>resources.requests</code></li>
</ul>
</div>


<p>The severity depends on how significantly requests change. A workload over-provisioned at 500m requests but using 150m would show 30% utilization. Rightsizing to 200m requests jumps utilization to 75%. Rightsizing further to 100m jumps to 150%.</p>



<h3 class="wp-block-heading" id="The-Documented-Anti-Pattern">The Documented Anti-Pattern</h3>



<p>Kubernetes documentation explicitly warns against running VPA and HPA on the same resource metric. The controllers lack coordination because VPA modifies requests based on historical usage, while HPA scales based on current utilization percentage. When VPA changes requests, HPA&#8217;s math changes underneath it.</p>



<p>The standard workaround is to choose one:</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>Use HPA for horizontal scaling, accept inaccurate resource requests</li>



<li>Use VPA for rightsizing, disable HPA or use custom metrics</li>
</ul>
</div>


<p>Neither option provides both accurate resource requests and stable horizontal scaling.</p>



<h2 class="wp-block-heading" id="Flaw-#3:-The-Histogram-Aggregation-Problem">Flaw #3: The Histogram Aggregation Problem</h2>



<p>The previous two flaws (reactive scaling and percentage math) create problems independently. This third flaw compounds them into a feedback loop.</p>



<h3 class="wp-block-heading" id="How-VPA-Builds-Recommendations">How VPA Builds Recommendations</h3>



<p>VPA uses historical usage data to recommend resource requests. For CPU, it maintains a decaying histogram of usage samples per container. The recommendation algorithm analyzes this histogram to suggest requests that would satisfy a target percentile (typically P90 or P95) of observed usage.</p>



<p>The key architectural detail: VPA collects samples per container, not per workload. Each pod&#8217;s container contributes individual data points to the histogram.</p>



<h3 class="wp-block-heading" id="The-Pollution-Mechanism">The Pollution Mechanism</h3>



<p>When HPA scales a deployment from 2 to 4 replicas, the total workload remains constant but distributes across more pods:</p>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>State</td><td>Replicas</td><td>Total Load</td><td>Per-Pod Load</td></tr><tr><td>Before scale-out</td><td>2</td><td>300m CPU</td><td>150m each</td></tr><tr><td>After scale-out</td><td>4</td><td>300m CPU</td><td>75m each</td></tr></tbody></table></figure>
</div>


<p>VPA&#8217;s histogram now receives samples showing 75m usage per container instead of 150m. Over time, these lower samples shift the histogram distribution downward.</p>



<p>VPA&#8217;s recommendation logic sees &#8220;historical usage is lower&#8221; and recommends reduced requests, even though the workload&#8217;s total resource consumption hasn&#8217;t changed.</p>



<h2 class="wp-block-heading" id="The-Feedback-Loop">The Feedback Loop</h2>



<p>This histogram pollution connects directly to Flaw #2 (percentage math):</p>



<figure class="wp-block-image size-large"><img alt="" class="wp-image-8341" height="1024" src="https://scaleops.com/content/uploads/2026/03/image-20260223-191801-1018x1024.png" width="1018" /></figure>



<p>Each iteration accelerates the next. VPA continuously recommends smaller requests because the histogram data is polluted by HPA&#8217;s scaling decisions. HPA continuously scales out because VPA&#8217;s request changes break the percentage math.</p>



<h3 class="wp-block-heading" id="The-Inverse-Problem">The Inverse Problem</h3>



<p>The loop also operates in reverse during scale-down:</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>Traffic decreases, HPA scales from 4 → 2 replicas</li>



<li>Load concentrates: per-pod usage doubles</li>



<li>VPA histogram sees higher samples, recommends larger requests</li>



<li>Larger requests reduce utilization percentage</li>



<li>HPA sees low utilization, scales down further</li>



<li>Oscillation between over-provisioned and under-provisioned states</li>
</ul>
</div>


<h3 class="wp-block-heading" id="Why-This-Is-Documented-as-an-Anti-Pattern">Why This Is Documented as an Anti-Pattern</h3>



<p>Kubernetes VPA documentation explicitly states that running VPA and HPA on the same CPU or memory metric should not be used. The controllers operate independently with no coordination mechanism:</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>VPA modifies requests based on per-container historical data</li>



<li>HPA modifies replica count based on current aggregate utilization</li>



<li>Neither is aware of the other&#8217;s actions or their downstream effects</li>
</ul>
</div>


<p>The standard guidance is to use HPA with custom or external metrics (queue depth, requests per second) while VPA manages resource requests. This separation prevents the feedback loop but requires additional metrics infrastructure.</p>



<h2 class="wp-block-heading" id="ScaleOps-Replica-Optimization:-The-Unified-Solution">ScaleOps Replica Optimization: The Unified Solution</h2>



<p>The three flaws described above share a root cause: <a href="https://scaleops.com/blog/hpa-vs-vpa-understanding-kubernetes-autoscaling-and-why-its-not-enough-in-2025/">HPA and VPA</a> operate independently with no coordination mechanism. ScaleOps addresses this through two integrated capabilities: <strong>Rightsizing</strong> (continuous resource request optimization) and <strong>Replica Optimization</strong> (horizontal scaling).</p>



<h2 class="wp-block-heading" id="Flaw-#1-Solved:-Proactive-Scaling">Flaw #1 Solved: Proactive Scaling</h2>



<p>Standard HPA reacts to observed metrics. ScaleOps Replica Optimization takes a different approach: it replaces the static, manually-set <code>minReplicas</code> with a data-driven, continuously-updated value — lower when you&#8217;re over-provisioned, higher when a spike is coming.</p>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>Scenario</td><td>Vanilla HPA</td><td>With Replica Optimization</td></tr><tr><td>Monday morning spike</td><td>Scales after CPU rises, 30s-3min delay</td><td><code>minReplicas</code> pre-warmed before spike</td></tr><tr><td>Daily lunch rush</td><td>Same reactive pattern every day</td><td>Pattern detected, capacity ready</td></tr><tr><td>End-of-month processing</td><td>Treats predictable spike as surprise</td><td>Historical baseline informs scaling floor</td></tr></tbody></table></figure>
</div>


<p>When seasonality patterns are detected, Replica Optimization adjusts <code>minReplicas</code> before traffic arrives. Pods are running and ready when users show up.</p>



<figure class="wp-block-image size-large"><img alt="" class="wp-image-8342" height="578" src="https://scaleops.com/content/uploads/2026/03/image-20260223-200826-1024x578.png" width="1024" /></figure>



<p>Replicas remain stable after Rightsizing event:</p>



<figure class="wp-block-image size-large"><img alt="" class="wp-image-8343" height="445" src="https://scaleops.com/content/uploads/2026/03/image-20260224-221713-1024x445.png" width="1024" /></figure>



<h2 class="wp-block-heading" id="Flaw-#2-Solved:-Stable-Scaling-Through-Rightsizing">Flaw #2 Solved: Stable Scaling Through Rightsizing</h2>



<p>When ScaleOps Rightsizing adjusts resource requests, Replica Optimization maintains consistent scaling behavior.</p>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>Scenario</td><td>Vanilla HPA</td><td>With Replica Optimization</td></tr><tr><td>Rightsizing reduces requests 200m → 100m</td><td>Utilization spikes to 150%, panic scaling</td><td>Scaling behavior unchanged</td></tr><tr><td>Manual request tuning</td><td>Unpredictable replica fluctuations</td><td>Consistent scaling response</td></tr><tr><td>Continuous optimization</td><td>Restart storms, oscillation</td><td>Requests and replicas managed in coordination</td></tr></tbody></table></figure>
</div>


<p>The scaling intent — &#8220;scale when the workload needs more capacity&#8221; — remains stable regardless of what values exist in <code>resources.requests</code>. Request changes don&#8217;t trigger spurious scaling events.</p>



<h2 class="wp-block-heading" id="Flaw-#3-Solved:-Accurate-Recommendations-Despite-Scaling">Flaw #3 Solved: Accurate Recommendations Despite Scaling</h2>



<p>ScaleOps Rightsizing analyzes resource consumption at the workload level, not per-container.</p>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>Scenario</td><td>Vanilla VPA</td><td>With ScaleOps Rightsizing</td></tr><tr><td>HPA scales 2 → 8 replicas</td><td>Histogram sees 75% less per-pod usage, recommends smaller requests</td><td>Recommendations stay stable</td></tr><tr><td>Traffic spike + scale out</td><td>Histogram skewed by temporary pod distribution</td><td>Workload profile stays accurate</td></tr><tr><td>Scale down after peak</td><td>Oscillating recommendations</td><td>Consistent sizing through all phases</td></tr></tbody></table></figure>
</div>


<p>Replica count changes don&#8217;t pollute the data used for rightsizing recommendations. Whether the workload runs on 2 pods or 20, the recommendation reflects actual resource requirements.</p>



<figure class="wp-block-image size-large"><img alt="" class="wp-image-8344" height="360" src="https://scaleops.com/content/uploads/2026/03/Screenshot-2026-02-25-at-00.10.58-1024x360.png" width="1024" /></figure>



<h3 class="wp-block-heading" id="The-Unified-System">The Unified System</h3>



<p>These capabilities work together rather than as independent fixes:</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li><strong><a href="https://scaleops.com/product/automated-pod-rightsizing/">Rightsizing</a></strong> optimizes resource requests based on actual workload behavior</li>



<li><strong><a href="https://scaleops.com/product/replicas-optimization/">Replica Optimization</a></strong> maintains stable horizontal scaling regardless of request values</li>



<li><strong>Seasonality detection</strong> pre-warms capacity before predictable demand</li>
</ul>
</div>


<p>This means resource requests reflect actual usage, horizontal scaling responds to real capacity needs, and predictable traffic patterns don&#8217;t cause repeated reactive scrambles.</p>



<h3 class="wp-block-heading" id="TaxiMetrics:-Before-and-After">TaxiMetrics: Before and After</h3>



<p>Applying ScaleOps to the TaxiMetrics deployment demonstrates the difference:</p>



<p><strong>Before (vanilla HPA + VPA):</strong></p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>Rightsizing event triggers replica spike from 3 → 7</li>



<li>VPA recommendations oscillate as HPA scales</li>



<li>Scheduled batch causes 45-second latency degradation while pods start</li>
</ul>
</div>


<p><strong>After (ScaleOps Rightsizing + Replica Optimization):</strong></p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>Rightsizing event: replica count unchanged</li>



<li>Recommendations stable through scaling events</li>



<li>capacity pre-warmed for scheduled batch, no latency impact</li>
</ul>
</div>


<p>Comparison of replicas, CPU requests and latency before and after ScaleOps:</p>



<figure class="wp-block-image size-large"><img alt="" class="wp-image-8345" height="371" src="https://scaleops.com/content/uploads/2026/03/image-20260226-104238-1024x371.png" width="1024" /></figure>



<p>Predictive Replica Optimization with ScaleOps:</p>



<figure class="wp-block-image size-large"><img alt="" class="wp-image-8346" height="526" src="https://scaleops.com/content/uploads/2026/03/image-20260225-105316-1024x526.png" width="1024" /></figure>



<h2 class="wp-block-heading" id="Practical-Implementation">Practical Implementation</h2>



<h3 class="wp-block-heading" id="Migration-Path">Migration Path</h3>



<p>ScaleOps is production-grade and scale-ready from day one. The typical adoption path:</p>



<p><strong>Phase 1: Read-Only Mode</strong></p>



<p>Enable both Rightsizing and Replica Optimization in read-only mode. No changes are applied to workloads: ScaleOps observes, analyzes, and generates optimization opportunities.</p>



<p>This phase reveals the gaps in current autoscaling (vertical and horizontal) behavior:</p>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>Metric</td><td>What You&#8217;ll See</td></tr><tr><td>Recommended vs. actual requests</td><td>How far current requests are from optimal</td></tr><tr><td>Recommended vs. actual minReplicas</td><td>The seasonality gap — where Replica Optimization would pre-warm capacity</td></tr><tr><td>Predicted scaling events</td><td>When ScaleOps would have adjusted before traffic arrived</td></tr></tbody></table></figure>
</div>


<p>The seasonality gap is particularly valuable: you can observe the time difference between when Replica Optimization <em>would</em> adjust minReplicas versus when your current HPA <em>actually</em> reacts. In automated mode, these align. But in read-only mode, the gap quantifies the latency impact you&#8217;re currently absorbing.</p>



<p><strong>Phase 2: Automate</strong></p>



<p>Once you&#8217;ve validated the recommendations match observed workload behavior, enable automation with one click. Rightsizing and Replica Optimization begin applying changes, and the gaps close.</p>



<p>No phased rollout required. No canary deployments of autoscaler configurations. The same system that generated accurate recommendations in read-only mode now applies them.</p>



<h2 class="wp-block-heading" id="What's-Next:-The-Metrics-Latency-Problem">What&#8217;s Next: The Metrics Latency Problem</h2>



<p>ScaleOps Replica Optimization addresses the death spiral: stable scaling through rightsizing, accurate recommendations despite replica changes, and proactive capacity management for predictable patterns.</p>



<p>But HPA has another architectural limitation that exists regardless of how you configure it: metrics latency.</p>



<h3 class="wp-block-heading" id="The-Metrics-Pipeline.1">The Metrics Pipeline</h3>



<p>HPA relies on metrics-server for resource utilization data. The pipeline introduces cumulative delay:</p>



<figure class="wp-block-image size-large"><img alt="" class="wp-image-8347" height="677" src="https://scaleops.com/content/uploads/2026/03/image-20260224-202414-1024x677.png" width="1024" /></figure>



<h2 class="wp-block-heading" id="The-Limitation">The Limitation</h2>



<p>For workloads where CPU and memory are accurate proxies for load, this pipeline works. For workloads where they aren&#8217;t (queue processors, latency-sensitive APIs, batch jobs with variable resource profiles) HPA scales on lagging indicators that don&#8217;t reflect actual capacity needs.</p>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>Workload Type</td><td>Useful Scaling Metric</td><td>Available via metrics-server</td></tr><tr><td>API service</td><td>Request latency, error rate</td><td>No</td></tr><tr><td>Queue processor</td><td>Queue depth, processing rate</td><td>No</td></tr><tr><td>CPU-bound batch</td><td>CPU throttling ratio</td><td>No</td></tr><tr><td>Memory-sensitive</td><td>Memory pressure, eviction signals</td><td>No</td></tr></tbody></table></figure>
</div>


<h2 class="wp-block-heading" id="KEDA:-A-Different-Approach">KEDA: A Different Approach</h2>



<p><a href="https://scaleops.com/blog/unlocking-the-power-of-kubernetes-autoscaling-navigating-scaleops-hpa-vpa-keda-and-cluster-autoscaler/">KEDA</a> (Kubernetes Event-Driven Autoscaler) addresses this by querying metrics sources directly — Prometheus, cloud provider APIs, message queues — with configurable polling intervals as low as 15 seconds.</p>



<pre class="wp-block-code"><code># KEDA ScaledObject example
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: taximetrics-api
spec:
  scaleTargetRef:
    name: taximetrics-api
  pollingInterval: 15
  triggers:
  - type: prometheus
    metadata:
      query: sum(rate(http_requests_total{app="taximetrics"}&#91;1m]))
      threshold: "100"</code></pre>



<p>No metrics-server in the path. No 60-second staleness. Scale on the metrics that actually indicate whether your application needs more capacity.</p>



<h3 class="wp-block-heading" id="The-Metrics-Question">The Metrics Question</h3>



<p>This points to a broader architectural question that the Kubernetes community is still working through: what should autoscaling actually respond to?</p>



<p>CPU and memory are lagging indicators. By the time CPU spikes, the queue is already backing up. By the time memory pressure appears, the OOM killer is already circling. You&#8217;re scaling based on symptoms, not causes — driving by looking in the rearview mirror.</p>



<p>Leading indicators tell a different story:</p>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>Lagging (HPA default)</td><td>Leading (requires custom metrics)</td></tr><tr><td>CPU utilization %</td><td>CPU throttling ratio</td></tr><tr><td>Memory usage</td><td>Memory pressure, eviction signals</td></tr><tr><td>Queue depth</td><td>Queue depth + growth rate</td></tr><tr><td>Average latency</td><td>P99 latency, error rate</td></tr></tbody></table></figure>
</div>


<p>The metrics-server design is a deliberate trade-off. Kubernetes chose simplicity and universal compatibility over metric richness. Every cluster has CPU and memory. Not every cluster has Prometheus, or application-level instrumentation, or the operational maturity to define meaningful custom metrics.</p>



<p>But if you&#8217;re running production workloads at scale, you&#8217;ve probably already crossed that threshold. The question becomes: are you using metrics that actually predict capacity needs, or just reacting to resource exhaustion?</p>



<p>KEDA opens the door to leading metrics. <a href="https://scaleops.com/blog/introducing-automated-replica-optimization-now-ga/">ScaleOps Replica Optimization</a> adds the intelligence layer with pattern detection, seasonality, and proactive scaling. But the fundamental shift is the same: moving from &#8220;scale when it hurts&#8221; to &#8220;scale before it matters.&#8221;</p>



<h2 class="wp-block-heading" id="Next-in-This-Series">Next in This Series</h2>



<p>This article covered HPA&#8217;s architectural limitations when combined with rightsizing — reactive scaling, percentage math instability, and histogram pollution — and how ScaleOps Rightsizing and Replica Optimization address them as a unified system.</p>



<p>But for workloads where CPU and memory aren&#8217;t accurate proxies for capacity needs, there&#8217;s a deeper question: should you be using HPA at all?</p>



<p>The next article explores KEDA in depth:</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li><strong>KEDA vs HPA + Prometheus Adapter</strong> — architecture, complexity, and failure modes</li>



<li><strong>When to use which</strong> — decision framework by workload type</li>



<li><strong>How ScaleOps integrates with event-driven scaling</strong> — combining KEDA with intelligent rightsizing</li>
</ul>
</div>


<p>If your scaling decisions depend on queue depth, request latency, or custom application metrics, that&#8217;s the one to read.</p>



<p><strong>Ready to see ScaleOps in action?</strong> Experience how Rightsizing and Replica Optimization eliminate the HPA/VPA death spiral — with read-only mode to validate before you automate.</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li><a href="#book-a-demo">Book a demo with an expert</a></li>



<li><a href="https://try.scaleops.com/">Start your free trial now</a></li>
</ul>
</div>

<div class="b-faq-section has-inner-container align wp-block-faq-section">
  <div class="b-faq-section__items flex flex-col gap-4">
    <div class="faq-section-innerblocks">

<h2 class="wp-block-heading">FAQ: Kubernetes HPA, VPA, and Autoscaling Stability</h2>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading">Why is Kubernetes HPA considered reactive?</h3>



<p>HPA scales based on observed CPU or memory utilization after traffic increases. Because it relies on metrics-server updates and pod readiness time, scaling decisions are delayed, which can cause temporary performance degradation during traffic spikes.</p>

</div>
</div>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading">Why do HPA and VPA conflict in Kubernetes?</h3>



<p>HPA calculates scaling decisions using a percentage of CPU or memory requests. When VPA adjusts those requests, the utilization percentage changes instantly—even if workload demand does not. This can trigger unnecessary replica scaling and instability.</p>

</div>
</div>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading">What is the Kubernetes HPA/VPA “death spiral”?</h3>



<p>The death spiral occurs when HPA scales replicas and VPA adjusts resource requests independently. Replica changes distort per-pod metrics, leading VPA to recommend smaller requests, which in turn causes HPA to scale further. This creates oscillating and unstable scaling behavior.</p>

</div>
</div>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading">How can vertical and horizontal Kubernetes autoscaling be stabilized?</h3>



<p>Autoscaling stability requires coordination between resource rightsizing and replica scaling. A unified system that analyzes workload-level consumption and adjusts replicas proactively—rather than reacting to CPU spikes—prevents scaling loops and reduces latency.</p>

</div>
</div>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading">When should you use KEDA instead of HPA?</h3>



<p>KEDA is better suited for workloads that require scaling based on leading indicators such as queue depth, request rate, or latency. Unlike HPA’s default CPU-based scaling, KEDA can scale on custom or event-driven metrics with lower latency.</p>

</div>
</div>

</div>
  </div>
</div>

<p>The post <a href="https://scaleops.com/blog/hpas-three-architectural-flaws-and-why-your-autoscaling-keeps-failing/">HPA&#8217;s Three Architectural Flaws (And Why Your Autoscaling Keeps Failing)</a> appeared first on <a href="https://scaleops.com">ScaleOps</a>.</p>
