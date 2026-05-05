---
title: "Why Pod Rightsizing Fails in Production: A Deep Dive into VPA and What Actually Works"
url: "https://scaleops.com/blog/why-pod-rightsizing-fails-in-production-a-deep-dive-into-vpa-and-what-actually-works/"
date: "Wed, 28 Jan 2026 11:21:01 +0000"
author: "Nic Vermandé"
feed_url: "https://scaleops.com/feed/"
---
<section class="afb-take-aways wp-block-take-aways">
  <div class="afb-take-aways__inner">

<h4 class="wp-block-heading has-text-color has-large-font-size" style="color: #5551FF; font-style: normal; font-weight: 600;">Main Takeaway</h4>



<p>Kubernetes VPA promises to fix drift automatically, but most teams avoid running it fully automated in production because updates can be disruptive, recommendations lack context, and it breaks HPA/KEDA.</p>



<p>ScaleOps doesn’t recommend resource changes. It autonomously manages rightsizing in production, with constraints built in. Workload-aware policies (stateful, stateless, batch, JVM, etc.), real-time burst reaction, and automatic failure mitigation ensure efficiency without risking stability.</p>

</div>
</section>



<h2 class="wp-block-heading">The Cost of Stagnation</h2>



<p>Kubernetes has evolved through three eras: survival (get containers running), scale (stretching clusters to thousands of nodes), and now <strong>efficiency, </strong>doing more with less while <a href="https://scaleops.com/blog/ai-infra-for-production-why-gpu-resource-management-in-kubernetes-demands-a-new-approach/">AI workloads</a> demand resources one minute and sit idle the next.</p>



<p>The numbers are brutal. <a href="https://www.sysdig.com/blog/millions-wasted-kubernetes">Sysdig&#8217;s analysis of billions of containers</a> found that 69% of purchased CPU goes unused. In a 1,000-node cluster, this represents tens of millions of dollars in &#8220;idle tax.&#8221;</p>



<p><a href="https://www.datadoghq.com/state-of-cloud-costs/">Datadog reports</a> that 83% of container spend is tied to underutilized resources, with median CPU utilization around 16%.</p>



<p>Every resources.requests block is a frozen hypothesis – a guess made during initial deployment. By Day 100, entropy has invalidated that guess.&nbsp;</p>



<p><strong>That frozen hypothesis is now technical debt with a monthly invoice.</strong></p>



<p>This isn&#8217;t just a cost issue, it’s a reliability risk. Over-provisioning burns through cloud budgets and under-provisioning leads to 3 AM pages when a pod OOMKills during a routine traffic surge.</p>



<p>The Kubernetes community built the Vertical Pod Autoscaler (VPA) to solve this, yet less than 1% of organizations use VPA in production. The <a href="https://www.datadoghq.com/state-of-cloud-costs/">tool designed</a> to solve the rightsizing problem is too dangerous to actually use.</p>



<p>VPA&#8217;s design yields predictable limitations at scale. Here is the math on why.</p>


<section class="afb-take-aways wp-block-take-aways">
  <div class="afb-take-aways__inner">

<p><em>If you want to see these limitations in your own cluster, <a href="http://www.try.scaleops.com">ScaleOps runs in read-only mode out of the box</a>. Two minutes to install, immediate visibility into your efficiency score and savings potential. No automation, no changes. Just clarity on what&#8217;s being left on the table.</em></p>

</div>
</section>



<h2 class="wp-block-heading">How VPA Actually Works (A Primer)</h2>



<p>To understand why <a href="https://scaleops.com/blog/kubernetes-vpa/">VPA</a> fails in production, we must look at its three core components:&nbsp;</p>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>Component</td><td>Responsibility</td><td>Limitation</td></tr><tr><td>Recommender</td><td>Calculate optimal resources from historical usage.</td><td>Backward-looking only; cannot react to real-time spikes.</td></tr><tr><td>Updater</td><td>Evict pods when recommendations diverge significantly.</td><td>Binary eviction; causes disruptive restarts.</td></tr><tr><td>Admission Controller</td><td>Apply recommendations at pod creation.</td><td>Cannot modify running pods.</td></tr></tbody></table></figure>
</div>


<h2 class="wp-block-heading">The Three Components of VPA</h2>



<p>VPA is a system of three cooperating components, each with a distinct responsibility:</p>



<h3 class="wp-block-heading">The recommender (VPA’s brain):&nbsp;</h3>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>Watches container resource usage via the Metrics Server.</li>



<li>Maintains decaying histograms of historical consumption.</li>



<li>Calculates recommended CPU and memory values.</li>



<li>Stores recommendations in the VPA object’s .status.recommendation field.</li>



<li>Runs continuously, updating recommendations every few minutes using a sliding historical window.</li>



<li>Samples one memory peak per day rather than full time-series data</li>



<li>Applies exponential decay weighting, with yesterday’s data weighted at ~50% of today’s</li>
</ul>
</div>


<p>The recommender is entirely backward-looking. No awareness of live pressure, future demand, or workload context</p>



<h3 class="wp-block-heading">The Updater (VPA’s enforcer)&nbsp;</h3>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>Monitors running pods and compares their current requests to the Recommender’s suggestions</li>



