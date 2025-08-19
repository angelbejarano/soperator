# SLURM Exporter

## Overview

The SLURM Exporter is a component of Soperator that collects metrics from SLURM clusters and exports them in Prometheus format. It provides comprehensive monitoring capabilities for SLURM cluster health, job status, node states, and controller performance metrics.

The exporter integrates seamlessly with the Prometheus monitoring stack and enables observability for SLURM workloads running on Kubernetes through Soperator.

### Key Features

- **Asynchronous metrics collection** with configurable intervals (default: 30s)
- **Real-time monitoring** of SLURM nodes, jobs, and controller performance
- **Prometheus-native metrics** with standardized naming conventions
- **Rich labeling** for detailed filtering and aggregation
- **Controller RPC diagnostics** similar to SLURM's `sdiag` command
- **Kubernetes-native deployment** as part of Soperator

## Configuration

The SLURM Exporter can be configured using either command-line flags or environment variables. Environment variables take precedence over defaults but are overridden by explicitly provided command-line flags.

### Configuration Priority

The configuration follows this priority order:
1. **Command-line flags** (highest priority) - Explicitly provided flags override all other settings
2. **Environment variables** - Used when flags are not provided
3. **Default values** (lowest priority) - Used when neither flags nor environment variables are set

### Configuration Options

All configuration options can be set via command-line flags or environment variables:

| Environment Variable | Flag | Description | Default |
|---------------------|------|-------------|---------|
| `SLURM_EXPORTER_CLUSTER_NAME` | `--cluster-name` | The name of the SLURM cluster (required) | *none* |
| `SLURM_EXPORTER_CLUSTER_NAMESPACE` | `--cluster-namespace` | The namespace of the SLURM cluster | `soperator` |
| `SLURM_EXPORTER_SLURM_API_SERVER` | `--slurm-api-server` | The address of the SLURM REST API server | `http://localhost:6820` |
| `SLURM_EXPORTER_COLLECTION_INTERVAL` | `--collection-interval` | How often to collect metrics from SLURM APIs | `30s` |
| `SLURM_EXPORTER_METRICS_BIND_ADDRESS` | `--metrics-bind-address` | Address for the main metrics endpoint | `:8080` |
| `SLURM_EXPORTER_MONITORING_BIND_ADDRESS` | `--monitoring-bind-address` | Address for the self-monitoring metrics endpoint | `:8081` |
| `SLURM_EXPORTER_LOG_FORMAT` | `--log-format` | Log format: `plain` or `json` | `json` |
| `SLURM_EXPORTER_LOG_LEVEL` | `--log-level` | Log level: `debug`, `info`, `warn`, `error` | `debug` |

## Exported Metrics

### Core Metrics (Node and Job)

| Metric Name & Type | Description & Labels |
|-------------------|---------------------|
| **slurm_node_info**<br>*Gauge* | Provides detailed information about SLURM nodes<br><br>**Labels:**<br>• `node_name` - Name of the SLURM node<br>• `instance_id` - Kubernetes instance identifier<br>• `state_base` - Base node state (IDLE, ALLOCATED, DOWN, ERROR, MIXED, UNKNOWN)<br>• `state_is_drain` - Whether node is in drain state ("true"/"false")<br>• `state_is_maintenance` - Whether node is in maintenance state ("true"/"false")<br>• `state_is_reserved` - Whether node is in reserved state ("true"/"false")<br>• `address` - IP address of the node |
| **slurm_node_gpu_seconds_total**<br>*Counter* | Total GPU seconds accumulated on SLURM nodes<br><br>**Labels:**<br>• `node_name` - Name of the SLURM node<br>• `state_base` - Base node state<br>• `state_is_drain` - Drain state flag<br>• `state_is_maintenance` - Maintenance state flag<br>• `state_is_reserved` - Reserved state flag |
| **slurm_node_fails_total**<br>*Counter* | Total number of node state transitions to failed states (DOWN/DRAIN)<br><br>**Labels:**<br>• `node_name` - Name of the SLURM node<br>• `state_base` - Base node state at time of failure<br>• `state_is_drain` - Drain state flag<br>• `state_is_maintenance` - Maintenance state flag<br>• `state_is_reserved` - Reserved state flag<br>• `reason` - Reason for the node failure |
| **slurm_job_info**<br>*Gauge* | Detailed information about SLURM jobs<br><br>**Labels:**<br>• `job_id` - SLURM job identifier<br>• `job_state` - Current job state (PENDING, RUNNING, COMPLETED, FAILED, etc.)<br>• `job_state_reason` - Reason for current job state<br>• `slurm_partition` - SLURM partition name<br>• `job_name` - User-defined job name<br>• `user_name` - Username who submitted the job<br>• `user_id` - Numeric user ID who submitted the job<br>• `standard_error` - Path to stderr file<br>• `standard_output` - Path to stdout file<br>• `array_job_id` - Array job ID (if applicable)<br>• `array_task_id` - Array task ID (if applicable)<br>• `submit_time` - When the job was submitted (Unix timestamp seconds, empty if not available or zero)<br>• `start_time` - When the job started execution (Unix timestamp seconds, empty if not available or zero)<br>• `end_time` - When the job completed (Unix timestamp seconds, empty if not available or zero). **Warning:** For non-terminal states like RUNNING, this may contain a future timestamp representing the forecasted end time based on the job's time limit<br>• `finished_time` - When the job actually finished for terminal states only (Unix timestamp seconds, empty for non-terminal states or if end_time is zero). Unlike `end_time`, this field only contains actual completion times, never forecasted values |
| **slurm_node_job**<br>*Gauge* | Mapping between jobs and the nodes they're running on<br><br>**Labels:**<br>• `job_id` - SLURM job identifier<br>• `node_name` - Name of the node running the job |
| **slurm_job_duration_seconds**<br>*Gauge* | Job duration in seconds. For running jobs, this is the time elapsed since the job started. For completed jobs, this is the total execution time.<br><br>**Labels:**<br>• `job_id` - SLURM job identifier<br><br>**Notes:**<br>• Only exported for jobs with a valid start time<br>• For non-terminal states (RUNNING, etc.): duration = current_time - start_time<br>• For terminal states (COMPLETED, FAILED, etc.): duration = end_time - start_time (only if end_time is valid) |

