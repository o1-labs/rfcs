# zkVM-Optimized Keccak

This RFC presents a design of a Kimchi chip for the Keccak hash function. It is optimized for the zkVM project, making use of our [generalized expression framework](./0007-generalized-expression-framework.md), and assuming a variant of Kimchi using a Nova-styled folding over BN128 and KZG commitments.

## Summary

This document presents a design to deeply support the Keccak hash function for efficient pre-image oracle access within the proof system. To do so, it leverages a *bitwise sparse representation* to compute boolean operations (XORs, ANDs), and sequences of them.

## Motivation

One component of the Optimism stack system is the pre-image data oracle which is responsible for getting information about the EVM into the MIPS world. The OP stack currently uses the existing ledger which is built on top of Keccak hashes as a baseline.

Having a dedicated chip inside Kimchi to prove correct computation of Keccak hashes from pre-image data (blocks) will result in a more efficient design of the MIPS zkVM as a whole, meeting the performance goals of the [OP RFP](https://github.com/ethereum-optimism/ecosystem-contributions/issues/61#issuecomment-1611488039).

As indicated in the [Keccak PRD](https://www.notion.so/minaprotocol/PRD-Optimized-keccak256-hashing-for-MIPS-VM-33c142f38467464a9b7c21184544dd1d) any member of the OP ecosystem who wanted to prove the state transition between two blocks of an OP stack chain executing the Cannon program in the O(1) MIPS zkVM should complete within 2 days (before optimizations) to meet the latency estimates from the RFP. 

The ultimate objective is to improve the performance of our current Keccak gadget introduced during the Ethereum primitives epic, to get over one hash per second. This is one of the [OKRs of H2 2023](https://www.notion.so/minaprotocol/8dfc798e277447ea860e747b026d9698?v=5daef538906d44db9d12e2e0fbbd4665). 
The new design would be a success if the number of hashes per second is faster than that of the current [SnarkyML Keccak PoC](https://github.com/MinaProtocol/mina/pull/13196). 

The optimizations in the proposed design rely on a more efficient representation of the `Xor64` operation (i.e. a gadget for 64-bit XOR). Targetting this boolean operation makes sense, as this is the most used component inside the permutation function of Keccak. In particular, the main source of overhead in the current design corresponds to this operation.

## Detailed design

The Keccak hash function (later standardized by NIST as SHA3, with small variations) is known to be a quantum-safe, and very efficient hash to compute. Nonethelss, its intensive use of boolean operations which are costly in finite field arithmetic makes it a SNARK-unfriendly function. In this section, we propose an optimized design for our proof system that should outperform our current [Keccak PoC](https://www.notion.so/minaprotocol/Keccak-gadget-PoC-PRD-59b024bce9d5441c8a00a0fcc9b356ae) to become usable in the zkVM for the OP stack.

The [Appendix](#appendix) covers a thorough description of the Keccak algorithm and specifies the required parameters needed for compatibility in our usecase. The reader shall skip this section to find the details of the actual proposal in this RFC.The configuration of Keccak must be the same as that of the Ethereum EVM, meaning specifically:

> * [pre-NIST variant of Keccak](https://keccak.team/files/Keccak-reference-3.0.pdf) (uses the $10^*1$ padding rule)
> * variable input **byte** length (not any bit length)
> * output length of 256 bits
> * input and output in big endian
> * state composed of 25 (5x5) 64-bit words
> * making a block length of 1600 bits ($b$)
> * 512 bits capacity ($c$)
> * 1088 length bit rate ($r := b - c$)
> * 24 rounds of permutation

### Bitwise-sparse representation

Here, we present the bitwise-sparse representation of bitstrings that will be used to obtain more efficient computations of boolean functions within our proof system. 

Given that simple binary operations are very costly within a SNARK, the new idea consists on arithmetizing boolean operations as much as possible, becoming more efficient with finite field algebra. We introduce the _bitwise-sparse representation_ of bitstrings for this goal.

#### Expansion

With this notation, input bitstrings are expanded inserting three `0` bits in between "real" bits (hence the _sparsity_). The motivation for this, is to use intermediate bits to store auxiliary information such as carry bits, resulting from algebraic computations of boolean functions. That means, for each bit in the original bitstring, its expansion will contain four bits. The following expression represents such mapping:

$$\forall X \in \{b_i\}^\ell, b\in\{0,1\}: expand(X) \to 0 \ 0 \ 0 \ b_{\ell-1} \ . \ . \ . \ 0 \ 0 \ 0 \ b_0$$

The reason behind the choice of three empty intermediate bits in the sparse representation follows from the concrete usecase of Keccak where no more than $2^4-1$ consecutive boolean operations are performed, and thus carries cannot overwrite the content of the bits to the leftmost bits.

> NOTE: the above means that after each step of the permutation, the expanded representation needs to be contracted, discard auxiliary bits, and expand again from scratch before starting the next step to ensure completeness of the encoding.

Even though in each field element of ~254 bits long, one could fit up to 63 real bits with this encoding, each witness value will only store 16 such real bits. Instead, the expansion of the full 64 bit words could be represented by composition of each of the four 16-bit parts. But this will not be computed in the circuit, since the expansion would not fit entirely in the field. The way to perform this mapping is through a 16-bit lookup table (of $2^{16}$ entries, larger than that would be too costly). We call this the **sparse** lookup table. 

Let $sparse(X)$ refer to a representation of $X$, where only the indices in the $4i$-th positions correspond to real bits, and the intermediate indices can contain auxiliary information. Connecting to the above, after performing a series of boolean operations to the $expand(X)$ one can obtain some $sparse(X)$, but both encode the same $X$. Concretely,

$$\forall X\in\{b_i\}^\ell:\quad X \equiv \sum_{0}^{\ell-1} 2^i \cdot sparse(X)_{4i} $$

Given that the new field size is 254 bits, the whole 64-bit word expansion cannot fit in one single witness element. Instead, they will be **split** into quarters of 64 bits each (corresponding to real 16 bits each), following this notation:

$$\forall i \in \{0, 3\}:\quad split_i(sparse(X)) := sparse(X_{16i}^{16i+15})$$

Meaning that $\forall X\in(0,2^{64})$:

$$ 
\begin{align*}
sparse(X) \equiv &split_0(sparse(X)) + 2^{64} \cdot split_1(sparse(X)) + \\
& 2^{128}\cdot split_2 (sparse(X)) + 2^{192} \cdot split_3(sparse(X))
\end{align*}
$$

Note that the sparse representation of full 64-bit words will never be computed for that reason. Instead, it will be compressed back again into their relevant bits form, resetting the intermediate bits between steps.

In the implementation, each split is assumed to be one entry of arrays of length 4, so the representation of $split_i(X)$ will be `x[i]`. If the input in question is a 64-bit word, then each entry will contain 16 bits. If the input is in sparse representation, then each entry will contain 64 sparse bits.

#### Compression

Another set of functions will be defined to be used in this new design. For $i\in\{0,3\}$, let $shift_i(X)$ be the function that performs the AND operation $(\wedge)$ of the sparse representation of the word $2^{\ell}-1$ ("all-ones" word in binary), shifted $i$ bits to the left, with the input word $X$. More formally,

$$\forall X \in \{b_i\}^\ell, i\in\{0,3\}: shift_i(X) := \left(\ expand(2^\ell - 1)<<_i 0\ \right)\ \wedge \ X$$ 

meaning, on input some $sparse(X)$, all four of:

$$ 
\begin{align*}
shift_0\left(sparse(X)\right) &:= 0 \ 0 \ 0 \ 1_{\ell-1} \ .\ .\ .\ 0 \ 0 \ 0 \ 1_{0} \ \wedge\ sparse(X) \equiv expand(X)\\
shift_1\left(sparse(X)\right) &:= 0 \ 0 \ 1_{\ell-1} \ 0 \ . \ . \ . \ 0 \ 0 \ 1_{0} \ 0 \ \wedge\ sparse(X) \\
shift_2\left(sparse(X)\right) &:= 0 \ 1_{\ell-1} \ 0 \ 0 \ . \ . \ . \ 0 \ 1_{0} \ 0 \ 0 \ \wedge\ sparse(X) \\
shift_3\left(sparse(X)\right) &:= \ 1_{\ell-1} \ 0 \ 0 \ 0 \ . \ . \ . \ 1_{0} \ 0 \ 0 \ 0 \wedge\ sparse(X) \\
\end{align*}
$$

The effect of each $shift_i$ is to null all but the $(4n+i)$-th bits of the expanded output. Meaning, they act as selectors of every other $(4n+i)$-th bit. That implies that one can rewrite the sparse representation of any word as:

$$
\begin{align*}
sparse(X) \iff & shift_0(sparse(X)) + shift_1(sparse(X)) \\
+ & shift_2(sparse(X)) + shift_3(sparse(X))
\end{align*}
$$

In order to check the real bits behind a sparse representation, one can perform a lookup with the result of the $shift_0$. Note that this table is equivalent to the sparse table presented above. Before, on input a 16-bit word (equivalent to the row index), the table contained the expanded representation. Here instead, on input the expansion (because $shift_0$ has intermediate bits set to zero), one can check the word. 

The correctness of the remaining shifts $(i\in\{1,3\})$ should also be checked. For this, the witness value $aux_i := shift_i/2^i$ will be stored in the witness in such a way that each $aux_i$ can be looked up reusing the table for $shift_0$, and then the following constraint should hold:

$$
\begin{align*}
0 = sparse(X) - (\ & shift_0(sparse(X)) + 2 \cdot aux_1(sparse(X)) \\
& + 4 \cdot aux_2(sparse(X)) + 8 \cdot aux_3(sparse(X))\ )
\end{align*}
$$

>Checking the correct form of the shifts should be done with a single-column lookup table with just the expanded values for the check, since the non-sparse pre-image is non-relevant here.

### Lookup tables

<center>

| Expansion of 16-bits (in binary) |
| ------------------------------- |

| row | expansion of each 16-bit input                                    |
| --- | ----------------------------------------------------------------- |
| $0$ |`0000000000000000000000000000000000000000000000000000000000000000` |
|     | ...                                                               |
| $i$ |`000`$b_{15}$`000`$b_{14}$`000`$b_{13}$`000`$b_{12}$`000`$b_{11}$`000`$b_{10}$`000`$b_{9}$`000`$b_{8}$`000`$b_{7}$`000`$b_{6}$`000`$b_{5}$`000`$b_{4}$`000`$b_{3}$`000`$b_{2}$`000`$b_{1}$`000`$b_{0}$             |
|     | ...                                                               |
| $2^{16}-1$ | `0001000100010001000100010001000100010001000100010001000100010001`        |

</center>

### Basic toolbox

> For efficiency reasons, the state is assumed to be stored in expanded form, so that back-and-forth conversions do not need to take place repeatedly.

#### XOR

The XOR of 16-bit can be computed using addition of their expansions. The 16-bit output of the XOR will be located on the $4n$ positions, whereas the intermediate positions will contain any possible carry terms.

Given expanded `left` and `right` inputs, the sparse representation of $XOR_{64}$ can be constrained in quarters of 16 bits, as:

$$\forall i\in\{0,3\}: sparse(xor_i) = expand(left_i) + expand(right_i)$$

#### Reset

After each step of Keccak, the sparse representation must be reset to avoid overflows of the intermediate bits. For $n \in \{0, 3\}, let $reset_n = shift_n(sparse)/2^n$, the correct decomposition of a 64-bit word can be constrained as

$$\forall i \in \{0,3\}: sparse_i = reset0_i + 2\cdot reset1_i + 4\cdot reset2_i + 8\cdot reset3_i$$

together with a lookup for each `reset_i`.

Each `sparse[i]` will be the output of previous boolean operations. The `reset0[i]` corresponds to the clean expand of the word, and could be the input of some upcoming boolean operations. The `reset1[i]` will be used by the AND. The 64-bit word computed from the `dense_i` will be used in rotation.

##### AND

Note that the AND of two expanded inputs corresponds to the terms in the $4n+1$-th positions. Meaning that $shift_1/2$ gives the expand value of the conjugation operation.
The AND operation can be constrained making use of XOR and Reset explained above. With no further constraints, XOR is used to obtain `left+right`, and then Reset provides the expanded result of AND in the witnesses `reset1`.

#### Negation

The NOT operation will be performed with a subtraction operation using the constant term $(2^{16}-1)$ for each 16-bit quarter, and only later it will be expanded. The mechanism to check that the input is at most 64 bits long, will rely on the composition of the sparse representation. Meaning, 64-bit words produce four 16-bit lookups, which implies that the real bits take no more than 64 bits. Given $x$ in expanded form (meaning, not any sparse representation of $x$ but the initial one with zero intermediate bits) the negation of one 64-bit word can be constrained as:

$$\forall i \in\{0, 3\}: expand(not_i) = 0x1111111111111111 - expand(x_i)$$

> Because the inputs are known to be 16-bits at most, it is guaranteed that the total length of $X$ is at most 64 bits, and is correctly split into quarters `x[i]`.

#### Rotation

> If we could fit the whole 256 expanded bits of the 64-bit word, we could simulate rotations by $x$ bits as a rotation by $4*x$ on the sparse-bit representation, so we could avoid unpacking altogether, which would buy us some additional efficiency gains.

Rotations of $b$ bits to the left will be performed directly on the full 64-bit word $X$ to obtain $Y$, using the algebraic observation that:

$$X \cdot 2^b = Q \cdot 2^{64} + R \quad , \quad Y = Q + R$$ 

Making use of an auxiliary function to compose four 16-bit quarters into full 64-bit terms as:

$$compose \to quarter_0 + 2^{16}quarter_1 + 2^{32}quarter_2 + 2^{48}quarter_3$$

Then, the rotation of `offset` bits to the left can be constrained as:

```rust
    constrain(0 == input * two_to_off - (quotient * 2^64 + remainder))
    constrain(0 == remainder + quotient - output)

    // Check quotient < 2^offset <=> quotient + 2^64 - 2^off < 2^64
    constrain(0 == bound - ( quotient + 2^64 - two_to_off ) )

    for i in [0..4) {
        lookup(rem_quarters[i]) // Check remainder < 2^64
        lookup(quo_quarters[i]) // Check quotient < 2^64
        lookup(aux_quarters[i]) // Check bound < 2^64
    }
```

### Keccak chip

Support for padding shall be provided. In the Keccak PoC, this step takes place at the Snarky layer. It checks that the correct amount of bits in the $10*1$ rule are added until reaching a multiple of 1088 bits, and then adds 512 more zero bits to each block to form a full state.

If the input is not previously expanded, the next step is to expand all 25 words. Each word of 64 bits will be split into 4 parts of 16 real bits each. The expansion itself will be performed through the lookup table containing all $2^{16}$ entries. This step would require $4\times25=100$ lookups.

The support for the Keccak hash function will require the following gate types:

- `StateXOR`: performs XOR of two $5\times5$ states
- `KeccakRound:` performs one full round of the Keccak permutation function

The high-level layout of the gates follows:

| Columns:   | [0...100) | [100...200) | [200...300) |
| ---------- | --------- | ----------- | ----------- |
| `StateXOR` | old_state |  new_state  |  xor_state  |

| Columns:      |            |              |             |           | 
| ------------- | ---------- | ------------ | ----------- | --------- |
| `KeccakRound` | theta_step |  pirho_step  |  chi_step   | iota_step |

#### Step theta

For each row `x in [0..5)` in the state `A`, compute the `C` state:

$$
\begin{align*}
C[x] := &\ A[x][0]\ \oplus\ A[x][1]\ \oplus\ A[x][2]\ \oplus\ A[x][3]\ \oplus\ A[x][4] \\
\iff
sparse(C[x]) := &\ expand(A[x][0]) + expand(A[x][1]) + expand(A[x][2]) +\ & expand(A[x][3]) + expand(A[x][4]) \\
\end{align*} 
$$

| Columns: | [0...100) | [100...120) |
| -------- | --------- | ----------- |
| Theta    | state_a   |  state_c    |

with the following 20 constraints 

```rust
for i in [0..4)
    for x in [0..5)
        constrain(state_c(x)[i] - ( state_a(x,0)[i] + state_a(x,1)[i] + state_a(x,2)[i] + state_a(x,3)[i] + state_a(x,4)[i]) )
```

Similarly, for each row `x in [0..5)`, perform the following operation splitting into quarters

$$
\begin{align*}
D[x] := &\ C[x-1]\ \oplus\ ROT(C[x+1],1) \\
\iff \\
sparse(D[x]) := &\ expand(C[x-1]) + ROT(expand(C[x+1]), 1)\\
\end{align*} 
$$

For this, the 5 possible inputs of the rotation need to be reset. The input of the XOR will use the reset version as well, to reset any previous round auxiliary bits.

| Columns: | [120...200) | [200...220) | [220...240) | [240...260) | [260...280) | [280...300) | [300...320) | [320...340) | 
| -------- | --------- | ----------- | ----------- | ----------- | ----------- | ----------- | ----- | ---- |
| Theta    | reset_c     | dense_c     | quotient_c  | remainder_c | bound_c | dense_rot_c | expand_rot_c | state_d |

This part uses the following 55 constraints

```rust
for x in [0..5)
    for i in[0..4)
        constrain( state_c(x) - (reset0_c(x)[i] + 2*reset1_c(x)[i] + 4*reset2_c(x)[i] + 8*reset3_c(x)[i] ) )

for x in [0..5)
    let word(x)      = dense_c(x)[0]     + 2^16*dense_c(x)[1]     + 2^32*dense_c(x)[2]     + 2^48*dense_c(x)[3]
    let quotient(x)  = quotient_c(x)[0]  + 2^16*quotient_c(x)[1]  + 2^32*quotient_c(x)[2]  + 2^48*quotient_c(x)[3]
    let remainder(x) = remainder_c(x)[0] + 2^16*remainder_c(x)[1] + 2^32*remainder_c(x)[2] + 2^48*remainder_c(x)[3]
    let bound(x)     = bound_c(x)[0]     + 2^16*bound_c(x)[1]     + 2^32*bound_c(x)[2]     + 2^48*bound_c(x)[3]
    let rotated(x)   = dense_rot_c(x)[0] + 2^16*dense_rot_c(x)[1] + 2^32*dense_rot_c(x)[2] + 2^48*dense_rot_c(x)[3]
    constrain( word(x) * 2^1 - ( quotient(x) * 2^64 + remainder(x)) )
    constrain( rotated(x) - (quotient(x) + remainder(x)) )
    constrain( bound(x) - (quotient(x) + 2^64 - 2^1) )

for x in [0..5)
    for i in [0..4)
        constrain( state_d(x)[i] - (reset0_c(x-1)[i] + expand_rot_c(x+1)[1]) )
```

and $112(=80+20+12)$ lookups.


Next, for each row `x in [0..5)` and column `y in [0..5)`, compute (with no need of resetting `D`):

$$
\begin{align*}
E[x][y] := &\ A[x][y]\ \oplus\ D[x] \\
\iff \\
sparse(E[x][y]) := &\ sparse(A[x][y]) + sparse(D[x])\\
\end{align*} 
$$

This part uses the following 100 constraints

```rust
for i in [0..4)
    for x in [0..5)
        for y in [0..5)
            constrain( state_e(x,y)[i] - (state_a(x,y)[i] + state_d(x)[i]) )
```

| Columns: | [340...440) |
| -------- | ----------- |
| Theta    | state_e     |

#### Step pi-rho

For each row `x in [0..5)` and column `y in [0..5)`, perform (into quarters):

$$
\begin{align*}
B[y][2x+3y] := &\ ROT(E[x][y], OFF[x][y]) \\
\iff \\
expand(B[y][2x+3y]) := &\ ROT(expand(E[x][y]), OFF[x][y])
\end{align*} 
$$

| Columns: | [440...840) | [840...940) | [940...1040) | [1040...1140) | [1140...1240) | [1240...1340) | [1340...1440) | [1440...1540) | 
| -------- | ----------- | ----------- | ------------ | ------------- | ------------- | ------------- | ----- | ---- |
| PiRho    | reset_e     | dense_e     | quotient_e   | remainder_e   | bound_e       | dense_rot_e   | expand_rot_e | state_b |

Recall that in order to perform the rotation operation, the state needs to be reset. This step can be carried out with the following $125$ constraints and $800 (=400+100+100+100+100)$ lookups:

```rust
for x in [0...5)
    for y in [0...5)
        constrain( state_e[i] - (reset0_e(x,y)[i] + 2*reset1_e(x,y)[i] + 4*reset2_e(x,y)[i] + 8*reset3_e(x,y)[i] ) )
        let word(x,y)      = dense_e(x,y)[0]     + 2^16*dense_e(x,y)[1]     + 2^32*dense_e(x,y)[2]     + 2^48*dense_e(x,y)[3]
        let quotient(x,y)  = quotient_e(x,y)[0]  + 2^16*quotient_e(x,y)[1]  + 2^32*quotient_e(x,y)[2]  + 2^48*quotient_e(x,y)[3]
        let remainder(x,y) = remainder_e(x,y)[0] + 2^16*remainder_e(x,y)[1] + 2^32*remainder_e(x,y)[2] + 2^48*remainder_e(x,y)[3]
        let bound(x,y)     = bound_e(x,y)[0]     + 2^16*bound_e(x,y)[1]     + 2^32*bound_e(x,y)[2]     + 2^48*bound_e(x,y)[3]
        let rotated(x,y)   = dense_rot_e(x,y)[0] + 2^16*dense_rot_e(x)[1] + 2^32*dense_rot_e(x,y)[2] + 2^48*dense_rot_e(x,y)[3]
        constrain( word(x,y) * 2^(OFF(x,y)) - ( quotient(x) * 2^64 + remainder(x)) )
        constrain( rotated(x,y) - (quotient(x,y) + remainder(x)) )
        constrain( bound(x,y) - (quotient(x,y) + 2^64 - 2^(OFF(x,y)) )
        constrain( state_b(y,2x+3y) - expand_rot_e(x,y) )
```

Note that the value `OFF(x,y)` will be a different constant for each index, according to the rotation table of Keccak.


#### Step chi

For each row `x in [0..5)` and column `y in [0..5)`, perform (into quarters):

$$
\begin{align*}
F[x][y] := &\ B[x][y] \oplus (\ \neg B[x+1][y] \wedge \ B[x+2][y]) \\
\iff \\
2 \cdot sparse(F[x][y]) := &\ 2 \cdot expand(B[x][y]) + shift_1((expand(2^{64}-1) - expand(B[x+1][y]) + expand(B[x+2][y]) ) )
\end{align*} 
$$


| Columns: | [1540...1940) | [1940...2340) | [2340...2440) |
| -------- | ------------- | ------------- | ------------- | 
| PiRho    | reset_b       | reset_sum     | state_f       |


This is constrained with the following $200$ constraints and $800$ lookups

```rust
for i in [0..4)
    for x in [0..5)
        for y in [0..5)
            let not(x,y)[i] = 0x1111111111111111 - reset0_b(x+1,y)[i]
            let sum(x,y)[i] = not(x+1,y)[i] + reset1_b(x+2,y)[i]
            constrain( sum[i] - (reset0_sum(x,y)[i] + 2*reset1_sum(x,y)[i] + 4*reset2_sum(x,y)[i] + 8*reset3_sum(x,y)[i] ) )
            let and(x,y)[i] = reset1_sum(x,y)[i] 
            constrain( state_f(x,y)[i] - (reset0_b(x,y)[i] + and(x,y)[i]) )
```

#### Step iota

On round $r$, update the word in the first row, first column xoring with the round constant as

$$
G[0][0] \oplus RC[r] \iff sparse(G[0][0]) + expand(RC[r])
$$

The round constants should be stored in expanded form, taking $24\times4$ witness cells as public inputs. This only requires $4$ constraints and no lookups.

```rust
for i in [0..4)
    constrain( state_g(0,0)[i] - (state_f(0,0)[i] + expand(RC[r]))[i] )
```

| Columns: | [2440...2441) |
| -------- | ------------- | 
| Iota     | g_0_0         | 

After this last step of the permutation function, `KeccakRound` will store state `G` in the first $100$ cells of the next row, to be chained with the upcoming `StateXOR`. This requires $100$ copy constraints. Recall that except for `g_0_0`, the rest of `G` is `state_f`.

### Performance

Counting the costs of the steps presented above, the following table summarizes the performance of the proposed design, for each of the 24 rounds of the permutation function, in each design of the Keccak gadget.

<center>

| Version  | Columns | Rows / block | Lookups / block | 
|----------|---------|--------------|-----------------|
| This RFC | 2441   | $24\times2=48$ | $24\times(112+800+800)=41,088$ |                
| Old RFC | 14       | $24\times(50+25+20+25+75+150+1)=8,304$ | $24\times(0+0+140+0+700+400+0)=29,760$ |                
| Old PoC  | 15      | $24\times(125+100+40+125+75+287.5+5)=18,180$ |$24\times(400+320+140+400+300+800+16)=57,024$ |      

</center>

> Possible miscount in the lookups

## Test plan and functional requirements

1. Testing goals and objectives: 
    * The resulting Keccak should behave just like the [implementations](https://keccak.team/software.html) suggested by the Keccak original team.
    * The performance of the system should not be affected.
    * It will produce the expected block hash.
2. Testing approach: 
    * Expected digests are obtained upon input pre-images.
3. Testing scope: 
    * Thorough unit tests of the gadget and its components (XOR, AND, NOT, rotation).
4. Testing requirements: 
    * When the project is in a more advanced state, run integration tests with other parts of the system (MIPS ISA Implementation, and sys-call for blocks via Cannon).
5. Testing resources: 
    * Run the official [test vectors](https://keccak.team/archives.html).
    * Obtain actual examples of blocks to hash, with different lengths.

## Drawbacks
[drawbacks]: #drawbacks

This gate uses much longer lookup tables. Understand if the gains in the number of rows of Keccak compensate for the necessity of a $2^{16}$ lookup table.

## Rationale and alternatives

## Prior art

The current [Keccak PoC in SnarkyML](https://www.notion.so/minaprotocol/Keccak-gadget-PoC-PRD-59b024bce9d5441c8a00a0fcc9b356ae) was introduced to support Ethereum primitives in MINA zkApps. Due to this blockchain's design, the gadget needed to be compatible with Kimchi: a Plonk-like SNARK instantiated with Pasta curves for Pickles recursion and IPA commitents. This means that the design choices for that gadget were determined by some features of this proof system, such as: 15-column width witness, 7 permutable witness cells per row, access to the current and next rows, up to 4 lookups per row, access to 4-bit XOR lookup table, and less than $2^{16}$ rows. As a result, proving the Keccak hash of a message of 1 block length (up to 1080 bits) took ~15k rows.

## Unresolved questions

* During the implementation of this RFC:
    * obtain exact measurements of the number of rows, columns, constraints, lookups, seconds, required per block hash;
    * find out if the round constants should be hardcoded (takes memory space) or generated (takes computation resources);
    * decide if the rotation offsets will be directly stored modulo 64 or not;
    * if the endianness of the target input format is little endian, it will need to be transformed into big endian;
    * if the input is given as a bitstring, pack it into bytes.

* Future work: 
    * support SHA3 (NIST variant of Keccak), different output lengths, and different state widths;


## Relevant links

* [Pros/Cons of MIPS VM](https://www.notion.so/minaprotocol/MIPS-VM-68c09743b79e4e2bab9e6952c27f0ab4)
* [RFP: OP Stack Zero Knowledge Proof](https://github.com/ethereum-optimism/ecosystem-contributions/issues/61#issuecomment-1611488039)
* [PRD: Optimized keccak256 hashing (for MIPS VM)](https://www.notion.so/minaprotocol/PRD-Optimized-keccak256-hashing-for-MIPS-VM-33c142f38467464a9b7c21184544dd1d)
* [O(1) Labs'  H2 2023 OKRs](https://www.notion.so/minaprotocol/8dfc798e277447ea860e747b026d9698?v=5daef538906d44db9d12e2e0fbbd4665)
* [SnarkyML Keccak PoC](https://github.com/MinaProtocol/mina/pull/13196)
* [Keccak gadget PoC PRD for Ethereum Primitives](https://www.notion.so/minaprotocol/Keccak-gadget-PoC-PRD-59b024bce9d5441c8a00a0fcc9b356ae)
* [The Keccak reference document](https://keccak.team/files/Keccak-reference-3.0.pdf)

## Appendix

### Keccak algorithm

The high level idea of this sponge-based algorithm iteratively is to apply a permutation function to an initial state for a number of rounds (absorb phase) and to finally crop the state, when needed, obtaining the digest (squeeze) as follows:

<center>

![](https://upload.wikimedia.org/wikipedia/commons/7/70/SpongeConstruction.svg)

</center>

The following represents the description of the Keccak algorithm tailored to our usecase configuration presented above.

1. **Input:** message
2. **Requirement:** message is a byte string in big endian
    - If the input was given as bits and its length was not a multiple of 8 bits, represent the MSB as a full byte before the next step.
3. **Pad:** pad the message with the padding rule $10^*1$
    - Append the byte `0x01`. 
    - Afterwards, append as many `0x00` bytes needed until the length of the resulting message is a multiple of the bit rate (1088 bits). 
    - XOR the last byte with `0x80`. 
    > **Edge cases:** input messages whose length is
    > - _0:_ reject, the "smallest" message is `0x00` which takes 8 bits in length, and will need 1080 padding bits.
    > - _(1088n) a multiple of the bit rate:_ the padding will take a whole new block of another 1088 bits.
    > - _(1088n - 1) just below the bit rate:_ the padding will take one single byte `0x81`.
4. **Initialization:** create a root state $s$ of 1600 bits set to zero.
    - For convenience, store it as a 5x5 matrix with 64-bit words in each cell.
5. **Notation:** 
    - `s[i][j][k]` refers to the k-th least significant byte of the i-th row of the j-th column of the state matrix. 
    - `message[n]` refers to the n-th most significant byte of the bytestring (big endian).
6. **Absorb:** for each block of 136 bytes (1088 bits) in the padded bytestring
    - Pad with 64 `0x00` bytes until reaching a total of 1600 bits.
    - Map the resulting bytestring to state format
    - Update the state xoring it with the current block state.
        - Meaning either the root state for the first block, or the previous state for the rest of blocks.
    - Apply the permutation function to the state.
7. **Stringify:** map the state to a bytestring and store the first 1088 bits in an output bytestring.
8. **Squeeze:** while the bitlength of the resulting bytestring is less than the desired output bit
    - Apply the permutation function to the state
    - Map the resulting state to a bytestring
    - Append the first rate number of bits to the output bytestring
9. **Hash:** Take the first 256 bits (32 bytes) of the output
10. **Return:** The hash digest obtained above

> **NOTE** The squeeze step does not take place with our output length of 256 bits and 1088 bit rate.

In order to perform the above steps correctly, mapping bytes into the state in the right order is key:

- `from_bytes_to_state`: the absorb step requires a mapper from a bytestring into the state format.
- `from_state_to_bytes`: the squeeze steps requires a mapper from a state into the bytestring format.

In particular, this is the order that should be followed. On input a list of bytes in big endian format (the first element corresponds to the left-most byte in a string in hexadecimal format), the first element is mapped to `M[0][0][0]` (the least significant byte of the word in the first column and first row). The following 7 bytes are mapped to the subsequent bytes of `M[0][0]`. The next 8 bytes will go to the cell in the first column, second row `M[1][0]`. The following table illustrates the order of the 25 64-bit words in the state: 

<center>

| x \ y | 0 | 1 |  2 |  3 |  4 |
| ----- | - | - | -- | -- | -- |
| **0** | 0 | 5 | 10 | 15 | 20 |
| **1** | 1 | 6 | 11 | 16 | 21 |
| **2** | 2 | 7 | 12 | 17 | 22 |
| **3** | 3 | 8 | 13 | 18 | 23 |
| **4** | 4 | 9 | 14 | 19 | 24 |

</center>

Then, within each word, the first bytes of the bytestring get mapped to the least significant bytes of the word in the state as:



More formally, the sequence to convert from bytestring to state follows the pseudocode:

```rust
for y in [0..5) { // for each column
    for x in [0..5) { // for each row
        for z in [0..8) { // for each byte in the word
            index = 8 * (5 * y + x) + z
            state[x][y] |= (byte[index]) << 8 * z
        }
    }
}
```

The same strategy (but reverse order) shall be followed to convert from the state back to bytestring at the end of the sponge, meaning:

```rust
for y in [0..5) { // for each column
    for x in [0..5) { // for each row
        for z in [0..8) { // for each byte in the word
            byte.push(((state[x][y] >> 8 * z) & 0xFF));
        }
    }
}
```

#### Permutation function

The main responsible for the security guarantees (pre-image and collision resistance) is the permutation function that is executed inside the sponge. It applies the same algorithm iteratively to a state, for 24 rounds.

> This number comes from the formula $12+2·\ell$ where $\ell=6$ in our case with ($2^6 =$)64-bit words. 

```rust
fn permutation(state) -> state {
    for r in [0..24) {
        state <- round(state, r)
    }
    return state
}
```

The round function itself is the concatenation of a number of simple algorithms whose duty is to randomize the state in a non-reversible manner. The notation choice here is meant to represent the letters chosen in the official docs of Keccak as closely as possible.

```rust
fn round(state_in, r) -> state_out {
    A <- state_in
    E <- theta(A)
    B <- pi_rho(E)
    F <- chi(B)
    D <- iota(F, r)
    return D
}
```

In the following, we will show the pseudocode of each of these algorithms. Note that positions in X and Y coordinates must be computed modulo 5.
> Make sure that your language performs the remainder operation instead of modulo. Otherwise, make sure to add `+5` in subtractions to avoid negative indices.

```rust
fn theta(state_a) -> state_e {
    A <- state_a
    for x in [0..5) {
        C[x] <- A[x][0] xor A[x][1] xor A[x][2] xor A[x][3] xor A[x][4]  
    } 
    for x in [0..5) {
        D[x] <- C[x-1] xor rot(C[x+1], 1) // Left rotation by 1 bit (towards more significant positions)
        for y in [0..5) {
            E[x][y] <- A[x][y] xor D[x]
        }
    }
    return E
}
```

```rust
fn pi_rho(state_e) -> state_b {
    E <- state_e
    for x in [0..5) {
        for y in [0..5) {
            B[y][2x+3y] <- rot(E[x][y], OFF[x][y])
        }
    }
    return B
}
```

```rust
fn chi(state_b) -> state_f {
    B <- state_b
    for x in [0..5) {
        for y in [0..5) {
            F[x][y] <- B[x][y] xor (not B[x+1][y] and B[x+2][y])
        }
    }
    return F
}
```

```rust
fn chi(state_f, r) -> state_g {
    G <- state_f
    G[0][0] <- F[0][0] xor RC[r]
    return G
}
```

#### Constants

Tha algorithms above make use of two sets of constants: 
- rotation table `OFF[x][y]` is a 5x5 matrix defining a constant rotation offset for each word in the state. For 64-bit words, it corresponds to:

<center>

| x \ y |  0 |  1 |  2 |  3 |  4 |
| ----- | -- | -- | -- | -- | -- |
| 0     |  0 | 36 |  3 | 41 | 18 |
| 1     |  1 | 44 | 10 | 45 |  2 |
| 2     | 62 |  6 | 43 | 15 | 61 |
| 3     | 28 | 55 | 25 | 21 | 56 |
| 4     | 27 | 20 | 39 |  8 | 14 |

</center>

- round constants `RC[r]` in our case is a 24-length array of 64-bit values which are defined as the output of the binary linear feedback shift register (LFSR) given by

$$RC[r] = (x^r \mod{x^8 + x^6 + x^5 + x^4 + 1} )\mod{x} $$

<center>

| `r` |                 `RC` |
| --- | -------------------- |
|  0  | `0x0000000000000001` |
|  1  | `0x0000000000008082` |
|  2  | `0x800000000000808A` |
|  3  | `0x8000000080008000` |
|  4  | `0x000000000000808B` |
|  5  | `0x0000000080000001` |
|  6  | `0x8000000080008081` |
|  7  | `0x8000000000008009` |
|  8  | `0x000000000000008A` |
|  9  | `0x0000000000000088` |
| 10  | `0x0000000080008009` |
| 11  | `0x000000008000000A` |
| 12  | `0x000000008000808B` |
| 13  | `0x800000000000008B` |
| 14  | `0x8000000000008089` |
| 15  | `0x8000000000008003` |
| 16  | `0x8000000000008002` |
| 17  | `0x8000000000000080` |
| 18  | `0x000000000000800A` |
| 19  | `0x800000008000000A` |
| 20  | `0x8000000080008081` |
| 21  | `0x8000000000008080` |
| 22  | `0x0000000080000001` |
| 23  | `0x8000000080008008` |

</center>
