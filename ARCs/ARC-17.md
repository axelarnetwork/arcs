# ARC-17: Sui Custom Token Linking

## Metadata

- **ARC ID**: 17
- **Author(s)**: Drew Taylor
- **Category**: Interchain Token Protocol
- **Status**: Final
- **Created**: 2025-11-3
- **Last Updated**: 2025-11-3
- **Deployment Status**: Live

## Summary

This ARC specifies the implementation of custom token linking for Sui in the Interchain Token Service (ITS). Custom token linking enables existing Sui coins to be linked with tokens on other chains while supporting different decimal precisions through ITS Hub's automatic scaling. This specification details the Sui-specific patterns, `Channel`-based identity model, `CoinManagement` structure, and key architectural differences from EVM implementations.

## Background and Motivation

ITS supports two distinct token deployment approaches:

**Native Interchain Tokens** are deployed directly by ITS using standardized contracts. They must have identical decimals across all chains, and use a simplified `deploy` → `deploy remote` workflow. No metadata registration is required since decimals are uniform.

**Custom Tokens** are existing tokens deployed independently that integrate with ITS through registration and linking. They support different decimal precisions across chains via ITS Hub's automatic scaling. The workflow requires metadata registration on each chain, followed by token registration and explicit linking. Custom tokens cannot use the `NATIVE_INTERCHAIN_TOKEN` manager type.

The ability to link tokens with different decimals is crucial for integrating existing token ecosystems. For example, USDC has 6 decimals on some chains but 18 on others. When linking such tokens, ITS Hub automatically scales amounts (e.g., by 10^12 when transferring from 6 to 18 decimals), ensuring correct token quantities while preserving each chain's native precision.

### Motivation for Sui-Specific Specification

Sui's implementation differs fundamentally from EVM chains in several areas:

1. **Channel-Based Identity**: Uses `Channel` objects instead of addresses for deployer identity
2. **Type-Based Token Identification**: Sui coins are identified by their full type name (e.g., `package::module::COIN`) rather than by their contract address
3. **CoinManagement Struct**: Replaces token manager contracts
4. **Object-Based Capabilities**: `TreasuryCap` objects replace role-based minter permissions
5. **Chain-Aware Token IDs**: Token ID derivation includes a chain name hash

### Goals

This ARC aims to:

1. Specify Sui's custom token linking implementation within the ITS framework
3. Clarify key differences from EVM token manager contracts
4. Provide guidance for developers integrating Sui tokens with ITS

## Requirements

### Technical Requirements

1. **Channel Identity**: Use `Channel` objects for deployer identity and authentication throughout the linking process
2. **Type Safety**: Utilize Sui's generic type system (`<T>`) for compile-time token type validation
3. **Token ID Derivation**: Include chain name hash in custom token ID calculation: `keccak256(prefix || chain_hash || deployer || salt)`
4. **CoinManagement**: Implement token management logic within `CoinManagement` package supporting both `LOCK_UNLOCK` and `MINT_BURN` patterns
5. **Treasury Cap Management**: Support transfer and reclaim `TreasuryCap` workflow for `MINT_BURN` token managers
6. **Message Format Compatibility**: Use ABI encoding compatible with EVM ITS implementations
7. **Salt Format**: Enforce 32-byte salt format (0x + 64 hex characters) matching Sui address format
8. **Salt Uniqueness**: Prevent reuse of same salt with same deployer `Channel`

### Security Requirements

1. **Channel Ownership Validation**: Verify `Channel` ownership during linking to ensure deployer authentication
2. **Token Manager Type Validation**: Prevent use of `NATIVE_INTERCHAIN_TOKEN` type for custom tokens
3. **Same-Chain Prevention**: Block linking tokens to the source chain itself
4. **Operator Authorization**: Validate `Channel`-based operator permissions for administrative operations
5. **Treasury Cap Reclamation**: Enable original deployer to reclaim `TreasuryCap` if needed using `TreasuryCapReclaimer`
6. **Flow Limit Enforcement**: Support flow limits with operator-only modification rights

## Design

### Process Flow Overview

[TODO: Diagram showing the complete Sui custom token linking flow from registration through linking and transfers]

Sui custom token linking follows this workflow:

```
1. User registers coin metadata on Sui via register_coin_metadata<T>
   → Sends token type name and decimals to ITS Hub
   
2. User registers coin metadata on destination chain
   → ITS Hub stores decimal mappings for both chains

3. User calls register_custom_coin<T> on Sui
   → Creates CoinManagement<T> (token manager)
   → Claims tokenId for deployer Channel + salt
   → Returns TokenId and optional TreasuryCapReclaimer<T>

4. User calls link_coin with destination token address
   → Sends LINK_TOKEN message to ITS Hub
   → ITS Hub calculates scaling factor from stored decimals
   
5. Destination chain receives LINK_TOKEN message
   → Deploys corresponding token manager
   
6. ITS Hub processes InterchainTransfer messages with automatic decimal scaling
```

### Channel-Based Identity Model

Sui uses `Channel` objects for authenticating identities rather than transaction sender addresses. A `Channel` is a Sui object with a unique UID that represents identity (much like an address) across ITS operations:

```move
public struct Channel has key, store {
    id: UID,
}
```

Since `Channel` is an object the can be owned by any user or contract, note that transferring a `Channel` object to another party transfers all of its capabilities and permissions including operatorship, distributorship, etc.

**Usage in Token Linking:**

The same `Channel` must be used for both `register_custom_coin` and `link_coin` calls which proves the coin deployer's identity, since the token ID is derived from the `Channel`'s address.

### CoinManagement: Sui's Token Manager

