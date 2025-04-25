
# ARC-8: Amplifier Coordinator One-Click Deployment

  

## Metadata

  

-  **ARC ID**: 8

-  **Author(s)**: Solomon Davidson

-  **Status**: Draft

-  **Created**: 2025-04-25

-  **Last Updated**: 2025-04-25

-  **Target Implementation**: Q2 2025

  

## Summary

This ARC defines the requirements and design for the one-click deployment of blockchains to the Axelar network using the Coordinator contract. The Coordinator will be able to deploy the Internal Gateway, Voting Verifier and Prover contracts for a particular blockchain from a single transaction.

## Motivation

  
### Background

When adding Amplifier support for a new blockchain on Axelar, there are three smart contracts that must be deployed for that blockchain. They are as follows:

- **Internal Gateway:** When sending a message from a source chain to a destination chain, a relayer will submit the hash of that message to the source chain’s corresponding internal Gateway.
- **Voting Verifier:** Submitting a message hash to an internal Gateway triggers a poll. Verifiers will then vote on whether or not that message succeeded on the source chain. The Voting Verifier contract manages these polls.
- **Prover:** Messages that have been verified will ultimately be routed to the destination chain’s internal Gateway. The Prover starts a process during which the verifiers for the destination chain sign the hash of each message. These signatures can be queried from the Prover, and submitted to the destination chain.

Instantiating these contracts can be tedious, as they must each be provided with the correct interdependent parameters. Consequently, there are deployment scripts that automate this process. This project takes this a step further, and allows for each contract to be correctly instantiated from a single transaction.

## Requirements



## Detailed Design


### References

  
### Changelog
