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

The goal is to represent these operations in finite field algebra more efficiently than they were in the [Keccak PoC](https://www.notion.so/minaprotocol/Keccak-gadget-PoC-PRD-59b024bce9d5441c8a00a0fcc9b356a), for which we took a naive approach where small values occupied full field elements; wasting a lot of empty space. For instance, each row could only xor 16 bits at a time, whereas the field could fit 255 bits. 

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


### Basic gadgets

> For efficiency reasons, the state is assumed to be stored in expanded form, so that back-and-forth conversions do not need to take place repeatedly.

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

Given expanded `left` and `right` inputs, the sparse representation of $XOR_{64}$ can be constrained in quarters of 16 bits, as:

```rust
for i in [0..4) {
    // Load witness
    let left[i]   // each quarter of expanded left input 
    let right[i]  // each quarter of expanded left input 
    let xor[i]    // sparse(xor(left,right))

    constrain(0 == xor[i] - (expand_left[i] + expand_right[i]))
}
```

The following table presents a candidate layout for a `Xor64` gate using 12 columns, where all of them are permutable (marked as `*`), assuming each input (`left`, `right`) and output (`xor`) is given in sparse representation.

<center>

| `Xor64` | `0*`  | `1*`  | `2*`  | `3*`  | `4*`   | `5*`   | `6*`   | `7*`   | `8*` | `9*` | `10*` | `11*` |
| ------- | ----- | ----- | ----- | ----- | ------ | ------ | ------ | ------ | ---- | ---- | ----- | ----- |
| `Curr`  | left0 | left1 | left2 | left3 | right0 | right1 | right2 | right3 | xor0 | xor1 | xor2  | xor3  | 

</center>

An alternative using just 6 permutable columns would instead use 2 rows instead.

<center>

| Gate    | `0*`  | `1*`  | `2*`   | `3*`   | `4*` | `5*` |
| ------- | ----- | ----- | ------ | ------ | ---- | ---- |
| `Xor64` | left0 | left1 | right0 | right1 | xor0 | xor1 | 
| `Zero`  | left2 | left3 | right2 | right3 | xor2 | xor3 |

</center>

#### AND

In the current [Keccak PoC](https://www.notion.so/minaprotocol/Keccak-gadget-PoC-PRD-59b024bce9d5441c8a00a0fcc9b356ae), AND was performed taking advantage of the following equivalence:

$$ a + b = XOR(a, b) + 2\cdot AND(a, b)$$

That means, adding two bits is equivalent to their exclusive OR, plus the carry term in the case both are $1$. This equation can be generalized for arbitrary length bitstrings to constraint the AND operation for 64-bit values, using generic operations (addition, multiplication by constant) and the above representation of XORs.

The challenge here is to deal with the addition, which takes place on the non-sparse representation, while we have access to the sparse representation. For this, the gate will receive as input the non-sparse 16-bit quarters, as well as its expansion (`Reset64` will be used). 

Given $a, b$ in non-sparse representation, the $AND_{64}$ can be constrained in quarters of 16 bits using 4 lookups: 2 to expand $a,b$, another lookup to expand an inner value `computed_xor[i]`, and 1 final lookup to perform $shift_0$ to check the constraint is zero :

```rust
for i in [0..4) {

    // Load witness
    let dense_left[i] // i-th quarter of word A
    let b[i]          // i-th quarter of word B
    let claim_and[i]  // witness value claimed AND16(a[i], b[i])

    // Compute additional terms
    let computed_xor[i] = a[i] + b[i] - 2 * claim_and
    let sparse_xor[i] = xor16(a[i], b[i]) // 2 lookups

    // Constrain that ( ADD - 2 AND ) = XOR
    constrain( 0 == shift0( sparse_xor[i] - expand(computed_xor[i])) )

    // If constraints are satisfied, the claimed value contains the non-sparse output of AND
    return claim_and[i]
}
```

Altogether, this requires $3\times4$ columns and $4\times4$ lookups per AND of a 64-bit word. 

The following table presents a candidate layout for the `And64` functionality, with the 2-row `Xor64`:

<center>

| `Gates` | `0*`  | `1*`  | `2*`   | `3*`   | `4*` | `5*` | `6*` |
| ------- | ----- | ----- | ------ | ------ | ---- | ---- | ---- |
| `And64` | left0
| 

followed by

| `Gates` | `0*`  | `1*`  | `2*`   | `3*`   | `4*` | `5*` | `6*` |
| ------- | ----- | ----- | ------ | ------ | ---- | ---- | ---- |
| `Xor64`  | left0 | left1 | right0 | right1 | xor0 | xor1 |  
|          | left2 | left3 | right2 | right3 | xor2 | xor3 |
| 

</center>

#### Negation

The current [Keccak PoC](https://www.notion.so/minaprotocol/Keccak-gadget-PoC-PRD-59b024bce9d5441c8a00a0fcc9b356ae) gave support for two different mechanisms to prove negation of 64-bit words: one of them uses the `Generic` gate to perform a subtraction with the value $2^{64}-1$, whereas the other uses XOR with the $2^{64}-1$ word. The former was more efficient, as it could fit up to two full negations per row. The latter however, required 5 rows to perform one single NOT operation. The advantage of the latter though, was that the XOR approach implicitly checked that the input was at most 64 bits in length, due to decomposition rules. This meant, that the `Generic` approach could only be used together with other mechanisms to assert this. Altogether, using the efficient approach was safe in the Keccak usecase, because the input to the negation was wired to the output of another `xor64` gadget, which used `Xor16` inside. 

In this new approach, making sure that the input word is $<2^{64}$ is also important. Here, the NOT operation will be performed with a subtraction operation using the constant term $(2^{16}-1)$ for each 16-bit quarter, and only later it will be expanded. In this case, the mechanism to check that the input is at most 64 bits long, will rely on the composition of the sparse representation. Meaning, 64-bit words produce four 16-bit lookups, which implies that the real bits take no more than 64 bits.

Given $x$ in expanded form (meaning, not any sparse representation of $x$ but the initial one with zero intermediate bits) the $NOT_{64}(x)$ can be computed in quarters of 16 bits each (`x[0], x[1], x[2], x[3]`) as:

```rust
for i in [0..4) {
    let x[i]     // expanded representation of 16-bit input
    let not16[i] // expanded representation of negated
    
    // expand(2^16-1) = 0001...0001 = 
    constrain(0 == not16[i] - (0x1111111111111111 - x[i]) ) 
}
```

> Because the inputs are wired to cells which are known to be 16-bits at most, it is guaranteed that the total length of $X$ is at most 64 bits, and is correctly split into quarters `x[i]`.

The following tables present candidate layouts to perform `Not64`. The first one is a specialized gate with 8 permutable cells, whereas the second one reuses doble generic gates with 6 permutable cells with 2 rows in total.

<center>

| `Not64` | `0*` | `1*` | `2*` | `3*` | `4*` | `5*` | `6*` | `7*` |
| ------- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| `Curr`  | x0   | x1   | x2   | x3   | not0 | not1 | not2 | not3 | 

VS

| Gates     | `0*` | `1*` | `2*` | `3*` | `4*` | `5*` |
| --------- | ---- | ---- | ---- | ---- | ---- | ---- | 
| `Generic` | x0   |      | not0 | x1   |      | not1 |  
| `Generic` | x2   |      | not2 | x3   |      | not3 |

with coefficients list

left0 | right0 | output0 | mul0 | const0 | left1 | right1 | output1 | mul1 | const1 |  
| --------- | ---- | ---- | ---- | ---- | ---- | ---- | -- | -- | - |
| -1 | 0 | -1 | 0 | `0x1111111111111111` | -1 | 0 | -1 | 0 | `0x1111111111111111` |

</center>


#### Rotation

> If we could fit the whole 256 expanded bits of the 64-bit word, we could simulate rotations by $x$ bits as a rotation by $4*x$ on the sparse-bit representation, so we could avoid unpacking altogether, which would buy us some additional efficiency gains.

Rotations can be performed directly on the full 64-bit word. A similar algebraic approach as in the [Keccak PoC](https://www.notion.so/minaprotocol/Keccak-gadget-PoC-PRD-59b024bce9d5441c8a00a0fcc9b356ae) is taken here. 
This time, it will make use of an auxiliary function to compose four 16-bit quarters into full 64-bit terms as 

$$compose \to quarter_0 + 2^{16}quarter_1 + 2^{32}quarter_2 + 2^{48}quarter_3$$

```rust
fn rot64() {
    // Load witness
    let input        // 64-bit input word
    let output       // 64-bit output word
    let quo_quarters // 4-array with up to 16-bit parts of quotient
    let rem_quarters //  4-array with up to 16-bit parts of remainder
    let aux_quarters //  4-array with up to 16-bit parts of auxiliary bound

    // Load coefficients 
    let two_to_off // 2^offset

    // Compose quarters into full <=64 bit words
    let quotient = compose(quo_quarters)
    let remainder = compose(rem_quarters)
    let bound = compose(aux_quarters)

    // Constrain rotation
    constrain(0 == input * two_to_off - (quotient * 2^64 + remainder))
    constrain(0 == remainder + quotient - output)

    // Check quotient < 2^offset <=> quotient + 2^64 - 2^off < 2^64
    constrain(0 == bound - ( quotient + 2^64 - two_to_off ) )

    for i in [0..4) {
        lookup(rem_quarters[i]) // Check remainder < 2^64
        lookup(quo_quarters[i]) // Check quotient < 2^64
        lookup(aux_quarters[i]) // Check bound < 2^64
    }
}
```

<center>

| `Rot64` | `0*`   | `1!` | `2!` | `3!` | `4!` | `5!` | `6!` | 
| ------- | ------ | ---- | ---- | ---- | ---- | ---- | ---- | 
| `Curr`  | input  | quo0 | quo1 | quo2 | quo3 | aux0 | aux1 | 
| `Next`  | output | rem0 | rem1 | rem2 | rem3 | aux2 | aux3 |

with one coefficient set to `2^{offset}`.

</center>

This gate requires 6 lookups per row. Following the two rows of rotation, we need 2 more rows to decompose the output into its 4 expanded quarters.

#### Reset

After each step of Keccak, the sparse representation must be reset to avoid overflows of the intermediate bits. For this, each row of the `Reset64` gate $13$ columns, of which the first $5$ can be permutable, using 8 lookups. It also includes Cells involved in a lookup are marked as `!`.

```rust
let word          // 64-bit non-sparse word
constrain(0 == word - (dense[0] + 2^16 * dense[1] 
                      + 2^32 * dense[2] + 2^48 * dense[3])
for i in [0..4) {
    let sparse[i] // sparse representation of quarter i
    let dense[i]  // non-sparse ith 16 bits  -> lookup 1   
    let reset0[i] // shift0(sparse) = expand -> lookup 1
    let reset1[i] // shift1(sparse) / 2      -> lookup 2
    let reset2[i] // shift2(sparse) / 4      -> lookup 3
    let reset3[i] // shift3(sparse) / 8      -> lookup 4

    constrain(0 == sparse[i] - ( reset0[i] + 2 * reset1[i] 
                             + 4 * reset2[i] + 8 * reset3[i]) )
}
```

| Gate | `0*`  | `1*`  | `2*`  | `3*!`  | `4*!`   | `5*!`   | `6*!`   | `7!`   | `8!` | `9!` | `10!` | `11!` | `12!` |
| ------- | ----- | ----- | ----- | ----- | ------ | ------ | ------ | ------ | ---- | - | - | - | - |
| `Reset64`  | word | sparse0 | sparse1 | dense0 | dense1 | reset0_0 | reset0_1 | reset1_0 | reset1_1 | reset2_0 | reset2_1 | reset3_0 | reset3_1 |
| `Zero`  |  | sparse2 | sparse3 | dense2 | dense3 reset0_2 | reset0_3 | reset1_2 | reset1_3 | reset2_2 | reset2_3 | reset3_2 | reset3_3 |


The `word` witness will be wired to the rotation gate. Each `sparse[i]` will be the output of previous boolean gates, so they must be permutable. Each `dense[i]` will be used as inputs of the AND gate. The `reset0[i]` corresponds to the expand, and will be the input of upcoming boolean gates (beginning of steps).

### Keccak gadget

Support for padding shall be provided. In the Keccak PoC, this step takes place at the Snarky layer. It checks that the correct amount of bits in the $10*1$ rule are added until reaching a multiple of 1088 bits, and then adds 512 more zero bits to each block to form a full state.

If the input is not previously expanded, the next step is to expand all 25 words. Each word of 64 bits will be split into 4 parts of 16 real bits each. The expansion itself will be performed through the lookup table containing all $2^{16}$ entries. This step would require $4\times25=100$ lookups.

> The following subsections provide indications to implement each step of the permutation round. Recall that this will be executed 24 times in our configuration of Keccak.

> Note that wiring cells which contain the same variable is key for soundness of the constructions.

After each round, the new state is xored with the previous state. That requires 25 64-bit XORs. 

<center>

| Version  | Rows / round          | Lookups / round        |
|----------|-----------------------|------------------------|
| This RFC | $5\times5\times2=50$  | $0$                    |                
| Old PoC  | $5\times5\times5=125$ | $5\times5\times16=400$ |                   

</center>

#### Step theta

For each row `x in [0..5)` in the state `A`, compute the `C` state using 4 calls to the `Xor64` gate:

$$
\begin{align*}
C[x] := &\ A[x][0]\ \oplus\ A[x][1]\ \oplus\ A[x][2]\ \oplus\ A[x][3]\ \oplus\ A[x][4] \\
\iff \\
sparse(C[x]) := &\ expand(A[x][0]) + expand(A[x][1]) + expand(A[x][2]) \\
+\ & expand(A[x][3]) + expand(A[x][4]) \\
\end{align*} 
$$

<center>

| Gates / x=[0..5) | Inputs    | Output | Lookups | 
|------------------|-----------|--------|---------|
| `Xor64` + `Zero` | a0, a1    | a01    | 0       |              
| `Xor64` + `Zero` | a01, a2   | a012   | 0       |                   
| `Xor64` + `Zero` | a012, a3  | a0123  | 0       |
| `Xor64` + `Zero` | a0123, a4 | c      | 0       |

| Version  | Rows / round          | Lookups / round         | 
|----------|-----------------------|-------------------------|
| This RFC | $5\times4\times2=40$  | $0$                     |                
| Old PoC  | $5\times4\times5=100$ | $5\times4\times16=320$  |                   

</center>

Similarly, for each row `x in [0..5)`, perform the following operation splitting into quarters

$$
\begin{align*}
D[x] := &\ C[x-1]\ \oplus\ ROT(C[x+1],1) \\
\iff \\
sparse(D[x]) := &\ sparse(C[x-1]) + ROT(expand(C[x+1]), 1)\\
\end{align*} 
$$

For this, the 5 possible inputs of the rotation need to be reset. The input of the xor can however be the sparse or the reset version of `C[x-1]`.

<center>

| Gates / x=[0..5)   | Inputs      | Output  | Lookups | 
|--------------------|-------------|---------|---------|
| `Reset64` + `Zero` | sparse      | reset   | $16$    |
| `Rot64` + `Zero`   | reset       | rot     | $12$    |              
| `Xor64` + `Zero`   | c[x-1], rot | d       | $0$     |                   

| Version  | Rows/round        | Lookups/round                    | 
|----------|-------------------|----------------------------------|
| This RFC | $5\times6=30$     | $5\times28=140$                  |                
| Old PoC  | $5\times(5+3)=40$ | $5\times(4\times4+4\times3)=140$ |                   

</center>

Next, for each row `x in [0..5)` and column `y in [0..5)`, compute (with no need of resetting `D`):

$$
\begin{align*}
E[x][y] := &\ A[x][y]\ \oplus\ D[x] \\
\iff \\
sparse(E[x][y]) := &\ sparse(A[x][y]) + sparse(D[x])\\
\end{align*} 
$$

<center>

| Gates / x=[0..5) y=[0..5) | Inputs | Output  | Lookups | 
|---------------------------|--------|---------|---------|             
| `Xor64` + `Zero`          | a, d   | e       | $0$     |    

| Version  | Rows/round            | Lookups/round          | 
|----------|-----------------------|------------------------|
| This RFC | $5\times5\times2=50$  | $0$                    |                
| Old PoC  | $5\times5\times5=125$ | $5\times5\times16=400$ |      

</center>

#### Step pi-rho

For each row `x in [0..5)` and column `y in [0..5)`, perform (into quarters):

$$
\begin{align*}
B[y][2x+3y] := &\ ROT(E[x][y], OFF[x][y]) \\
\iff \\
expand(B[y][2x+3y]) := &\ ROT(expand(E[x][y]), OFF[x][y])
\end{align*} 
$$

Recall that in order to perform the rotation operation, the state needs to be reset.

<center>

| Gates / x=[0..5) y=[0..5) | Inputs | Output  | Lookups | 
|---------------------------|--------|---------|---------|             
| `Reset64` + `Zero`        |  e     | reset   | $16$    |    
| `Rot64` + `Zero`          | reset  | b       | $12$     |    

| Version  | Rows/round            | Lookups/round          | 
|----------|-----------------------|------------------------|
| This RFC | $5\times5\times4=100$ | $5\times5\times28=700$ |                
| Old PoC  | $5\times5\times3=75$  | $5\times5\times12=300$ |      

</center>

#### Step chi

For each row `x in [0..5)` and column `y in [0..5)`, perform (into quarters):

$$
\begin{align*}
F[x][y] := &\ B[x][y] \oplus (\ \neg B[x+1][y] \wedge \ B[x+2][y]) \\
\iff \\
sparse(F[x][y]) := &\ 
\end{align*} 
$$

<center> 

| Version  | Rows/round | Lookups/round | 
|----------|------------|---------------|
| This RFC | $5\times5\times?$        | $5\times5\times?$           |                
| Old PoC  | $5\times5\times(1.5+5+5)=287.5$ | $5\times5\times32=800$ |      

</center>

#### Step iota

On round $r$, update the word in the first row, first column xoring with the round constant as

$$
G[0][0] \oplus RC[r] \iff sparse(G[0][0]) + expand(RC[r])
$$

The round constants should be stored in expanded form, taking $24\times4$ witness cells as public inputs. 

<center>

| Gates            | Inputs | Output | Lookups | 
|------------------|--------|--------|---------|             
| `Xor64` + `Zero` | f, rc  | g      | $0$     |    

| Version  | Rows/round | Lookups/round | 
|----------|------------|---------------|
| This RFC | $2$        | $0$           |                
| Old PoC  | $5$        | $16$          |      

</center>

### Performance

Counting the costs of the steps presented above, the following table summarizes the performance of the proposed design, for each of the 24 rounds of the permutation function, in each design of the Keccak gadget.

<center>

| Version  | Rows / block | Lookups / block | 
|----------|--------------|-----------------|
| This RFC | $24\times(50+40+30+50+100+_+2)=$        | $24\times(0+0+140+0+700+_+0)=$           |                
| Old PoC  | $24\times(125+100+40+125+75+287.5+5)=18,180$ |$24\times(400+320+140+400+300+800+16)=57,024$ |      

</center>

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

This gate uses many more columns than our usual version of Kimchi. We need to understand if this could have an unacceptable effect on the proof length in our usecase.

## Rationale and alternatives

The generalized expression framework will provide the ability to create arbitrary number of columns and lookups per row for custom gates. This allows more optimized approaches to address boolean SNARK-unfriendly relations (among other advantages) such as XORs. 

Given that this design makes use of 16-bit lookup tables (eating up $2^{16}$ rows), it could make sense to wonder if an approach using 8-bit XOR lookup tables could make sense. This would imply that each 64-bit XOR would need 8 lookups (as opposed to the 16 in the PoC). 

## Prior art

The current [Keccak PoC in SnarkyML](https://www.notion.so/minaprotocol/Keccak-gadget-PoC-PRD-59b024bce9d5441c8a00a0fcc9b356ae) was introduced to support Ethereum primitives in MINA zkApps. Due to this blockchain's design, the gadget needed to be compatible with Kimchi: a Plonk-like SNARK instantiated with Pasta curves for Pickles recursion and IPA commitents. This means that the design choices for that gadget were determined by some features of this proof system, such as: 15-column width witness, 7 permutable witness cells per row, access to the current and next rows, up to 4 lookups per row, access to 4-bit XOR lookup table, and less than $2^{16}$ rows.

With these constraints in mind, the Keccak PoC extensively used a chainable custom gate called `Xor16`, which performs one 16-bit XOR per row. In particular, xoring 64-bit words (as used in Ethereum), required 4 such gates followed by 1 final `Zero` row. As a result, proving the Keccak hash of a message of 1 block length (up to 1080 bits) took ~15k rows.

## Unresolved questions

* During the implementation of this RFC:
    * make sure if 3 bits for expansion are required (16 boolean operations) or 2 is enough (8 ops) which would lead to 12-bit lookups instead;
    * obtain exact measurements of the number of rows, columns, constraints, lookups, seconds, required per block hash;
    * find out if the round constants should be hardcoded (takes memory space) or generated (takes computation resources);
    * decide if the rotation offsets will be directly stored modulo 64 or not;
    * if the endianness of the target input format is little endian, it will need to be transformed into big endian;
    * if the input is given as a bitstring, pack it into bytes.

* Future work: 
    * support SHA3 (NIST variant of Keccak), different output lengths, and different state widths;
    * improve chaining between gates


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
