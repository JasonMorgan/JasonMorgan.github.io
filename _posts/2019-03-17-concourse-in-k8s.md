---
title:  "Concourse in Kubernetes"
categories: [ci]
tags: [ci,concourse,kubernetes]
published: true
---

# Beginning Words

[Concourse](https://concourse-ci.org/) is a handy tool for running build jobs or any other arbitrary code you want, otherwise known as a CI server. It has it's own [DSL](https://en.wikipedia.org/wiki/Domain-specific_language), domain specific language, that you have to learn but it wasn't too much of a burden for me to pick up. If you're looking for some help getting started with concourse I'd highly recommend [this delightful tutorial](https://concoursetutorial.com/) from the folks over at Starke & Wayne.

Please keep in mind that a lot of this is just personal preference and you should feel free to chop up this advice in the way that best meets your needs.

## Deploying Concourse

You have a lot of options for deploying your own concourse instance. A few examples below:

* a Bosh release
* Pivotal provides a somewhat curated release over at network.pivotal.io
* Engineer Better has [Concourse-Up](https://github.com/EngineerBetter/concourse-up)
  * This will deploy
    * a Bosh Director, think kubernetes for VMs
    * Credhub, a secret store similar to Vault
      * it will even connect concourse to credhub so it can securely store and retrieve credentials
    * A prometheus instance with Grafana
  * It's pretty sweet overall and if you're running in GCP or AWS it'll work well for you
* BUCC from Starke and Wayne
  * Supposed to be similar to Concourse-up but more IaaS agnostic
  * I personally haven't gotten it working so take that for what it's worth

You'll notice there's nothing about kubernetes in that list. Let's dive into the steps you'll want to follow to get concourse running there now.

## Concourse in K8s

I think bosh is cool but I'm often impatient. Kubernetes lets me do a lot of what I do with bosh but it does it with containers which I can start and modify a lot faster than VMs. It also does a bunch of other really cool things that I like which we can get into later.

### Helm Chart

I'm not going to get into what [helm](https://helm.sh/) is and the rest of this article will assume you have some familiarity there. If you want me to write an article about that tweet me or something. Someone out there, I'm not sure who, is maintaining the stable/concourse chart. It works really well and they've been doing a pretty good job keeping it up to date.

### Getting into the code

Kubernetes is YAML so now we're going to dive into that.

#### Pre reqs

* a Kubernetes Cluster
  * I'm going to assume RBAC is enabled as that seems pretty standard
* Helm
* kubectl
* Some kind of editor
* A terminal
* A clean working directory

#### Getting Started with Helm

Open your terminal and get your connection to your kube cluster squared away. You can validate that it's up and running with this command.

`kubectl --version`

Now we're going to set up our tiller account and initialize helm. If you have any questions while you're doing this please refer to [helm's guide](https://helm.sh/docs/using_helm/#role-based-access-control) to installing tiller with RBAC enabled.

##### Tiller Account and Cluster Role Binding

I have a file laying around with my tiller service account and cluster role binding. Save this file as tiller.yaml in my current working directory.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

The big thing here is making sure that the tiller account exists and has appropriate permissions to start launching new applications.

After saving the tiller file we're going to apply it to our kube cluster.

```bash
kubectl apply -f tiller.yaml
```

you should see output that looks super similar to this

```bash
serviceaccount/tiller created
clusterrolebinding.rbac.authorization.k8s.io/tiller created
```

##### Helm Init

With our service account in place we're clear to start tiller.

```bash
helm init --service-account tiller
```

You can test the install by running

```bash
helm version
```

You should get feedback about both the helm and tiller versions.

If you want to be done with the article now and just run the vanilla version of concourse that comes with the chart run `helm install stable/concourse` and follow the instructions in the output to access your instance. The rest of the article is just diving into the values file.

#### Getting the chart

To get started you can find the chart by running `helm search concourse` or just going to the [stable/concourse folder](https://github.com/helm/charts/tree/master/stable/concourse) in the stable helm chart repo.

I like to download a copy locally to get started. Also as a side note I'm going to avoid the most recent version of the chart as I'm not yet ready to dig into the concourse 5.0.0 release. You can view chart versions by running `helm search stable/concourse -l`

To download the chart locally

```bash
helm fetch stable/concourse --version 3.8.0 --untar
```

Keep in mind you have 2 version numbers in play here, the version of the app the chart is managing and the version of the chart. 3.8.0 refers to the chart version and 4.2.2 is the version of concourse it's running. We're actually going to fix that a little when we deploy the app as 4.2.2 still has the bad old version of runc so we're going to fix that by going to 4.2.3.

I always maintain my own version of a chart's values file even when I accept the defaults.

```bash
cp concourse/values.yaml values.yaml
```

#### Important Modifications

Just to get started the helm chart itself bundles a pretty good explanation of values right in the values file. The [repo page](https://github.com/helm/charts/tree/master/stable/concourse) also does a pretty good job getting you up and running.

##### Secrets

So the Chart ships with [some secrets built into it](https://github.com/helm/charts/tree/master/stable/concourse#secrets) to make it easier to use. Obviously if you plan on actually using concourse you ought to swap those out. Lucky for me [the documentation](https://github.com/helm/charts/tree/master/stable/concourse#secrets) has that subject pretty well covered so I'll let you follow those steps yourself.

##### Persistence

Read [this section](https://github.com/helm/charts/tree/master/stable/concourse#persistence) carefully. They aren't joking about the workers filling up your local disks, I managed to bring down 3 workers when I tried to save a little money by skipping the PVCs.
