---
title:  "Installing code-server in Kubernetes"
categories: [kubernetes]
tags: [code-cerver,kubernetes,k8s]
published: true
---

Hey folks! I wanted to write a quick article today on getting started using code-server on kubernetes. Code-server is an open source project from the folks at [Coder](coder.com) that makes it easy to run and manage a cloud based vscode instance that you can connect to and operate remotely. Installing it in kubernetes has some neat benefits like allowing you to develop your app alongside any versions of other services you want. If you're following a strong gitops flow, it's pretty straightforward to build a dev cluster that closely mirrors one of your production environments; that and it allows you to log into the same vscode instance from any device and location you want.

## Getting Started

Lately, I've gotten into the habbit of writing "getting started" guides and hosting them in Github. You can find my "getting started" guide for code-server [here](https://github.com/JasonMorgan/code-server-getting-started). Feel free to hop over there if you want to get up and running, the rest of this post will be all about what choices I made, and why I made them, while building the manifest and container.

## The Dockerfile

```Dockerfile
FROM codercom/code-server
COPY .vscode /home/coder/.local/share/code-server
RUN curl -Lo shellcheck-v0.7.1.linux.aarch64.tar.xz https://github.com/koalaman/shellcheck/releases/download/v0.7.1/shellcheck-v0.7.1.linux.aarch64.tar.xz \
  && tar -xvf shellcheck-v0.7.1.linux.aarch64.tar.xz \
  && chmod +x shellcheck-v0.7.1/shellcheck && sudo mv shellcheck-v0.7.1/shellcheck /usr/local/bin/ \
  && rm -rf shellcheck* && sudo chown -R coder:coder /home/coder/.local/share/code-server \
  && curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl \
  && chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```

If you aren't really familiar with Dockerfile command words I dig through the FROM/RUN/COPY stuff a bit more in the [git repo](https://github.com/JasonMorgan/code-server-getting-started/blob/master/docker-container.md).

For my instance I decided I wanted my code-server to mirror my local vscode environment, and the best way to do that was to just copy over my extensions directly. I won't dig into it now, but it turns out in spite of vscode being open source some components, like the extension store, have special access rules around them. Long story short, if you're using a variant of vscode, like code-server, you don't necessarily have access to install all the vscode extensions you might be using. You can get around any concern about extension stores by just copying over your current .vscode directory into the container. Just be aware the default extensions directory in code-server is `~/.local/share/code-server` and once you move the extensions over you need to take ownership of the files. Other than that some extensions, like shellcheck and kubernetes, require you to have the binary available in your path so you'll need to modify your dockerfile to pull down anything you need.

If you want to see a bit more detail about the dockerfile check out my comments in the [getting started guide](https://github.com/JasonMorgan/code-server-getting-started/blob/master/docker-container.md).

## The Manifest

Before I go anywhere I want to point out that I got my initial template from the folks over at DigitalOcean. You can see [their post](https://www.digitalocean.com/community/tutorials/how-to-set-up-the-code-server-cloud-ide-platform-on-digitalocean-kubernetes) about deploying code-server on kubernetes. I thought the whole thing was really well done and it's probably worth your time to read through, especially if you want a second opinion.

A lot of this is going to be pretty standard, we have a deployment, a service, and a persistent volume claim. I want to make sure I run my code server, using the image I customized and pushed to dockerhub, I want to be able to access it reliably via a service, and I want to persist it's data disk so I don't lose my work if something happens to the pod.

When we get to persisting the data we have to watch out for a couple things. First and foremost, we have file permissions. I didn't have any issues on docker desktop but mounting volumes on AWS set the user permissions on the file system to root, and code-server by default runs as coder, so I modified my prep script to chown the home directory. After that I built out a little script to set up a clean working directory and to try and clone whatever repo I told the init container to pull. You can find all this in the manifest file [here](https://github.com/JasonMorgan/code-server-getting-started/blob/master/code-server.yaml).

## The Ingress and TLS

This ended up being trickier than I anticipated as code-server uses websockets to get that fancy editor experience and project contour didn't, maybe doesn't, have great documentation around how to get that working. I swapped over to nginx's ingress which supports web sockets by default and was up and running. If you aren't using cert-manager I'd higly recommend you check it out. The ACME HTTP challenge works great and cert-manager and nginx, or contour, will auto generate certificates for you on demand. This blog has some articles about setting up cert-manager and contour has a great tutorial [here](https://projectcontour.io/guides/cert-manager/). If you want to use nginx instead, like I did, you can checkout the how to on cert-manager's [docs page](https://cert-manager.io/docs/tutorials/acme/ingress/). The helm stuff is a bit dated but installing nginx's ingress is pretty straightforward at this point.

I ended up using the following ingress:

```yaml
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: code-server
  namespace: code-server
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    kubernetes.io/tls-acme: "true"
spec:
  tls:
  - secretName: code-server
    hosts:
    - code-server.my-domain.com
  rules:
  - host: code-server.my-domain.com
    http:
      paths:
      - backend:
          serviceName: code-server
          servicePort: 8080
```

Be sure to add some annotations for forcing ssl so you don't accidentally hit the unencrypted endpoint.

## The Wrap Up

That's all I have folks! Hope you enjoyed reading it and if there's anything you'd like to read about in the future hit me up on the kubernetes slack, @jmo, or on twitter @rjasonmorgan.
