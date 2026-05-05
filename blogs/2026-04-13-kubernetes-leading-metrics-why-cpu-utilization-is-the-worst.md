---
title: "Kubernetes Leading Metrics: Why CPU Utilization Is the Worst Scaling Signal (And What to Use Instead)"
url: "https://scaleops.com/blog/blog-kubernetes-leading-scaling-metrics/"
date: "Mon, 13 Apr 2026 05:43:00 +0000"
author: "Nic Vermandé"
feed_url: "https://scaleops.com/feed/"
---
<section class="afb-take-aways wp-block-take-aways">
  <div class="afb-take-aways__inner">

<h4 class="wp-block-heading has-text-color has-large-font-size" style="color: #5551FF; font-style: normal; font-weight: 600;">Key Takeaways</h4>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>CPU utilization percentage has a 30-second to 3-minute delay through the Horizontal Pod Autoscaler pipeline — by the time it triggers horizontal scaling, users have already experienced degradation.</li>



<li>CPU throttling ratio provides a kernel-level signal approximately 25 seconds ahead of the Metrics Server pipeline — it is the earliest real-time indicator available for CPU-bound workloads with limits set.</li>



<li>PSI (Pressure Stall Information) measures CPU contention without requiring limits, available as a beta feature since Kubernetes 1.34 via the <code>KubeletPSI</code> feature gate.</li>



<li>In KEDA&#8217;s MAX-across-triggers evaluation model, <code>deriv()</code> is structurally unable to gate horizontal scaling decisions in most configurations — the unit mismatch between messages and messages-per-second means the backlog trigger almost always produces more replicas.</li>



<li><code>predict_linear</code> triggered first horizontal scale-out 2.66 seconds earlier than backlog-only in controlled GKE cluster testing — with no code changes and no custom instrumentation required.</li>



<li>Memory pressure is primarily a vertical scaling signal — adding replicas does not reduce per-pod memory consumption, and by the time memory utilization triggers horizontal scaling, pods are already being OOMKilled.</li>
</ul>
</div>
</div>
</section>



<h2 class="wp-block-heading" id="The-Spectrum-of-Kubernetes-Leading-Metrics">The Spectrum of Kubernetes Leading Metrics</h2>



<p>The <a href="https://scaleops.com/blog/kubectl-autoscale-hpa-a-production-maturity-guide-for-2025/" id="5563" type="post">Horizontal Pod Autoscaler (HPA)</a> shipped in Kubernetes 1.1, back in 2015. At the time, CPU and memory were effectively the only scaling signals available. Metrics Server didn&#8217;t exist yet. The Custom Metrics API wouldn&#8217;t arrive until Kubernetes 1.6, two years later. &#8220;Scale when CPU exceeds 80%&#8221; was a reasonable default because most clusters ran a handful of stateless API servers behind a load balancer.</p>



<p>That world doesn&#8217;t exist anymore.</p>



<p>Today&#8217;s production clusters run fundamentally different workload types side by side: CPU-bound API servers, queue-based workers consuming from NATS or Kafka, memory-heavy caches like Redis, ML inference services with variable GPU utilization, and latency-sensitive gRPC endpoints with strict SLA budgets. Each of these workload types has different failure modes, different pressure signals, and different scaling needs. But most teams are still horizontally scaling on the same metric they configured when they first set up HPA: CPU utilization percentage.</p>



<p>CPU utilization is a lagging metric. It tells you what already happened, not what&#8217;s about to happen. By the time average CPU crosses your threshold, your users have already experienced degraded latency. Instead of scaling proactively, you&#8217;re reacting to symptoms.</p>



<p>The Kubernetes Custom Metrics API that could fix this has been available since 2017. But most teams never made the switch because the configuration is painful, the documentation is scattered across Prometheus Adapter READMEs and half-finished blog posts, and there&#8217;s no clear guidance on which metric to use for which workload.</p>



<p>This article is that guidance — a practical map of Kubernetes leading metrics for every major workload type.</p>



