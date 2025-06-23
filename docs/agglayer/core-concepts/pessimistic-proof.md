# Pessimistic Proof

## Overview

The pessimistic proof provides each chain connected to the Agglayer a way to cryptographically prove that any withdrawal claims made to the Agglayer are backed by deposits made to the unified bridge contract. The pessimistic proof is a novel ZK proof whose program logic is written in Rust and proven using the SP1 zkVM and the Plonky3 proving system.


## Design & Safety

The Agglayer is designed to be flexible enough to support blockchain architectures with different state transition functions, including Proof of Stake–style consensus or ZK rollups that use different proving systems.

The pessimistic proof does not extend security to chains integrated with the Agglayer; rather, each chain connected to the Agglayer is as secure as it would be if it were not integrated with the Agglayer.

The Agglayer uses per-chain pessimistic proofs to ensure a complete view of all token and message transfers occurring over the protocol. This allows chains that may not trust one another to safely interoperate.

## Integration with State Transition Proof

The pessimistic proof works as part of the State Transition Proof system, specifically in the cross-chain verification layer. After the Aggchain Proof verifies both internal chain operations and bridge constraints, the pessimistic proof provides the final verification step to ensure asset balance consistency across the network.

### Verification Flow

1. **State Transition Proof**: Verifies internal chain operations (ECDSA or Validity Proof)
2. **Aggchain Proof**: Verifies bridge constraints and combines internal verification with bridge verification
3. **Pessimistic Proof**: Final verification of asset balance changes and nullifier updates


## Building a Pessimistic Proof

For any cross-chain token transfer to be finalized such that the token may be withdrawn on the underlying L1, a valid pessimistic proof must be generated. Each chain connected to the Agglayer is required to provide some of the inputs necessary for building a valid pessimistic proof.

The pessimistic proof now receives validated bridge events from the Aggchain Proof, ensuring that only verified cross-chain operations are processed.

Note: For more on how the Agglayer settles bridge claims to the underlying L1, see: [Unified Bridge](https://docs.agglayer.dev/agglayer/core-concepts/unified-bridge/)


## Chain State

Each chain has a Local Network State composed of three Sparse Merkle Trees.

* **Local Exit Tree** <br>
    * The Local Exit Tree tracks only the messages and token transfers originating from it. In the unified bridge, Ethereum holds a Global Exit Tree, which represents a tree containing all of the Local Exit Roots of chains integrated with the Agglayer.

* **Local Balance Tree** <br>
    * The Local Balance Tree*tracks the total balance for all tokens of the given chain. The tokens which originate from one chain (as per the token info) are not tracked in the Local Balance Tree because they have the capacity to withdraw unlimited funds as the token supplier.

* **Nullifier Tree** <br>
    * The Nullifier Tree tracks all the imported bridge exits which are claimed. The main purpose of this tree is to prevent double spend through a replay attack by nullifying every claim after proper update of the source chain’s Local Balance Tree.

!!! note  
    The settlement of one pessimistic proof updates solely the Local Network State of the given source chain.  
    In particular: no update is performed on the Local Network State of the destination chains.  
    The destination chain gets updated only upon submission of a verifiable proof by the destination chain, which would potentially include received claims.


The two chain operations update the source chain Local Network State differently:

* **Operation 1: Withdrawals**
    * Append the withdrawal in the Local Exit Tree
    * Debit the token balance in the Local Balance Tree

* **Operation 2: Claims** 
    * Append a unique identifier of the claim (so-called "global index") in the Local Nullifier Tree**
    * Credit the token balance in the Local Balance Tree



## Proving Statement 

The pessimistic proof cryptographically attests to two statements:

1. **No chain withdraws more than what it claimed.**
    * Ensured with the Local Balance Tree accounting.
2. **A chain can claim only what has been received from a settled L1 state.**
    * Ensured by verifying the inclusion proofs against the L1 Info Tree (committing to all bridge transactions) for all claims.

If the computation performed within the pessimistic proof is consistent, a valid proof is generated.

