---
title: "Kubernetes Swap for AI Inference: LimitedSwap, Memory Hierarchy, and GPU Workload Sizing"
url: "https://scaleops.com/blog/kubernetes-swap-gpu-workload-sizing/"
date: "Mon, 20 Apr 2026 12:13:14 +0000"
author: "Nic Vermandé"
feed_url: "https://scaleops.com/feed/"
---
<section class="afb-take-aways wp-block-take-aways">
  <div class="afb-take-aways__inner">

<h4 class="wp-block-heading has-text-color has-large-font-size" style="color: #5551FF; font-style: normal; font-weight: 600;">Main takeaways from this article</h4>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>Kubernetes swap (KEP-2400, stable since v1.34) gives Burstable QoS pods proportional swap access via cgroup v2, while Guaranteed and BestEffort pods get zero swap.</li>



<li>Two levers control pod survival: your cgroup memory limit must cover steady-state runtime, and your swap entitlement covers burst spikes above that.</li>



<li>The LimitedSwap formula uses memory request, not limit: <code>swap_limit = (request / node_capacity) × node_swap</code>.</li>



<li>Undersized swap is worse than no swap — measured 60% longer outage than a clean OOMKill.</li>



<li>For vLLM workloads, model weights live in GPU VRAM (outside the cgroup). The CPU-side runtime (tokeniser, scheduler, PagedAttention swap pool) is what the cgroup limits and what the kernel swaps.</li>
</ul>
</div>
</div>
</section>



<h2 class="wp-block-heading"><strong>Why Kubernetes Swap Matters for GPU Inference Workloads</strong></h2>



<p>Kubernetes swap is the ability to configure Linux swap memory on Kubernetes nodes so that pods can use disk-backed virtual memory as overflow when physical RAM is exhausted. This feature has been a long time coming. KEP-2400 introduced swap support as alpha in v1.22 (2021) and took four years to reach GA. The critical milestone was v1.30, where the Kubernetes team removed UnlimitedSwap entirely. That mode let any container consume unlimited swap, allowing a single misbehaving pod to fill the swap device and crash the node. What survived is LimitedSwap, stable since <a href="https://scaleops.com/blog/kubernetes-1-34-features-explained-faster-safer-and-cheaper-clusters/" id="5528" type="post">v1.34</a> (August 2025). It&#8217;s cgroup v2-integrated: the kubelet sets <code>memory.swap.max</code> per container based on a proportional formula:</p>



<p><code>swap_limit = (memory_request / node_memory_capacity) × node_swap_size</code></p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>Guaranteed QoS pods (requests equal limits) get <code>memory.swap.max = 0</code> , so complete swap isolation.</li>



<li>BestEffort pods also get zero swap, because with no memory request the formula&#8217;s numerator is zero.</li>



<li>Only Burstable QoS pods (where requests are lower than limits) get proportional swap access.</li>
</ul>
</div>


<p>I verified this formula byte-exact on GKE: a 128MiB request on a 16GB-capacity node with 4GB swap yielded <code>memory.swap.max</code> of exactly 34,365,440 bytes, page-rounded to 4KiB boundaries.</p>



<p>If you&#8217;re running AI inference workloads on Kubernetes — vLLM, Triton, TGI, or any model serving framework on GPU nodes — you&#8217;ve probably been told to disable swap. The kubelet won&#8217;t even start if swap is detected, unless you set <code>failSwapOn: false</code> in the kubelet configuration and enable the <code>memorySwap</code> settings.</p>



<p>But if you&#8217;ve also dealt with OOMKilled pods during traffic spikes on expensive GPU nodes, you know the other side. A single A100 node on GKE costs roughly $2,700 per month. When your inference pod gets OOMKilled during a burst and takes 60-120 seconds to reload model weights into VRAM, you&#8217;re losing both compute and revenue. This is the gap that tools like <a href="https://scaleops.com/">ScaleOps</a> address: autonomous resource management that sets your memory requests and limits based on observed workload behaviour rather than developer guesswork. But even with optimal sizing, you still need a safety net for unpredictable spikes. That&#8217;s where Kubernetes swap comes in.</p>



<p>I spent two months testing Kubernetes swap on real GPU hardware: 30+ test iterations across two clusters (GKE 1.34 and kubeadm v1.29 with an NVIDIA L4), running vLLM with Mistral-7B, phi-2, and Whisper under various swap configurations. </p>



<p>I presented the results at KubeCon EU 2026 in Amsterdam. This article is the extended version, deeper on the Linux memory internals, more precise on the sizing, and corrected with data I discovered during final validation.</p>



<p><strong>What you&#8217;ll learn:</strong></p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>The three-tier memory hierarchy most discussions about Kubernetes swap completely miss (GPU VRAM → CPU RAM → NVMe swap)</li>



<li>How Linux actually manages memory under cgroup pressure — anonymous vs file-backed pages, demand paging, and the kernel reclaim sequence before OOMKill</li>



