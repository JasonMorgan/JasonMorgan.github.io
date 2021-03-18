---
title:  "Gitops with Linkerd, Flux, and Flagger"
categories: [gitops ]
tags: [gitops, "service mesh", linkerd, flux, kubernetes]
published: false
---

Hey folks! Thanks for stopping by! Today I'm going to walk through using [Flux](https://github.com/fluxcd/flux2), [Flagger](https://github.com/fluxcd/flagger), and [Linkerd](https://github.com/linkerd/linkerd2) to automate provisioning a kubernetes cluster and deploying apps. I've been using Kubernetes for awhile now and one problem that I run into again and again is how can I provision a new cluster, lay down a runtime environment, and then deploy some apps in a consistent, and ideally automated way. If that interestes you then you're in the right place. The rest of this article is going to dive into doing just that. We're also going to see how to use a progressive delivery tool like flagger to gate our application rollouts.

You can find me here: [twitter](https://twitter.com/RJasonMorgan) or [Linkedin](https://www.linkedin.com/in/jasonmorgan2/).

Want to walk through getting up and running in your own environment? Follow along with the steps below.

Thanks!

## The Set Up

What we're using:

* [k3d](https://k3d.io/)
  * v4.3.0
* kubectl
  * v1.19.7
* linkerd
  * stable-2.10.0
* flux
  * 0.9.1

## Getting Started

So we're going to start off deploying a local cluster, you can use any tooling you like, minikube, docker desktop, or any number of others will work if you're deploying to your computer. If you're working with an IaaS or one of the *ks providers, AKS, GKE, EKS, etc, please go ahead and get your cluster setup with whatever tooling you prefer. This tutorial is going to use k3d because I like it and it makes it easy to expose an ingress locally.

By the end of this article you should be familiar with what flagger, flux, and linkerd are. You should also be comfortable using a gitops flow to configure a cluster and deploy an application. With Luck you'll also have a sense of how easy it can be to get up and running with a service mesh and leveraging it for progressive delivery.

If you want to skip the explanations click [here](###Get) or just scroll down to the next section.

### Understanding the Tools and Concepts

So what is GitOps? What's a service mesh? And what is progressive delivery?

#### GitOps

In addition to something every marketer in the cloud native space is talking about, GitOps refers to a concept that, I believe, was pioneered by the folks over at [Weave](weave.works). I read about it for the first time [here](https://www.weave.works/blog/gitops-operations-by-pull-request) and I loved the concept so much I ported it to Cloud Foundry and loved it. Today GitOps is generally associated with Kubernetes for a few reasons, one of the top ones being that everything you do in kubernetes can be defined in a yaml manifest. GitOps, is the name for holding your Kubernetes config files in a git repo and programatically applying them to your Kubernetes cluster. There are a few models and patterns associated with it so I'm not really going to dive in. For those CNCF enthusiasts there are currently 2 CNCF projects that aim to deliver GitOps to all of us, [Argo](https://argoproj.github.io/) and Flux.

#### Service Mesh

Istio, or Envoy, or something every kubernetes company wants to sell you. Not really though, a service mesh is a relatively complex name for a fairly simple concept. You can read a whole [Meshifesto](https://buoyant.io/service-mesh-manifesto/) is you really want to understand it or you can take this definition for what it's worth:

```text
A service mesh is a tool that controls the interactions between your applications. It works by inserting proxies between all in mesh communications, we call the meshed proxies the data plane. The data plane does stuff with your inter app communications, stuff like applying policy, making routing decisions, and surfacing data about what the apps are doing. The data plane takes instructions from a separate control plane. 
```

I'm a developer evangelist for Buoyant so I'm going to talk about Linkerd. If you want to know why I think Linkerd is awesome, and way better than Istio go [check it out](https://linkerd.io/2.10/getting-started/). The TL;DR is that Linkerd is small, fast, and works right out the gate.

#### Progressive Delivery

This one is all about how do you handle deploying a new version of an app then shift traffic to it while your old version is still out there. Ideally using small increments of traffic that let you see how that new version is doing.

### Get me to it

Cool, that was a lot of words. Now lets do some stuff.

#### Deploy a cluster

So if you're using anything other than k3d skip this part because you should already have it working.

```bash
k3d cluster create gitops -a 3  -p "8081:80@loadbalancer"
```

That command will deploy a k8s cluster on your local machine, it will also point port 8081 and the clusters ingress. You won't need that but I like to be able to use an ingress so I added it.

### Install Flux

It seems weird to me but the only way I've seen to deploy flux is the flux cli ¯\_(ツ)_/¯.
Still it's pretty simple, you can run flux install if you're doing it to try it out `flux install` works great. If you're going to take it into your prod systems the makers recommend you use `flux bootstrap`. Flux bootstrap does a lot of stuff including adding a new key to your github repo and potentially creating a github repo to manage your cluster.

I don't want to mess with that so I use flux install:

```bash
# Check that you can in fact install flux
flux check --pre

# Install flux
flux install

# Check flux installed correctly
flux check
```

Flux install will get you 4 new pods in the `flux-system` namespace. It also has the handy side effect of blocking your terminal till flux is ready just in case you're using it as part of a script.

```text
flux-system   source-controller-798bd8fffb-4js84        1/1     Running     0          29s
flux-system   kustomize-controller-84fdd79d5b-b58d9     1/1     Running     0          29s
flux-system   helm-controller-7fd55b8c9f-hzvck          1/1     Running     0          29s
flux-system   notification-controller-d9464dbdf-khdqq   1/1     Running     0          29s
```

### Gitops Away!

Now that we have

## Wrap Up

Tell em what you told them. And what I wanted them to learn

Thanks so much for reading and I'd love to hear any feedback you have,

I'm: [twitter](https://twitter.com/RJasonMorgan) or [Linkedin](https://www.linkedin.com/in/jasonmorgan2/).

Jason