### Controller RPC Metrics

These metrics provide insights into SLURM controller performance, similar to the output of the `sdiag` command, and were implemented to address [issue #1027](https://github.com/nebius/soperator/issues/1027).

| Metric Name & Type | Description & Labels |
|-------------------|---------------------|
| **slurm_controller_rpc_calls_total**<br>*Counter* | Total count of RPC calls by message type<br><br>**Labels:**<br>• `message_type` - Type of RPC message (e.g., REQUEST_NODE_INFO, REQUEST_JOB_INFO, REQUEST_PING) |
| **slurm_controller_rpc_duration_seconds_total**<br>*Counter* | Total time spent processing RPCs by message type (converted from microseconds)<br><br>**Labels:**<br>• `message_type` - Type of RPC message |
| **slurm_controller_rpc_user_calls_total**<br>*Counter* | Total count of RPC calls by user<br><br>**Labels:**<br>• `user` - Username making the RPC calls<br>• `user_id` - Numeric user ID |
| **slurm_controller_rpc_user_duration_seconds_total**<br>*Counter* | Total time spent on user RPCs (converted from microseconds)<br><br>**Labels:**<br>• `user` - Username making the RPC calls<br>• `user_id` - Numeric user ID |
| **slurm_controller_server_thread_count**<br>*Gauge* | Number of server threads in the SLURM controller<br><br>**Labels:** None |

### Self-Monitoring Metrics

The exporter provides self-monitoring metrics to track its own health and performance. These metrics are available on a separate endpoint (default port 8081) to avoid mixing operational metrics with business metrics.

| Metric Name & Type | Description & Labels |
|-------------------|---------------------|
| **slurm_exporter_collection_duration_seconds**<br>*Gauge* | Duration of the most recent metrics collection from SLURM APIs<br><br>**Labels:** None |
| **slurm_exporter_collection_attempts_total**<br>*Counter* | Total number of metrics collection attempts<br><br>**Labels:** None |
| **slurm_exporter_collection_failures_total**<br>*Counter* | Total number of failed metrics collection attempts<br><br>**Labels:** None |
| **slurm_exporter_metrics_requests_total**<br>*Counter* | Total number of requests to the `/metrics` endpoint<br><br>**Labels:** None |
| **slurm_exporter_metrics_exported**<br>*Gauge* | Number of metrics exported in the last scrape<br><br>**Labels:** None |

### Accessing Self-Monitoring Metrics

To access self-monitoring metrics:

```bash
# Default monitoring port
curl http://localhost:8081/metrics

# Or with custom monitoring address
./soperator-exporter --monitoring-bind-address=:9090
curl http://localhost:9090/metrics
```

## Grafana Dashboard Example

The SLURM Exporter integrates with existing Grafana dashboards. Here's an example based on the production dashboard from [nebius-solutions-library](https://github.com/nebius/nebius-solutions-library/blob/release/soperator/soperator/modules/monitoring/templates/dashboards/cluster_health.json).

