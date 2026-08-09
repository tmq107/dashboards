# Deployment Right-Sizing Dashboard

Grafana dashboard for comparing Kubernetes resource requests and limits with observed Deployment usage.

Source dashboard: [`dashboard.json`](./dashboard.json)

## Dashboard settings

| Setting | Value |
|---|---|
| Dashboard title | Deployment Right-Sizing |
| Datasource | Prometheus, selected through `$datasource` |
| Refresh | 30 seconds |
| Default time range | `now-7d` to `now` |
| Timezone | Browser timezone |
| Resource scope | All namespaces |

## How to use

1. Select a Prometheus datasource.
2. Choose a time range with enough historical data.
3. Review Overview for cluster-wide totals.
4. Review Deployment CPU sizing and Deployment Memory sizing for current configuration versus observed usage.
5. Use Startup CPU and Startup Memory to identify startup bursts.
6. Use CPU Recommended Sizing and Memory Recommended Sizing as candidate values, not automatic production settings.
7. Validate recommendations against traffic, autoscaling, rollout behavior, and application requirements before changing manifests.

## Panel map

| Row | Panel | Type | Queries |
|---|---|---|---:|
| Overview | Deployments in cluster | Stat | 1 |
| Overview | All CPU requests | Stat | 1 |
| Overview | All CPU limits | Stat | 1 |
| Overview | All memory requests | Stat | 1 |
| Overview | All memory limits | Stat | 1 |
| Overview | Current CPU usage | Stat | 1 |
| Overview | Current memory usage | Stat | 1 |
| Deployment CPU | Deployment CPU sizing | Table | 7 |
| Deployment Memory | Deployment Memory sizing | Table | 7 |
| Deployment Startup CPU | Deployment Startup CPU | Table | 2 |
| Deployment Startup Memory | Deployment Startup Memory | Table | 2 |
| CPU Recommended Sizing | CPU Recommended Sizing | Table | 4 |
| Memory Recommended Sizing | Memory Recommended Sizing | Table | 5 visible fields, 7 queries |

The row panels are Grafana layout groups. They do not contain PromQL queries.

## Shared query logic

Most deployment panels use this pod-name pattern:

```text
.*-[a-z0-9]{9,10}-[a-z0-9]{5}$
```

This matches common Deployment-managed Pod names generated through a ReplicaSet. Queries then:

1. Aggregate container metrics to Pod level.
2. Extract the Deployment name from the Pod name with `label_replace`.
3. Build `namespace/deployment` with `label_join` into the `workload` label.
4. Aggregate or compare values by `workload` and `deployment`.
5. Join table query results by `workload`.

Common filters:

- `container!=""`: exclude metrics without a container label.
- `container!="POD"`: exclude the Kubernetes pause container.
- `image!=""`: exclude empty-image series.
- `resource="cpu"` or `resource="memory"`: select resource type.

`$__range` represents the selected Grafana time range. `$__rate_interval` is Grafana's rate interval calculated for the panel resolution.

### How P95 across Pod-level P95 values is calculated

CPU steady-state sizing uses two percentile stages:

1. Calculate time-based P95 CPU usage for each Pod.
2. Calculate P95 across those Pod-level P95 values for each Deployment.

PromQL expression pattern:

```promql
quantile by (workload, deployment) (0.95, <per-Pod P95 expression>)
```

This is neither an average nor a maximum. Prometheus sorts the values and interpolates when the percentile falls between two values. Prometheus uses:

```text
rank = quantile × (number_of_values - 1)
```

Example with fictional Pod-level P95 values:

```text
0.20, 0.21, 0.22, 0.23, 0.24, 0.25, 0.26, 0.27, 0.28, 0.90
```

For 10 values:

```text
rank = 0.95 × (10 - 1) = 8.55
```

Rank `8.55` lies 55% between index 8 (`0.28`) and index 9 (`0.90`):

```text
0.28 + (0.90 - 0.28) × 0.55
= 0.28 + 0.62 × 0.55
= 0.28 + 0.341
= 0.621 cores
```

