# Control plane Health

## TODO

- istiod logs
  istioctl admin log
- are there errors in the logs?
- metrics:
  pilot error rate
  istio validation error rate
  sidecar injection error rate
- monitor the control plane - specific metrics to look for
  - the istio control plane grafana dashboard

- what governs how much work istio has to do?
  - number of running workloads / services
  - frequency of deployment changes (adding, removing, updating)
  - frequency of mesh configuration changes
  - service discoverability scope (use sidecar resources)