<li>Determines whether a pod is “out of policy” based on configurable thresholds</li>



<li>Decision model is binary: either evict the pod or do nothing</li>



<li>Marks pods as “out of policy” and evicts them when necessary, respecting PodDisruptionBudgets if configured.</li>



<li>Relies on the Admission Controller to apply correct resources to replacement pods.</li>
</ul>
</div>


<p>The updater has no understanding of workload criticality, latency sensitivity, or failure domains</p>



<h3 class="wp-block-heading">The Admission Controller (VPA’s last mile)</h3>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>Implemented as a mutating webhook.</li>



<li>Intercepts pod creation requests and overwrites resource requests with the Recommender’s current values.</li>



<li>Historically required pod recreation because it could only modify pods at creation time.</li>



<li>With Kubernetes v1.35, In-Place Pod Resize is GA, and VPA’s InPlaceOrRecreate mode can now attempt live updates.</li>
</ul>
</div>


<p>But the Recommender feeding those updates hasn’t changed. And that’s where the real problems live.</p>



<h2 class="wp-block-heading">VPA’s Modes of Operation</h2>



<p>VPA&#8217;s modes tell a story of escalating risk: from safe but passive, to automated but fragile, to actively disruptive. Each step trades operational safety for automation.&nbsp;</p>



<h4 class="wp-block-heading">Off Mode (Visibility only)&nbsp;</h4>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>VPA calculates resource recommendations and writes them to the VPA object’s status field&nbsp;</li>



<li>No pods are modified and no actions are taken&nbsp;</li>
</ul>
</div>


<p><strong>What this really means: </strong>You get insight without risk. VPA acts as a sizing calculator, not an automation system. This is useful for initial baselines, but it does nothing to prevent long-term resource drift.</p>



<p>Most teams stay here.</p>



<h3 class="wp-block-heading">Initial Mode (Day-1 Automation)&nbsp;</h3>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>VPA calculates and applies recommendations <strong>only when pods are first created&nbsp;</strong></li>



<li>Running pods are never updated afterward</li>
</ul>
</div>


<p><strong>What this really means: </strong>Day-1 sizing is automated but by Day-100 reality is ignored. Workloads drift and VPA observes, but never intervenes.</p>



<p>This mode automates the initial guess, not ongoing correctness.</p>



<h3 class="wp-block-heading">Recreate Mode (Disruptive Automation)&nbsp;</h3>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>The Updater compares running pods to current recommendations&nbsp;</li>



<li>If the deviation exceeds a threshold, the pod is evicted</li>



<li>The Admission Controller applies new values when the replacement pod starts</li>
</ul>
</div>


<p><strong>What this really means: </strong>VPA enforcement means restarting pods.<br /></p>



<p>And in practice, eviction causes real damage:&nbsp;</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>JVM loses its JIT-compiled code</li>



<li>Redis cache goes cold</li>



<li>PostgreSQL connections reset.&nbsp;</li>
</ul>
</div>


<h3 class="wp-block-heading">InPlaceOrRecreate Mode (The forward-looking option)</h3>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li><a href="https://scaleops.com/blog/kubernetes-1-35-release-overview/">Kubernetes 1.35</a> graduated InPlacePodVerticalScaling to GA (Dec 2025) and VPA 1.5.0 promoted this mode to Beta</li>



<li>VPA attempts to modify pod resources in-place by adjusting cgroups without restarting the container</li>



<li>If in-place fails, VPA falls back to eviction.</li>
</ul>
</div>


<p>This sounds like the fix everyone was waiting for. It isn&#8217;t.</p>



<p>VPA&#8217;s InPlaceOrRecreate is a &#8220;try-and-fail&#8221; approach. It attempts in-place resize without checking feasibility first.&nbsp;</p>



<p>In-place fails if:</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>The node lacks capacity</li>



<li>The resize would change the pod&#8217;s QoS class</li>



<li>The container&#8217;s resizePolicy requires a restart</li>
</ul>
</div>


<p>And then VPA falls back to eviction anyways. The &#8220;OrRecreate&#8221; in the name says it all: the fallback is baked into the design.</p>



<p>The delivery mechanism is finally production-ready. The recommendation engine remains unchanged. And the update strategy still doesn&#8217;t check before it acts.</p>



<h2 class="wp-block-heading">The &#8220;DIY&#8221; Trap: Why Manual Tuning &amp; Recommendations Fail</h2>



<p>If VPA&#8217;s automation is too risky for production, what about using it as a recommendation engine and applying changes manually?&nbsp;</p>



<p>This is the path most teams take. It doesn&#8217;t work either.</p>



<h3 class="wp-block-heading">The Goldilocks Mirage</h3>



<p>Tools like <a href="https://scaleops.com/blog/scaleops-vs-goldilocks-a-quick-comparison/">Goldilocks</a> are excellent for initial discovery. They deploy VPA in recommendation mode, surface sizing suggestions through a dashboard, and generate YAML that humans can review. For a one-off audit or initial cluster setup, this is genuinely useful.</p>



<p>The problem is what happens after Day 1.</p>



