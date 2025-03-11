# ARC-4: Stellar Governance Contract Design

# Metadata
- ARC ID: 4
- Author(s): Tanvir Deol
- Status: Draft
- Created: 2025-03-10
- Last Updated: 2025-03-11

# Summary
We wish to add a governance contract for Stellar closely based on the [EVM implementation](https://github.com/axelarnetwork/axelar-gmp-sdk-solidity/blob/main/contracts/governance/AxelarServiceGovernance.sol). The purpose of this contract is to handles cross-chain governance actions like creating, cancelling, approving and executing governance proposals. This ARC defines the interface functions and design of the contract.


# Motivation

### Background
There are two relevant governance contracts on the EVM side
- `InterchainGovernance.sol`: Handles cross-chain governance actions. It includes functionality to create, cancel, and execute governance proposals.
- `AxelarServiceGovernance.sol`: Inherits the Interchain Governance contract with added functionality to approve and execute operator proposals.

We wish to port functionality from both contracts into rust Stellar. Below is an overview of what the methods would be and rust/soroban-specific implementation details.

### Goals
- Implement functionality of `InterchainGovernance.sol` and `AxelarServiceGovernance.sol` in Rust for Stellar.
- Enable cross-chain governance actions, including creation, cancellation, approval, and execution of proposals.
- Implement modifiers (onlyOperator, onlyOperatorOrSelf, onlyGovernance, onlySelf) to ensure authorized actions within the contract.
- Implement a TimeLock module to handle proposal scheduling, cancellation, and execution delays.
- Design a comprehensive set of tests for the governance contract.

# Modifiers
### onlyOperator
**Description:** Modifier to check if the caller is the operator

**Implementation Details:**
- We can use the `#[only_operator]` macro on top of functions instead
```
onlyOperator()
```
---
### onlyOperatorOrSelf
**Description:** Modifier to check if the caller is the operator or the contract itself (I assume contract owner)

**Implementation Details:**
- We can use both the `#[only_owner]` and `#[only_operator]` macro on top of functions instead
```
onlyOperatorOrSelf()
```
---
### onlyGoverance
**Description:** Modifier to check if the caller is the governance contract (checks `source_chain` and `source_address`)

**Implementation Details:**
- We will likely have to create a new function for this to validate it's the governance contract.
```
fn only_governance(&self, env: &Env, source_chain: &str, source_address: &str);
```
---
### onlySelf
**Description:** Modifier to check if the caller is the contract itself

**Implementation Details:**
- We can use the `#[only_owner]` macro on top of functions instead
```
onlySelf()
```
---
# Contract Interface

### constructor
**Description:** Initializes the contract

**Implementation Details:**
- assign given params to global variables
- Use rust-equivalent libraries for `keccak256()` hashing -> `soroban_sdk::crypto::keccack256`
```
pub fn constructor(
    env: Env,
    gateway: Address,
    governance_chain: String,
    governance_address: String,
    minimum_time_delay: Uint256,
)
```
---
### is_operator_proposal_approved
**Description:** Returns whether an operator proposal has been approved

**Implementation Details:**
- use rust hashmap `(Bytes32 -> bool)` to track operator proposal approvals called `operatorApprovals`
- use `_getProposalHash()` defined in this contract
- is read-only
```
pub fn is_operator_proposal_approved(
    env: Env,
    target: Address,
    call_data: Bytes,
    native_value: Uint256,
) -> bool
```
---
### execute_operator_proposal
**Description:** Executes an operator proposal

**Implementation Details:**
- use rust/soroban equivalent function for `_call()` -> `soroban_sdk::invoke_contract()`
- create the `OperatorProposalExecuted` event in `event.rs` 
- use rust hashmap `(Bytes32 -> bool)` to track operator proposal approvals called `operatorApprovals`
```
pub fn execute_operator_proposal(
    env: Env,
    target: Address,
    call_data: Bytes,
    native_value: Uint256,
)
```
---
### transfer_operatorship
**Description:** Transfers the operator address to a new address

**Implementation Details:**
- function can only be called by itself or the operator -> use `#[only_owner]` and `#[only_operator]`
- use `transfer_operatorship` from `operatable.rs`, create a function wrapper around it
- create the `OperatorshipTransferred` event in `event.rs` 
```
pub fn transfer_operatorship(
    env: Env,
    new_operator: Address,
)
```
---
### process_command
**Description:** Internal function to process a governance command

**Implementation Details:** 
- Handles the following cases based on `command_type`
    - Proposal creation
    - Proposal cancellation
    - Approving proposal by operator
    - Cancelling proposal by operator
- create the following events in `event.rs` 
    - `ProposalScheduled`
    - `ProposalCancelled`
    - `OperatorProposalApproved`
    - `OperatorProposalCancelled`
- use rust hashmap `(Bytes32 -> bool)` to track operator proposal approvals called `operatorApprovals`
- use rust/soroban equivalent function for `_scheduleTimeLock` -> see `TimeLock` module below
- use rust/soroban equivalent function for `_cancelTimeLock` -> see `TimeLock` module below
```
fn process_command(
    env: &Env,
    command_type: Uint256,
    target: Address,
    call_data: Bytes,
    native_value: Uint256,
    eta: Uint256,
)
```
---
### get_proposal_eta
**Description:** Returns the ETA (estimated time of arrival) of a proposal

**Implementation Details:** 
- is read-only; doesn't call a transaction
- use rust/soroban equivalent function for `_getTimeLockEta` -> see `TimeLock` module below
```
pub fn get_proposal_eta(
    env: Env,
    target: Address,
    call_data: Bytes,
    native_value: Uint256,
) -> Uint256
```
---
### execute_proposal
**Description:** Executes a proposal. The proposal is executed by calling the target contract with calldata. Native value is transferred with the call to the target contract.

**Implementation Details:** 
- use rust/soroban equivalent function for `_call()` -> `soroban_sdk::invoke_contract()`
- use rust/soroban equivalent function for `_finalizeTimeLock` -> see `TimeLock` module below
- create event in `event.rs` called `ProposalExecuted()`
```
pub fn execute_proposal(
    env: Env,
    target: Address,
    call_data: Bytes,
    native_value: Uint256,
)
```
---
### withdraw
**Description:** Withdraws native token from the contract

**Implementation Details:** 
- function can only be called by itself -> use `#[only_owner]`
```
pub fn withdraw(
    env: Env,
    recipient: Address,
    amount: Uint256,
)
```
---
### execute
**Description:** Internal function to execute a proposal action

**Implementation Details:** 
- Use rust-equivalent libraries to use `abi.encode()` to decode payload -> `soroban_sdk::xdr::ToXdr`
- Calls another function ` _processCommand()` with decoded payload params; the function is defined here. 

```
fn execute(
    env: &Env,
    command_id: BytesN<32>,
    source_chain: String,
    source_address: String,
    payload: Bytes,
)
```
---
### get_proposal_hash 
**Description:** Get proposal hash using the target, callData, and nativeValue

**Implementation Details:** 
- Use rust-equivalent libraries for `keccak256()` and `abi.encode()` ->  `soroban_sdk::crypto::keccack256` and `soroban_sdk::xdr::ToXdr`
```
pub fn get_proposal_hash(
    target: Address,
    call_data: Bytes,
    native_value: Uint256,
) -> BytesN<32>
```
---

# TimeLock

We have to implement the TimeLock as a module `timelock.rs` with the following functions.

Overall Implementation Details:
- use Rust contract storage to store TimeLock values (get/set)
---

Description: Returns the timestamp after which the TimeLock may be executed.
```
pub fn get_time_lock(&self, env: &Env, hash: BytesN<32>) -> u64;
```
Description: Schedules a new timelock.
```
pub fn schedule_time_lock(&self, env: &Env, hash: BytesN<32>, eta: u64) -> u64;
```
Description: Cancels an existing timelock by setting its eta to zero.
```
pub fn cancel_time_lock(&self, env: &Env, hash: BytesN<32>);
```
Description: Finalizes an existing timelock and sets its eta back to zero.
```
pub fn finalize_time_lock(&self, env: &Env, hash: BytesN<32>);
```
Description: Returns the timestamp after which the timelock with the given hash can be executed.
```
fn get_time_lock_eta(&self, env: &Env, hash: BytesN<32>) -> u64;
```
Description: Sets a new timestamp for the timelock with the given hash.
```
fn set_time_lock_eta(&self, env: &Env, hash: BytesN<32>, eta: u64);
```