<li>Why two pods with identical swap entitlements had completely different outcomes, and the critical correction I discovered during validation</li>



<li>Measured results from four swap configurations on identical hardware, including why two pods with identical swap entitlement had opposite outcomes</li>



<li>A practical sizing guide for vLLM memory requests and limits with LimitedSwap</li>



<li>A decision framework for when to enable Kubernetes swap, when to keep it off, and when misconfigured swap is worse than no swap at all</li>
</ul>
</div>


<h2 class="wp-block-heading"><strong>The 8x Degradation That Started This Investigation</strong></h2>



<p>The investigation started with a straightforward question: what actually happens to AI inference latency when Kubernetes swap is active on a GPU node? The answer turned out to be more nuanced than most discussions suggest, and the measured data was striking enough to warrant a full test campaign</p>



<p>I ran Microsoft&#8217;s phi-2 (a 2.7B parameter model) on CPU-only inference with constrained pod memory and UnlimitedSwap enabled on a kubeadm v1.29 cluster. The baseline p50 latency was 14.2 seconds per request — phi-2 on CPU is inherently slow, but that&#8217;s the point. Under swap pressure with <code>memory.swap.max</code> uncapped, that p50 jumped to 116.5 seconds. An 8.19x degradation. The kernel had swapped 1.6GB of the process&#8217;s anonymous memory to the NVMe swap device, 94,000 page-ins and 437,000 page-outs over the test window. Every inference step was blocked on disk I/O for pages that should have been in RAM.</p>



<p>Second test: GPU node, Mistral-7B on an NVIDIA L4. I constrained the pod&#8217;s CPU memory below what vLLM needs for its runtime overhead. The model weights live in GPU VRAM (outside the cgroup entirely), but vLLM&#8217;s CPU-side runtime — the tokeniser, scheduler, output buffers, and the PagedAttention KV cache eviction pool (<code>--swap-space</code>) — all live in CPU RAM, inside the cgroup. The kernel swapped 1.6GB. GPU utilisation dropped from 94% to 73.5%. Latency rose 26%. The GPU was computing, but it was constantly waiting on the CPU to fault pages back from the swap device before it could process the next batch. Prometheus showed 195,000 page-ins during the test window.</p>



<p>Now here&#8217;s the contrast. Same pod spec, same traffic, Kubernetes swap disabled: the pod OOMKilled immediately with CrashLoopBackOff. It was a total outage with zero requests served.</p>



<p>The question isn&#8217;t &#8220;should you enable Kubernetes swap.&#8221; It&#8217;s &#8220;which failure mode do you prefer?&#8221; Degraded service at 73.5% GPU utilisation, or a dead pod at 0%? And more importantly: can you configure LimitedSwap so it catches memory spikes without the 8x performance collapse?</p>



<p>That&#8217;s what LimitedSwap — stable since Kubernetes 1.34 — is designed to solve. But to configure it correctly, you need to understand the full memory hierarchy that AI inference workloads create on a GPU node.</p>



<h2 class="wp-block-heading"><strong>The Two Types of Swap Nobody Talks About</strong></h2>



<p>Most discussions about Kubernetes swap focus on the node-level configuration — should swap be on or off? That framing misses a critical architectural distinction for AI inference workloads. On a GPU node running vLLM, memory doesn&#8217;t live in a single pool. There are three tiers, and two of them involve swapping — managed by entirely different systems with entirely different levels of intelligence about what to evict.</p>



<p>Think of it like a CPU cache hierarchy: L1, L2, L3. Each level is larger, slower, and managed differently.</p>



<p><strong>L1: GPU VRAM (HBM — 2 TB/s bandwidth)</strong></p>



<p>This is where model weights and the active KV cache live during inference. For Mistral-7B in FP16, that&#8217;s roughly 14GB of weights permanently resident in VRAM, plus the dynamically-growing attention context (keys and values) for every active request. The KV cache is the stored attention context — one entry per token per layer per active conversation. More concurrent users means longer conversations, more KV cache entries, and more VRAM consumed.</p>



<p>VRAM is managed by vLLM&#8217;s PagedAttention system and the PyTorch CUDA allocator. Critically, <strong>VRAM is outside the Kubernetes cgroup,</strong> which means the <code>memory.max</code> cgroup limit does not apply to GPU memory, only to CPU RAM.</p>



<p><strong>L2: vLLM CPU Memory Pool (RAM — 100 GB/s bandwidth)</strong></p>



<p>When VRAM fills up, PagedAttention doesn&#8217;t crash the process. It evicts the least-recently-used KV cache blocks from GPU VRAM to CPU RAM via PCIe at 64 GB/s. This CPU-side buffer is what the <code>--swap-space</code> parameter controls:</p>



<figure class="wp-block-image size-large"><img alt="" class="wp-image-8908" height="453" src="https://scaleops.com/content/uploads/2026/04/image-1-1024x453.png" width="1024" /></figure>