The example values are illustrative. Dashboard values come from Prometheus data.

## Panel details

## Overview

### Deployments in cluster

Description: current number of Deployments known by kube-state-metrics.

**Query A**

```promql
count(max by (namespace, deployment) (kube_deployment_spec_replicas)) or vector(0)
```

What it does:

- Groups Deployment replica specifications by namespace and Deployment.
- Counts resulting Deployment series.
- Returns zero when no series exist.

Unit: short count.

### All CPU requests

Description: total CPU requests across all Pods.

**Query A**

```promql
sum(max by (namespace, pod, container) (kube_pod_container_resource_requests{resource="cpu"})) or vector(0)
```

What it does:

- Selects configured CPU requests.
- Keeps one value per namespace, Pod, and container.
- Sums all container requests across the cluster.

Unit: CPU cores.

### All CPU limits

Description: total CPU limits across all Pods.

**Query A**

```promql
sum(max by (namespace, pod, container) (kube_pod_container_resource_limits{resource="cpu"})) or vector(0)
```

What it does:

- Selects configured CPU limits.
- Keeps one value per namespace, Pod, and container.
- Sums all container limits across the cluster.

Unit: CPU cores.

### All memory requests

Description: total memory requests across all Pods.

**Query A**

```promql
sum(max by (namespace, pod, container) (kube_pod_container_resource_requests{resource="memory"})) or vector(0)
```

What it does:

- Selects configured memory requests.
- Keeps one value per namespace, Pod, and container.
- Sums all container requests across the cluster.

Unit: bytes.

### All memory limits

Description: total memory limits across all Pods.

**Query A**

```promql
sum(max by (namespace, pod, container) (kube_pod_container_resource_limits{resource="memory"})) or vector(0)
```

What it does:

- Selects configured memory limits.
- Keeps one value per namespace, Pod, and container.
- Sums all container limits across the cluster.

Unit: bytes.

### Current CPU usage

Description: current CPU usage across all Pods.

**Query A**

```promql
sum(max by (namespace, pod, container) (rate(container_cpu_usage_seconds_total{container!="",container!="POD",image!=""}[$__rate_interval]))) or vector(0)
```

What it does:

- Calculates per-container CPU usage rate.
- Aggregates container series to Pod/container identity.
- Sums current usage across all Pods.

Unit: CPU cores.

### Current memory usage

Description: current memory working set across all Pods.

**Query A**

```promql
sum(max by (namespace, pod, container) (container_memory_working_set_bytes{container!="",container!="POD",image!=""})) or vector(0)
```

What it does:

- Selects container working-set memory.
- Excludes pause and empty-image series.
- Sums current working-set memory across all Pods.

Unit: bytes.

## Deployment CPU sizing

Panel description: Deployment name is derived from Deployment-managed Pod names. Historical Pod samples remain usable while Prometheus retains them. Startup-specific metrics are excluded. Request and limit columns show distinct current values from live Pods; multiple values indicate a rollout or inconsistent Pod configuration. Empty request or limit values display as `N/A`.

All values are joined by `workload`, displayed as `deployment`, and sorted by deployment name.

| Query | Output column | Meaning | Visibility/use |
|---|---|---|---|
| A | `cpu request set / pod` | Distinct current configured CPU requests across live Pods | Visible |
| B | `p95 cpu usage / pod (p95 across pods)` | P95 across Pod-level P95 CPU values | Visible sizing baseline |
| C | `p95 cpu usage / request (p95 across pods)` | Historical P95 usage divided by the current average request setting, expressed as percent | Visible |
| D | `max cpu usage / pod` | Maximum observed CPU usage per Pod over the selected range | Visible safety signal |
| E | `cpu request / max` | Request divided by maximum usage, expressed as percent | Hidden diagnostic |
| F | `cpu limit set / pod` | Distinct current configured CPU limits across live Pods | Visible |
| G | `cpu throttling %` | Maximum observed CPU throttling ratio | Visible |

