# ARC-9: Prometheus Metrics Integration for Ampd

## Metadata

- **ARC ID**: 9

- **Author(s)**: Yidan Wang

- **Category**: Amplifier Protocol

- **Status**: Implementation

- **Created**: 2024-06-13

- **Last Updated**: 2024-08-14

- **Target Implementation**: Q3 2025

## Summary

This ARC proposes integrating an in-process monitoring server into AMPD that exposes a Prometheus-compatible `/metrics` endpoint. The server will publish essential operational metrics to provide real-time visibility into AMPD's health, performance, and operational status. It operates locally with configurable host and port settings, and can be optionally disabled when monitoring is not required.

## Background and Motivation

AMPD currently lacks built-in monitoring capabilities, making it difficult to observe operational health and performance in real time. For a system coordinating critical cross-chain workflows, this monitoring gap increases the risk that performance degradations or operational failures remain invisible until they directly impact system reliability and user experience.

The goal is to provide clear, consistent operational signals so issues are caught early, diagnosed quickly, and turned into actionable alerts. The design should focus on operational performance, add minimal overhead, stay decoupled from core components, and scale with on-call workflows as the system and its supported chains grow.


## Design

The proposed monitoring system implements a client-server architecture that provides operational visibility into AMPD's performance and health. The system consists of three main components:

- **Monitoring Client**: A lightweight client that can be cloned across components, using bounded channels to send metric updates asynchronously
- **Monitoring Server**: A configurable server that exposes HTTP endpoints for metrics collection and health checks
- **Message Types**: Type-safe definitions for various operational events

### Monitoring Client:

Monitoring client has two modes: when enabled, it sends typed events over a bounded channel; when disabled, it discards all metrics without any overhead. 

```rust

#[derive(Clone, Debug)]
pub enum Client {
    WithChannel { sender: mpsc::Sender<Msg> },
    Disabled,
}

impl Client {
    pub fn record_metric(&self, msg: Msg) {
        match self {
            Client::Disabled => (),
            Client::WithChannel { sender } => {
               let _ = sender.try_send(msg.clone());
            }
        }
    }
}
```

The `MonitoringClient` is injected into AMPD's core components (e.g., event handlers, broadcaster) and emits metric updates asynchronously.

```rust
// Example: Recording event timeout in event processor
pub async fn consume_events(
    // ... other parameters
    monitoring_client: monitoring::Client, // monitoring client injected here
) {
    // ... other consume event logic
    match stream_status {
        StreamStatus::TimedOut => {
            warn!("event stream timed out");
            monitoring_client
                .metrics()
                .record_metric(Msg::EventStreamTimeout);
        }
    }
}
```


### 2. Monitoring Server 

The monitoring server operates as a **central consumer** that processes incoming metric messages and maintains the Prometheus registry. It runs as a background task that continuously processes messages from the bounded channel, updating various metric types including **counters** and **gauges** based on the message content.

**HTTP Endpoints:**
- **`/metrics`**: Prometheus-formatted metrics collection endpoint that retrieves data from the internal Prometheus registry and formats it according to the OpenMetrics standard for external monitoring systems to scrape
- **`/status`**: Simple health check endpoint that verifies the monitoring system is operational

**Operational Modes:**
- **Enabled**: HTTP endpoints are exposed and metrics are processed normally
- **Disabled**: No HTTP endpoints are exposed, all metric messages are discarded, but the monitoring interface remains available to the application. This design allows the system to operate with minimal overhead when monitoring is not required.

### 3. Message Types

The monitoring system uses typed message enums to ensure type safety and prevent cardinality issues. Each message type represents a specific operational event that contributes to system visibility. The system tracks various operational aspects including block processing, verification votes, stage performance, system health, and error conditions.


```rust
pub enum Msg {
    /// Increment the count of blocks received
    BlockReceived,  
    /// Record the number of errors in message enqueue operations          
    MessageEnqueueError,    
     /// Record the number of timeouts in event stream
    EventStreamTimeout,      
    // ... other message types
}
```

The following table describes the currently implemented metrics:

| Metric Name | Metric Type | Labels | Description |
|-------------|-------------|---------|-------------|
| `blocks_received_total` | Counter | None | Total number of blocks received by the system |
| `verification_votes_total` | Counter | `vote`, `chain_name` | Verification vote results for cross-chain messages by handlers |
| `msg_enqueue_error_total` | Counter | None | Number of failures in message enqueue operations |
| `event_stream_timeout_total` | Counter | None | Number of timeouts in event stream processing |
| `event_publisher_error_total` | Counter | None | Number of errors in event publisher operations |
| `grpc_service_error_total` | Counter | None | Number of errors in gRPC service calls |
| `stage_processed_total` | Counter | `stage` | Number of successfully processed items per stage (event handling, transaction broadcast, transaction confirmation) |
| `stage_duration_seconds` | Counter | `stage` | Duration of processing items per stage in milliseconds (event handling, transaction broadcast, transaction confirmation) |
| `stage_failed_total` | Counter | `stage` | Number of failed items per stage (event handling, transaction broadcast, transaction confirmation) |
| `rpc_calls_total` | Counter | `chain_name` | Total number of RPC calls made to external chains |
| `rpc_calls_failed_total` | Counter | `chain_name` | Number of RPC calls that failed |


## Changelog

| Date       | Revision | Author           | Description                                 |
|------------|----------|------------------|---------------------------------------------|
| 2025-06-15 | v1.0 | Yidan Wang, Andrew Jia | Initial draft |
| 2025-07-04 | v1.1 | Christian Gorenflo | Restructure monitoring design |
| 2025-08-14 | v1.2 | Yidan Wang | Add metrics implementation and restructure ARC |
