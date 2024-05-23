# Control plane Health

## TODO

- `istiod` logs
- [`admin log` command](https://istio.io/latest/docs/reference/commands/istioctl/#istioctl-admin-log)
- Are there errors in the logs?
- Metrics:
    - Pilot error rate
    - Istio validation error rate
    - Sidecar injection error rate
- Monitor the control plane
    - Specific metrics to look for
    - The Istio control plane grafana dashboard

- What governs how much work Istio has to do?
    - Number of running workloads / services
    - Frequency of deployment changes (adding, removing, updating)
    - Frequency of mesh configuration changes
    - Service discoverability scope (use sidecar resources)

- Review control plane metrics - istiod footprint, and envoys
  Then apply sidecar resources to the ensemble and measure the improvement.