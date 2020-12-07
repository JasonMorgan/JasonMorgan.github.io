---
title:  "Multi Cluster Service Mesh on TKG with Linkerd"
categories: [kubernetes]
tags: [linkerd,kubernetes,k8s,mesh]
published: false
---

Hey folks! Thanks for stopping by! Today I'm going to dive into using the [linkerd](https://linkerd.io/) service mesh to route traffic between 2 [Tanzu Kubernetes Grid](https://tanzu.vmware.com/kubernetes-grid), or tkg, clusters.

## The Set Up

What we're using:

* The tkg cli
  * to create, manage, and scale our k8s clusters
  * you can download it [here]()
* The linkerd cli
  * to do all our linkerd work
* Our podinfo app
* Tanzu Mission Control (Optional component)
  * To manage our tkg clusters
* The tmc cli (Optional component)
  * To connect our clusters to Tanzu Mission Control (TMC)
  * You can download this from your Tanzu Mission Control portal

## Creating and Managing our Clusters

I've previously set up a tkg management cluster in AWS and used it to provision two new clusters.

```bash
tkg create cluster -p dev -w 1 workload1
tkg create cluster -p dev -w 1 workload2
```

Once tkg finished with the cluster create I pulled down the relevant kubeconfig files and added them to Tanzu Mission Control. You have a couple options for adding them to TMC but I chose to use the cli.

```bash
# Get the kubeconfigs
tkg get credentials workload1 --export-file ~/configs/workload1
tkg get credentials workload1 --export-file ~/configs/workload2
```

Here you have the optional TMC commands, if you aren't using TMC please skip ahead.

```bash
# login to TMC
tmc login

# Add your clusters
tmc cluster attach -n workload1 -g jmo -k ~/configs/workload1
tmc cluster attach -n workload2 -g jmo -k ~/configs/workload2
## I had already created a cluster group in TMC
```

Once the clusters appear in TMC you can move on to the next step.

![tmc-screen-grab](/images/tmc-screen-grab.png)

## Install Linkerd in Multi Cluster Mode

### Setting up the CA

I'm following the linkerd multi cluster docs you can find [here](https://linkerd.io/2/tasks/multicluster/). I start out creating a new root certificate authority with [step](https://smallstep.com/cli/)

```bash
# Generate the root ca

step certificate create identity.linkerd.cluster.local root.crt root.key \
  --profile root-ca --no-password --insecure --san identity.linkerd.cluster.local

# Generate an intermediary ca

step certificate create identity.linkerd.cluster.local issuer.crt issuer.key \
  --profile intermediate-ca --not-after 8760h --no-password --insecure \
  --ca root.crt --ca-key root.key --san identity.linkerd.cluster.local

```

We use the the ca to establish a common trust between the mesh in both clusters. You can find a lot more useful details about the certificates and how they're used [here](https://linkerd.io/2/tasks/generate-certificates/) and [here](https://linkerd.io/2/features/automatic-mtls/#how-does-it-work).

### Install the Mesh

#### Test your Clusters

First things first, lets ensure our clusters are good to go. Linkerd's cli comes with a handy dandy check feature to see if your cluster is ready.

```bash
linkerd check --pre
```

Depending on your kubernetes api and linkerd versions you may get some warnings about the crd api version is but that won't impact the install.

If you are using TMC you'll get additional warnings about Pod Security Policies (PSPs), you can safely ignore them provided you aren't enforcing them, otherwise you'll need to appropriately configure your PSPs which is beyond the scope of this article. 

#### Running the Install

Provided you're happy with the results of your pre checks it's time to run the install. I like to set up two terminals, one for each cluster but whatever mechanism you use works provided 

```bash
# Run this once per cluster

## Be sure your paths to the certs are valid, either by executing this in the same directory as the step command or by fixing the path.

linkerd install \
  --identity-trust-anchors-file root.crt \
  --identity-issuer-certificate-file issuer.crt \
  --identity-issuer-key-file issuer.key \
  | kubectl apply -f -
```

Once that's wrapped wait a few minutes for the pods to boot up. You can watch the linkerd namespace or, if you're using tmc you can take a peak at the workloads there.

```bash
# Run this for each cluster
watch kubectl get pods -n linkerd
```

or 

