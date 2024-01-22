# Vinegar

Vinegar is a compatibility layer that allows "importing" target proofs that are not directly pickles-compatible, into pickles. This is achieved by designing two circuits, similar to pickles's Step and Wrap, that act like as a compatibility layer, hiding the mismatches between the target proof system and kimchi.

## Kimchi and Pickles overview

The kimchi verifier receives a proof in the form of mainly commitments, evaluations, and challenges; this comes together with a public input the proof is verified against. Fix a curve $\mathcal{E}$, it has two fields: the base field (for $(x,y) \in \mathcal{E}$, both $x$ and $y$ are base field elements) and the scalar field (for $G$ the generator and group elements $r G \in \mathcal{E}$, $r$ is a scalar field element).

The verification procedure can broadly be broken down into the following stages:
1. Absorb commitments to the polynomials and squeeze challenges, over several rounds as needed. Sample an evaluation point for the polynomials.
    - In `proof-systems/kimchi`, this corresponds to the first part of `verifier.rs/oracles`: absorbing verifier index, absorbing `prev_chalenges`, absorbing `public_comm` (public input polynomial), absorbing `w_comm`, absorbing some lookup tables, computing `joint_combiner`, absorbing lookup commits, squeezing out `beta`, `gamma`, absorbing lookup polynomial and `z_comm`, squeezing `alpha`, absorbing `t_comm`, and finally squeezing `zeta` and computing `zetaw` (both are evaluation points). Up to this point we are using `fq_sponge`, the sponge that consumes points (i.e. group elements, a pair of base field elements) of the proof's curve, and squeezes elements of this curve's scalar field (as a first step, it squeezes out also base field elements, but we can convert those to scalar field elements).
1. Absorb the evaluations, and check that the evaluations satisfy the 'combined constraint polynomial'.
    - Continuing in `verifier.rs/oracles`, now we instantiate the `fr_sponge` --- a sponge which absorbs and produces elements in the scalar field. First we absorb the previous recursion challenges (`prev_challenge_digest = prev_challenges.chals`, which are scalars over the same group), compute `public_evals` (from `public_input`, also scalar field element), absorb `ft_eval1` together with `public_evals`. Then squeeze out recombination challenges `v`,`u` (also called `polyscale`,`evalscale`). And then compute and return (1) previous recursion challenges evaluation `polys`, (2) combined evaluation `evals`, (3) evaluation `ft_eval0`, (4) the combined constraint polynomial `combined_inner_product`.
    - Now, looking into `verifier.rs/to_batch` -- after `OraclesResult` that corresponds to the previous computation, we compute `f_comm`, and start collecting evaluations, now into `evaluations`, to return them together with previous data in `BatchEvaluationProof`.
    - Note: `prev_challenges` that are the input to `verifier.rs` are coming from 2 itertaions before --- not from the previous proof, but from the one before that. Hence they are over the same group (`prev.challenges.comm` is a commitment, that is group element, and `prev_challenges.chals` are scalar field elements) as the target one for the proof that we are verifying. Hence `prev_challenges.chals` are absorbed in this step.
    - volhovm: what is this "checking that the evaluations satisfy the combined constraint polynomial"? I don't think there's any *checking* or *verification* happening at this point, we merely compute thes polynomial, no?
1. Verify that the polynomial commitments and their evaluations  are consistent by checking the 'opening proof' of the polynomial commitment scheme.
    - This corresponds to `OpeningProof::verify` call in the end of `batch_verify`. Internally, this will combine all the evaluations together, and check that they are consistent with the commitments, jointly.

In order to *recursively* verify a kimchi proof in pickles, we need to be able to efficiently run these steps inside a circuit. However, some of the operations are run on elliptic curve points -- represented as tuples in the *base field* -- and others are run against the *scalar field*, that is the field $F_q$ of scalars for elliptic curve points that form a group. To handle both without an expensive simulation of foreign field arithmetic in the verifier circuit, instead we generate a pair of circuits, where each shares some of the work.

