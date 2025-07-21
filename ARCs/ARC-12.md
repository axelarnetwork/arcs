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
        address_format: AddressFormat,
        msg_id_format: MessageIdFormat,
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
    event_id: EventId,
    event_data: EventData,
}

pub struct EventId {
    // chain that emitted the event in question
    source_chain: ChainName,
    // same message id type as used for GMP
    message_id: String,
    // address of contract emitting the event
    contract_address: Address

}

```

#### Event Type

The `Event` enum will be designed to support different event types across various blockchain networks.
Each chain will have a specific format for events on the chain (they are not just arbitrary blobs).
Adding structure here allows for better error handling, makes the API easier to call for the client
(don't need to serialize the event to a blob), and easier parsing by ampd (don't need to convert from
blob to chain specific event format).


New event formats will require a contract migration (code only).

```rust
#[cw_serde]
pub enum EventFormat {
    Evm,
    // additional variants for other blockchains can be added here
}

#[cw_serde]
pub enum EventData {
    Evm {
        topics: Vec<Uint256>, // 1-4 topics
        data: HexBinary,      // arbitrary length hex data
    },
    // Additional event variants for other blockchain types can be added here
}
```

### Message ID

The message ID will be handled the same way as GMP, via a string that must meet certain validation rules.
It is possible to actually give the message ID proper typing, as the event data. However, we already have a
lot of code in the amplifier repo to handle message IDs as strings, as well as off chain systems (axelarons, axelarscan)
that handle message id's as strings, that it will be less engineering work to simply continue handling
the message id as a string.

Furthermore, the existing underlying message ID types (`HexTxHashAndEventIndex`, for example) are mostly internal and
were not designed to be passed as user input. For example, the `Hash` type, which is used in many of the message ID types, 
serializes to a vector of integers, which is cumbersome for a caller to pass. The `tx_digest` of `Base58TxDigestAndEventIndex`
is of type `Hash`, so callers cannot actually pass the tx digest as base58 (which is how callers pass the tx digest now, 
as a string, which get converted to a hash internally). We would need to modify the existing types to be able to pass them
as user input. Or we could create wrapper types that failably decode to the internal types; this is probably a cleaner approach,
so we can avoid retrofitting the existing types, some of which we may not even need (i.e. for chains not supported).

Lastly, having a single string to use as an identifier is useful in various contexts, both on chain and off chain.

## Future Work

Potential planned follow-ups to this ARC.

## References

Citations or resources used in creating this ARC.

## Changelog

Chronological log of changes to this ARC.

|  Date  | Revision  | Author |  Description  |
|--------|-----------|--------|---------------|
| 2025-07-16 | v1.0 | cjcobb23 | Add background and requirements | 
| 2025-07-21 | v1.1 | cjcobb23 | Add API | 