<p>The <code>--swap-space 4</code> means &#8220;allocate 4GB of CPU RAM as a landing zone for evicted attention context.&#8221; This is application-level swap: vLLM understands which KV blocks are hot and which are cold. It makes intelligent eviction decisions based on request activity. When those cold blocks are needed again (e.g., a user sends a follow-up message), vLLM copies them back from CPU RAM to VRAM.</p>



<p>This CPU memory pool lives <strong>inside the cgroup</strong>. It counts toward your pod&#8217;s <code>memory.max</code> limit.</p>



<p><strong>L3: Kubernetes Swap (NVMe — 3-5 GB/s / HDD — 0.1-0.2 GB/s)</strong></p>



<p>When total CPU RAM usage — vLLM&#8217;s runtime overhead plus the L2 KV cache pool plus everything else the process allocates — exceeds the cgroup memory limit, the Linux kernel starts evicting pages from CPU RAM to the swap device. This is Kubernetes swap. It&#8217;s configured at the node level through the kubelet&#8217;s <code>memorySwap.swapBehavior</code> setting and enforced per-container through <code>memory.swap.max</code> in the cgroup.</p>



<p>Unlike vLLM&#8217;s PagedAttention, the kernel has no understanding of what it&#8217;s swapping. It uses LRU (Least Recently Used) heuristics on anonymous memory pages. It doesn&#8217;t know whether it&#8217;s evicting a critical attention buffer that the GPU needs in 50 milliseconds, or a cold scheduler data structure that won&#8217;t be touched for another minute. That&#8217;s why L3 swap introduces latency unpredictably.</p>



<p><strong>The full chain:</strong></p>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>Layer</td><td>What</td><td>Bandwidth</td><td>Managed by</td><td>Intelligence</td></tr><tr><td><strong>L1</strong></td><td>KV cache in GPU VRAM</td><td>2 TB/s (HBM)</td><td>vLLM PagedAttention</td><td>Knows which blocks are hot/cold</td></tr><tr><td><strong>L2</strong></td><td>KV cache evicted to CPU RAM</td><td>64 GB/s (PCIe)</td><td>vLLM PagedAttention</td><td>Application-level, intelligent</td></tr><tr><td><strong>L3</strong></td><td>Anonymous memory evicted to disk</td><td>3-5 GB/s (NVMe)</td><td>Linux kernel</td><td>LRU heuristics, blind to workload</td></tr></tbody></table></figure>
</div>


<p>The design principle: always prefer higher-level swap. When VRAM fills, you want PagedAttention handling the overflow, because it understands transformer workloads. Kubernetes swap is what happens when L2 is also exhausted, it’s the kernel-level emergency catch. Application-level memory management handles the common case, and Kubernetes swap handles the failure case. You need both layers.</p>



<p>The performance degradation I measured in the Mistral-7B test (GPU utilisation dropping to 73.5% with 195,000 page-ins over the test window) is what happens when L3 activates on the inference hot path. The GPU wasn&#8217;t slow because of a model problem or a scheduling issue. It was waiting on the kernel to fault anonymous pages back from NVMe. That&#8217;s the cost of blind, kernel-level swap on a latency-sensitive workload.</p>



<h2 class="wp-block-heading"><strong>How Linux Actually Manages Memory Under Pressure</strong></h2>



<p>Understanding why Kubernetes swap behaves the way it does on GPU nodes requires looking one level deeper — at how the Linux kernel categorises, loads, and reclaims memory. This is the context most Kubernetes swap tutorials skip, and it&#8217;s the reason that two pods with identical swap entitlements can have completely different outcomes.</p>



<h3 class="wp-block-heading"><strong>Anonymous vs File-Backed Memory</strong></h3>



<p>The kernel divides all memory pages into two categories, and the distinction determines what happens when the cgroup hits its limit.</p>



<p><strong>File-backed pages</strong> are memory loaded from a file on disk — shared libraries like <code>libpython3.so</code>, the CUDA runtime (<code>libcudart.so</code>), or model weight files loaded via <code>mmap()</code>. Each page tracks which file it came from and at what byte offset. The key property: the kernel can evict these pages by simply dropping them from RAM. The original data is still on disk in the file. If the process needs that data again, the kernel re-reads that specific 4KB page from the file. No swap device involved. No write I/O. This makes file-backed eviction essentially free.</p>



<p><strong>Anonymous pages</strong> are everything the process allocates at runtime that has no file backing. For a vLLM process, this includes the Python interpreter&#8217;s heap, PyTorch tensor allocations, the tokeniser vocabulary tables, the request scheduler state, KV cache blocks that have been evicted from VRAM into the CPU swap pool, and output token assembly buffers. When the kernel needs to evict anonymous pages, the only place to put them is the swap device — because there is no original file to write them back to. This requires disk I/O, and it&#8217;s what shows up in the <code>pswpin</code>/<code>pswpout</code> counters in <code>/proc/vmstat</code>.</p>



