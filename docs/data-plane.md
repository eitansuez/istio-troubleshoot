# The data plane

In the previous exploration we looked at issues relating to misconfiguration.

In this exploration, we investigate issues with the data plane:  everything is configured correctly, but some traffic flow isn't functioning, and we need to find out why.

## Ingress

We have ingress configured for the bookinfo application, routing requests to the `productpage` destination.

### No Healthy Upstream (UH)

What if for some reason the backing workload is not accessible?

Simulate this situation by scaling the `productpage-v1` deployment to zero replicas:

```shell
kubectl scale deploy productpage-v1 --replicas 0
```

In one terminal, tail the logs of the ingress gateway:

```shell
kubectl logs --follow -n istio-system -l istio=ingressgateway
```

In another terminal, send a request in through the ingress gateway:

```shell
curl http://localhost/productpage
```

In the logs you should see the following line:

```console
"GET /productpage HTTP/1.1" 503 UH no_healthy_upstream - "-" 0 19 0 - "10.42.0.1" "curl/8.7.1" "c4c58af1-2066-4c45-affb-d1345d32fc66" "localhost" "-" outbound|9080||productpage.default.svc.cluster.local - 10.42.0.7:8080 10.42.0.1:60667 - -
```

Note the UH [response flag](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#config-access-log-format-response-flags):  No Healthy Upstream.

These response flags clearly communicate to the operator the reason for which the request did not succeed.

### No Route Found (NR)

As another example, make a request to a route that does not match any routing rules in the virtual service:

```shell
curl http://localhost/productpages
```

The log entry responds with a 404 "NR", for "No Route Found":

```console
"GET /productpages HTTP/1.1" 404 NR route_not_found - "-" 0 0 0 - "10.42.0.1" "curl/8.7.1" "2606aaa9-8c5c-4987-9ba7-86b89f901d34" "localhost" "-" - - 10.42.0.7:8080 10.42.0.1:13819 - -
```

#### UpstreamRetryLimitExceeded (URX)

Delete the `bookinfo` Gateway and VirtualService resources:

```shell
kubectl delete -f artifacts/mesh-config/bookinfo-gateway.yaml
```

In its place, configure ingress for the `httpbin` workload:

```shell
kubectl apply -f artifacts/data-plane/httpbin-gateway.yaml
```

```yaml linenums="1" hl_lines="31-33"
--8<-- "data-plane/httpbin-gateway.yaml"
```

The VirtualService is configured with three retry attempts in the event of a 503 response.

Call the `httpbin` endpoint that returns a 503:

```shell
curl http://localhost/status/503
```

The Envoy gateway logs will show the response flag URX:  UpstreamRetryLimitExceeded:

```console
"GET /status/503 HTTP/1.1" 503 URX via_upstream - "-" 0 0 120 119 "10.42.0.1" "curl/8.7.1" "dcb3b100-e296-4031-8f45-1234d20b0f20" "localhost" "10.42.0.9:8080" outbound|8000||httpbin.default.svc.cluster.local 10.42.0.7:38902 10.42.0.7:8080 10.42.0.1:51761 - -
```

That is, the gateway got a 503, retried the request up to three times, and then gave up.

Envoy's response flags provide insight into why a request to a target destination workload might have failed.

## Sidecar logs

We are not restricted to inspecting the logs of the ingress gateway.  We can also check the logs of the Envoy sidecars.

Tail the logs for the sidecar of the httpbin destination workload:

```shell
kubectl logs --follow deploy/httpbin -c istio-proxy
```

Repeat the call to the httpbin 503 endpoint:

```shell
curl http://localhost/status/503
```

You will see four inbound requests received by the sidecar, i.e. 3 retry attempts.
