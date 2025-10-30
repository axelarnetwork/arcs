# ARC-15: Implement Threshold + Proportional Reward Distribution

## Metadata

- **ARC ID**: 15
- **Author(s)**: Ayush Tiwari
- **Category**: Amplifier Protocol
- **Status**: Draft
- **Created**: 2025-10-22
- **Last Updated**: 2025-10-30
- **Target Implementation**: Q1 2026

## Summary

The current reward distribution system uses equal distribution among qualified verifiers, where all verifiers meeting the participation threshold receive identical rewards regardless of their actual participation level. This proposal introduces a **Threshold + Proportional** distribution method that maintains a minimum participation threshold while distributing rewards proportionally based on actual participation levels among qualified verifiers.

## Background and Motivation

### Current System Limitations

The existing equal distribution system has several limitations:

1. **No Performance Differentiation**: A verifier with 90% participation receives the same reward as one with 100% participation.
2. **Binary Incentive Structure**: Verifiers only need to meet the minimum threshold, with no incentive for exceptional performance.
3. **Suboptimal Resource Allocation**: Reward pools may not reflect actual network contribution levels.

### Current Threshold Check Implementation

The current threshold validation is implemented in the `verifiers_to_reward()` function:

```rust
// From axelar-amplifier/contracts/rewards/src/state.rs:182-192
fn verifiers_to_reward(&self) -> Vec<Addr> {
    self.participation
        .iter()
        .filter_map(|(verifier, participated)| {
            Threshold::try_from((*participated, self.event_count))
                .ok()
                .filter(|participation| participation >= &self.params.participation_threshold)
                .map(|_| Addr::unchecked(verifier))
        })
        .collect()
}
```

The current equal distribution logic is implemented in the `rewards_by_verifier()` function:

```rust
// From axelar-amplifier/contracts/rewards/src/state.rs:162-180
pub fn rewards_by_verifier(&self) -> HashMap<Addr, Uint128> {
    let verifiers_to_reward = self.verifiers_to_reward();
    let total_rewards: Uint128 = self.params.rewards_per_epoch.into();

    let rewards_per_verifier = total_rewards
        .checked_div(Uint128::from(verifiers_to_reward.len() as u128))
        .unwrap_or_default();

    if rewards_per_verifier.is_zero() {
        return HashMap::new();
    }

    verifiers_to_reward
        .into_iter()
        .map(|verifier| (verifier, rewards_per_verifier))
        .collect()
}
```

### Current Formula

```rust
// Equal distribution among qualified verifiers
rewards_per_verifier = total_rewards_per_epoch ÷ number_of_qualified_verifiers
```

### Proposed Formula

```rust
// Proportional distribution among qualified verifiers
verifier_reward = total_rewards_per_epoch × (verifier_participation ÷ total_qualified_participation)
```

Or in simple mathematical form:

```
reward[v] = total_rewards * (P[v] / Σ P[v])  for v ∈ Q
reward[v] = 0                                for v ∉ Q
```

Where:
- `v` = verifier address
- `P[v]` = participation count for verifier v (number of events they participated in)
- `Q` = set of qualified verifiers (verifiers who have met the participation threshold)
- `Σ P[v]` = total_qualified_participation (sum of participation counts for all qualified verifiers)
- `total_rewards` = total reward pool for the epoch

**Key Definitions:**
- **Qualified verifiers (Q)**: Verifiers whose participation rate meets or exceeds the threshold (e.g., 9/10 events = 90% threshold)
- **Total qualified participation (Σ P[v])**: Sum of all participation counts from verifiers who qualified for rewards
- **Participation count (P[v])**: Number of events a specific verifier participated in during the epoch

## Requirements

### Functional Requirements

#### Threshold + Proportional Logic

- Verifiers must meet the existing participation threshold to qualify for rewards.
- Among qualified verifiers, rewards are distributed proportionally based on participation levels.
- Maintain the existing 2-epoch delay for reward distribution.

#### Validation

- **Input Validation**: All participation counts must be non-negative.
- **Overflow Protection**: Use `checked_mul` and `checked_div` for safe arithmetic.
- **Threshold Validation**: Maintain existing threshold validation logic.
- **Empty Set Guard**: Return an empty map if no verifiers qualify or denominator is zero.

### Example Scenarios

#### Scenario 1: Current Equal Distribution

```rust
// Pool: 1000 AXL tokens, Threshold: 9/10 events (90%)
// Ted: 10/10 events (100%)
// Robin: 9/10 events (90%)
// Barney: 8/10 events (80%)

// Current method: Equal distribution
// Ted: 1000 ÷ 2 = 500 AXL
// Robin: 1000 ÷ 2 = 500 AXL
// Barney: 0 AXL
```

#### Scenario 2: New Threshold + Proportional Distribution

```rust
// Same scenario with new method
// Qualified verifiers (Q): Ted (10), Robin (9) - both meet 90% threshold
// Barney (8) - does NOT qualify (80% < 90% threshold)
// Total qualified participation (Σ P[v]): 10 + 9 = 19

// New method: Proportional distribution among qualified verifiers only
// Ted: 1000 × (10 ÷ 19) = 526.32 AXL
// Robin: 1000 × (9 ÷ 19) = 473.68 AXL  
// Barney: 0 AXL (not in qualified set Q)

// Verification: 526.32 + 473.68 = 1000 AXL ✓
```

## Expected Benefits

1. **Better Incentive Alignment**: Rewards reflect actual network contribution.
2. **Increased Participation**: Incentive for verifiers to participate in more events.
3. **Improved Network Security**: Higher overall participation rates.
4. **Fairer Distribution**: More granular reward differentiation.

## References

- Current rewards contract implementation: [https://github.com/axelarnetwork/axelar-amplifier/tree/main/contracts/rewards](https://github.com/axelarnetwork/axelar-amplifier/tree/main/contracts/rewards)
- Epoch tally and participation tracking: [contracts/rewards/src/state.rs](https://github.com/axelarnetwork/axelar-amplifier/blob/main/contracts/rewards/src/state.rs)

## Changelog

| Date       | Revision | Author       | Description   |
| ---------- | -------- | ------------ | ------------- |
| 2025-10-22 | v1.0     | Ayush Tiwari | Initial draft |