<p>This distinction is why model weight storage format matters for memory pressure behaviour. The <code>.safetensors</code> format (now the default for HuggingFace models) is specifically designed for <code>mmap()</code>: it has a small JSON header describing each tensor&#8217;s name, dtype, shape, and byte offset, followed by the raw tensor data at those known offsets. PyTorch can map the file into virtual memory and access weight data directly without copying it into the heap. Those weight pages are file-backed — cheap to evict.</p>



<p>Compare that to the older pickle-based <code>.bin</code> format, which requires a full <code>read()</code> of the file followed by deserialisation into Python objects. The deserialised weights end up in anonymous memory (heap-allocated PyTorch tensors). Those are expensive to evict — they have to go to swap.</p>



<p>For GPU inference, this distinction is largely academic because the weights are copied to VRAM during model loading and the CPU-side copies can be released. But for CPU-only inference (like the phi-2 test that produced the 8.19x degradation), the storage format directly affects how much anonymous memory the process carries — and therefore how much swap pressure the kernel faces.</p>



<h3 class="wp-block-heading"><strong>Demand Paging: How the Kernel Loads Files</strong></h3>



<p>The kernel doesn&#8217;t load entire files into RAM. When a process calls <code>mmap()</code> on a model file, the kernel creates a Virtual Memory Area (VMA) — a mapping from a range of virtual addresses to byte offsets in the file. No data is loaded at this point.</p>



<p>When the process first accesses a specific virtual address in that range, the CPU&#8217;s memory management unit (MMU) finds no physical page mapped and triggers a page fault. The kernel handles the fault by calculating which 4KB chunk of the file corresponds to that virtual address — simple arithmetic based on the VMA&#8217;s start address and file offset — then reads that single page from disk into a free RAM page and updates the page table.</p>



<p>This is called demand paging: pages load only when the application actually touches them. A 14GB model file doesn&#8217;t require 14GB of free RAM to <code>mmap()</code> — the kernel loads individual 4KB pages as needed, and can evict cold pages back to the file at any time.</p>



<p>The same mechanism applies to shared libraries. When <code>libpython3.so</code> is loaded, the dynamic linker reads the ELF header (which describes where each section of the binary lives within the file), then creates separate <code>mmap()</code> regions for the executable code, read-only data, and writable data sections. Individual 4KB pages of code are loaded on first execution, not upfront.</p>



<h3 class="wp-block-heading"><strong>The Kernel Reclaim Sequence</strong></h3>



<p>When a container&#8217;s memory usage approaches its cgroup limit (<code>memory.max</code>), the kernel doesn&#8217;t immediately kill the process. It enters a reclaim sequence — a prioritised set of attempts to free memory before resorting to the OOM killer.</p>



<p>The sequence, in order:</p>


<div class="wp-block-list--wrapper">
<ol class="wp-block-list">
<li><strong>Drop clean file-backed pages</strong> — free. These pages are identical to their on-disk source. The kernel simply removes them from RAM. No I/O required. If the process needs them again, the kernel re-reads from the file (a page fault, but no swap I/O).</li>



<li><strong>Write back dirty file-backed pages</strong> — cheap. These are file-backed pages that have been modified in RAM (for example, a log file being appended to). The kernel writes them back to their source file, then drops them. Sequential I/O, relatively fast.</li>



<li><strong>Swap anonymous pages to the swap device</strong> — expensive. The kernel picks anonymous pages using LRU heuristics, writes them to the swap partition or swapfile on NVMe (or HDD), and frees the RAM. When the process needs those pages again, the kernel reads them back from the swap device (a page fault with disk I/O).</li>



<li><strong>OOM kill</strong> — last resort. Only after all reclaim options are exhausted and the kernel still cannot free enough memory to satisfy the allocation does the OOM killer terminate the process.</li>
</ol>
</div>


<p>This means the effective memory ceiling for a container is not just <code>memory.max</code>. It&#8217;s:</p>



<pre class="wp-block-code"><code>effective_ceiling = memory.max + memory.swap.max</code></pre>



<p>The cgroup limit is a reclaim trigger, not a kill line. The kernel will use every available byte of swap entitlement before giving up.</p>



<p>This is exactly what the test data showed in one of the constrained-memory tests I&#8217;ll detail in the next section. The pod had a 512Mi cgroup limit and ~196MB of swap entitlement. vLLM&#8217;s runtime needed approximately 1GB. The kernel recorded 145,000 page-outs — it was actively writing anonymous pages to the swap device, trying to keep the process alive. But 196MB of swap couldn&#8217;t cover the ~500MB deficit between what vLLM needed and what the cgroup allowed. After exhausting all reclaim options, the OOM killer fired.</p>



<p>The kernel didn&#8217;t fail silently, it fought for the pod but didn&#8217;t have enough swap budget to win.</p>



<h2 class="wp-block-heading"><strong>Four Configurations, Same Hardware: What the Data Actually Shows</strong></h2>



