# zkVM-Optimized Keccak

This RFC presents a design of a Kimchi chip for the Keccak hash function. It is optimized for the zkVM project, making use of our [generalized expression framework](./0007-generalized-expression-framework.md), and assuming a variant of Kimchi using a Nova-styled folding over BN128 and KZG commitments. The generalized expression framework will provide the ability to create arbitrary number of columns and lookups per row for custom gates. This allows more optimized approaches to address boolean SNARK-unfriendly relations.

## Summary

This document presents a design to deeply support the Keccak hash function for efficient pre-image oracle access within the proof system. To do so, it leverages a *bitwise sparse representation* to compute boolean operations (XORs, ANDs), and sequences of them.

## Motivation

One component of the Optimism stack system is the pre-image data oracle which is responsible for getting information about the chains. The OP stack currently uses the existing ledger which is built on top of Keccak hashes as a baseline.

Having a dedicated chip inside Kimchi to prove correct computation of Keccak hashes from pre-image data (blocks) will result in a more efficient design of the MIPS zkVM as a whole, meeting the performance goals of the [OP RFP](https://github.com/ethereum-optimism/ecosystem-contributions/issues/61#issuecomment-1611488039).

As indicated in the PRD, any member of the OP ecosystem who wanted to prove the state transition between two blocks of an OP stack chain executing the `op-program` program in the O(1) MIPS zkVM should complete within 2 days (before optimizations) to meet the latency estimates from the RFP. 

The ultimate objective is to improve the performance of our current Keccak gadget, to get over one hash per second. The new design would be a success if the number of hashes per second is faster than that of the current [SnarkyML Keccak PoC](https://github.com/MinaProtocol/mina/pull/13196). 

The optimizations in the proposed design rely on a more efficient representation of the `Xor64` operation (i.e. a gadget for 64-bit XOR). Targetting this boolean operation makes sense, as this is the most used component inside the permutation function of Keccak. In particular, the main source of overhead in the current design corresponds to this operation.

## Detailed design

The Keccak hash function (later standardized by NIST as SHA3, with small variations) is known to be a quantum-safe, and very efficient hash to compute. Nonethelss, its intensive use of boolean operations which are costly in finite field arithmetic makes it a SNARK-unfriendly function. In this section, we propose an optimized design for our proof system that should outperform our current Keccak to become usable in the zkVM for the OP stack.

The [Appendix](#appendix) covers a practical description of the Keccak algorithm and specifies the required parameters needed for compatibility in our use case. The configuration of Keccak must be the same as that of the Ethereum EVM, meaning specifically:

> * [pre-NIST variant of Keccak](https://keccak.team/files/Keccak-reference-3.0.pdf) (uses the $10^*1$ padding rule)
> * variable input **byte** length (not any bit length)
> * output length of 256 bits
> * input and output in big endian
> * state composed of 25 (5x5) 64-bit words
> * making a block length of 1600 bits ($b$)
> * 512 bits capacity ($c$)
> * 1088 length bit rate ($r := b - c$)
> * 24 rounds of permutation

This section focuses on the actual changes for the proposed gadget.

### Input-Output format

The Keccak gadget is triggered from the sys call. Inputs need to be written into memory and outputs are read from the sys call. One should expect inputs and outputs to be found in bytes serialization, with a big-endian encoding. One can only read from Canon 4 bytes at a time, what means that the hash function will have to wait until the input is fully loaded onto memory.

### Bitwise-sparse representation

Here, we present the bitwise-sparse representation of bitstrings that will be used to obtain more efficient computations of boolean functions within our proof system. 

Given that simple binary operations are very costly within a SNARK, the new idea consists on arithmetizing boolean operations as much as possible, becoming more efficient with finite field algebra. We introduce the _bitwise-sparse representation_ of bitstrings for this goal.

#### Expansion

With this notation, input bitstrings are expanded inserting three `0` bits in between "real" bits (hence the _sparsity_). The motivation for this, is to use intermediate bits to store auxiliary information such as carry bits, resulting from algebraic computations of boolean functions. That means, for each bit in the original bitstring, its expansion will contain four bits. The following expression represents such mapping:

$$\forall X \in (b_i)^\ell, b\in[0,1]: expand(X) \to 0 \ 0 \ 0 \ b_{\ell-1} \ . \ . \ . \ 0 \ 0 \ 0 \ b_0$$

The reason behind the choice of three empty intermediate bits in the sparse representation follows from the concrete use case of Keccak where no more than $2^4-1$ consecutive boolean operations are performed, and thus carries cannot overwrite the content of the bits to the leftmost bits.

> NOTE: the above means that after each step of the permutation, the expanded representation needs to be reset before starting the next step to ensure completeness of the encoding. 

Even though in each field element of ~254 bits long, one could fit up to 63 real bits with this encoding, each witness value will only store 16 such real bits. Instead, the expansion of the full 64 bit words could be represented by composition of each of the four 16-bit parts. But this will not be computed in the circuit, since the expansion would not fit entirely in the field. The way to perform this mapping is through a 16-bit 2-column lookup table that we call `Reset` (of $2^{16}$ entries, anything significantly larger than that such as 32 bits would be too costly).

Let $sparse(X)$ refer to a representation of $X$, where only the indices in the $4i$-th positions correspond to real bits, and the intermediate indices can contain auxiliary information. Connecting to the above, on input a word $X$, we can expand it in order to perform a series of boolean operations (up to 15) which results in some $sparse(X)$, encoding the same $X$. Then,

$$\forall X\in(b_i)^\ell:\quad expand(X) \equiv reset(sparse(X)) $$

#### Compression

Another set of functions will be defined to be used in this new design. For $i\in[0,3]$, let $mask_i(X)$ be the function that performs the AND operation $(\wedge)$ of the sparse representation of the word $2^{\ell}-1$ ("all-ones" word in binary), shifted $i$ bits to the left, with the input word $X$. More formally,

$$\forall X \in (b_i)^\ell, i\in[0,3]: mask_i(X) := \left(\ expand(2^\ell - 1)<<_i 0\ \right)\ \wedge \ X$$ 

meaning, on input some $sparse(X)$, all four of:

$$ 
\begin{align*}
mask_0\left(sparse(X)\right) &:= 0 \ 0 \ 0 \ 1_{\ell-1} \ .\ .\ .\ 0 \ 0 \ 0 \ 1_{0} \ \wedge\ sparse(X) \equiv expand(X) \equiv reset(sparse(X))\\
mask_1\left(sparse(X)\right) &:= 0 \ 0 \ 1_{\ell-1} \ 0 \ . \ . \ . \ 0 \ 0 \ 1_{0} \ 0 \ \wedge\ sparse(X) \\
mask_2\left(sparse(X)\right) &:= 0 \ 1_{\ell-1} \ 0 \ 0 \ . \ . \ . \ 0 \ 1_{0} \ 0 \ 0 \ \wedge\ sparse(X) \\
mask_3\left(sparse(X)\right) &:= \ 1_{\ell-1} \ 0 \ 0 \ 0 \ . \ . \ . \ 1_{0} \ 0 \ 0 \ 0 \wedge\ sparse(X) \\
\end{align*}
$$

The effect of each $mask_i$ is to null all but the $(4n+i)$-th bits of the expanded output. Meaning, they act as selectors of every other $(4n+i)$-th bit. That implies that one can rewrite the sparse representation of any word as:

$$
\begin{align*}
sparse(X) \iff & mask_0(sparse(X)) + mask_1(sparse(X))
+ mask_2(sparse(X)) + mask_3(sparse(X))
\end{align*}
$$

In order to check correctness of a decomposition, we can rely on 4 lookups. The first one is a lookup to the `Reset` table using the "dense" word for the first column, and `mask_0` for the second column (because it is equivalent to the expansion). 
The correctness of the remaining masks $(i\in[1,3])$ should also be checked. For this, the witness value $shift_i := mask_i/2^i$ will be stored in the witness in such a way that each $shift_i$ can be looked up using the "second column" of the `Reset` table (more about this later), and then the following constraint should hold:

$$
\begin{align*}
0 = sparse(X) - (\ & reset_0(sparse(X)) + 2 \cdot shift_1(sparse(X)) \\
& + 4 \cdot shift_2(sparse(X)) + 8 \cdot shift_3(sparse(X))\ )
\end{align*}
$$

### Basic toolbox

#### XOR

The XOR of 16-bit can be computed using addition of their expansions. The 16-bit output of the XOR will be located on the $4n$ positions, whereas the intermediate positions will contain any possible carry terms.

Given expanded `left` and `right` inputs, the sparse representation of $XOR_{64}$ can be constrained in quarters of 16 bits, as:

$$\forall i\in[0,3]: sparse(xor_i) = expand(left_i) + expand(right_i)$$

Nonetheless, if no more than 15 boolean operations have been performed since the latest reset, one can use sparse representations of the left and right inputs as well.

#### Reset

After each step of Keccak, the sparse representation must be reset to avoid overflows of the intermediate bits. For $n \in [0, 3], let $shift_n = mask_n(sparse)/2^n$, the correct decomposition of a 64-bit word can be constrained as

$$\forall i \in [0,3]: sparse_i = shift0_i + 2\cdot shift1_i + 4\cdot shift2_i + 8\cdot shift3_i$$

together with a lookup for each `shift_i` (more about this later).

Each `sparse[i]` will be the output of previous boolean operations. The `shift0[i]` corresponds to the clean expand of the word, and could be the input of some upcoming boolean operations. The `shift1[i]` will be used by the AND. The 64-bit word computed from the `dense[i]` will be used in rotation.

##### AND

Note that the AND of two expanded inputs corresponds to the $4n+1$-th positions after an expanded XOR (addition). Meaning that $shift_1 = mask_1/2$ gives the expanded representation of the conjugation operation.
The AND operation can be constrained making use of XOR and Reset explained above. With no further constraints, XOR is used to obtain `left+right`, and then Reset provides the expanded result of AND in the `shift1` witnesses.

#### Negation

The NOT operation will be performed with a subtraction operation using the constant term $(2^{16}-1)$ for each 16-bit quarter, and only later it will be expanded. The mechanism to check that the input is at most 64 bits long, will rely on the composition of the sparse representation. Meaning, 64-bit words produce four 16-bit lookups, which implies that the real bits take no more than 64 bits. Given $x$ in reset form (meaning, not any sparse representation of $x$ but the initial expanded one with zero intermediate bits) the negation of one 64-bit word can be constrained as:

$$\forall i \in[0, 3]: expand(not_i) = 0\text{x}1111111111111111 - expand(x_i)$$

> Because the inputs are known to be 16-bits at most, it is guaranteed that the total length of $x$ is at most 64 bits, and is correctly split into quarters `x[i]`.

#### Rotation

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
```

Together with $3\times4=12$ lookups to check that the remainder, quotient and bound are $<2^{64}$.

> Assuming that we execute rotation starting form reset values, then one could perform the above operations directly on the expanded terms, since $4\cdot64-3=253$ encoded bits (removing the three leading zeros) would fit in the BN254 curve. In this scenario, rotations by $x$ bits would be mapped to rotations by an offset of $4\cdot x$ bits instead. Moreover, the constraints would need to be slightly different, to avoid overflows. In particular: `word = quotient * 2^(253-4*offset) + shift` and `output = quotient + shift * 2^(4*offset)`, and the output would already be in expanded format.  Nonetheless, in this case we would also need to perform bound checks, which would be much more costly due to the bitlengths. And in any case, each dense word should be expanded afterwards to be processed in later steps.

### Constraints


> For efficiency reasons, the state is assumed to be stored in expanded form, so that back-and-forth conversions do not need to take place repeatedly.

The support for the Keccak hash function will require the following gate types:

**PRECONDITIONS:** the message to be hashed should be padded with the $10^*1$ rule before 

- `KeccakSponge`: performs formatting actions related to the Keccak sponge.
- `KeccakRound`: performs one full round of the Keccak permutation function.

The high-level layout of the gates follows:

| `KeccakSponge` | [0...100) | [100...168) | [168...200) | [200...400) | [400...800) |
| -------------- | --------- | ----------- | ----------- | ----------- | ----------- |
| Curr           | old_state | new_block   | zeros       | bytes      | shifts      |
| Next           | xor_state |

| `KeccakRound` | [0...440)   | [440...1540) | [1540...2440) | 
| ------------- | ----------- | ------------ | ------------- | 
| Curr          | theta_step  | pirho_step   |  chi_step     |

| `KeccakRound` | [0...100)   |
| ------------- | ----------- |
| Next          | iota_step   |

#### Sponge

Support for padding shall be provided. In the Keccak PoC, this step took place at the Snarky layer. It checks that the correct amount of bits in the $10^*1$ rule are added until reaching a multiple of 1088 bits, and then adds 512 more zero bits to each block to form a full state.

If the input was not previously expanded, the next step would be to expand all 25 words. Each word of 64 bits would have to be split into 4 parts of 16 real bits each. The expansion itself would be performed through the `Reset` lookup table. This step would require $4\times25=100$ lookups.

There are two main modes: absorb and squeeze. Inside the absorb mode, there are two additional submodes called pad and root, which can as well happen at the same time. 


- **Absorb mode**

    In absorb mode, the first coefficient is set to 1: `absorb := coeff[0] = 1`, and the second coefficient is 0.

    The gate takes a block as 68 chunks of 16 bits expanded, pads with zeros until reaching 100 chunks (1600 bits) to fit one whole state, and XORs it with the previous state.

    The zero padding is enforced performing the following 32 constraints:

    ```rust
    constrain(absorb * zeros[i])
    ```

    The XOR part of the gate uses $100$ constraints 

    ```rust
    constrain(absorb * (xor_state[i] - (old_state[i] + new_state[i])))
    ```
    where `new_state` is the concatenation of `new_block` and `zeros`.

    Additionally, the gate must check the decomposition into shifts of the new block.

    ```rust
    for i in [0..100) {
        constrain(absorb * (new_block[i] - (
                            shifts0(i)
                            + 2 * shifts1(i)
                            + 4 * shifts2(i)
                            + 8 * shifts3(i) )))
    }
    ```

    The correspondence between the shifts and the dense bytes will happen inside a lookup pattern.

    - **Root mode**

        In this mode, the third coefficient is set to 1: `root := coeff[2] = 1`, and `absorb = 1, squeeze = 0`. 

        This mode is only performed once per hash, and it must happen in the first absorb. The only difference between this one and any other absorb is that the root mode sets the initial state to all zeros. For that reason, apart from the constraints above, it checks:

        ```rust
        constrain(root * old_state[i])
        ```
    
    - **Pad mode** 

        Padding with the $10^*1$ rule only happens once per hash. But unlike the root mode, padding happens only in the last absorb. That is because the input message is padded until reaching a length that is a multiple of 136 bytes, which always fully fits in one whole block. This means, in the case that the input message is $\leq 135$ bytes in length, the single absorb will run in both `root` and `pad` modes.

        When in pad mode, we will have 136 selector flags (coefficient indices `[4..140)`) that will activate at most 136 positions corresponding to the padding. Additionally, each of the 136 coefficients corresponding to the bytes of the new block are set to either `0x00`, `0x01`, `0x80`, or `0x81`, which are all the possible combinations that a padding byte can take. In this mode, `absorb = 1, squeeze = 0`. 
        
        Altogether, in pad mode we obtain the following 136 constraints:

        ```rust
        constrain( flag[i] * (pad[i] - bytes[i]) )
        ```

        Moreover, the gate must show the link between the bytes and the actual state. For that, 



- **Squeeze mode**

    The gate takes the first 256 bits of the digest (first 4 words of the first column of the state, meaning first 16 expanded terms), and decomposes from the expanded state to form 32 bytes corresponding to the hash. 

    In squeeze mode, the second coefficient is set to 1: `squeeze := coeff[1] = 1`, and rest of coefficients are 0.

    Here, the state being decomposed into shifts is the one resulting from the most recent permutation round:

    ```rust
    for i in [0..16) {
        constrain(squeeze * (old_state[i] - 
                                (shift[4i] 
                                    + 2*shift[4i+1] 
                                    + 4*shift[4i+2] 
                                    + 8*shift[4i+3]))) 
    }
    ```

    The correspondence between the shifts and the dense bytes will happen inside a lookup pattern.

#### Step theta

For each row `x in [0..5)` in the state `A`, compute the `C` state:

$$
\begin{align*}
C[x] := &\ A[x][0]\ \oplus\ A[x][1]\ \oplus\ A[x][2]\ \oplus\ A[x][3]\ \oplus\ A[x][4] \\
\iff
sparse(C[x]) := &\ expand(A[x][0]) + expand(A[x][1]) + expand(A[x][2]) + expand(A[x][3]) + expand(A[x][4]) \\
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
        constrain( state_c(x)[i] - (reset0_c(x)[i] + 2*reset1_c(x)[i] + 4*reset2_c(x)[i] + 8*reset3_c(x)[i] ) )

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
        constrain( state_d(x)[i] - (reset0_c(x-1)[i] + expand_rot_c(x+1)[i]) )
```

and $180$ lookups.


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
| -------- | ------------ | ------------- | ------------- | ------------- | ------------- | ------------- | ----- | ---- |
| PiRho    | reset_e      | dense_e       | quotient_e    | remainder_e   | bound_e       | dense_rot_e   | expand_rot_e | state_b |

Recall that in order to perform the rotation operation, the state needs to be reset. This step can be carried out with the following $275(=75+200)$ constraints and $800 (=400+100+100+100+100)$ lookups:

```rust
for x in [0...5)
    for y in [0...5)
        let word(x,y)      = dense_e(x,y)[0]     + 2^16*dense_e(x,y)[1]     + 2^32*dense_e(x,y)[2]     + 2^48*dense_e(x,y)[3]
        let quotient(x,y)  = quotient_e(x,y)[0]  + 2^16*quotient_e(x,y)[1]  + 2^32*quotient_e(x,y)[2]  + 2^48*quotient_e(x,y)[3]
        let remainder(x,y) = remainder_e(x,y)[0] + 2^16*remainder_e(x,y)[1] + 2^32*remainder_e(x,y)[2] + 2^48*remainder_e(x,y)[3]
        let bound(x,y)     = bound_e(x,y)[0]     + 2^16*bound_e(x,y)[1]     + 2^32*bound_e(x,y)[2]     + 2^48*bound_e(x,y)[3]
        let rotated(x,y)   = dense_rot_e(x,y)[0] + 2^16*dense_rot_e(x)[1] + 2^32*dense_rot_e(x,y)[2] + 2^48*dense_rot_e(x,y)[3]
        constrain( word(x,y) * 2^(OFF(x,y)) - ( quotient(x) * 2^64 + remainder(x)) )
        constrain( rotated(x,y) - (quotient(x,y) + remainder(x)) )
        constrain( bound(x,y) - (quotient(x,y) + 2^64 - 2^(OFF(x,y)) )
        for i in [0...4)
            constrain( state_e(x,y)[i] - (reset0_e(x,y)[i] + 2*reset1_e(x,y)[i] + 4*reset2_e(x,y)[i] + 8*reset3_e(x,y)[i] ) )
            constrain( state_b(y,2x+3y)[i] - expand_rot_e(x,y)[i] )            
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


| Columns: | [1540...1940) | [1940...2340) | [2340...2344) |
| -------- | ------------- | ------------- | ------------- | 
| Chi      | reset_b       | reset_sum     | f_0_0     |


This is constrained with the following $300$ constraints and $800$ lookups

```rust
for i in [0..4)
    for x in [0..5)
        for y in [0..5)
            let not(x,y)[i] = 0x1111111111111111 - reset0_b(x+1,y)[i]
            let sum(x,y)[i] = not(x,y)[i] + reset1_b(x+2,y)[i]
            constrain( state_b(x,y)[i] - (reset0_b(x,y)[i] + 2*reset1_b(x,y)[i] + 4*reset2_b(x,y)[i] + 8*reset3_b(x,y)[i] ) )
            constrain( sum(x,y)[i] - (reset0_sum(x,y)[i] + 2*reset1_sum(x,y)[i] + 4*reset2_sum(x,y)[i] + 8*reset3_sum(x,y)[i] ) )
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

| Columns: | [0...4) | [4...100) |
| -------- | ------- | --------- | 
| Iota     | g_0_0   | state_f   |

After this last step of the permutation function, `Keccak` will store state `G` in the first $100$ cells of the next row, to be chained with the upcoming `StateXOR`. This requires $100$ copy constraints. Recall that except for `g_0_0`, the rest of `G` is `state_f`.


### Lookups

The design uses a 2-column lookup table called `Reset`` containing the elements from $0$ to $2^{16}-1$, and their expansion. As such, it can be used to check that the right expansion was used for a given chunk of 16 bits. 

Additionally, we create two more tables which correspond to the first and second columns of the former (called `Bytes16` and `Sparse` respectively). `Bytes16` can be used to range check some terms in rotation, whereas `Sparse` can be used to check the correct form of the shifts (since the non-sparse pre-image is non-relevant for $shift_i$ for $i\in[1,3]$). 

<center>

| row | expansion of each 16-bit input                                    |
| --- | ----------------------------------------------------------------- |
| $0$ |`0000000000000000000000000000000000000000000000000000000000000000` |
|     | ...                                                               |
| $i$ |`000` $b_{15}$ `000` $b_{14}$ `000` $b_{13}$ `000` $b_{12}$ `000` $b_{11}$ `000` $b_{10}$ `000` $b_{9}$ `000` $b_{8}$ `000` $b_{7}$ `000` $b_{6}$ `000` $b_{5}$ `000` $b_{4}$ `000` $b_{3}$ `000` $b_{2}$ `000` $b_{1}$ `000` $b_{0}$ |
|     | ...                                                               |
| $2^{16}-1$ | `0001000100010001000100010001000100010001000100010001000100010001` |

</center>

The `KeccakRound` gate performs $1,760$ lookups as indicated below (parenthesis means that they are part of another lookup and should not be counted twice):

| Columns | [120...140) | [140...200) | [200...220) | [220...240) | [240...260) | [260...280) | [280...300) | [300...320)
| ------- | --------- | ----------- | - | - | - | - | - | - |
| `Curr`  | 20`Reset` | 60`Sparse` | (20`Reset`) | 20`Bytes16` | 20`Bytes16` | 20`Bytes16` | 20`Reset` | (20`Reset`) |
| Theta  | reset0_c | reseti_c | dense_c | quotient_c | remainder_c | bound_c | dense_rot_c | expand_rot_c | 

| Columns | [440...540) | [540...840) | [840...940) | [940...1040) | [1040...1140) | [1140...1240) | [1240...1340) | [1340...1440) |
| -------- | ----------- | ----------- | ------------ | ------------- | ------------- | ------------- | ----- | ---- | 
| `Curr` | 100`Reset` | 300`Sparse` | (100`Reset`) | 100`Bytes16` | 100`Bytes16` | 100`Bytes16` | 100`Reset` | (100`Reset`) |
| PiRho    | reset0_e | reseti_e   | dense_e     | quotient_e   | remainder_e   | bound_e       | dense_rot_e   | expand_rot_e |

| Columns: | [1540...1940) | [1940...2340) | 
| -------- | ------------- | ------------- |
| `Curr`   | 400`Sparse`   | 400`Sparse`   |
| Chi      | reset_b       | reset_sum     |

The design makes use of a smaller table containing all bytes (values up to 8 bits), or reuses `Bytes16` or `Bytes12` with a scaling factor.

<center>

| row |
| --- |
| $0$ |
| ... |
| $255$ |

</center>

The `KeccakSponge` gate performs 200*2 + 100 + 300 lookups:

| `KeccakSponge` | [200...400) | [400...500)  | [500...800)      |
| -------------- | ---------------- | ------------- | ------------------ | 
| Curr           | 200`Bytes` + (100`Reset`) | 100`Reset` | 300`Sparse` |

The 100 lookups to `Reset` are performed using a linear combination of the bytes to obtain the corresponding 16 dense bits for each `shift0` term. Take into account that bytes shall be laid out in big-endian format, whereas each dense pair is in little-endian.

$$shift_0[i] = expand(\ bytes[2\cdot i] + 2^8 \cdot bytes[2\cdot i+1]\ )$$

### Performance

Counting the costs of the steps presented above, the following table summarizes the performance of the proposed design, for each of the 24 rounds of the permutation function, in each design of the Keccak gadget.

<center>

| Version  | Columns | Rows / block | Lookups / block | 
|----------|---------|--------------|-----------------|
| This RFC | 2344    | $1+24+1(+1)$ | $24\times(180+800+800)+800=43,520$ |    

</center>

### Input / Output 

We need to provide some mechanism to link blocks of the message to be hashed with the location in memory / public input, without relying on the permutation argument. Based on the [`syscalls` RFC](https://github.com/o1-labs/rfcs/pull/30), we want to have a Keccak communication channel between components of the system, where the syscalls could do:

`lookup_aggreg += 1/(x + keccak_channel_table_id + value * joint_combiner^1)`

and then the Keccak chip handles reading them with

`lookup_aggreg -= 1/(x + keccak_channel_table_id + value * joint_combiner^1)`

so that the final lookup aggregation is unchanged iff the Keccak chip processed and proved all of the messages.


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

This gate uses 3 lookup tables of length $2^{16}$. Even if other parts of the system need tables this large, it comes with a cost.

## Rationale and alternatives

Other configurations have been considered for the new Keccak gadgets, using the same theory underneath. Nonetheless, these approaches did not take that much advantage of the generalized expression framework. Instead of one single gate per round, one could take an alternative layout with a maximum column width of $14$, up to $8$ permutable cells, up to $12$ lookups per row, with $4$ custom gate types (`Xor64`, `Reset64`, `Rot64`, `Not64`), and a lookup table of $2^{16}$ entries. Here, the number of total lookups would not change, but the number of rows and copy constraints would be much higher. 

| Version  | Columns | Rows / block | Lookups / block | 
|----------|---------|--------------|-----------------|
| Alternative | 14   | $24\times(50+25+20+25+75+150+1)=8,304$ | $24\times(180+800+800)=42,720$ |      

These gates would provide some level of chainability between outputs of current row and inputs for next row. In particular, the layout of the alternative gates would be the following:

| Gate    | `0*`  | `1*`  | `2*`  | `3*`  | `4*`   | `5*`   | `6*`   | `7*`   |
| ------- | ----- | ----- | ----- | ----- | ------ | ------ | ------ | ------ |
| `Xor64` | left0 | left1 | left2 | left3 | right0 | right1 | right2 | right3 | 
| `Zero`  | xor0  | xor1  | xor2  | xor3  |

| Gate      | `0*!`    | `1*!`    | `2*!`    | `3*! `   | `4*!!`    | `5*!!`  | `6*` | `7!!`    | `8!!`    | `9!!`    | `10!!`   | `11`   | `12`   |
| --------- | -------- | -------- | -------- | -------- | -------- | -------- | ---- | -------- | -------- | -------- | -------- | ------ | ------ | 
| `Reset64` | sparse0  | sparse1  | sparse2  | sparse3  | reset1_0 | reset1_1 | word | reset2_0 | reset2_1 | reset3_0 | reset3_1 | dense0 | dense1 | 
| `Zero`    | reset0_0 | reset0_1 | reset0_2 | reset0_3 | reset1_2 | reset1_3 |      | reset2_2 | reset2_3 | reset3_2 | reset3_3 | dense2 | dense3 |

| `Not64` | `0*` | `1*` | `2*` | `3*` | `4*` | `5*` | `6*` | `7*` |
| ------- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| `Curr`  | x0   | x1   | x2   | x3   | not0 | not1 | not2 | not3 | 

| `Rot64` | `0*`   | `1*`   | `2!` | `3!` | `4!` | `5!` | `6!` | `7!` | `8!` | `9!` | `10!` | `11!` | `12!` | `13!` | 
| ------- | ------ | ------ | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ----- | ----- | ----- | ----- |
| `Curr`  | input  | output | quo0 | quo1 | quo2 | quo3 | rem0 | rem1 | rem2 | rem3 | bnd0  | bnd1  | bnd2  | bnd3  |

with one coefficient set to `2^{offset}`.

## Prior art

The current Keccak PoC in SnarkyML was introduced to support Ethereum primitives in MINA zkApps. Due to this blockchain's design, the gadget needed to be compatible with Kimchi: a Plonk-like SNARK instantiated with Pasta curves for Pickles recursion and IPA commitents. This means that the design choices for that gadget were determined by some features of this proof system, such as: 15-column width witness, 7 permutable witness cells per row, access to the current and next rows, up to 4 lookups per row, access to 4-bit XOR lookup table, and less than $2^{16}$ rows. As a result, proving the Keccak hash of a message of 1 block length (up to 1080 bits) took ~15k rows.

| Version  | Columns | Rows / block | Lookups / block | 
|----------|---------|--------------|-----------------|
| Old PoC  | 15      | $24\times(125+100+40+125+75+287.5+5)=18,180$ |$24\times(400+320+140+400+300+800+16)=57,024$ |   

## Unresolved questions

* During the review of this RFC:
    * Resolve how to deal with input and output.

* During the implementation of this RFC:
    * obtain exact measurements of the number of rows, columns, constraints, lookups, seconds, required per block hash;
    * if the endianness of the target input format is little endian, it will need to be transformed into big endian;
    * pack the bytestring given in chunks of 4 bytes.

* Future work: 
    * support SHA3 (NIST variant of Keccak), different output lengths, and different state widths;
    * improve the lookup argument to reuse the `Reset` table also for single-column lookups. Perhaps this can work adding some logic such as performing lookups modulo a constant ($2^{64}$ in this case), or shifting right by a given number of positions ($64$ in this case);
    * optimize the `KeccakRound` to remove redundant intermediate states (some preliminar ideas have been performed);
    * optimize range checks or think of a different approach towards rotation.


## Appendix

### Keccak algorithm

The high level idea of this sponge-based algorithm iteratively is to apply a permutation function to an initial state for a number of rounds (absorb phase) and to finally crop the state, when needed, obtaining the digest (squeeze) as follows:

<center>

![](https://upload.wikimedia.org/wikipedia/commons/7/70/SpongeConstruction.svg)

</center>

The following represents the description of the Keccak algorithm tailored to our use case configuration presented above.

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

>  Note that positions in X and Y coordinates must be computed modulo 5. Make sure that the language performs the remainder operation instead of modulo. Otherwise, make sure to add `+5` in subtractions to avoid negative indices.

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

| `r` |                 `RC` |        `expand3(RC)` |        `expand2(RC)` |        `expand1(RC)` |        `expand0(RC)` |
| --- | -------------------- | -------------------- | -------------------- | -------------------- | -------------------- |
|  0  | `0x0000000000000001` | `0x0000000000000000` | `0x0000000000000000` | `0x0000000000000000` | `0x0000000000000001` |
|  1  | `0x0000000000008082` | `0x0000000000000000` | `0x0000000000000000` | `0x0000000000000000` | `0x1000000010000010` |
|  2  | `0x800000000000808A` | `0x1000000000000000` | `0x0000000000000000` | `0x0000000000000000` | `0x1000000010001010` |
|  3  | `0x8000000080008000` | `0x1000000000000000` | `0x0000000000000000` | `0x1000000000000000` | `0x1000000000000000` |
|  4  | `0x000000000000808B` | `0x0000000000000000` | `0x0000000000000000` | `0x0000000000000000` | `0x1000000010001011` |
|  5  | `0x0000000080000001` | `0x0000000000000000` | `0x0000000000000000` | `0x1000000000000000` | `0x0000000000000001` |
|  6  | `0x8000000080008081` | `0x1000000000000000` | `0x0000000000000000` | `0x1000000000000000` | `0x1000000010000001` |
|  7  | `0x8000000000008009` | `0x1000000000000000` | `0x0000000000000000` | `0x0000000000000000` | `0x1000000000001001` |
|  8  | `0x000000000000008A` | `0x0000000000000000` | `0x0000000000000000` | `0x0000000000000000` | `0x0000000010001010` |
|  9  | `0x0000000000000088` | `0x0000000000000000` | `0x0000000000000000` | `0x0000000000000000` | `0x0000000010001000` |
| 10  | `0x0000000080008009` | `0x0000000000000000` | `0x0000000000000000` | `0x1000000000000000` | `0x1000000000001001` |
| 11  | `0x000000008000000A` | `0x0000000000000000` | `0x0000000000000000` | `0x1000000000000000` | `0x0000000000001010` |
| 12  | `0x000000008000808B` | `0x0000000000000000` | `0x0000000000000000` | `0x1000000000000000` | `0x1000000010001011` |
| 13  | `0x800000000000008B` | `0x1000000000000000` | `0x0000000000000000` | `0x0000000000000000` | `0x0000000010001011` |
| 14  | `0x8000000000008089` | `0x1000000000000000` | `0x0000000000000000` | `0x0000000000000000` | `0x1000000010001001` |
| 15  | `0x8000000000008003` | `0x1000000000000000` | `0x0000000000000000` | `0x0000000000000000` | `0x1000000000000011` |
| 16  | `0x8000000000008002` | `0x1000000000000000` | `0x0000000000000000` | `0x0000000000000000` | `0x1000000000000010` |
| 17  | `0x8000000000000080` | `0x1000000000000000` | `0x0000000000000000` | `0x0000000000000000` | `0x0000000010000000` |
| 18  | `0x000000000000800A` | `0x0000000000000000` | `0x0000000000000000` | `0x0000000000000000` | `0x1000000000001010` |
| 19  | `0x800000008000000A` | `0x1000000000000000` | `0x0000000000000000` | `0x1000000000000000` | `0x0000000000001010` |
| 20  | `0x8000000080008081` | `0x1000000000000000` | `0x0000000000000000` | `0x1000000000000000` | `0x1000000010000001` |
| 21  | `0x8000000000008080` | `0x1000000000000000` | `0x0000000000000000` | `0x0000000000000000` | `0x1000000010000000` |
| 22  | `0x0000000080000001` | `0x0000000000000000` | `0x0000000000000000` | `0x1000000000000000` | `0x0000000000000001` |
| 23  | `0x8000000080008008` | `0x1000000000000000` | `0x0000000000000000` | `0x1000000000000000` | `0x1000000000001000` |

</center>

> Note that the hexadecimal expanded representation of each chunk of the `RC` corresponds to the binary representation of those 4 bits.