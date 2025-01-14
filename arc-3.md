
# ARC-3: Chain Info Endpoint For Coordinator

  

## Metadata

  

-  **ARC ID**: 3

-  **Author(s)**: Houmaan Chamani

-  **Status**: Draft

-  **Created**: 2025-01-14

-  **Last Updated**: 2025-01-14

-  **Target Implementation**: Q1 2025

  

## Summary

  

Implement a unified query system for the Coordinator contract that provides aggregated data through one single endpoint. This reduces the need for multiple specific query messages, which are hard to maintain. It also helps with the scalability and flexibility of the system.

  
  

## Motivation

  

### Background
The way we have been treating most query endpoints so far required individual messages for each use case. This led to: 
- Overhead of maintaining numerous specific endpoints 
- Reduced code maintainability 
- Potential performance issues with multiple separate queries, where a client has to send multiple requests to get what they want

  

### Goals
- Simplify the query interface by consolidating related queries into unified endpoints 
- Provide filtering capabilities to handle various use cases 
- Implement efficient pagination to manage large result sets 
- Shift data extraction to client-side for specific needs 
- Reduce the need for frequent message modifications

  

## Detailed Design

  

### Overview
The proposed system introduces a standardized query structure with three main components: 
1. Query Key: Identifies the type of data being queried
2. Filter: Allows for precise data filtering 
3. Pagination: Manages response size and performance

  

### Technical Specification
#### Query Structure 
```
struct QueryRequest { 
	key: QueryKey, 
	filter: Option<QueryFilter>, 
	pagination: Option<PageRequest>, 
}

enum QueryKey { 
	ByChainName: ChainName, 
	ByGatewayAddress: Addr, 
}

struct QueryFilter { 
	conditions: Vec<FilterCondition>, 
	operator: FilterOperator,
}

struct FilterCondition { 
	field: String, 
	operator: ComparisonOperator, 
	value: Value, 
}

struct PageRequest { 
	page: u32, 
	limit: u32, 
	order_by: Option<String>, 
}
```


#### Response Structure 
```
struct QueryResponse<T> {
    data: Vec<T>,
    pagination: PageResponse,
}

struct PageResponse {
    next_key: Option<Binary>,
    total: u64,
}

// Example Chain Response
struct ChainData {
    chain_endpoint: ChainEndpoint,
    gateway: Gateway,
}

```
  

  

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|---------|------------|------------|
| Query Performance | High | Medium | TODO! |
| Response Size | Medium | High | Enforce pagination, compress responses |
| Filter Complexity | Medium | Low | Limit filter depth, validate inputs |



### References

  

### Changelog

  

| Date | Revision | Author | Description |
|------|-----------|---------|-------------|
| 2024-01-14 | v1.0 | Houmaan Chamani | Initial ARC draft |
