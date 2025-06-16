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

AMPD is a core component in Axelar's infrastructure that verifies cross-chain messages. Its reliability and correctness are critical for secure interchain communication. As the complexity and scale of the system grow, observability becomes essential to:

- Detect and investigate anomalies (e.g., failed transactions, verifier issues)
- Identify performance bottlenecks (e.g., block processing delays)
- Provide confidence to operators and SREs through quantitative signals

This ARC ensures Prometheus metrics are exposed in a structured, consistent, and low-overhead manner.

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

## Detailed Design (will be updated in the future)

### Overview

The metrics system is integrated into Ampd's core components through a message-based client-server architecture. The MetricsClient is passed to key modules (ex: event handlers), enabling them to report metrics without knowing the internals of the Prometheus registry.


### Message Types Specification

Metrics updates are modeled as enum variants for type safety and extensibility:

```rust
#[derive(Debug, Clone)]
pub enum MetricsMsg {
    IncBlockReceived,
    // will add other metrics here with explanation
}
```

### Client Implementation

The MetricsClient provides a non-blocking interface to send metric update messages over a bounded channel:

```rust
pub struct MetricsClient {
    sender: mpsc::Sender<MetricsMsg>,
}

impl MetricsClient {
    pub fn record_metric(&self, msg: MetricsMsg) -> Result<(), MetricsError> {
        self.sender
            .try_send(msg)
            .change_context(MetricsError::MetricUpdateFailed)?;
        Ok(())
    }
}
```

### Server Implementation

The Server component initializes a receiver and an optional HTTP listener:

```rust
pub struct Server {
    bind_address: Option<SocketAddrV4>,
    metrics_rx: mpsc::Receiver<MetricsMsg>,
}

impl Server {
    pub fn new(bind_address: Option<SocketAddrV4>) -> Result<(Self, MetricsClient), MetricsError> {
        let (tx, rx) = mpsc::channel(CHANNEL_SIZE);
        let client = MetricsClient::new(tx);
        let server = Self {
            bind_address,
            metrics_rx: rx,
        };
        Ok((server, client))
    }
}
```

The server processes incoming metric messages and exposes /metrics via an Axum router.

**Note:** If no bind address is provided, the server starts in a dummy mode. In this mode, it receives metric messages but does nothing with them—no HTTP endpoints are exposed, and no metrics are collected or exported. This ensures that the metrics interface remains available to the application even when metrics collection is not required.

### Metrics Collection

Metrics are stored in a struct and registered to a central prometheus::Registry. Messages are handled using a match block:

```rust
pub struct Metrics {
    metrics: IntCounter,
}

impl Metrics {
    pub fn new(registry: &Registry) -> Result<Self, MetricsError> {
        let block_received = IntCounter::new("metrics_name", "description")
            .change_context(MetricsError::MetricSpawnFailed)?;
        // register metrics
        registry
            .register(Box::new(block_received.clone()))
            .change_context(MetricsError::MetricRegisterFailed)?;

        Ok(Self { block_received })
    }

    pub fn handle_message(&self, msg: MetricsMsg) {
        match msg {
            MetricsMsg::MetricsMsgType=> {
                self.metrics.inc(); // handling logic here , will be different depending on the metrics
                ...
            }
        }
    }
}
```

### Component Integration

The metrics client is injected during application setup. For example, event handlers receive it during configuration:

```rust
fn create_handler_task<L, H>(
    &mut self,
    label: L,
    handler: H,
    event_processor_config: event_processor::Config,
    metric_client: MetricsClient,  // Injected here
) -> CancellableTask<Result<(), event_processor::Error>>
```

Within the event handler, metrics are recorded as follows:

```rust
if let StreamStatus::Active(Event::BlockEnd(height)) = &stream_status {
    if let Err(err) = metric_client.record_metric(MetricsMsg::IncBlockReceived) {
        warn!(
            handler = handler_label,
            height = height.value(),
            err = %err,
            "failed to update metrics for block end event"
        );
    }
}
```

### HTTP Endpoints

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

## Future Work

### Metrics Expansion and Deployment

- Define and implement additional metrics for verifier status, transaction results, and error rates
- Update CI/CD pipelines to deploy Ampd with Prometheus endpoints enabled
- Extend Dockerfile and Helm charts to support metrics exposure and scraping

### Custom Event Metrics Interface

- Introduce a developer-friendly interface to define custom metrics per event type
- Create a base schema for common metrics and utility functions for recording
- Document best practices for metrics instrumentation within Ampd

## Changelog

| Date       | Revision | Author           | Description                                 |
|------------|----------|------------------|---------------------------------------------|
| 2025-xx-xx | v1.0     | Yidan Wang, Andrew Jia | Initial draft                           |
| 2025-xx-xx  | v1.1    |            |        |
| 2025-xx-xx  | v1.2    |             | |