<p>A leading metric in Kubernetes horizontal scaling is a signal that indicates emerging stress before it impacts user experience — as opposed to lagging metrics like CPU utilization that reflect what already happened. The distinction matters because your Horizontal Pod Autoscaler can only be as fast as the signals you give it.</p>



<p>But &#8220;leading&#8221; is itself misleading. It implies all kubernetes leading metrics are equally predictive. They&#8217;re not. The reality is a spectrum of signal latency:</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li><strong>Truly real-time</strong>: kernel-level signals like CPU throttling ratio, which detect contention approximately 25 seconds before Metrics Server reports it through the HPA pipeline</li>



<li><strong>Early-warning</strong>: tail latency indicators like P99 that degrade before averages move — but by the time they shift, some users already had bad requests</li>



<li><strong>Predictive trends</strong>: queue growth rate and memory pressure acceleration, signaling trajectory rather than current state</li>



<li><strong>Historical averages</strong>: CPU and memory utilization — where most teams are stuck</li>
</ul>
</div>


<p> The right metric depends on your workload type. There is no universal best:</p>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table><tbody><tr><td class="has-text-align-left">Workload Type</td><td class="has-text-align-left">Lagging (what mosts teams use)</td><td class="has-text-align-left">Leading (what to use instead)</td><td class="has-text-align-left">Native Scaling Approach</td><td class="has-text-align-left">With ScaleOps</td><td class="has-text-align-left">PromQL Example</td></tr><tr><td class="has-text-align-left">CPU-bound</td><td class="has-text-align-left">CPU utilization %</td><td class="has-text-align-left">Throttling ratio, P99 latency</td><td class="has-text-align-left">HPA + Prometheus Adapter</td><td class="has-text-align-left">Replica Optimization: pre-warms minReplicas from learned patterns; trigger math stays stable through rightsizing</td><td class="has-text-align-left"><code>rate(container_cpu_cfs_throttled_seconds_total...)</code></td></tr><tr><td class="has-text-align-left">Queue-bound</td><td class="has-text-align-left">Queue depth</td><td class="has-text-align-left">Growth rate + productivity (composite)</td><td class="has-text-align-left">KEDA composite trigger</td><td class="has-text-align-left">KEDA integration: replica floor + trigger synchronisation when CPU trigger sits alongside queue triggers</td><td class="has-text-align-left"><code>predict_linear(nats_consumer_num_pending...)</code></td></tr><tr><td class="has-text-align-left">Memory-bound</td><td class="has-text-align-left">Memory utilization %</td><td class="has-text-align-left">Memory pressure, eviction rate</td><td class="has-text-align-left">HPA or VPA (Off mode)</td><td class="has-text-align-left">Rightsizing: sets memory requests from observed working set, prevents OOMKill cliff</td><td class="has-text-align-left"><code>container_memory_working_set_bytes / limit</code></td></tr><tr><td class="has-text-align-left">Latency-bound</td><td class="has-text-align-left">Average latency</td><td class="has-text-align-left">Error rate, timeout ratio, P99</td><td class="has-text-align-left">HPA + Custom Metrics API</td><td class="has-text-align-left">Rightsizing + Replica Optimization: request accuracy reduces baseline latency; replica floor absorbs traffic spikes before P99 degrades</td><td class="has-text-align-left"><code>histogram_quantile(0.99, ...)</code></td></tr></tbody></table></figure>
</div>


<p>And beyond leading metrics, there&#8217;s a predictive layer — scaling based on learned traffic patterns and seasonality rather than reacting to signals at all. That requires getting the metrics right first. We&#8217;ll come back to it at the end.</p>



<p>This is not a &#8220;what is HPA&#8221; tutorial. It assumes you&#8217;ve configured horizontal pod scaling before and are familiar with the <code>autoscaling/v2</code> API. What follows is the metrics layer that sits underneath HPA, KEDA, and VPA — the signals that determine whether your scaling decisions are chasing symptoms or catching problems early.</p>



<h2 class="wp-block-heading">CPU Throttling Ratio: The Kernel Knows Before Your Horizontal Pod Autoscaler</h2>