<p>I ran four swap configurations on identical hardware to isolate exactly what determines whether a pod survives memory pressure on a GPU node. The setup:</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>kubeadm v1.29 on a GCE instance with an NVIDIA L4 (24GB VRAM), 64GB RAM, 48GB NVMe swapfile, running Mistral-7B-Instruct via vLLM.</li>



<li>The memory request was 256Mi across all four scenarios</li>



<li>The only variables were the swap mode and the cgroup memory limit.</li>
</ul>
</div>

<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table><tbody><tr><td>Scenario</td><td>Swap Mode</td><td>Limit</td><td>memory.swap.max</td><td>Outcome</td><td>GPU</td><td>Errors</td></tr><tr><td><strong>A</strong></td><td>NoSwap</td><td>512Mi</td><td>0</td><td>OOMKilled</td><td>0%</td><td>100%</td></tr><tr><td><strong>B</strong></td><td>UnlimitedSwap</td><td>512Mi</td><td>unlimited</td><td>Degraded alive</td><td>73.5%</td><td>0% (P3)</td></tr><tr><td><strong>C</strong></td><td>LimitedSwap</td><td>512Mi</td><td>~196MB</td><td>OOMKilled</td><td>0%</td><td>100%</td></tr><tr><td><strong>D</strong></td><td>LimitedSwap</td><td>2Gi</td><td>~196MB</td><td>Pod survived</td><td>stable</td><td>0%</td></tr></tbody></table></figure>
</div>


<p><strong>Scenario A</strong> is the baseline most Kubernetes operators experience today: swap disabled, pod exceeds its memory limit, and OOM killer fires immediately. This leads to CrashLoopBackOff with 100% request failure, and the GPU stays idle. This is the default failure mode for any GPU inference pod under memory pressure.</p>



<p><strong>Scenario B</strong> is the mode Kubernetes removed in v1.30. UnlimitedSwap lets the kernel swap without any per-container ceiling. The pod survived — zero errors during sustained load — but at significant cost. GPU utilisation dropped from 94% to 73.5% as the GPU waited for swapped-in pages on every inference batch. The kernel recorded 1.6GB of swap usage with 195,000 page-ins and 168,000 page-outs. This is why UnlimitedSwap was removed: it preserves liveness but creates unpredictable, unbounded performance degradation, and a single pod can exhaust the node&#8217;s entire swap device.</p>



<p><strong>Scenario C</strong> is where the interesting finding starts. LimitedSwap with a 512Mi cgroup limit. The swap entitlement from the formula — <code>(256Mi / 64GB) × 48GB</code> — works out to approximately 196MB. vLLM&#8217;s CPU-side runtime needs roughly 1GB. The cgroup allows 512Mi. That&#8217;s a ~500MB deficit, so the kernel tried to compensate. I measured 145,000 page-outs, meaning it actively wrote anonymous pages to the swap device. But 196MB of swap headroom against a 500MB deficit isn&#8217;t enough. The OOM killer fired after exhausting all reclaim options.</p>



<p><strong>Scenario D</strong> used the same LimitedSwap mode and the same 256Mi request, which means the same ~196MB swap entitlement as Scenario C. The only change was the cgroup memory limit: 2Gi instead of 512Mi. The pod survived with zero errors and zero restarts. The 2Gi limit gave vLLM enough room to fit its ~1GB runtime overhead without exceeding the cgroup at all. Swap was available as a safety net but wasn&#8217;t needed at steady-state traffic.</p>



<h3 class="wp-block-heading"><strong>Two Levers, Not One</strong></h3>



<p>This correction reveals the actual model for sizing Kubernetes swap on GPU nodes. There are two independent levers:</p>



<p><strong>Lever 1: Cgroup memory limit.</strong> This must cover your workload&#8217;s steady-state runtime — everything the process needs to run under normal traffic. For vLLM, that&#8217;s the Python runtime, PyTorch CPU-side state, tokeniser, scheduler, and the <code>--swap-space</code> KV cache buffer. If your limit is too small (Scenario C), the kernel will try to swap, but if the deficit exceeds the swap entitlement, OOMKill follows anyway.</p>



<p><strong>Lever 2: Swap entitlement.</strong> This covers burst spikes above the limit — the traffic surges that temporarily push memory usage beyond your cgroup ceiling. The entitlement is determined by your <code>memory.request</code> via the formula, so getting your request right matters. This is where tools like <a href="https://scaleops.com/">ScaleOps</a> provide direct value: by observing actual workload memory consumption over time, they can set requests that generate a swap entitlement matched to your real burst profile — rather than a number someone guessed during a sprint planning meeting.</p>



<p>If Scenario D didn&#8217;t need swap at steady state, why configure it at all? Because D was running under normal traffic conditions. In a separate test campaign on GKE, I tested the burst scenario: traffic ramped to 3x, pushing memory above a correctly-sized cgroup limit. With correctly-sized swap, the pod survived every overflow level. Without swap, OOMKilled on every run. With mis-sized swap (only 33MB of entitlement against a large burst), the pod OOMKilled and took 60% longer to fail than the no-swap baseline — the kernel spent time writing to swap before ultimately running out of budget and killing the process anyway.</p>



