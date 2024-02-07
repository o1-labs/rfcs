# Vinegar, the Compatibility Layer for Pickles

## Summary

Vinegar is a compatibility layer that allows "importing" target proofs that are not directly pickles-compatible, into pickles, which is immensely useful for integrating pickles with other not immediately compatible proof systems.

## Motivation

Presently, the pickles verifier is hard-coded to use the 'berkeley' configuration. However, as we create new proof systems on top of kimchi (e.g. the MIPS zkVM), we would like to have a way to 'import' those proofs into the pickles recursion system. Same may apply to some other kimchi-like plonk-compatible systems that users might want to integrate into mina ecosystem. These proof systems will not necessarily have a 'public input' feature, and almost certainly will not have the public input structured as pickles would expect. This means need to take a different approach towards importing them. Vinegar suggests creating a proof systems that can "absorb" external profs and make them pickles-compatible.

## Background: Kimchi and Pickles overview

The kimchi verifier receives a proof in the form of mainly commitments, evaluations, and challenges; this comes together with a public input the proof is verified against.

Fix a curve $\mathcal{E}$, it has two fields: the base field (for $(x,y) \in \mathcal{E}$, both $x$ and $y$ are base field elements) and the scalar field (for $G$ the generator and group elements $r G \in \mathcal{E}$, $r$ is a scalar field element).

### Proof verification algorithm

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

### Verifying a proof recursively

In order to *recursively* verify a kimchi proof in pickles, we need to be able to efficiently run these steps inside a circuit. However, some of the verification operations are run on base field, and some on scalar. To handle both without an expensive simulation of foreign field arithmetic in the verifier circuit, instead we split verification into a pair of circuits, each sharing the "native" part of the work.

Pickles is our current implementation of a recursive verifier for the kimchi proof system.

Pickles uses a pair of curves called Pallas and Vesta. Their base and scalar fields form a cycle:
- The base field of Pallas is equal to Vesta's scalar field, and Vesta's base field is equal to the scalar field of Pallas.

Pickles is implemented using a pair of circuits Step and Wrap, instantiating two "parallel" variants of kimchi.
- The Step circuit is a kimchi proof over the Vesta curve.
    - Its commitments are Vesta curve points, and its witness values are Vesta scalar field elements.
    - Thus it can "reason natively" about Vesta scalar field = Pallas base field.
- The Wrap circuit is a kimchi proof over the Pallas curve.
    - Can "reason natively" about Pallas scalar field = Vesta base field.

### Verifying a proof in a circuit

Assume that we are verifying a Pallas (wrap) proof, but the below applies equivalently to Vesta (with step and wrap swapped below). Here is how the list in the previous section maps on our curves:

1. The logic for absorbing the commitments is run inside a Vesta proof.
    - The commitments are pairs of Vesta scalar field elements, so we can use native arithmetics.
    - See the first half of `wrap_verifier.ml/incrementally_verify_proof`: in several rounds we absorb `sg_old`, `x_hat`, `w_comm`, lookup data, `z_comm`, `t_comm`; while in parallel squeezing out `index_digest`, `joint_combiner`, `beta`, `gamma`, `alpha`, `zeta`.
1. The 'combined constraint polynomial' is checked inside a Pallas proof.
    - This is deferred to the *next wrap proof*, that is the one that follows Step that verifies the target Wrap proof.
    - This part corresponds to `wrap_verifier.ml/finalize_other_proof`. This checks correctness of computation of `polyscale` and `evalscale`, of `combined_inner_product`, and of the `b` value (combined evaluation of the IPA challenge polynomial).
1. The 'opening proof' is run inside a Vesta proof.
    - This is again `wrap_verifier.ml/incrementally_verify_proof`, but the second half of it, which calls `check_bulletproof`.

Since we have 2 proofs in parallel (one for each curve), we need to *communicate state* between them. We use the public input as the communication mechanism. For example, the Pallas proof will expose the challenges from the first step, as well as the random oracle's state imported into the opening proof.

### Composing pickles circuits

Pickles verifies proofs as a DAG, at every stage emitting a 'partially verified proof' and an 'unverified proof'. This structure looks roughly like:
```
step -> wrap -\
               +-> step -> wrap
step -> wrap -/
```
where every step proof may have 0, 1, or 2 previous wrap proofs, and each wrap proof has exactly 1 previous step proof. Only step proofs contain actual "application logic", but a 'pickles proof' is conventionally assumed to be one of the wrap proofs, since we wrap a step proof immediately after producing it.

## Detailed Design

The goal of vinegar is to create compatibility layer to "absorb" other non-kimchi proofs.

### Vinegar circuits

This is achieved by designing two circuits, similar to pickles's Step and Wrap, that mimic hiding the mismatches between the target proof system and kimchi. The 2 new 'generic' circuits we will call VinegarStep and VinegarWrap. Let us call the external proof we are consuming "target proof" --- we assume that it is a Step-like proof over Vesta. The structure of two new circuits will then look like:

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

