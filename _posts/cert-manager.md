---
title:  "Issuing Certificates with Cert-Manager and Let's Encrypt"
categories: [certificates]
tags: [certificates,LetsEncrypt,kubernetes,k8s, cert-manager]
published: false
---

I go out of my way to secure all my sites with a valid https certificate. I'm also fairly cheap, so this used to leave me with a bit of a dilemma and honestly AWS's certificate manager used to be my go to option. It was handy but I was effectively stuck using AWS and having an Elastic Load Balancer, ELB, to terminate my tls connections. Before we go too deep here if any of the terms I'm using are a little ambiguous I'd recommend checking out [this article](https://www.websecurity.symantec.com/security-topics/what-is-ssl-tls-https) from symantec on the meaning of ssl, tls, and https. The first section covers the definitions well enough that hopefully you feel comfortable with the difference between the terms.

## Concepts

### Let's Encrypt

[Let's Encrypt](https://letsencrypt.org/) is a free service that allows you to programmatically generate tls certificates. It's not always super well documented or easy to use but once you get it in place you can generate, use, and renew certificates on a regular basis without a ton of manual work.

### Cert-Manager

[cert-manager](https://docs.cert-manager.io/en/latest/#) is a kubernetes service that will interact with LetsEncrypt, or another CA, for you to programmatically request, generate, and renew certificates. We're going to limit our scope today to working with Let'sEncrypt. Go [here](https://docs.cert-manager.io/en/latest/getting-started/install.html#installing-with-helm) to ignore some of this guide and just install the latest cert-manager. They're moving to a non default helm repo and have their own getting started instructions. I'll cover it in more detail in the [Getting to the Code](#Code) section.

#### ACME and ACMEv2

Let's Encrypt originally came out with a protocol they called ACME, which was a mechanism for programmatically doing [domain validation](https://en.wikipedia.org/wiki/Domain-validated_certificate). This year they got ACME accepted as an [IETF standard](https://tools.ietf.org/html/rfc8555) and they called it ACMEv2. From a practical standpoint ACME/ACMEv2 supports two mechanisms for domain validation.

For Let'sEncrypt we want to be careful to manage our interactions with the API. Let'sEncrypt provides a free service to anyone that wants certificates. Awesome right? The downside is they throttle requests against their production APIs. Let'sEncrypt has been changing their rate limits as they mature their infrastructure, you can keep up to date with their rate limits [here](https://letsencrypt.org/docs/rate-limits/). The impact of that is that you don't want to test that your issuer and request are valid against the production API. In order to validate your issuer and request before hitting the production API Let'sEncrypt provides a staging API that allows you to happily mess up as much as you like. ***Clarification*** the staging API is also rate limited but it's much more forgiving than the prod API, please don't spam the staging API.

##### DNS

DNS Validation, covered in [section 8.4 of the IETF RFC](https://tools.ietf.org/html/rfc8555#section-8.4), is when you prove you're authoritative for the domain by creating a custom DNS entry.

##### HTTP

HTTP Validation, covered in [section 8.3 of the IETF RFC](https://tools.ietf.org/html/rfc8555#section-8.3), involves putting a specific file on a web server at a specific URL.

#### Issuers

Issuers refer to the service that will act to issue a given certificate. Ultimately your CA issues the actual certificate. The issuer in the context of cert-manager refers to the broker between your kubernetes cluster and the CA.

In order to get your issuer up and running correctly you need to pick a domain validation method. Let'sEncrypt offers free Domain Validation (DV) certs which basically just check that the entity requesting the certificate has effective control of the domain in question. If that doesn't sound like a whole ton of validation you're right, it's not. [Troy Hunt](https://www.troyhunt.com/cloudflare-ssl-and-unhealthy-security-absolutism/) among others actually has a lot of good talk tracks on what exactly SSL/TLS do, and more importantly don't do, to secure a given site. The long and the short of it being a TLS connection makes it really hard to snoop on the traffic between a browser and a site. That's it.

Back to DV: Let'sEncrypt has been pioneering programatic ways to do Domain Validation and they recently got ACMEv2 adopted as an [IETF](https://tools.ietf.org/html/rfc8555) standard. With cert-manager you have two options for DV, HTTP and DNS.

#### Certificates

Certificates refer to the actual certificate you intend to generate. Basically what URL/Host are you trying to secure and where do you want to store your secret? I honestly prefer to think of these objects as requests as the actual certificate is stored in kubernetes as a kubernetes secret but that doesn't really matter.

## Code

This guide is working with cert-manager version 0.7.0, for the latest docs checkout [cert-manager's site.](https://docs.cert-manager.io/en/latest/#)

### Installing cert-manager

The helm chart has a good [installation guide](https://github.com/jetstack/cert-manager/blob/release-0.7/deploy/charts/cert-manager/README.md#installing-the-chart), which I wont spend a ton of time talking about here.

We need to first apply the cert-manager [CRDs](https://raw.githubusercontent.com/jetstack/cert-manager/release-0.7/deploy/manifests/00-crds.yaml).

```bash
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.7/deploy/manifests/00-crds.yaml
```

Next create the cert-manager namespace, `kubectl create namespace cert-manager`, although personally I create and maintain a yaml version of the namespace that I can apply in bulk as necessary. Once the namespace is up you need to apply a label to disable cert-manager validation. `kubectl label namespace cert-manager certmanager.k8s.io/disable-validation="true"`

For the namespace definition I use:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager
  labels:
    certmanager.k8s.io/disable-validation: "true"
```

If you're using helm you can complete the install with this:

```bash

## Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

## Install the cert-manager helm chart
helm install \
  --name cert-manager \
  --namespace cert-manager \
  --version v0.7.0 \
  jetstack/cert-manager

```

### Configuring a Cluster Issuer

I start from the assumption that I won't run multi tenant clusters so I always build out cluster issuers as opposed to creating individual issuers by namespace.

In order to test out my issuer I create a ClusterIssuer that will connect to the staging API. I modify the `spec.acme.server` to use the staging API, https://acme-staging-v02.api.letsencrypt.org/directory. The full staging issuer is included below.

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-stage # Arbitrary string, you'll reference this when you create a certificate
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory # LetsEncrypt API URL
    email: jason@59s.io # Your email address

    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-stage # Arbitrary string

    # ACME DNS-01 provider configurations
    dns01:

      # Here we define a list of DNS-01 providers that can solve DNS challenges
      providers:
        - name: dns # arbitrary string, you'll reference this later in your request.
          route53:
            region: us-east-1 # Region of the zone
            hostedZoneID: IMAVALIDHOSTEDZONEID
            # optional if ambient credentials are available; see ambient credentials documentation
            accessKeyID: IMAVALIDAWSACCESSKEY
            secretAccessKeySecretRef: # Because we're using a cluster issuer this secret, with the properties you see below, need to be placed in the cert-manager namespace.
              name: secret-name
              key: property-name
```

#### Using Staging

Use staging. Until you're super comfortable that your certificate request and issuer work well stick with the staging API. The staging API will generate an invalid certificate and store it in your kubernetes cluster. You don't actually want to use the staging certificate for anything other than to validate that your issuer and certificate request are valid.

### Configuring a certificate request

Once your issuer is up and running you'll request a certificate. Again, we start with the staging API then swap over to the prod API later.

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: 59s-io-stage # Arbitrary string
  namespace: n1
spec:
  secretName: 59s-io-tls-stage # Arbitrary string, name of the secret you want to create
  issuerRef:
    name: letsencrypt-stage
    kind: ClusterIssuer
  commonName: '*.59s.io'
  dnsNames:
  - 59s.io
  acme:
    config:
    - dns01:
        provider: dns # use the provider name from your issuer
      domains:
      - '*.59s.io' # Super sweet wildcard certificates let me use a single ingress for all my sites. LetsEncrypt started issuing wildcard certs back in 2018.
      - 59s.io
```

I like to watch the logs at this point to see what cert-manager is doing and ensure it's able to issue my certificate. Once it's issued you can begin migrating over to the production API.

### Moving to the Production API

We're going to effectively copy the issuer and certificate request from above. Swap out the names so that prod replaces staging. Also we need to update the URL.

#### Cluster Issuer

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod # Arbitrary string, you'll reference this when you create a certificate
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory # LetsEncrypt API URL
    email: jason@59s.io # Your email address

    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod # Arbitrary string

    # ACME DNS-01 provider configurations
    dns01:

      # Here we define a list of DNS-01 providers that can solve DNS challenges
      providers:
        - name: dns # arbitrary string, you'll reference this later in your request.
          route53:
            region: us-east-1 # Region of the zone
            hostedZoneID: IMAVALIDHOSTEDZONEID
            # optional if ambient credentials are available; see ambient credentials documentation
            accessKeyID: IMAVALIDAWSACCESSKEY
            secretAccessKeySecretRef: # Because we're using a cluster issuer this secret, with the properties you see below, need to be placed in the cert-manager namespace.
              name: secret-name
              key: property-name
```

#### Certificate Request

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: 59s-io-prod # Arbitrary string
  namespace: n1
spec:
  secretName: 59s-io-tls-prod # Arbitrary string, name of the secret you want to create
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: '*.59s.io'
  dnsNames:
  - 59s.io
  acme:
    config:
    - dns01:
        provider: dns # use the provider name from your issuer
      domains:
      - '*.59s.io' # Super sweet wildcard certificates let me use a single ingress for all my sites. LetsEncrypt started issuing wildcard certs back in 2018.
      - 59s.io
```

## Conclusion

Use certificates to secure your sites. They're free, the technical burden of requesting and maintaining them is relatively low, and the process can be fully automated using tools like cert-manager. Also Kubernetes has mechanisms to simplify the process of issuing, maintaining, and requesting certificates.

I'll do a follow on post on using a wildcard certificate with an NGINX ingress and a wildcard DNS entry and wildcard certificate to automatically secure any sites you want to build on a given cluster.