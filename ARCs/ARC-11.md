# ARC-11: Chain Codec Contract

## Metadata

- **ARC ID**: 11
- **Author(s)**: Christoph Otter
- **Category**: Amplifier Protocol
- **Status**: Draft
- **Created**: 2025-07-15
- **Last Updated**: 2025-07-25
- **Target Implementation**: Q3 2025

## Summary

This ARC proposes the introduction of a `chain-codec` contract to encapsulate chain-specific logic, such as payload transformation and address validation. This decouples that logic from the `multisig-prover` and `voting-verifier` contracts to simplify chain integrations and make them easier to maintain.

## Background and Motivation

Currently, integrating a new chain requires deploying and maintaining three contracts: the `gateway`, `multisig-prover`, and `voting-verifier`. This ARC focuses on the latter two.
The following sub-sections describe the current architecture and problems with it.

### Multisig-Prover

The multisig-prover:

- receives proof construction requests from the relayer and starts a signing session for the proof in the `multisig` contract. The signing does not happen on the full payload, but rather a digest / hash of the chain-specifically encoded payload.
- replies to proof queries from the relayer with the proof status and some metadata. If the proof is completed, this contains the encoded bytes the relayer should send to the destination chain gateway.
- manages an active verifier set and its updates.

Only the payload transformation and digest calculation are chain-specific.

This logic is currently implemented in the multisig-prover contract using an `EncoderExt` trait. The specific implementation to be used is passed in as an enum in the `InstantiateMsg`. This leads to code duplication and maintenance overhead when integrating new chains because they have to fork the whole contract.
Here is the `EncoderExt` trait for later reference:

```rust
pub trait EncoderExt {
    fn digest(
        &self,
        domain_separator: &Hash,
        verifier_set: &VerifierSet,
        payload: &Payload,
    ) -> Result<Hash, ContractError>;

    fn execute_data(
        &self,
        domain_separator: &Hash,
        verifier_set: &VerifierSet,
        sigs: Vec<SignerWithSig>,
        payload: &Payload,
    ) -> Result<HexBinary, ContractError>;
}
```

#### Differences between Chain Integrations

In some chain integrations, such as XRPL, the proof construction logic requires more customization. Here, the payload is not generated inside the contract, but passed into the contract by the relayer. The contract only validates it. In these integrations, the multisig-prover contract needs to receive the already-encoded payload bytes in the `ConstructProof` message from the relayer, which is not supported by the current design, so this is also a reason for forking.

During the proof construction, the XRPL integration also stores some additional information in the contract state.

There is also another case where forking is currently required: the Aleo integration requires a `sig_verifier` address to be passed to the `multisig` contract, but in the multisig-prover contract, this is currently hardcoded to `None`.

### Voting-Verifier

The `voting-verifier` contract is responsible for managing the verification process of incoming messages before they are routed.
It does the following:

- validates that the messages' source chain is correct and the sender address is correctly formatted
- creates and manages polls for incoming messages and verifier set updates and records the votes of verifiers

There are two chain-specific parts here:

- the validation of the address format
- in one place it splits the message id into its constituents, which works differently depending on the chain, in order to emit the `tx_id` and `event_index` as event attributes. These attributes are deprecated, but might still be used.

### Problems with the Current Approach

Chain-specific logic is embedded within each contract. Some of it, like the `MessageIdFormat` and `AddressFormat`, is bundled outside of the contracts in the `axelar-wasm-std` crate, which is better, but leads to a lot of dependencies on external crates and still requires upstreaming changes or forking. Whenever we change something in these contracts, all the forks need to be updated.

Overall, this shows limited modularity, code duplication, and therefore, maintenance overhead.

## Proposed Solution Design

Introduce a **dedicated `chain-codec` contract** to encapsulate the chain-specific logic as a single contract per chain. Both the `multisig-prover` and `voting-verifier` will query / call this contract instead of implementing such logic internally.

### Chain-Codec Responsibilities

1. **Payload Transformation**:
    - provides `digest` for the multisig-prover's `ConstructProof` message. This is implemented as an ExecuteMsg to allow the contract to persist additional state if needed (like e.g. the XRPL integration described above).
    - provides `execute_data` for the multisig-prover's `Proof` query (via `QueryMsg`).

2. **Address Validation**:
    - validates sender addresses according to the chain-specific format.