<p>Goldilocks gives you a static snapshot derived from historical data. By the time you open the PR to update the YAML, traffic patterns have shifted. A new feature deployed. A dependency upgraded. You&#8217;re replacing one frozen hypothesis with a slightly newer frozen hypothesis. The fundamental problem remains unsolved.</p>



<p>In large fleets, this model becomes a treadmill. Dozens or hundreds of PRs per week, each requiring manual review, approval, merge, and rollout. Goldilocks is a read-only tool. It generates a perpetual to-do list of sizing recommendations that an engineer must manually apply via kubectl patch or a GitOps workflow. The operational load stays the same. You&#8217;ve just moved it from &#8220;guess at deploy time&#8221; to &#8220;review PRs weekly.&#8221;</p>



<p>And when HPA is managing the same workload, Goldilocks recommendations become actively misleading. The tool sees VPA&#8217;s output without understanding that HPA is controlling replicas. During scale-out, each pod uses less CPU. VPA interprets this as &#8220;pods are over-provisioned&#8221; and recommends lower requests. Apply those recommendations, and when HPA scales back down, your pods are undersized for the concentrated load.</p>



<h3 class="wp-block-heading">The Tuning Fallacy</h3>



<p>The next instinct is to tune VPA itself. Adjust the flags. Shorten the aggregation window. Change the percentile targets.</p>



<p>You can&#8217;t tune your way out of architectural constraints.</p>



<p>VPA exposes a handful of global knobs: target-cpu-percentile, target-memory-percentile, recommendation-margin-fraction. But they apply to every workload in your cluster. The same histogram logic serves your latency-sensitive API gateway and your nightly batch ETL. The same decay model applies to your stateful database and your stateless worker fleet.</p>



<p>These workloads have nothing in common. They need different strategies. VPA gives them identical treatment.</p>



<p>The tuning trade-offs are lose-lose. Shorten the aggregation window and recommendations become jittery, triggering restart loops on minor usage fluctuations. Lengthen it and you smooth out the spikes that actually matter. Your P90 load becomes invisible to the recommender.</p>



<p>You cannot tune a global algorithm to fit workloads with fundamentally different needs. A stateful database needs stability and conservative updates. A stateless worker needs agility and fast reaction. VPA forces you to pick one profile and accept the consequences for everything else.</p>



<h2 class="wp-block-heading">When to Use What</h2>



<p>Before we dive into VPA&#8217;s deeper architectural failures, here&#8217;s when each approach actually makes sense:</p>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>Scenario</td><td>Tool</td><td>Why</td></tr><tr><td>Single-cluster dev/staging</td><td>VPA in Off mode</td><td>Low risk. Use recommendations for initial sizing.</td></tr><tr><td>Periodic audits &#8211; no automation</td><td>Goldilocks</td><td>Generates PRs with VPA-backed recommendations. Good for visibility.</td></tr><tr><td>Memory-only sizing (CPU managed by HPA)</td><td>VPA in Initial or Recreate</td><td>Avoids HPA conflict if you tolerate occasional evictions.</td></tr><tr><td>Production fleets at scale</td><td>ScaleOps</td><td>Continuous adjustment, workload-aware policies, real-time reaction.</td></tr><tr><td>Migrating from VPA</td><td>ScaleOps (Read-only) → ScaleOps (Automated)</td><td>See your savings potential first. Enable automation with one click.</td></tr></tbody></table></figure>
</div>


<h2 class="wp-block-heading">The Architecture of Failure: Deconstructing VPA</h2>



<figure class="wp-block-embed is-type-video is-provider-youtube wp-block-embed-youtube wp-embed-aspect-16-9 wp-has-aspect-ratio"><div class="wp-block-embed__wrapper">

</div></figure>



<h3 class="wp-block-heading">The Disruption Tax</h3>



<p>When VPA decides a pod needs different resources, it evicts the pod. The replacement starts with the new values. In a test cluster, this is fine. In production, eviction triggers a cascade of secondary failures.</p>



<p>Your JVM loses its JIT-compiled code and restarts cold. Your Redis instance drops its in-memory cache, triggering a miss storm against your database. Your PostgreSQL connection pools reset, causing transient errors across dependent services. Your user&#8217;s WebSocket session disconnects mid-transaction.</p>



<p>This is why the Datadog report found less than 1% VPA adoption in production. The tool designed to optimize resources became a source of outages.</p>



<p>InPlaceOrRecreate mode fixes the delivery mechanism. Pods can now receive new resource values without restarting. But if VPA recommends the wrong value, it still applies the wrong value. You&#8217;ve eliminated the restart tax while keeping the bad recommendation. That&#8217;s progress on one axis, regression risk on another.</p>



<p>But &#8220;without restarting&#8221; deserves scrutiny. VPA doesn&#8217;t check feasibility before attempting the resize. It tries, and if it fails, it falls back to eviction. The &#8220;OrRecreate&#8221; in the name tells you everything. Here&#8217;s what can go wrong:</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li><strong>Node lacks capacity:</strong> Resize fails. Eviction anyway.</li>



<li><strong>QoS class would change:</strong> Kubernetes rejects it. Eviction anyway.</li>



<li><strong>Container&#8217;s </strong>resizePolicy<strong> requires restart:</strong> You configured for in-place, but the policy forces a restart anyway.</li>



