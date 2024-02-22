# zkVM-Optimized Keccak (Revised)

This RFC presents a design of a Kimchi chip for the Keccak hash function. It is optimized for the zkVM project, making use of our [generalized expression framework](./0007-generalized-expression-framework.md), and assuming a variant of Kimchi using a Nova-styled folding over BN128 and KZG commitments. The generalized expression framework will provide the ability to create arbitrary number of columns and lookups per row for custom gates. This allows more optimized approaches to address boolean SNARK-unfriendly relations.

More abstractly, the design of the Keccak circuits can be seen as a Turing machine with access to a memory tape containing the witness data at each step. Moreover, it will make use of a shared tape as a communication channel between the Keccak and the MIPS circuits. This is widely explained in the [Keccak syscalls RFC](./0012-keccak-syscalls.md). 

## Summary

This document presents a design to deeply support the Keccak hash function for efficient pre-image oracle access within the proof system. To do so, it leverages a *bitwise sparse representation* to compute boolean operations (XORs, ANDs), and sequences of them.

## Motivation

One component of the Optimism stack system is the pre-image data oracle which is responsible for getting information about the chains. The OP stack currently uses the existing ledger which is built on top of Keccak hashes as a baseline.

Having a dedicated chip inside Kimchi to prove correct computation of Keccak hashes from pre-image data (blocks) will result in a more efficient design of the MIPS zkVM as a whole, meeting the performance goals of the [OP RFP](https://github.com/ethereum-optimism/ecosystem-contributions/issues/61#issuecomment-1611488039).

As indicated in the PRD, any member of the OP ecosystem who wanted to prove the state transition between two blocks of an OP stack chain executing the `op-program` program in the O(1) MIPS zkVM should complete within 2 days (before optimizations) to meet the latency estimates from the RFP. 

The ultimate objective is to improve the performance of our current Keccak gadget, to get over one hash per second. The new design would be a success if the number of hashes per second is faster than that of the current [SnarkyML Keccak PoC](https://github.com/MinaProtocol/mina/pull/13196). 

The optimizations in the proposed design rely on a more efficient representation of the `Xor64` operation (i.e. a gadget for 64-bit XOR). Targeting this boolean operation makes sense, as this is the most used component inside the permutation function of Keccak. In particular, the main source of overhead in the current design corresponds to this operation.

## Detailed design

The Keccak hash function (later standardized by NIST as SHA3, with small variations) is known to be a quantum-safe, and very efficient hash to compute. Nonethelss, its intensive use of boolean operations which are costly in finite field arithmetic makes it a SNARK-unfriendly function. In this section, we propose an optimized design for our proof system that should outperform our current Keccak to become usable in the zkVM for the OP stack. Moreover, the particular design of the circuit must be universal in style, so that instances of the circuit can be folded.

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

Thinking of the MIPS and the Keccak instances as two Turing machines is a useful abstraction. In this sense, the interpreter of the syscall reading preimage data from the oracle must trigger the Keccak proving workflow once full preimage data have been processed.  One should expect inputs and outputs to be found in bytes serialization, with a big-endian encoding. One can only read from Canon 4 bytes at a time, what means that the hash function will have to wait until the input is fully loaded onto memory.

That means, inputs and outputs of the Keccak hash function need to be passed between these two circuits. Because these are two independent machines, any kind of communication between them will happen through a shared memory tape. More concretely, inputs to the Keccak hash function will be written into the tape from the MIPS side, and read from the trape from the Keccak side; and the output of the hash function will be written into the tape from the Keccak side, and read from the tape from the MIPS side.

Using RAMLookups, the idea behind this shared memory is to enable a read/write table whose content must sum to zero at the end of the execution. That will mean that for each value that was added to the channel, the corresponding process consumed it correctly.

In order to write a lookup:

`lookup_aggreg += 1/(x + keccak_channel_table_id + value * joint_combiner^1)`

and then to read a lookup:

`lookup_aggreg -= 1/(x + keccak_channel_table_id + value * joint_combiner^1)`

so that the final lookup aggregation is unchanged iff the Keccak chip processed and proved all of the messages.

### Bitwise-sparse representation

Here, we present the bitwise-sparse representation of bitstrings that will be used to obtain more efficient computations of boolean functions within our proof system. 

Given that simple binary operations are very costly within a SNARK, the new idea consists on arithmetizing boolean operations as much as possible, becoming more efficient with finite field algebra. We introduce the _bitwise-sparse representation_ of bitstrings for this goal.

#### Expansion

With this notation, input bitstrings are expanded inserting three `0` bits in between "real" bits (hence the _sparsity_). The motivation for this, is to use intermediate bits to store auxiliary information such as carry bits, resulting from algebraic computations of boolean functions. That means, for each bit in the original bitstring, its expansion will contain four bits. The following expression represents such mapping:

$$\forall X \in \mathbb B^\ell: expand(X) \to 0 \ 0 \ 0 \ b_{\ell-1} \ . \ . \ . \ 0 \ 0 \ 0 \ b_0$$

The reason behind the choice of three empty intermediate bits in the sparse representation follows from the concrete use case of Keccak where no more than $2^4-1$ consecutive boolean operations are performed, and thus carries cannot overwrite the content of the bits to the leftmost bits.

> NOTE: the above means that after each step of the permutation, the expanded representation needs to be reset before starting the next step to ensure completeness of the encoding. 

Even though in each field element of ~254 bits long, one could fit up to 63 real bits with this encoding, each witness value will only store 16 such real bits. Instead, the expansion of the full 64 bit words could be represented by composition of each of the four 16-bit parts. But this will not be computed in the circuit, since the expansion would not fit entirely in the field. The way to perform this mapping is through a 16-bit 2-column lookup table that we call `Reset` (of $2^{16}$ entries, anything significantly larger than that such as 32 bits would be too costly).

Let $sparse(X)$ refer to a representation of $X$, where only the indices in the $4i$-th positions correspond to real bits, and the intermediate indices can contain auxiliary information. Connecting to the above, on input a word $X$, we can expand it in order to perform a series of boolean operations (up to 15) which results in some $sparse(X)$, encoding the same $X$. Then,

$$\forall X\in\mathbb B^\ell:\quad expand(X) \equiv reset(sparse(X)) $$

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
sparse(X) \iff &mask_0(sparse(X)) + mask_1(sparse(X)) \\
& + mask_2(sparse(X)) + mask_3(sparse(X))
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

The XOR of 16-bit can be computed using addition over the underlying field of their expansions. The 16-bit output of the XOR will be located on the $4n$ positions, whereas the intermediate positions will contain any possible carry terms.

Given expanded `left`, `right` inputs both in $\mathbb B^{64}$, the sparse representation of $XOR_{64}$ can be constrained in quarters of 16 bits, as:

$$\forall i\in[0,3]: sparse(xor_i) = expand(left_i) + expand(right_i)$$

Nonetheless, if no more than 15 boolean operations have been performed since the latest reset, one can use sparse representations of the left and right inputs as well.

#### Reset

After each step of Keccak, the sparse representation must be reset to avoid overflows of the intermediate bits. For $n \in [0, 3]$, let $shift_n = mask_n(sparse)/2^n$, the correct decomposition of a 64-bit word can be constrained as

$$\forall i \in [0,3]: sparse_i = shift0_i + 2\cdot shift1_i + 4\cdot shift2_i + 8\cdot shift3_i$$

where $sparse_i\in\mathbb B^{64}$ and $sparse\in\mathbb B^{256}$, together with a lookup for each `shift_i` (more about this later).

Each `sparse[i]` will be the output of previous boolean operations. The `shift0[i]` corresponds to the clean expand of the word, and could be the input of some upcoming boolean operations. The `shift1[i]` will be used by the AND. The 64-bit word computed from the `dense[i]` will be used in rotation.

##### AND

Note that the AND of two expanded inputs corresponds to the $4n+1$-th positions after an expanded XOR (addition). Meaning that $shift_1 = mask_1/2$ gives the expanded representation of the conjunction operation.
The AND operation can be constrained making use of XOR and Reset explained above. With no further constraints, XOR is used to obtain `left+right`, and then Reset provides the expanded result of AND in the `shift1` witnesses.

#### Negation

The NOT operation will be performed with a subtraction operation using the constant term $(2^{16}-1)$ for each 16-bit quarter, and only later it will be expanded. The mechanism to check that the input is at most 64 bits long, will rely on the composition of the sparse representation. Meaning, 64-bit words produce four 16-bit lookups, which implies that the real bits take no more than 64 bits. Given $x$ in **reset** form (meaning, not any sparse representation of $x$ but the initial expanded one with zero intermediate bits, the canonical representation) the negation of one 64-bit word can be constrained as:

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

Each Keccak circuit will have $2^{15}$ rows. The resulting instance must be identical to any other Keccak circuit, so that they are foldable. This has a series of implications, but the most relevant one in this section is the following: 

_Unlike Kimchi or base Plonk, where circuit gates could have public coefficients to configure each row, folding schemes require that the relation remains unchanged across instances._

This implies that any public flag or constant that could be used to define the actual step must be treated as private witnesses together with additional lookups and constraints to prove their correctness (as they are no longer accessible by the verifier).

Altogether, this means that the main design chip will consist of a single universal row encoding any kind of Keccak step (just like MIPS rows encode any possible instruction). Thanks to some witness flags, the chip will distinguish between two modes of operation:
- **Sponge**: the row will perform formatting actions related to the Keccak sponge. A sponge step can be either an absorb or a squeeze; and absorbs can optionally be root absorbs (the first one) and pad (the last one).
- **Round**: the row will perform one full round of the Keccak permutation function. This means a round step will run all of theta, pi-rho, chi, and iota algorithm in one single circuit row.

Making use of the generic expression framework, we can define the `KeccakColumn` enum, which will be used to refer to variables in the circuit, whose variants are: 
- `HashIndex`: keeps track of which Keccak hash inside the circuit this row belongs to.
- `StepIndex`: keeps track of the Keccak step number of the current hash.
- `FlagRound`:indicates the Keccak round number corresponding to the current row, if the row is not a sponge. It ranges between 0 and 24 (excluded).
- `FlagAbsorb`: indicates whether the current row is an absorb sponge (boolean value).
- `FlagSqueeze`: indicates whether the current row is a squeeze sponge (boolean value). 
- `FlagRoot`: indicates whether the current row corresponds to the first absorb sponge (boolean value). 
- `PadLength`: indicates the length in bytes of the padding of the block (if any). It ranges between 0 and 136 (excluded).
- `InvPadLength`: equals the inverse of `PadLength` when this is nonzero, used to check if the current row corresponds to the last absorb sponge.
- `TwoToPad`: stores the value $2^{\text {PadLength}}$, used to check the correctness of the padding.
- `PadBytesFlags[0..136)`: contains 136 boolean values indicating whether the i-th byte of the block to be absorbed is involved in the padding or not.
- `PadSuffix[0..4)`: contains 5 field elements containing the claimed pad for `PadLength` bytes. The first chunk contains the most significant 12 bytes, whereas the others contain 31 bytes each (so that the 1088 bits fit in 5 field elements).
- `RoundConstants[0..3)`: contains the round constants used by the iota algorithm of the current round, stored as four quarters expressed in bitwise sparse representation.
- `Input[0..100)`: contains 100 field elements which are the inputs of the current row. If it is a sponge step, it corresponds to the old state; if it is a round step, it corresponds to the state A of the theta algorithm. _Disclaimer: it does not refer to the input of the entire hash function, that is handled in the syscalls communication channel._
- `Output[0..100)`: contains 100 field elements which are the outputs of the current row. It if is a sponge step, it corresponds to the XOR of the blocks of the absorb; if it is a round, it corresponds to the state G of the iota algorithm. _Disclaimer: it does not refer to the output of the entire hash function, that is handled in the syscalls communication channel._
- `SpongeNewState[0..100)`: contains the new bolock to be absorbed padded with 64 bytes to zero (the last 32 positions), producing a state of 1600 bits stored as 100 expanded elements.
- `SpongeBytes[0..200)`: contains the decomposition into bytes of the new state if it is an absorb, or the old state of the squeeze to produce the output of the hash.
- `SpongeShifts[0..400)`: contains the shifts of the new state if it is an absorb to prove the expansion, or the old state of the squeeze to help prove the byte decomposition above.

Apart from those variants, it also contains the following columns whose meaning will be clearer when reading the constraints where these are used. Note that some intermediate states of the Keccak permutation function have been removed from the witness because their correctness can be embedded into other constraints, making the proof shorter. Keep in mind that the actual columns will be single-indexed encoded, but here they are presented with multiple indices to better understand where the sizes come from:

- `ThetaShiftsC[0..80)` with `i=[0..4), x=[0..5), q=[0..4)`,
- `ThetaDenseC[0..20)` with `x=[0,5), q=[0,4)`,
- `ThetaQuotientC[0..5)` with `x=[0..5)`,
- `ThetaRemainderC[0..20)` with `x=[0..5), q=[0..4)`,
- `ThetaDenseRotC[0..20)` with `x=[0..5), q=[0..4)`,
- `ThetaExpandRotC[0..20]` with `x=[0..5), q=[0..4)`,
- `PiRhoShiftsE[0..400)` with `i=[0..4), y=[0..5), x=[0..5), q=[0..4)`,
- `PiRhoDenseE[0..100)` with `y=[0..5), x=[0..5), q=[0..4)`,
- `PiRhoQuotientE[0..100)` with `y=[0..5), x=[0..5), q=[0..4)`,
- `PiRhoRemainderE[0..100)` with ` y=[0..5), x=[0..5), q=[0..4)`,
- `PiRhoDenseRotE[0..100)` with `y=[0..5), x=[0..5), q=[0..4)`,
- `PiRhoExpandRotE[0..100)` with `y=[0..5), x=[0..5), q=[0..4)`,
- `ChiShiftsB[0..400)` with `i=[0..4), y=[0..5), x=[0..5), q=[0..4)`,
- `ChiShiftsSum[0..400)` with `i=[0..4), y=[0..5), x=[0..5), q=[0..4)`.

The above are just aliases for the underneath witness columns. The round alone requires 1965 columns to store the above values, whereas the sponge alone requires 800. Because all rows need to look identical, this means sponge rows will pay for the full length of the columns. That means we can reuse some free space of sponge witnesses to store flags that are only required when the step is a sponge. This means, instead of requiring a trivial witness vector of 2219 entries, the implementation can also work with 2074 instead. The high-level layout of the actual witness values results as follows:

| `Common` | hash_index | step_index | mode_flags | curr[0..100) | next
| -------------- | --------- | ----------- | ----------- | --- | --- |
|  | `HashIndex` | `StepIndex`   | `FlagRound` | `Input` | `Output`
|  |             |               | `FlagAbsorb` | 
|  |             |               | `FlagSqueeze` |


| `Sponge` | curr[100...200) |  curr[200...400) | curr[400...800) | curr[800..804) | curr[804..940) | curr[940..945) | 
| -------------- | ----------- | ----------- | ----------- | ---- | --- | --- |
|  | `SpongeNewState` where: | `SpongeBytes`       | `SpongeShifts` | `FlagRoot` | `PadBytesFlags` | `PadSuffix`
|  | new block [100..168) |  |  | `PadLength`
|  | zeros [168..200)  |    |  | `InvPadLength`
|  |                   |    |  | `TwoToPad`


| `Round` | curr[100...265) | curr[265...1165) | curr[1165...1965) | curr[1965..1969) |
| ------- | --------------- | ---------------- | ----------------- | ---------------- |
| | `ThetaShiftsC` | `PiRhoShiftsE` | `ChiShiftsB` | `RoundConstants` |
| | `ThetaDenseC` | `PiRhoDenseE` | `ChiShiftsSum` |
| | `ThetaQuotientC` | `PiRhoQuotientE` |  |
| | `ThetaRemainderC` | `PiRhoRemainderE` |  |
| | `ThetaDenseRotC` | `PiRhoDenseRotE` |  |
| | `ThetaExpandRotC` | `PiRhoExpandRotE` |  |

#### Flags

Because the concrete configuration of the gate is hidden inside witnesses instead of public coefficients, there need to be constraints to check the correctness of the operation mode. These basically consist of booleanity and mutual-exclusivity checks:

Check that absorb, squeeze, and root flags, are either true or false:
```rust
constrain(is_boolean(is_absorb));
constrain(is_boolean(is_squeeze));
constrain(is_boolean(is_root));
```

Check that the 136 leading bytes of the new block are either involved in padding or not:
```rust
for i in [0..136)
    constrain(is_pad * is_boolean(in_padding(i)));
```

Check that root nor pad can happen in squeeze nor rounds:
```rust
constrain(either_false(is_squeeze, is_root));
constrain(either_false(is_squeeze, is_pad));
constrain(either_false(is_round, is_root));
constrain(either_false(is_round, is_pad));
```

Check that absorb and squeeze cannot happen at the same time:
```rust
constrain(either_false(is_absorb, is_squeeze));
```
Note that other checks hold trivially by construction of the variables involved. For example, `is_round` is defined as the negation of `is_sponge`, so it must aslo be boolean, and they are mutually exclusive. Similarly, `is_pad` is defined as whether `PadLength` is nonzero (using the inverse), so it is also boolean. Regarding the rest of flags which are not always boolean, the correctness will follow from the lookup tables themselves.

#### Sponge

There are two main modes: absorb and squeeze. Inside the absorb mode, there are two additional submodes called pad and root, which can as well happen at the same time (if there is only one block to be hashed, and then the first absorb is also the last one).

- **Absorb mode**

    In absorb mode, `FlagAbsorb` is set to 1, whereas `FlagSqueeze` and `FlagRound` remain as 0.

    The gate takes a block as 68 chunks of 16 bits expanded, pads with zeros until reaching 100 chunks (1600 bits) to fit one whole state, and XORs it with the previous state.

    The zero padding is enforced performing the following 32 constraints:

    ```rust
    constrain(is_absorb * zeros[i])
    ```

    The XOR part of the gate uses $100$ constraints 

    ```rust
    constrain(is_absorb * (xor_state[i] - (old_state[i] + new_state[i])))
    ```
    where `new_state` is the concatenation of `new_block` and `zeros`, and `old_state` corresponds to the columns in `Input`.

    Additionally, the gate must check the decomposition into shifts of the new block (where `shiftsi()` corresponds to every 100th chunk of `SpongeShifts(idx)`):

    ```rust
    for i in [0..100) {
        constrain(is_absorb * (new_block[i] - (
                            shifts0(i)
                            + 2 * shifts1(i)
                            + 4 * shifts2(i)
                            + 8 * shifts3(i) )))
    }
    ```

    The correspondence between the shifts and the dense bytes will happen inside a lookup pattern.

    - **Root mode**

        In this mode, which can only happen when `is_absorb=1`, `FlagRoot` is set to 1.

        This mode is only performed once per hash, and it must happen in the first absorb. The only difference between this one and any other absorb is that the root mode sets the initial state to all zeros. For that reason, apart from the constraints above, it checks:

        ```rust
        constrain(is_root * old_state[i])
        ```
    
    - **Pad mode** 

        Padding with the $10^*1$ rule only happens once per hash, in the last absorb. That is because the input message is padded until reaching a length that is a multiple of 136 bytes, which always fully fits in one whole block. This means, in the case that the input message is $\leq 135$ bytes in length, the single absorb will run in both `is_root` and `is_pad` modes.

        When in pad mode, we will have 136 selector flags `PadBytesFlags` that will activate at most 136 positions corresponding to the padding. Moreover, `PadLength` will take a nonzero value between 1 and 135. 
        
        Additionally, each of the 136 bytes of the new block are set to either `0x00`, `0x01`, `0x80`, or `0x81`, which are all the possible combinations that a padding byte can take. These known values are aggregated into 5 field elements inside `PadSuffix`, which will be used inside a lookup table to check that the padding was done correctly for a given padding length. The gate must show the link between the bytes and the actual state as: 
        ```rust
        for i in [0..5)
            constrain(is_pad * (block_in_padding(i) - pad_suffix(i)))
        
        ```

        Moreover, the padding location must be checked to be at the end of the message. This is done using the `TwoToPad` value and the idea that if the padding is located at the end, then the last and consecutive `PadLength` bytes in `PadBytesFlags` must be set to one. 
        ```rust
        let pad_at_end := [0..136).fold(0, |acc, i| 2 * acc  + sponge_byte(i));

        constrain(is_pad * (two_to_pad - 1 - pad_at_end));
        ```

- **Squeeze mode**

    The gate takes the first 256 bits of the digest (first 4 words of the first column of the state, meaning first 16 expanded terms), and decomposes from the expanded state to form 32 bytes corresponding to the hash. 

    In squeeze mode, `FlagSqueeze` is set to 1; leaving `FlagAbsorb`, `FlagRoot`, `PadLength`, and `FlagRound` to 0.

    Here, the state being decomposed into shifts is the one resulting from the most recent permutation round:

    ```rust
    for i in [0..16)
        constrain(is_squeeze * (old_state[i] - 
                                (shift[4i] 
                                    + 2*shift[4i+1] 
                                    + 4*shift[4i+2] 
                                    + 8*shift[4i+3]))) 
    ```

    The correspondence between the shifts and the dense bytes will happen inside a lookup pattern.

#### Round

When the Keccak step being performed is a round, `FlagRound` is assigned to the i-th Keccak round (value from 0 to 23), and `RoundConstants` are instantiated accordingly.

- **Theta algorithm**

Here, `state_a` is contained inside the witness variable `Input`. 

For each row `x in [0..5)` in the state `A`, compute the `C` state as:

$$
\begin{align*}
C[x] := &\ A[x][0]\ \oplus\ A[x][1]\ \oplus\ A[x][2]\ \oplus\ A[x][3]\ \oplus\ A[x][4] \\
\iff
sparse(C[x]) := &\ expand(A[x][0]) + expand(A[x][1]) + expand(A[x][2]) + expand(A[x][3]) + expand(A[x][4]) \\
\end{align*} 
$$

and constrain its value by embedding it into the following 20 shifts decomposition constraints

```rust
for x in [0..5)
    for q in [0..4)
        state_c(x)[q] := state_a(x,0)[q] + state_a(x,1)[q] + state_a(x,2)[q] + state_a(x,3)[q] + state_a(x,4)[q];
            constrain(is_round * (state_c(x)[q] - ( shift0_c(x)[q] 
                                                + 2*shift1_c(x)[q] 
                                                + 4*shift2_c(x)[q] 
                                                + 8*shift3_c(x)[q] )))
```

Similarly, for each row `x in [0..5)`, perform the following operation splitting into quarters

$$
\begin{align*}
D[x] := &\ C[x-1]\ \oplus\ ROT(C[x+1],1) \\
\iff \\
sparse(D[x]) := &\ expand(C[x-1]) + ROT(expand(C[x+1]), 1)\\
\end{align*} 
$$

Here, indices operations must be performed modulo 5, so `x+1` should be `(x+1)%5` whereas `x-1` should be `(x+4)%5`.

```rust
    for x in [0..5)
        for q in [0..4)
            state_d(x)[q] := shifts0_c(x-1)[q] + expand_rot_c(x+1)[q]
```

For this, the 5 possible inputs of the rotation need to be reset. The input of the XOR will use the canonical version as well, to reset any previous round auxiliary bits.

This part uses the following 15 constraints

```rust
for x in [0..5)
    let word(x)      = dense_c(x)[0]     + 2^16*dense_c(x)[1]     + 2^32*dense_c(x)[2]     + 2^48*dense_c(x)[3]
    let remainder(x) = remainder_c(x,0) + 2^16*remainder_c(x)[1] + 2^32*remainder_c(x)[2] + 2^48*remainder_c(x)[3]
    let rotated(x)   = dense_rot_c(x,0) + 2^16*dense_rot_c(x)[1] + 2^32*dense_rot_c(x)[2] + 2^48*dense_rot_c(x)[3]
    constrain(is_round * (word(x) * 2 - (quotient_c(x) * 2^64 + remainder_c(x))));
    constrain(is_round * (rotated(x) - (quotient_c(x) + remainder(x))));
    constrain(is_round * (is_boolean(quotient_c(x))));
```

and $140$ lookups.


Next, for each row `x in [0..5)` and column `y in [0..5)`, compute (with no need of resetting `D`):

$$
\begin{align*}
E[x][y] := &\ A[x][y]\ \oplus\ D[x] \\
\iff \\
sparse(E[x][y]) := &\ sparse(A[x][y]) + sparse(D[x])\\
\end{align*} 
$$

```rust
for i in [0..4)
    for x in [0..5)
        for y in [0..5)
            state_e(x,y)[q] := state_a(x,y)[q] + state_d(x)[q];
```

- **Pi-Rho algorithm**

For each row `x in [0..5)` and column `y in [0..5)`, perform (into quarters):

$$
\begin{align*}
B[y][2x+3y] := &\ ROT(E[x][y], OFF[x][y]) \\
\iff \\
expand(B[y][2x+3y]) := &\ ROT(expand(E[x][y]), OFF[x][y])
\end{align*} 
$$

Recall that in order to perform the rotation operation, the state needs to be reset. This step can be carried out with the following $150$ constraints and $800$ lookups:

```rust
for x in [0...5)
    for y in [0...5)
        let word(x,y)      = dense_e(x,y)[0]     + 2^16*dense_e(x,y)[1]     + 2^32*dense_e(x,y)[2]     + 2^48*dense_e(x,y)[3]
        let quotient(x,y)  = quotient_e(x,y)[0]  + 2^16*quotient_e(x,y)[1]  + 2^32*quotient_e(x,y)[2]  + 2^48*quotient_e(x,y)[3]
        let remainder(x,y) = remainder_e(x,y)[0] + 2^16*remainder_e(x,y)[1] + 2^32*remainder_e(x,y)[2] + 2^48*remainder_e(x,y)[3]
        let rotated(x,y)   = dense_rot_e(x,y)[0] + 2^16*dense_rot_e(x)[1] + 2^32*dense_rot_e(x,y)[2] + 2^48*dense_rot_e(x,y)[3]
        constrain( is_round * (word(x,y) * 2^(OFF(x,y)) - ( quotient(x) * 2^64 + remainder(x))) )
        constrain( is_round * (rotated(x,y) - (quotient(x,y) + remainder(x)) ))
        for q in [0...4)
            let state_b(y,2x+3y)[q] = expand_rot_e(x,y)[q];
            constrain( is_round * (state_e(x,y)[q] - (shift0_e(x,y)[q] + 2*shift1_e(x,y)[q] + 4*shift2_e(x,y)[q] + 8*shift3_e(x,y)[q] ) ))
```

Note that the value `OFF(x,y)` will be a different constant for each index, according to the rotation table of Keccak.


- **Chi algorithm**

For each row `x in [0..5)` and column `y in [0..5)`, perform (into quarters):

$$
\begin{align*}
F[x][y] := &\ B[x][y] \oplus (\ \neg B[x+1][y] \wedge \ B[x+2][y]) \\
\iff \\
2 \cdot sparse(F[x][y]) := &\ 2 \cdot expand(B[x][y]) + shift_1((expand(2^{64}-1) - expand(B[x+1][y]) + expand(B[x+2][y]) ) )
\end{align*} 
$$

This is constrained with the following $200$ constraints and $800$ lookups

```rust
for q in [0..4)
    for x in [0..5)
        for y in [0..5)
            let not(x,y)[q] = 0x1111111111111111 - shift0_b(x+1,y)[q]
            let sum(x,y)[q] = not(x,y)[q] + shift1_b(x+2,y)[q]
            constrain( is_round * (state_b(x,y)[i] - (shift0_b(x,y)[q] + 2*shift1_b(x,y)[q] + 4*shift2_b(x,y)[q] + 8*shift3_b(x,y)[q] ) ))
            constrain( is_round * (sum(x,y)[q] - (shift0_sum(x,y)[q] + 2*shift1_sum(x,y)[q] + 4*shift2_sum(x,y)[q] + 8*shift3_sum(x,y)[q] ) ))
            let and(x,y)[i] = shift1_sum(x,y)[i] 
            let state_f(x,y)[q] = shift0_b(x,y)[q] + and(x,y)[q];
```

- **Iota algorithm**

On round $r$, update the word in the first row, first column xoring with the round constant as

$$
G[0][0] \oplus RC[r] \iff sparse(G[0][0]) + expand(RC[r])
$$

The round constants should be stored in expanded form in `RoundConstants`. This only requires $4$ constraints and $1$ lookup.

```rust
for q in [0..4)
    constrain( state_g(0,0)[q] - (state_f(0,0)[q] + round_constants(q) )
```

After this last step of the permutation function, `Keccak` will store state `G` in `Output`. Recall that except for `g_0_0`, the rest of `G` is `state_f`.


### Lookups

Each row of Keccak needs 2342 read lookups.

#### Tables

The design uses a 2-column lookup table called `Reset`` containing the elements from $0$ to $2^{16}-1$, and their expansion. As such, it can be used to check that the right expansion was used for a given chunk of 16 bits. 

<center>

| `Reset` | expansion of each 16-bit input                                    |
| --- | ----------------------------------------------------------------- |
| $0$ |`0000000000000000000000000000000000000000000000000000000000000000` |
|     | ...                                                               |
| $i$ |`000` $b_{15}$ `000` $b_{14}$ `000` $b_{13}$ `000` $b_{12}$ `000` $b_{11}$ `000` $b_{10}$ `000` $b_{9}$ `000` $b_{8}$ `000` $b_{7}$ `000` $b_{6}$ `000` $b_{5}$ `000` $b_{4}$ `000` $b_{3}$ `000` $b_{2}$ `000` $b_{1}$ `000` $b_{0}$ |
|     | ...                                                               |
| $2^{16}-1$ | `0001000100010001000100010001000100010001000100010001000100010001` |

</center>

Additionally, we create two more tables which correspond to the first and second columns of the former (called `RangeCheck16` and `Sparse` respectively). `RangeCheck16` can be used to range check some terms in rotation, whereas `Sparse` can be used to check the correct form of the shifts (since the non-sparse pre-image is non-relevant for $shift_i$ for $i\in[1,3]$). 

<center>

| `RangeCheck16` | 
| --- | 
| $0$ |
| ... |                                      
| $2^{16}-1$ |

</center>

<center>

| `Sparse` |
| --- | 
| `0000000000000000000000000000000000000000000000000000000000000000` |
| ...            |
| `000` $b_{15}$ `000` $b_{14}$ `000` $b_{13}$ `000` $b_{12}$ `000` $b_{11}$ `000` $b_{10}$ `000` $b_{9}$ `000` $b_{8}$ `000` $b_{7}$ `000` $b_{6}$ `000` $b_{5}$ `000` $b_{4}$ `000` $b_{3}$ `000` $b_{2}$ `000` $b_{1}$ `000` $b_{0}$ |
|  ...             |
| `0001000100010001000100010001000100010001000100010001000100010001` |

</center>

The design makes use of a smaller table containing all bytes (values up to 8 bits).

<center>

| `Byte` |
| --- |
| $0$ |
| ... |
| $255$ |

</center>

In order to check the correctness of the round constants, there is a table containing each round index and the 4 expanded quarters of the corresponding round constant.

<center>

| `RoundConstants` | | | | |
| --- | - | - | - | - |
| $0$ | expand(`0x0000`) | expand(`0x0000`) | expand(`0x0000`) | expand(`0x0001`) |
| ... |
| $23$ | expand(`0x8000`) | expand(`0x0000`) | expand(`0x8000`) | expand(`0x8008`) |

</center>

In order to check the padding, the `Pad` table stores information about the pad length, the power of two of this value, and the 5 padding suffix blocks.

| `Pad` | | | | | | |
| --- | - | - | - | - | - | - |
| $1$ | $2$ | 0 | 0 | 0 | 0 | 0x81 |
| ... |
| $135$ | $2^{135}$ | 0x01 | 0 | 0 | 0 | 0x80 |

</center>

#### Step lookups

- **Round step**

The round mode performs $1,741$ read lookups as indicated below:

- _Theta algorithm_

Theta requires 140 lookups.

In round steps, `ThetaRemainderC` values need to be checked to be 64 bits at most, which is done using one `RangeCheck16` lookups per quarter with the value:
```rust
vec![remainder_c(x)[q]]
```

In round steps, we check that `ThetaExpandRotC` is the expansion of `ThetaDenseRotC` and `ThetaShiftC0` is the expansion of `ThetaDenseC` 
 with 40 lookups to the `Reset` table with the values:
```rust
vec![dense_rot_c(x)[q], expand_rot_c(x)[ q]]
vec![dense_c(x)[q], shifts0_c(x)[q]]
```

Finally, the rest of `shifts_c` must be looked up in the `Sparse` table.
```rust
vec![shifts1_c(x)[q]]
vec![shifts2_c(x)[q]]
vec![shifts3_c(x)[q]]
```

- _Pi-Rho algorithm_

PiRho requires 800 lookups.

In round steps, the values of the remainder and quotient of state E must be range checked to be at most 64 bits, by looking up these values in the `RangeCheck16` table:
```rust
vec![quotient_e(x,y)[q]]
vec![remiander_e(x,y)[q]]
```

In round steps, we must check that `PiRhoExpandRotE` is the expansion of `PiRhoDenseRotE` and `PiRhoShift0E` is the expansion of `PiRhoDenseE` with the following lookups to the `Reset` table:
```rust
vec![dense_rot_e(x,y)[q], expand_rot_e(x,y)[q]]
vec![dense_e(x,y)[q], shifts0_e(x,y)[q]]
```

For the rest of shifts, they must be checked to be correct sparse values with the `Sparse` table:
```rust
vec![shifts1_e(x,y)[q]]
vec![shifts2_e(x,y)[q]]
vec![shifts3_e(x,y)[q]]
```

- _Chi algorithm_

The chi algorithm requires 800 lookups to check that `ChiShiftsB` and `ChiShiftsSum` are in the `Sparse` table.
```rust
vec![shiftsi_b(x,y)[q]]
vec![shiftsi_sum(x,y)[q]]
```

- _Iota algorithm_

The iota algorithm requires 1 lookup to check the correctness of the `RoundConstants`:
```rust
vec![round(),
    round_constants()[0],
    round_constants()[1],
    round_constants()[2],
    round_constants()[3],
    ]
```

- **Sponge step**

The sponge mode gate performs $601$ read lookups:

When the step is a padding, this entry will be looked up from the `Pad` table:

```rust
vec![ pad_length(),
      two_to_pad(),
      pad_suffix(0),
      pad_suffix(1),
      pad_suffix(2),
      pad_suffix(3),
      pad_suffix(4),
    ],
;
```

The byte decomposition in sponge steps will be checked with the following 200 lookups to the `Bytes` table: 
```rust
vec![ sponge_byte(i)]
```

The shifts of sponge rows will be checked to be correct sparse values looking up these 300 values in the `Sparse` table:
```rust
vec![sponge_shifts1(i)]
vec![sponge_shifts2(i)]
vec![sponge_shifts3(i)]
```

Similarly for the canonical value stored in `sponge_shifts0`, these 100 lookups to the `Reset` table additionally check that each pair of bytes is composed into the correct dense quarters. Note that these 100 lookups to `Reset` are performed using a linear combination of the bytes to obtain the corresponding 16 dense bits for each `shift0` term. Take into account that bytes shall be laid out in big-endian format, whereas each dense pair is in little-endian.
```rust
let dense = sponge_byte(2 * i) + sponge_byte(2 * i + 1) * 2^8;
vec![dense, sponge_shifts0(i)]
```

#### Inter-step lookups

Because there are no copy constraints between rows to keep instances identical for folding, the design needs to account for a mechanism to connect consecutive Keccak steps inputs and outputs. For this, we use the `KeccakStep` table, used to store the hash index, step number, and corresponding input/output. This way, the output of step `i` is written when the step is not a squeeze (in whose case the output shall be written to the `Syscall` table instead), and read as input of step `i+1` when the step is not a root absorb(in whose case the input shall be read from the `Syscall` table instead).


### Performance

Counting the costs of the steps presented above, the following table summarizes the performance of the proposed design, for each of the 24 rounds of the permutation function, in each design of the Keccak gadget.

<center>

| Version  | Columns | Rows / block | Lookups / block | 
|----------|---------|--------------|-----------------|
| This RFC | 2074    | $25(+1)$ | $58k$ |    

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

## Appendix

The official documentation and resources regarding the Keccak hash function can be found in the [Keccak team website](https://keccak.team/).

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