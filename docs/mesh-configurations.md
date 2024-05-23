# Mesh Configurations

This section explores troubleshooting mesh configurations, and what tools are at your disposal to validate and diagnose configuration-related issues.

## Deploy the `bookinfo` sample application

Take a moment to familiarize yourself with the Istio sample application [`bookinfo`](https://istio.io/latest/docs/examples/bookinfo/).

Verify that the `default` namespace is labeled for sidecar injection:

```shell
kubectl get ns -Listio-injection
```

If it isn't labeled, the first step is to label it:

```shell
kubectl label ns default istio-injection=enabled
```

Deploy the bookinfo sample application to the `default` namespace:

```shell
kubectl apply -f istio-{{istio.version}}/samples/bookinfo/platform/kube/bookinfo.yaml
```

The sample application consists of a half dozen deployments:  `productpage-v1`, `ratings-v1`, `details-v1`, and `reviews-v[1,2,3]`.

Verify that all workloads are running, and have sidecars:

```shell
kubectl get pod
```

## Configure traffic management

Focus on two Istio custom resources, located in the `artifacts/mesh-config` folder:

- `dr-reviews.yaml`:  A [DestinationRule](https://istio.io/latest/docs/reference/config/networking/destination-rule/) that defines the subsets v1, v2, and v3 of the `reviews` service.

    ```yaml linenums="1"
    --8<-- "mesh-config/dr-reviews.yaml"
    ```

- `vs-reviews-v1.yaml`: A [VirtualService](https://istio.io/latest/docs/reference/config/networking/virtual-service/) that directs all traffic to the reviews service to the `v1` subset.

    ```yaml linenums="1"
    --8<-- "mesh-config/vs-reviews-v1.yaml"
    ```

### Validate resources

Istio provides a [command to perform basic validation](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-validate) on Istio custom resources.

Run the `istioctl validate` against each of these files:

```shell
istioctl validate -f artifacts/mesh-config/dr-reviews.yaml
```

```shell
istioctl validate -f artifacts/mesh-config/vs-reviews-v1.yaml
```

We learn that the resource syntax is valid.
That does not necessarily imply correctness.

The command cannot catch mistakes made in referencing label keys or values, or references to host names.  The `validate` command is more of a schema check.

- Misspelling the keyword `subsets` (perhaps spelled in the singular) is something the `validate` command would catch.  Try it.

- `validate` will also catch a wrong enum name.  For example, try to validate a version of `dr-reviews` where the loadBalancer value of RANDOM is instead spelled using lowercase.

Apply the DestinationRule and VirtualService to the cluster:

```shell
kubectl apply -f artifacts/mesh-config/dr-reviews.yaml
```

And:

```shell
kubectl apply -f artifacts/mesh-config/vs-reviews-v1.yaml
```

### Ascertain the applied configurations

#### Was the resource created?

Did I get an error message when the resources were applied?

The console output should confirm the resource was created:

```console
destinationrule.networking.istio.io/reviews created
```

#### Is the resource present?

Can I display the resource?

```shell
kubectl get destinationrule
```

#### Did I reference the right namespace?

This is a common "gotcha", though in this example we are targeting the `default` namespace, and it's unlikely we got this wrong.

#### Are references correct?

The DestinationRule references a host.  Is the hostname spelled correctly?  If the host resides in a different namespace, the hostname should be fully qualified.

The DestinationRule also references labels.  Are those labels correct?  They could be misspelled.

We are also establishing a naming convention whereby each subset has the name v1, v2, and v3.

The Virtual Service resource references a host, a destination host, and a subset.  Make sure they match the desired host name, target host name, and subset name.

### `istioctl analyze`

The [analyze command](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-analyze) is more powerful than validate, and runs a catalog of analyses against resources applied to the cluster.

Run the command against the default namespace:

```shell
istioctl analyze
```

There should be no issues.

Edit the virtualservice and misspell the host field from `reviews` to, say, `reviewss`.

Apply the updated virtualservice to the cluster.

Re-run analyze.

```console
Error [IST0101] (VirtualService default/reviews) Referenced host not found: "reviewss"
Error [IST0101] (VirtualService default/reviews) Referenced host+subset in destinationrule not found: "reviewss+v1"
Error: Analyzers found issues when analyzing namespace: default.
See https://istio.io/v1.22/docs/reference/config/analysis for more information about causes and resolutions.
```

To get an idea of the variety of checks that the analyze command performs, follow the [above suggested link](https://istio.io/latest/docs/reference/config/analysis/).

In this instance we're warned of the invalid references.

We will revisit the `analyze` command in subsequent example scenarios.

### `istioctl x describe`

The describe command is yet another useful way to independently verify the configuration against the `reviews` service.

```shell
istioctl x describe svc reviews
```

Here is the output:

```console
Service: reviews
   Port: http 9080/HTTP targets pod port 9080
DestinationRule: reviews for "reviews.default.svc.cluster.local"
   Matching subsets: v1,v2,v3
   Policies: load balancer
VirtualService: reviews
   1 HTTP route(s)
```

The above output confirms that multiple subsets are defined, that a VirtualService is associated with the service in question, and that the load balancer policy was altered.

### Do we have the desired behavior?

Make repeated calls from `sleep` to the reviews service.  Do all requests go to v1?

```shell
kubectl exec deploy/sleep -- \
  curl -s reviews.default.svc.cluster.local:9080/reviews/123  | jq
```

Note the value of `podname` in each response.  It should have the prefix `reviews-v1`.

That, ultimately, is the evidence we're looking for.

### Inspect the Envoy configuration

We can ask the question:  how is `sleep` configured to route requests to the `reviews` service?

We can ask Istio to show us the Envoy configuration of the sidecar of the client, in this case `sleep`:

```shell
istioctl proxy-config routes deploy/sleep --name 9080 -o yaml
```

Here is the relevant section from the output:

```yaml linenums="1" hl_lines="19"
- domains:
  - reviews.default.svc.cluster.local
  - reviews
  - reviews.default.svc
  - reviews.default
  - 10.43.207.109
  includeRequestAttemptCount: true
  name: reviews.default.svc.cluster.local:9080
  routes:
  - decorator:
      operation: reviews.default.svc.cluster.local:9080/*
    match:
      prefix: /
    metadata:
      filterMetadata:
        istio:
          config: /apis/networking.istio.io/v1alpha3/namespaces/default/virtual-service/reviews
    route:
      cluster: outbound|9080|v1|reviews.default.svc.cluster.local
      maxGrpcTimeout: 0s
      retryPolicy:
        hostSelectionRetryMaxAttempts: "5"
        numRetries: 2
        retriableStatusCodes:
        - 503
        retryHostPredicate:
        - name: envoy.retry_host_predicates.previous_hosts
          typedConfig:
            '@type': type.googleapis.com/envoy.extensions.retry.host.previous_hosts.v3.PreviousHostsPredicate
        retryOn: connect-failure,refused-stream,unavailable,cancelled,retriable-status-codes
      timeout: 0s
```

On line 19, the cluster reference is to the subset "v1".

Understanding Envoy, how it's designed and configured, can go a long way to helping understand how to read these configuration files, and knowing where to look.

#### Another example

How can we verify that the load balancer policy specified in the DestinationRule has been applied?

That policy is associated with the destination service, which Envoy calls a "cluster".

```shell
istioctl proxy-config cluster deploy/sleep --fqdn reviews.default.svc.cluster.local -o yaml | grep lbPolicy
```

Is the value "RANDOM"?  It should match what we specified in the DestinationRule.
There are four lbPolicies, one for each subset (v1, v2, v3, plus the top-level service).

In contrast, try to look at the lbPolicy configured in the `sleep` sidecar for the `httpbin` service.
What lbPolicy is used for calls to that destination?

We will revisit the `istioctl proxy-config` command in subsequent scenarios.

The page titled [Debugging Envoy and Istiod](https://istio.io/latest/docs/ops/diagnostic-tools/proxy-cmd/) does a great job of explaining the `istioctl proxy-config` command in more detail.


## Using the correct workload selector

[Workload selectors](https://istio.io/latest/docs/reference/config/type/workload-selector/#WorkloadSelector) feature in the configuration of many Istio resources.  They are used to specify which Envoy proxies to target, for different purposes.

For example, an [EnvoyFilter](https://istio.io/latest/docs/reference/config/networking/envoy-filter/) uses workload selectors to identify which proxies to program.
An [AuthorizationPolicy](https://istio.io/latest/docs/reference/config/security/authorization-policy/) uses workload selectors to target the sidecars that control access to their workloads.

Apply the following authorization policy, designed to allow only the `productpage` service to call the `reviews` service:

```shell
kubectl apply -f artifacts/mesh-config/authz-policy-reviews.yaml
```

```yaml linenums="1"
--8<-- "mesh-config/authz-policy-reviews.yaml"
```

Verify that other workloads, say `sleep`, can now no longer call reviews:

```shell
kubectl exec deploy/sleep -- curl -s http://reviews:9080/reviews/123
```

Was the request denied?

Run analyze:

```shell
istioctl analyze
```

```console
Warning [IST0127] (AuthorizationPolicy default/allowed-reviews-clients) No matching workloads for this resource with the following labels: app=reviewss
```

Run the following command, which lists all authorization policies that applies to the reviews service:

```shell
istioctl x authz check svc/reviews
```

Do any rules show up in the output?

It seems we have a typo in the workload selector, in the value specified for the label:  `reviewss` instead of `reviews`.

Fix the authorization policy to use the correctly-spelled label.

Now re-run analyze:

```shell
istioctl analyze
```

The validation issues should have gone away.

Try the authz check again:

```shell
istioctl x authz check svc/reviews
```

There should now be a matching policy in the output.

Finally, try to call reviews from sleep again:

```shell
kubectl exec deploy/sleep -- curl -s http://reviews:9080/reviews/123
```

This time the response should say `RBAC: access denied`.

We can look at the configuration of the inbound listener on the sidecar associated with the `reviews-v1` workload:

```shell
istioctl proxy-config listener deploy/reviews-v1 --port 15006 -o yaml
```

The output is quite lengthy.  Here is the relevant portion, showing the [Envoy RBAC filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/rbac_filter) present in the filter chain, and configured to allow `productpage` through:

```yaml linenums="1"
- name: envoy.filters.http.rbac
  typedConfig:
    '@type': type.googleapis.com/envoy.extensions.filters.http.rbac.v3.RBAC
    rules:
      policies:
        ns[default]-policy[allowed-reviews-clients]-rule[0]:
          permissions:
          - andRules:
              rules:
              - any: true
          principals:
          - andIds:
              ids:
              - orIds:
                  ids:
                  - authenticated:
                      principalName:
                        exact: spiffe://cluster.local/ns/default/sa/bookinfo-productpage
```

## Ingress Gateways

Explore troubleshooting issues with ingress configuration.

Apply the following resource to the cluster:

```shell
kubectl apply -f artifacts/mesh-config/bookinfo-gateway.yaml
```

```yaml linenums="1"
--8<-- "mesh-config/bookinfo-gateway.yaml"
```

`bookinfo-gateway.yaml` configures both a [Gateway](https://istio.io/latest/docs/reference/config/networking/gateway/) resource and a VirtualService to route incoming traffic to the `productpage` app.

Assuming the local cluster deployed with k3d in [setup](setup.md#kubernetes), the ingress gateway is reachable on localhost, port 80:

```shell
export GATEWAY_IP=localhost
```

Then:

```shell
curl http://$GATEWAY_IP/productpage
```

There are many opportunities for misconfiguration:

1. The Gateway selector that selects the ingress gateway.

    If for example we get the selector wrong (edit the value to "ingressgateways"), running `istioctl analyze` will catch the invalid reference:

    ```console
    Error [IST0101] (Gateway default/bookinfo-gateway) Referenced selector not found: "istio=ingressgateways"
    ```

1. The VirtualService that references the gateway.

    Edit the gateway name to "bookinfo-gateways".
    Here too `istioctl analyze` will catch the invalid reference:

    ```console
    Error [IST0101] (VirtualService default/bookinfo) Referenced gateway not found: "bookinfo-gateways"
    Warning [IST0132] (VirtualService default/bookinfo) one or more host [*] defined in VirtualService default/bookinfo not found in Gateway default/bookinfo-gateways.
    ```

1. The routing rule with the destination workload name and port.

    Mistype the destination workload and watch `istioctl analyze` catch that too:

    ```console
    Error [IST0101] (VirtualService default/bookinfo) Referenced host not found: "productpages"
    ```
