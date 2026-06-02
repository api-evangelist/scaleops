---
title: "Kubernetes GPU Sharing: MIG vs. MPS vs. Time-Slicing Explained"
url: "https://scaleops.com/blog/kubernetes-gpu-sharing/"
date: "2026-05-26"
author: "Adi Steiner"
feed_url: "https://scaleops.com/feed/"
---
GPU sharing in Kubernetes lets multiple pods use the same physical GPU, rather than forcing each pod to reserve a full device. The three main NVIDIA-supported options are time-slicing, Multi-Process Service (MPS), and Multi-Instance GPU (MIG). Each solves that problem differently.
