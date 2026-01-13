# Oblivious Signing With POETs and MuSig2

This README.md describes at a high level the Sigbash V2 platform's policy generation and proof creation flow, from initial signing policy specification to proof approval and transaction signing. 

By leveraging merkle trees, simple graph theory, and integrating zero-knowledge proof gating into a MuSig2 signing ceremony, we are able to achieve perfect privacy from both the signing server and from blockchain analysis across arbitrary predicates, with remarkable performance properties. The system is robust in it's structural simplicity.


## POET Creation and Policy Commitment

**POET (Policy Operand Expression Tree)** policies are created as boolean predicates over atomic conditions like spending limits, fee requirements, address allowlists, time windows, and usage counting constraints. The logical structure of AND, OR, NOT (etc.) operators forms a tree that defines all valid ways a transaction can be approved. After operators and parameters are encoded into a POET, the tree is reduced to a sparse Merkle tree of depth=1, by walking the tree and enumerating all satisfying paths through the policy logic, packing each path's constraints into a compact data structure called a PathLeaf, and inserting these along with global configuration data (numeric parameters, address list roots, nullifier configurations) as leaves in the tree. The root hash of this sparse Merkle tree becomes the policy commitment, a single 32-byte value that cryptographically commits to the entire policy definition and every valid way to satisfy it.

### Example POET Structures

**Diagram 1: AND Tree with Three Conditions**

```
Example POET: AND tree requiring all conditions

           AND
          / | \
         /  |  \
        /   |   \
      C1   C2   C3

Where:
  C1 = "Amount < 1 BTC"
  C2 = "Fee >= 10 sats/vB"
  C3 = "Recipient in allowlist"

All three conditions must be satisfied for the transaction to be approved.
```

**Diagram 2: OR Tree with Three Conditions**

```
Example POET: OR tree

       OR
      / | \
     /  |  \
   C1  C2  C3

Where:
  C1 = "Monday 9am-5pm"
  C2 = "Tuesday 9am-5pm"
  C3 = "Emergency override key"

Transaction succeeds if ANY condition is met.
```

**Diagram 3a: AND Tree Reduction to Merkle Commitment**

An AND tree with multiple conditions produces a single PathLeaf because all conditions must be satisfied together:

```
AND Tree Reduction (All conditions in ONE path)

    AND                     PathLeaf              Sparse Merkle Tree
   / | \                    Enumeration
  /  |  \        →→→        Path1:                      ROOT
 C1 C2 C3                   {C1,C2,C3}        →→→      /    \
                                                      H1    H2
                                                     / \    / \
                                                   P1 [P1] P2 P3

                               ↓
                  PolicyCommitment = 0xabcd1234...ef56 (32 bytes)
```

The AND tree produces a single PathLeaf because all three conditions must be satisfied together. The PathLeaf contains all three constraint checks, representing the only valid way to satisfy this policy.

**Diagram 3b: OR Tree Reduction to Merkle Commitment**

An OR tree with multiple conditions produces separate PathLeaves because each condition can independently satisfy the policy:

```
OR Tree Reduction (Each condition is a separate path)

     OR                      PathLeaf              Sparse Merkle Tree
   / | \                     Enumeration
  /  |  \        →→→         Path1: {C1}                 ROOT
 C1 C2 C3                    Path2: {C2}        →→→     /    \
                             Path3: {C3}               H1    H2
                                                      / \    / \
                                                    P1 P2  P3 [Empty]

                               ↓
                  PolicyCommitment = 0xabcd1234...ef56 (32 bytes)
```

The OR tree produces three separate PathLeaves because any single condition can satisfy the policy independently. Each PathLeaf represents one valid way to satisfy the policy—the client can choose any of the three paths.

**Path Enumeration Transformation Process:**

This transformation process applies to both AND and OR trees:
1. Enumerates all valid paths through the POET logic (1 path for AND, 3 paths for OR in examples above)
2. Packs each path's constraints into a fixed-size PathLeaf structure
3. Constructs a sparse Merkle tree from all PathLeaves
4. Produces a single 32-byte root hash as the policy commitment

## Path Enumeration and PathLeaf Structure

Each satisfying path through the POET becomes a PathLeaf—a fixed-size structure containing all the constraints required for that specific path. A PathLeaf bundles together the numeric limits (spending ranges, fee requirements), time restrictions (weekday and hour windows), address list references (pointers to allowlist/denylist Merkle roots), and nullifier configuration for stateful constraints applicable to that path. When a client wants to prove their transaction satisfies the policy, they select one PathLeaf that matches their transaction and generate a Merkle inclusion proof showing that this specific PathLeaf is part of the committed sparse Merkle tree. This inclusion proof demonstrates to the verifier that the path being claimed is genuinely one of the valid ways defined in the original policy, without requiring the verifier to know what other paths exist or see the full policy structure.

### PathLeaf Structure Details

**Diagram 1: PathLeaf Structure Overview**

```
PathLeaf Structure (Fixed-Size)
┌─────────────────────────────────┐
│ PathID                          │ ← Unique identifier for this path
├─────────────────────────────────┤
│ Numeric Constraints             │ ← Spending limits, fee requirements
├─────────────────────────────────┤
│ Time Constraints                │ ← Weekday/hour windows
├─────────────────────────────────┤
│ Address List References         │ ← Allowlist/denylist Merkle roots
├─────────────────────────────────┤
│ Nullifier Configuration         │ ← Stateful constraint parameters
└─────────────────────────────────┘
```

**Diagram 2: Merkle Inclusion Proof**

Proving a PathLeaf is part of the committed policy:

