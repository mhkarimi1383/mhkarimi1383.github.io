+++
title = "Reflector"
date = "2023-07-10T18:28:38+03:30"
author = "mhkarimi1383"
authorTwitter = "mhkarimi1383" #do not include @
tags = ["kubernetes", "helm", "docker", "reflector", "k8s", "operator"]
keywords = ["kubernetes", "helm", "docker", "reflector", "k8s", "operator"]
+++

# Share and manage Kubernetes resources between namespaces

Sometimes that's hard to manage Kubernetes resources that you need to have them in multiple namespaces (e.g. pullSecrets, TLSCerts
or some shared web server plugins) to handle that we have to use a Reflector in our cluster, I found one on GitHub, but I had
two problems with that, first you need a source resource to annotate or label but sometimes you are creating resources yourself and
there is no need to have an extra namespace for starting point, So I made a K8s Operator that is able to handle both ;)

Before getting started, If you found the project useful don't forget to give it a start [here](https://github.com/mhkarimi1383/reflector)

## Installing Reflector

To install reflector, Of course you need a K8s Cluster to work with, and [helm 3](https://helm.sh/docs/intro/install/) installed you can verify that by `helm version`

Then we will install the Reflector itself by using Helm command

short version:

```bash
helm install reflector reflector -n reflector --repo https://reflector.karimi.dev --create-namespace
```

long version:

```bash
helm repo add reflector https://reflector.karimi.dev
helm repo update
helm install reflector reflector/reflector -n reflector --create-namespace
```

both will create a namespace called `reflector` and a helm release called `reflector` you can customize all of them as your needs

Also if you want to customize helm values checkout [here](https://github.com/mhkarimi1383/reflector/blob/main/charts/reflector/values.yaml) to see default values.

## Working with Reflector

We have two ways of using reflector

### Using CRD

Here is an example of using `Reflect` CRD

```yaml
---
apiVersion: "k8s.karimi.dev/v1"
kind: Reflect
metadata:
  name: test
  namespace: test
spec:
  namespaces:
    - ns1
    - ns2
  items:
    - apiVersion: v1
      kind: ConfigMap
      metadata:
        name: test-config
      data:
        foo: bar
```

as you can see `namespaces` property is used to manage the list of namespaces to reflect `items` also

you can put any resource with any kind in `items` with no limits

> Every namespace in namespaces list should exist before creation, you will get validation error if operator can't find one or more namespaces. Here is an error example 'error: reflects.k8s.karimi.dev "test" could not be patched: admission webhook "reflect-validator.k8s.karimi.dev" denied the request: Namespaces (not-found-ns2 not-found-ns1) does not exist. all of the namespaces should exist before creating new Reflect'
>
> Also you cannot edit or remove resources that are managed by `reflector` directly, it will get blocked

Also `Reflect` CRD has some other name for easier usage when using `kubectl` commands (like `delete`, `edit`, etc.)

other names are:

* ref
* rf
* rfl
* rfct
* rft

> we do not support removing namespaces from list yet

### Using Secret `Labels`/`Annotations`

You need to add two properties to your secret that you want to reflect

in `Labels` add

```yaml
dev.karimi.k8s/reflect: "true"
```

and in `Annotations` add

```yaml
dev.karimi.k8s/reflect-namespaces: ns1,ns2
```

Here is an full example

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: example.com-tls
  labels:
    dev.karimi.k8s/reflect: "true" # important
  annotations:
    dev.karimi.k8s/reflect-namespaces: ns1,ns2 # important
stringData:
  foo: bar
```

> we do not support removing namespaces from list yet
>
> Like Reflect CRD namespaces should exist and also if not you will get an error like that
>
> Also you cannot edit or remove secrets that are managed by `reflector` directly, it will get blocked

#### tools like Cert-Manager

We are supporting tools like `Cert-Manager` you have to just inject required `Labels` and `Annotations` into their template

here is an example for `Certificate` resource

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-tls
  namespace: default
spec:
  secretTemplate:
    labels:
      dev.karimi.k8s/reflect: "true"
    annotations:
      dev.karimi.k8s/reflect-namespaces: ns1,ns2
  dnsNames:
    - example.karimi.dev
  issuerRef:
    group: cert-manager.io
    kind: ClusterIssuer
    name: http-issuer
  secretName: example-tls
  usages:
  - digital signature
  - key encipherment
```
