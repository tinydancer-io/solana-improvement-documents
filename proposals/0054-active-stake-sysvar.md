---
simd: '0054'
title: Sysvar for Active Stake
authors:
  - x19 (Tinydancer)
  - Anoushk Kharangate (Tinydancer)
category: Standard
type: Core
status: Draft
created: 2023-06-07
feature: (fill in with feature tracking issues once accepted)
---

## Summary

We propose to add a new sysvar that contains vote account pubkeys and
their corresponding active stake.

## Motivation

Currently, if a validator wants to prove its active stake to a program, it needs
to pass in all of the stake accounts which have delegated to it. This is
infeasible due to the large number of stake accounts this would require.

Using the proposed sysvar, a program can look up the corresponding
vote account and verify the amount of active stake it has delegated to it.

This sysvar would unlock new use cases which use stake amount in their logic
including on-chain governance, attestations, and more.

## Alternatives Considered

One alternative is to pass stake weights in the first votes of the epoch so that
they can be retrieved by parsing vote signatures. If an on-chain program want to
verify active stake, the validator simply provides them with those signatures and
their pubkey and the program verified the signature and parses the data to extract
the stake weights and slot information.

However this approach has a few drawbacks:

- Vote size limits may not allow larger stake weight mappings
- Someone needs to store the first votes of the epoch else this data can never be
  verified on-chain
- A sysvar program enables much better UX as this data is just always available
  to any on chain program

## New Terminology

None

## Detailed Design

- The sysvar structure would be:
  `Vec<(vote_account: Pubkey, active_stake_in_lamports: u64)>`
- The sysvar address would be: `SysvarStakeWeight11111111111111111111111111`

Stake weight information should be already be available on full node clients
since it's required to construct the leader schedule. Since stake weights can
only be modified on a per-epoch basis, validators will only need to update this
account on epoch boundaries.

We would also need a new feature gate to activate this sysvar.

## Impact

Implementing the proposed sysvar will enable new types of programs which are not
possible now,improving Solana's ecosystem.

## Security Considerations

None

## Backwards Compatibility

Existing programs are not impacted at all.
