---
title: "The Real Cost of Always-On AI Agents on Kubernetes (and How to Kill Idle GPU Burn)"
url: "https://scaleops.com/blog/the-real-cost-of-always-on-ai-agents-on-kubernetes-and-how-to-kill-idle-gpu-burn/"
date: "2026-08-21"
author: "Nic Vermandé"
feed_url: "https://scaleops.com/feed/"
---
Running AI agents on Kubernetes means hosting long-lived, stateful agent processes as workloads on a cluster. In most production deployments the agent process and the GPU are in different pods. The agent itself is orchestration logic, tool definitions, and session state, usually built with a framework like LangChain or LlamaIndex, and it is CPU-only.