### Query A: CPU request per Pod

```promql
label_join(label_replace(sum by (namespace, pod) (kube_pod_container_resource_requests{resource="cpu",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment")
```

Aggregates CPU requests across containers for each Pod and derives Deployment identity. Grafana groups live Pod values by Deployment and displays all unique request values. No average is calculated.

### Query B: P95 CPU usage per Pod

```promql
quantile by (workload, deployment) (0.95, label_join(label_replace(quantile_over_time(0.95, (sum by (namespace, pod) (max by (namespace, pod, container) (rate(container_cpu_usage_seconds_total{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}[$__rate_interval]))))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))
```

Calculates each Pod's CPU P95 over the selected range, then calculates P95 across those Pod-level P95 values for each Deployment.

### Query C: Per-Pod CPU usage divided by request

```promql
100 * (quantile by (workload, deployment) (0.95, label_join(label_replace(quantile_over_time(0.95, (sum by (namespace, pod) (max by (namespace, pod, container) (rate(container_cpu_usage_seconds_total{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}[$__rate_interval]))))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment")) / clamp_min((avg by (workload, deployment) (label_join(label_replace(sum by (namespace, pod) (kube_pod_container_resource_requests{resource="cpu",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))), 0.001))
```

Divides historical Deployment P95 CPU usage, including old Pods, by the current average request setting. This intentionally compares retained historical usage against the current configuration. `clamp_min` prevents division by values below `0.001` cores. Thresholds: yellow below 10%, green from 10% to 100%, yellow from 101% to 120%, red from 121%.

### Query D: Maximum CPU usage per Pod

```promql
max by (workload, deployment) (label_join(label_replace(max_over_time((sum by (namespace, pod) (max by (namespace, pod, container) (rate(container_cpu_usage_seconds_total{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}[$__rate_interval]))))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))
```

Finds maximum Pod CPU usage over the selected range, then keeps the highest value per Deployment.

### Query E: CPU request divided by maximum usage (hidden diagnostic)

```promql
100 * (avg by (workload, deployment) (label_join(label_replace(sum by (namespace, pod) (kube_pod_container_resource_requests{resource="cpu",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))) / clamp_min((max by (workload, deployment) (label_join(label_replace(max_over_time((sum by (namespace, pod) (max by (namespace, pod, container) (rate(container_cpu_usage_seconds_total{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}[$__rate_interval]))))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))), 0.001)
```

Divides average CPU request by maximum observed CPU usage. Field hidden from dashboard output; use maximum CPU usage directly as safety signal.

### Query F: CPU limit per Pod

```promql
label_join(label_replace(sum by (namespace, pod) (kube_pod_container_resource_limits{resource="cpu",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment")
```

Aggregates CPU limits across containers for each Pod and derives Deployment identity. Grafana groups live Pod values by Deployment and displays all unique limit values. No average is calculated.

### Query G: CPU throttling percentage

```promql
max by (workload, deployment) (label_join(label_replace(max_over_time((100 * (sum by (namespace, pod) (rate(container_cpu_cfs_throttled_periods_total{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}[$__rate_interval]))) / clamp_min((sum by (namespace, pod) (rate(container_cpu_cfs_periods_total{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}[$__rate_interval]))), 0.001))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))
```

Calculates throttled CFS periods divided by total CFS periods, converts to percent, finds the maximum over the selected range, and keeps the highest value per Deployment. Thresholds: yellow from 5%, red from 10%.

## Deployment Memory sizing

Panel description: Deployment name is derived from Deployment-managed Pod names. Historical Pod samples remain usable while Prometheus retains them. Startup-specific metrics are excluded. Memory request and limit columns show distinct current values from live Pods; multiple values indicate a rollout or inconsistent Pod configuration. Empty request or limit values display as `N/A`.