<p>Undersized swap is worse than no swap. It adds I/O overhead to the failure path without preventing the failure.</p>



<h3 class="wp-block-heading"><strong>When Kubernetes Swap Destroys Performance</strong></h3>



<p>Not every workload benefits from the swap safety net. Two categories where enabling swap is the wrong choice:</p>



<p><strong>Latency-sensitive inference.</strong> I tested Whisper (speech recognition) under constrained memory with swap active. The kernel swapped 503MB of process memory with 82,000 page-ins during the test window. For any workload with a sub-100ms SLA, that swap I/O is an uncontrolled variable in the latency path — every page fault adds milliseconds that break real-time contracts. The safe configuration: set <code>requests</code> equal to <code>limits</code> to get Guaranteed QoS, which sets <code>memory.swap.max = 0</code> regardless of the node&#8217;s swap configuration.</p>



<p><strong>Training workloads with checkpoint saves.</strong> A LoRA fine-tuning run under constrained memory showed 1.4GB of swap with 372,000 page-outs. The wall-clock overhead was only 1.01x — NVMe absorbed the swap I/O at this scale. But that was a short training job on a small model. At production scale with multi-hour runs and periodic checkpoint saves, every checkpoint serialises the full model state, triggering a burst of swap I/O that compounds with model size and training duration. The recommendation: NVMe swap may be acceptable for training, but benchmark on your actual workload before committing to it in production.</p>



<h2 class="wp-block-heading"><strong>How to Size Memory for vLLM with Kubernetes Swap</strong></h2>



<p>The test data establishes the principles. This section translates them into a practical sizing approach for vLLM deployments running with LimitedSwap.</p>



<h3 class="wp-block-heading"><strong>What Lives in CPU RAM</strong></h3>



<p>On a GPU node, the model weights live in VRAM — outside the cgroup. Everything else vLLM needs lives in CPU RAM, inside the cgroup. The components and their approximate footprints:</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li><strong>Python runtime + PyTorch + vLLM framework code:</strong> 300–500MB</li>



<li><strong>Tokeniser vocabulary and encoding tables:</strong> 50–100MB</li>



<li><strong>Request scheduler, batch manager, routing state:</strong> 50–100MB</li>



<li><strong>vLLM CPU swap pool (</strong><code>-swap-space</code><strong>):</strong> whatever you configure (default 4GB) — this is the L2 landing zone for evicted KV cache blocks</li>



<li><strong>Output token assembly buffers:</strong> scales with concurrent requests</li>
</ul>
</div>


<p>For a typical vLLM deployment with the default <code>--swap-space 4</code>, steady-state CPU memory consumption is approximately 5–6GB. This is the number your cgroup limit must cover.</p>



<h3 class="wp-block-heading"><strong>The Sizing Formula</strong></h3>



<figure class="wp-block-image size-large"><img alt="" class="wp-image-8909" height="464" src="https://scaleops.com/content/uploads/2026/04/image-2-1024x464.png" width="1024" /></figure>



<p>Setting request lower than limit gives the pod Burstable QoS — the only class that receives proportional swap under LimitedSwap. The swap entitlement that results from this configuration depends on your node:</p>



<pre class="wp-block-code"><code>swap_entitlement = (6Gi / node_capacity) × node_swap

# Example: 64GB node with 16GB swap
swap_entitlement = (6 / 64) × 16 = 1.5GB</code></pre>



<p>That 1.5GB of swap entitlement covers burst spikes that push memory above the 10Gi cgroup limit:</p>


<div class="wp-block-list--wrapper">
<ul class="wp-block-list">
<li>KV cache overflow during traffic surges</li>



<li>Temporary allocation spikes during batch processing</li>



<li>Concurrent model loading in multi-model setups.</li>
</ul>
</div>


<h3 class="wp-block-heading"><strong>Swap Device Sizing</strong></h3>



<p>The node-level swap device affects all Burstable pods proportionally. A practical starting point: <strong>25% of node RAM on local NVMe</strong>. For a 64GB node, that&#8217;s 16GB of swap. For the 48GB we used in testing (75% of RAM), that&#8217;s generous but appropriate for a dedicated GPU inference node where swap headroom matters more than on a general-purpose worker.</p>



<p>Local NVMe only. Not HDD (1000x slower), not network-attached storage (latency variance). The performance difference between NVMe and HDD for swap is the difference between a useful safety net and an unusable one.</p>



<h3 class="wp-block-heading"><strong>Monitoring Kubernetes Swap</strong></h3>



<p>Three Prometheus metrics to watch once swap is enabled:</p>



<p><code>container_memory_swap</code> — per-container swap usage in bytes. Alert if any container exceeds 50% of its <code>memory.swap.max</code> for more than 5 minutes. Brief spikes are the expected use case. Sustained swap usage means you have a resource planning problem, not a burst.</p>