<li><strong>Memory decrease below current usage:</strong> Resize enters limbo. Spec says 2GB, cgroup stays at 4GB, pod runs in purgatory until someone notices PodResizeInProgress buried in status.</li>



<li><strong>Application doesn&#8217;t hot-reload:</strong> JVM&#8217;s -Xmx was set at startup. Node.js max-old-space-size doesn&#8217;t change. PostgreSQL shared_buffers won&#8217;t expand. The cgroup changed. The process didn&#8217;t notice.</li>



<li><strong>Sidecars ignored:</strong> VPA rightsizes your main container. Your Istio proxy (different resource patterns, possibly no metrics) stays at original sizing.</li>



<li><strong>Single replica with strict PDB:</strong> VPA can&#8217;t evict, and if in-place resize fails for any reason, the pod sits with wrong resources indefinitely. No alert, no action. Just waiting for the next loop that never fixes anything.</li>
</ul>
</div>


<p>The delivery mechanism is production-ready, but the failure modes haven&#8217;t really changed – unless your rightsizing solution checks node capacity before recommending, understands that JVMs need heap tuning not just cgroup changes, and knows which workloads can tolerate in-place updates versus which need zero-downtime pod replacement. I&#8217;ll let you guess who does that with zero configuration.</p>



<h3 class="wp-block-heading">The One-Size-Fits-All Problem</h3>



<p>Consider two workloads in the same cluster: a PostgreSQL database serving your production API, and a nightly batch job that processes yesterday&#8217;s logs.</p>



<p>The database needs conservative sizing. It should never be undersized, tolerates being slightly over-provisioned, and absolutely cannot tolerate mid-day evictions. You&#8217;d configure it at P98 with a 15% safety buffer and a maintenance window at 3 AM.</p>



<p>The batch job has different requirements entirely. It runs for 20 minutes, spikes hard, then disappears. It needs aggressive optimization (P85 would be fine), but it absolutely cannot be interrupted mid-execution. Kill it halfway through, and you&#8217;ve wasted the compute and have to start over.</p>



<p>VPA gives them both the same P90. Same 8-day histogram. Same 24-hour decay. Same update logic. And more importantly: VPA has no concept of &#8220;wait for this job to finish”. It sees a pod, decides it needs different resources, and evicts. Mid-execution? VPA doesn&#8217;t know. Doesn&#8217;t care.</p>



<p>The flags that control this behavior are global to the Recommender binary. Change them for your database, and you&#8217;ve changed them for your batch job. There&#8217;s no per-workload configuration in the VPA CRD for algorithm behavior. You get modes and min/max bounds. You don&#8217;t get per-workload percentile targets. You don&#8217;t get job lifecycle awareness.</p>



<p>The workaround works: run multiple VPA Recommender instances with different flags, sharded by namespace. But now you&#8217;re operating multiple control planes for what should be a single policy decision. And you still don&#8217;t have job completion awareness.</p>



<h3 class="wp-block-heading">The Historian Problem</h3>



<p>At its core, VPA is a historian, not a first responder.</p>



<p>The Recommender maintains decayed histograms over an eight-day window with a 24-hour half-life. Yesterday&#8217;s data carries half the weight of today&#8217;s. Data from four days ago carries 6.25% weight. This exponential decay means VPA is mathematically incapable of reacting to what&#8217;s happening right now.</p>



<p>When a traffic spike hits, VPA doesn&#8217;t see it as an emergency. It sees it as one data point in a week-long histogram. By the time the spike accumulates enough weight to shift the recommendation, your pods have been throttling for hours or days.</p>



<p>VPA is driving by looking in the rearview mirror. The recommender flags tell you this explicitly: cpu-histogram-decay-half-life, memory-histogram-decay-half-life, recommender-interval. Every parameter is about how to weight the past, not how to detect the present.</p>



<h3 class="wp-block-heading">The Ratio Preservation Trap</h3>



<p>Here&#8217;s where VPA can actively make performance worse.</p>



<p>By default, VPA updates both requests and limits to preserve your original ratio. Your pod had 100m request and 400m limit, a 4:1 ratio. VPA sees low average usage and recommends 60m. The limit drops to 240m. Same ratio. 40% less burst ceiling. When the next traffic spike arrives, your pod throttles, even though the node has available CPU. The scheduler didn&#8217;t starve you. VPA did.</p>



<p>There&#8217;s no alert. Nothing highlights that your safety margin evaporated. You find out when latency spikes.</p>



<p>VPA offers controlledValues: RequestsOnly as a workaround, leave limits alone. But now you&#8217;re managing limits manually, and VPA still recommends tiny requests based on P90 averages. The scheduler bin-packs your pods tighter. Instead of hitting a limit, your app fights neighbors for CPU. Same throttling, different cause.</p>



<p>This is why enabling VPA can cause <em>more</em> throttling, not less.</p>



<p>That&#8217;s the architecture. Now let&#8217;s see what happens when these design decisions meet production workloads that don&#8217;t behave like steady-state averages.</p>



<h2 class="wp-block-heading">Case Study: The TaxiMetrics Meltdown</h2>



