
# ARC-3: Chain Info Endpoint For Coordinator

  

## Metadata

  

-  **ARC ID**: 3

-  **Author(s)**: Houmaan Chamani

-  **Status**: Draft

-  **Created**: 2025-01-14

-  **Last Updated**: 2025-01-17

-  **Target Implementation**: Q1 2025

  

## Summary

This ARC proposes a standardized query system for retrieving chain data from the Coordinator contract. It introduces two main query endpoints: one for retrieving filtered lists of chain endpoints with pagination support, and another for accessing detailed chain configurations including its prover and ITS (Interchain Token Service) settings. This design is aimed at increasing the maintainability of the system by providing aggregated data rather than having multiple specific query messages for each need.

  

## Motivation

  

### Background
The way most amplifier query endpoints have been treated so far required individual messages for each use case. This led to: 
- Overhead of maintaining numerous specific endpoints. This is because small changes in design or the code might require multiple endpoints and the their logical paths to be updated.
- Potential performance issues with multiple separate queries, where a client has to send multiple requests to get what they want
- Limited query reusability. Each data request requires a specific query that returns only narrowly-defined information, making these queries difficult to repurpose for other use cases.

  

### Goals
- Simplify the query interface by consolidating related queries into unified endpoints 
- Provide filtering capabilities to handle various use cases 
- Implement efficient pagination to manage large result sets 
- Shift data extraction to client-side for specific needs 
- Reduce the need for frequent message modifications

  

## Detailed Design

  

### Overview
The proposed system introduces two standardized query structures.

  

### Technical Specification
#### Query Structure 

```rust
pub enum QueryMsg {
    // Get chains filtered by frozen status, with pagination
    GetChains {
        frozen_filter: Option<FrozenFilter>,
        pagination: Option<PageConfig>,
    },

    // Get detailed chain info by identifier
    GetChainInfo {
        identifier: ChainIdentifier,
    }
}

// Chain identifier can be either name or gateway address
enum ChainIdentifier {
    ByChainName: ChainName, 
	ByGatewayAddress: Addr,
}

// Filter with predefined options
enum FrozenFilter {
    FrozenIncoming,
    FrozenOutgoing,
    FrozenBidirectional,
    NotFrozen,
}

// Pagination
struct PageConfig {
    // Start after this key (None for first page)
    start_after: Option<String>,  

    // Maximum items per page
    limit: u32,
}
```



#### Response Structure and Example

```rust
pub struct ChainListResponse {
    chains: Vec<ChainEndpoint>,
    pagination: PageInfo,
}

pub struct PageInfo {
    // Key to use for next page query (None if no more pages)
    next_key: Option<String>,
    total: u64,
}

// Detailed chain info response
pub struct ChainInfoResponse {
    chain_endpoint: ChainEndpoint,
    provers_config: Vec<ProverConfig>,
    its_config: ITSChainConfig,
}

pub struct ChainEndpoint {
	pub name: ChainName,
	pub gateway: Gateway,
	pub frozen_status: FlagSet<GatewayDirection>,
	pub msg_id_format: MessageIdFormat,
}

struct ProverConfig {
	pub prover_address: ProverAddress,
	pub active_verifiers: HashSet<VerifierAddress>,
}

pub struct ITSChainConfig {
	pub its_address: Address,
	pub truncation: TruncationConfig,
	frozen: bool,
}

pub struct TruncationConfig {
	pub max_uint: nonempty::Uint256,
	pub max_decimals_when_truncating: u8,
}

```




## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|---------|------------|------------|
| Query Performance | High | Medium | TODO! |
| Response Size | Medium | High | Enforce pagination, compress responses |
| Filter Complexity | Medium | Low | Limit filter depth, validate inputs |

### Detailed Risk Analysis

#### Query Performance
**Risk**: The coordinator contract needs to aggregate data from multiple sources to fulfill queries, which could lead to poor performance if data needs to be collected (pullled) during query execution.

**Mitigation**: 
- Implement a push-based state update system where contracts update their state in the coordinator
- Each contract pushes relevant state changes to the coordinator when they occur
- Coordinator maintains an up-to-date cache of all required data in its state
- Queries can be served directly from coordinator state without cross-contract communication

#### Response Size
**Risk**: Query responses may contain large data depending on filter criteria. This could potentially cause gas issues or timeout problems.

**Mitigation**:
- Implement cursor-based pagination using the response's `next_key` field
- Set mandatory page size limits (e.g., maximum 50 items per page)
- Allow clients to specify smaller page sizes if needed
- Include total count in response to help clients plan pagination strategy


#### Filter Complexity
**Risk**: Complex or arbitrary filter conditions could lead to implementation difficulties and potential performance issues.

**Mitigation**:
- Define a fixed set of allowed filter criteria rather than allowing arbitrary filters to avoid the need for custom filter interpreters
- Clearly document supported filter operations
- Return clear error messages for unsupported filter combinations



### References

  

### Changelog

  

| Date | Revision | Author | Description |
|------|-----------|---------|-------------|
| 2024-01-14 | v1.0 | Houmaan Chamani | Initial ARC draft |
| 2024-01-15 | v1.1 | Houmaan Chamani | Add details to response structure |
| 2024-01-17 | v1.2 | Houmaan Chamani | Risk analysis and endpoint definition refinement |