| Query | Output column | Meaning | Visibility/use |
|---|---|---|---|
| A | `memory request set / pod` | Distinct current configured memory requests across live Pods | Visible |
| B | `p99 mem usage / pod` | Maximum per-Deployment Pod-level P99 working-set memory | Visible sizing baseline |
| C | `p99 mem usage / avg request` | P99 usage divided by average request, expressed as percent | Visible |
| D | `max mem usage / pod` | Maximum observed working-set memory per Pod | Visible safety signal |
| E | `mem request / max` | Request divided by maximum usage, expressed as percent | Hidden diagnostic |
| F | `memory limit set / pod` | Distinct current configured memory limits across live Pods | Visible |
| G | `oom kills` | Total OOM events over the selected range | Visible |

### Query A: Memory request per Pod

```promql
label_join(label_replace(sum by (namespace, pod) (kube_pod_container_resource_requests{resource="memory",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment")
```

Aggregates memory requests across containers for each Pod and derives Deployment identity. Grafana groups live Pod values by Deployment and displays all unique request values. No average is calculated.

### Query B: P99 memory usage per Pod

```promql
max by (workload, deployment) (label_join(label_replace(quantile_over_time(0.99, (sum by (namespace, pod) (max by (namespace, pod, container) (container_memory_working_set_bytes{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"})))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))
```

Calculates Pod working-set memory, computes the 99th percentile over the selected range, then keeps the highest Pod-level P99 for each Deployment.

### Query C: Memory request divided by P99 usage

```promql
100 * (max by (workload, deployment) (label_join(label_replace(quantile_over_time(0.99, (sum by (namespace, pod) (max by (namespace, pod, container) (container_memory_working_set_bytes{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"})))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment")) / clamp_min((avg by (workload, deployment) (label_join(label_replace(sum by (namespace, pod) (kube_pod_container_resource_requests{resource="memory",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))), 1))
```

Divides the Deployment's highest P99 memory usage by average memory request. `clamp_min` prevents division by values below 1 byte. Thresholds: yellow below 10%, green from 10% to 100%, yellow from 101% to 120%, red from 121%.

### Query D: Maximum memory usage per Pod

```promql
max by (workload, deployment) (label_join(label_replace(max_over_time((sum by (namespace, pod) (max by (namespace, pod, container) (container_memory_working_set_bytes{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"})))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))
```

Finds maximum working-set memory over the selected range, then keeps the highest value per Deployment.

### Query E: Memory request divided by maximum usage (hidden diagnostic)

```promql
100 * (avg by (workload, deployment) (label_join(label_replace(sum by (namespace, pod) (kube_pod_container_resource_requests{resource="memory",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))) / clamp_min((max by (workload, deployment) (label_join(label_replace(max_over_time((sum by (namespace, pod) (max by (namespace, pod, container) (container_memory_working_set_bytes{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"})))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))), 1)
```

Divides average memory request by maximum observed memory usage. Field hidden from dashboard output; use maximum memory usage directly as safety signal.

### Query F: Memory limit per Pod

```promql
label_join(label_replace(sum by (namespace, pod) (kube_pod_container_resource_limits{resource="memory",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment")
```

Aggregates memory limits across containers for each Pod and derives Deployment identity. Grafana groups live Pod values by Deployment and displays all unique limit values. No average is calculated.

### Query G: OOM kills

```promql
sum by (workload, deployment) (label_join(label_replace(sum by (namespace, pod) (increase(container_oom_events_total{container!="",container!="POD",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}[$__range])), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))
```

Counts OOM events per Pod over the selected range and sums them by Deployment. Any value at or above 1 is colored red.

## Deployment Startup CPU

Panel description: startup window is defined in the dashboard description as Pod start until first Ready time.

Actual query behavior: both queries include samples while `time() - kube_pod_start_time` is less than 300 seconds. They therefore measure the first five minutes after Pod start, not the interval ending at first Ready time.

| Query | Output column | Meaning |
|---|---|---|
| D | `startup max cpu / pod p95` | P95 of startup maximum CPU per Deployment |
| E | `startup max cpu / pod max` | Maximum startup CPU per Deployment |