<p>TaxiMetrics is a ride-sharing analytics platform with ML-powered fare predictions. The application stack is the following:</p>



<figure class="wp-block-image size-large"><img alt="" class="wp-image-7535" height="740" src="https://scaleops.com/content/uploads/2026/01/image-5-1024x740.png" width="1024" /></figure>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>Component</td><td>Type</td><td>Resource Pattern</td></tr><tr><td><code>taxi-api</code></td><td>Deployment (HPA 2-10)</td><td>Bursty. Serves prediction endpoints. 10x spikes during surge pricing.</td></tr><tr><td><code>ml-trainer</code></td><td>Job (on-demand)</td><td>Batch. Needs 2 CPU and 4GB for 5 minutes, then terminates.</td></tr><tr><td><code>ml-server</code></td><td>Deployment (1 replica)</td><td>Steady. Loads trained models from PVC. Low baseline.</td></tr><tr><td><code>postgresql</code></td><td>StatefulSet (1 replica)</td><td>Steady. 3.3M+ taxi records. Needs stability over optimization.</td></tr></tbody></table></figure>
</div>


<p>The application has two main flows:</p>



<h3 class="wp-block-heading">Inference Flow (Real Time)</h3>


<div class="wp-block-list--wrapper">
<ol class="wp-block-list">
<li>User POSTs trip details to predict tip</li>



<li>taxi-api validates &amp; checks redis cache</li>



<li>ml-server loads model from NFS</li>