<p>To understand why CPU utilization percentage is late, you need to trace the full metrics pipeline that feeds horizontal pod scaling decisions.</p>



<p>A traffic spike hits your application:</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>CPU usage rises immediately at the process level.</li>



<li>The kubelet scrapes container metrics every 10 to 15 seconds.</li>



<li>Metrics Server polls the kubelet roughly every 60 seconds.</li>



<li>The Horizontal Pod Autoscaler queries Metrics Server on its default 15-second sync period.</li>



<li>It detects the threshold breach, calculates desired replicas, and the scheduler places new pods.</li>



<li>Those pods pull their image, start, and pass readiness probes.</li>
</ul>
</div>


<p>Total elapsed time from traffic spike to new capacity serving requests: 30 seconds to over 3 minutes.</p>



<p>The Linux kernel, meanwhile, knew the pod was constrained approximately 25 seconds into that chain. It was already throttling processes via CFS bandwidth control. That information was available, but nothing in the horizontal scaling pipeline asked for it.</p>



<p>CFS (Completely Fair Scheduler) enforces CPU limits by allocating a bandwidth quota per scheduling period (typically 100ms). When a container exhausts its quota within a period, the kernel throttles it: the process sleeps until the next period. This throttling is recorded in kernel counters that cAdvisor exposes to Prometheus. That metric is <code>container_cpu_cfs_throttled_seconds_total</code>.</p>



<p>There’s also a simple PromQL query that turns this into a scaling signal:</p>