### Query D: Startup CPU P95

```promql
quantile by (workload, deployment) (0.95, label_join(label_replace(max_over_time(((sum by (namespace, pod) (max by (namespace, pod, container) (rate(container_cpu_usage_seconds_total{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}[1m])))) * on(namespace, pod) group_left() ((time() - kube_pod_start_time{pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}) < bool 300))[$__range:1m]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))
```

Calculates one-minute CPU rate, keeps samples from the first 300 seconds after Pod start, finds each Pod's startup maximum, and calculates the Deployment-level P95.

### Query E: Startup CPU maximum

```promql
max by (workload, deployment) (label_join(label_replace(max_over_time(((sum by (namespace, pod) (max by (namespace, pod, container) (rate(container_cpu_usage_seconds_total{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}[1m])))) * on(namespace, pod) group_left() ((time() - kube_pod_start_time{pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}) < bool 300))[$__range:1m]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))
```

Uses the same first-five-minutes startup filter and returns the maximum startup CPU value per Deployment.

## Deployment Startup Memory

Panel description: startup window is defined in the dashboard description as Pod start until first Ready time.

Actual query behavior: both queries include samples while `time() - kube_pod_start_time` is less than 300 seconds. They measure the first five minutes after Pod start, not the interval ending at first Ready time.

| Query | Output column | Meaning |
|---|---|---|
| A | `startup max mem / pod p95` | P95 of startup maximum memory per Deployment |
| B | `startup max mem / pod max` | Maximum startup memory per Deployment |

### Query A: Startup memory P95

```promql
quantile by (workload, deployment) (0.95, label_join(label_replace(max_over_time(((sum by (namespace, pod) (max by (namespace, pod, container) (container_memory_working_set_bytes{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}))) * on(namespace, pod) group_left() ((time() - kube_pod_start_time{pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}) < bool 300))[$__range:1m]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))
```

Calculates startup working-set memory, finds each Pod's maximum during the first 300 seconds, and calculates the Deployment-level P95.

### Query B: Startup memory maximum

```promql
max by (workload, deployment) (label_join(label_replace(max_over_time(((sum by (namespace, pod) (max by (namespace, pod, container) (container_memory_working_set_bytes{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}))) * on(namespace, pod) group_left() ((time() - kube_pod_start_time{pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}) < bool 300))[$__range:1m]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))
```

Uses the same first-five-minutes startup filter and returns the maximum startup memory value per Deployment.

## CPU Recommended Sizing

Panel description: request uses P95 across Pod-level P95 CPU usage over the selected range. CPU limit candidates use the higher of startup max CPU P95 and steady-state P95 across Pod-level P95 values, with 50%, 2x, and 3x headroom options. Values are converted from cores to millicores before display: `0.1` becomes `100m`, `0.02` becomes `20m`, and `0.0023` becomes `2.3m`.

| Query | Output column | Formula | Use |
|---|---|---|---|
| A | `recommended cpu request` | P95 across Pod-level P95 CPU usage | Candidate CPU request |
| B | `recommended cpu limit +50%` | max(startup max CPU P95, steady-state P95) x 1.50 | Candidate CPU limit |
| C | `recommended cpu limit x2` | max(startup max CPU P95, steady-state P95) x 2.00 | Conservative CPU limit |
| D | `recommended cpu limit x3` | max(startup max CPU P95, steady-state P95) x 3.00 | Highest CPU limit option |

Query A uses P95 across Pod-level steady-state P95 CPU values. Queries B-D combine two separate phases, add a temporary `source` label, select the higher value per Deployment, and apply headroom. They use `max`, not addition, because startup and steady-state CPU normally occur at different times.

### Query A: Recommended CPU request

```promql
1000 * (quantile by (workload, deployment) (0.95, label_join(label_replace(quantile_over_time(0.95, (sum by (namespace, pod) (max by (namespace, pod, container) (rate(container_cpu_usage_seconds_total{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}[$__rate_interval]))))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment")))
```

