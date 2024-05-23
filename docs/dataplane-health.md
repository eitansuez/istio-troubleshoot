# Data plane Health

## Objective

To get acquainted with observability tools, how they help us understand the behavior of our system, with a focus on the health of the data plane.

## Introduction

The data plane consists of the gateways, sidecars and workloads through which traffic flows.

A healthy data plane should be available, responsive, there should be no (or few) errors.

We focus on RED metrics:  Requests, Errors, and Durations

## Setup

Splitting traffic to the `reviews` service between subsets `v1` and `v3` will allow us to observe traffic distribution:

```shell
kubectl apply -f istio-{{istio.version}}/samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml
```

## Kiali

[Kiali](https://kiali.io/) is a monitoring solution designed specifically for Istio.
It is an open-source project, and is primarily known for its call graph visualizations.

Launch the Kiali dashboard:

```shell
istioctl dashboard kiali
```

In the navigation bar, select "Traffic Graph", and point the namespace pulldown menu to the `default` namespace.

A basic visualization will appear.  The call graph is constructed from live traffic flows within the system.

We can decorate this visualization with additional information.  We can turn on traffic animations, security information, traffic rate (requests per second), traffic distribution (%), request and response throughput (bytes per second), and response times (ms).  The arrows between services are color-code to signal whether requests are currently succeeding or failing.

You should see a 50/50 traffic split between `v1` and `v3` subsets of the `reviews` service, low latencies, no errors, acknowledgements that the communication between all services is mutual-tls enabled.

We also gain an understanding of the call graph:  the interactions between services, and their respective volumes.

Kiali also allows us to inspect and modify Istio custom resources, look at specific services, workloads and their metrics, and will produce call graphs in the context of (from the perspective of) a specific service or workload.

## Zipkin

The focus of the Zipkin dashboard is to uncover errors and latency issues in a call graph.

```shell
istioctl dashboard zipkin
```

Search for traces involving the `productpage` serviceName, click "Run Query".
We can also look for traces with durations above a certain minimum.

Select a trace by clicking the "Show" button.
The resulting visualization helps answer questions such as:

- What services were called?
- How much time was spent inside each service?
- How much time is spent on the network?
- Are multiple services called in parallel or serially?

We can also look at metadata that was collected with each trace and with each span, details of the requests, the HTTP method, the url, the upstream or downstream peer, etc..

## Grafana

In our setup, metrics are exposed by Envoys, and collected by Prometheus.
Grafana has Prometheus pre-configured as its data source.
The dashboards you will explore expose those metrics in an intelligible way.

```shell
istioctl dashboard grafana
```

### The mesh dashboard

Start with the mesh dashboard, designed to provide a high-level overview of the state of the mesh:

- the global request volume
- global success rate
- 4xx and 5xx errors
- the number and kind of Istio resources currently applied to the cluster:  Gateways, VirtuaServices, DestinationRules, Authorization Policies, etc..

For each service we also get their salient metrics:  request volume, latency distribution, and success rate.

### Service dashboards

We can drill down a step further by looking at the service dashboard.
Select the service to monitor from the pulldown menu at the top of the page.

The "General" panel contains the RED metrics for that service:  request volume, success rate, and request durations (percentiles).

Under "Client workloads" we can see a breakdown of requests to this service by client.  In our case this is not so informative since most services have a single distinct client:  productpage is called by the ingressgateway, details is called by productpage, etc..

The third panel, "Service Workloads", breaks down the metrics by backing workloads.  For example, the `reviews` service is backed by one instance of `reviews-v1` and one instance of `reviews-v3`.
Istio service & workload Grafana dashboards

## Workload dashboards

At the most detailed level, the workload dashboards allow us to look at metrics for a specific workload.  For example look at the workload `reviews-v3`.

We can get the RED metrics for this workload to the exclusion of other workloads.
We see incoming requests from the `productpage` service, and outbound requests to the `ratings` service.

### Data plane metrics

When monitoring the data plane, look for:

- High 4xx error rate
- High 5xx error rate
- High request latency
- Latency 99 percentile

## The Prometheus dashboard and PromQL queries

[tbd]

## TODO

Consider adding fault injection to show what error scenarios look like.