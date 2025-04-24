# ARC-7: Service Parameters Override

## Metadata

- **ARC ID**: 7

- **Author(s)**: Edwin Guajardo

- **Status**: Draft

- **Created**: 2025-04-22

- **Last Updated**: 2025-04-22

- **Target Implementation**: Q2 2025

## Summary

This ARC proposes a way to override some service parameters per chain.

## Background and Motivation

Currently, the protocol has been using a single service for all the chains it supports. However, sometimes when a new chain is integrated, it is necessary to gradually onboard the verifiers, which means using a service with a higher verifier set size requirement would block the chain from functioning until enough verifiers are onboarded. This could be solved by using a different service with a lower verifier set size, but this would require verifiers to bond again for the new service and, once the chain is ready to use higher requirements, the Voting Verifier and Multisig Prover would need a migration to switch to the original more restrictive service. Adding the capability to override relevant service parameters per chain would simplify the process of onboarding new chains and would allow the Voting Verifier and Multisig Prover to be used in the same way as the current service.

## Solution Constraints

This solution proposes a set of parameters that can be overridden per chain, while the rest of the parameters must not be overridden in order to maintain the protocol's consistency and the integrity of the bonding mechanism.

The following parameters can be overridden:

- min_num_verifiers
- max_num_verifiers

The following parameters cannot be overridden:

- name
- description
- coordinator_contract
- min_verifier_bond
- bond_denom
- unbonding_period_days

If any of the restricted parameters needs to be changed, it is advised to use a different service instead of overriding the parameters.

## Registering Parameters Override

In order to register a new service parameters override, the proposal is to add `OverrideServiceParams` variant to the ServiceRegistry's `ExecuteMsg`, similar to how the `UpdateService` message is used to update the service parameters.

```rust
#[permission(Governance)]
OverrideServiceParams {
    service_name: String,
    chain: ChainName,
    service_params_override: ServiceParamsOverride,
},
```

```rust
pub struct ServiceParamsOverride {
    pub min_num_verifiers: Option<u16>,
    pub max_num_verifiers: Option<Option<u16>>,
}
```

All `ServiceParamsOverride` fields are optional, and if not provided, the default values will be used.

The overriding parameters will be stored using a `Map` with `(&ServiceName, &ChainName)` as key, and the `ServiceParamsOverride` as value. If the map already contains an entry for the given service and chain, then the new override will overwrite the previous one.

## Applying The Override

To mitigate accidentally ignoring the override parameters, it is proposed to restrict the visibility of the `SERVICES` state constant in the ServiceRegistry, so that it would only be accessible through a function call which receives the service name and the chain name as parameters.

```rust
pub fn service(service_name: &ServiceName, chain: &ChainName) -> Result<Service, ContractError>
```

If there is an override registered for the given service and chain combination, then the corresponding override parameters will overwrite the default values before being returned. Otherwise, the default service parameters will be used.

If there is need to retrieve the default values, the proposal is to add a `default_service_params` function, which will return the default service parameters for the given service name. This way, it will be more explicit that the information is coming from the default values, and not from an override.

## Remove Override

In order to remove a previously registered override, the proposal is to add a `RemoveServiceParamsOverride` variant to the ServiceRegistry's `ExecuteMsg`. Executing this message would handle the removal of the override for the given service and chain.

```rust
#[permission(Governance)]
RemoveServiceParamsOverride {
    service_name: String,
    chain: ChainName,
},
```

## Future Work

1. Evaluate whether we should validate that chain names provided to the ServiceRegistry as input are in fact supported by the protocol. Currently, verifiers can register chain support, and with this ARC, parameter overrides per chain would be possible. However, there's no mechanism to validate if a provided chain is valid without querying the Router or requiring chains to be registered with the ServiceRegistry.
2. Clean up both ServiceRegistry and the Coordinator contracts to ensure consistency on the assumptions about the use of multiple services and overrides.
