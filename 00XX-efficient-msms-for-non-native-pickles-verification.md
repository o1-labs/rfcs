# Efficient MSMs in kimchi circuits

## Summary

This RFC describes a protocol for computing elliptic curve MSMs (multi-scalar
multiplications) over 'large' bases.

## Motivation

We would like to support bridging the Mina state to EVM-compatible chains,
creating an efficiently-verifiable proof that wraps Mina's blockchain proofs.
In particular, to achieve this we need to verify the Pasta IPA proofs emitted
by Pickles in a proof over a different backend.

The goal of this work is to complement the existing efforts by Lambdaclass to
verify the proof using Kimchi with the provided Bn254 KZG backend, in such a
way that the MSM part of verification can be offloaded to this protocol, but
their existing code can be reused (mostly) as-is.

Hypotheses:
* This RFC's proposal will be sufficiently performant to support a Mina -> EVM state bridge.
* All candidate implementations without this RFC's proposal -- or an equivalent -- are not sufficiently performant to support a Mina -> EVM state bridge.
* The cryptography and protocol details are sufficiently similar to existing work done by the crypto team that it will be possible to implement this in a reasonable about of time.

This RFC focuses explicitly on minimising the size of the circuit used for this expensive part of the protocol. Happily, this also forces us to use a relatively simple algorithm.

*Product motivation for a Mina -> EVM state bridge deliberately elided.*

## Detailed design

### Background

A verifier for the Mina blockchain is an algorithm comprising:
* validation of consensus rules for the protocol state, and against the previous best / accepted protocol state,
* verification of a pickles proof, whose public input is derived from the protocol state and some additional, pickles-specific data.

