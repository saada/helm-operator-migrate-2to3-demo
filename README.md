# Helm Operator Migrate 2to3 Demo

> How do I migrate my environments from Helm 2 to 3 if I use Helm Operator?

With the latest version of Helm Operator [v1.2.0](https://github.com/fluxcd/helm-operator/releases/tag/v1.2.0) we now have support for [Helm 2to3 automatic migration support]( https://github.com/fluxcd/helm-operator/pull/462).

This repo demonstrates how to apply this migration in your clusters.

## Prerequisites

- Helm 2
- Helm 3
- kind
- kubectl
- HelmReleases are all using `apiVersion: helm.fluxcd.io/v1`

## Setup

```sh
# setup a k8s cluster
kind create cluster

# install tiller
kubectl -n kube-system create serviceaccount tiller
kubectl create clusterrolebinding tiller \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:tiller
helm2 init --service-account tiller

# install helm operator
kubectl create namespace flux
helm3 repo add fluxcd https://charts.fluxcd.io
kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/v1.2.0/deploy/crds.yaml
helm3 upgrade -i helm-operator fluxcd/helm-operator \
    --namespace flux \
    --set image.tag="1.1.0" \
    --set helm.versions="v2\,v3" # Default with support for Helm 2 and 3 enabled
```

## Install Helm 2 release that we want to migrate

```sh
kubectl apply -f v2.yaml
helm2 ls # you should see your helm release show up in tiller
```

## Migrate the HelmRelease from v2 to v3

The only changes in [v3.yaml](./v3.yaml) are:

1. Added annotation `helm.fluxcd.io/migrate: "true"`
2. Set `spec.helmVersion: v3`

```sh
# upgrade to latest operator version v1.2.0 that supports 2to3 migrations
helm3 upgrade -i helm-operator fluxcd/helm-operator \
    --namespace flux \
    --set image.tag="1.2.0" \
    --set helm.versions="v2\,v3"
# upgrade the helmrelease with migration changes
kubectl apply -f v3.yaml
helm2 ls # the release should disappear from tiller
helm3 ls # the release should show up in helm3 client output
```

If no changes in the chart itself or values were made, your pods
should remain running even after the migration without interruption.

## Restrict to Helm3 manifests

Once everything is migrating safely without issues, we can turn off v2 support.

```sh
helm3 upgrade -i helm-operator fluxcd/helm-operator \
    --namespace flux \
    --set image.tag="1.2.0" \
    --set helm.versions="v3" # we explicitly no longer want to support new manifests set to "v2"
```
## Monitor the migration

```sh
kubectl logs -f -n flux deploy/helm-operator
kubectl describe hr weave-scope
```

Here's a sample of a successful migration log

```
ts=2020-07-29T22:25:04.119426014Z caller=release.go:79 component=release release=weave-scope targetNamespace=default resource=default:helmrelease/weave-scope helmVersion=v3 info="starting sync run"
ts=2020-07-29T22:25:07.067531338Z caller=release.go:328 component=release release=weave-scope targetNamespace=default resource=default:helmrelease/weave-scope helmVersion=v3 info="running 2to3 migration" phase=migrate
ts=2020-07-29T22:25:07.097633032Z caller=logwriter.go:28 info="2020/07/29 22:25:07 [Helm 2] ReleaseVersion \"weave-scope.v1\" will be deleted."
ts=2020-07-29T22:25:07.115255983Z caller=logwriter.go:28 info="2020/07/29 22:25:07 [Helm 2] ReleaseVersion \"weave-scope.v1\" deleted."
ts=2020-07-29T22:25:07.121587249Z caller=release.go:353 component=release release=weave-scope targetNamespace=default resource=default:helmrelease/weave-scope helmVersion=v3 info="running upgrade" action=upgrade
ts=2020-07-29T22:25:07.15084338Z caller=helm.go:69 component=helm version=v3 info="preparing upgrade for weave-scope" targetNamespace=default release=weave-scope
ts=2020-07-29T22:25:07.154397227Z caller=helm.go:69 component=helm version=v3 info="resetting values to the chart's original version" targetNamespace=default release=weave-scope
ts=2020-07-29T22:25:07.327730795Z caller=helm.go:69 component=helm version=v3 info="performing update for weave-scope" targetNamespace=default release=weave-scope
ts=2020-07-29T22:25:07.340166596Z caller=helm.go:69 component=helm version=v3 info="creating upgraded release for weave-scope" targetNamespace=default release=weave-scope
ts=2020-07-29T22:25:07.348401215Z caller=helm.go:69 component=helm version=v3 info="checking 8 resources for changes" targetNamespace=default release=weave-scope
ts=2020-07-29T22:25:07.463503895Z caller=helm.go:69 component=helm version=v3 info="updating status for upgraded release for weave-scope" targetNamespace=default release=weave-scope
ts=2020-07-29T22:25:07.506436867Z caller=release.go:364 component=release release=weave-scope targetNamespace=default resource=default:helmrelease/weave-scope helmVersion=v3 info="upgrade succeeded" revision=1.1.10 phase=upgrade
```

Profit!