### Query B: CPU limit plus 50%

```promql
1000 * (1.50 * max by (workload, deployment) (label_replace(quantile by (workload, deployment) (0.95, label_join(label_replace(max_over_time(((sum by (namespace, pod) (max by (namespace, pod, container) (rate(container_cpu_usage_seconds_total{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}[1m])))) * on(namespace, pod) group_left() ((time() - kube_pod_start_time{pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}) < bool 300))[$__range:1m]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment")), "source", "startup", "workload", ".+") or label_replace(quantile by (workload, deployment) (0.95, label_join(label_replace(quantile_over_time(0.95, (sum by (namespace, pod) (max by (namespace, pod, container) (rate(container_cpu_usage_seconds_total{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}[$__rate_interval]))))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment")), "source", "steady", "workload", ".+")))
```

### Query C: CPU limit x2

```promql
1000 * (2.00 * max by (workload, deployment) (label_replace(quantile by (workload, deployment) (0.95, label_join(label_replace(max_over_time(((sum by (namespace, pod) (max by (namespace, pod, container) (rate(container_cpu_usage_seconds_total{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}[1m])))) * on(namespace, pod) group_left() ((time() - kube_pod_start_time{pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}) < bool 300))[$__range:1m]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment")), "source", "startup", "workload", ".+") or label_replace(quantile by (workload, deployment) (0.95, label_join(label_replace(quantile_over_time(0.95, (sum by (namespace, pod) (max by (namespace, pod, container) (rate(container_cpu_usage_seconds_total{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}[$__rate_interval]))))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment")), "source", "steady", "workload", ".+")))
```

### Query D: CPU limit x3

```promql
1000 * (3.00 * max by (workload, deployment) (label_replace(quantile by (workload, deployment) (0.95, label_join(label_replace(max_over_time(((sum by (namespace, pod) (max by (namespace, pod, container) (rate(container_cpu_usage_seconds_total{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}[1m])))) * on(namespace, pod) group_left() ((time() - kube_pod_start_time{pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}) < bool 300))[$__range:1m]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment")), "source", "startup", "workload", ".+") or label_replace(quantile by (workload, deployment) (0.95, label_join(label_replace(quantile_over_time(0.95, (sum by (namespace, pod) (max by (namespace, pod, container) (rate(container_cpu_usage_seconds_total{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"}[$__rate_interval]))))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment")), "source", "steady", "workload", ".+")))
```

## Memory Recommended Sizing

Panel description: steady-state recommendation only. Request is based on per-Pod P99 memory over the selected range. Limit suggestions are based on per-Pod maximum memory over the selected range. Raw P99 and maximum usage fields remain as hidden query baselines; only recommendation fields are displayed.

| Query | Output column | Formula | Visibility/use |
|---|---|---|---|
| A | `p99 mem usage / pod` | P99 memory usage | Hidden baseline for request recommendations |
| B | `max mem usage / pod` | Maximum memory usage | Hidden baseline for limit recommendations |
| C | `recommended mem request +10%` | P99 usage x 1.10 | Visible candidate request |
| D | `recommended mem request +20%` | P99 usage x 1.20 | Visible conservative request |
| E | `recommended mem limit +50%` | Maximum usage x 1.50 | Visible candidate limit |
| F | `recommended mem limit x2` | Maximum usage x 2.00 | Visible conservative limit |
| G | `recommended mem limit x3` | Maximum usage x 3.00 | Visible highest limit option |

### Query A: P99 memory usage (hidden baseline)

```promql
max by (workload, deployment) (label_join(label_replace(quantile_over_time(0.99, (sum by (namespace, pod) (max by (namespace, pod, container) (container_memory_working_set_bytes{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"})))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))
```

### Query B: Maximum memory usage (hidden baseline)

