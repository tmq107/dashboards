# CPU Throttling and Kubernetes CPU Limits

CPU and memory are different resources. CPU is compressible: a workload can run slower when the node is busy. Memory is not compressible: exceeding a memory limit can trigger an OOM kill.

This dashboard helps identify CPU throttling caused by restrictive CPU limits. It does not automatically decide whether a limit should be removed.

## What a CPU limit does

On Linux nodes, Kubernetes enforces container CPU limits through the Linux CFS bandwidth controller. The default quota period is 100ms, although cluster configuration can change it.

For example:

```yaml
resources:
  requests:
    cpu: 500m
  limits:
    cpu: 500m
```

A `500m` CPU limit permits approximately 50ms of aggregate CPU time per 100ms period. A bursty, multi-threaded process can consume that quota quickly. CFS then throttles the cgroup until more quota becomes available.

This can produce a misleading symptom:

- Average CPU usage looks low.
- No memory pressure exists.
- Error rate remains normal.
- p95 or p99 request latency increases.

Average CPU usage is measured over a longer interval. CFS throttling happens in short quota periods, so average CPU alone cannot show this problem.

## Requests and limits

CPU requests have two important roles:

1. The scheduler uses requests when placing Pods on nodes.
2. CPU requests influence relative CPU weight during contention.

CPU limits are not used for normal scheduler placement. They enforce a hard runtime ceiling. A limit can prevent a container from using idle CPU that would otherwise be available.

Requests are not an absolute latency or CPU guarantee. Actual behavior also depends on node contention, cgroup version, CPU Manager policy, system workloads, and cluster configuration.

## Dashboard signals

### Deployment CPU sizing

The Deployment CPU table contains two throttling views:

- `cpu throttling %`: maximum observed throttling ratio across the selected Grafana range and Deployment Pods.
- `cpu throttling % (5m)`: current throttling ratio over the last five minutes, using the highest current Pod value for the Deployment.

The 5-minute value is better for tracking an active incident. The historical value is useful for finding intermittent problems across a longer time range.

The ratio is calculated from CFS enforcement periods:

```promql
100 * (
  sum by (namespace, pod) (
    rate(container_cpu_cfs_throttled_periods_total[5m])
  )
  /
  clamp_min(
    sum by (namespace, pod) (
      rate(container_cpu_cfs_periods_total[5m])
    ),
    0.001
  )
)
```

The dashboard maps Pod names to Deployments using the ReplicaSet Pod-name pattern described in `README.md`.

### Throttled CPU seconds

Throttled-period percentage measures how often throttling occurred. It does not measure how long the workload was delayed. When available, also inspect throttled CPU seconds:

```promql
sum by (namespace, pod, container) (
  rate(container_cpu_cfs_throttled_seconds_total[5m])
)
```

Use both signals with request latency. A high period ratio with very little throttled time may have less impact than a smaller ratio with long throttling pauses.

## Correlate throttling with latency

CPU throttling is more actionable when it rises at the same time as request tail latency.

- p95: 5 percent of requests are slower.
- p99: 1 percent of requests are slower.

For kgateway and Envoy, the separate Upstream Service latency table uses:

```promql
histogram_quantile(
  0.99,
  sum by (le, envoy_cluster_name) (
    rate(envoy_cluster_upstream_rq_time_bucket[5m])
  )
)
```

This measures latency from kgateway/Envoy to an upstream cluster. It is Service or cluster level, not directly Deployment level.

Interpretation:

- Throttling rises and Envoy p99 rises: CPU limits may be contributing to latency.
- Throttling rises but p99 stays normal: do not change limits automatically.
- Envoy p99 rises without backend throttling: investigate dependencies, network behavior, kgateway, or the upstream service.

## Recommended investigation

1. Check `cpu throttling % (5m)` during the incident window.
2. Check `cpu throttled seconds` if the metric is available.
3. Compare the same time window with application p95 or p99 latency.
4. Confirm the container has a CPU limit and inspect its CPU request.
5. Check node CPU saturation and other competing workloads.
6. Change one workload at a time.
7. Watch latency, throttling, errors, restarts, and node capacity after the change.

Keep CPU requests accurate. A common option for latency-sensitive workloads is to keep the CPU request and remove or raise the CPU limit, but this requires workload and cluster validation.

## When CPU limits still make sense

CPU limits remain reasonable for:

- Untrusted or genuinely multi-tenant workloads.
- Deliberate performance isolation.
- Predictable staging or capacity testing.
- Static CPU Manager workloads using exclusive cores.
- Chargeback or showback policies requiring hard ceilings.
- Workloads with an explicitly tested CPU budget.

Make CPU limits a workload-specific decision, not an automatic copy-pasted default.

## Memory remains different

Do not apply CPU guidance blindly to memory. Memory limits can contain an application's memory usage and reduce its node-level blast radius, but exceeding a memory limit can result in an OOM kill and restart.

Memory limits equal to memory requests can provide strong isolation and Guaranteed QoS when all containers in the Pod have matching CPU and memory requests and limits. Validate memory usage, startup peaks, and OOM events before changing memory settings.

## References

- [Kubernetes resource management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes resource requests and limits](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#requests-and-limits)
- [Kubernetes in-place Pod resize](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#pod-resize-inplace)
- [Linux CFS bandwidth control](https://www.kernel.org/doc/html/latest/scheduler/sched-bwc.html)