<p><code>node_memory_SwapFree</code> — remaining swap capacity on the node. Alert below 20%. At that point, the next pod that spikes will OOMKill anyway because the swap device is nearly exhausted.</p>



<p><code>container_swap_io_count</code> — swap I/O operations per second. This is the thrashing detector. In the Mistral-7B test, I measured 195,000 page-ins in a single test window. High sustained swap I/O means pages are being constantly faulted in and out — alert on sustained rates above your NVMe&#8217;s rated IOPS.</p>



<h3 class="wp-block-heading"><strong>The Decision Framework</strong></h3>



<p>Not every workload should enable Kubernetes swap. The decision depends on your latency tolerance and memory access pattern:</p>


<div class="wp-block-table--wrapper">
<figure class="wp-block-table"><table class="has-fixed-layout"><tbody><tr><td>Workload</td><td>K8s Swap?</td><td>QoS Class</td><td>Rationale</td></tr><tr><td>Multi-model serving (KServe)</td><td>LimitedSwap</td><td>Burstable</td><td>Brief concurrent spikes overflow safely</td></tr><tr><td>Burst traffic, SLA &gt; 200ms</td><td>LimitedSwap</td><td>Burstable</td><td>Safety net for KV cache spikes</td></tr><tr><td>Real-time inference, SLA &lt; 100ms</td><td>NoSwap</td><td>Guaranteed</td><td>Swap I/O breaks real-time contracts</td></tr><tr><td>Training (checkpoint saves)</td><td>LimitedSwap (NVMe only)</td><td>Burstable</td><td>Benchmark at your scale first</td></tr><tr><td>Databases, etcd, control plane</td><td>NoSwap</td><td>Guaranteed</td><td>Data integrity, non-negotiable</td></tr></tbody></table></figure>
</div>


<p>The pattern: if your workload can tolerate brief latency spikes during memory pressure, LimitedSwap is a cost-effective safety net that prevents OOMKills. If your workload has strict latency contracts, set requests equal to limits for Guaranteed QoS — <code>memory.swap.max</code> becomes zero regardless of node configuration, and the pod is fully isolated from swap.</p>



<h3 class="wp-block-heading"><strong>Automating the Two Levers</strong></h3>



<p>The sizing approach above works when you know your workload&#8217;s memory profile. In practice, that profile changes — new model versions, shifting traffic patterns, evolving batch sizes. Getting the request wrong means your swap entitlement is wrong. Getting the limit wrong means your cgroup headroom is wrong. Either misconfiguration leads to the outcomes I measured: OOMKills, degraded performance, or swap overhead on the failure path.</p>



<p>This is where <a href="https://scaleops.com/">ScaleOps</a> fits into the swap equation. ScaleOps observes actual workload memory consumption over time and continuously adjusts both requests and limits based on measured behaviour. For the two-lever model: it sets requests to generate a swap entitlement that matches your real burst profile, and sets limits to provide cgroup headroom that covers your actual steady-state runtime — not a number someone estimated during initial deployment.</p>



<p>The principle holds regardless of tooling: <strong>size your cgroup limit for what you can measure. Size your swap entitlement for what you can&#8217;t predict.</strong> Whether you do that manually with Prometheus dashboards or automatically with a rightsizing platform, the two levers need to be calibrated to your workload.</p>



<h2 class="wp-block-heading"><strong>Conclusion: From Manual Sizing to Continuous Optimization</strong></h2>



<p>Kubernetes swap for AI workloads is no longer a question of &#8220;should we enable it?&#8221; The data from 30+ test iterations across two clusters answers that definitively: it depends on the workload. The more useful question — and the one this article set out to answer — is &#8220;where, how much, and for which workloads?&#8221;</p>



<p>The core model is two levers. Your cgroup memory limit must cover your workload&#8217;s steady-state runtime needs — the Python runtime, the tokeniser, the scheduler, and the vLLM CPU swap pool. Your swap entitlement, determined by the LimitedSwap formula based on your memory request, covers the burst spikes you can&#8217;t predict. Misconfigure the first lever and the pod dies regardless of swap (Scenario C). Ignore the second lever and a traffic spike takes down your inference service at the worst possible moment.</p>



<p>The sizing guide in this article gives you concrete numbers to start with. But workload memory profiles are not static — model versions change, traffic patterns shift, batch sizes evolve. What was a correctly-sized request last month may be an undersized request after a model update. This is where manual sizing hits its limits and continuous, observation-based resource management becomes essential. <a href="https://scaleops.com/">ScaleOps</a> automates both levers: it observes actual memory consumption over time and continuously adjusts requests (which determine your swap entitlement) and limits (which determine your cgroup headroom) based on measured workload behaviour. That&#8217;s the production path from the principles in this article to a system that adapts as your workloads change.</p>



<h3 class="wp-block-heading"><strong>See Your Swap Sizing Opportunities</strong></h3>



<p>The two-lever model in this article — cgroup limits for steady-state, swap entitlement for burst — depends on knowing your workload&#8217;s actual memory profile. Guessing those numbers is how you end up in Scenario C.</p>



