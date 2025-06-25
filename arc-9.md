# ARC-9: Prometheus Metrics Integration for Ampd

## Metadata

- **ARC ID**: 9

- **Author(s)**: Yidan Wang

- **Status**: Draft

- **Created**: 2024-06-13

- **Last Updated**: 2024-06-13

- **Target Implementation**: Q3 2025

## Summary

This ARC proposes integrating Prometheus metrics into AMPD to enhance observability. The system will expose key operational metrics via a Prometheus-compatible endpoint, enabling real-time monitoring of AMPD’s health and performance.

## Background and Motivation

As AMPD grows in complexity, observability becomes essential for maintaining security, performance, and operational reliability.

To address this, we are introducing structured, consistent, and low-overhead Prometheus metrics across core components of AMPD. 

The scope of this proposal includes:

- **Core services and modules**:
    Metrics will be instrumented in the AMPD daemon, chain connector modules (e.g., Ethereum, Cosmos, Sui), and verifier modules (e.g., Multisig, Voting).

- **Deployment environments**:
    The metrics system is designed to support both local development and production deployments, including containerized setups and Kubernetes-based infrastructure. 

This integration will enable us to:

- **Detect and investigate anomalies**  
  Counters such as `ampd_errors_total` and `contract_execution_failures_total` help surface issues such as stuck transactions, verifier misbehavior, or degraded RPC performance—enabling faster debugging and recovery.

- **Identify performance bottlenecks**  
    Histograms such as `ampd_transaction_duration_seconds` allow us to pinpoint slow chains or inefficient verification steps, supporting throughput optimization and latency reduction.

- **Ensure operational transparency and readiness**  
  Resource usage metrics like `ampd_cpu_usage_percent` and `ampd_memory_usage_bytes` help ensure AMPD runs within safe resource thresholds.


## Architecture

The proposed metrics system will be implemented as a client-server architecture with the following key components:

1. **Metrics Client**
   - Lightweight and thread-safe
   - Uses non-blocking channels to send metric updates

2. **Metrics Server**
   - Collects and exposes metrics via `/metrics` and `/status` endpoints
   - Central registry for all metrics
   - Message processing and metric updates

3. **Metrics Message**
   - Type-safe, enum-based definitions for all metrics update messages

## Detailed Design 

### Overview

The metrics system is integrated into Ampd's core components through a message-based client-server architecture. The `MetricsClient` is passed into AMPD’s core components (e.g., event handlers) and is used to emit metric updates asynchronously. Messages are processed by a central `MetricsServer`, which updates the corresponding metrics and exposes them via HTTP endpoints for external scraping.


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

The MetricsClient provides a simple API for recording metrics. It uses a bounded channel with a capacity of 1000 (CHANNEL_SIZE) to queue metric update messages. 
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

The `Server` component is responsible for receiving metric messages and exposing Prometheus-compatible endpoints. It operates in one of the two modes: 

#### Dummy Server Mode

If the Prometheus server is disabled, the server runs in dummy mode. In this mode:

- No HTTP endpoints are exposed.
- All received metric messages are discarded.
- The metrics interface remains available to the application, but no metrics are collected or exported.

#### Live Server Mode

If the Prometheus server is enabled, the server will expose HTTP endpoints for metrics collection and health checks. It processes incoming metric messages and updates the Prometheus registry accordingly.

The live server listens on the specified bind address and exposes two endpoints:

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
if let StreamStatus::Active(Event::BlockEnd(height)) = &stream_status { // specific event
    metric_client.record_metric(MetricsMsg::IncBlockReceived).ok(); // use metrics client to update metrics
}
```


## Changelog

| Date       | Revision | Author           | Description                                 |
|------------|----------|------------------|---------------------------------------------|
| 2025-xx-xx | v1.0     | Yidan Wang, Andrew Jia | Initial draft                           |
| 2025-xx-xx  | v1.1    |            |        |
| 2025-xx-xx  | v1.2    |             | |
