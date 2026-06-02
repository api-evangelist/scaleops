---
title: "The Kubernetes Scheduler: How Pod Placement, Bin Packing, and Autoscalers Actually Fit Together"
url: "https://scaleops.com/blog/kubernetes-scheduler/"
date: "2026-05-25"
author: "Nic Vermandé"
feed_url: "https://scaleops.com/feed/"
---
Most production Kubernetes clusters look 30–40% utilized while the cluster autoscaler refuses to scale down. The Pods are placed correctly, the Nodes show plenty of headroom, but every attempt to consolidate fails. The reflex is to blame the autoscaler, the workloads, or the resource requests.
