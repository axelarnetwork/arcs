# ARC-12: Rewards Contract Reconfiguration with off-chain rewards computation

## Metadata

- **ARC ID**: 12
- **Author(s)**: Ayush Tiwari
- **Category**: Amplifier Protocol
- **Status**: Draft
- **Created**: 2025-10-16
- **Last Updated**: 2025-10-16
- **Target Implementation**: Q4 2025

## Summary

This ARC proposes to:
1. Introduce a batched participation API in `rewards` to record multiple verifiers for a single event in one call (fixing the TODO in `voting-verifier`).
2. Decommission on-chain epoch payouts (`rewards_per_epoch`) and switch to a monthly, off-chain payout based on a per-verifier base fee (cost) plus up to 40% profit at full participation, with AXL/USD conversion at payout time.
3. Keep the `rewards` contract as the source of truth for participation only (record + query), while payouts are executed off-chain once per month.

## Background and Motivation

### AXL Price Risk (denomination vs. economics)
- Rewards are denominated in AXL, but costs and targets are in USD.
  - If AXL price rises: validators earn more USD, integrator’s fixed USD converts to fewer tokens, Axelar’s AXL budget buys more USD than intended.
  - If AXL price falls: validators earn less USD than targeted, integrator’s USD converts to more AXL yet may still not meet USD operating costs.

### Current limitations
- The on-chain model pays per epoch with a fixed `rewards_per_epoch` and pushes transfers, creating fan-out and price-misalignment risks.
- `voting-verifier` issues one `RecordParticipation` per consensus participant; batching would reduce gas and latency.

## Proposed Solution Design

### High level
- Remove on-chain `rewards_per_epoch` budgeting and token pushes.
- Keep on-chain participation accounting (events → per-epoch tallies per pool).
- Once per month, an off-chain job aggregates on-chain participation, applies a proportional per-epoch payout against a monthly cap (`base_fee_usd * 1.40` at 100% participation), converts to AXL using price at payout, and pays out.

### Contract: interface changes (backward-compatible)
- Add a new execute to `rewards`:
  - `RecordParticipationBatch { chain_name, event_id, verifiers: Vec<String> }`
    - Behaves like `RecordParticipation`, but accepts a list of verifier addresses.
    - The pool is still derived from `info.sender` to prevent spoofing across pools.
- Keep existing executes for compatibility (`RecordParticipation`, `AddRewards`, etc.), but deprecate payout-related usage in favor of off-chain payouts.

### Contract: responsibilities after change
- Source of truth for participation only:
  - Continue recording events into per-epoch tallies (per `PoolId = (chain_name, contract)`).
  - Provide queries to fetch participation for a given epoch or range.
- No on-chain token distribution or `rewards_per_epoch` logic.

### Off-chain payout (monthly)
1. Snapshot participation data for the month from the `rewards` contract (all pools, all epochs finishing in the month).
2. For each pool and verifier, compute per-epoch participation ratio = participated_e / required_per_epoch and pay proportionally (no curve/scaling).
3. Define a per-verifier base fee in USD that reflects their cost: `base_fee_usd`. Set a profit cap at full participation: `profit_cap = 0.40` (40%).
4. Compute the total USD cap per verifier for the month at 100% participation: `total_usd_cap = base_fee_usd * (1 + profit_cap)`.
5. Convert to AXL using price at payout time: `total_axl_cap = total_usd_cap / axl_price_usd`. Split equally across the epochs in the month: `base_axl_per_epoch = total_axl_cap / E`, where `E` is the number of epochs included in the month.
6. For each epoch `e`, pay `payout_e = base_axl_per_epoch × (participated_e / required_per_epoch)`. Sum across epochs to obtain the final AXL payout per verifier. This proportionally scales both cost recovery and profit based on delivered participation.

#### Payout computation (off-chain)
- Inputs (per pool, per month):
  - Verifier base fee (USD): `base_fee_usd`
  - Profit cap at full participation: `profit_cap = 0.40`
  - AXL/USD price at payout: `axl_price_usd` (spot or TWAP)
  - Number of epochs in month: `E`
  - For each epoch `e`: `participated_e` and `required_per_epoch` (e.g., 7/10)
- Computation:
  - `total_usd_cap = base_fee_usd * (1 + profit_cap)`
  - `total_axl_cap = total_usd_cap / axl_price_usd`
  - `base_axl_per_epoch = total_axl_cap / E`
  - `payout_e = base_axl_per_epoch × (participated_e / required_per_epoch)`
  - `total_axl = Σ_e payout_e`
  - Round at the end (e.g., 6 decimals): `final_axl = floor(total_axl * 1e6) / 1e6`
- One-line formula: `total_axl = Σ_e [ (base_fee_usd * (1 + profit_cap) / axl_price_usd) / E × (participated_e / required_per_epoch) ]`

## Changes to Existing Contracts

### Rewards
- New execute (batch): `RecordParticipationBatch { chain_name, event_id, verifiers }`.
- New query helpers (optional):
  - `MonthlyParticipation { pool_id, month } -> { per_verifier_counts, total_events }`.
  - Or reuse existing epoch participation queries and aggregate off-chain.
- Remove reliance on `rewards_per_epoch` for economics; do not send tokens on-chain.

### Voting-Verifier
- Replace multiple single-recipient `RecordParticipation` messages with one `RecordParticipationBatch` per finished poll.

### Multisig
- No interface change required; may optionally batch if multiple signatures are submitted at once.

## Data and Computation Model

- Participation storage (unchanged):
  - `TALLIES[(PoolId, epoch)] -> { event_count, participation: { verifier -> count }, params_at_epoch }`.
  - `EVENTS[(event_id, PoolId)]` for deduplication.
- Off-chain aggregation:
  - Sum per-verifier votes and total events across the epochs that fall in the monthly window.
  - Compute per-verifier ratio = votes / total_events.
  - Apply proportional per-epoch weighting as defined in “Off-chain payout (monthly)”.
  - Convert USD cap derived from `base_fee_usd` and profit cap to AXL using price at payout.

## Backward Compatibility and Migration

- Keep existing participation APIs; add batch variant.
- Deprecate on-chain payouts and any monitors relying on `rewards_per_epoch` value.
- No state migration needed beyond the new execute and optional queries.

## Advantages & Disadvantages

### Advantages
- USD-anchored outcomes: aligns validator income with base costs plus capped profit, independent of AXL volatility.
- Operational simplicity: one monthly payout run, fewer on-chain fan-out transactions.
- Gas reduction: batched participation recording.

### Disadvantages
- Off-chain dependency for payouts and pricing (oracle/feed management, reconciliation).
- Additional operational runbooks and monitoring for monthly payouts.

## Open Questions
- Price source (oracle, etc), snapshot time, and fallback in case of missing data.
- Exact definition of “verifier expectations” and per-verifier base fee inputs (by chain).
- Whether to expose monthly aggregation via contract queries or leave entirely to off-chain.
- Handling of proxy addresses for payouts off-chain (mirror on-chain proxy semantics?).

## References

- Rewards participation storage and APIs in the `rewards` contract.
- Voting-Verifier poll completion flow and current per-verifier `RecordParticipation` messages.

## Changelog

|  Date       | Revision | Author       | Description   |
|-------------|----------|--------------|---------------|
| 2025-10-16  | v1.0     | Ayush Tiwari | Initial draft |
