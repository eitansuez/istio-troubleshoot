# Installation

The Istio CLI provides a number of commands that relate to the installation of Istio.

## Precheck

The [precheck command](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-experimental-precheck) helps ascertain that Istio can be installed or upgraded on the current Kubernetes cluster:

```shell
istioctl x precheck
```

## Install Istio

Istio is often [installed with the `istioctl` CLI](https://istio.io/latest/docs/setup/install/istioctl/) in sandbox environments.

[`helm`](https://istio.io/latest/docs/setup/install/helm/) is typically the method preferred for QA, staging, and production environments.

Use the Istio CLI to install Istio with the [default configuration profile](https://istio.io/latest/docs/setup/additional-setup/config-profiles/), which deploys `istiod` and the Istio ingress gateway component:

```shell
istioctl install -f artifacts/install/trace-config.yaml
```

Above, we also reference an installation configuration that configures distributed tracing.

```yaml linenums="1"
--8<-- "install/trace-config.yaml"
```

## Is istio installed properly?

Given an environment with Istio installed, we can verify the installation with the [verify-install command](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-verify-install):

```shell
istioctl verify-install
```

## What version of Istio am I running?

The [version command](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-version) provides version information for the CLI, the control plane, and the data plane (proxies):

```shell
istioctl version
```

!!! question

    Part of the output from `istioctl version` is:

    ```console
    data plane version: {{istio.upgrade_from}} (1 proxies)
    ```

    Can you explain what this means?  What proxies are being referred to?

## Access Logging

When using the `default` configuration profile, Envoy sidecars and gateways are not default-configured with access logging to standard output.

We can enable [Envoy Access logging](https://istio.io/latest/docs/tasks/observability/logs/access-log/) using Istio's Telemetry API.

Apply the following resource to your Kubernetes cluster:

```shell
kubectl apply -f artifacts/install/telemetry.yaml
```

```yaml linenums="1"
--8<-- "install/telemetry.yaml"
```

Note that we also set the tracing provider to zipkin.  More on that later.
