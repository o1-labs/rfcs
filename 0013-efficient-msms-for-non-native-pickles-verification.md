# Efficient MSMs in Kimchi Circuits

## Summary

This RFC describes a protocol for computing elliptic curve MSMs (multi-scalar multiplications) over 'large' bases in an arithmetized zk circuit.

## Motivation

We would like to support bridging the Mina state to EVM-compatible chains, creating an efficiently-verifiable proof that wraps Mina's blockchain proofs. Bridging is necessary for achieving secure cross-chain data transfers, including tokens, leveraging mina's ZK functionality on other blockchains, and a variety of other (deliberately elided) reasons motivating the Mina <-> EVM state bridge. In particular, to achieve this we need to verify the Pasta IPA proofs emitted by Pickles in a proof over a different backend.

The goal of this work is to complement the existing efforts by Lambdaclass to verify the proof using Kimchi with the provided Bn254 KZG backend, in such away that the MSM part of verification can be offloaded to this protocol, but their existing code can be reused (mostly) as-is.

Hypotheses:
* This RFC's proposal will be sufficiently performant to support a Mina -> EVM state bridge.
* All candidate implementations without this RFC's proposal -- or an equivalent -- are not sufficiently performant to support a Mina -> EVM state bridge.
* The cryptography and protocol details are sufficiently similar to existing work done by the crypto team that it will be possible to implement this in a reasonable about of time.

This RFC focuses explicitly on minimising the size of the circuit used for this expensive part of the protocol. Happily, this also forces us to use a relatively simple algorithm.

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
* compute the evaluation of $h(X)$ (formed from the folding challenges; sometimes called $b(X)$) at each evaluation point, combined by powers of the evaluation combiner;
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

### Protocol summary

Our protocol will primarily consist of two parts: computing MSM coefficients from IPA challenges, and the actual MSM algorithm. The latter is much more complex.

