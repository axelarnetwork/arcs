
# ARC-3: Query System Design for Amplifier State

  

## Metadata

  

-  **ARC ID**: 3

-  **Author(s)**: Houmaan Chamani

-  **Status**: Draft

-  **Created**: 2025-01-14

-  **Last Updated**: 2025-01-21

-  **Target Implementation**: Q1 2025

  

## Summary

This ARC defines the requirements and design for query interfaces across the Amplifier contracts. It aims to identify all necessary queries required by engineering teams, determine the appropriate contract for each query, and define their specific interfaces. The goal is to clearly define the new responsiblities of each contract and ensure all necessary state information is accessible through maintainable query endpoints.


## Motivation

  

### Background

Amplifier requires various components to expose their state through query interfaces. Currently, there is no standardized approach to designing these queries, leading to several challenges:
- Missing queries for newly added contracts such as ITS
- Lack of clear visibility into which contract owns and should expose specific state data
- Uncertainty about what queries are needed by different engineering teams and clients

  

### Goals
- Document comprehensive query requirements from all engineering teams
- Establish clear ownership of state data and corresponding queries in appropriate contracts
- Design consistent interfaces for common query patterns (filtering, pagination)
- Ensure efficient access to necessary state data with minimal query complexity
- Create maintainable query endpoints that respect contract boundaries
  



## Requirements

### Chain-Ops
- Interchain-Token-Service:
    - Extend `ItsContract` query to include truncation configuration
    - Add contract status information (`Disabled` or `Enabled`)
    - Retrieve list of frozen chains (chains affected by `FreezeChain`)
    - Retrieve list of active chains (unfrozen)
- Router:
    - Get the paginated list of chains with filter for frozen status and direction


### Axelarons
- Implement a unified lookup query that accepts any one of these identifiers and returns the complete set of associated addresses:
    - Chain name
    - Gateway address
    - Voting verifier address
    - Multisig prover address


No additional query requirements were identified from other engineering teams at this time.




## Detailed Design

  

### Overview
The proposed system introduces six standardized query structures.
  

### Technical Specification

#### Interchain-Token-Service

```rust
pub struct ChainConfig {
    pub chain: ChainNameRaw,
    pub its_edge_contract: Address,
    pub truncation: TruncationConfig,
}


pub enum QueryMsg {

    #[returns(Option<ChainConfig>)]
    ChainInfo { chain: ChainNameRaw },

    #[returns(Vec<ChainNameRaw>)]
    AllFrozenChains,

     #[returns(Vec<ChainNameRaw>)]
    AllActiveChains,

    #[returns(bool)]
    IsEnabled,

    // existing queries
}
```


#### Router
```rust
pub struct ChainEndpoint {
	pub name: ChainName,
	pub gateway: Gateway,
	pub frozen_status: FlagSet<GatewayDirection>,
	pub msg_id_format: MessageIdFormat,
}

// Filter with predefined options
enum FrozenFilter {
    FrozenIncoming,
    FrozenOutgoing,
    FrozenBidirectional,
    NotFrozen,
}

pub enum QueryMsg {

    // Returns a filtered (optional) list of chains registered with the router
    #[returns(Vec<ChainEndpoint>)]
    Chains {
        frozen_filter: Option<FrozenFilter>,
        start_after: Option<ChainName>,
        limit: Option<u32>,
    },

    // existing queries
}
```


#### Coordinator

```rust
pub struct ContractDiscoveryResponse {
    chain_name: ChainName,
    gateway_address: Address,
    verifier_address: Address,
    prover_address: Address,
}

pub enum ContractIdentifier {
    ChainName(ChainName),
    GatewayAddress(Address),
    VerifierAddress(Address),
    ProverAddress(Address),
}

pub enum QueryMsg {

    #[returns(ContractDiscoveryResponse)]
    ContractDiscovery {
        identifier: ContractIdentifier,
    },

    // existing queries
}
```



## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|---------|------------|------------|
| Router Query: Response Size | Medium | High | Enforce pagination, page size |
| Router Query: Filter Complexity | Medium | Low | Pre-defined filter options |

### Detailed Risk Analysis

#### Router Query: Response Size
**Risk**: Query responses may contain large data depending on filter criteria. This could potentially cause gas issues or timeout problems.

**Mitigation**:
- Implement pagination using the `start_after` field
- Set mandatory page size limits (e.g., maximum 50 items per page)


#### Router Query: Filter Complexity
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
| 2024-01-21 | v1.3 | Houmaan Chamani | Reformat the arc and break down queries for each contract |
