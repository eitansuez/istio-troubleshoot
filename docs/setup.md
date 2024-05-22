# Setup

## Kubernetes

Before we can begin exploring Istio debugging, we need a Kubernetes cluster.

Provision a local Kubernetes cluster with [k3d](https://k3d.io/):

```shell
k3d cluster create my-k8s-cluster \
  --k3s-arg "--disable=traefik@server:0" \
  --port 80:80@loadbalancer \
  --port 443:443@loadbalancer
```

```shell
kubectl config get-contexts
```

## Istio distribution

Download a copy of the Istio distribution:

```shell
curl -L https://istio.io/downloadIstio | ISTIO_VERSION={{istio.upgrade_from}} sh -
```

Copy the file `istio-{{istio.upgrade_from}}/bin/istioctl` to your PATH.

Verify that `istioctl` is in your PATH by running these commands:

```shell
istioctl help
```

And:

```shell
istioctl version
```

## Workshop artifacts

Download all yaml artifacts referenced in all scenarios as a single .tgz file [here](artifacts.tgz).

