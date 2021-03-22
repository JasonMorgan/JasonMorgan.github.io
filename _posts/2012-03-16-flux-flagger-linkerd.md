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
* [linkerd](https://linkerd.io)
  * stable-2.10.0
* [flux](https://github.com/fluxcd/flux2)
  * 0.9.1

## Contents

- [The Set Up](#the-set-up)
- [Contents](#contents)
- [Getting Started](#getting-started)
- [Understanding the Tools and Concepts](#understanding-the-tools-and-concepts)
  - [GitOps](#gitops)
  - [Service Mesh](#service-mesh)
  - [Progressive Delivery](#progressive-delivery)
- [Get me to it](#get-me-to-it)
  - [Deploy a cluster](#deploy-a-cluster)
  - [Install Flux](#install-flux)
  - [Gitops Away](#gitops-away)
    - [Git to it](#git-to-it)
      - [Creating a runtime repo](#creating-a-runtime-repo)
      - [Tell Flux to use our repo](#tell-flux-to-use-our-repo)
    - [Runtime Config](#runtime-config)
    - [Deploy our App](#deploy-our-app)
  - [Canary Deployments](#canary-deployments)
- [Wrap Up](#wrap-up)

## Getting Started

So we're going to start off deploying a local cluster, you can use any tooling you like, minikube, docker desktop, or any number of others will work if you're deploying to your computer. If you're working with an IaaS or one of the *ks providers, AKS, GKE, EKS, etc, please go ahead and get your cluster setup with whatever tooling you prefer. This tutorial is going to use k3d because I like it and it makes it easy to expose an ingress locally.

By the end of this article you should be familiar with what flagger, flux, and linkerd are. You should also be comfortable using a gitops flow to configure a cluster and deploy an application. With Luck you'll also have a sense of how easy it can be to get up and running with a service mesh and leveraging it for progressive delivery.

If you want to skip the explanations click [here](##get-me-to-it) or just scroll down to the next section.

## Understanding the Tools and Concepts

So what is GitOps? What's a service mesh? And what is progressive delivery?

### GitOps

In addition to something every marketer in the cloud native space is talking about, GitOps refers to a concept that, I believe, was pioneered by the folks over at [Weave](weave.works). I read about it for the first time [here](https://www.weave.works/blog/gitops-operations-by-pull-request) and I loved the concept so much I ported it to Cloud Foundry and loved it. Today GitOps is generally associated with Kubernetes for a few reasons, one of the top ones being that everything you do in kubernetes can be defined in a yaml manifest. GitOps, is the name for holding your Kubernetes config files in a git repo and programatically applying them to your Kubernetes cluster. There are a few models and patterns associated with it so I'm not really going to dive in. For those CNCF enthusiasts there are currently 2 CNCF projects that aim to deliver GitOps to all of us, [Argo](https://argoproj.github.io/) and Flux.

### Service Mesh

Istio, or Envoy, or something every kubernetes company wants to sell you. Not really though, a service mesh is a relatively complex name for a fairly simple concept. You can read a whole [Meshifesto](https://buoyant.io/service-mesh-manifesto/) is you really want to understand it or you can take this definition for what it's worth:

```text
A service mesh is a tool that controls the interactions between your applications. It works by inserting proxies between all in mesh communications, we call the meshed proxies the data plane. The data plane does stuff with your inter app communications, stuff like applying policy, making routing decisions, and surfacing data about what the apps are doing. The data plane takes instructions from a separate control plane. 
```

I'm a developer evangelist for Buoyant so I'm going to talk about Linkerd. If you want to know why I think Linkerd is awesome, and way better than Istio go [check it out](https://linkerd.io/2.10/getting-started/). The TL;DR is that Linkerd is small, fast, and works right out the gate.

### Progressive Delivery

This one is all about how do you handle deploying a new version of an app then shift traffic to it while your old version is still out there. Ideally using small increments of traffic that let you see how that new version is doing.

## Get me to it

Cool, that was a lot of words. Now lets do some stuff.

### Deploy a cluster

So if you're using anything other than k3d skip this part because you should already have it working.

```bash
k3d cluster create gitops -p "8081:80@loadbalancer"
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

### Gitops Away

Now that we have flux installed we need to get it working. In this example we're going to be using 2 flux constructs, [kustomizations](https://toolkit.fluxcd.io/components/kustomize/kustomization/) and [gitrepositories](https://toolkit.fluxcd.io/components/source/gitrepositories/). Flux can also use [s3 compatible buckets](https://toolkit.fluxcd.io/components/source/buckets/) and deploy [helm charts](https://toolkit.fluxcd.io/components/source/helmcharts/) but they're out of scope for this article. Send me a message somewhere if you're looking for a deep dive into something else.

#### Git to it

Before we do anything else we need to provide flux with a git repo to pull from. I'm using my [linkerd-demos](https://github.com/JasonMorgan/linkerd-demos) repo but feel free to make your own. I'll be sure to talk about how you'd do that as we go along.

##### Creating a runtime repo

In order to keep things organized, and be compliant with the way flux's kustomize resources works I went with the following structure. I made a folder in a common repo for all my flux deployments. One folder for applications and one for the common runtime. In each folder I split the kubernetes yaml files between a manifests and source folder. Manifests contain the flux files, any helm, gitrepository, or kustomization files needed to deploy the apps. Under source I put the kubernetes files for the things being deployed.

Linkerd has a handy cli which I used to generate the linkerd and linkerd-viz manifests. They were each put into their respective folders under source. The separation allows the kustomize resources to target the appropriate component and keeps them easy to maintain. Linkerd allows you to generate manifests with the cli, you can also generate the same templates via helm if that's your preference.

```bash
linkerd install > path/to/folder/linkerd.yaml

linkerd viz install > path/to/folder/linkerd-viz.yaml
```

##### Tell Flux to use our repo

In order to create a git repo we can use the flux cli, or we can just make a yaml object. I tend to make yaml objects but I'll show the flux cli for completeness.

```bash
# create a git repo
flux create source git gitops --url https://github.com/JasonMorgan/linkerd-demos.git --branch main

# You can run the same command with the export flag to have flux output a yaml object.
flux create source git gitops --url https://github.com/JasonMorgan/linkerd-demos.git --branch main --export
```

By default it'll put everything in the flux-system namespace. You can override that if you like by providing the `--namespace` flag.

Here's the yaml output for that flux create source command:

```text
✚ generating GitRepository source
► applying GitRepository source
✔ GitRepository source created
◎ waiting for GitRepository source reconciliation
✔ GitRepository source reconciliation completed
✔ fetched revision: main/164832204e2e20915ad0a06c23bd19e85287aabd
```

And here's the yaml output of the command with the export flag.

```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: gitops
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  url: https://github.com/JasonMorgan/fleet-infra.git
```

#### Runtime Config

In my example I've got a pretty simple cluster config, or runtime, or platform, or whatever work you use for it. I'm installing Linkerd 2.10 and the Linkerd viz extension, along with flagger. Unfortunately for me they need to be installed in sequence. By that I mean that linkerd needs to be installed before linkerd viz, and flagger needs the prometheus instance in linkerd viz before it can get up and running. We're in luck though because flux implements a wait mechanism we can use, `dependsOn`.

To get our cluster's runtime configured we're going to apply a [runtime yaml manifest](https://github.com/JasonMorgan/linkerd-demos/blob/main/gitops/flux/runtime/manifests/dev_cluster.yaml). Let's dig in and see what it does.

You're going to see 3 kustomization objects, linkerd, linkerd-viz, and flagger. Each one is instrumented with [health checks](https://toolkit.fluxcd.io/components/kustomize/kustomization/#health-assessment) that will allow flux to know when a given kustomization is finished. I've put some comments inline in the yaml.

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: linkerd # Our kustomization name
  namespace: flux-system
spec:
  interval: 30s
  path: ./gitops/flux/runtime/source/linkerd # The folder in your git repo where the linkerd manifest is.
  prune: true
  sourceRef:
    kind: GitRepository
    name: gitops # The name of the git repo we created earlier
  validation: client
  healthChecks: # Our health checks. These are important, the dependsOn statements won't work without these. I've built one health check for each deployment in the app.
    - apiVersion: apps/v1
      kind: Deployment
      name: linkerd-proxy-injector
      namespace: linkerd
    - apiVersion: apps/v1
      kind: Deployment
      name: linkerd-identity
      namespace: linkerd
    - apiVersion: apps/v1
      kind: Deployment
      name: linkerd-controller
      namespace: linkerd
    - apiVersion: apps/v1
      kind: Deployment
      name: linkerd-sp-validator
      namespace: linkerd
    - apiVersion: apps/v1
      kind: Deployment
      name: linkerd-destination
      namespace: linkerd
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: linkerd-viz # Our kustomization name
  namespace: flux-system
spec:
  dependsOn: # This is where we tell linker-viz to wait for linkerd to finish installing
  - name: linkerd
  interval: 30s
  path: ./gitops/flux/runtime/source/linkerd-viz
  prune: true
  sourceRef:
    kind: GitRepository
    name: gitops
  validation: client
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: tap
      namespace: linkerd-viz
    - apiVersion: apps/v1
      kind: Deployment
      name: metrics-api
      namespace: linkerd-viz
    - apiVersion: apps/v1
      kind: Deployment
      name: prometheus
      namespace: linkerd-viz
    - apiVersion: apps/v1
      kind: Deployment
      name: grafana
      namespace: linkerd-viz
    - apiVersion: apps/v1
      kind: Deployment
      name: tap-injector
      namespace: linkerd-viz
    - apiVersion: apps/v1
      kind: Deployment
      name: web
      namespace: linkerd-viz
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: flagger # Our kustomization name
  namespace: flux-system
spec:
  dependsOn: # This is where we tell linker-viz to wait for linkerd-viz to finish installing
  - name: linkerd-viz
  interval: 30s
  path: ./gitops/flux/runtime/source/flagger
  prune: true
  sourceRef:
    kind: GitRepository
    name: gitops
  validation: client
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: flagger
      namespace: linkerd-viz
```

You can find that yaml file in [this repo](https://github.com/JasonMorgan/linkerd-demos.git). I'm going to pull it down locally so we can run the rest of our commands from a local copy of the repo.

```bash
git clone https://github.com/JasonMorgan/linkerd-demos.git

cd linkerd-demos
```

With the repo pulled down we can use the kubernetes cli to pass that manifest over to the kubernetes API. Flux created a bunch of [CRDs](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) we can use so now we can just work with the kubernetes API itself for everything else.

```bash
kubectl apply -f gitops/flux/runtime/manifests/dev_cluster.yaml
```

You should see output like this:

```text
kustomization.kustomize.toolkit.fluxcd.io/linkerd created
kustomization.kustomize.toolkit.fluxcd.io/linkerd-viz created
kustomization.kustomize.toolkit.fluxcd.io/flagger created
```

Now we wait for flux to install our runtime.

```bash
watch kubectl get kustomizations -A
# Press ctrl+c to break when flux is done

# Feel free to check the state of your cluster before moving on
kubectl get pods -A
```

Once those kustomizations complete their install, you can tell by their ready value going to `true`, our runtime is now configured. With that done we can move on to deploying our app.

Example kustomization output:

```text
NAMESPACE     NAME          READY   STATUS                                                            AGE
flux-system   linkerd-viz   True    Applied revision: main/47480dac566c4d1158cb3ee7295a2fd4b172f2dc   39h
flux-system   linkerd       True    Applied revision: main/47480dac566c4d1158cb3ee7295a2fd4b172f2dc   39h
flux-system   flagger       True    Applied revision: main/47480dac566c4d1158cb3ee7295a2fd4b172f2dc   39h
```

#### Deploy our App

With our runtime install out of the way we've got a cluster that's ready to run and manage our application. So we might as well get our app installed. In order to keep things simple our app is located in the same repo as our runtime. I don't necessarily have an opinion on how to structure your repos for a gitops flow. I personally like to have all my manifests in one repo but I have no strong opinion, do what works for you.

With that all siad lets deploy the app, we're using the same kustomize files we used for our runtime. You can do the same dependency chaining as you did for your runtime.

First let's look at the manifest we're applying for our app:

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 1m0s
  path: ./gitops/flux/apps/source/podinfo
  prune: true
  sourceRef:
    kind: GitRepository
    name: gitops
  validation: client
```

It's a fairly simple manifest that will cause flux to apply any kubernetes manifests found in the gitops repo under the `./gitops/flux/apps/source/podinfo` path.

```bash
kubectl apply -f gitops/flux/apps/manifests/podinfo.yaml
```

Which out puts something like:

```text
kustomization.kustomize.toolkit.fluxcd.io/podinfo created
```

Once again just wait for the pods to be ready and track the kustomization object.

```bash
kubectl get kustomization -A

# or

kubectl get pods -n podinfo
```

Now that podinfo is deployed things are about to get weird. Let's dive into flagger and what exactly it's doing to our podinfo objects.

### Canary Deployments

This is where things start to get a little complicated. If you look at the kustomization for podinfo you'll see that it creates 1 podinfo deployment, 1 podinfo service, 3 pods, a canary object, and a horizontal pod autoscaler. The expectation with a gitops workflow, and the way kubernetes works in general, is that that is exactly what we should see in the podinfo namespace. When you look at podinfo however you'll find an environment that's significantly different.

Let's start by looking at our deployments

```bash
kubectl get deploy -n podinfo
```

```text
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
podinfo-primary   2/2     2            2           20h
frontend          1/1     1            1           20h
generator         1/1     1            1           20h
podinfo           0/0     0            0           20h
```

frontend and generator look just like the deployments from our original manifest but podinfo has been scaled down to 0 and there's a new deployment call podinfo-primary. I'll cut the suspense here and get to it, flagger has been modifying our environment. When we deployed flagger we told it we'd be using linkerd as our routing tool and when we deployed podinfo we created a canary object responsible for managing the podinfo deployment. The way flagger does that is by creating new objects to handle rollouts. `podinfo-primary` is the new deployment that flagger is using to host the current stable version of podinfo.

```bash
# Kubernetes services
kubectl get service -n podinfo
```

Once again very similar to our deployments we have some new services. `podinfo-primary` and `podinfo-canary` are newly created and , as their names suggest, they'll be used to handly traffic to both the primary, and canary versions of our applications.

```text
NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
podinfo-canary    ClusterIP   10.43.101.162   <none>        9898/TCP   21h
podinfo-primary   ClusterIP   10.43.202.29    <none>        9898/TCP   21h
frontend          ClusterIP   10.43.49.225    <none>        8080/TCP   21h
podinfo           ClusterIP   10.43.178.68    <none>        9898/TCP   21h
```



```bash

```

```text

```

## Wrap Up

Tell em what you told them. And what I wanted them to learn

Thanks so much for reading and I'd love to hear any feedback you have,

I'm: [twitter](https://twitter.com/RJasonMorgan) or [Linkedin](https://www.linkedin.com/in/jasonmorgan2/).

Jason