```
        Policy Root (committed)
            /     \
          H1       H2  ← Sibling hashes (inclusion proof)
         /  \     /  \
       H3   H4  H5  H6
       /\   /\  /\  /\
     P1 P2 P3 [PathLeaf] P5 P6 P7 P8
                  ↑
            Client proves THIS PathLeaf
            is part of committed policy
```

The client provides:
- The PathLeaf data structure
- Sibling hashes (H1, H5, H6) needed to reconstruct the path to the root
- The verifier recomputes the root and confirms it matches the committed policy hash

## Zero-Knowledge Proof Generation

Proving that a transaction satisfies a PathLeaf's constraints requires demonstrating that the transaction inputs match all the path's requirements without revealing the transaction details. The client constructs this proof by first committing to transaction data using Fiat-Shamir accumulators that compress output addresses, input references, and derived fields into single field elements—accumulators that are information-theoretically too small to reveal underlying values but are cryptographically bound to Bitcoin's BIP-341 transaction hash components.

All constraints from the PathLeaf (numeric ranges, address membership, time windows) are expressed as arithmetic circuits that can be proven using existing zero-knowledge proof systems (e.g Groth16/Plonk). Each constraint type (numeric ranges, set membership, time validation) can be implemented as a separate constraint satisfaction circuit, with the resulting proofs combined into the final proof bundle. The prover demonstrates that the committed transaction values satisfy all constraints without revealing the actual values themselves. The proof bundle includes the PathLeaf data, its Merkle inclusion proof, the compressed transaction commitments, and the zero-knowledge proofs—collectively demonstrating policy compliance while maintaining complete transaction privacy.

### Zero-Knowledge Proof Generation Flow

```
Zero-Knowledge Proof Generation Flow:

┌──────────────┐
│ Transaction  │ ──┐
│ (Private)    │   │
└──────────────┘   │
                   ├─→ ┌─────────────────┐
┌──────────────┐   │   │  ZKP Circuits   │    ┌──────────┐
│  PathLeaf    │ ──┤   │ (Groth16/PLONK/ │ ──→│  Proof   │
│ Constraints  │   │   │    STARKs)      │    └──────────┘
└──────────────┘   │   └─────────────────┘          ↓
                   │            ↓                   │
┌──────────────┐   │      Proves:                   │
│ Policy Root  │ ──┘   "Transaction                 │
└──────────────┘        satisfies                   │
                        PathLeaf"                   │
                                                    │
                                             Server verifies
                                             (never sees
                                              transaction)
```

The proof generation process:
1. Client commits to transaction data using cryptographic accumulators
2. For each constraint type, a specialized circuit generates a proof:
   - Numeric range constraints → range proof circuit
   - Address membership → set membership circuit
   - Time window validation → timestamp circuit
3. All individual constraint proofs are combined into a single proof bundle
4. Server verifies the proof without learning transaction details

## Stateful Constraints Integration

For constraints that track state across multiple transactions - e.g. limiting a spending path to five uses per week or enforcing time-based access windows - the system leverages a dual-tree architecture. The static policy tree (described above) exists on the client side and contains immutable definitions of these constraints: maximum use counts, time windows, reset intervals, and scope identifiers. A separate global revocation tree tracks consumed nullifiers across all users to prevent reuse. Each signing operation generates a unique cryptographic commitment binding the constrained value (a counter index or timestamp), the user's policy, the constraint scope, the time epoch, and the transaction hash. This commitment is proven valid via zero-knowledge range proofs and then burned to the global revocation tree. Privacy is preserved because every signing operation burns exactly one commitment regardless of whether stateful constraints are actually enforced, preventing observers from distinguishing real enforcement from dummy operations or correlating uses by the same user.

## Blind Signing and Bitcoin Integration

The final step connects the zero-knowledge proof to an actual Bitcoin signature through a blind signing protocol. After the server verifies the proof bundle—confirming the PathLeaf inclusion, validating the zero-knowledge proof, and checking that all nullifier commitments are fresh — it participates in a MuSig2 signing ceremony using the real BIP-341 sighash of the transaction. Critically, the server never sees the transaction contents: it only validates the cryptographic proof that some transaction satisfies the policy. The client binds the proof to specific transaction inputs using Schnorr proofs-of-knowledge that demonstrate control of the spending key and link that key to the proof bundle, preventing proof replay or substitution attacks. This architecture achieves clean separation of concerns: the client proves predicate satisfaction using zero-knowledge cryptography, the server conditionally signs based on proof validity, and the Bitcoin network validates the final transaction and signature according to consensus rules. The result is privacy-preserving conditional signing where policy enforcement happens without any party learning transaction details beyond what Bitcoin's public ledger reveals after broadcast. Note that the scheme is a) not true blind MuSig2, but rather a blind Schnorr protocol with MuSig2 tooling; specifically, we only maintain blindness in a client-server setting where Alice (the client) does not want to reveal her message to Bob (the signing server) but is also never asked to sign a message from Bob, and where Bob in turn never asks for a message or to see Alice's signature share but must ensure Alice is unable to execute Wagner's birthday attack or a one-more-signature forgery attack and b) the scheme is not claimed to be concurrently secure; parallel signing sessions are not supported, rate limiting backoffs are implemented, and the 'Poelstra trick' of randomly aborting sessions are all employed to keep Alice honest.

More reading on blind MuSig2 and predicate based signing can be found here:

https://gnusha.org/pi/bitcoindev/CAJvkSsc_rKneeVrLkTqXJDKcr+VQNBHVJyXVe=7PkkTZ+SruFQ@mail.gmail.com/
https://www.di.ens.fr/~fuchsbau/blindSchnorr.pdf
https://eprint.iacr.org/2022/1676.pdf