Pickles is our current implementation of a recursive verifier for the kimchi proof system, implemented as a pair of circuits Step and Wrap implemented on top of kimchi.

The Step circuit is a kimchi proof over the Vesta curve (its commitments are Vesta curve points); the Wrap circuit is a kimchi proof over the Pallas curve.

### Verifying a proof in a circuit

Assume that we are verifying a Pallas (wrap) proof, but the below applies equivalently to Vesta. Here is how the list in the previous section maps on our curves:

1. The logic for absorbing the commitments is run inside a Vesta proof.
    - Wrap commitments are Pallas points made of two Pallas base field elements. These are Vesta scalar field elements, which makes them easy to reason about in the Vesta proof -- the next Step proof that verifies a Wrap.
    - See the first half of `wrap_verifier.ml/incrementally_verify_proof`: in several rounds we absorb `sg_old`, `x_hat`, `w_comm`, lookup data, `z_comm`, `t_comm`; while in parallel squeezing out `index_digest`, `joint_combiner`, `beta`, `gamma`, `alpha`, `zeta`.
1. The 'combined constraint polynomial' is checked inside a Pallas proof.
    - This is deferred to the *next wrap proof*, that is the one that follows Step that verifies the target Wrap proof.
    - This part corresponds to `wrap_verifier.ml/finalize_other_proof`. This checks correctness of computation of `polyscale` and `evalscale`, of `combined_inner_product`, and of the `b` value (combined evaluation of the IPA challenge polynomial).
1. The 'opening proof' is run inside a Vesta proof.
    - This is again `wrap_verifier.ml/incrementally_verify_proof`, but the second half of it, which calls `check_bulletproof`.

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

Broadly then, the goal of vinegar is to create 2 new 'generic' circuits, which we will call VinegarStep and VinegarWrap. Let us call the external proof we are consuming "target proof" --- we assume that it is a Step-like proof over Vesta. The structure of two new circuits will then look like:

```
target proof -------------+
    \                      \
     \                      +-> VinegarWrap
      \                    /
       +--> VinegarStep --+
```

The responsibilities of VinegarStep will be to:
* Run the 'scalar field' checks for the target kimchi proof
  - What is currently `step_verifier.ml/finalize_other_proof`
* Ingest the 'recursion bulletproof challenges' generated by the target proof
  - Similar to how `prev_challenges` are currently absorbed in `prover.rs/create_recursive`.
* Expose the states before and after the scalar field checks as its public input, for access by VinegarWrap.
  - For all the `PreviousProofStatement` fields in the Step diagram that refer to the previous Step proofs, they must be filled in with what VinegarStep outputs.
  - Same goes for all the fields of `prev_statement: StepStatement` that VinegarWrap (similarly to Wrap) will operate on.


The responsibilities of VinegarWrap will then be to:
* Verify the remainder of the target proof over the 'base field'
  - What is currently `wrap_verifier/incrementally_verify_proof`.
* Perform initial verification of the 'VinegarStep' proof over the 'base field'
  - Also `wrap_verifier/incrementally_verify_proof`, but now again for the VinegarStep. Our VinegarWrap, unlike Wrap, wraps two proofs at a time.
* Expose the remaining unverified information from 'VinegarStep' in the public input, so that it can be finalized by a pickles step proof
    - During 'VinegarWrap', the VinegarStep's `challenge_poly_commitments` and `old_bulletproof_challenges` will create `MessageForNextWrap` that will be a part of the VinegarWrap statement. Then they will be passed to the Step proof, and from there consumed in `step_verifier/finalize_other_proof`.
* Expose a public input compatible with the pickles format, so that pickles recursion can operate over the proof.

The operations needed are conceptually very similar to the operations in the existing pickles circuits. However, because we want to avoid hard-coding the particular details of the target kimchi proof into the circuit, we will need to be careful to construct 'VinegarStep' and 'VinegarWrap' in a generic way, with some mechanism to describe the verification steps required for a particular target kimchi-style proof.

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
