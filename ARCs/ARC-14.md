# ARC-14: Reward Adjustments via Operator Pool Param Updates

## Metadata

- **ARC ID**: 14
- **Author(s)**: Ayush Tiwari
- **Category**: Amplifier Protocol
- **Status**: Draft
- **Created**: 2025-10-17
- **Last Updated**: 2025-10-30
- **Target Implementation**: Q1 2026

## Summary

Authorize an operator (Elevated permission) to adjust `rewards_per_epoch` on a frequent cadence (e.g., weekly or per epoch) so verifier payouts remain aligned with a USD-equivalent target as market conditions change. This proposal requires a single permission update; reward accounting, participation checks, and permissionless distribution remain unchanged.

## Current System

- Only governance can update pool parameters (including `rewards_per_epoch`).
- `rewards_per_epoch` is set at pool creation and can become misaligned as AXL price fluctuates.
- Governance updates are slow, limiting responsiveness and causing verifier payouts to drift from intended USD value.

**Current code (before):**

```rust
// axelar-amplifier/contracts/rewards/src/msg.rs (permission attribute)
#[permission(Governance)]
UpdatePoolParams { params: Params, pool_id: PoolId },
```

## Proposed System

- Change `UpdatePoolParams` permission to `Elevated` (operator or governance).
- The operator recalculates and updates `rewards_per_epoch` off-chain when price moves materially (weekly or per epoch).
- No changes to participation tracking, epoch processing, or the permissionless `DistributeRewards` flow.

**Code (after):**

```rust
#[permission(Elevated)]
UpdatePoolParams { params: Params, pool_id: PoolId },
```

At the start of each epoch (or on a weekly schedule), the operator submits the new value. Subsequent distributions for that epoch use the latest `rewards_per_epoch`.

## Example Scenario

| Epoch | AXL Price (USD) | rewards_per_epoch |
| ----- | --------------- | ----------------- |
| 1     | $0.14           | =700/0.14 = 5,000 |
| 2     | $0.20           | =700/0.20 = 3,500 |
| 3     | $0.11           | =700/0.11 = 6,364 |

Each payout remains close to $700 USD per epoch.

## Edge Cases & Safeguards

- Operator should use a multi-sig or monitored account
- Optionally add validation to prevent zero/absurd reward values
- Governance can still override if needed

## Changelog

| Date       | Revision | Author       | Description                             |
| ---------- | -------- | ------------ | --------------------------------------- |
| 2025-10-25 | v2.0     | Ayush Tiwari | Operator-controlled `rewards_per_epoch` |
| 2025-10-16 | v1.0     | Ayush Tiwari | Initial draft                           |