<p>ScaleOps installs in minutes and starts in read-only mode — no changes to your cluster, no risk. It analyses your GPU workloads&#8217; real memory consumption and shows you exactly where your requests and limits are misconfigured: pods that would benefit from swap headroom, pods that need tighter Guaranteed QoS isolation, and the cost savings from rightsizing nodes you&#8217;re currently overprovisioning for memory safety margins.</p>



<p><a href="https://try.scaleops.com/"><strong>Try ScaleOps free →</strong></a> See your optimization opportunities before changing a single pod spec.</p>



<p><a href="#book-a-demo"><strong>Book a demo →</strong></a> Walk through your cluster&#8217;s memory profile with our team and see how the two-lever model applies to your GPU workloads.</p>


<div class="b-faq-section has-inner-container align wp-block-faq-section">
  <div class="b-faq-section__items flex flex-col gap-4">
    <div class="faq-section-innerblocks">

<h2 class="wp-block-heading">Frequently Asked Questions</h2>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading">What is the Kubernetes LimitedSwap formula?</h3>



<p>The kubelet sets <code>memory.swap.max</code> per container using the formula: <code>swap_limit = (container_memory_request / node_total_memory) × total_pods_swap_available</code>. The numerator is the container&#8217;s memory request — not the limit. I verified this byte-exact on GKE: a 128MiB request on a 16GB-capacity node with 4GB swap produced <code>memory.swap.max</code> of exactly 34,365,440 bytes, page-rounded to 4KiB boundaries.</p>

</div>
</div>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading">Does Kubernetes swap work with GPU workloads?</h3>



<p>Yes, but the interaction is specific. GPU VRAM is outside the Kubernetes cgroup — <code>memory.max</code> and <code>memory.swap.max</code> only apply to CPU RAM. For inference workloads running vLLM, the model weights live in VRAM while the CPU-side runtime (tokeniser, scheduler, KV cache eviction pool, output buffers) lives inside the cgroup. Kubernetes swap catches overflow of that CPU-side memory, not GPU memory.</p>

</div>
</div>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading">What is the difference between vLLM -swap-space and Kubernetes swap?</h3>



<p>They operate at different tiers. vLLM&#8217;s <code>--swap-space</code> allocates CPU RAM as a landing zone for KV cache blocks evicted from GPU VRAM — this is application-level swap (L2) with intelligent eviction based on request activity. Kubernetes swap is kernel-level swap (L3) that pages anonymous memory from CPU RAM to the NVMe swap device using LRU heuristics, with no awareness of what the application considers important. vLLM&#8217;s swap-space buffer itself lives inside the cgroup, so if it plus the runtime overhead exceeds the cgroup limit, Kubernetes swap catches the overflow.</p>

</div>
</div>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading">How do I prevent OOMKill on GPU inference pods?</h3>



<p>Two levers: first, set your cgroup memory limit high enough to cover your workload&#8217;s steady-state CPU memory needs (runtime overhead plus vLLM&#8217;s <code>--swap-space pool</code>). Second, ensure your memory request generates a swap entitlement — via the LimitedSwap formula — large enough to absorb burst traffic spikes above that limit. If the first lever is wrong (limit too small), swap cannot compensate. If the second lever is missing (no swap configured), traffic spikes trigger OOMKill.</p>

</div>
</div>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading">Is Kubernetes swap bad for performance?</h3>



<p>It depends on the workload. For latency-sensitive inference with sub-100ms SLAs (speech recognition, real-time translation), swap I/O introduces unpredictable page fault latency that breaks real-time contracts — use Guaranteed QoS (<code>requests = limits</code>) to disable swap for those pods. For workloads that tolerate brief latency spikes (chatbots, batch inference, content generation with SLAs above 200ms), LimitedSwap provides a cost-effective safety net that prevents OOMKills during traffic surges.</p>

</div>
</div>


<div class="b-faq-item overflow-hidden ">
  <div class="b-faq-item__inner">

<h3 class="wp-block-heading">Should I use NVMe or HDD for the Kubernetes swap device?</h3>



<p>NVMe only. The access latency difference is approximately 1000x: NVMe provides 3–5 GB/s throughput with 3–5 microsecond access times, while HDD provides 0.1–0.2 GB/s with 10–15 millisecond access times. On HDD, every page fault during swap reclaim blocks the process for milliseconds — long enough to cascade into request timeouts on an inference workload. NVMe swap introduces measurable but bounded latency; HDD swap introduces catastrophic latency.</p>

</div>
</div>

</div>
  </div>
</div>




<p></p>
<p>The post <a href="https://scaleops.com/blog/kubernetes-swap-gpu-workload-sizing/">Kubernetes Swap for AI Inference: LimitedSwap, Memory Hierarchy, and GPU Workload Sizing</a> appeared first on <a href="https://scaleops.com">ScaleOps</a>.</p>
