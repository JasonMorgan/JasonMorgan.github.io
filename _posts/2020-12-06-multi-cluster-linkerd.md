---
title:  "Multi Cluster Service Mesh on TKG with Linkerd"
categories: [kubernetes]
tags: [linkerd,kubernetes,k8s,mesh]
published: false
---

Hey folks! Thanks for stopping by! Today I'm going to dive into using the [linkerd](https://linkerd.io/) service mesh to route traffic between 2 [Tanzu Kubernetes Grid](https://tanzu.vmware.com/kubernetes-grid), or tkg, clusters.

## The Set Up

What we're using:

* Tanzu Mission Control
  * To manage our tkg clusters
* The tmc cli (Optional component)
  * To connect our clusters to Tanzu Mission Control
  * You can download this from your Tanzu Mission Control portal
* The tkg cli
  * to create, manage, and scale our k8s clusters
  * you can download it [here]()
* The linkerd cli
  * to do all our linkerd work
* Our podinfo app