For the Vesta proof, [`log2(domain_size) = 16`](https://github.com/MinaProtocol/mina/blob/8814cea6f2dfbef6fb8b65cbe9ff3694ee81151e/src/lib/crypto/kimchi_backend/pasta/basic/kimchi_pasta_basic.ml#L17), and for the Pallas proof, [`log2(domain_size) = 15`](https://github.com/MinaProtocol/mina/blob/8814cea6f2dfbef6fb8b65cbe9ff3694ee81151e/src/lib/crypto/kimchi_backend/pasta/basic/kimchi_pasta_basic.ml#L16). The size of the SRS over BN254 is $N = 2^{15}$, which is so far the largest existing SRS that is available in this context.

Note: for each curve we will need to verify *two MSMs*, since every step consumes two wraps, and thus each proof is carrying a pair of IPA commitments to check.

To reiterate on the curve choices: assuming we verify an MSM for a Step proof:
- $\mathbb{F}_{scalar}(\mathrm{BN254})$ is the field the circuit has to be expressed in
- $\mathbb{F}_{scalar}(\mathrm{Vesta})$ is the field for the scalar used in the MSM
- $\mathbb{F}_{base}(\mathrm{Vesta})$ is the field for the coordinates of the curve

Let $`n = \mathtt{domain_size}`$ (either 15 or 16).
As inputs to the whole procedure we have $`\{\mathsf{chal}\}_{i=1}^{\log(n)}`$ which are a (logarithmic) set of IPA challenges. Our task is to perform MSM using coefficients $`\{c_i\}`$ that are coefficients of $h(X)$, computed in the IPA procedure as recombinations of the IPA challenges as follows. The polynomial $h(x)$ is defined as:

```math
h(X) = (1 + \mathsf{chal}_{-1} X) \cdot (1 + \mathsf{chal}_{-2} X^2) \cdot (1 + \mathsf{chal}_{-3} X^3) \ldots
```

and can be easily computed using a standard circuit of size `domain_size` using the algorithm [here](https://github.com/o1-labs/proof-systems/blob/cfc829220b44c1122863eca0db411560b99d6c8e/poly-commitment/src/commitment.rs#L294).


As inputs for the MSM algorithm, we take a list of coefficients $`\{c_{i}\}_{i=1}^n`$ for each of the 2 groups, and the set of (SRS) bases $`\{G_i\}`$. The goal is to compute $`\sum\limits_{i=1}^n c_i G_i`$ within a kimchi circuit.


Once the coefficients $c_i$ of $h(X)$ have been determined, we want to provably construct its polynomial commitment by computing the MSM formed by each coefficient and the commitment from the URS that reprepresents the corresponding `x^i`.

### Unwrapping IPA challenges

As input to this part of the protocol we take $`\{\mathsf{chal}\}_{i=1}^{\log(n)}`$  in the 88-bit limb size representation. Assuming $n = 15$ each row of the circuit will do the following:
1. Expect all $`\{\mathsf{chal}\}_{i=1}^{\log(n)}`$ as separate rows. We need them all in one row since we don't have permutation constraints.
1. Convert all of them to $15$-bit representation that will be used for our FFA circuit.
1. In row number $i$, compute $`c_i = \prod\limits_{j=0}^{n-1} \mathsf{chal}_j^{j_i}`$ where $`j_i`$ denotes the $i$th bit of $j$.
   - In other words, we compute all possible products between challenges, one product per row.

For the $n = 16$ case we need two circuits to do this.

### Description of the MSM algorithm

Let $k = 15$ be a fixed integer parameter defining a bucket size. Observe that we can decompose our 254-bit scalars into sums of smaller, $k$-bit scalars, in the following way $`c_i = \sum c_{i,j} 2^{j \cdot k}`$. Define $l = 255/k$ as a number of buckets computed as bitlength of Pallas/Vesta field (both are 255 bits) divided by the bucket size $k$.

Then our target computation can be expressed as follows:

```math
\begin{align*}
\sum_{i=1}^n c_i G_i &=
  \underbrace{\sum_{j=1}^{l}
    \Big(\sum_{i=1}^n \overbrace{c_{i,j}}^{\in\ \mathbb{F}_{scalar}(\text{Vesta})}
        (\overbrace{G_i 2^{j \cdot k}}^{\text{Coordinates in $\mathbb{F}_{base}(\mathrm{Vesta})$, computed externally}}
        )\Big)}_{\text{Encoded in $\mathbb{F}_{scalar}(\textrm{BN254})$}} \\
&= \sum_{j=1}^{l} \Big(\sum_{i=1}^n \underbrace{c_{i,j}}_{\text{Coefficients for limb $j$}} \cdot \overbrace{(2^{j \cdot k} \cdot G_i )}^{\text{"Bases" for limb $j$}}\Big) \\
\end{align*}
```

<!-- Note that we have now N * l base elements, not only N -->

The coefficients $c_{i, j}$ will be encoded on $2^k$ bits, with $k$ small compared to the field size (15 bit). The scaled base elements $2^{j \cdot k} \cdot G_{i}$ can be pre-computed outside of the circuit.

In the rest of the section we describe the MSM algorithm that efficiently computes the inner sum. The main strength of the MSM algorithm is that due to coefficients being small we can use RAM lookups on `buckets` which speeds up things quite a bit. The MSM algorithm is implementing a standard 'bucketing' trick with $2^k$ buckets to avoid doing any doublings or scalings at all, requiring only $l \cdot n + 2^{k+1}$ curve point additions.

Note how in the formula above we can iterate over all the possible coefficients $c_{i,j}$ without knowing the indices in advance -- a coefficient $c$ and limb number $j$ defines positions $i$ in which the index is used. For a given $c$, let $\hat G_{c,j}$ be the value equal to $\sum_i 2^{j \cdot k} \cdot G_i$ for all $c_{i,j}$ (coefficient number $i$ for limb number $j$) equal to $c$. Then the previous decomposition equation can be expressed as:

```math
\begin{align*}
\sum_{i=1}^n c_i G_i &= \sum_{j=1}^{l} \Big(\sum_{c \in [0,2^k-1]} c \cdot \hat G_{c,j}\Big) \\
&= \sum_{c \in [0,2^k-1]} c \cdot (\sum_{j=1}^{l} \hat G_{c,j})
\end{align*}
```

This suggests that we can first accumulate the values of $`\hat G_{c,j}`$ into the correct "buckets", where each bucket corresponds to a coefficient $c_i \in [0,2^k-1]$, and then sum all of them scaling by $c_{i}$.

A lookup table will be used to fetch the corresponding $G_i \cdot 2^{j \cdot k}$. Therefore, the only operations that we need to encoded is the addition of $`\mathbb{F}_{base}(\mathrm{Vesta})`$ elements in $`\mathbb{F}_{scalar}(\mathrm{BN254})`$. Note that the elements $G_i \cdot 2^{j \cdot k}$ will have coordinates in $`\mathbb{F}_{base}(\mathrm{Vesta})`$. Therefore, the table will require more than one limbs for each coordinates.

Assuming the additive notation for the group, the formal code description of the algorithm is as follows:

```rust
fn fill_buckets(buckets: [C; 2^k], coeffs: Vec<Field>, bases: Vec<Field>) {
    for (coefficient, commitment) in coeffs.iter().zip(bases.iter()) {
        buckets[coefficient] += commitment;
    }
}

/// Computes the total MSM: \sum_{i=1}^N coeffs_i \cdot bases_i
fn compute_msm(coeffs: Vec<Field>, bases: Vec<Group>, k: uint, H: Group) {
    // Initialize the buckets with the blinding term H
    let mut buckets: [C; 2^k] = [H; 2^k];
    let l = ceil(254/k); // number of limbs
    for i in 0..l {
        // Temporary k-bit coeffs for limb #i
        let coeffs_curlimb = coeffs[i..i+k];
        // Temporary bases for limb #i: can be pre-computed outside of this algorithm
        let bases_curlimb = (0..N).map(|j| 2^{k i} * bases[j]);
        fill_buckets(buckets, coeffs_curlimb, bases_curlimb);
    }
    let mut total = -H.scale(2^k * (2^k - 1) / 2);
    let mut right_sum = 0;
    for i in 1..buckets.length() {
        right_sum += buckets[buckets.length() - i];
        total += right_sum;
    }
}
```

Let us explain the correctness of the protocol. The loop in the end does a double partial-sum accumulation, which computes the value $`\sum\limits_{c \in [0,2^k-1]} c \cdot (\sum\limits_{j=1}^{l} \hat G_{c,j})`$. The $H$ generator is a standard technique used to avoid dealing with elliptic curve infinity point. The initial value of `total` is there to exactly cancel the factors of the blinding element `H` that are introduced in the beginning of the MSM algorithm. Let `total_0 = - H.scale(2^k * (2^k - 1) / 2)`, then:

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

First, note that we correctly aggregate $2^i$ coefficients along the way. And given that each `buckets[i]` contains an `H`, the terms except for `total_0` will contain $`\sum\limits_{i=1}^{2^k-1} i \cdot H`$ of blinding terms, which is exactly the (negated) amount in `total_0`.



### Implementing Foreign Field Gates

Part of the project is implementing FFA (foreign field arithmetics, additions and multiplications) and FFEC (foreign field EC additions) libraries --- practically these will be gates within a kimchi circuit.

Regarding the foreign field additions and multiplications, native Kimchi already supports these:
- https://o1-labs.github.io/proof-systems/kimchi/foreign_field_add.html
- https://o1-labs.github.io/proof-systems/kimchi/foreign_field_mul.html

However, these were designed with very different constraints in mind, and their performance is wholly unsuitable for our needs.

Instead, we start right away with a naive solution --- by using big integers arithmetic, taking limb size to be 15 bits and range checking these. We can revisit the technique and optimise the circuit if required later.


o1js also contains an implementation of ECDSA some relevant ECDSA gadgets. These are based on the existing, unsuitable gates, but still serve as useful reference:
- https://github.com/o1-labs/o1js/blob/main/src/lib/gadgets/elliptic-curve.ts#L99

Non-zero elliptic curve points are most succinctly represented in [affine coordinates](https://www.hyperelliptic.org/EFD/g1p/auto-shortw.html); we will do so here.


### Algorithm Performance, Circuit Layout, and Multi-Circuit Folding

The circuit that we are implementing is large, and its layout should be carefully considered. We implement an algorithm on a *variant* of Kimchi which can allow long rows, additive lookups (random memory access), and support for folding. This will be used to speed the algorithm up. Two last features are (being) developed in the zkVM project.

#### Estimating circuit size

Recall $n$ is the size of the MSM, and $N = 2^{15}$ is the number of rows available. Note that the run of the MSM algorithm with limited buckets takes $C_{\mathsf{msm}} = l * n + 2^{k+1} - 2$ elliptic curve additions at most:
- The cycle that fills buckets using `to_scale_pairs` takes $n$ additions each, and there are $l$ invocations.
- The end loop performs two additions per each iteration. All our buckets are non-zero with overwhelming probability (remember they are initialized by a non-zero $H$), so the loop takes $2 \cdot 2^{k} - 2$ additions (we dont iterate over the zero bucket).

In total:
- For the bigger $n = 2^{16}$ MSM and $k = 15$, we get $C_{\mathsf{msm}} = l \times 2^{16} + (2^{16} - 2)$ additions, which is still more than the $N = 2^{15}$ rows budget.
- Even for the smaller $n = 2^{15}$ MSM and $k = 15$  we still have $C_{\mathsf{msm}} = l \times 2^{15} + (2^{16} - 2)$.

#### Splitting the circuit

We intend to have *one EC addition per row*, split the total $C_{\mathsf{msm}}$ computation into chunks that fit into our $N$-element SRS, and recombine this chunks.

For $n=2^{15}$ the sub-circuits can be as follows:
1. First sub-circuit will be proving just the initial MSM bucket initialisation, `fill_buckets` --- $2^{15}$ additions, this sub-circuit/section we will repeated/folded $l$ times
2. Second sub-section doing the main going-over-the-buckets loop, but only half of it --- another $2^{15}$ additions, this section we only repeat once.
3. Third sub-section is same as the second one, but goes over the second half of the buckets, taking another $2^{15}$ additions, only repeated once.


In the larger $n = 2^{16}$ MSM case, we will have to repeat the first `fill_bucket` chunk $2\cdot l$ times instead of just $l$ (each iteration will handle half of the bases). The second and third chunks are same as before, since $k$, and thus the number of buckets, is the same.


#### Using all the circuits rows

Our circuit uses all or almost all the available rows. However, it is likely we will be able to fit any of the sections of the MSM algorithm (each taking $`\approx 2^{15}`$ rows with one FFEC addition per row) into *exactly* $N = 2^{15}$ rows.

The first chunk is easy --- assuming that our RAM lookup state is shared across all the $l$ or $2l$ invocations of the `fill_buckets`, there is no need for any further communication.

Sharing state between second and third sub-section can be done by reusing some of the buckets as the memory bus.

We will need to adjust the algorithm a bit. Note that we don't have to use `total` and `right_sum` as variables.
Instead, `total` will go into the zero bucket, and `right_sum` can go into the bucket number $2^{k} - 1$ (the last one), and then the second loop in the MSM algorithm can be uniform without any extra single row for aggregation. And thus we are able to split it into sub-circuits 2 and 3 while maintaining common state without any variables, just with RAM lookup. The modified second part of the MSM algorithm is as follows:
```rust
// ... (loop with fill_buckets) ...
// Instead of being a variable, `total` can be "stored" in buckets[0] for efficiency
bucket[0] = -H.scale(2^k * (2^k - 1) / 2);
// Similarly `right_sum` can be "stored" in buckets[2^k-1] for efficiency
// We unroll the first loop iteration because the buckets[2^k-1] slot already contains
// the correct first-iteration value.
bucket[0] += buckets[2^k-1];
// Done in chunk 2
for i in 2..buckets.length()/2 {
    buckets[2^k-1] += buckets[2^k - i];
    buckets[0] += buckets[2^k-1];
}
// Done in chunk 3, state is communicated through the RAM lookup now
for i in bucket.length/2..buckets.length() {
    buckets[2^k-1] += buckets[2^k - i];
    buckets[0] += buckets[2^k-1];
}
```

Finally, to assert that the final computed `buckets[0]` value is what we expect, we need to slightly adjust the additive lookup argument to externally (by modifying lookup boundary conditions) assert that the accumulated computation result (the discrepancy) is contained in the last constraint of the last folding iteration.
- Our additive lookup constraint has a form $`\phi(X) = \phi(X\cdot \omega) + \mathsf{write}(X) - \mathsf{read(X)}`$ where $`\mathsf{read}(\omega^i) = \frac{1}{\alpha + v}`$, and this constraint is run every circuit.
- While building $`\mathsf{read}`$ and $`\mathsf{write}`$ we can use an alternative accumulator boundary condition $`\mathsf{acc}' = \mathsf{acc} +
\frac{1}{\alpha + 0} - \frac{1}{\alpha + v_{0}}`$ embedded into the constraint, where $v_0$ is the value contained in the zero slot. In such a way enforce $v_0$ to be present at the zero address. This approach is already taken in the zkVM implementation.



### Additive Lookups

In the zkVM project there is already an implementation of an additive lookup algorithm. We will need to have RAM variant of it implemented for operations on buckets.
- Even though BN254 supports a pairing and pairing-based additive lookups exist, we need the pairing-free version of the additive lookups, since for RAM lookups it's cheaper and more straightforward.
- We must make sure that the lookup argument supports lookups of the $k$ bit size, so the parameter must align well with the lookup protocol.

Our lookups will be RAM lookups shared across all the circuits.

### Folding and IVC

For folding to work correctly "in its full recursive power", additionally to merely instance folding itself, we will need an IVC (interactive verifiable computation) part that will verify the previous folding iteration within the circuit.
- To implement the IVC for folding we will need to emulate the operations on Pasta non-natively.
    - IVC for folding makes sure two Pasta-based instances "combine" into third Pasta instance.
    - The non-native emulation will use the same foreign field arithmetics we implemented earlier.
    - (this is /not/ related to the Goblin Plonk approach)
- It also needs to be noted that IVC circuit is running "in parallel" --- it is not part of our target circuit, so we can use all the $N = 2^{15}$ available rows (SRS size limit) without worrying about size of the IVC.

### Circuit for converting inputs


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
1. Assemble the /core/ target proving system (parallel with everything before), without folding.
    - Build a variant (clone) of Kimchi with a higher number of columns, and additive lookups (but no folding).
      - These components are now implemented in optimism project to different degrees --- they have to be all brought (ideally reused, practicall probably copied) to a project folder.
      - A MIPS demo repository (https://github.com/o1-labs/mips-demo) contains a variant that we can start from: a kimchi with additive lookups and wide rows.
    - Make sure the proving system works on some simple examples (at least).
1. Implement POC FFA sub-circuit for the modified target proof system. Test and benchmark.
   - A single wide row implementation according to the (hopefully optimal) algorithm chosen in the previous step.
   - Follow the intuitive approach with simple bignum implementation using $15$ bits per limb.
1. Implement POC FFEC sub-circuit for the modified target proof system. Test and benchmark.
   - This will probably be just standard affine addition. Should be less problematic than the previous FFA step.
1. Implement the MSM algorithm in the circuit suggested above. Test and benchmark.
   - The MSM algorithm for now can be implemented without folding in mind. That is, the circuit can be wider, and maybe it can pass more data through inputs, and verify only parts of the MSM of smaller MSM sizes. The point is to have the algorithm working.
1. First milestone: Releasing a reduced many-proofs version of MSM algorithm.
   - This task suggests implementing our MSM algorithm /without/ folding.
   - Instead, we try to verify "small MSM" where everything fits within one circuit. Then we can release $l$ (or $3l$, e.g. 32 or more) independent proofs that will verify *parts* of the MSM.


Phase 2:
1. Bring folding with IVC into our variant of Kimchi.
   - Analyze folding and try to use it in a simple circuit with one of the Pasta curve. Must be able to prove and verify a circuit. It is independent of this work.
1. Implement the MSM algorithm with folding and IVC.
   - The MSM circuit from the previous step should be now properly split into sections and folded.

Phase 3:
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

### Considerations regarding foreign field algorithm

The RFC suggests going forward with [affine](https://www.hyperelliptic.org/EFD/g1p/auto-shortw.html) coordinates. However, [Jacobian](https://www.hyperelliptic.org/EFD/g1p/auto-shortw-jacobian.html#doubling-dbl-2007-bl) and [Projective](https://www.hyperelliptic.org/EFD/g1p/auto-shortw-projective.html) representations are also widely used in cryptography. Again, in practice affine coordinates will be most likely the most efficient for our scenario, and also easy to implement, so the suggested path is to start by implementing those, and revisiting the technique much later in the project timeline.


## Prior art

Regarding FF arithmetics and elliptic curve libraries, we might find inspiration looking at the following existing implementations (however, they are optimised for different scenarios, and generally quite complicated):
- https://github.com/privacy-scaling-explorations/halo2wrong
- https://github.com/DelphinusLab/halo2ecc-s
- https://hackmd.io/@arielg/B13JoihA8


To the best of our knowledge, our main task -- implementing a large MSM verification inside a SNARK -- has not been realised yet by anyone. Even though MSM algorithm exist in the aforementioned libraries, they are usually prohibitively expensive for our MSM size.

<!--Discuss prior art, both the good and the bad, in relation to this proposal.

Prior art is any evidence that your feature (invention, change, proposal) is already known.

Think about the lessons from other blockchain projects or similar updates and provide readers of your RFC with a fuller picture. If there is no prior art, that is fine. Your ideas are interesting whether they are new or adapted from another source.-->

## Unresolved questions

<!-- 1. When splitting MSM into chunks --- is it not problematic that we have to enforce the right order of the chunks? The soundness of the approach relies heavily on the impossibility to swap these chunks around.
Not relevant at the moment
-->


<!--* What parts of the design do you expect to resolve through the RFC process before this RFC gets merged?
* What parts of the design do you expect to resolve through the implementation of this feature before merge?
* What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?-->
