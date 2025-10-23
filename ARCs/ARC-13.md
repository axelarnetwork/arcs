# ARC-13: AXL price should be calculated when DistributeRewards is executed

## Metadata

- **ARC ID**: 13
- **Author(s)**: Ayush Tiwari
- **Category**: Amplifier Protocol
- **Status**: Draft
- **Created**: 2025-10-17
- **Last Updated**: 2025-10-22
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

### Example Scenarios

#### Scenario 1: AXL Price Volatility Impact Over Time
```rust
// Chain added in January 2025
let chain_addition_date = "2025-01-15";
let axl_price_at_addition = 0.15; // AXL price when chain was added

// Pool configuration (example values)
let monthly_infra_cost_per_verifier = 1000; // USD (assumption)
let verifier_profit_percent = 0.40; // 40% profit on infra cost 
let number_of_verifiers = 30; // (assumption)
let number_of_pools = 2; // (one for voting verifier another for multisig)
let number_of_epochs_in_a_month = 30; // 30 days (calculated, 1 epoch = 1 day)

// Current system: Fixed reward calculation at chain addition
// Total compensation = infra_cost + (infra_cost * profit_percent) = 1000 + (1000 * 0.40) = 1400 USD
let fixed_reward_per_epoch = (1400 * 30) / (2 * 30 * 0.15) = 4666.667 AXL = 700 USD

// Problem: AXL price changes over time
// March 2025: AXL price = 0.25 (67% increase from Jan)
// May 2025: AXL price = 0.10 (33% decrease from Jan)
// July 2025: AXL price = 0.30 (100% increase from Jan)

// Result: Verifiers receive same AXL amount but USD value fluctuates dramatically
// March: 4666.667 AXL = $1166.667 USD (vs intended $700 USD)
// May: 4666.667 AXL = $466.667 USD (vs intended $700 USD)  
// July: 4666.667 AXL = $1400 USD (vs intended $700 USD)
```

#### Scenario 2: New System with USD-Denominated Rewards
```rust
// Same pool configuration
let monthly_infra_cost_per_verifier = 1000; // USD (assumption)
let verifier_profit_percent = 0.40; // 40% profit on infra cost 
let number_of_verifiers = 30; // (assumption)
let number_of_pools = 2; // (one for voting verifier another for multisig)
let number_of_epochs_in_a_month = 30; // 30 days (calculated, 1 epoch = 1 day)

// Current system: Fixed reward calculation at chain addition
// Total compensation = infra_cost + (infra_cost * profit_percent) = 1000 + (1000 * 0.40) = 1400 USD
let fixed_reward_per_epoch = (1400 * 30) / (2 * 30) = 700 USD per epoch

// At distribution time, convert to AXL using current VWAP
// March 2025: AXL VWAP = 0.25, AXL reward = 700 / 0.25 = 2800 AXL
// May 2025: AXL VWAP = 0.10, AXL reward = 700 / 0.10 = 7000 AXL
// July 2025: AXL VWAP = 0.30, AXL reward = 700 / 0.30 = 2333.332 AXL

// Result: Consistent USD value ($700) regardless of AXL price fluctuations. This way verifier can ensure consistent profit irrespective of AXL price fluctuations.
```


## Requirements

- AXL price should be calculated using **Volume Weighted Average Price (VWAP)** during the distribution cycle period.  
    - Using VWAP helps smooth out short-term price volatility and provides a fairer average price by weighting each trade by its volume, ensuring that periods of high trading activity have more influence on the calculated AXL price. This prevents manipulation or outlier trades from disproportionately impacting rewards payouts.
    - VWAP formula: `VWAP = Σ(Typical_Price × Volume) / Σ(Volume)` where `Typical_Price = (High + Low + Close) / 3`
    - Example: Over 30 days with daily data, VWAP = (TP₁×V₁ + TP₂×V₂ + ... + TP₃₀×V₃₀) / (V₁ + V₂ + ... + V₃₀)
- Function permission needs to be elevated since `axl_price_distribution_cycle` is sensitive for the actual payout
- Include outlier protection to filter prices deviating >20% from median

## Future Work

- Decentralize `axl_price_distribution_cycle` calculation

## References

- Rewards participation storage and APIs in the `rewards` contract: [https://github.com/axelarnetwork/axelar-amplifier/tree/main/contracts/rewards](https://github.com/axelarnetwork/axelar-amplifier/tree/main/contracts/rewards)
- VWAP Formula and Implementation: [Volume Weighted Average Price - WallStreetMojo](https://www.wallstreetmojo.com/volume-weighted-average-price/)
- VWAP Trading Strategy Guide: [VWAP Tutorial - QuantInsti](https://blog.quantinsti.com/vwap-strategy/)
## Changelog

Chronological log of changes to this ARC.

| Date       | Revision | Author       | Description   |
| ---------- | -------- | ------------ | ------------- |
| 2025‑10‑16 | v1.0     | Ayush Tiwari | Initial draft |