```promql
max by (workload, deployment) (label_join(label_replace(max_over_time((sum by (namespace, pod) (max by (namespace, pod, container) (container_memory_working_set_bytes{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"})))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))
```

### Query C: Memory request plus 10%

```promql
1.10 * max by (workload, deployment) (label_join(label_replace(quantile_over_time(0.99, (sum by (namespace, pod) (max by (namespace, pod, container) (container_memory_working_set_bytes{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"})))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))
```

### Query D: Memory request plus 20%

```promql
1.20 * max by (workload, deployment) (label_join(label_replace(quantile_over_time(0.99, (sum by (namespace, pod) (max by (namespace, pod, container) (container_memory_working_set_bytes{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"})))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))
```

### Query E: Memory limit plus 50%

```promql
1.50 * max by (workload, deployment) (label_join(label_replace(max_over_time((sum by (namespace, pod) (max by (namespace, pod, container) (container_memory_working_set_bytes{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"})))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))
```

Uses historical maximum working-set memory plus 50% headroom. Memory is not throttled like CPU; exceeding the limit can trigger an OOM kill.

### Query F: Memory limit x2

```promql
2.00 * max by (workload, deployment) (label_join(label_replace(max_over_time((sum by (namespace, pod) (max by (namespace, pod, container) (container_memory_working_set_bytes{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"})))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))
```

Uses twice historical maximum working-set memory for more bursty or restart-sensitive applications.

### Query G: Memory limit x3

```promql
3.00 * max by (workload, deployment) (label_join(label_replace(max_over_time((sum by (namespace, pod) (max by (namespace, pod, container) (container_memory_working_set_bytes{container!="",container!="POD",image!="",pod=~".*-[a-z0-9]{9,10}-[a-z0-9]{5}$"})))[$__range:]), "deployment", "$1", "pod", "^(.*)-[a-z0-9]{9,10}-[a-z0-9]{5}$"), "workload", "/", "namespace", "deployment"))
```

Uses three times historical maximum working-set memory for the most conservative limit option.

## Transformations and display behavior

Each table panel uses these transformations:

1. `joinByField`: outer join query results by `workload`.
2. `organize`: removes duplicate time and label fields, orders columns, and renames query values.
3. `convertFieldType`: converts displayed query values to numbers.

Null and NaN values display as `N/A` with red coloring.

Usage-to-request percentage columns use threshold coloring:

- Yellow: below 10%, possible over-requesting.
- Green: 10% to 100%.
- Yellow: 101% to 120%, low headroom warning.
- Red: 121% and above, usage exceeds request.

These ratios use usage divided by average request. Values above 100% indicate observed high usage exceeding average request.

CPU throttling uses balanced right-sizing thresholds:

- Green: below 10%.
- Yellow: 10% and above.
- Red: 25% and above.

OOM kills use:

- Green: zero.
- Red: one or more.

## Caveats and validation checklist

- Pod-to-Deployment mapping depends on the ReplicaSet Pod-name regex.
- StatefulSets, DaemonSets, Jobs, and custom naming schemes are not represented reliably.
- Startup descriptions mention first Ready time, but startup queries use a fixed 300-second window after Pod start.
- Memory limit recommendations use steady-state maximum memory. Compare them with startup maximum memory before changing limits.
- Memory limits do not throttle usage. Exceeding a memory limit can cause an OOM kill and container restart.
- Requests and limits are averaged per Pod in sizing tables. Verify this matches the intended rollout and replica behavior.
- Deployment tables use the highest qualifying Pod value per Deployment for several usage metrics. One unusually large Pod can influence recommendations.
- P95 and P99 values depend on Prometheus sample density, retention, and selected time range.
- Memory working set is not always identical to application heap or total resident memory.
- CPU throttling can indicate restrictive limits, but interpretation depends on workload behavior and CPU quota configuration.
- OOM metrics must be available from the configured container metrics exporter.
- Validate candidate values against HPA/VPA behavior, startup probes, traffic peaks, rollout surge, node capacity, and SLOs.