### Vinegar architecture and development plan

The implementaiton is suggested to be done in the `proof-systems` repository directly. This will create, de facto, a proving system that is quite similar to pickles (written in `mina` repo in ocaml), which is intenional.

The plan for vinegar is as follows:
1. Move the necessary pickles datatypes (such as `MessagesForNextStep` and so on) into `proof-systems`
1. Write the computation of `deferred_values`.
    - Or find / reassemble from existing codebase -- it already exists, but it is not clear it can be conveniently used. Make sure it can be.
1. Implement the `finalize_other_proof` circuit function, generically over both curves if possible.
   - Write a test. Create a test circuit that verifies just that part.
   - Generating a reasonable sample input for `finalize_other_proof` should help with debugging. Make sure the sample input is modelling the potentially more generic target proof system. This might require a more generic version of `kimchi`, or something that mimics it.
1. Implement the `incrementally_verify_proof` circuit function (also generically).
   - Write a test. Same as for the previous point.
1. Build the `VinegarWrap` circuit by combining the two functions.
   - In practice this requires also making sure that some datatypes are hashed/passed correctly. Expect a lot of small details.
1. Build the `VinegarStep` circuit.
1. Test how two Vinegar circuits work together in a rust test, that is that `VinegarStep` accepts target proof and `VinegarWrap` accepts `VinegarStep`. Use the stub target prove system written before.
1. Test Vinegar with actual pickles and stub target prover.
1. Test Vinegar with actual pickles and actual target prover if the latter is available.

## Test plan and functional requirements

See the development plan before. For the final project the requirements are as follows:
1. Goals and objectives: Vinegar must be able to correctly consume a target proof system in a way that is pickles-compatible and secure.
2. Testing approach:
   - Incremental strategy as explained before: creating a stub prover, testing parts of the circuit separately, then two circuits together within `proof-systems`, then testing Vinegar against actual `mina` pickles implementation.
3. Testing scope: incremental, as explained before.
4. Testing requirements: only correctness testing to start with. The security argument of Vinegar will probably be very similar to the security argument of pickles. The PRs during the development process must be reviewed as usual.
5. Testing resources: stub target proof model, pickles implementation, potentially zkVM implementation if available.

<!--
1. Testing goals and objectives:
    * Specify the overall goals and objectives of testing for the proposed feature or project. This can help set the expectations for testing efforts once the implementation details are finalized.
2. Testing approach:
    * Outline the general approach or strategy for testing that will be followed. This can include mentioning the types of testing to be performed (e.g., unit testing, integration testing, performance testing) and any specific methodologies or tools that will be utilized.
3. Testing scope:
    * Define the scope of testing by identifying the key areas or functionalities that will be covered by testing efforts.
4. Testing requirements:
    * Specify any specific testing requirements that need to be considered, such as compliance requirements, security testing, or specific user scenarios to be tested.
5. Testing resources:
    * Identify the resources required for testing, such as testing environments, test data, or any additional tools or infrastructure needed for effective testing.
-->

## Drawbacks

1. The scope of Vinegar is relatively small in that it does not consider potential future variants of kimchi. Vinegar could be much more generic.
   - Counterargument: Vinegar is already complex enough to start with. Future scope questions may be addressed later.
2. Creating another variant of pickles creates code duplication, which leads to maintenance problems.
   - Counterargument: True, but it simultaneously gives us the ability to (potentially) simplify the mina codebase by reusing some rust functions that will be written in vinegar, or to unify parts of the tooling in rust. NB: none of the mentioned alternatives are factually planned.

## Rationale and alternatives

### Why this design?

Why is this design the best in the space of possible designs? What other designs have been considered and what is the rationale for not choosing them?
- Creating a compatibility circuit for importing proofs seems to be the only practical way to integrate external proofs.
- Implementing the project in `proof-systems` as opposed to `mina` can speed up the development quite a lot since one does not have to


### What is the impact of not doing this?

1. Not being able to import zkVM proofs in mina is a big shortcoming of the mina ecosystem.
2. Not having a way to transition proof systems will be a blocker for adopting new versions of Kimchi. As we plan to generalize current Kimchi and Pickles code to be more generic, we might need to have a compatibility layer between the previous less general version, and the new, more general one.


### What are other advantages of doing this?

1. Having pickles-like code in rust that can potentially be used for generalizing the toolstack / integrating with other tools, etc. It removes the integration with ocaml barrier.
2. Vinegar will be a good platform for developing similar compatibility layers between future (more generic and optimal) versions of Kimchi or similar proving systems.

## Prior Art

## Unresolved questions

1. In pickles, the wrap proof expands the previous step for the next step. This includes computing step's deferred values (xi, combined inner product, b, r). In Vinegar, it seems that co-step needs to work directrly with the target proof. Is this a problem? Will we require foreign field proof expansion? Or will this still be handled by the next Wrap retroactively?
