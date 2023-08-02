---
simd: '0052'
title: Transaction Receipts and Receipt tree for Light clients
authors:
  - Anoushk Kharangate (Tinydancer)
  - Harsh Patel (Tinydancer)
  - Richard Patel (Jump Firedancer)
category: Standard
type: Core
status: Draft
created: 2023-05-30
feature: (fill in with feature tracking issues once accepted)
---

## Summary

This SIMD describes the overall design and changes required that allows users to
verify that a supermajority of the validators has voted on the slot that their
transaction was included in the block without fully trusting the RPC provider.

The main change includes:
  -  Modifying the bankhash to add a Receipt root of the receipt merkle tree that
	  includes transaction signatures and statuses. 

This SIMD is the first step in implementing a consensus verifying client as first
described in [SIMD #10](https://github.com/solana-foundation/solana-improvement-documents/pull/10)
and a majority of the changes mentioned in
the accepted [Simple Payment and State Verification Proposal](https://docs.solana.com/proposals/simple-payment-and-state-verification).

## Motivation

Currently, for a user to validate whether their transaction is valid and included
in a block it needs to trust the confirmation from the RPC. This has been a glaring
attack vector for malicious actors that could lie to users if it's in their own interest.

To combat this, mature entities like exchanges run full nodes that process the
entire ledger and can verify entire blocks. The downside of this is that it's
very costly to run a full node which makes it inaccessible to everyday users,
exposing users to potential attacks from malicious nodes.
This is where diet clients come in, users run the client to verify
the confirmation of their transaction without trusting the RPC.

However, this is only the consensus verifying stage of the client, and with only
these changes, the RPC provider can still trick users, hence we also discuss future
work that will be implemented in futureSIMDs to provide a fully trustless setup.

## Alternatives Considered

None

## New Terminology

Receipt: A structure containing transaction signature and its execution status.
Receipt root: The root hash of a binary merkle tree of Receipts.

## Detailed Design

### Modifying the Bankhash

We propose two new changes:
1) Introduce a new data structure called `Receipt`
```rust
  pub struct Receipt {
    pub signature: [u8; 64],
    pub status: u8,
  }
```
2) Add a transaction receipt root to the bankhash calculation where the receipt
   root is the root of the merkle tree of receipts. This root would be a sha256
   hash constructed as a final result of the binary merkle tree of receipts.
   The receipt root would be added to the bankhash as follows:
``` rust
  let mut hash = hashv(&[
  	self.parent_hash.as_ref(),
  	accounts_delta_hash.0.as_ref(),
  	receipt_root,
  	&signature_count_buf,
  	self.last_blockhash().as_ref(),
  ]);
```
Note: The second change would initially be feature gated with a flag and can 
be activated once we have enough consensus on the activation.

Fig #1 shows an example receipt merkle tree constructed from the corresponding transactions.
<img width="940" alt="Screenshot 2023-08-02 at 11 29 45 PM" src="https://github.com/tinydancer-io/solana-improvement-documents/assets/50767810/61e354f8-64ea-4b3f-a479-23c68741682c">


#### Benchmarks

We have performed benchmarks comparing two merkle tree implementations, 
the benchmark was done on 1 million leaves:
1) Solana labs merkle tree: This implementation uses the BLAKE3 hashing algorithm and is
   implemented purely in rust which can be found in the [Solana labs repository](https://github.com/solana-labs/solana/tree/master/merkle-tree)
2) Firedancer binary merkle tree (bmtree): Implemented in C and uses firedancer's
   SHA-256 implementation as it's hashing algorithm. However the benchmarks were
   performed using its rust FFI bindings. More details: [Firedancer](https://github.com/firedancer-io/firedancer/tree/main/src/ballet/bmtree)
<img width="1010" alt="r1" src="https://github.com/tinydancer-io/solana-improvement-documents/assets/50767810/6c8d0013-1d62-4c7b-8264-4ec71ea28d7c">

More details with an attached flamegraph can be found in our [repository](https://github.com/tinydancer-io/merkle-bench).

## Impact

This proposal will improve the overall security and decentralization of the Solana
network allowing users to access the blockchain in a trust minimized way unlike
traditionally where users had to fully trust their RPC providers. Dapp developers
don't have to make any changes as wallets can easily integrate the client making
it compatible with any dapp. 

## Security Considerations


### Trust Assumptions and Future Work

While this SIMD greatly reduces the user's trust in an RPC, the light client will
 still need to make certain trust assumptions. This includes finding a trusted
 source for the validator set per epoch (including their pubkeys and stake weights)
 and trusting that all transactions are valid (in case the supermajority is corrupt).
 We plan to solve these problems in future SIMDs to provide a full trustless setup
 including data availability sampling and fraud proving which will only require a
 single honest full node.

## Backwards Compatibility *(Optional)*
