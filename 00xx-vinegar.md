# Vinegar

Vinegar is a compatibility layer that allows "importing" target proofs that are not directly pickles-compatible, into pickles. This is achieved by designing two circuits, similar to pickles's Step and Wrap, that act like as a compatibility layer, hiding the mismatches between the target proof system and kimchi.

## Kimchi and Pickles overview

The kimchi verifier receives a proof in the form of mainly commitments, evaluations, and challenges; this comes together with a public input the proof is verified against. The verification procedure can broadly be broken down into the following stages:
1. Absorb commitments to the polynomials and squeeze challenges, over several rounds as needed.
1. Sample an evaluation point for the polynomials, absorb the evaluations, and check that the evaluations satisfy the 'combined constraint polynomial'.
1. Verify that the polynomial commitments and their evaluations  are consistent by checking the 'opening proof' of the polynomial commitment scheme.

*volhovm: TODO elaborate on these three stages*

In order to *recursively* verify a kimchi proof in pickles, we need to be able to efficiently run these steps inside a circuit. However, some of the operations are run on elliptic curve points -- represented as tuples in the *base field* -- and others are run against the *scalar field*, that is the field $F_q$ of scalars for elliptic curve points that form a group. To handle both without an expensive simulation of foreign field arithmetic in the verifier circuit, instead we generate a pair of circuits, where each shares some of the work.

Pickles is our current implementation of a recursive verifier for the kimchi proof system, implemented as a pair of circuits Step and Wrap implemented on top of kimchi.

The Step circuit is a kimchi proof over the vesta curve; the Wrap circuit is a kimchi proof over the pallas curve.

### Verifying a proof in a circuit

Assume that we are verifying a Vesta proof, but the below applies equivalently to Pallas. Here is how the list in the previous section maps on our curves:

1. The logic for absorbing the commitments is run inside a Pallas proof.
1. The 'combined constraint polynomial' is checked inside a Vesta proof.
1. The 'opening proof' is run inside a 'pallas' proof.

Since we have 2 proofs in parallel (one for each curve), we need to *communicate state* between them. We use the public input as the communication mechanism. For example, the Pallas proof will expose the challenges from the first step, as well as the random oracle's state imported into the opening proof.

### Verification structure

Pickles verifies proofs as a DAG, at every stage emitting a 'partially verified proof' and an 'unverified proof'. This structure looks roughly like:
```
step -> wrap -\
               +-> step -> wrap
step -> wrap -/
```
where every step proof may have 0, 1, or 2 previous wrap proofs, and each wrap proof has exactly 1 previous step proof. Only step proofs contain actual "application logic", but a 'pickles proof' is conventionally assumed to be one of the wrap proofs, since we wrap a step proof immediately after producing it.

## Vinegar: goals and circuit structure

Presently, the pickles verifier is hard-coded to use the 'berkeley' configuration. However, as we create new proof systems on top of kimchi (e.g. the MIPS zkVM), we would like to have a way to 'import' those proofs into the pickles recursion system. Also, since the proof systems will not necessarily have a 'public input' feature -- and almost certainly will not have the public input structured as pickles would expect -- we need to take a different approach towards importing them.

Broadly then, the goal of vinegar is to create 2 new 'generic' circuits, which we will call 'vinegar-step' and 'vinegar-wrap'. Let us call the external proof we are consuming "target proof". The structure of two new circuits will then look like:
```
target proof --------------+
    \                       \
     \                       +-> vinegar-wrap
      \                     /
       +--> vinegar-step --+
```

The responsibilities of vinegar-step will be to:
* run the 'scalar field' checks for the kimchi proof
* ingest the 'recursion bulletproof challenges' generated from the kimchi proof
* expose the states before and after the scalar field checks as its public input, for access by vinegar-wrap.

The responsibilities of vinegar-wrap will then be to:
* verify the remainder of the kimchi proof over the 'base field'
* perform initial verification of the 'vinegar-step' proof over the 'base field'
* expose the remaining unverified information from 'vinegar-step' in the public input, so that it can be finalized by a pickles step proof
* expose a public input compatible with the pickles format, so that pickles recursion can operate over the proof.

The operations needed are conceptually very similar to the operations in the existing pickles circuits. However, because we want to avoid hard-coding the particular details of the kimchi proof into the circuit, we will need to be careful to construct 'vinegar-step' and 'vinegar-wrap' in a generic way, with some mechanism to describe the verification steps required for a particular kimchi-style proof.

#### Scratch space; incomplete/draft notes

* In a loop, as many times as required:
  - Absorb (into the random oracle sponge) any polynomial commitments to columns that can be computed using the information available so far.
  - Squeeze (from the random oracle sponge) any challenges required to compute further columns (e.g. challenges for the lookup / permutation arguments)
* Generate a commitment representing the 'constraints' that the proof satisfies:
  - Squeeze a 'constraint combiner' challenge (we call this alpha in kimchi).
  - Compute the

The kimchi verifier runs the following sequence of operations:
* absorb the commitments to the fixed columns (aka the 'verifier index' or 'verifier key') into a random oracle
* absorb the commitments to any 'witness' columns in the proof (including the public input, which the verifier explicitly computes) into the same random oracle
* squeeze one or more challenges for use in the constraints
* absorb the commitments that depend on those commitments

The [Halo2-style IPA](src/lib/crypto/proof-systems/poly-commitment/src/commitment.rs:667) used by the kimchi proof system (and by pickles for recursion) is