![pods](/images/tmc-linkerd-node-pods.png)

After the control plane is available you can check the status of linkerd using the cli. linkerd goes out of it's way to be easy to use and debug. With that in mind they added the check option into the cli so you can get an at a glance health check. Once again be sure you're running this for each workload cluster.

```bash
# Run this once per cluster
linkerd check
```

#### Adding Multi Cluster Support

Now that we're happy with the per cluster install we need to extend it to handle multi cluster.

```bash
# Run this once per cluster
linkerd multicluster install | kubectl apply -f - 
```

You can check the status of your multicluster install a few different ways. We're going to exercise the linkerd cli a bit, then take a peak at our kubernetes objects. The big thing we're looking for is to ensure our new gateway has a load balancer assigned to it.

```bash
# Run a check with linkerd
linkerd check --multicluster

# Check the new pod
kubectl get pods -n linkerd-multicluster

# Check for a loadbalancer attached to the linkerd-gateway service
kubectl get svc -n linkerd-multicluster

# Ensure the load balancer has been assigned
```

With our new gateways up and running we now want to "link" our clusters. You can get a more detailed explenations in the [docs](https://linkerd.io/2/tasks/multicluster/#linking-the-clusters) but the short version is that running the link commands will allow the clusters to talk and build/maintain the service mirrors.

This part can be a little tricky so I'll include a screen grab after. What we need to do is run our linkerd link command to generate the yaml from 1 cluster then use kubectl to apply it to the other cluster.

```bash

# Run for each cluster
## I accomplish this by running two terminals

### one with the $KUBECONFIG variable set to ~/configs/workload1
#### Then run:
linkerd multicluster link --cluster-name workload2 --kubeconfig ~/configs/workload2 | kubect apply -f - 

### one with the $KUBECONFIG variable set to ~/configs/workload2
#### Then run:
linkerd multicluster link --cluster-name workload1 --kubeconfig ~/configs/workload1 | kubect apply -f -

```

You can see that a little more clearly in the screen grab below.

![terminal-view](/images/terminal-linkerd-mc-link.png)

Next you'll want to validate the clusters are properly linked. Start by running `linkerd multicluster gateways` for each cluster. You should see something like this:

![terminal-gws](/images/terminal-linkerd-mc-gws.png)

Note that each cluster sees the other's gateway. On top of that we can rerun our linkerd check to see more multicluster health check outputs.

```bash
linkerd check --multicluster
```

With that all your checks should be passing and your clusters are now ready for some workloads.

## Getting our Test Service Installed

The default docs want you to use two podinfo configs mapped to the values east and west. I wanted to pair it up to my cluster names so I pulled the app config into it's [own repo](https://github.com/JasonMorgan/podinfo). You can also use the east/west names from the docs by pulling app definitions from the [linkerd website](https://github.com/linkerd/website/tree/master/multicluster).

When running the commands be sure you run a different version of the app manifest for each cluster.

```bash
# the podinfo repo at the path below has the config for 2 clsuters, workload1 and workload2
kubectl apply -k github.com/jasonmorgan/podinfo/workload1/
```

## Linking, or Exporting, a Service

When we decide we want to share a service between clusters with linkerd we need to let linkerd know which services to mirror. We do that with `mirror.linkerd.io/exported=true` label, alternatively if you'd like to modify the key you can find it on the `links.multicluster.linkerd.io` object in the linkerd-multicluster namespace.

```bash
# You'll only need to run this on the workloads in one cluster. I ran it on workload2
kubectl label svc -n test podinfo mirror.linkerd.io/exported=true
```

You can now check the other cluster for the new service. I got the following output on workload1:

```bash
kubectl get svc

NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
frontend            ClusterIP   100.65.191.192   <none>        8080/TCP            38m
podinfo             ClusterIP   100.70.110.153   <none>        9898/TCP,9999/TCP   38m
podinfo-workload2   ClusterIP   100.68.52.124    <none>        9898/TCP,9999/TCP   7s
```

At this point you can move on to the next section where we split traffic. If you'd like some more detailed tests checkout [this](https://linkerd.io/2/tasks/multicluster/#exporting-the-services) section of the docs or see a walkthrough of validating TLS look [here](https://linkerd.io/2/tasks/multicluster/#security).

## Splitting Traffic

