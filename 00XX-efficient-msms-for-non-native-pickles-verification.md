# Efficient MSMs in Kimchi Circuits

## Summary

This RFC describes a protocol for computing elliptic curve MSMs (multi-scalar multiplications) over 'large' bases in an arithmetized zk circuit.

## Motivation

We would like to support bridging the Mina state to EVM-compatible chains,creating an efficiently-verifiable proof that wraps Mina's blockchain proofs. In particular, to achieve this we need to verify the Pasta IPA proofs emitted by Pickles in a proof over a different backend.

The goal of this work is to complement the existing efforts by Lambdaclass to verify the proof using Kimchi with the provided Bn254 KZG backend, in such away that the MSM part of verification can be offloaded to this protocol, but their existing code can be reused (mostly) as-is.

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

All the work needs to be done on the kimchi / `proof-systems` level. In particular, o1js is out of scope, and we do not need to expose the optimised MSM circuit or any parts of it to o1js.


#### Curves

The three curves in question are BN254, Vesta, and Pallas. From the specification of BN254 and docs on [Pasta](https://o1-labs.github.io/proof-systems/specs/pasta.html), the parameters of the curves are as follows:
- BN254:
    - Base field ($\mathbb{F}_{base}$): 21888242871839275222246405745257275088696311157297823662689037894645226208583 (254 bits)
    - Scalar field ($\mathbb{F}_{scalar}$): 21888242871839275222246405745257275088548364400416034343698204186575808495617 (254 bits)
    - Equation: Weierstrass curve -- $y^2 = x^3 + 3$
- Vesta:
    - Base field ($\mathbb{F}_{base}$): 28948022309329048855892746252171976963363056481941647379679742748393362948097 (255 bits)
    - Scalar field ($\mathbb{F}_{scalar}$): 28948022309329048855892746252171976963363056481941560715954676764349967630337 (255 bits)
    - Equation: Weierstrass curve -- $y^2 = x^3 + 5$
- Pallas:
    - Base field ($\mathbb{F}_{base}$): 28948022309329048855892746252171976963363056481941560715954676764349967630337 (255 bits)
      - (Equal to Vesta's scalar)
    - Scalar field ($\mathbb{F}_{scalar}$): 28948022309329048855892746252171976963363056481941647379679742748393362948097 (255 bits)
      - (Equal to Vesta's base)
    - Equation: Weierstrass curve -- $y^2 = x^3 + 5$

The relationship between the fields is:

```math
\mathbb{F}_{scalar}(\mathrm{BN254}) < \mathbb{F}_{base}(\mathrm{BN254}) < \mathbb{F}_{scalar}(\mathrm{Vesta}) < \mathbb{F}_{base}(\mathrm{Vesta})
```


#### Kimchi proofs

We describe how Kimchi (and later Pickles) proof is structured to give the context in which the MSM verification task is located.

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
* compute the evaluation of $h(X)$ (formed from the folding challenges) at each evaluation point, combined by powers of the evaluation combiner;
* **compute the commitment to $h(X)$ (formed from the folding challenges) using a large MSM**;
  - this is the 'recursion polynomial commitment' that we use in the first stage of the protocol when we're using recursion
* scale and add a small number of other commitments (details elided; [see here](https://github.com/o1-labs/proof-systems/blob/cfc829220b44c1122863eca0db411560b99d6c8e/poly-commitment/src/commitment.rs#L767));
* check that the sum of the other scaled commitments equals the commitment to $h(X)$.

The focus of this RFC will on optimising the '**compute the commitment to $h(X)$**' step, which is the majority of computation required for verifying the proof.

#### Pickles proofs

A pickles proof is formed of a Pallas proof and a partially-verified Vesta proof, which are related by the structure of their respective public inputs.

When we say that the Vesta proof is 'partially-verified', we mean that the Pallas proof has already run stages 1 ('Absorbing the polynomial commitments') and 3 ('Verifying the opening proof') of the verification algorithm, except for computing the recursion polynomial commitment in stage 3. As part of the pickles recursion, we are able to 'check' the recursion polynomial commitment by its evaluations at a random point in the *next* round of recursion (i.e. in the next 'opening proof'), so that we only ever have to run 1 full IPA check.

The Pallas proof requires all steps of verification to be run as part of the non-native verifier circuit, but any previous Pallas proofs (i.e. any previous pickles proofs) that we have recursed over will also be amortised into a single Pallas recursion polynomial commitment.

The values shared between the stages are exposed from the public inputs of the pickles Pallas proof, as well as a hash of the 'pickles public input' to be used by the user circuits embedded in the pickles instance. The details of this are elided for brevity.

### High-level description of the algorithm

As inputs for the MSM algorithm, we take a list of coefficients $`\{c_{i} \}`$, which will have length $n$ equal to `domain_size` for each of the 2 groups, and the set of (SRS) bases $`\{G_i\}`$. The goal is to compute $`\sum\limits_{i=1}^n c_i G_i`$ within a kimchi circuit.

For the Vesta proof, [`log2(domain_size) = 16`](https://github.com/MinaProtocol/mina/blob/8814cea6f2dfbef6fb8b65cbe9ff3694ee81151e/src/lib/crypto/kimchi_backend/pasta/basic/kimchi_pasta_basic.ml#L17), and for the Pallas proof, [`log2(domain_size) = 15`](https://github.com/MinaProtocol/mina/blob/8814cea6f2dfbef6fb8b65cbe9ff3694ee81151e/src/lib/crypto/kimchi_backend/pasta/basic/kimchi_pasta_basic.ml#L16). The size of the SRS over BN254 is $2^{15}$, which is so far the largest existing SRS that is available in this context.

To reiterate on the curve choices: assuming we verify an MSM for a Step proof:
- $\mathbb{F}_{scalar}(\mathrm{BN254})$ is the field the circuit has to be expressed in
- $\mathbb{F}_{scalar}(\mathrm{Vesta})$ is the field for the scalar used in the MSM
- $\mathbb{F}_{base}(\mathrm{Vesta})$ is the field for the coordinates of the curve

Recall that the coefficients $`\{c_i\}`$ we perform MSM on are coming from the IPA polynomial commitment. Assuming $`\{\mathsf{chal}\}_{i=1}^{\mathsf{domain_size}}`$ is a (logarithmic) set of IPA challenges, we then to compute the polynomial $h(x)$, which is defined by


```math
h(X) = (1 + \mathsf{chal}_{-1} X) \cdot (1 + \mathsf{chal}_{-2} X^2) \cdot (1 + \mathsf{chal}_{-3} X^3) \ldots
```

and can be easily computed using a standard circuit of size `domain_size` using the algorithm [here](https://github.com/o1-labs/proof-systems/blob/cfc829220b44c1122863eca0db411560b99d6c8e/poly-commitment/src/commitment.rs#L294).

Once the coefficients $c_i$ of $h(X)$ have been determined, we want to provably construct its polynomial commitment by computing the MSM formed by each coefficient and the commitment from the URS that reprepresents the corresponding `x^i`.

Let $k$ be a fixed integer parameter defining a bucket size. Observe that we can decompose our 254-bit scalars into sums of smaller, $k$-bit scalars, in the following way $`c_i = \sum c_{i,j} 2^{j \cdot k}`$. Define $l = 255/k$ as a number of buckets computed as bitlength of Pallas/Vesta field (both are 255 bits) divided by the bucket size $k$.

Then our target computation can be expressed as follows:

```math
\begin{align*}
\sum_{i=1}^n c_i G_i &=
  \underbrace{\sum_{j=1}^{l}
    (\sum_{i=1}^n \overbrace{c_{i,j}}^{\in\ \mathbb{F}_{scalar}(\text{Vesta})}
        (\overbrace{G_i 2^{j \cdot k}}^{\text{Coordinates in $\mathbb{F}_{base}(\mathrm{Vesta})$, computed externally}}
        )}_{\text{Encoded in $\mathbb{F}_{scalar}(\textrm{BN254})$}} \\
&= \sum_{j=1}^{l} (\sum_{i=1}^n c_{i,j} (2^{j \cdot k} \cdot G_i )) \\
&= \sum_j \mathsf{subres}_j
\end{align*}
```

<!-- Note that we have now N * l base elements, not only N -->

Where each $\mathsf{subres}_j$ is the result of the individual inner step --- we will accumulate $`\sum\limits_{j=1}^i \mathsf{subres}_j`$ after each iteration.


The coefficients $c_{i, j}$ will be encoded on $2^k$ bits, with $k$ small
compared to the field size (around 15). The scaled base elements $2^{k j} G_{i}$ can be
pre-computed outside of the circuit. A lookup table will be used to fetch the
corresponding $G_i \cdot 2^{j \cdot k}$. Therefore, the only operations that we
need to encoded is the addition of $`\mathbb{F}_{base}(\mathrm{Vesta})`$
elements in $`\mathbb{F}_{scalar}(\mathrm{BN254})`$. Note that the elements $G_i
\cdot 2^{j \cdot k}$ will have coordinates in Vesta($\mathbb{F}_{base}$).
Therefore, the table will require more than one limbs for each coordinates.


Let us call the inner sum computation $`\sum\limits_{i=1}^n c_{i,j} (G_i \cdot 2^{j \cdot k})`$ the "sub-MSM" --- it is structurally similar to the original MSM, but it uses the smaller decomposed $`\{c_{i,j}\}`$ and a different set of bases.

In the rest of the section we describe the sub-MSM algorithm that efficiently computes the inner sum. The main strength of the sub-MSM algorithm is that due to coefficients being small we can use (non-ZK) RAM lookups on `buckets` which speeds up things quite a bit.

The sub-MSM algorithm is implementing a standard 'bucketing' trick with $2^k$ buckets to avoid doing any doublings or scalings at all, requiring only $3 n$ curve point additions. Assuming the additive notation for the group:

```rust
fn sub_msm(H: Group, to_scale_pairs: Vec<Field,Group>, k: uint) {
    // Initialize the buckets with the blinding factor H
    let mut buckets: [C; 2^k] = [H; 2^k];
    // Zero bucket is unused. In practice it can
    buckets[0] = 0;
    for (coefficient, commitment) in to_scale_pairs {
        buckets[coefficient] += commitment;
    }
    // Instead of being a variable, `total` can be "stored" in buckets[0] for efficiency
    let mut total = -H.scale(2^k * (2^k - 1) / 2);
    let mut right_sum = 0;
    for i in 1..buckets.length() {
        right_sum += buckets[buckets.length() - i];
        total += right_sum;
    }
}
```

For now assuming the blinding factor is $0$, notice that this works because after successive iterations we have:
```
// First iteration
right_sum = buckets[2^k - 1]
total = buckets[2^k - 1]

// Second iteration
right_sum = buckets[2^k - 1] + buckets[2^k - 2]
total = 2*buckets[2^k - 1] + buckets[2^k - 2]
...
// `i`th iteration
right_sum = buckets[2^k - 1] + buckets[2^k - 2] + ... + buckets[2^k - i]
total = i*buckets[2^k - 1] + (i-1)*buckets[2^k - 2] + ... + buckets[2^k - i]
...
// `2^k-1`th iteration
right_sum = buckets[2^k - 1] + ... + buckets[1]
total = (2^k - 1) * buckets[2^k - 1] + (2^k - 2) * buckets[2^k - 2] + ... + buckets[1]
```

The $H$ generator is a standard technique used to avoid dealing with elliptic curve infinity point. The initial value of `total` is there to exactly cancel the `H` factors introduced in the beginning of the sub-MSM algorithm. The correctness still holds. Let `total_0 = - H.scale(2^k * (2^k - 1) / 2)`:

```
// First iteration
right_sum =  buckets[2^k - 1]
total = total_0 + buckets[2^k - 1]

// Second iteration
right_sum = H + buckets[2^k - 1] + buckets[2^k - 2]
total = total_0 + 2*buckets[2^k - 1] + buckets[2^k - 2]
...
// `i`th iteration
right_sum = H + buckets[2^k - 1] + buckets[2^k - 2] + ... + buckets[2^k - i]
total = total_0 + i*buckets[2^k - 1] + (i-1)*buckets[2^k - 2] + ... + buckets[2^k - i]
...
// `2^k-1`th iteration
right_sum = buckets[2^k - 1] + ... + buckets[1]
total = total_0 + (2^k - 1) * buckets[2^k - 1] + (2^k - 2) * buckets[2^k - 2] + ... + buckets[1]
```

Given that each `buckets[i]` contains an `H`, the terms except for `total_0` will contain $`\sum\limits_{i=1}^{2^k-1} i \cdot H`$ of blinding terms, which is exactly the (negated) amount in `total_0`.




### Implementing Foreign Field Gates

Part of the project is implementing FFA (foreign field arithmetics, additions and multiplications) and FFEC (foreign field EC additions) libraries --- practically these will be gates within a kimchi circuit.

Regarding the foreign field additions and multiplications, native Kimchi already supports these:
- https://o1-labs.github.io/proof-systems/kimchi/foreign_field_add.html
- https://o1-labs.github.io/proof-systems/kimchi/foreign_field_mul.html

However, merely reusing these will most likely be quite suboptimal due to the high data passing overhead, which we can avoid by exploiting wider rows. Also, understanding the existing codebase and techniques might take considerable time at first.

Ideally, the approach taken for their implementation should highly depend on the comparitive efficiency within our limitations (additive lookups, wide circuits). **In practice though** we might want to start right away with a naive solution --- by using big integers arithmetics, taking limb size to be 16 bits and range checking these. We can revisit the technique and optimise the circuit aftert the first working prototype is done.


Regarding ECC, o1js also contains an implementation that is part of ECDSA library. It is unclear though whether this can be useful in the kimchi context, as this library is written in o1js/typescript --- probably it can be used as an inspiration.
- https://github.com/o1-labs/o1js/blob/main/src/lib/gadgets/elliptic-curve.ts#L99

Additionally, one will need to decide which coordinates would are better to use. Available options are:
- [Affine](https://www.hyperelliptic.org/EFD/g1p/auto-shortw.html)
- [Jacobian](https://www.hyperelliptic.org/EFD/g1p/auto-shortw-jacobian.html#doubling-dbl-2007-bl)
- [Projective](https://www.hyperelliptic.org/EFD/g1p/auto-shortw-projective.html)

Again, **in practice** affine coordinates will be most likely the most efficient, and also easy to implement, so the suggested path is to start by implementing those, and revisiting the technique much later in the project timeline.


We might find inspiration and decide which encoding is more efficient by looking at existing FF/ECC implementations:
- https://github.com/privacy-scaling-explorations/halo2wrong
- https://github.com/DelphinusLab/halo2ecc-s
- https://hackmd.io/@arielg/B13JoihA8


### Algorithm Performance, Circuit Layout, and Multi-Circuit Folding

The circuit that we are implementing is large, and its layout should be carefully considered. Luckily, we have some liberty in design decisions -- the algorithm needs to be implemented on a *variant* of Kimchi which can allow long rows, additive lookups (memory access), and support for folding. Two last features are (being) developed in the zkVM project.

Note that each iteration (one run) of the sub-MSM algorithm with limited buckets takes $3n$ additions at most:
- The cycle in the beginning populates the buckets and it takes $n$ additions because that's how much multiplications are there in `to_scale_pairs`.
- The end loop does two additions per each iteration (and each iteration assumes a non-zero bucket, assume there are $n'$ of these buckets), so we have $2 \cdot n'$ additions. Since $n' < n$, the worst case is $2 n$ additions.

In addition to $3n$ addition per sub-MSM run, we will need to sum all the results of it. Luckily, this computation is relatively cheap --- we require one extra addition per run.

In total, the whole algorithm then requires $3n \cdot l$ additions because we need to run the inner algorithm $l$ times, and then sum the results (which is negligibly small and can be accumulated along the way). Assuming worst (of the two) case of $n = 2^{16}$ and $k = 15$, we get about $2^{17}$ additions, which is still more than $2^{16}$ budget. But even with the easier $n = 2^{15}$ we still have to fit $3$ times more FF EC additions than there are rows available.


With that in mind, we will use folding to split the total computation into chunks that fit into our $n$-element SRS. Each folding repetition will externally look like an elliptic curve addition w.r.t. the total accumulator. We will elaborate on the design of a concrete circuit later, but so far we can assume that the only shared states between the rounds of folding are (1) total accumulator for the computed value $\sum\limits_{j=1}^i \mathsf{subres}_j$, (2) RAM lookups. Note that the RAM lookups are not ZK and they don't need to be.


Now, there are several approach that can be taken in terms of a concrete circuit design, and deciding on the approach is part of the implementation effort itself since it is (arguably) too complicated for the RFC. The approaches are as follows:
1. Making the circuit wide enough to fit the whole sub-MSM algorithm in one circuit.
1. Splitting the sub-MSM algorithm into sub-algorithms.

Let us elaborate on the two approaches. For simplicity, let us first consider the case of $n=2^{15}$.

Starting with the first one --- it is possible to fit the whole sub-MSM algorithm into one circuit, assuming that each row will contain multiple FF EC additions. For example, given $n = 2^{15}$ and the same number of rows, we will need to fit about $3n$ additions into the circuit. Assuming that we fit $3$ additions per row, the whole sub-MSM product will most likely fit --- the concerns related to the "tightness" of the fit (that is, maybe we will need an extra addition for aggregation, or we will need to have 3 or 4 rows for ZK) we will discuss later separately.
- The advantage of the first approach is that each circuit is exactly the same. Note that it seems to be a matter of convenience and not theoretical limitation --- as far as the current implementation goes, it does not seem problematic though to have different circuits in different iterations of the folding algorithm.
- The disadvantage of this approach is that longer rows mean bigger IVC circuit, plus bigger end verification --- in the very end after the whole folded scheme is proven secure, the resulting proof will still have as much columns as our circuits. So using less columns results in a smaller final proof.

The second approach suggests to instead split the sub-MSM algorithm into several sections to then prove them progressively in the right order. For example, one can imagine one chunk proving just the initial sub-MSM bucket initialisation, the next chunk doing half of the main loop, and another chunk the other half of that loop.
- Certain things are more unclear about this approach:
    - Is it unproblematic that we have to enforce the right order of the chunks? The soundness of the approach relies heavily on the impossibility to swap these chunks around.
    - Lack of circuit space concerns still apply.

Regarding the concerns about potential lack of space in the circuit implementation --- there is reasonable hope that we will be able to fit any of the three sections of the sub-MSM algorithm (each taking $2^15$ rows with one FFEC addition per row) into /exactly/ $2^{15}$ gates.

We can adjust the additive lookup argument to externally (by modifying lookup boundary conditions) assert that the accumulated computation result (the discrepancy) is contained in the last constraint of the last folding iteration. We use the zero bucket for communicating the accumulator between fold iterations.
- Our additive lookup constraint has a form of like $\frac{1}{r + v}$, and we can use an alternative accumulator boundary condition $`\mathsf{acc}' = \mathsf{acc} +
\frac{1}{r + 0} - \frac{1}{r + v_{0}}`$ embedded into the constraint, where $v_0$ is the value contained in the zero slot. In such a way enforce $v_0$ to be present at the zero address. This approach is already taken in the zkVM implementation.
- `total` will go into the zero bucket, and `right_sum` can go into the $2^{k} - 1$ bucket (the last one), and then the second loop in the sub-MSM algorithm can be uniform without any extra single row for aggregation.
- @volhovm @dw: This needs to be explained a bit better, it's important and very technical.


Regarding the algorithm in the harder case of $n = 2^{16}$, the two approaches can be still applied, but need to be modified.
- In the first one, the circuit width needs to grow two times.
- In the second one, the sub-MSM algorithm needs to be split into 6 sections.
- A hybrid approach is possible where width is traded-off with the number of folds -- e.g. we can make the circuit twice as long (each row containing 2 additions) and still have 3 sections of the sub-MSM algorithm.

### Additive Lookups

In the zkVM project there is already an implementation of an additive lookup algorithm.
- We must make sure, when moving it into our project, that it does not rely heavily on pairings -- even though BN254 supports a pairing, it is better (for future compatibility with IPA Kimchi) to ignore this.
- We must make sure that the lookup argument supports lookups of the $k$ bit size, so the parameter must align well with the lookup protocol.

### Folding's IVC

For folding to work correctly "in its full recursive power", additionally to merely instance folding itself, we will need an IVC (interactive verifiable computation) part that will verify the previous folding iteration within the circuit.
- We consider two approaches regarding folding IVC: either supporting a cycle of curves (BN254 / Grumpkin cycle) or non-native emulation.
    - The cycle of curves approach relies on switching between two curves on every folding iteration. Verification of KZG can be encoded with Grumpkin curve (see [this aztec blog post](https://hackmd.io/@aztec-network/ByzgNxBfd#2-Grumpkin---A-curve-on-top-of-BN-254-for-SNARK-efficient-group-operations)).
    - The non-native emulation uses foreign field arithmetics to proceed. (Note that this is /not/ the Goblin Plonk approach)
    - We expect to start the project by implementing the FFA/FFEC circuit first. When it is time to start implementing folding with IVC, if only the curves approach is available, it will be our first choice. However the second approach is strongly preferred in the long run.
- It also needs to be noted that IVC circuit is running "in parallel" --- it is not part of our target circuit, so we can use all the $n = 2^{15}$ available rows (SRS size limit) without worrying about size of the IVC.


<!-- This foreign field emulation is (probably) what Aztec call "goblin plonk" technique. Link: https://hackmd.io/@aztec-network/B19AA8812 (see CF Istanbul talks from Zac). -->

<!--In general:

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

* The spec is inline comments. Prefer [cargo-spec](https://github.com/mimoo/cargo-specification) format, so we can extract them later.-->

## Test plan and functional requirements

<!--1. Testing goals and objectives:
    * Specify the overall goals and objectives of testing for the proposed feature or project. This can help set the expectations for testing efforts once the implementation details are finalized.
2. Testing approach:
    * Outline the general approach or strategy for testing that will be followed. This can include mentioning the types of testing to be performed (e.g., unit testing, integration testing, performance testing) and any specific methodologies or tools that will be utilized.
3. Testing scope:
    * Define the scope of testing by identifying the key areas or functionalities that will be covered by testing efforts.
4. Testing requirements:
    * Specify any specific testing requirements that need to be considered, such as compliance requirements, security testing, or specific user scenarios to be tested.
5. Testing resources:
    * Identify the resources required for testing, such as testing environments, test data, or any additional tools or infrastructure needed for effective testing.-->

Phase 1:
1. Investigate existing approaches to FFA (foreign field arithmetics) and FFEC (foreign field elliptic curves), including ones implemented for the standard Kimchi. Get basic insight.
    - Have a high level look at foreign field addition and multiplication gate in Kimchi
        - Reading doc in the book + the code. Maybe starting without details (half a day), after that jump on the code to implement, and come back after that for a deeper understanding.
    - Examine the avaliable FFEC implementations / algorithms.
      - This includes the existing implementation of ECDSA / FFEC in o1js. See if it can be used as a comparison for our implementation -- e.g. as a baseline, if it's performant enough.
      - But also other implementations (not part of Kimchi/o1js).
    - Implement the addition of points of (non-native) Vesta with (native) BN254(Fp) using the existing FFA library within Kimchi.
      - The ECC addition operation must be implemented from scratch, but it relies on the foreign field additions/multiplications support which already exists in Kimchi.
      - Start with the affine coordinates. If time allows, see if projective or Jacobian can be more performant.
      - Important: we need to verify that the conditions are respected for the scalar field of BN254. Initially, it has been written for the scalar field of Vesta/Pallas.
    - Evaluate the complexity of the implementation
       - Check the number of constraints, proof size, verifier time, etc. Write benchmarks.
1. Decide on the optimal algorithm for (primarily) FFA, and FFEC.
    - Probably just go with the most intuitive and simple bignum implementation using $16$ bits per limb.
    - In terms of FFEC we can probably just go forward with affine coordinates.
1. Assemble the /core/ target proving system (parallel with everything before), without folding.
    - Build a variant (clone) of Kimchi with a higher number of columns, and additive lookups (but no folding).
      - These components are now implemented in optimism project to different degrees --- they have to be all brought (ideally reused, practicall probably copied) to a project folder.
      - A MIPS demo repository (https://github.com/o1-labs/mips-demo) contains a variant that we can start from: a kimchi with additive lookups and wide rows.
    - Make sure the proving system works on some simple examples (at least).
1. Implement POC FFA sub-circuit for the modified target proof system. Test and benchmark.
   - A single wide row implementation according to the (hopefully optimal) algorithm chosen in the previous step.
1. Implement POC FFEC sub-circuit for the modified target proof system. Test and benchmark.
   - This will probably be just standard affine addition. Should be less problematic than the previous FFA step.
1. Implement the MSM algorithm in the circuit suggested above. Test and benchmark.
   - The MSM algorithm for now can be implemented without folding in mind. That is, the circuit can be wider, and maybe it can pass more data through inputs, and verify only parts of the MSM of smaller MSM sizes. The point is to have the algorithm working.
1. First milestone: Releasing a reduced many-proofs version of MSM algorithm.
   - This task suggests implementing our MSM algorithm /without/ folding.
   - Instead, we try to verify "small MSM" where everything fits within one circuit. Then we can release $l$ (or $3l$, e.g. 32 or more) independent proofs that will verify *parts* of the MSM.


Phase 2:
1. Bring folding with IVC into our variant of Kimchi.
   - If available, use FF IVC (preferred), otherwise the BN254/Grumpkin cycle (temporary fallback).
   - Analyze folding and try to use it in a simple circuit with one of the Pasta curve. Must be able to prove and verify a circuit. It is independent of this work.
1. Implement the MSM algorithm with folding and IVC.
   - The MSM circuit from the previous step should be now properly split into sections and folded.
1. Evaluate the system and potentially optimise the algorithms.
   - It is perhaps more wise to not spend too much time at the earlier stages choosing the most optimal approach. This task is for revisit their performance now when the whole proving system is in place. Swapping an FFA or FFEC algorithm for another (if more optimal one is found) should be much easier at this step.


## Drawbacks

1. Cloned Kimchi might diverge with the main variant.
    - Implementing the whole project in a clone of Kimchi can be problematic to unify with the main version if two diverge significantnly.
    - Alternatively, one could modify the original Kimchi to support arbitrary number of columns first, support additive lookups, and implement folding. Or, perhaps, at least any of the three. This leads to a more elegant and unified solution and contributes to the main proof system that the company implements.
    - However this approach becomes a bottleneck for delivering the results, since Kimchi needs to be integrated with Pickles and support backwards compatibility.
    - The future plan is to generalize existing Kimchi so that we can just *use* it in this project. However at the moment customizability of Kimchi is still WIP, so it is more optimal to start with a clone of Kimchi, and unify them later. Same goes for folding and logups (unless it is easy to reuse them instead of cloning).
1. Current plan does not seem to imply releasing sub-algorithms as independent libraries that might be of value. The whole project is quite complex but at the same highly tailored to one task.
   - Again, perhaps time constraints are inevitable, and perhaps the code can be later reused and generalised for public use.
   - The MSM verification task is arguably very important, and even though it's highly tailored to Pickles it might be of big value not just practically, but also as a proof-of-feasibility.

<!--Why should we *not* do this?-->

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

This design is optimised to minimise the number of rows, memory accesses, and non-native elliptic curve operations required to compute the polynomial commitment generated from the MSM.

### What other designs have been considered and what is the rationale for not choosing them?

The MSMs can be implemented using the existing primitives exposed to o1js. However, the efficiency of these operations is unacceptable for the current goals.

The large MSMs required for pickles are
* a Vesta MSM of size $2^{16}$, and
* a Pallas MSM of size $2^{15}$.

The most naive way to implement these is to do each of these $`2^{15}+2^{16}`$ scalings in o1js. Each of these scalings requires approximately 254 double-and-add elliptic curve operations, of which each takes ~50 rows (guesstimate, actual numbers not immediately available). This means that we require
```math
(2^{15} + 2^{16}) \times 254 \times 50 = 1,248,460,800
```
rows based on this estimate. It isn't practical to prove a witness this large using o1js, since we will massively exceed the 4GB WASM memory limit, and because the proof will take multiple days (in the best case).

There are alternative MSM algorithms that have better performance on native hardware, but those require either a higher number of memory accesses (which become additional lookups, with a corresponding additional expense), or leverage parallelism that isn't available to us in a zk context.

### What is the impact of not doing this?

It is not practical to build a Mina -> EVM state bridge without this or an similar alternative, given the practicalities of creating a proof using a more naive method.

## Prior art

We mentioned before the implementations of FFA and FFEC.

**TODO** research if anyone actually already implemented MSM verification inside a SNARK. Scroll? Aztec?

<!--Discuss prior art, both the good and the bad, in relation to this proposal.

Prior art is any evidence that your feature (invention, change, proposal) is already known.

Think about the lessons from other blockchain projects or similar updates and provide readers of your RFC with a fuller picture. If there is no prior art, that is fine. Your ideas are interesting whether they are new or adapted from another source.-->

## Unresolved questions

1. Engineering problem: how exactly does the trick with the lookup tables work?
   - If we go with the circuit layout that fits $2^{15}$ additions into $2^{15}$ rows having exactly one addition per row, we need to make sure the total joint accumulator (between folding iterations) is being updated properly. This requires altering the folding protocol checks.


<!--* What parts of the design do you expect to resolve through the RFC process before this RFC gets merged?
* What parts of the design do you expect to resolve through the implementation of this feature before merge?
* What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?-->