The details of the consensus algorithm are elided here, but can be found in the [consensus specification](https://github.com/MinaProtocol/mina/blob/develop/docs/specs/consensus/README.md) in the Mina repo. This RFC will only directly address the proof-system elements of the bridge project.

#### Kimchi proofs

When we verify a kimchi proof, all of the required information is included in either the *verification key* or the *proof*. We describe the contents of the verification key as 'fixed' -- they define the circuit / statement to be proved -- and the contents of the proof represent the specific instance that satisfies that circuit / statement.

At a high-level, the kimchi verification protocol runs through 3 distinct stages of the protocol. The full algorithm in the reference implementation can be found [here](https://github.com/o1-labs/proof-systems/blob/master/kimchi/src/verifier.rs). This algorithm operates over a curve `C` (Pallas or Vesta in the context of pickles).
Absorbing the polynomial commitments:
* setup a 'sponge' for the Fiat-Shamir transformation, using the Poseidon hash function over the base field of `C`;
* absorb the verification key's contents into the sponge;
* absorb any 'recursion polynomial commitments' that accompany the proof into the sponge;
* absorb any witness columns (some of which may be computed by the verifier, e.g. public input);
* as many times as required for the aggregation algorithms:
  - squeeze one or more 'challenges' from the sponge, and
  - absorb the 'aggregation columns' that depend on those challenges;
* squeeze an 'evaluation point' from the sponge;
* take a copy of the sponge state, which we'll call the 'pre-evaluation sponge';
* compute a 'digest' from the sponge.

Absorbing the evaluations:
* initialise a new sponge, using the Poseidon hash function of the scalar field of `C`;
* absorb the 'digest' from the old sponge into the new sponge;
  - (from here, 'sponge' will refer to the 'new sponge')
* absorb any 'recursion challenges' that accompany the proof into the sponge;
  - each recursion challenge corresponds to one of the earlier recursion polynomial commitments
* evaluate the proof-system's constraints at the evaluation point, divided by the 'vanishing polynomial', using the evaluations of each of the columns;
* absorb the evaluations of each of the columns into the sponge;
* squeeze a challenge for combining polynomials, and another for combining evaluations;
* compute the ['combined inner product'](https://github.com/o1-labs/proof-systems/blob/cfc829220b44c1122863eca0db411560b99d6c8e/poly-commitment/src/commitment.rs#L449) of the evaluations.

Verifying the 'opening proof':
* resume using the 'pre-evaluation sponge', which operates over the base field of `C`;
* absorb the combined inner product into the sponge;
* for each of the `log2(domain_size)` rounds, absorb the 'left' and 'right' commitments, and squeeze a challenge for 'folding' the split commitments together;
* compute the evaluation of `b_poly` (formed from the folding challenges) at each evaluation point, combined by powers of the evaluation combiner;
* **compute the commitment to `b_poly` (formed from the folding challenges) using a large MSM**;
  - this is the 'recursion polynomial commitment' that we use in the first stage of the protocol when we're using recursion
* scale and add a small number of other commitments (details elided; [see here](https://github.com/o1-labs/proof-systems/blob/cfc829220b44c1122863eca0db411560b99d6c8e/poly-commitment/src/commitment.rs#L767));
* check that the sum of the other scaled commitments equals the commitment to `b_poly`.

The focus of this RFC will on optimising the '**compute the commitment to `b_poly`**' step, which is the majority of computation required for verifying the proof.

#### Pickles proofs

A pickles proof is formed of a Pallas proof and a partially-verified Vesta proof, which are related by the structure of their respective public inputs.

When we say that the Vesta proof is 'partially-verified', we mean that the Pallas proof has already run stages 1 ('Absorbing the polynomial commitments') and 3 ('Verifying the opening proof') of the verification algorithm, except for computing the recursion polynomial commitment in stage 3. As part of the pickles recursion, we are able to 'check' the recursion polynomial commitment by its evaluations at a random in the *next* round of recursion (i.e. in the next 'opening proof'), so that we only ever have to run 1 full IPA check.

The Pallas proof requires all steps of verification to be run as part of the non-native verifier circuit, but any previous Pallas proofs (i.e. any previous pickles proofs) that we have recursed over will also be amortised into a single Pallas recursion polynomial commitment.

The values shared between the stages are exposed from the public inputs of the pickles Pallas proof, as well as a hash of the 'pickles public input' to be used by the user circuits embedded in the pickles instance. The details of this are elided for brevity.

### High-level description of the algorithm

As inputs, we take a list of 'recursion challenges', which will have length `log2(domain_size)` for each of the 2 groups.

For the Vesta proof, [`log2(domain_size) = 16`](https://github.com/MinaProtocol/mina/blob/8814cea6f2dfbef6fb8b65cbe9ff3694ee81151e/src/lib/crypto/kimchi_backend/pasta/basic/kimchi_pasta_basic.ml#L17), and for the Pallas proof, [`log2(domain_size) = 15`](https://github.com/MinaProtocol/mina/blob/8814cea6f2dfbef6fb8b65cbe9ff3694ee81151e/src/lib/crypto/kimchi_backend/pasta/basic/kimchi_pasta_basic.ml#L16).

For each of these, we have to compute the polynomial `b_poly`, which is defined by
```
b_poly(x) = (1 + chal[-1] x)(1 + chal[-2] x^2)(1 + chal[-3] x^4)...
```
and can be easily computed using a standard circuit of size `domain_size` using the algorithm [here](https://github.com/o1-labs/proof-systems/blob/cfc829220b44c1122863eca0db411560b99d6c8e/poly-commitment/src/commitment.rs#L294).

Once the coefficients of `b_poly` have been determined, we want to provably construct its polynomial commitment by computing the MSM formed by each coefficient and the commitment from the URS that reprepresents the corresponding `x^i`.

In general:

* Be specific. This document is meant to share intent to your colleagues. Share what you believe you will actually do.
* Be decisive. No maybes. Any uncertainty can be captured in the unresolved questions section at the end.
* Provide design contex so that we can align on and commit to a technical design. 

Beyond the design of the change itself, also include details around:

* Security implications
* Performance
* The impact of this change on other components or systems
* Dissect edge cases with examples

**Evergreen, wide-sweeping Details**

Evergreen details are hypothesized to be true for the lifetime of this component or system. They are also hard to pinpoint a location in a spec as they are too widesweeping.

Evergreen details are included directly in the RFC in this section.

**Ephemeral details**

Ephemeral details must live in the spec so that they can evolve over time.

In this section, link to one or more lines of code in committed GitHub code or one or more lines within a PR or PR draft.

When in doubt, "the spec" can be a block comment in source code, but there are a few conventions for our existing systems:

In proof systems-related projects:
* The spec is inline comments that are generated by using [cargo-spec](https://github.com/mimoo/cargo-specification)

For Mina Daemon:

* Large areas are captured by a [separate specs area](https://github.com/MinaProtocol/mina/tree/develop/docs/specs)
* Default to inline comments. Prefer [cargo-spec](https://github.com/mimoo/cargo-specification) format, so we can extract them later.

For SnarkyJS and other zkApps-related projects:

* The spec is inline comments. Prefer [cargo-spec](https://github.com/mimoo/cargo-specification) format, so we can extract them later.

## Test plan and functional requirements

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

## Drawbacks

Why should we *not* do this?

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

This design is optimised to minimise the number of rows, memory accesses, and non-native elliptic curve operations required to compute the polynomial commitment generated from the MSM.

### What other designs have been considered and what is the rationale for not choosing them?

The MSMs can be implemented using the existing primitives exposed to o1js. However, the efficiency of these operations is unacceptable for the current goals.

The large MSMs required for pickles are
* a Vesta MSM of size 2^16, and
* a Pallas MSM of size 2^15.

The most naive way to implement these is to do each of these `2^15+2^16` scalings in o1js. Each of these scalings requires approximately 254 double-and-add elliptic curve operations, of which each takes ~50 rows (guesstimate, actual numbers not immediately available). This means that we require
```
(2^15 + 2^16) * 254 * 50 = 1,248,460,800
```
rows based on this estimate. It isn't practical to prove a witness this large using o1js, since we will massively exceed the 4GB WASM memory limit, and because the proof will take multiple days (in the best case).

There are alternative MSM algorithms that have better performance on native hardware, but those require either a higher number of memory accesses (which become additional lookups, with a corresponding additional expense), or leverage parallelism that isn't available to us in a zk context.

### What is the impact of not doing this?

It is not practical to build a Mina -> EVM state bridge without this or an similar alternative, given the practicalities of creating a proof using a more naive method.

## Prior art

Discuss prior art, both the good and the bad, in relation to this proposal. 

Prior art is any evidence that your feature (invention, change, proposal) is already known. 

Think about the lessons from other blockchain projects or similar updates and provide readers of your RFC with a fuller picture. If there is no prior art, that is fine. Your ideas are interesting whether they are new or adapted from another source.

## Unresolved questions

* What parts of the design do you expect to resolve through the RFC process before this RFC gets merged?
* What parts of the design do you expect to resolve through the implementation of this feature before merge?
* What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?