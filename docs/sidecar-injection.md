# Sidecar Injection

## Objective

To explore troubleshooting issues relating to sidecar injection in Istio.

## Introduction

The Istio documentation contains [a guide on sidecar injection](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/) that explains the mechanism for configuring sidecar injection, and ways in which sidecar injection can be controlled.

### Main lesson

The most important thing to understand about sidecar injection is that **it occurs at pod creation time**.

This means that _if you configure a pod or a namespace for sidecar injection, it won't affect any pods that are already running_.

## Example

Deploy the `httpbin` workload to the default namespace:

```shell
kubectl apply -f ~/istio-{{istio.upgrade_from}}/samples/httpbin/httpbin.yaml
```

The simplest way to tell whether sidecar injection took place is to display the number containers running in the pod:

```shell
kubectl get pod
```

Note the READY column shows 1/1 (one out of one) containers.

Label the `default` namespace for sidecar injection:

```shell
kubectl label ns default istio-injection=enabled
```

The above act is passive:  **nothing will happen until a pod is created in the `default` namespace**.

Either:

- Delete the pod and let the deployment create a new one in its place:

    ```shell
    kubectl delete pod -l app=httpbin
    ```

- Submit a command to the kube API server to restart the deployment:

    ```shell
    kubectl rollout restart deploy httpbin
    ```

List the pods in the `default` namespace once more:

```shell
kubectl get pod
```

The READY column now shows 2/2 containers.  We have a proxy.

## Ways to specify sidecar injection

### Automatic at the namespace level

Above, we used _automatic sidecar injection_.  This means that deployment manifests do not need to be modified.  Instead, a mutating admission webhook is given the chance to mutate the deployment specification before it is applied to the Kubernetes cluster.

Verify that the mutating webhook `istio-sidecar-injection` exists on the cluster:

```shell
kubectl get mutatingwebhookconfigurations
```

The template used by the sidecar injector is encoded in the ConfigMap named `istio-sidecar-injector` in the `istio-system` namespace:

```shell
kubectl get cm -n istio-system istio-sidecar-injector -o yaml
```

Automatic sidecar injection at the namespace level is convenient, and the recommended method.

### Automatic at the pod level

Istio provides a mechanism to [control sidecar injection](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/#controlling-the-injection-policy) at the pod level, by labeling the pod with `sidecar.istio.io/inject` with the value "true".

Instead of labeling the namespace, we could have just applied the label to the `httpbin` workload.

### Manual

The Istio CLI provides the [`kube-inject` command](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-kube-inject) to render the template against a deployment manifest.

For example:

```shell
istioctl kube-inject -f ~/istio-{{istio.upgrade_from}}/samples/httpbin/httpbin.yaml > injected.yaml
```

Inspect the generated file `injected.yaml` and confirm that the deployment resource now specifies two containers.

!!! question

    What name does Istio use for the sidecar container?

## Troubleshooting

The typical questions one should ask when troubleshooting sidecar injection include:

- Was the namespace labeled for sidecar injection?

    This can be verified with:

    ```shell
    kubectl get ns -Listio-injection
    ```

- Was the deployment restarted after labeling the namespace?

    This is one of the more common reasons that the sidecar is not injected.

### :question: How many proxies does Istio see?

Remember the `istioctl version` command?  Run it again.
How many proxies are listed in the data plane?
The original count was one.
If the count still shows "1", it means that no additional proxies were deployed.

### Proxy status

Yet another diagnostic command is the [proxy-status](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-proxy-status) command

```shell
istioctl proxy-status
```

If `httpbin` is not listed in the output, it's an indication that Istio does not know about it, possibly because no sidecar was injected.

We will revisit the `proxy-status` command later.

### Check injection

The Istio CLI provides the [check-inject command](https://istio.io/latest/docs/ops/diagnostic-tools/check-inject/) to check whether sidecar injection has taken place for a given workload:

```shell
istioctl x check-inject -l app=httpbin
``` 

Look for a green checkmark under the INJECTED column.

## Consequences of a missing sidecar

Fundamentally, a missing sidecar means that all traffic in and out of the pod will not be controlled by Istio.

This implies that:

- Istio Custom Resources applied, targeting that workload, will have no effect, as there is no proxy to program.
- The workload will not be considered a part of the mesh.  No service discovery information will be communicated to peer workloads.

In specific circumstances, it could mean that communication between the workload and other mesh workloads will not function.

### Example scenario

Remove the sidecar injection label from the namespace:

```shell
kubectl label ns default istio-injection-
```

Deploy the `sleep` sample, a convenience client from which to `curl` other workloads:

```shell
kubectl apply -f ~/istio-{{istio.upgrade_from}}/samples/sleep/sleep.yaml
```

Note that the `sleep` worklaod has no sidecar.

Yet, we can still call `httpbin`:

```shell
kubectl exec deploy/sleep -- curl httpbin:8000/get
```

However, if our mesh was configured with strict mutual TLS [Peer Authentication](https://istio.io/latest/docs/reference/config/security/peer_authentication/):

```shell
kubectl apply -f artifacts/injection/mtls-strict.yaml
```

An attempt to call `httpbin` once more will fail:

```shell
kubectl exec deploy/sleep -- curl httpbin:8000/get
```

The command fails with a `Connection reset by peer` error, because there is no forward proxy on `sleep` to upgrade the connection to mutual TLS, which is now a requirement.

## Sidecar injection problems

The Istio docs have a page titled [sidecar injection problems](https://istio.io/latest/docs/ops/common-problems/injection/) that catalogs sidecar injection errors and their remedies.

## Communication between proxies and `istiod`

It is important to realize that `istiod` has a constant line of communication with all of the Envoy's:  both gateways and sidecars.

When we run `istioctl proxy-status` we get insight into all the proxies that `istiod` controls, and whether the latest configurations have been sent and synced with each proxy.

A common issue is forgetting to upgrade the sidecars after an Istio upgrade.

To illustrate the issue, upgrade Istio in place.

### Upgrading Istio and "dangling" sidecars

Download a newer version of Istio, version {{istio.version}}:

```shell
curl -L https://istio.io/downloadIstio | ISTIO_VERSION={{istio.version}} sh -
```

Replace the `istioctl` CLI in your PATH with the one from the new distribution.

Verify that the Istio CLI version is now version {{istio.version}}:

```console
client version: {{istio.version}}
control plane version: {{istio.upgrade_from}}
data plane version: {{istio.upgrade_from}} (2 proxies)
```

Upgrade Istio in-place:

```shell
istioctl upgrade -f artifacts/install/trace-config.yaml
```

Re-run `istioctl version` and note the output:

```console
client version: {{istio.version}}
control plane version: {{istio.version}}
data plane version: {{istio.upgrade_from}} (1 proxies), {{istio.version}} (1 proxies)
```

What is going on here?

1. The Istio CLI was upgraded
1. The control plane was also upgraded
1. The ingress gateway component, running Envoy, was also upgraded
1. However, note that there's still one proxy associated with the older version of Istio

Get more information with:

```shell
istioctl proxy-status
```

We can see that `httpbin` was left alone.  It still has a sidecar, but it's not associated with the new control plane.
It's a sort of "orphaned" sidecar in that the new, updated Istio is not communicating with it.

After an upgrade, it's important to be aware that the sidecars (the data plane) are not updated automatically.

To update `httpbin`'s sidecar, restart the deployment:

```shell
kubectl rollout restart deploy httpbin
```

Re-run `istioctl proxy-status` and verify that the `httpbin` sidecar is now associated with Istio version {{istio.version}}.