In EVM, each token gets a separate `TokenManager` contract that handles locking/unlocking or minting/burning. Sui replaces this with the `CoinManagement` package—a generic structure stored within ITS that manages individual token operations.
```move
public struct CoinManagement<phantom T> has store {
    treasury_cap: Option<TreasuryCap<T>>,// For MINT_BURN
    balance: Option<Balance<T>>, // For LOCK_UNLOCK
    distributor: Option<address>, // Minting/burning authority
    operator: Option<address>, // Administrative authority
    flow_limit: FlowLimit, // Rate limiting
    dust: u256, // Rounding dust tracking
}
```

#### Key Architectural Differences

| Aspect | EVM TokenManager | Sui CoinManagement |
|--------|------------------|-------------------|
| **Deployment** | Separate contract per token | Stored in ITS Bag collection |
| **Type Safety** | Runtime via address lookup | Compile-time via generic `<T>` |
| **Authority Model** | Role-based (operator, minter) | Object-based (`TreasuryCap`, `Channel` addresses) |
| **Minting & Burning** | Calls token.mint(), token.burn() with minter role | Directly mints and burns via `TreasuryCap` |
| **Locking** | Transfers tokens to/from contract | Joins/splits Balance<T> |

#### CoinManagement Variants

**LOCK_UNLOCK Type:**
```move
public fun new_locked<T>(): CoinManagement<T> {
    CoinManagement<T> {
        treasury_cap: option::none(),
        balance: option::some(balance::zero()),
        distributor: option::none(),
        operator: option::none(),
        flow_limit: flow_limit::new(),
        dust: 0,
    }
}
```

- Stores `Balance<T>` for holding locked tokens
- No TreasuryCap needed
- Tokens are locked by joining to balance, unlocked by splitting from balance

**MINT_BURN Type:**
```move
public fun new_with_cap<T>(treasury_cap: TreasuryCap<T>): CoinManagement<T> {
    CoinManagement<T> {
        treasury_cap: option::some(treasury_cap),
        balance: option::none(),
        distributor: option::none(),
        operator: option::none(),
        flow_limit: flow_limit::new(),
        dust: 0,
    }
}
```

- Stores `TreasuryCap<T>` for minting/burning
- No balance storage needed
- Tokens are burned by decreasing supply, minted by increasing supply

#### `CoinManagement` Core Operations

**Taking Tokens (Outbound Transfer):**
```move
public(package) fun take_balance<T>(
    self: &mut CoinManagement<T>, 
    to_take: Balance<T>, 
    clock: &Clock
): u64 {
    self.flow_limit.add_flow_out(to_take.value(), clock);
    let amount = to_take.value();
    if (self.has_treasury_cap()) {
        self.burn(to_take);  // MINT_BURN: burn tokens
    } else {
        self.balance.borrow_mut().join(to_take);  // LOCK_UNLOCK: lock tokens
    };
    amount
}
```

**Giving Tokens (Inbound Transfer):**
```move
public(package) fun give_coin<T>(
    self: &mut CoinManagement<T>, 
    amount: u64, 
    clock: &Clock, 
    ctx: &mut TxContext
): Coin<T> {
    self.flow_limit.add_flow_in(amount, clock);
    if (self.has_treasury_cap()) {
        self.mint(amount, ctx)  // MINT_BURN: mint new tokens
    } else {
        coin::take(self.balance.borrow_mut(), amount, ctx)  // LOCK_UNLOCK: unlock tokens
    }
}
```

**Flow Limit Management:**
```move
public(package) fun set_flow_limit<T>(
    self: &mut CoinManagement<T>, 
    channel: &Channel, 
    flow_limit: Option<u64>
) {
    assert!(self.operator.contains(&channel.to_address()), ENotOperator);
    self.set_flow_limit_internal(flow_limit);
}
```

Only the assigned operator `Channel` can modify flow limits, providing security against unauthorized rate limit changes.

### Supported Token Managers

Unlike EVM and other chains, Sui only supports two token manager types for custom tokens:

#### LOCK_UNLOCK (`2`)

Tokens are locked on source chain and unlocked from pre-existing supply on destination chain. Use this manager type for coins that meet the following conditions:
- Existing tokens with established supply distribution
- No minting authority available or desired
- Simple integration without authority transfer

To use this manager type, create a `CoinManagement` using the `new_locked<T>()` function.

#### MINT_BURN (`4`)

Tokens are burned on source chain and minted on destination chain. Use this manager type for coins that meet the following conditions:
- Dynamic supply management desired
- New tokens being deployed cross-chain
- Lock/unlock liquidity insufficient

To use this manager type. create a `CoinManagement` using the `new_with_cap<T>(treasury_cap)` function.

**`MINT_BURN` Requirements:**
- Deployer must provide `TreasuryCap<T>` 
- `TreasuryCap` must be transferred to ITS
- Returns `TreasuryCapReclaimer<T>` for reclaiming the `TreasuryCap`

### Complete Workflow Example

[TODO: Sequence diagram showing: User creates Channel → registers metadata on both chains → registers custom coin → links to destination → optional treasury cap transfer → cross-chain transfer flow]

## References

- [ARC-1](./ARC-1.md)
- [Sui Interchain Token Service](https://github.com/axelarnetwork/axelar-cgp-sui)
- [EVM Interchain Token Service](https://github.com/axelarnetwork/interchain-token-service)
- [Sui Token Linking](#tbd)
- [Sui Channels](#tbd)

## Changelog

|  Date  | Revision  | Author |  Description  |
|--------|-----------|--------|---------------|
| 2025-11-3 | v1.0 | Drew Taylor | Initial ARC specification for Sui custom token linking |
