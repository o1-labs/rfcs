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

### Background

This subsection covers a thorough description of the Keccak algorithm and specifies the required parameters needed for compatibility in our usecase. The reader shall skip this section to find the details of the actual proposal in this RFC. 

The configuration of Keccak must be the same as that of the Ethereum EVM, meaning specifically:

> * [pre-NIST variant of Keccak](https://keccak.team/files/Keccak-reference-3.0.pdf) (uses the $10^*1$ padding rule)
> * variable input **byte** length (not any bit length)
> * output length of 256 bits
> * input and output in big endian
> * state composed of 25 (5x5) 64-bit words
> * making a block length of 1600 bits ($b$)
> * 512 bits capacity ($c$)
> * 1088 length bit rate ($r := b - c$)
> * 24 rounds of permutation

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

### Bitwise-sparse representation

Here, we present the bitwise-sparse representation of bitstrings that will be used to obtain more efficient computations of boolean functions within our proof system. 

The goal is to represent these operations in finite field algebra more efficiently than they were in the [Keccak PoC](https://www.notion.so/minaprotocol/Keccak-gadget-PoC-PRD-59b024bce9d5441c8a00a0fcc9b356a), for which we took a naive approach where small values occupied full field elements; wasting a lot of empty space. For instance, each row could only xor 16 bits at a time, whereas the field could fit 255 bits. 

Given that simple binary operations are very costly within a SNARK, the new idea consists on arithmetizing boolean operations as much as possible, becoming more efficient with finite field algebra. We introduce the _bitwise-sparse representation_ of bitstrings for this goal.

With this notation, input bitstrings are expanded inserting three `0` bits in between "real" bits (hence the _sparsity_). The motivation for this, is to use intermediate bits to store auxiliary information such as carry bits, resulting from algebraic computations of boolean functions. That means, for each bit in the original bitstring, its expansion will contain four bits. The following expression represents such mapping:

$$\forall X \in \{b_i\}^\ell, b\in\{0,1\}: expand(X) \to 0 \ 0 \ 0 \ b_{\ell-1} \ . \ . \ . \ 0 \ 0 \ 0 \ b_0$$

Even though in each field element of ~254 bits long, one could fit up to 63 real bits with this encoding, each witness value will only store 16 such real bits. This is because the way to perform this mapping is through a 16-bit lookup table (of $2^{16}$ entries, larger than that would be too costly). Then, the expansion of the full 64 bit words could be represented by composition of each of the four 16-bit parts. But this will not be computed in the circuit, since the expansion would not fit in the field.

Let $sparse(X)$ refer to a representation of $X$, where only the indices in the $4i$-th positions correspond to real bits, and the intermediate indices can contain auxiliary information. Concretely,

$$\forall X\in\{b_i\}^\ell:\quad X \equiv \sum_{0}^{\ell-1} 2^i \cdot sparse(X)_{4i} $$

Another set of functions will be defined to be used in this new design. For $i\in\{0,3\}$, let $shift_i(X)$ be the function that performs the XOR operation of the sparse representation of the word $2^{\ell}-1$ ("all-ones" word in binary), shifted $i$ bits to the left, with the input word $X$. More formally,

$$\forall X \in \{b_i\}^\ell, i\in\{0,3\}: shift_i(X) := \left(\ expand(2^\ell - 1)<<_i 0\ \right)\ \oplus \ X$$ 

meaning, on input $expand(X)$, all four of:

$$ 
\begin{align*}
shift_0\left(expand(X)\right) := 0 \ 0 \ 0 \ 1_{\ell-1} \ .\ .\ .\ 0 \ 0 \ 0 \ 1_{0} \ \oplus\ 0 \ 0 \ 0 \ b_{\ell-1} \ . \ . \ . \ 0 \ 0 \ 0 \ b_0 \\
shift_1\left(expand(X)\right) := 0 \ 0 \ 1_{\ell-1} \ 0 \ . \ . \ . \ 0 \ 0 \ 1_{0} \ 0 \ \oplus\ 0 \ 0 \ 0 \ b_{\ell-1} \ . \ . \ . \ 0 \ 0 \ 0 \ b_0 \\
shift_2\left(expand(X)\right) := 0 \ 1_{\ell-1} \ 0 \ 0 \ . \ . \ . \ 0 \ 1_{0} \ 0 \ 0 \ \oplus\ 0 \ 0 \ 0 \ b_{\ell-1} \ . \ . \ . \ 0 \ 0 \ 0 \ b_0 \\
shift_3\left(expand(X)\right) := \ 1_{\ell-1} \ 0 \ 0 \ 0 \ . \ . \ . \ 1_{0} \ 0 \ 0 \ 0 \oplus\ 0 \ 0 \ 0 \ b_{\ell-1} \ . \ . \ . \ 0 \ 0 \ 0 \ b_0 \\
\end{align*}
$$

The effect of each $shift_i$ is to negate the $(4n+i)$-th bits of the expanded output, while leaving the rest of bits unchanged. That implies that one can rewrite the input as:

$$X \iff shift_0(X) + 2 \cdot shift_1(X) + 2^2 \cdot shift_2(X) + 2^3\cdot shift_3(X) ???$$

<details>
<summary>TODO</summary>
The above is wrong. Ask Matthew to explain it differently. Why are shifts needed in the first place and we don't just use XOR as explained below?
</details>

> Checking the correct form of the shifts should be done with a single-column lookup table with just the expanded values for the check, since the non-sparse pre-image is non-relevant here.

Given that the new field size is 254 bits, the whole 64-bit word expansion cannot fit in one single witness element. Instead, they will be split into quarters of 64 bits each (corresponding to real 16 bits each), following this notation:

$$\forall i \in \{0, 3\}:\quad split_i(sparse(X)) := sparse(X_{16i}^{16i+15})$$

Meaning that $\forall X\in(0,2^{64})$:

$$ 
\begin{align*}
sparse(X) \equiv &split_0(sparse(X)) + 2^{64} \cdot split_1(sparse(X)) + \\
& 2^{128}\cdot split_2 (sparse(X)) + 2^{192} \cdot split_3(sparse(X))
\end{align*}
$$

#### XOR

In order to perform the XOR operation, we rely on the fact that the exclusive OR of two bits is equivalent to their addition, modulo 2. In particular, when adding the boolean values `1+1=10`, the bit in the left position corresponds to the carry term. For clarity:

<center>

| a | b | xor | add |
| - | - | --- | --- |
| 0 | 0 |  0  |  0  |
| 0 | 1 |  1  |  1  |
| 1 | 0 |  1  |  1  |
| 1 | 1 |  0  |  2  |

</center>

Altogether, we will represent the XOR of 16-bit values using addition of their expansions. The 16-bit output of the XOR will be located on the $4n$ positions, whereas the intermediate positions will contain any possible carry terms.

The reason behind the choice of three empty intermediate bits in the sparse representation follows from the concrete usecase of Keccak where no more than $2^4-1$ consecutive boolean operations are performed, and thus carries cannot overwrite the content of the bits to the leftmost bits. In particular, each round of the permutation function concatenates no more than 12 consecutive boolean operations. 

> NOTE: the above means that after each round of the permutation, the expanded representation needs to be contracted, discard auxiliary bits, and expand again from scratch before starting the next round to ensure completeness of the encoding.

#### AND

In the current [Keccak PoC](https://www.notion.so/minaprotocol/Keccak-gadget-PoC-PRD-59b024bce9d5441c8a00a0fcc9b356ae), AND was performed taking advantage of the following equivalence (that was implicitly used above as well):

$$ a + b = XOR(a, b) + 2\cdot AND(a, b)$$

That means, adding two bits is equivalent to their exclusive OR, plus the carry term in the case both are $1$. This equation can be generalized for arbitrary length bitstrings to constraint the AND operation for 64-bit values, using generic operations (addition, multiplication by constant) and the above representation of XORs.

#### Negation

The current [Keccak PoC](https://www.notion.so/minaprotocol/Keccak-gadget-PoC-PRD-59b024bce9d5441c8a00a0fcc9b356ae) gave support for two different mechanisms to prove negation of 64-bit words: one of them uses the `Generic` gate to perform a subtraction with the value $2^{64}-1$, whereas the other uses XOR with the $2^{64}-1$ word. The former was more efficient, as it could fit up to two full negations per row. The latter however, required 5 rows to perform one single NOT operation. The advantage of the latter though, was that the XOR approach implicitly checked that the input was at most 64 bits in length, due to decomposition rules. This meant, that the `Generic` approach could only be used together with other mechanisms to assert this. Altogether, using the efficient approach was safe in the Keccak usecase, because the input to the negation was wired to the output of another `xor64` gadget, which used `Xor16` inside. 

In this new approach, making sure that the input word is $<2^{64}$ is also important. Here, the NOT operation will be performed with a subtraction operation using the sparse representation of the constant term $2^{64}$. In this case, the mechanism to check that the input is at most 64 bits long, will rely on the composition of the sparse representation. Meaning, 64-bit words produce four 16-bit lookups, which implies that the real bits take no more than 64 bits.

$$sparse(NOT_{64}(X)) := expand(2^{64}-1) - expand(X)$$

<details>
<summary>TODO</summary>
Check carefully the mechanism to check it is 64-bits. When will they take place? How many of them are needed? Detail better what lookups happen.
</details>

#### Rotation

<details>
<summary>TODO</summary>
Need to convert before?
</details>

> It is equivalent to do any rotation by x as a rotation by 4*x on the sparse-bit representation, so we can avoid unpacking altogether, which buys us some additional efficiency gains.

### Custom gate

The first step is to expand all 25 words of the initial state. Each word of 64 bits will be split into 4 parts of 16 real bits each. The expansion itself will be performed through a lookup table containing all $2^{16}$ entries. This step will require $4\times25$ lookups.

> For efficiency reasons, the state is assumed to be stored in expanded form, so that back-and-forth conversions do not need to take place repeatedly.

Support for the Keccak hash function will be obtained using one single custom gate of 1 row, using 1108 columns and 1408 lookups. The layout of the witness in the gate is determined by the algorithms inside the permutation function itself, as we unleash below. 

#### Step theta

For each row `x in [0..5)` in the state `A`, perform

$$
\begin{align*}
C[x] := &\ A[x][0]\ \oplus\ A[x][1]\ \oplus\ A[x][2]\ \oplus\ A[x][3]\ \oplus\ A[x][4] \\
\iff \\
sparse(C[x]) := &\ expand(A[x][0]) + expand(A[x][1]) + expand(A[x][2]) \\
+\ & expand(A[x][3]) + expand(A[x][4]) \\
\end{align*} 
$$

From the initial step, the expanded version of the 25 words in state `A` is already in the witness. Since the new field size cannot hold full 256-bit expansion, the above will be performed in 4 parts of 64-bits each (corresponding to 16 bits of the input), computing all four $split_i(sparse(C[x]))$ instead.

Similarly, for each row `x in [0..5)`, perform the following operation splitting into quarters

$$
\begin{align*}
D[x] := &\ C[x-1]\ \oplus\ ROT(C[x+1],1) \\
\iff \\
sparse(D[x]) := &\ sparse(C[x-1]) + ROT(sparse(C[x+1]), 1)\\
\end{align*} 
$$

Next, for each row `x in [0..5)` and column `y in [0..5)`, compute

$$
\begin{align*}
E[x][y] := &\ A[x][y]\ \oplus\ D[x] \\
\iff \\
sparse(E[x][y]) := &\ sparse(A[x][y]) + sparse(D[x])\\
\end{align*} 
$$

#### Step pi-rho

For each row `x in [0..5)` and column `y in [0..5)`, perform (into quarters):

$$
\begin{align*}
B[y][2x+3y] := &\ ROT(E[x][y], OFF[x][y]) \\
\iff \\
sparse(B[y][2x+3y]) := &\ ROT(sparse(E[x][y]), OFF[x][y])
\end{align*} 
$$

<details>
<summary>TODO</summary>
How to do this without transforming the format back into 64 bits
</details>

#### Step chi

For each row `x in [0..5)` and column `y in [0..5)`, perform (into quarters):

$$
\begin{align*}
F[x][y] := &\ B[x][y] \oplus (\ \neg B[x+1][y] \wedge \ B[x+2][y]) \\
\iff \\
sparse(F[x][y]) := &\ 
\end{align*} 
$$

#### Step iota

On round $r$, update the word in the first row, first column xoring with the round constant as

$$
G[0][0] \oplus RC[r] \iff sparse(G[0][0]) + expand(RC[r])
$$

### Performance

Counting the costs of the steps presented above, the following table summarizes the performance of the proposed design, for each of the 24 rounds of the permutation function:

| step | xors | rots | misc | columns | lookups | 
| ---- | ---- | ---- | ---- | ------- | ------- |
| setup |     |      | $expand$ | | $4\times5\times5$
| theta C | $4\times4\times5$
| theta D | $4\times5\times1$ | $5\times1$ | transform? 
| theta E | $4\times5\times5\times1$ |
| pi-rho | $0$ | $5\times5\times1$ | transform? |
| chi |  |  
| iota | $4 \times 1$ | | | | |

```
xor64
=====
8 * bit expand
8 * bit split
8 * bit contract
= 24 rows, 16 lookups
25 * xor
5 * expanded bitshift
5 * expanded xor (
25 * rot
Entire state can be expanded, saves rows
Expanded xor64
============
8 * split
8 * expanded lookups
= 8 columns, 8 lookups
Rot64
============
2 temp
12 lookup
25 * xor
5 * rot
5 * xor
25 * xor
25 * rot
25 * (xor + and + xor)
1 * xor
= 131 * xor + 30 * rot
= 131 * 8 + 30 * 2 columns, 131 * 8 + 30 * 12 lookup
= 1108 columns, 1408 lookups
Can expand more to fit more xors
x_1 xor ... xor x_5 takes an expansion into 2^8
16 columns, 16 lookups
x_1 xor ((not x_2) and x_3) is
(expand (2^64-1) - x_2) + x_3 + 2*x_1
12 columns, 12 lookups
```


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

Why should we *not* do this?

## Rationale and alternatives

The generalized expression framework will provide the ability to create arbitrary number of columns and lookups per row for custom gates. This allows more optimized approaches to address boolean SNARK-unfriendly relations (among other advantages) such as XORs. 

* Why is this design the best in the space of possible designs?
* What other designs have been considered and what is the rationale for not choosing them?
* What is the impact of not doing this?

## Prior art

The current [Keccak PoC in SnarkyML](https://www.notion.so/minaprotocol/Keccak-gadget-PoC-PRD-59b024bce9d5441c8a00a0fcc9b356ae) was introduced to support Ethereum primitives in MINA zkApps. Due to this blockchain's design, the gadget needed to be compatible with Kimchi: a Plonk-like SNARK instantiated with Pasta curves for Pickles recursion and IPA commitents. This means that the design choices for that gadget were determined by some features of this proof system, such as: 15-column width witness, 7 permutable witness cells per row, access to the current and next rows, up to 4 lookups per row, access to 4-bit XOR lookup table, and less than $2^{16}$ rows.

With these constraints in mind, the Keccak PoC extensively used a chainable custom gate called `Xor16`, which performs one 16-bit XOR per row. In particular, xoring 64-bit words (as used in Ethereum), required 4 such gates followed by 1 final `Zero` row. As a result, proving the Keccak hash of a message of 1 block length (up to 1080 bits) took ~15k rows.

## Unresolved questions

* What is the endianness of the target input format? If it is little endian, it will need to be transformed into big endian.

* Will the input be given as a bitstring or will it be packed into bytes?

* Find out if the round constants should be hardcoded (takes memory space) or generated (takes computation resources).

* Whether negation could accelerate the overflow of the auxiliary bits.

* Find out if the rotation offsets will be directly stored modulo 64 or not.

* Clarify the use/need of $shift$

* Confirm if we really want one single row custom gate

* Understand if rotation can be done without translation

* During the implementation of this RFC, obtain exact measurements of the number of rows, columns, constraints, lookups, seconds, required per block hash.

* Future work: support SHA3 (NIST variant of Keccak), different output lengths, and different state widths. 


## Relevant links

* [Pros/Cons of MIPS VM](https://www.notion.so/minaprotocol/MIPS-VM-68c09743b79e4e2bab9e6952c27f0ab4)
* [RFP: OP Stack Zero Knowledge Proof](https://github.com/ethereum-optimism/ecosystem-contributions/issues/61#issuecomment-1611488039)
* [PRD: Optimized keccak256 hashing (for MIPS VM)](https://www.notion.so/minaprotocol/PRD-Optimized-keccak256-hashing-for-MIPS-VM-33c142f38467464a9b7c21184544dd1d)
* [O(1) Labs'  H2 2023 OKRs](https://www.notion.so/minaprotocol/8dfc798e277447ea860e747b026d9698?v=5daef538906d44db9d12e2e0fbbd4665)
* [SnarkyML Keccak PoC](https://github.com/MinaProtocol/mina/pull/13196)
* [Keccak gadget PoC PRD for Ethereum Primitives](https://www.notion.so/minaprotocol/Keccak-gadget-PoC-PRD-59b024bce9d5441c8a00a0fcc9b356ae)
* [The Keccak reference document](https://keccak.team/files/Keccak-reference-3.0.pdf)