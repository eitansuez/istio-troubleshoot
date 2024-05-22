# Observability setup

The ability to troubleshoot depends on our ability to see what goes on in a system.

The Envoy sidecars in Istio produce and expose metrics that can be ingested by Prometheus or some other metrics collection tool.
Istio provides Grafana dashboards to support monitoring the health of both the data plane and the control plane.

Distributed traces complement metrics, and help make sense of call graphs, and specifically highlight where time is spent.
Envoys help here, though applications must be configured to propagate trace headers through the call graph.
We will use the Zipkin dashboard to view distributed traces.

Finally, Kiali is an open-source observability tool designed specifically for Istio, containing many features.
We will focus on the call graph visualizations that Kiali generates.

## Deploy observability tools

Istio provides deployment manifests for each tool under the `samples/addons` folder in the Istio distribution.
Each tool will be deployed to the `istio-system` namespace.
In production, more work will be needed to properly deploy, and configure access to each tool.

### Deploy Prometheus

```shell
kubectl apply -f ~/istio-{{istio.upgrade_from}}/samples/addons/prometheus.yaml
```

### Deploy Grafana

```shell
kubectl apply -f ~/istio-{{istio.upgrade_from}}/samples/addons/grafana.yaml
```

### Deploy Zipkin

```shell
kubectl apply -f ~/istio-{{istio.upgrade_from}}/samples/addons/extras/zipkin.yaml
```

### Deploy Kiali

```shell
kubectl apply -f ~/istio-{{istio.upgrade_from}}/samples/addons/kiali.yaml
```

Wait on each deployment to be ready.  Check on the workloads with:

```shell
kubectl get pod -n istio-system
```

## Reconfigure ingress for `bookinfo`

Delete the ingress configuration to `httpbin`:

```shell
kubectl delete -f artifacts/data-plane/httpbin-gateway.yaml
```

In its place, configure ingress for `bookinfo`:

```shell
kubectl apply -f artifacts/mesh-config/bookinfo-gateway.yaml
```

Scale the productpage deployment back to one replica:

```shell
kubectl scale deploy productpage-v1 --replicas 1
```

Verify that ingress is functioning:

```shell
curl http://$GATEWAY_IP/productpage
```

## Generate a load against `bookinfo`

Many tools exist for sending traffic through a system; [fortio](https://fortio.org/) is one.

Here we will use a simple bash loop to send a slow, steady flow of requests through the ingress gateway:

```shell
bash -c "while true; do curl --head http://$GATEWAY_IP/productpage; sleep 0.5; done"
```

Leave this while loop running in its own terminal.

## Access the dashboards

You will primarily work with Grafana, Zipkin and Kiali.

Each dashboard can be accessed through the [dashboard command](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-dashboard).

For example:

```shell
istioctl dashboard grafana
```

This should automatically cause a browser to open to the URL of the Grafana dashboard.

You will find an Istio folder under "Dashboards" in the side navigation bar, containing a variety of pre-built monitoring dashboards.