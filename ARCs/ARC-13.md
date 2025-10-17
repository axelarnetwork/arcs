# ARC-13: AXL price should be calculated when DistributeRewards is executed

## Metadata

- **ARC ID**: 13
- **Author(s)**: Ayush Tiwari
- **Category**: Amplifier Protocol
- **Status**: Draft
- **Created**: 2025-10-17
- **Last Updated**: 2025-10-17
- **Target Implementation**: Q4 2025

## Summary

`DistributeRewards` function should be updated to take AXL price as an input argument and use it to calculate rewards associated with a given distribution cycle.

## Background and Motivation

Right now we set a `rewards_per_epoch` configuration against every chain added which represents the payout Axelar wants to give to the verifiers in a single epoch. This value is calculated in terms of USD when a chain is added, where key factors behind the calculation can be represented as as:

```
reward_per_epoch (in AXL) = (monthly_infra_cost_per_verifier * (1 + verifier_profit_percent) * number_of_verifiers) / (number_of_pools * number_of_epochs_in_a_month * axl_price)
```

Here, `axl_price` is calculated at the time of chain addition, but it continuously fluctuates, making it difficult for verifiers to calculate profits on top of operational costs.

The proposed approach suggests saving the `rewards_per_epoch` for every chain in terms of USD value, which will make it a constant. Then calculate payout in AXL directly when `DistributeRewards` is executed, using `axl_price_distribution_cycle` at the time of execution. Therefore chain additions will save, rewards per epoch as:

```
reward_per_epoch (in USD) = (monthly_infra_cost_per_verifier * (1 + verifier_profit_percent) * number_of_verifiers) / (number_of_pools * number_of_epochs_in_a_month)
```

And `DistributeRewards` function signature will be updated to:

```
#[permission(Elevated)]
DistributeRewards {
        pool_id: PoolId,
        epoch_count: Option<u64>,
        axl_price_distribution_cycle: Option<u64>,
    }
```

The function implementation can use `axl_price_distribution_cycle` to calculate `reward_per_epoch` in AXL.

## Requirements

- AXL price should be calculated using volume weighted average pricing of AXL during the start and end timestamp of the distribution cycle
- Function permission needs to be elevated since `axl_price_distribution_cycle` is sensitive for the actual payout

## Future Work

- Decentralize `axl_price_distribution_cycle` calculation

## References

- Rewards participation storage and APIs in the `rewards` contract: [https://github.com/axelarnetwork/axelar-amplifier/tree/main/contracts/rewards](https://github.com/axelarnetwork/axelar-amplifier/tree/main/contracts/rewards)
## Changelog

Chronological log of changes to this ARC.

| Date       | Revision | Author       | Description   |
| ---------- | -------- | ------------ | ------------- |
| 2025‑10‑16 | v1.0     | Ayush Tiwari | Initial draft |
