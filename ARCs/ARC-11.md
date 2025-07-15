# ARC-11: Chain Codec Contract

## Metadata

- **ARC ID**: 11
- **Author(s)**: Christoph Otter
- **Category**: Amplifier Protocol
- **Status**: Draft
- **Created**: 2025-07-15
- **Last Updated**: 2025-07-15
- **Target Implementation**: Q3 2025

## Summary

This ARC proposes the introduction of a new contract interface to simplify chain integrations and make them easier to maintain.
It proposes to move chain-specific logic out of the multisig-prover and voting-verifier contracts into a new contract that both of them will query.

## Background and Motivation

Currently when adding a new chain, three contracts need to be provided / implemented: the gateway, the voting-verifier and the multisig-prover. This proposal specifically focuses on the multisig-prover and voting-verifier contracts.

The multisig-prover's main function is to manage proofs about messages or validator set updates.
To do that, it needs to send signing requests for them to the multisig contract, keep track of verifier sets and transform payloads into a format that can be processed by the destination chain. In most cases, the logic for proof construction and tracking the verifier set is not chain-specific, but the contract had to be forked to adjust the payload transformation logic, leading to code duplication and maintenance overhead.

A similar situation exists for the voting-verifier contract, which is responsible for managing the verification process before the messages are routed and signed (e.g. that the messages really were sent on the source chain). It validates that the address of the sender address is a correct according to the source chain's address format. Other than that, it's the same for most chains (see Aleo and Solana contract).

The goal of this ARC is to adjust the multisig-prover and voting-verifier contracts in such a way that all this chain-specific logic can be implemented without the need for contract duplication.

## Requirements

It should still be possible for chains to fork the multisig-prover and voting-verifier contracts if they need to implement chain-specific logic that cannot be expressed through the new interface.

## Design

This ARC proposes to move the chain-specific logic into a separate chain-codec contract, which will be responsible for transforming the payloads and validating addresses. The multisig-prover and voting-verifier contracts will then query the chain-codec contract to perform their chain-specific conversions.

For the multisig-prover, the chain-specific part seems to be limited to the transformation of the payload into a format that can be processed by the destination chain (see for example the Solana contract linked below). This logic is currently controlled by an enum passed in the `InstantiateMsg` of the multisig-prover contract.
It will be replaced by a query to the chain-codec contract.

For the voting-verifier, the chain-specific part is the validation of the address format. This logic is currently implemented in the `validate_address` function of the `axelar-wasm-std` crate. It will also be replaced by a query to the chain-codec contract.

Consequently, the chain-codec contract will implement the following interface:

```rust
pub enum QueryMsg {
    /// Returns a digest for identifying a payload. This is what gets signed by the verifiers.
    #[returns(Hash)]
    PayloadDigest {
        domain_separator: Hash,
        signer: VerifierSet,
        payload: Payload,
    }
    /// Encodes the execute data for the target chain that the relayer will submit.
    #[returns(HexBinary)]
    EncodeExecData {
        domain_separator: Hash,
        verifier_set: VerifierSet,
        signers: Vec<SignerWithSig>,
        payload: Payload,
    }
    /// Returns `true` iff the given address is formatted as a valid address on the chain.
    #[returns(bool)]
    ValidateAddress {
        address: Address,
    }
}
```

In addition to this new contract, we should also add the optional `sig_verifier` address (used in multisig-prover when constructing a proof) to the multisig-prover's `InstantiateMsg`, since that is also a reason to fork the contract at the moment (see for example the Aleo contract). It can be stored in the contract's config and retrieved when needed.

The new chain-codec contract will have to be deployed before the multisig-prover and voting-verifier contracts and provided to them in their `InstantiateMsg` as a parameter. This means we will have to make some changes to the one-click deployment described in [ARC-8](./ARC-8.md). It will also have to deploy the chain-codec contract and pass its address to the multisig-prover and voting-verifier contracts.
This will require either deploying the chain-codec contract first and only deploying the others in a reply to the `InstantiateMsg` or using CosmWasm's `Instantiate2` feature to deploy the chain-codec contract with a predetermined address.

## References

Aleo contracts: https://github.com/eigerco/axelar-amplifier/tree/aleo-its-mr/contracts

Solana contracts: https://github.com/eigerco/axelar-amplifier/tree/solana-cosmwasm/contracts

## Changelog

Chronological log of changes to this ARC.

|  Date  | Revision  | Author |  Description  |
|--------|-----------|--------|---------------|
| 2025-07-15 | v1.0 | Christoph Otter | Initial draft |