<li>Returns prediction (e.g. &#8220;$4.50 tip&#8221;)</li>
</ol>
</div>


<h3 class="wp-block-heading">Batch Flow (every 10 min)</h3>


<div class="wp-block-list--wrapper">
<ol class="wp-block-list">
<li>CronJob triggers job-controller</li>



<li>data-processor loads 3.3M taxi trips</li>



<li>Writes to PostgreSQL</li>



<li>ml-trainer trains &amp; saves models to NFS</li>
</ol>
</div>


<p>Each component needs a different sizing strategy.</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li><code>taxi-api</code> needs burst headroom.</li>



<li><code>ml-trainer</code> needs massive resources briefly, then nothing.</li>



<li><code>ml-server</code> needs stability.</li>



<li><code>postgresql</code> needs predictability.</li>
</ul>
</div>


<p>VPA applies the same algorithm to all four.</p>



<h3 class="wp-block-heading">Failure Mode 1: The Throttling Cascade (taxi-api)</h3>



<p>Example of taxi-api throttling cascade:</p>



<figure class="wp-block-image size-large"><img alt="" class="wp-image-7536" height="526" src="https://scaleops.com/content/uploads/2026/01/image-6-1024x526.png" width="1024" /></figure>



<p>Notes:&nbsp;</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>Values shown are the sum across all taxi-api pods (2 replicas). Divide by 2 for per-pod values. VPA settings were accelerated for demo purposes.&nbsp;</li>



<li>The histogram half-life was reduced from 24 hours → 2 minutes (720× faster). 1 minute on the graph ≈ 12 hours in production.</li>
</ul>
</div>

<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>Time</td><td>Event</td><td>Total Limit</td><td>What Happened</td></tr><tr><td>~11:20</td><td>Baseline&nbsp;</td><td>2000m</td><td>VPA observing usage, starting to lower</td></tr><tr><td>~11:23-11:26</td><td><strong>Throttle #1</strong></td><td>1500m</td><td>Spike hit lowered ceiling → 19.6 ops/s throttling</td></tr><tr><td>~11:26-11:30</td><td>Recovery</td><td>7000m</td><td>VPA reacted, increased resources</td></tr><tr><td>~11:30-11:35</td><td>Idle decay</td><td>4000m</td><td>VPA sees low usage, lowers again</td></tr><tr><td>~11:35-11:37</td><td><strong>Throttle #2</strong></td><td>2500m</td><td>Another spike, ceiling too low again</td></tr><tr><td>~11:38-11:41</td><td><strong>Throttle #3</strong></td><td>3000m</td><td>Third spike, VPA still catching up</td></tr><tr><td>~11:45</td><td>Stabilized</td><td>8800m</td><td>VPA finally learned, set high buffer</td></tr></tbody></table></figure>
</div>


<p>Friday, 5 PM. Rush hour hits after 22 hours of low utilization.</p>



<p>taxi-api originally had 200m request and 800m limit, a 4:1 ratio. VPA watched, calculated, and recommended: drop the request from 200m to 120m. The limit followed proportionally, 800m down to 480m. Same 4:1 ratio. 40% less burst ceiling. No alert. No indication that safety margin had evaporated.</p>



<p>When Friday traffic spiked, the pods had CPU available on the node. The problem was the 480m limit VPA had silently imposed. The pods hit their ceiling and throttled. container_cpu_cfs_throttled_seconds_total climbed while the node sat with headroom to spare. Latency spiked. Users complained.</p>



<p>The ratio preservation behavior removed burst headroom exactly when it was needed. VPA optimized for yesterday&#8217;s average while destroying today&#8217;s peak capacity. And if <a href="https://scaleops.com/blog/hpa-vs-vpa-understanding-kubernetes-autoscaling-and-why-its-not-enough-in-2025/">HPA is watching</a>? It sees high CPU utilization, scales out, and now you have 6 pods throttling instead of 3. The cloud bill increases while the problem persists.</p>



<h3 class="wp-block-heading">Failure Mode 2: The Batch Job Blind Spot (ml-trainer)</h3>



<p>ml-trainer is a Kubernetes Job. It retrains 5 ML models, needs 2 CPU and 4GB for about 5 minutes, then terminates. And here&#8217;s where VPA&#8217;s architecture hits a wall.</p>



<p>VPA can technically provide recommendations for Jobs. But its eviction model doesn&#8217;t understand batch semantics. A Job should run to completion – that&#8217;s the whole point. VPA&#8217;s Updater doesn&#8217;t know that. It sees a pod, decides resources should change, and evicts. Minute 3 of a 5-minute training run? It Doesn&#8217;t matter, the job restarts from zero. The compute is wasted and the training is delayed.</p>



<p>So you run VPA in Off mode for Jobs, which means no automation at all. Or you leave it on and accept that your batch workloads might get killed mid-execution whenever recommendations shift.</p>



<p>That&#8217;s an architectural boundary that shows up the moment you have workloads that aren&#8217;t &#8220;run forever.&#8221;</p>



<h3 class="wp-block-heading">Failure Mode 3: The Release Loop (ml-server)</h3>



<p>The ML team ships v2 with two additional models, bumping memory requirements from 256MB to 512MB. The deployment passes staging and rolls out Tuesday morning. By afternoon, pages start firing, and the pod keeps OOMKilling on startup.</p>



<p>VPA is still recommending 285MB because its histogram doesn&#8217;t reset on deployments. It has no concept of &#8220;new version.&#8221; What it sees is eight days of v1 running comfortably at 240MB, weighted by a 24-hour half-life that makes yesterday&#8217;s samples worth 50% and last week&#8217;s worth almost nothing. The handful of samples v2 generated before crashing can&#8217;t outweigh that history.</p>



<p>So you wait.</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>Day one, the recommendation climbs to ~320MB – still crashing.</li>



<li>Day two hits ~400MB, and crashes become less frequent.</li>



<li>By day four, VPA finally recommends enough memory to run v2 stable. Four days of production instability because the algorithm needed time to unlearn the version you replaced.</li>
</ul>
</div>


<p>That&#8217;s what happens when your rightsizing tool doesn&#8217;t understand what it&#8217;s scaling. VPA applied the same algorithm to a bursty API, a batch job, and a version upgrade – and failed differently each time. Same algorithm, different failures. ScaleOps starts from a different assumption: workloads aren&#8217;t identical, so their optimization shouldn&#8217;t be either.</p>



<h2 class="wp-block-heading">The ScaleOps Approach: Context-Aware Rightsizing</h2>



<figure class="wp-block-embed is-type-video is-provider-youtube wp-block-embed-youtube wp-embed-aspect-16-9 wp-has-aspect-ratio"><div class="wp-block-embed__wrapper">

</div></figure>



<p>Where VPA offers one algorithm and hopes it fits, ScaleOps matches the optimization strategy to the workload. Auto-detected. Continuously managed. Node-aware.</p>



<h3 class="wp-block-heading">Safety-First Automation</h3>



<p>Most teams disable VPA&#8217;s Auto Mode because they can&#8217;t control when or how updates happen. ScaleOps inverts this.</p>



<p>Application detection drives update strategy: PostgreSQL, MongoDB, Kafka, Redis, and Java apps are identified automatically. Databases get conservative updates that respect connection handling. Java workloads get JVM-aware restarts that account for heap warmup. Stateless services get more aggressive optimization.</p>



<p>Workload detection adds another layer: Jobs and CronJobs are never evicted mid-execution. ScaleOps waits for natural completion, then right-sizes the next instance. StatefulSets get ordinal-aware updates. Deployments with HPA get production-safe policies that won&#8217;t conflict with horizontal scaling.</p>



<p>No labels, no annotations. Detection happens automatically as workloads are deployed.</p>



<figure class="wp-block-image size-large"><img alt="" class="wp-image-7538" height="346" src="https://scaleops.com/content/uploads/2026/01/image-8-1024x346.png" width="1024" /></figure>



<h3 class="wp-block-heading">Real-Time Burst Reaction</h3>



<p>ScaleOps monitors real-time resource usage against historical baselines. When usage shows a sustained increase beyond normal patterns, Burst Reaction kicks in rather than waiting for histograms to catch up. Optimizations shift to higher percentiles, giving your workload headroom until normal patterns resume..</p>



<p>Auto-healing handles memory the same way. When a pod gets OOMKilled, ScaleOps immediately bumps the memory request with the new allocation. No waiting for the next recommender pass. The failure happened, the fix ships.</p>



<p><em>Burts Reaction in action, catching a spike early:</em></p>



<figure class="wp-block-image size-large"><img alt="" class="wp-image-7537" height="353" src="https://scaleops.com/content/uploads/2026/01/image-7-1024x353.png" width="1024" /></figure>



<h3 class="wp-block-heading">Policy Granularity</h3>



<p>ScaleOps doesn&#8217;t apply a single algorithm to every workload. Policies can vary by workload, by namespace, or across the entire cluster.</p>



<p>At the workload level: different percentile targets for bursty APIs versus steady-state services. Different observation windows for batch jobs versus long-running processes. Different limit strategies for memory-sensitive workloads versus CPU-bound compute.</p>



<p>At the namespace level: production namespaces get conservative headroom margins. Development namespaces get aggressive optimization. Cost-sensitive teams can tune differently than reliability-focused teams.</p>



<p>For most users, the defaults handle this automatically. But when teams need control, every parameter is exposed. No black boxes.</p>



<h3 class="wp-block-heading">Node Context Intelligence</h3>



<p>VPA operates in a vacuum. It sees pod metrics and calculates recommendations with no awareness of the node underneath. A liveness probe fails because the node is CPU-starved? An OOM happens because a noisy neighbor exhausted memory? A recommendation exceeds what any node can schedule? VPA can&#8217;t tell the difference – it just sees symptoms and guesses at fixes.</p>



<p>ScaleOps connects pod behavior to node conditions. When liveness probes fail, it checks whether the node is stressed and whether the container is hitting limits – if neither, it concludes &#8220;not a resource issue&#8221; and doesn&#8217;t blindly bump requests. When OOMs occur, it distinguishes node-level kills from container limit breaches, so you know whether to fix the pod or investigate the node.</p>



<p>Recommendations are capped at what your nodes can actually schedule. Fewer false positives. Faster root cause. No pods stuck Pending on impossible requests.</p>



<h2 class="wp-block-heading">The Recovery: TaxiMetrics with ScaleOps</h2>



<p>ScaleOps enabled on the taximetrics namespace. No policy configuration. No tuning. Single Helm install.</p>



<p><strong>What ScaleOps detected:</strong></p>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>Component</td><td>Detection</td><td>Policy Assigned</td></tr><tr><td>taxi-api</td><td>Deployment + HPA</td><td>production</td></tr><tr><td>ml-server</td><td>Deployment</td><td>production</td></tr><tr><td>ml-trainer</td><td>Job</td><td>batch</td></tr><tr><td>postgresql</td><td>StatefulSet + PostgreSQL</td><td>high-availability</td></tr></tbody></table></figure>
</div>


<p>No labels, no annotations. The detection happened automatically as workloads were already running.</p>



<p><strong>Failure Mode 1 → Fixed:</strong> When Friday traffic spiked on taxi-api, Burst Reaction detected the sustained increase and overrode the percentile calculation to capture the spike. Because ScaleOps preserves original limits by default, the burst ceiling stays intact. Zero throttling.</p>



<p><strong>Failure Mode 2 → Fixed:</strong> ml-trainer was detected as a Job and assigned the batch policy. When it ran, ScaleOps observed the actual training profile, not 23 hours of zeros. No mid-execution eviction. The next job instance got right-sized resources based on real execution data.</p>



<p><strong>Failure Mode 3 → Fixed:</strong> When ml-server v2 OOMKilled on first deploy, auto-healing bumped memory immediately. One restart, not four days of loops. The decay model still exists, but failure signals get priority.</p>



<p><strong>The Results (7-Day Comparison):</strong></p>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>Metric</td><td>Before</td><td>After</td><td>Delta</td></tr><tr><td>Avg CPU Utilization</td><td>35%</td><td>71%</td><td>+103%</td></tr><tr><td>P95 Latency (taxi-api)</td><td>420ms</td><td>180ms</td><td>-57%</td></tr><tr><td>Throttling Incidents</td><td>12/day</td><td>0</td><td>-100%</td></tr><tr><td>OOMKills (weekly)</td><td>8-12</td><td>0</td><td>-100%</td></tr><tr><td>Node Hours (weekly)</td><td>1,680</td><td>672</td><td>-60%</td></tr><tr><td>Est. Monthly Spend</td><td>$43k</td><td>$17k</td><td>-$26k</td></tr></tbody></table></figure>
</div>


<p>Rightsizing drove bin packing. Bin packing drove node consolidation. Node consolidation drove savings. The 33% reduction in node hours came from workloads fitting better on fewer nodes, not from running anything slower.</p>



<h2 class="wp-block-heading">The Bottom Line</h2>



<p>We showed you three failure modes. A bursty API throttled because ratio preservation ate its burst headroom. A batch job evicted mid-execution because VPA has no concept of job lifecycle. A version upgrade that triggered four days of OOMKill loops because decay math can&#8217;t adapt to change.</p>



<p>Same recommender. Same histogram logic. Same global percentile. And three different ways to break production.</p>



<p>VPA isn&#8217;t broken – it does exactly what it was designed to do. It calculates percentile-based recommendations from historical usage and applies them. The problem is that production workloads need more than a histogram calculator. They need context.</p>



<p><strong>We demonstrated that ScaleOps adds:</strong></p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li><strong>Workload awareness</strong> that auto-detects batch jobs, stateful workloads, and stateless services then automatically applies the right optimization strategy for each</li>



<li><strong>Burst Reaction</strong> that overrides percentile calculations when sustained spikes happen, instead of waiting for histograms to catch up</li>



<li><strong>Auto-healing</strong> that responds to failures immediately and surfaces root cause – node pressure vs. container limits vs. noisy neighbors</li>



<li><strong>Node context</strong> that caps recommendations at what your cluster can actually schedule</li>
</ul>
</div>


<p><strong><a href="https://scaleops.com/blog/kubernetes-workload-rightsizing/">Rightsizing is stage one.</a></strong> Get it right and you unlock the rest: HPA scales on accurate signals instead of noise. Bin packing improves because workloads are sized to fit. Nodes consolidate because the scheduler has room to work with. The savings compound.</p>



<p>The TaxiMetrics recovery wasn&#8217;t magic. It was the same four workloads, same cluster, same traffic patterns. The difference was a rightsizing layer that understands what it&#8217;s optimizing.</p>



<p>Kubernetes gave you the syscalls. Now you need the userspace.</p>



<h2 class="wp-block-heading"><strong>Your next step: stop guessing.</strong></h2>


<div class="wp-block-list--wrapper">
<ol class="wp-block-list">
<li><a href="#book-a-demo">Book a demo</a> to see how ScaleOps handles your specific workloads – bursty APIs, batch jobs, and all your applications.</li>



<li>Start your<a href="https://try.scaleops.com/"> free trial</a> and get optimization opportunities in under 5 minutes. No code changes. No manifest rewrites.</li>
</ol>
</div>

<div class="b-faq-section has-inner-container align wp-block-faq-section">
  <div class="b-faq-section__items flex flex-col gap-4">
    <div class="faq-section-innerblocks">

<h2 class="wp-block-heading b-faq-section__title">Frequently asked questions</h2>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading b-faq-item__question">What is Kubernetes VPA and why don&#8217;t teams use it in production?</h3>



<p class="b-faq-item__answer-text">Kubernetes Vertical Pod Autoscaler (VPA) automatically calculates and applies resource recommendations for pods, but less than 1% of organizations run it in production because it evicts pods to apply changes, causing service disruptions, and uses a one-size-fits-all algorithm that can&#8217;t distinguish between different workload types like databases versus batch jobs.</p>

</div>
</div>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading b-faq-item__question">How does VPA&#8217;s recommendation engine actually work?</h3>



<p class="b-faq-item__answer-text">VPA&#8217;s recommender maintains decaying histograms over an eight-day window with a 24-hour half-life, meaning yesterday&#8217;s data carries 50% weight and four-day-old data carries just 6.25% weight, making it fundamentally backward-looking and unable to react to real-time traffic spikes or sudden workload changes.</p>

</div>
</div>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading b-faq-item__question">Why does enabling VPA sometimes cause more CPU throttling instead of less?</h3>



<p class="b-faq-item__answer-text">VPA preserves your original request-to-limit ratio by default, so when it lowers a pod&#8217;s CPU request from 200m to 120m, it also drops the limit from 800m to 480m, reducing burst ceiling by 40% exactly when traffic spikes arrive, causing throttling even when the node has available CPU.</p>

</div>
</div>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading b-faq-item__question">What happens when VPA manages a Kubernetes Job or CronJob?</h3>



<p class="b-faq-item__answer-text">VPA&#8217;s updater doesn&#8217;t understand batch semantics and will evict a Job mid-execution if recommendations change, wasting the compute already spent and forcing the job to restart from zero, teams either disable VPA for Jobs entirely or accept that batch workloads get killed randomly.</p>

</div>
</div>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading b-faq-item__question">How does ScaleOps differ from VPA&#8217;s approach to rightsizing?</h3>



<p class="b-faq-item__answer-text">ScaleOps auto-detects workload types (databases, batch jobs, stateless services, JVMs) and manages context-specific optimization policies instead of one global algorithm, waits for Jobs to complete before rightsizing, and uses Burst Reaction to override percentile calculations during sustained traffic spikes rather than waiting days for histograms to catch up.</p>

</div>
</div>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading b-faq-item__question">What is ScaleOps Burst Reaction and how does it prevent throttling?</h3>



<p class="b-faq-item__answer-text">Burst Reaction monitors real-time resource usage against historical baselines and immediately shifts optimizations to higher percentiles when sustained increases occur, giving workloads headroom during traffic spikes instead of waiting for VPA&#8217;s eight-day histogram to accumulate enough weight to change recommendations.</p>

</div>
</div>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading b-faq-item__question">Why can&#8217;t you tune VPA to work for different workload types?</h3>



<p class="b-faq-item__answer-text">VPA&#8217;s tuning flags like <code>target-cpu-percentile</code> and <code>recommendation-margin-fraction</code> apply globally to every workload in your cluster, forcing you to choose between conservative settings that waste resources on stateless apps or aggressive settings that destabilize databases, there&#8217;s no per-workload configuration in the VPA CRD.</p>

</div>
</div>

</div>
  </div>
</div>

<p>The post <a href="https://scaleops.com/blog/why-pod-rightsizing-fails-in-production-a-deep-dive-into-vpa-and-what-actually-works/">Why Pod Rightsizing Fails in Production: A Deep Dive into VPA and What Actually Works</a> appeared first on <a href="https://scaleops.com">ScaleOps</a>.</p>
