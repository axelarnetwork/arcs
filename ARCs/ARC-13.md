# ARC-13: AXL price should be calculated when `DistributeRewards` is executed

## Metadata

- **ARC ID**: 13
- **Author(s)**: Ayush Tiwari
- **Category**: Amplifier Protocol
- **Status**: Draft
- **Created**: 2025-10-17
- **Last Updated**: 2025-10-23
- **Target Implementation**: Q4 2025

## Summary

`DistributeRewards` function should be updated to take the **AXL price** as an input argument and use it to calculate the actual AXL payout corresponding to a fixed USD-denominated target reward for a given distribution cycle.  
This ensures that payouts remain stable in USD terms, even as the AXL/USD exchange rate fluctuates.

## Background and Motivation

Currently, every chain stores a configuration parameter called `rewards_per_epoch` which represents the payout Axelar wants to give to the verifiers in a single epoch.  
This value is computed **in terms of USD** but then converted to AXL _at the time of chain addition_ using the then-current AXL price:

```
reward_per_epoch (AXL) = (monthly_infra_cost_per_verifier * (1 + verifier_profit_percent) * number_of_verifiers)
                        / (number_of_pools * number_of_epochs_in_a_month * axl_price)
```

Here, `axl_price` is fixed during chain addition, but in reality, it fluctuates constantly.  
This means that as AXL appreciates or depreciates, verifier earnings (in USD terms) drift away from the intended payout — creating volatility and potential misalignment between network cost and verifier reward.

To solve this, we propose saving `rewards_per_epoch` as a **USD constant** and converting it to AXL _at distribution time_ using the VWAP-based price (`axl_price_distribution_cycle`).

This means chain additions will now save:

```
reward_per_epoch (USD) = (monthly_infra_cost_per_verifier * (1 + verifier_profit_percent) * number_of_verifiers)
                        / (number_of_pools * number_of_epochs_in_a_month)
```

and when distributing rewards, the Rewards contract will fetch or be given the AXL price applicable to the distribution cycle and convert accordingly.

### Updated Function Signature

```rust
#[permission(Elevated)]
DistributeRewards {
    pool_id: PoolId,
    epoch_count: Option<u64>,
    axl_price_distribution_cycle: Option<Decimal>, // VWAP-based AXL price
}
```

The function implementation then computes the AXL reward dynamically using the provided price.

---

## Example Scenarios

### Scenario 1: AXL Price Volatility Impact (Current System)

```rust
// Chain added in January 2025
let chain_addition_date = "2025-01-15";
let axl_price_at_addition = 0.15; // USD/AXL

// Example parameters
let monthly_infra_cost_per_verifier = 1000; // USD
let verifier_profit_percent = 0.40; // 40%
let number_of_verifiers = 30;
let number_of_pools = 2;
let number_of_epochs_in_a_month = 30;

// Total compensation = 1000 + (1000 * 0.40) = 1400 USD
let fixed_reward_per_epoch = (1400 * 30) / (2 * 30 * 0.15) = 4666.667 AXL = $700 USD

// Price movement after chain addition:
AXL Price:
- March 2025 → $0.25
- May 2025 → $0.10
- July 2025 → $0.30

// Impact:
- March: 4666.667 AXL = $1,166.67
- May:   4666.667 AXL = $466.67
- July:  4666.667 AXL = $1,400
```

**Observation:** Verifiers receive the same AXL quantity but experience huge USD swings (±100%).  
This distorts the economic balance intended by the protocol.

---

### Scenario 2: USD-Denominated Rewards (Proposed System)

```rust
// Using same configuration
let monthly_infra_cost_per_verifier = 1000; // USD
let verifier_profit_percent = 0.40;
let number_of_verifiers = 30;
let number_of_pools = 2;
let number_of_epochs_in_a_month = 30;

// Fixed USD reward per epoch
let reward_per_epoch_usd = (1400 * 30) / (2 * 30) = 700 USD

// Conversion at distribution time using VWAP
March 2025: VWAP = 0.25 → 700 / 0.25 = 2800 AXL
May 2025:   VWAP = 0.10 → 700 / 0.10 = 7000 AXL
July 2025:  VWAP = 0.30 → 700 / 0.30 = 2333.33 AXL
```

**Result:**  
Verifiers always earn ~$700 per epoch regardless of AXL’s market price.  
This stabilizes payouts and ensures predictable economics for all participants.

---

## Requirements

- **VWAP-based AXL price:**  
  The price used during distribution must be calculated using **Volume Weighted Average Price (VWAP)** across the distribution cycle.

  - VWAP smooths out short-term volatility and weights price by traded volume, giving a more representative rate.
  - Formula:
    ```
    VWAP = Σ(TPᵢ × Vᵢ) / Σ(Vᵢ)
    where TPᵢ = (Highᵢ + Lowᵢ + Closeᵢ) / 3
    ```
  - Example (30-day window):
    ```
    VWAP = (TP₁×V₁ + TP₂×V₂ + ... + TP₃₀×V₃₀) / (V₁ + V₂ + ... + V₃₀)
    ```

- **Outlier protection:**  
  Samples deviating more than **20% from the median** should be excluded to avoid manipulation.

- **Precision:**  
  `axl_price_distribution_cycle` must be of type `Decimal` (not integer) to capture fractional prices.

- **Permissioning:**  
  Function requires elevated permission since it controls real reward distribution.

---

## Pseudocode

```text
fn distribute(pool_id, epoch_count_opt, axl_price_opt):
    usd_target = load_reward_per_epoch_usd(pool_id)
    price = axl_price_opt.unwrap_or_else(fetch_vwap_with_outlier_filter)
    require(price > 0)

    axl_per_epoch = usd_target / price
    epochs = epoch_count_opt.unwrap_or(default_epoch_count())
    total_axl = axl_per_epoch * epochs

    credit_rewards(pool_id, total_axl)
```

---

## Future Work

- Decentralize `axl_price_distribution_cycle` calculation.
- Introduce quorum-based or oracle-verified price attestations.

---

## References

- Rewards participation storage: [axelar-amplifier/contracts/rewards](https://github.com/axelarnetwork/axelar-amplifier/tree/main/contracts/rewards)
- VWAP concepts: [WallStreetMojo](https://www.wallstreetmojo.com/volume-weighted-average-price/), [QuantInsti VWAP Guide](https://blog.quantinsti.com/vwap-strategy/)

---

## Changelog

| Date       | Revision | Author       | Description   |
| ---------- | -------- | ------------ | ------------- |
| 2025-10-16 | v1.0     | Ayush Tiwari | Initial draft |