<pre class="wp-block-code"><code>rate(container_cpu_cfs_throttled_seconds_total&#91;2m])
/ (rate(container_cpu_cfs_throttled_seconds_total&#91;2m])
+ rate(container_cpu_usage_seconds_total&#91;2m]))</code></pre>



<p>This gives you the throttling ratio: the fraction of CPU time your container spent being throttled rather than executing. It&#8217;s a direct kernel-level pressure indicator, not a derived average.</p>



<p>As a starting point, you can use the following thresholds:</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>Below 5% is healthy</li>



<li>Between 5% and 10% is a warning, something is tightening.</li>



<li>Above 10%, scale now.</li>
</ul>
</div>


<p>These are not universal, you should tune them against your latency SLOs. But they&#8217;re a defensible starting point for most CPU-bound services.</p>



<h3 class="wp-block-heading" id="The-limits-controversy-—-and-what-to-do-if-you-don't-set-them">The limits controversy — and what to do if you don&#8217;t set them</h3>



<p>CPU throttling ratio only works if you set CPU limits. The &#8220;remove all limits&#8221; camp — popularized by several prominent voices in the Kubernetes community — argues that limits cause unnecessary throttling and should be eliminated. They have a point about throttling. But they rarely mention what you lose.</p>



<p>If you remove CPU limits, you lose the single best real-time signal the Linux kernel gives you about container pressure. That&#8217;s a tradeoff worth making consciously, as long as you have full awareness of what you&#8217;re giving up — not by default because the internet said limits are evil.</p>



<p>But here&#8217;s what changed: <strong>you no longer have to choose between limits and observability.</strong></p>



<p>Since <a href="https://scaleops.com/blog/kubernetes-1-34-features-explained-faster-safer-and-cheaper-clusters/" id="5528" type="post">Kubernetes 1.34</a>, PSI (Pressure Stall Information) is available as a beta feature, enabled by default via the <code>KubeletPSI</code> feature gate. PSI measures something fundamentally different from CFS throttling. Where throttling ratio tells you &#8220;this container hit its CPU quota and was forced to wait,&#8221; PSI tells you &#8220;this container&#8217;s tasks wanted to run but couldn&#8217;t because the CPU was busy.&#8221; Throttling requires limits, but that’s not the case for PSI.</p>



<p>The metric is <code>container_pressure_cpu_waiting_seconds_total</code>, exposed per-container via the kubelet&#8217;s cAdvisor Prometheus endpoint. The PromQL query:</p>



<pre class="wp-block-code"><code>rate(container_pressure_cpu_waiting_seconds_total{container="your-app"}&#91;2m])</code></pre>



<p>This gives you the fraction of time your container&#8217;s tasks were stalled waiting for CPU — regardless of whether limits are set. It works on any cluster running cgroupv2 with a Linux kernel 4.20 or newer, which at this point is effectively every modern Kubernetes distribution.</p>



<p>The practical model becomes three tiers:</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li><strong>You set CPU limits</strong>: Use throttling ratio. It&#8217;s the most precise signal — CFS-level, kernel-native, measures exactly how much your container is being capped.</li>



<li><strong>You don&#8217;t set CPU limits</strong>: Use PSI. It measures contention without requiring limits. You lose the capping signal but gain a pain signal.</li>



<li><strong>You want the full picture</strong>: Use both. Throttling tells you &#8220;this container is being artificially constrained by its quota.&#8221; PSI tells you &#8220;this container is experiencing real resource contention.&#8221; These are different questions with different operational implications.</li>
</ul>
</div>


<p>ScaleOps ingests both throttling and PSI signals as part of its workload observation — translating kernel-level pressure data into rightsizing and replica decisions without requiring you to configure Prometheus Adapter or custom HPA metrics manually.</p>



<p>One caveat that the PSI documentation doesn&#8217;t make obvious: PSI currently cannot distinguish between pressure caused by genuine resource contention and pressure caused by CPU throttling from limits you explicitly configured. A pod with a 20m CPU limit running a compute-heavy workload will show 99% CPU pressure — technically accurate (the pod is under pressure), but the pressure is self-inflicted by the limit, not by competing neighbors. At the per-container cgroup level this is less of a concern since you&#8217;re looking at the container&#8217;s own experience. But if you aggregate PSI to the node level for scheduling decisions, this conflation becomes a real problem. <a href="https://github.com/kubernetes/enhancements/issues/5062">KEP-4205 in the Kubernetes enhancements repo</a> documents this limitation in detail.</p>



<h3 class="wp-block-heading" id="Gotcha-—-burstable-workloads">Gotcha — burstable workloads</h3>



<p>Throttling ratio has a blind spot for Burstable QoS class workloads with a wide gap between requests and limits. A pod running at 300m CPU with requests set to 100m and limits set to 500m will show 0% throttling — it&#8217;s well within its limit. But it&#8217;s consuming 3x its requested allocation, and if the node gets busy, it&#8217;ll be the first to lose that burst headroom. The throttling ratio says &#8220;everything is fine&#8221; because technically it is — until it isn&#8217;t. For Burstable workloads, combine throttling ratio with actual usage-to-request ratio, or better yet, PSI — which will register contention the moment burst headroom disappears.</p>



<h2 class="wp-block-heading" id="Queue-Growth-Rate:-The-Gatekeeper-That-Doesn't-Drive-KEDA-Scaling">Queue Growth Rate: The Gatekeeper That Doesn&#8217;t Drive KEDA Scaling</h2>



<p>&#8220;If the queue is growing, we should scale.&#8221; </p>



<p>This is one of the most intuitive ideas in horizontal scaling for queue-based workloads. It&#8217;s also incomplete in a way that most teams never discover — until their KEDA configuration does something unexpected.</p>



<p>Queue depth tells you the stock: how many messages are waiting. Queue growth rate (the derivative) tells you the trend: is the backlog getting worse or stabilising? Together, they seem like they should give you everything you need. In practice, the way KEDA evaluates composite triggers creates a dynamic most people don&#8217;t anticipate.</p>



<p>Here&#8217;s a real KEDA ScaledObject configuration from a production-style setup. I used this in a Cloud Native Days France 2026 talk on advanced horizontal scaling with HPA, VPA, and KEDA, where I presented <code>deriv()</code> as a guardrail on top of backlog-based scaling:</p>



<pre class="wp-block-code"><code>triggers:
  # Stock: absolute backlog
  - type: prometheus
    metadata:
      query: |
        jetstream_consumer_num_pending{
          stream_name="MEMES", consumer_name="meme-backend"}
      threshold: "10"
  # Trend: growth rate as qualifier
  - type: prometheus
    metadata:
      query: |
        (deriv(nats_consumer_num_pending{
          stream_name="MEMES"}&#91;5m]) &gt; 10)
        AND
        (avg(memegenerator_pod_productivity) &lt; 0.6)</code></pre>



<p>Notice the structure. The backlog trigger is the primary replica calculator — KEDA divides pending messages by the threshold to compute desired replicas. The <code>deriv()</code> trigger is an AND condition: it only fires when the queue is actively growing AND per-pod productivity has dropped. It qualifies, but it doesn&#8217;t drive.</p>



<p>This matters because of how KEDA evaluates multiple triggers: it takes the <strong>MAX across all triggers</strong>. Whichever trigger computes the highest replica count wins.</p>



<p>After the talk, I went back and tested this properly. What I found challenged the approach I&#8217;d presented on stage.</p>



<h3 class="wp-block-heading" id="The-unit-comparison-trap">The unit comparison trap</h3>



<p>The reason <code>deriv()</code> rarely contributes becomes obvious when you look at the units. The backlog trigger works in messages: 500 pending messages ÷ threshold of 10 = 50 desired replicas. The deriv trigger works in messages per second: a growth rate of 200 msg/s ÷ threshold of 10 msg/s = 20 desired replicas.</p>



<p>These are fundamentally different units. KEDA doesn&#8217;t know that. It compares 50 and 20, takes the max, and scales to 50. Backlog wins — not because it&#8217;s a better signal, but because its unit produces a larger number at typical production scales.</p>



<p>But if you miscalibrate the thresholds, growth rate can accidentally become the driver. Set the backlog threshold to 100 and the deriv threshold to 5, and suddenly at moderate load the derivative produces more replicas than the backlog. Your &#8220;qualifier&#8221; just became the primary signal without you realising it.</p>



<h3 class="wp-block-heading" id="Controlled-Testing:-deriv()-vs-Backlog-in-KEDA's-MAX-Across-Triggers-Model">Controlled Testing: deriv() vs Backlog in KEDA&#8217;s MAX-Across-Triggers Model</h3>



<p>I ran controlled A/B experiments on a live GKE cluster with NATS JetStream, using the same meme-generator application from the talk.</p>



<p>The first finding was sobering: the production configuration had broken deriv PromQL. The query used <code>deriv((pending + ack_pending)[2m])</code>, which Prometheus rejects — ranges are only allowed on vector selectors. KEDA logged <code>PartialTriggerError</code> with a high failure count. The deriv branch was effectively dead in production. It was difficult to notice as first, because the backlog trigger handled everything on its own.</p>



<p>After correcting the query and running controlled tests with the consumer disabled (to isolate true queue dynamics), the results were consistent: deep backlog with flat growth — backlog dominated every time. Shallow backlog with acceleration — deriv won briefly at 1-second sampling resolution, but at HPA/KEDA decision-window cadence, backlog still produced more replicas.</p>



<p>The deriv trigger wasn&#8217;t gating anything. It wasn&#8217;t contributing to any scaling decision. It was dead weight in the ScaledObject — invisible to the operator, invisible in KEDA&#8217;s status, and structurally unable to influence the outcome due to MAX semantics.</p>



<h3 class="wp-block-heading" id="The-concept-was-right.-The-implementation-was-wrong.">The concept was right. The implementation was wrong.</h3>



<p>I was right in that trajectory matters. You absolutely should care whether the queue is getting worse or stabilising. A queue at 1,000 messages with <code>deriv()</code> at zero is a fundamentally different situation from a queue at 100 messages with <code>deriv()</code> at 50 msg/s. The first might drain on its own. The second will be at 1,100 in 20 seconds.</p>



<p>But <code>deriv()</code> inside KEDA&#8217;s MAX-across-triggers model is the wrong vehicle for expressing that insight. KEDA&#8217;s evaluation model can only scale MORE — it takes the highest replica count. It can&#8217;t express &#8220;scale LESS because this other signal says wait.&#8221; And the AND clause embedded inside the PromQL query is opaque: the operator never sees &#8220;growth rate says yes, productivity says no&#8221; in any dashboard or KEDA status output. The guardrail is hidden.</p>



<p>The strongest practical alternative I found was <code>predict_linear</code>. Instead of measuring instantaneous rate of change, it forecasts where the backlog will be in 30 seconds:</p>



<pre class="wp-block-code"><code>clamp_min(predict_linear(jetstream_consumer_num_pending{stream_name="MEMES"}&#91;2m], 30), 0)</code></pre>



<p>In a clean comparison, <code>predict_linear</code> triggered first horizontal scale-out <strong>2.66 seconds earlier</strong> than backlog-only. No code changes required. No custom instrumentation. Just a smarter PromQL query that captures the same trajectory concept <code>deriv()</code> was trying to capture — but expressed as a forecast that KEDA can actually use to compute replicas.</p>



<p>That 2.66 seconds might not sound like much. In a microburst scenario where your queue goes from 0 to 500 in 10 seconds, it&#8217;s the difference between scaling before users notice and scaling after your P99 has already degraded.</p>



<h3 class="wp-block-heading" id="Beyond-PromQL:-Coordinated-Scaling-with-ScaleOps">Beyond PromQL: Coordinated Scaling with ScaleOps</h3>



<p><code>predict_linear</code> is the best you can do with standard PromQL and no code changes. But it&#8217;s still forecasting from a short window. It catches ramps in progress but can&#8217;t anticipate a traffic pattern it hasn&#8217;t seen yet.</p>



<p>Queue horizontal scaling isn&#8217;t just about one threshold. In production, queue pressure, CPU demand, and baseline replica needs evolve together. The configuration you deployed three months ago may no longer reflect how your workload actually behaves. This is where the set-and-forget nature of KEDA becomes visible: <code>threshold</code>, <code>minReplicaCount</code>, and <code>resources.requests.cpu</code> are all static values encoding assumptions about a workload that changes over time.</p>



<p>ScaleOps continuously reconciles those layers. It rightsizes resource requests based on observed workload behaviour — CPU and memory consumption, replica history — which for queue workloads naturally correlates with queue activity. It adjusts the replica floor from learned patterns, so capacity is already warm when recurring load arrives rather than waiting for the queue to fill first. And when a CPU utilization trigger sits alongside the queue trigger in the same ScaledObject (a common pattern where CPU acts as a safety net), ScaleOps keeps that trigger&#8217;s scaling intent stable as requests change, so the two triggers don&#8217;t drift out of alignment after rightsizing.</p>



<p>If your queue traffic is genuinely unpredictable, i.e., no recurring patterns, no periodicity, then learned behaviour won&#8217;t help and you need faster reaction instead. ScaleOps Burst Reaction addresses this by shifting from historical baselines to real-time resource usage when it detects sustained spikes, giving each pod more headroom to process messages faster even before horizontal scaling kicks in. In a follow-up article, I&#8217;ll also show how custom instrumentation like queue-wait histograms, semaphore saturation gauges, and time-to-drain estimates can push queue-based horizontal scaling from reactive to genuinely proactive for those cases. Standard PromQL gets you 80% of the way. The last 20% requires knowing not just how many messages are waiting, but how long they&#8217;ve been waiting and how fast your pods are actually draining them.</p>



<h2 class="wp-block-heading" id="P99-Latency:-Early-Warning,-Not-a-Leading-Metric">P99 Latency: Early Warning, Not a Leading Metric</h2>



<p>P99 latency occupies an awkward position in the metrics spectrum. It is closer to leading than CPU utilization — tail behaviour shifts before averages move — but by the time your 99th percentile degrades, 1% of your users have already experienced bad requests. The signal is early, but the damage has started.</p>



<p>That distinction matters for horizontal scaling decisions. CPU throttling ratio tells you the kernel is constraining the container before users feel anything. Queue growth rate tells you demand is accelerating before the backlog is deep enough to cause latency. P99 tells you latency has already degraded for the tail. It is the earliest lagging signal, not a leading one.</p>



<p>The PromQL:</p>



<pre class="wp-block-code"><code>histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket&#91;2m])) by (le))</code></pre>



<p>This requires application-level histogram instrumentation. Unlike CPU metrics (which the kubelet provides automatically) or queue depth (which your message broker exports natively), P99 latency demands that your application records request durations into Prometheus histogram buckets. That means code you have to write, or a service mesh you have to deploy. It is not free.</p>



<p>The instrumentation has a gotcha that affects almost every team at least once: bucket boundaries. Prometheus histograms use predefined buckets — <code>[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]</code> by default. <code>histogram_quantile</code> interpolates within those buckets. If your SLO is 200ms but your nearest bucket boundaries are 100ms and 250ms, you cannot distinguish a 110ms response from a 240ms one — they land in the same bucket. The quantile calculation assumes uniform distribution within the bucket, which for tail latency is almost never accurate.</p>



<p>For scaling decisions, this means your trigger threshold needs to account for bucket resolution, not just the SLO number. A P99 target of 200ms with a 250ms bucket boundary will never fire until you have already exceeded your SLO.</p>



<p>Where P99 earns its place: as a confirmation signal alongside throttling or queue metrics for request-serving workloads. If throttling ratio says the container is under pressure AND P99 is climbing, you have high confidence that horizontal scaling is warranted. Not just CPU contention, but user-visible impact. For batch processors and queue consumers, P99 of individual request handling time is less meaningful — queue depth and processing rate tell a more direct story.</p>



<h2 class="wp-block-heading" id="Memory-Pressure:-The-Forgotten-Kubernetes-Autoscaling-Signal">Memory Pressure: The Forgotten Kubernetes Autoscaling Signal</h2>



<p>Memory behaves fundamentally differently from CPU as a scaling signal. CPU degrades gradually —latency climbs, throttling increases, requests slow down, and the gradient gives you time to observe and react. Memory consumption typically grows over time as connection pools fill, caches warm, and buffers accumulate, with no equivalent of CPU throttling to indicate proximity to a limit. A container moves from 70% of its memory limit to 85% to 95%, and then the kernel&#8217;s OOM killer terminates it. There is no graceful degradation step between &#8220;approaching the limit&#8221; and &#8220;process killed.&#8221;</p>



<p>This characteristic makes memory one of the most difficult metrics to use as a horizontal scaling trigger. By the time memory pressure is high enough to cross an HPA threshold, pods are already being OOMKilled.</p>



<p>The metric to watch is <code>container_memory_working_set_bytes</code>, not <code>container_memory_usage_bytes</code>. The distinction is important: <code>usage_bytes</code> includes reclaimable filesystem cache — memory the kernel can reclaim under pressure without terminating anything. <code>working_set_bytes</code> is the memory the container is actively using and the kernel cannot reclaim. This is what the OOM killer evaluates. Scaling decisions based on <code>usage_bytes</code> will fire too early, because cache inflation looks like memory pressure when it is not.</p>



<p>The PromQL for the ratio that matters:</p>



<pre class="wp-block-code"><code>container_memory_working_set_bytes{container="your-app"}
/ container_spec_memory_limit_bytes{container="your-app"}</code></pre>



<p>Even with the correct metric, using memory as a horizontal scaling trigger is structurally limited. Adding replicas does not reduce per-pod memory consumption. If your application has a memory leak or an unbounded cache, ten pods will each consume the same amount. Memory pressure is primarily a vertical scaling signal: the correct response is larger memory requests, not more pods.</p>



<p>This is where continuous rightsizing matters more than for any other metric. Developer-guessed memory requests — which is how most deployments start — are almost always inaccurate. Too low, and you get OOMKills under production load. Too high, and you waste capacity cluster-wide. ScaleOps sets memory requests from observed usage patterns, maintaining headroom above the working set without leaving the gap that allows memory consumption to reach the limit undetected.</p>



<p>One development that changes this equation: Kubernetes swap support (beta since 1.28, with <code>LimitedSwap</code> behaviour stabilising through 1.33-1.35) gives memory-pressured containers a degradation path that does not end in immediate termination. Performance degrades, but the process survives. I covered the tradeoffs, benchmarks, and decision framework in my KubeCon EU 2026 talk, &#8220;<a href="https://canva.link/fwk9qa0c1r7fyui">To Swap or Not to Swap – Memory Management Design Patterns for AI Workloads in Kubernetes 1.34+</a>&#8220;.</p>



<h2 class="wp-block-heading" id="From-Reactive-Metrics-to-Predictive-Kubernetes-Autoscaling">From Reactive Metrics to Predictive Kubernetes Autoscaling</h2>



<p>Every kubernetes leading metric covered in this article is still fundamentally reactive. Throttling ratio is faster than CPU utilization. <code>predict_linear</code> is faster than raw backlog. PSI works where throttling does not. But each of these responds to something that has already started happening — the signal is earlier in the chain, not ahead of it.</p>



<p>Predictive horizontal scaling operates differently: it raises capacity before any metric fires, based on learned workload patterns rather than real-time signals. The reactive trigger becomes the safety net rather than the primary mechanism.</p>



<p>This is what ScaleOps Replica Optimization does. It replaces the static, manually-set <code>minReplicas</code> with a continuously-updated one — lower when observed patterns indicate over-provisioning, higher when a recurring load pattern is approaching. The result is that horizontal scaling triggers fire less often, because the baseline capacity already reflects what the workload needs.</p>



<p>A concrete starting point: add CPU throttling ratio as a metric on a single workload this week and compare its timing against your existing CPU utilization trigger. When you see how much earlier the kernel reports pressure, the case for moving up the metrics spectrum becomes clear.</p>



<p>ScaleOps works with your existing HPA and KEDA configuration — no migration, no rearchitecture. <a href="https://scaleops.com/">Start free</a> to see how your current scaling signals compare to what your workloads actually need, or <a href="https://scaleops.com/demo">book a demo</a> to see Replica Optimization running on your own cluster data.</p>


<div class="b-faq-section has-inner-container align wp-block-faq-section">
  <div class="b-faq-section__items flex flex-col gap-4">
    <div class="faq-section-innerblocks">

<h2 class="wp-block-heading">Kubernetes Leading Metrics: Frequently Asked Questions</h2>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading">What is the difference between leading and lagging metrics in Kubernetes?</h3>



<p>A leading metric signals emerging stress before it impacts users — CPU throttling ratio, queue growth rate, or PSI contention data. A lagging metric reports what already happened — CPU utilization percentage, average response time, or error rate. Kubernetes horizontal scaling is only as fast as the metric it reacts to.</p>

</div>
</div>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading">Does CPU throttling ratio work without CPU limits?</h3>



<p>No. CFS throttling only occurs when a container exceeds its CPU limit quota, so removing limits eliminates the signal. Since Kubernetes 1.34, PSI (Pressure Stall Information) provides an alternative that works without limits — <code>container_pressure_cpu_waiting_seconds_total</code> measures CPU contention regardless of whether limits are set. The practical model is three tiers: throttling ratio if you set limits, PSI if you don&#8217;t, both for the full picture.</p>

</div>
</div>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading">When should I use KEDA instead of HPA for custom metrics?</h3>



<p>KEDA adds value when you need scale-to-zero, composite triggers combining multiple signals, or native scaler support for message queues and streams. For a single Prometheus-based custom metric — such as throttling ratio or P99 latency — HPA with Prometheus Adapter involves fewer moving parts and works well. KEDA&#8217;s strength is orchestrating multiple event sources, not replacing HPA for single-metric horizontal scaling.</p>

</div>
</div>

</div>
  </div>
</div>

<p>The post <a href="https://scaleops.com/blog/blog-kubernetes-leading-scaling-metrics/">Kubernetes Leading Metrics: Why CPU Utilization Is the Worst Scaling Signal (And What to Use Instead)</a> appeared first on <a href="https://scaleops.com">ScaleOps</a>.</p>
