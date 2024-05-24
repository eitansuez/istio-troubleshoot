# Control plane Health

Part of ensuring the health of the mesh includes ensuring the health of the control plane.
That would be `istiod` and all of the activities it performs.

## Logs

We can tail the logs for `istiod`:

```shell
kubectl logs --follow -n istio-system deploy/istiod
```

Among other things, you should see lines reflecting activity such as pushing configuration to sidecars for different workloads.

### Configuring log levels

Similar to the `istioctl proxy-config log` command, Istio also provides a [command for configuring logging for the controlplane](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-admin-log).

Inspect the log level for each logger with:

```shell
istioctl admin log
```

Alter the log level for a logger with the `--level` flag:

```shell
istioctl admin log --level ads:warn
```

Unlike the log command for Envoys, the command will not echo back the update state of the loggers.

To reset the log level, use the `--reset` flag:

```shell
istioctl admin log --reset
```

## Dashboards

In Grafana, the principal dashboard for monitoring the control plane is, as its name implies, the Control Plane dashboard.

```shell
istioctl dashboard grafana
```

On the control plane dashboard, you will find information pertaining to:

- Resource usage for `istiod`
- Frequency of pushes (of Envoy configurations)
- "Pilot" errors
- The number of services that Istio is connected to
- Number of connections to Envoys
- Times that connections were established
- The size of XDS requests and responses

In addition, the Performance dashboard also contains (or repeats) resource usage for `istiod`, in addition to gateways and proxies.

## What affects Istio's resource utilization?

Istio recently switched to using the Delta XDS protocol, which is more efficient in comparison to the "state of the world" version, in terms of the amount of configuration that has to be communicated to each Envoy.

The factors listed below affect Istio's resource usage:

- the number of running workloads and services
- the frequency of deployment changes
- the frequency of mesh configuration changes
- the scope of service discoverability

The last item on the list is something we can do something about.

By default, Istio programs all workloads with information about every other workload.

In our sandbox environment, when we realize, for example, that:

- httpbin doesn't need to know about all of the bookinfo services
- `productpage` is the main service that calls most other `bookinfo` services

Through the [Sidecar resource](https://istio.io/latest/docs/reference/config/networking/sidecar/), we can communicate to Istio what services each workload needs to know about.

This can go a long way to reducing Istio's footprint.

Here is an example set of Sidecar resources that fine-tune the list of services that each deployment needs to know about:

```yaml linenums="1"
--8<-- "cp-health/sidecars.yaml"
```

Apply the sidecars to your cluster:

```shell
kubectl apply -f artifacts/cp-health/sidecars.yaml
```

The control plane dashboard will show an XDS push that updates the sidecar configurations accordingly.

In this sandbox environment with so few services, this hardly makes a difference.
