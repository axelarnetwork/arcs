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

- One contract should handle requests for all chains (from the callers perspective, can call other contracts internally).

- Ampd should query for the event via RPC to the external chain, and vote on the validity of the event. The event should be considered valid if it occurred on the chain with finality exactly as the caller specified (data should match exactly, transaction hash is correct, etc.)

- Once verification completes, a cosmwasm event should be emitted indicating verification has completed.

- Event verification contract should provide an API to query for the verification result.

## Design

Details on how this ARC will be structured and implemented.

## Future Work

Potential planned follow-ups to this ARC.

## References

Citations or resources used in creating this ARC.

## Changelog

Chronological log of changes to this ARC.

|  Date  | Revision  | Author |  Description  |
|--------|-----------|--------|---------------|
| 2025-07-16 | v1.0 | cjcobb23 | Add background and requirements | 