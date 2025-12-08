# ARC-12: Event Verification Service

## Metadata

- **ARC ID**: 12
- **Author(s)**: cjcobb23
- **Category**: Amplifier Protocol
- **Status**: Draft
- **Created**: 2025-07-16
- **Last Updated**: 2025-07-16
- **Target Implementation**: Q3
- **Deployment Status**: Not deployed

## Summary

This ARC proposes a generic event verification service, consisting of an onchain contract and
offchain workers. Callers can request verification of an arbitrary event on an arbitrary chain.
Off chain workers will process this request, look up the event and vote on it's validity. The
result of the verification request will be available in the contract once the verification 
process completes, which the caller can query.

## Background and Motivation

This service is requested by Squid, where a user requests a cross chain transfer via Squid's UI/app,
a market maker facilitates the transfer, and prior to distributing locked funds to the market maker 
that provided the liquidity for the transfer, Squid needs to verify the corresponding transfers did 
indeed occur on the source and destination chains with finality.

This flow does not use Axelar GMP. This process is cheaper than a typical GMP flow, because no 
additional transactions need to be made on any remote chain beyond the transfers themselves, 
whereas GMP has the overhead of calling the gateway on both the source and destination.

Squid could verify the event themselves, but that requires Squid to run nodes for every chain, or
use an RPC provider. Using Axelar reduces Squid's operational burden, while also increasing the
security of the verification process, since the event verification service we build will be
decentralized and fault tolerant.

## Requirements

- Implement an event verification contract with an API to request verification for a list of events.

- The API should be able to verify events emitted by a contract call, as well as raw transaction details (which may not emit an event)
  For example, the API should be able to verify a native asset transfer, and/or the calldata passed in a given transaction

- API should be generic across all chains and event types.

- One contract should handle requests for all chains (from the caller's perspective, can call other contracts internally).

- Ampd should query for the event via RPC to the external chain, and vote on the validity of the event. The event should be considered valid if it occurred on the chain with finality exactly as the caller specified (data should match exactly, transaction hash is correct, etc.)

- Once verification completes, a cosmwasm event should be emitted indicating verification has completed.

- Event verification contract should provide an API to query for the verification result.

## Design

### API Definition

The contract will expose an interface to verify events and register chains, as well as to query
for the verifications status of events.

#### Execute Messages

```rust
#[cw_serde]
pub enum ExecuteMsg {
    // Used to request verification of an arbitrary event on an arbitrary chain
    // Errors if the request is malformed (wrong message id format, wrong address format, wrong event format, unsupported chain)
    #[permission(Any)]
    VerifyEvents {
        events_to_verify: Vec<EventToVerify>
    },
    // Registers a chain with the verifier contract. This registration process is mainly to
    // better handle user errors, i.e. making sure the chain name is correct, all of the params
    // are formatted correctly, etc
    // This is not strictly necessary, we could just let ampd reject malformed user requests.
    // But the issue is then opaque to the caller, since they will just see that the event did
    // not verify, which could happen for any number of reasons
    #[permission(Elevated)]
    RegisterChain {
        chain_name: ChainName,
        event_format: EventFormat,
    },
}

pub enum QueryMsg {
    // for each event, returns whether the event was verified successfully,
    // not found, still being voted on, etc.
    #[returns(Vec<VerificationStatus>)]
    EventStatus(Vec<EventToVerify>),

    // returns msg_id_format, event_format and address_format associated with the chain
    #[returns(ChainInfo)]
    ChainInfo(ChainName)
}


pub struct EventToVerify {
    source_chain: ChainName,
    event_data: EventData,
}


```

#### Event Type

The `EventData` enum will be designed to support different event types across various blockchain networks.
Each chain will have a specific format for events on the chain (they are not just arbitrary blobs).
Adding structure here allows for better error handling, makes the API easier to call for the client
(don't need to serialize the event to a blob), and easier parsing by ampd (don't need to convert from
blob to chain specific event format). The contract itself will just accept a JSON string representation of the 
`EventData` type, which ampd will deserialize. Callers can use the type to construct the JSON correctly,
but the contract itself does not need to be upgraded or migrated to support new event types.


```rust
pub enum EventData {
    Evm (EvmEventData)
    // Additional event variants for other blockchain types can be added here
}

pub struct EvmEventData {
    // hash of the transaction containing the event
    transaction_id: HexBinary,
    // if present, verifies the transaction details
    transaction_details: Option<EvmTransactionDetails>,
    // verifies the presence of each event specified
    events: Vec<EvmEvent>,
}

pub struct EvmTransactionDetails {
    pub calldata: HexBinary,
    pub from: Address,
    pub to: Address,
    pub value: Uint256,
}

pub struct EvmEvent {
    pub contract_address: Address, // address of contract emitting the event
    pub event_index: u64,          // index of the event in the transaction
    pub topics: Vec<HexBinary>,    // 1-4 topics
    pub data: HexBinary,           // arbitrary length hex data
}
```


### Integration with other amplifier contracts

The event verifier will use the service registry to determine which verifiers support given chain. The existing chain names
will be reused, which means all verifiers that support chain X for GMP also support chain X for event verification.

## Changelog

Chronological log of changes to this ARC.

|  Date  | Revision  | Author |  Description  |
|--------|-----------|--------|---------------|
| 2025-07-16 | v1.0 | cjcobb23 | Add background and requirements | 
| 2025-07-21 | v1.1 | cjcobb23 | Add API | 
| 2025-08-01 | v1.2 | cjcobb23 | Update API | 
| 2025-08-20 | v1.3 | cjcobb23 | Remove address_format and msg_id_format from RegisterChain | 
| 2025-09-08 | v1.4 | cjcobb23 | Clarify event_data JSON string parsed by ampd |