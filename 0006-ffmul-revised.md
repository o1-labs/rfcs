# Foreign field multiplication gate

## Summary

We implement a revised version of the foreign field multiplication (ffmul) custom gate described in the [original ffmul RFC](https://o1-labs.github.io/proof-systems/rfcs/foreign_field_mul.html). The gate constrains integers $a$, $b$ (inputs), $q$ (quotient) and $r$ (remainder) to satisfy

$$\tag{1}
a b = qf + r
$$

where $f$ (the modulus) is any number up to 259 bits. This means that $ab = r \bmod{f}$, so if $f$ is a prime we are constraining multiplication in the foreign field $\mathbb{F}_f$.

The revised ffmul gate is proved sound, which means that the constraints imposed by the gate, together with additional constraints that must be added alongside the gate, imply the intended relation $(1)$ in a strict mathematical sense.

## Motivation

Implementing foreign field multiplication as a custom gate is a requirement for efficiently supporting many important operations in provable code, such as ECDSA signature verification which unlocks interoperability with Ethereum.

Revising the existing ffmul RFC and implementation is necessary because we found soundness issues which [allow a prover to provide incorrect values](https://hackmd.io/@mitschabaude/SJsLdMgFn#Counter-example-1-Skipping-q_2-in-0-2ell) of $r$ for most inputs $a$ and $b$, leading to devastating attacks such as verifying (multiple different) invalid versions of a signature which is supposed to be deterministic.

This RFC optimizes for provable correctness while requiring minimal changes to the pre-existing design and implementation.

**Expected outcome**: An implementation of the ffmul gate and supporting gadgets which we are confident in to be secure.

## Detailed design

This section is intended to be readable in a self-contained way. However, for a fuller picture, especially of the motivations of many choices made in the design, we recommend to read the [original ffmul RFC](https://o1-labs.github.io/proof-systems/rfcs/foreign_field_mul.html).

Foreign field elements $x$ and other integers of size comparable to $f$ are represented by three native field element limbs $(x_0, x_1, x_2)$, via

$$
x = x_0 + 2^\ell x_1 + 2^{2\ell} x_2.
$$

The limb size $\ell = 88$ is chosen to enable efficient $\ell$-bit range checks, so we can prove that $x_i \in [0, 2^\ell)$. This allows us to express any integer $x \in [0, 2^{3\ell}) = [0, 2^{264})$ in limb form, covering important 256-bit finite fields like secp256k1. From now on, we work under the assumption that $f < 2^{3\ell}$.

The ffmul gate represents $a$, $b$, $q$ in 3-limb form. However, $r$ is represented in an alternative compact 2-limb form $(r_{01}, r_2)$ where the two bottom limbs are combined into a single variable,

$$
r = r_{01} + 2^{2\ell}r_2.
$$

Our range-check (RC) gates support a "compact mode" where a variable $r_{01}$ is checked to be split like $r_{01} = r_0 + 2^\ell r_{1}$, and $r_0$ and $r_1$ have $\ell$ bits. This will be used to range-check $r$ and expose its individual limbs to other gates. The motivation for using the compact form for $r$ is to free up permutable cells in the gate layout, which is described below.

### Constraints and soundness proof

We will now derive the constraints necessary to prove that

$$\tag{1}
a b = qf + r
$$

At the same time, we will prove soundness -- i.e., the constraints we add imply $(1)$.

Let $n$ be the _native_ field modulus (for example, $n$ is one of the Pasta primes). Our first constraint is to check $(1)$ modulo $n$:

$$\tag{C1: native}
ab = qf + r\mod n
$$

Note: In the implementation, variables in this constraint are expanded into their limbs, like

$$
ab = (a_0 + 2^\ell a_1 + 2^{2\ell} a_2)(b_0 + 2^\ell b_1 + 2^{2\ell} b_2).
$$

Equation $\text{(C1: native)}$ implies that there is an $\varepsilon\in\mathbb{Z}$ such that

$$
ab - qf - r =  \varepsilon n.
$$

If the foreign field was small enough, we could prove $|ab - qf - r| < n$ (after adding a few range checks), and we would be finished at this point, because we could conclude that $\varepsilon = 0$. However, for the moduli $f$ we care about, $ab \approx qf \approx f^2$ is much larger than $n$, so we need to do more work.

The broad strategy is to also constrain $(1)$ modulo $2^{3\ell}$, which implies that it holds modulo the product $2^{3\ell}n$ (by the [chinese remainder theorem](https://en.wikipedia.org/wiki/Chinese_remainder_theorem)). The modulus $2^{3\ell}n$ is large enough that it can replace $n$ in the last paragraph and the argument actually works.

#### Expanding into limbs

First, we assume the following bounds. These have to be ensured using $\ell$-bit range checks:

$$
\begin{align}
\tag{RC $a$}
& a_0, a_1, a_2 \in [0,2^\ell) \\
\tag{RC $b$}
& b_0, b_1, b_2 \in [0,2^\ell) \\
\tag{RC $q$}
& q_0, q_1, q_2 \in [0,2^\ell) \\
\tag{RC $r$}
& r_{01} \in [0,2^{2\ell}), r_2 \in [0,2^\ell) \\
\end{align}
$$

Also, we define $f' := 2^{3\ell} - f > 0$ with limbs $(f'_0, f'_1, f'_2) \in [0, 2^\ell)^3$ and write

$$
ab - qf - r = ab + qf' - r - 2^{3\ell}q
$$

This is a trick from the original RFC to [avoid negative intermediate values](https://o1-labs.github.io/proof-systems/rfcs/foreign_field_mul.html#avoiding-borrows). Note that $ab + qf'$ and all of its limbs are positive, and that since we will work modulo $2^{3\ell}$, we can ignore the extra term $2^{3\ell}q$.

Next, we expand our equation into limbs, but collect all terms higher than $2^{3\ell}$ into a single term $w$.

$$
\begin{align}
& a b - q f - r = \\
&  (a_0 b_0 + q_0 f_0') + 2^{\ell} (a_0 b_1 + a_1 b_0 + q_0 f_1' + q_1 f_0') - r_{01} \\
&+ 2^{2\ell} (a_0 b_2 + a_2 b_0 + q_0 f_2' + q_2 f_0' + a_1 b_1 + q_1 f_1' - r_2) \\
&+ 2^{3\ell} w
\end{align}
$$

To make our equations easier to read, we define

$$
\begin{align}
& p_0 := a_0b_0 + q_0f'_0 \\
& p_1 := a_0b_1 + a_1b_0 + q_0f'_1 + q_1f'_0 \\
& p_2 := a_0b_2 + a_2b_0 + q_0f'_2 + q_2f'_0 + a_1b_1 + q_1f'_1
\end{align}
$$

Note: we don't introduce separate witnesses to hold $p_i$. Instead, the implementation will expand out the $p_i$ into their definitions wherever they are used in constraints. Now our limb-wise equation reads

$$\tag{2}
a b - q f - r = (p_0 + 2^{\ell} p_1 - r_{01}) + 2^{2\ell} (p_2 - r_2) + 2^{3\ell} w
$$

This equation holds over the integers, by definition of the terms involved.

#### Splitting the middle limb

We want to constrain the right-hand side (RHS) of $(2)$ in parts that fit within the native modulus $n$ without overflow. The two parts we want to use are $(p_0 + 2^{\ell} p_1 - r_{01})$ and $(p_2 - r_2)$. The only problem with that is that $2^\ell p_1$ doesn't quite fit within $n$.

In fact, thanks to our range-checks, we know that a product such as $a_0 b_0$ satisfies $0 \le a_0 b_0 < 2^{\ell} \cdot 2^{\ell} = 2^{2\ell}$. This gives us estimates on $p_0, p_1, p_2$:

$$
\begin{align}
& 0 \le p_0 = a_0b_0 + q_0f'_0 < 2 \cdot 2^{2\ell} \\
& 0 \le p_1 = a_0b_1 + a_1b_0 + q_0f'_1 + q_1f'_0 < 4 \cdot 2^{2\ell} \\
& 0 \le p_2 = a_0b_2 + a_2b_0 + q_0f'_2 + q_2f'_0 + a_1b_1 + q_1f'_1 < 6 \cdot 2^{2\ell}
\end{align}
$$

In particular, $p_1$ has at most $2\ell+2$ bits and so $2^\ell p_1$ might be larger than $2^{3\ell} > n$. This is the motivation to split $p_1$ into an $\ell$-bit bottom half and an $(\ell+2)$-bit top half:

$$
p_1 = p_{10} + 2^\ell p_{11}.
$$

To facilitate RCs, the top half is further split into

$$
p_{11} = p_{110} + 2^\ell p_{111}
$$

where $p_{110}$ has $\ell$ bits and $p_{111}$ has the remaining 2 bits.

* **Introduce 3 witnesses**: $p_{10}$, $p_{110}$ and $p_{111}$

In summary, the split of $p_1$ gives us our second constraint:

$$\tag{C2: split middle}
p_1 = p_{10} + 2^\ell p_{110} + 2^{2\ell} p_{111} \mod n
$$

Constraining $p_{111}$ to two bits is our third constraint:

$$\tag{C3: 2-bit $p_{111}$}
p_{111}(p_{111}-1)(p_{111}-2)(p_{111}-3) = 0 \mod n
$$

And we add two RCs that have to be performed externally:

$$
\begin{align}
\tag{RC $p_{1}$}
& p_{10}, p_{110} \in [0,2^\ell) \\
\end{align}
$$

Using these RCs and our earlier estimate on $p_1$, we can prove that $\text{(C2)}$ not only holds modulo $n$, but over the integers:

$$
|p_1 - p_{10} - 2^\ell p_{110} - 2^{2\ell} p_{111}| < 4\cdot2^{2\ell} + 2^{\ell} + 2^{2\ell} + 4\cdot 2^{2\ell} < n
$$

Therefore, we can use $\text{(C2)}$ to expand $p_1$ in our earlier equation $(2)$:

$$\tag{$2'$}
a b - q f - r = (p_0 + 2^{\ell} p_{10} - r_{01}) + 2^{2\ell} (p_2 - r_2 + p_{11}) + 2^{3\ell} w
$$

#### Constraining limb-wise mod $2^{3\ell}$

Now both terms in brackets of $(2')$ fit within $n$. We add a constraint that the first bracket is zero, except for a possible carry $c_0$:

$$\tag{C4: bottom limbs}
p_0 + 2^\ell p_{01} - r_{01} = 2^{2\ell} c_0\mod n
$$

* **Introduce 1 witness**: $c_{0}$

Note first that we can constrain $c_0$ to be non-negative since $r_{01}$ is at most $2\ell$ bits and the lower $2\ell$ bits of the LHS are supposed to be zero. Let's estimate the LHS to compute how many bits $c_0$ can have:

$$
p_0 + 2^\ell p_{01} - r_{01} < 2^{2\ell + 1} + 2^{2\ell} - 0 < 2^{2\ell + 2}
$$

This shows that $c_0$ will have at most 2 bits, which motivates our next constraint:

$$\tag{C5: 2-bit $c_0$}
c_0(c_0-1)(c_0-2)(c_0-3) = 0 \mod n
$$

This constraint -- together with our estimate of $p_0$ and RCs on $p_{01}$ and $r_{01}$ -- allows us to prove that $\text{(C4)}$ doesn't overflow $n$ and holds over the integers:

$$
|p_0 + 2^\ell p_{01} - r_{01} - 2^{2\ell}c_0| < (2 + 1 + 1 + 4)\cdot 2^{2\ell} < n
$$

Therefore, we can use $\text{(C4)}$ in $(2')$ to eliminate the bottom part:

$$\tag{$2''$}
a b - q f - r = 2^{2\ell} (p_2 - r_2 + p_{11} + c_0) + 2^{3\ell} w
$$

The next step is to constrain the $2^{2\ell}$ term, to be zero in its lower $\ell$ bits:

$$\tag{C6: high limb}
p_2 - r_2 + p_{11} + c_0 = 2^{\ell} c_1\mod n
$$

Again, we introduce a carry value $c_1$ which is supposed to be positive, since $r_2$ has at most $\ell$ bits. Estimating the LHS shows us how many bits $c_1$ will at most have:

$$
p_2 - r_2 + p_{11} + c_0 < 6 \cdot 2^{2\ell} - 0 + 2^{\ell + 2} + 4 < 2^{2\ell + 3}
$$

We see that $c_1$ has at most $\ell + 3$ bits. To save rows and not fill more space among the permutable witness cells, we won't pass $c_1$ to a RC gate. Instead, we inline the entire RC into the ffmul gate itself, using a combination of 7x 12-bit lookups plus 3x 2-bit constraints plus 1x 1-bit constraint, giving us $7 \cdot 12 + 3 \cdot 2 + 1 = 91 = \ell + 3$ bits. We name the 11 new carry witnesses after the lowest bit of $c_1$ they cover (counting bits from 0 to 90):

* **Introduce 11 witnesses**: $c_{1,0}$, $c_{1,12}$, $c_{1,24}$, $c_{1,36}$, $c_{1,48}$, $c_{1,60}$, $c_{1,72}$, $c_{1,84}$, $c_{1,86}$, $c_{1,88}$, $c_{1,90}$

$$
\begin{align}
\tag{12-bit lookups}
& c_{1,0}, c_{1,12}, c_{1,24}, c_{1,36}, c_{1,48}, c_{1,60}, c_{1,72} \in [0,2^{12}) \\
\tag{C7: 2-bit}
& c_{1,84}(c_{1,84}-1)(c_{1,84}-2)(c_{1,84}-3) = 0 \bmod n \quad \\
\tag{C8: 2-bit}
& c_{1,86}(c_{1,86}-1)(c_{1,86}-2)(c_{1,86}-3) = 0 \bmod n \quad \\
\tag{C9: 2-bit}
& c_{1,88}(c_{1,88}-1)(c_{1,88}-2)(c_{1,88}-3) = 0 \bmod n \quad \\
\tag{C10: 1-bit}
& c_{1,90}(c_{1,90}-1) = 0 \mod n \\
\end{align}
$$

In place of $c_1$, the gate uses the expression

$$
c_1 = \sum_i c_{1,i} 2^{i}
$$

so $c_1$ doesn't have to be stored as a witness.

By design, $c_1 \in [0, 2^{\ell + 3})$. In combination with our estimate on $p_2$ and RCs on $r_2$ and $p_{11} = p_{110} + 2^\ell p_{111}$, we can prove that $\text{(C6: high limb)}$ does not overflow $n$ and holds over the integers:

$$
|p_2 - r_2 + p_{11} + c_0 - 2^{\ell} c_1| < 6\cdot 2^{2\ell} + 2^{\ell} + 2^{\ell + 2} + 4 + 2^{2\ell + 3} < 2^{2\ell + 4} < n
$$

Therefore, we can plug in $\text{(C6)}$ into $(2'')$ to prove that

$$\tag{$2'''$}
a b - q f - r = 2^{3\ell} (w + c_1)
$$



By our first constraint $\text{(C1)}$, the LHS of $(2''')$ is a multiple of $n$, so we have


$$
2^{3\ell} (w + v_{1}) = 0 \mod{n}
$$

Because $2^{3\ell}$ and $n$ are co-prime, we can multiply with $2^{-3\ell}$ to see that


$$
w + v_{1} = 0 \mod{n}
$$

Therefore, we can write $w + v_{1} = \delta n$ for some $\delta \in \mathbb{Z}$. From $(2''')$ we conclude that

$$\tag{$2''''$}
a b - q f - r = 2^{3\ell} n \delta
$$

#### Final bound

There is one step remaining to make our gate sound: We must prove that $a b - q f - r$ cannot overflow $2^{3\ell} n$, so $\delta$ is in fact zero.

To do this, our strategy is to add an additional bound on the highest limbs of $a$, $b$, $q$. Skipping the lower limbs and $r$ is fine because they all contribute a negligible amount to the overall estimate.

For a variable $x = (x_0, x_1, x_2)$ define the _bounds check_ as the following two constraints:

* $x_2 + (2^\ell - f_2 - 1) = z \mod n$
* $z \in [0,2^\ell)$

We write this succinctly as $x_2 + (2^\ell - f_2 - 1) \in [0,2^\ell)$ but we must not forget that "+" is finite field addition.

For a field element $x_2 \in [0, n)$ the bounds check implies that

$$x_2 \in [0, f_2 + 1) \cup [n - 2^\ell + f_2 + 1, n).$$

If, in addition, $x_2$ is range-checked to be at most $\ell$ bits, the high interval is excluded:

$$x_2 \in [0, 2^\ell) \quad\text{and}\quad x_2 + (2^\ell - f_2 - 1) \in [0,2^\ell) \quad\Longrightarrow\quad x_2 \in [0, f_2 + 1)$$

> NB: It's important to use $x_2 + (2^\ell - f_2 - 1)$ and not $x_2 + (2^\ell - f_2)$ in the bounds check, so that $x_2$ can be at most $f_2$ and not $f_2 - 1$. Otherwise, the check would exclude valid foreign field elements for which $x_2 = f_2$.

For $x = (x_0, x_1, x_2)$, an $\ell$-bit range check on all limbs plus bounds check imply $x < 2^{2\ell}(f_2 + 1)$, as follows:

$$
x = 2^{2\ell} x_2 + 2^\ell x_1 + x_0  < 2^{2\ell} f_2 + 2^{2\ell} = 2^{2\ell}(f_2 + 1)
$$

This estimate is what we need for $a$, $b$ and $q$. We assume that bounds checks happen externally for $a$ and $b$:

$$
\begin{align}
\tag{Bounds check: $a$}
& a_2 + (2^\ell - f_2 - 1) \in [0,2^\ell) \\
\tag{Bounds check: $b$}
& b_2 + (2^\ell - f_2 - 1) \in [0,2^\ell) \\
\end{align}
$$

To help with the bounds check on $q$, we use some free space in the ffmul gate to expose $q'_2 := q_2 + (2^\ell - f_2 - 1)$ as a witness.

* **Introduce 1 witness**: $q'_2$

The addition is another constraint:

$$\tag{C11: bound for $q$}
q'_2 = q_2 + (2^\ell - f_2 - 1) \mod n
$$

And a range check on $q'_2$ completes the bounds check on $q$:

$$\tag{RC: bound for $q$}
q'_2 \in [0,2^\ell)
$$

This implies that $a,b,q \in[0, 2^{2\ell}(f_2 + 1))$. For $r$, we already know that $r \in [0, 2^{3\ell})$.

We now get the following upper bound and lower bound:

$$
ab - qf - r \le ab < 2^{4\ell}(f_2 + 1)^2
$$

$$
-ab + qf + r \le qf + r < 2^{4\ell}(f_2 + 1)^2 + 2^{3\ell}
$$

In summary, we have

$$
|ab - qf - r| < 2^{4\ell}(f_2 + 1)^2 + 2^{3\ell} = 2^{3\ell}\cdot(2^\ell (f_2 + 1)^2 + 1)
$$

In the case that $f < 2^{259}$, we have $(f_2 + 1) \le 2^{259 - 2\ell} = 2^{83}$, and our estimate works out to be

$$
|ab - qf - r| < 2^{3\ell} \cdot (2^{254} + 1)
$$

For $n > 2^{254}$, wich is true for both Pasta moduli, we conclude that

$$
|ab - qf - r| < 2^{3\ell}n
$$

and so $ab = qf + r$, which proves the soundness of the ffmul gate, when used together with the listed external checks.

### Gate layout

The following 14 witnesses have to be made available to other gates:

* The 6 input limbs $a_0, a_1, a_2$ and $b_0, b_1, b_2$
* The 5 output limbs $q_0, q_1, q_2$ and $r_{01}$, $r_2$
* Intermediate values $p_{10}$ and $p_{110}$ for the RC
* The high-limb qotient bound $q'_2$ for the RC

To make them available, they have to go in permutable cells. Kimchi has 7 permutable cells per row, so we exactly fit these values in the current and next row of a single-row gate.

| FFmul  | 0p       | 1p    | 2p    | 3p    | 4p    | 5p     | 6p        | 7   | 8   | 9   | 10  | 11  | 12  | 13  | 14  |
| ------ | -------- | ----- | ----- | ----- | ----- | ------ | --------- | --- | --- | --- | --- | --- | --- | --- | --- |
| _Curr_ | $a_0$    | $a_1$ | $a_2$ | $b_0$ | $b_1$ | $b_2$  | $p_{10}$  |     |     |     |     |     |     |     |     |
| _Next_ | $r_{01}$ | $r_2$ | $q_0$ | $q_1$ | $q_2$ | $q'_2$ | $p_{110}$ |     |     |     |     |     |     |     |     |

Another constraint on the gate layout is that we can only do 4 lookups per row, and we need 7 lookups on the carry chunks $c_{1,0},\ldots,c_{1,72}$. So these have to be divided up between the two rows.

The remaining witnesses are just put into any free cells to define the final gate layout:

| FFmul  | 0p       | 1p    | 2p    | 3p    | 4p    | 5p     | 6p        | 7          | 8          | 9          | 10         | 11         | 12         | 13         | 14         |
| ------ | -------- | ----- | ----- | ----- | ----- | ------ | --------- | ---------- | ---------- | ---------- | ---------- | ---------- | ---------- | ---------- | ---------- |
| _Curr_ | $a_0$    | $a_1$ | $a_2$ | $b_0$ | $b_1$ | $b_2$  | $p_{10}$  | $c_{1,0}$  | $c_{1,12}$ | $c_{1,24}$ | $c_{1,36}$ | $c_{1,84}$ | $c_{1,86}$ | $c_{1,88}$ | $c_{1,90}$ |
| _Next_ | $r_{01}$ | $r_2$ | $q_0$ | $q_1$ | $q_2$ | $q'_2$ | $p_{110}$ | $p_{111}$  | $c_{1,48}$ | $c_{1,60}$ | $c_{1,72}$ | $c_0$      |            |            |            |

The limbs of $f' = 2^{3\ell} - f$, which feature in a few of the constraints, are embedded into the gate as coefficients. The $\text{(C1)}$ constraint features $f$, but the gate implementation expands it out as $f = 2^{3\ell} - f' = 2^{3\ell} - (f_0' + 2^\ell f_1' + 2^{2\ell} f_2')$ to be able to use the same coefficients.

However, the $q$ bound $\text{(C11)}$ properly features $f_2$ in a way that can't be always replaced by the same expression involving $f_2'$, therefore $f_2$ is made a gate coefficient as well.

### FFmul gadget

As the soundness proof showed, the ffmul gate is not complete without a number of external checks. The job of an ffmul _gadget_ (a provable method which wraps the gate) is to take care of these checks and provide an API without pitfalls.

As usual, we want the gadget to add all necessary checks to its outputs $q$ and $r$ but not to its inputs $a$ and $b$ (because, if all gadgets fully constrained their inputs, then we would add these constraints multiple times if we used the same inputs in multiple gadgets, which can be very inefficient).

In addition, there should be an advanced flag to skip range checks on $r$, for use cases where $r$ is later asserted to equal a value which is already known to be in the range (for example, the constant 1 when constraining a foreign field inverse).

By default, we will do even more checks on $r$ than required in our soundness proof: We also add a bounds check for $r$ which shows that $r < 2^{2\ell}(f_2 + 1)$. This enables us to use $r$ as input to another ffmul gate or other foreign field operations without extra checks, which is what we expect from a safe API.

From these considerations, we propose the following logic of the ffmul gadget in pseudo-code:

```ts
function multiply(
  a: ForeignField,
  b: ForeignField,
  externalChecks: ExternalChecks,
  foreignModulus: bigint,
  skipRemainderCheck?: boolean
): {
  q: ForeignField;
  r: Option<ForeignField>;
  rCompact: [Field, Field];
} {
  // compute witnesses
  let { q, r01, r2, p10, p11, c0, c1, qBound } = exists(FFMulWitness, () => {
    /* ...*/
  });

  // add the ffmul gate
  addFFmulGate({ a, b, q, r01, r2, p10, p11, c0, c1, qBound }, foreignModulus);

  // add 3 range checks on limbs of q
  addMultiRangeCheck(q);

  // add single range check on q bound to an accumulator for external checks
  // (because these need to be in batches of 3 for efficiency)
  externalChecks.addRangeCheck(qBound);

  // if we're skipping the r checks, only return q and r in compact form
  if (skipRemainderCheck) {
    return { q, r: None, rCompact: [r01, r2] };
  }

  // split up the compact r01 limb to get witnesses for r range check
  let [r0, r1] = exists([Field, Field], () => splitCompactLimb(r01));
  let r = ForeignField.fromLimbs([r0, r1, r2]);

  // add range check on r (compact mode)
  addCompactRangeCheck(r, r01);

  // compute r bound using provable native field addition
  let [_f0, _f1, f2] = toLimbs(foreignModulus);
  let rBound = r2.add(2n ** limbSize - f2 - 1n);

  // add range check on r bound to external checks
  externalChecks.addRangeCheck(rBound);

  // return q, r, rCompact
  return { q, r: Some(r), rCompact: [r01, r2] };
}
```

### Spec

[WIP Kimchi PR with spec change](https://github.com/o1-labs/proof-systems/blob/046412a135079acf134640ef1d9d39c00117b17e/book/src/specs/kimchi.md#foreign-field-multiplication) (TODO: update)

## Test plan and functional requirements

This RFC builds on an existing implementation with plenty of tests. The test plan is to get the existing tests to run with the new layout. No additional tests are planned, but some existing tests will need to be adapted.

## Drawbacks

I'm not aware of drawbacks.

## Rationale and alternatives

We considered [two alternative gate designs similar to this one](https://hackmd.io/@mitschabaude/ByEVBlXKn), but everyone involved agreed that this one is the best.

An alternative not mentioned in that document would be to not split the middle limb, but instead have 3 limb-wise equations of the form

$$
p_i - r_i = 2^{\ell} c_i
$$

where $c_i$ is a carry of about $\ell$ bits. This would need 3 RCs on the 3 carries, the same amount as our current design uses for intermediate values. So efficiency-wise there is no difference. The advantage would be to make the gate equations more uniform and thus easier to reason about. However, I don't think this is worth changing the existing design.

Yet another alternative which is quite different from the present design is to not rely on the CRT at all. This means we constrain all 5 instead of just 3 limb-wise equations, because we're not taking the equation modulo $2^{3\ell}$ or $n$. A benefit is that we don't have to bounds-check $q$, and even don't require bounds checks on $a$ and $b$, and the entire argument becomes even simpler. This is balanced by the fact that a fourth range check becomes necessary on carry values, and we need more witnesses in total and so need three rows for the gate. However, the last row is not as stuffed as the second row in the present design, and so we'd be able to chain multiplications (output row of one gate = input row of next gate). Overall, the efficiency of this design is probably similar or slightly worse than the present one.

In summary, we chose the gate design presented in this RFC because:

- It's the most efficient design we came up with so far
- It's close to the original design and therefore easy to adapt

## Prior art

[Original ffmul RFC](https://o1-labs.github.io/proof-systems/rfcs/foreign_field_mul.html)

[Ariel Gabizon's write-up on CRT technique](https://hackmd.io/@arielg/B13JoihA8)

## Unresolved questions

Currently, there are no unresolved questions.
