# Using kustomize to install linkerd

This project aims to use kustomize to install linkerd and to automate certs
in order to minimize the operational tasks of rotating certs ever so often.

## Goals

1. Use cert-manager to automate ssl rotation
2. Auto-generate linkerd manifests using the linkerd cli

After the installation is complete you could leave linkerd running
for more than a year without compromising the security of the cluster through outdated certificates.

You shouldn't leave it running that long though since there are other
oprational tasks that could improve the security of installation:

1. Upstream security fixes through upgrades
2. The proxy injector, sp validator, and tap components are not rotated through
   this method and will need some additional work.

## Tools

1. [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
2. [Kustomize](https://github.com/kubernetes-sigs/kustomize/releases/tag/kustomize/v3.5.4)
3. [Linkerd CLI](https://github.com/linkerd/linkerd2/releases)
4. Optional [k3d](https://github.com/rancher/k3d/releases)


This project was tested with linkerd 2.7+ since earlier versions do not have
the external cert feature.

The version of `kustomize` bundled with `kubectl` does not provide the plugin
feature we need to use custom generators.

I used k3d to spin up a multi-node local kubernetes cluster for testing out my changes.

## Installation

1. (Optional) Spin up a local kubernetes cluster with 5 nodes

```bash
$ k3d create -n my-k3s-cluster -w 5
$ export KUBECONFIG="$(k3d get-kubeconfig --name='my-k3s-cluster')"
$ kubectl get nodes # To ensure the nodes came up
```

2. Install cert-manager

```bash
$ kustomize build cert-manager | kubectl apply --validate=false -f -
$ kubectl -n cert-manager get po
```

Wait for cert-manager pods to become ready.

3. Disable injection on kube-system namespace

```bash
$ kubectl annotate ns kube-system linkerd.io/inject=disabled
$ kubectl label ns kube-system linkerd.io/is-control-plane=true config.linkerd.io/admission-webhooks=disabled
```

4. Install linkerd certs

```bash
$ kustomize build certs | kubectl apply -f -
```

5. Finally install linkerd

```bash
$ KUSTOMIZE_PLUGIN_HOME=`pwd`/linkerd/plugins kustomize build --enable_alpha_plugins linkerd | kubectl apply -f -
```

6. Check installation

```bash
$ linkerd check --proxy --namespace=linkerd
```

## Uninstallation

Remove linkerd certs and installation

1. Uninstall linkerd and certs

```bash
$ KUSTOMIZE_PLUGIN_HOME=`pwd`/linkerd/plugins kustomize build --enable_alpha_plugins linkerd | kubectl delete -f -
```

2. Uninstall cert-manager

```bash
$ kustomize build cert-manager | kubectl delete -f -
```

3. (Optional) Destroy the k3s cluster

```bash
$ k3d del -n my-k3s-cluster
```


## Production Ready ?

Short answer is No.

This project is by no means production ready. There are still three components
of linkerd that require cert rotation that cannot be automated through the
linkerd cli. You'd need to use the helm chart for that.

I had to use helm and the process described in the linkerd docs to generate and
semi-automate cert rotation. See linkerd [docs here](https://linkerd.io/2/tasks/manually-rotating-control-plane-tls-credentials/)
and [here](https://linkerd.io/2/tasks/automatically-rotating-control-plane-tls-credentials/)

Also see these [helm variables](https://github.com/linkerd/linkerd2/blob/stable-2.7.0/charts/linkerd2/values.yaml#L134-L154)

## Road to full tls automation

In the near future when the proxy injector, sp validator, and tap components
are able to read from tls secrets (instead of opaque ones) we could potentially
use cert-manager to rotate those as well as inject the `caBundle` into their webhooks and api service
using the [ca injector](https://cert-manager.io/docs/concepts/ca-injector/).

1. proxy injector [webhook](https://github.com/linkerd/linkerd2/blob/stable-2.7.0/charts/linkerd2/templates/proxy-injector-rbac.yaml#L92), [deployment](https://github.com/linkerd/linkerd2/blob/stable-2.7.0/charts/linkerd2/templates/proxy-injector.yaml#L90) and [secret](https://github.com/linkerd/linkerd2/blob/stable-2.7.0/charts/linkerd2/templates/proxy-injector-rbac.yaml#L56-L70)

2. sp validator [webhook](https://github.com/linkerd/linkerd2/blob/stable-2.7.0/charts/linkerd2/templates/sp-validator-rbac.yaml#L80), [deployment](https://github.com/linkerd/linkerd2/blob/stable-2.7.0/charts/linkerd2/templates/sp-validator.yaml#L104), and [secret](https://github.com/linkerd/linkerd2/blob/stable-2.7.0/charts/linkerd2/templates/sp-validator-rbac.yaml#L44-L58)

3. tap [api service](https://github.com/linkerd/linkerd2/blob/stable-2.7.0/charts/linkerd2/templates/tap-rbac.yaml#L125), [deployment](https://github.com/linkerd/linkerd2/blob/stable-2.7.0/charts/linkerd2/templates/tap.yaml#L119), and [secret](https://github.com/linkerd/linkerd2/blob/stable-2.7.0/charts/linkerd2/templates/tap-rbac.yaml#L94-L108)

Once these components can read from actual tls secrets and the command line
allows creating them externally, the commented out pieces of yaml can be
uncommented to automate all tls tasks for linkerd completely.

I'll attempt to file a ticket for this and link it back here.