Note that the `MessageIdFormat` is not part of the new `chain-codec` interface, as its usage in `voting-verifier` is deprecated anyway and it can continue to live in the `axelar-wasm-std` crate.

### Changes to Existing Contracts

To support integrations like the XRPL integration, the multisig-prover contract will be updated to include a `payload: HexBinary` field in the `ConstructProof` message when compiled with a new `receive-payload` feature flag. This allows the relayer to pass the encoded payload bytes directly to the multisig-prover contract, which will then forward it to the `chain-codec` contract. This is required by some integrations, such as XRPL.

The `multisig-prover` and `voting-verifier` contracts will be updated to query / call the `chain-codec` contract for the necessary transformations and validations instead of implementing them directly. This includes adding the `chain-codec` contract address to their `InstantiateMsg` and config.

An optional `sig_verifier` address will be added to the `multisig-prover`'s `InstantiateMsg` and config, and passed to the `multisig` contract to eliminate forks like in the Aleo integration.

The `domain_separator` (as shown in the `EncoderExt` trait above) will be moved into the `chain-codec` contract, as it is not used anywhere else in the `multisig-prover` contract.

The [one-click deployment (ARC-8)](./ARC-8.md) in the `coordinator` will be updated to deploy the `chain-codec` contract together with the `multisig-prover` and `voting-verifier` contracts, and pass its address to them in their `InstantiateMsg`. It will also have to be updated to reflect the changes to the `multisig-prover` and `voting-verifier` contracts' `InstantiateMsg`s.

### New Crate: `chain-codec-api`

To make implementing `chain-codec` contracts easier, a new `chain-codec-api` crate will be introduced. This crate will define the interface for the `chain-codec` contracts, including the `QueryMsg` and `ExecuteMsg` enums, providing documentation on how to implement them.

It will also feature a `receive-payload` feature flag, analogous to the multisig-prover contract, which enables digest calculation to receive the payload bytes passed into the `ConstructProof` message.

The interface will look like this:

```rust
pub enum QueryMsg {
    /// Encodes the execute data for the target chain that the relayer will submit.
    #[returns(HexBinary)]
    EncodeExecData {
        verifier_set: VerifierSet,
        signers: Vec<SignerWithSig>,
        payload: Payload,
    },
    /// This query must error if the address is malformed for the chain.
    /// Not erroring is interpreted as a successful validation.
    #[returns(Empty)]
    ValidateAddress {
        address: Address,
    }
}

pub enum ExecuteMsg {
    /// This should return a digest for identifying the payload in the `Response::data`. This is what gets signed by the verifiers.
    /// It's called by the multisig-prover contract during proof construction.
    /// You can save additional information to the contract state if needed.
    PayloadDigest {
        verifier_set: VerifierSet,
        payload: Payload,
        /// This field is only available if the multisig-prover contract was compiled with the `receive-payload` feature flag.
        /// Therefore, it is also feature-gated in this crate.
        /// Please note that you should validate this in some way.
        #[cfg(feature = "receive-payload")]
        payload_bytes: HexBinary,
    },
}
```

### Advantages & Disadvantages

#### Advantages

- **Decouples logic**: Shared contracts remain generic and clean.
- **Reduces forking**: Chain-specific differences live in one place.
- **Simplifies integration**: New chains only require a codec contract.
- **Improves maintainability**: Updates are isolated to the respective contracts and don't require updating forks.

#### Disadvantages

- **Increased complexity**: Introducing a new contract adds a bit of complexity internally. It also requires deploying an additional contract per chain.
- **No central point of access**: Address validation is no longer easily accessible in a central place. We will keep the `AddressFormat` enum in the `axelar-wasm-std` crate, but it might not support all chains in the future.

## References

Aleo contracts: https://github.com/eigerco/axelar-amplifier/tree/aleo-its-mr/contracts

Solana contracts: https://github.com/eigerco/axelar-amplifier/tree/solana-cosmwasm/contracts

XRPL contracts: https://github.com/commonprefix/axelar-amplifier/tree/xrpl/contracts

## Changelog

Chronological log of changes to this ARC.

|  Date  | Revision  | Author |  Description  |
|--------|-----------|--------|---------------|
| 2025-07-15 | v1.0 | Christoph Otter | Initial draft |
| 2025-07-23 | v1.1 | Christoph Otter | Change API to support more chains |
| 2025-07-25 | v1.2 | Christoph Otter | Restructure ARC for better readability |
