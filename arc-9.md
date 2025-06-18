# ARC-9: Prometheus Metrics Integration for Ampd

## Metadata

- **ARC ID**: 9

- **Author(s)**: Yidan Wang

- **Status**: Draft

- **Created**: 2024-06-13

- **Last Updated**: 2024-06-13

- **Target Implementation**: Q3 2025

## Summary

This ARC proposes the integration of Prometheus metrics into Ampd to provide comprehensive monitoring capabilities. The implementation will expose key operational metrics through a Prometheus-compatible endpoint, enabling better observability of Ampd's performance and health.

## Background and Motivation

As AMPD grows in complexity—supporting more chains, verification mechanisms, and runtime environments—observability becomes essential for maintaining security, performance, and operational reliability.

To address this, we are introducing structured, consistent, and low-overhead Prometheus metrics across core components of AMPD. The scope includes:

- **The core AMPD daemon** (transaction execution, event handling)
- **Chain connector modules** (e.g., Ethereum, Cosmos, Sui)
- **Verifier modules** (e.g., Multisig, Voting)
- **Runtime environments** (e.g., local and production deployments)

This enables us to:

- **Detect and investigate anomalies**  
  Metrics like `ampd_errors_total` and `contract_execution_failures_total` help surface issues such as stuck transactions, verifier misbehavior, or degraded RPC performance—enabling faster debugging and recovery.

- **Identify performance bottlenecks**  
  Histograms such as `ampd_transaction_duration_seconds` allow us to pinpoint slow chains or inefficient verification steps, supporting throughput optimization and latency reduction.

- **Ensure operational transparency and readiness**  
  Resource usage metrics like `ampd_cpu_usage_percent` and `ampd_memory_usage_bytes` help ensure AMPD runs within safe resource thresholds, especially in production environments with strict limits.



## Architecture

The proposed metrics system will be implemented as a client-server architecture with the following key components:

1. **Metrics Client**
   - A thread-safe interface for components to record metrics
   - Non-blocking message-based communication
   - Simple API for recording metrics without exposing Prometheus internals

2. **Metrics Server**
   - HTTP server exposing Prometheus-compatible endpoints
   - Central registry for all metrics
   - Message processing and metric updates
   - Health check endpoint

3. **Metrics Message**
   - Type-safe enum-based message definitions
   - Allows for modifying specific messages

## Detailed Design 

### Overview

The metrics system is integrated into Ampd's core components through a message-based client-server architecture. The MetricsClient is passed to key modules (ex: event handlers), enabling them to report metrics without knowing the internals of the Prometheus registry.


### Message Types Specification

Metrics updates are modeled as enum variants for type safety and extensibility:

```rust
#[derive(Debug, Clone)]
pub enum MetricsMsg {
    IncBlockReceived,
    // to be extended
}
```

### Client Implementation

The MetricsClient provides a thread-safe, non-blocking interface for recording metrics. It uses a bounded channel with a capacity of 1000 (CHANNEL_SIZE) to queue metric update messages. 
This number is chosen to handle expected peak loads, ensuring the system can process bursts of updates without exceeding resource limits.


```rust
const CHANNEL_SIZE: usize = 1000;

pub struct MetricsClient {
    sender: mpsc::Sender<MetricsMsg>,
}

impl MetricsClient {
    pub fn record_metric(&self, msg: MetricsMsg) -> Result<(), MetricsError> {
        self.sender
            .try_send(msg);
        Ok(())
    }
}
```

### Server Implementation

The `Server` component is responsible for receiving metric messages and exposing Prometheus-compatible endpoints. It operates in two modes: 

#### Live Server Mode

In this mode, the server exposes HTTP endpoints for metrics collection and health checks. It processes incoming metric messages and updates the Prometheus registry accordingly.

The live server listens on the specified bind address and exposes two HTTP endpoints:

The metrics server exposes two routes:

1. `/metrics` — Returns Prometheus-formatted metrics:
```rust
async fn gather_metrics(registry: &Registry) -> (StatusCode, String) {
    match metrics::gather(registry) {
        Ok(metrics) => (StatusCode::OK, metrics),
        Err(e) => (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()),
    }
}
```

2. `/status` — Returns a simple OK response for health checks:
```rust
async fn status() -> (StatusCode, Json<Status>) {
    (StatusCode::OK, Json(Status { ok: true }))
}
```

#### Dummy Server Mode

If no bind address is provided, the server runs in dummy mode. In this mode:

- No HTTP endpoints are exposed.
- All received metric messages are discarded.
- The metrics interface remains available to the application, but no metrics are collected or exported.



### Metrics Collection

Metrics are stored in a struct and registered to a central Registry. Incoming messages are handled using a match block to update the appropriate metrics.

```rust
pub struct Metrics {
    metrics: MetricsType,
}

impl Metrics {
    pub fn new(registry: &Registry) -> Result<Self, MetricsError> {
        let metric = IntCounter::new("metrics_name", "description");
        // register metrics
        registry
            .register(Box::new(metric.clone()));

        Ok(Self { metric })
    }

    pub fn handle_message(&self, msg: MetricsMsg) {
        match msg {
            MetricsMsg::MetricsMsgType=> {
                self.metrics.inc(); // Each metric type may have unique handling requirements
                ...
            }
        }
    }
}
```

### Component Integration

The MetricsClient is injected into key components during application setup, enabling them to record metrics. For example, an event handler can use the MetricsClient to track metrics for specific events:


```rust
if let StreamStatus::Active(Event::BlockEnd(height)) = &stream_status {
    metric_client.record_metric(MetricsMsg::IncBlockReceived).ok();
}
```



## Future Work

### Metrics Expansion and Deployment

- Define and implement additional metrics for verifier status, transaction results, and error rates
- Extend Dockerfile and Helm charts to support metrics exposure and scraping

### Custom Event Metrics Interface

- Introduce a developer-friendly interface to define custom metrics per event type
- Create a base schema for common metrics and utility functions for recording


## Changelog

| Date       | Revision | Author           | Description                                 |
|------------|----------|------------------|---------------------------------------------|
| 2025-xx-xx | v1.0     | Yidan Wang, Andrew Jia | Initial draft                           |
| 2025-xx-xx  | v1.1    |            |        |
| 2025-xx-xx  | v1.2    |             | |
