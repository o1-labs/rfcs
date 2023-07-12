# Foreign field multiplication gate

**Changelog**

| Author(s) | Date | Details |
|-|-|-|
| Joseph Spadavecchia, Anaïs Querol | October 2022 | Initial Design of foreign field multiplication in Kimchi |
| Gregor Mitscha-Baude, Anaïs Querol | July 2023 | Revised version

## Summary

This document explains a foreign field multiplication (i.e. non-native field multiplication) custom gate (`ForeignFieldMul`) in the Kimchi proof system.

The gate constrains integers $a$, $b$ (inputs), $q$ (quotient) and $r$ (remainder) to satisfy

$$\tag{1}
a b = qf + r
$$

where $f$ (the modulus) is any number up to 259 bits. This means that $ab = r \bmod{f}$, so if $f$ is a prime we are constraining multiplication in the foreign field $\mathbb{F}_f$.

The revised ffmul gate is proved sound, which means that the constraints imposed by the gate, together with additional constraints that must be added alongside the gate, imply the intended relation $(1)$ in a strict mathematical sense.

## Motivation

Implementing foreign field multiplication as a custom gate is a requirement for efficiently supporting many important operations in provable code, such as ECDSA signature verification which unlocks interoperability with Ethereum.

Revising the existing Foreign Field RFC and implementation is necessary because we found soundness issues which allow a prover to provide incorrect values of $r$ for most inputs $a$ and $b$, leading to devastating attacks such as verifying (multiple different) invalid versions of a signature which is supposed to be deterministic. More details can be found in the [Appendix](#appendix-l-soundness).

This RFC optimizes for provable correctness while requiring minimal changes to the pre-existing design and implementation.

**Expected outcome**: An implementation of the `ForeignFieldMul` gate and supporting gadgets which we are confident in to be secure.

## Detailed design

This section is intended to be readable in a self-contained way. However, for a fuller picture, especially of the motivations of many choices made in the design, we recommend to read the [Appendix](#appendix-b-parameter-selection).

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

> In the following, we will use the numbering used in the implementation, which might not correspond to the chronological order where they appear explained here in the text.

Let $n$ be the _native_ field modulus (for example, $n$ is one of the Pasta primes). Our first constraint is to check $(1)$ modulo $n$:

$$\tag{C5: native}
ab = qf + r\mod n
$$

Note: In the implementation, variables in this constraint are expanded into their limbs, like

$$
ab = (a_0 + 2^\ell a_1 + 2^{2\ell} a_2)(b_0 + 2^\ell b_1 + 2^{2\ell} b_2).
$$

Equation $\text{(C5: native)}$ implies that there is an $\varepsilon\in\mathbb{Z}$ such that

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

This is a trick to avoid negative intermediate values (more can be found in the [Appendix](#appendix-c-borrows)). Note that $ab + qf'$ and all of its limbs are positive, and that since we will work modulo $2^{3\ell}$, we can ignore the extra term $2^{3\ell}q$.

Next, we expand our equation into limbs, but collect all terms that are multiples of $2^{3\ell}$ into a single term $2^{3\ell} w$.

$$
\begin{align*}
& a b - q f - r = \\
&  (a_0 b_0 + q_0 f_0') + 2^{\ell} (a_0 b_1 + a_1 b_0 + q_0 f_1' + q_1 f_0') - r_{01} \\
&+ 2^{2\ell} (a_0 b_2 + a_2 b_0 + q_0 f_2' + q_2 f_0' + a_1 b_1 + q_1 f_1' - r_2) \\
&+ 2^{3\ell} w
\end{align*}
$$

To make our equations easier to read, we define

$$
\begin{align*}
& p_0 := a_0b_0 + q_0f'_0 \\
& p_1 := a_0b_1 + a_1b_0 + q_0f'_1 + q_1f'_0 \\
& p_2 := a_0b_2 + a_2b_0 + q_0f'_2 + q_2f'_0 + a_1b_1 + q_1f'_1
\end{align*}
$$

> We don't introduce separate witnesses to hold $p_i$. Instead, the implementation will expand out the $p_i$ into their definitions wherever they are used in constraints. 

Now our limb-wise equation that holds over the integers (by definition of the terms involved) reads

$$\tag{2}
a b - q f - r = (p_0 + 2^{\ell} p_1 - r_{01}) + 2^{2\ell} (p_2 - r_2) + 2^{3\ell} w
$$

#### Splitting the middle limb

We want to constrain the right-hand side (RHS) of $(2)$ in parts that fit within the native modulus $n$ without overflow. The two parts we want to use are $(p_0 + 2^{\ell} p_1 - r_{01})$ and $(p_2 - r_2)$. The only problem with that is that $2^\ell p_1$ doesn't quite fit within $n$.

In fact, thanks to our range-checks, we know that a product such as $a_0 b_0$ satisfies $0 \le a_0 b_0 < 2^{\ell} \cdot 2^{\ell} = 2^{2\ell}$. This gives us estimates on $p_0, p_1, p_2$:

$$
\begin{align*}
& 0 \le p_0 = a_0b_0 + q_0f'_0 < 2 \cdot 2^{2\ell} \\
& 0 \le p_1 = a_0b_1 + a_1b_0 + q_0f'_1 + q_1f'_0 < 4 \cdot 2^{2\ell} \\
& 0 \le p_2 = a_0b_2 + a_2b_0 + q_0f'_2 + q_2f'_0 + a_1b_1 + q_1f'_1 < 6 \cdot 2^{2\ell}
\end{align*}
$$

Details about these upper bounds can be found in the [Appendix](#appendix-d-gate-constraints).

In particular, $p_1$ has at most $2\ell+2$ bits and so $2^\ell p_1$ might be larger than $2^{3\ell} > n$. This is the motivation to split $p_1$ into an $\ell$-bit bottom "half" and an $(\ell+2)$-bit top "half":

$$
p_1 = p_{10} + 2^\ell p_{11}.
$$

To facilitate RCs, the top half is further split into

$$
p_{11} = p_{110} + 2^\ell p_{111}
$$

where $p_{110}$ has $\ell$ bits and $p_{111}$ has the remaining 2 bits.

* **Introduce 3 witnesses**: $p_{10}$, $p_{110}$ and $p_{111}$

In summary, the split of $p_1$ gives us our constraint:

$$\tag{C3: split middle}
p_1 = p_{10} + 2^\ell p_{110} + 2^{2\ell} p_{111} \mod n
$$

Constraining $p_{111}$ to two bits is our third constraint:

$$\tag{C3: 2-bit $p_{111}$}
p_{111}(p_{111}-1)(p_{111}-2)(p_{111}-3) = 0 \mod n
$$

And we add two limb RCs that have to be performed externally:

$$
\begin{align}
\tag{RC $p_{1}$}
& p_{10}, p_{110} \in [0,2^\ell) \\
\end{align}
$$

Using these RCs and our earlier estimate on $p_1$, we can prove that $\text{(C3)}$ not only holds modulo $n$, but over the integers:

$$
|p_1 - p_{10} - 2^\ell p_{110} - 2^{2\ell} p_{111}| < 4\cdot2^{2\ell} + 2^{\ell} + 2^{2\ell} + 4\cdot 2^{2\ell} < n
$$

Therefore, we can use $\text{(C3)}$ to expand $p_1$ in our earlier equation $(2)$:

$$\tag{3}
a b - q f - r = (p_0 + 2^{\ell} p_{10} - r_{01}) + 2^{2\ell} (p_2 - r_2 + p_{11}) + 2^{3\ell} w
$$

#### Constraining limb-wise $\mod 2^{3\ell}$

Now both terms in brackets of $(3)$ fit within $n$. We add a constraint that the first bracket is zero, except for a possible carry $c_0$:

$$\tag{C4: bottom limbs}
p_0 + 2^\ell p_{01} - r_{01} = 2^{2\ell} c_0\mod n
$$

* **Introduce 1 witness**: $c_{0}$

Note first that we can constrain $c_0$ to be non-negative since $r_{01}$ is at most $2\ell$ bits and the lower $2\ell$ bits of the LHS are supposed to be zero. Let's estimate the LHS to compute how many bits $c_0$ can have:

$$
p_0 + 2^\ell p_{01} - r_{01} < 2^{2\ell + 1} + 2^{2\ell} - 0 < 2^{2\ell + 2}
$$

This shows that $c_0$ will have at most 2 bits, which motivates our next constraint:

$$\tag{C2: 2-bit $c_0$}
c_0(c_0-1)(c_0-2)(c_0-3) = 0 \mod n
$$

This constraint – together with our estimate of $p_0$ and RCs on $p_{01}$ and $r_{01}$ – allows us to prove that $\text{(C4)}$ doesn't overflow $n$ and holds over the integers:

$$
|p_0 + 2^\ell p_{01} - r_{01} - 2^{2\ell}c_0| < (2 + 1 + 1 + 4)\cdot 2^{2\ell} < n
$$

Therefore, we can use $\text{(C4)}$ in $(3)$ to eliminate the bottom part:

$$\tag{4}
a b - q f - r = 2^{2\ell} (p_2 - r_2 + p_{11} + c_0) + 2^{3\ell} w
$$

The next step is to constrain the $2^{2\ell}$ term, to be zero in its lower $\ell$ bits:

$$\tag{C10: top part}
p_2 - r_2 + p_{11} + c_0 = 2^{\ell} c_1\mod n
$$

Again, we introduce a carry value $c_1$ which is supposed to be positive, since $r_2$ has at most $\ell$ bits. Estimating the LHS shows us how many bits $c_1$ will at most have:

$$
p_2 - r_2 + p_{11} + c_0 < 6 \cdot 2^{2\ell} - 0 + 2^{\ell + 2} + 4 < 2^{2\ell + 3}
$$

We see that $c_1$ has at most $\ell + 3$ bits. To save rows and not fill more space among the permutable witness cells, we won't pass $c_1$ to a RC gate. Instead, we inline the entire RC into the `ForeignFieldMul` gate itself, using a combination of $\times 7$ 12-bit lookups plus $\times 3$ 2-bit constraints plus $\times 1$ 1-bit constraint, giving us $7 \cdot 12 + 3 \cdot 2 + 1 = 91 = \ell + 3$ bits. We name the 11 new carry witnesses after the lowest bit of $c_1$ they cover (counting bits from 0 to 90):

* **Introduce 11 witnesses**: $c_{1,0}$, $c_{1,12}$, $c_{1,24}$, $c_{1,36}$, $c_{1,48}$, $c_{1,60}$, $c_{1,72}$, $c_{1,84}$, $c_{1,86}$, $c_{1,88}$, $c_{1,90}$

$$
\begin{align}
\tag{12-bit lookups}
& c_{1,0}, c_{1,12}, c_{1,24}, c_{1,36}, c_{1,48}, c_{1,60}, c_{1,72} \in [0,2^{12}) \\
\tag{C6: 2-bit}
& c_{1,84}(c_{1,84}-1)(c_{1,84}-2)(c_{1,84}-3) = 0 \bmod n \quad \\
\tag{C7: 2-bit}
& c_{1,86}(c_{1,86}-1)(c_{1,86}-2)(c_{1,86}-3) = 0 \bmod n \quad \\
\tag{C8: 2-bit}
& c_{1,88}(c_{1,88}-1)(c_{1,88}-2)(c_{1,88}-3) = 0 \bmod n \quad \\
\tag{C9: 1-bit}
& c_{1,90}(c_{1,90}-1) = 0 \mod n \\
\end{align}
$$

In place of $c_1$, the gate uses the expression where $i\in\{0,12,36,48,60,72,84,86,88,90\}$, so that $c_1$ doesn't have to be stored as a witness.

$$
c_1 = \sum_i c_{1,i} 2^{i}
$$

By design, $c_1 \in [0, 2^{\ell + 3})$. In combination with our estimate on $p_2$ and RCs on $r_2$ and $p_{11} = p_{110} + 2^\ell p_{111}$, we can prove that $\text{(C10: top part)}$ does not overflow $n$ and holds over the integers:

$$
|p_2 - r_2 + p_{11} + c_0 - 2^{\ell} c_1| < 6\cdot 2^{2\ell} + 2^{\ell} + 2^{\ell + 2} + 4 + 2^{2\ell + 3} < 2^{2\ell + 4} < n
$$

Therefore, we can plug in $\text{(C10)}$ into $(4)$ to prove that

$$\tag{$5$}
a b - q f - r = 2^{3\ell} (w + c_1)
$$

By our native constraint $\text{(C5)}$, the LHS of $(5)$ is a multiple of $n$, so we have

$$
2^{3\ell} (w + v_{1}) = 0 \mod{n}
$$

Because $2^{3\ell}$ and $n$ are co-prime, we can multiply with $2^{-3\ell}$ to see that


$$
w + v_{1} = 0 \mod{n}
$$

Therefore, we can write $w + v_{1} = \delta n$ for some $\delta \in \mathbb{Z}$. From $(5)$ we conclude that

$$\tag{6}
a b - q f - r = 2^{3\ell} n \delta
$$

#### Final bound

There is one step remaining to make our gate sound: We must prove that $a b - q f - r$ cannot overflow $2^{3\ell} n$, so $\delta$ is in fact zero.

To do this, our strategy is to add an additional bound on the highest limbs of $a$, $b$, $q$. Skipping the lower limbs and $r$ is fine because they all contribute a negligible amount to the overall estimate.

For a variable $x = (x_0, x_1, x_2)$ define the _bounds check_ as the following two constraints:

* $x_2 + (2^\ell - f_2 - 1) = z \mod n$
* $z \in [0,2^\ell)$

We write this succinctly as $x_2 + (2^\ell - f_2 - 1) \in [0,2^\ell)$ but we must not forget that $+$ is finite field addition.

For a field element $x_2 \in [0, n)$ the bounds check implies that

$$x_2 \in [0, f_2] \cup [n - 2^\ell + f_2 + 1, n).$$

If, in addition, $x_2$ is range-checked to be at most $\ell$ bits, the high interval is excluded:

$$x_2 \in [0, 2^\ell) \quad\text{and}\quad x_2 + (2^\ell - f_2 - 1) \in [0,2^\ell) \quad\Longrightarrow\quad x_2 \in [0, f_2]$$

> NB: It's important to use $x_2 + (2^\ell - f_2 - 1)$ and not $x_2 + (2^\ell - f_2)$ in the bounds check, so that $x_2$ can be at most $f_2$ and not $f_2 - 1$. Otherwise, the check would exclude valid foreign field elements for which $x_2 = f_2$.

For $x = (x_0, x_1, x_2)$, an $\ell$-bit range check on all limbs plus bounds check imply $x < 2^{2\ell}(f_2 + 1)$, as follows:

$$
x = 2^{2\ell} x_2 + 2^\ell x_1 + x_0  < 2^{2\ell} f_2 + 2^{2\ell} = 2^{2\ell}(f_2 + 1)
$$

This estimate is what we need for $a$, $b$ and $q$. We assume that bounds checks happen externally for $a$ and $b$:

$$
\begin{align}
\tag{High limb bound check: $a$}
& a_2 + (2^\ell - f_2 - 1) \in [0,2^\ell) \\
\tag{High limb bound check: $b$}
& b_2 + (2^\ell - f_2 - 1) \in [0,2^\ell) \\
\end{align}
$$

To help with the bounds check on $q$, we use some free space in the `ForeignFieldMul` gate to expose $q'_2 := q_2 + (2^\ell - f_2 - 1)$ as a witness.

* **Introduce 1 witness**: $q'_2$

The addition is another constraint:

$$\tag{C11: high bound for $q$}
q'_2 = q_2 + (2^\ell - f_2 - 1) \mod n
$$

And a limb range check on $q'_2$ completes the bounds check on $q$:

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

and so $ab = qf + r$, which proves the soundness of the `ForeignFieldMul` gate, when used together with the listed external checks.

> Note: We showed that the simple conditions $n > 2^{254}, f < 2^{259}$ are sufficient for our estimates. For any specific $n > 2^{254}$, the maximum allowed value for $f$ is actually a bit larger than $2^{259}$. As the argument above shows, a more precise condition on $f$ is $2^\ell (f_2 + 1)^2 < n$. In the case of the Pasta moduli $n$, $f$ can just be a tiny bit larger than $2^{259}$. For ease of communicating the maximum, and because primes slightly larger than 256 bits don't have any practical use as far as I can tell, I propose to make $f = 2^{259}-1$ the officially documented maximum.

### Gate layout

The following 14 witnesses have to be made available to other gates:

* The 6 input limbs $a_0, a_1, a_2$ and $b_0, b_1, b_2$
* The 5 output limbs $q_0, q_1, q_2$ and $r_{01}$, $r_2$
* Intermediate values $p_{10}$ and $p_{110}$ for the RC
* The high-limb qotient bound $q'_2$ for the RC

To make them available, they have to go in permutable cells. Kimchi has 7 permutable cells per row, so we exactly fit these values in the current and next row of a single-row gate.


| FFmul  | 0p | 1p | 2p | 3p | 4p | 5p | 6p |
| -------| -- | -- | -- | -- | -- | -- | -- |
| _Curr_ | $a_0$ | $a_1$ | $a_2$ | $b_0$ | $b_1$ | $b_2$ | $p_{10}$ |
| _Next_ | $r_{01}$ | $r_2$ | $q_0$ | $q_1$ | $q_2$ | $q'_2$ | $p_{110}$ |  |  |  |  |  |  |  | 

Another constraint on the gate layout is that we can only do 4 lookups per row, and we need 7 lookups on the carry chunks $c_{1,0},\ldots,c_{1,72}$. So these have to be divided up between the two rows. In order to reuse the same `LookupPattern` in both rows of the gadget, we must place a value that is known to be less than 12 bits in the same column of the `Next` row (or leave it empty with a zero).

The remaining witnesses are just put into any free cells to define the final gate layout. 

| FFmul  | 0p | 1p | 2p | 3p | 4p | 5p | 6p | 7  | 8  | 9  | 10 | 11 | 12 | 13 | 14 |
| -------| -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- |
| _Curr_ | $a_0$ | $a_1$ | $a_2$ | $b_0$ | $b_1$ | $b_2$ | $p_{10}$ | $c_{1,0}$ | $c_{1,12}$ | $c_{1,24}$ | $c_{1,36}$ | $c_{1,84}$ | $c_{1,86}$ | $c_{1,88}$ | $c_{1,90}$ | 
| _Next_ | $r_{01}$ | $r_2$ | $q_0$ | $q_1$ | $q_2$ | $q'_2$ | $p_{110}$ | $p_{111}$ | $c_{1,48}$ | $c_{1,60}$ | $c_{1,72}$ | $c_0$ |  |  |  | 

The limbs of $f' = 2^{3\ell} - f$, which feature in a few of the constraints, are embedded into the gate as coefficients. The $\text{(C1)}$ constraint features $f$, but the gate implementation expands it out as $f = 2^{3\ell} - f' = 2^{3\ell} - (f_0' + 2^\ell f_1' + 2^{2\ell} f_2')$ to be able to use the same coefficients.

However, the $q$ bound $\text{(C11)}$ properly features $f_2$ in a way that can't be always replaced by the same expression involving $f_2'$, therefore $f_2$ is made a gate coefficient as well.

### `ForeignFieldMul` gadget

As the soundness proof shows, the `ForeignFieldMul` gate is not complete without a number of external checks. The job of an `ForeignFieldMul` _gadget_ (a provable method which wraps the gate) is to take care of these checks and provide an API without pitfalls.

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

[WIP Kimchi PR with spec change](https://github.com/o1-labs/proof-systems/blob/ffmul/rangecheck/book/src/specs/kimchi.md#foreign-field-multiplication) 
(TODO: update)

## Test plan and functional requirements

This RFC builds on an existing implementation with plenty of tests. The test plan is to get the existing tests to run with the new layout. Planning to test against the counter example used in the soundness proof to attest for the necessity of the new range checks.

## Drawbacks

Due to the limited number of cells accessible to gates, we are not able to chain multiplications into multiplications. We can chain foreign field additions into foreign field multiplications, but currently do not support chaining multiplications into additions (though there is a way to do it, using a completely different approach towards multiplication).

Note that our proposed `ForeignFieldMul` design obtains a correct result in the class of $r+k\cdot f$, meaning that it becomes the canonical $r$ when performing modulo $f$. But when this gadget is being used in situations where only the canonical element is valid, then a [full bound check](#appendix-f-full-bound-checks) needs to be carried out in order to make sure that $r < f$. This step involves performing one `ForeignFieldAdd`, one `Zero`, and one multi range check. Nonetheless, taking this tradeoff for the gate designer results in major performance improvements in a larger circuit with intense `ForeignFieldMul` presence.

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

[Ariel Gabizon's write-up on CRT technique](https://hackmd.io/@arielg/B13JoihA8)

## Unresolved questions

Currently, there are no unresolved questions.

> The following long sections describe the mathematical background in fine detail behind the `ForeignFieldMul` gate. This is meant to motivate some choices being made throughout the design, and be a resource of information for the curious reader.

## Appendix A: Limb representation

In foreign field multiplication the foreign field modulus $f$ could be bigger or smaller than the native field modulus $n$.  When the foreign field modulus is bigger, then we need to emulate foreign field multiplication by splitting the foreign field elements up into limbs that fit into the native field element size.  When the foreign modulus is smaller everything can be constrained either in the native field or split up into limbs.

Since our projected use cases are when the foreign field modulus is bigger (more on this below) we optimize our design around this scenario. For this case, not only must we split foreign field elements up into limbs that fit within the native field, but we must also operate in a space bigger than the foreign field. This is because we check the multiplication of two foreign field elements by constraining its quotient and remainder.

$$
a \cdot b = q \cdot f + r,
$$

where the maximum size of $q$ and $r$ is $f - 1$ and so we have

$$
\begin{aligned}
a \cdot b &\le \underbrace{(f - 1)}_q \cdot f + \underbrace{(f - 1)}_r \\
&\le f^2 - 1.
\end{aligned}
$$

Thus, the maximum size of the multiplication is quadratic in the size of foreign field.

### Naïve approach

Naïvely, this implies that we have elements of size $f^2 - 1$ that must split them up into limbs of size at most $n - 1$.  For example, if the foreign field modulus is $256$ and the native field modulus is $255$ bits, then we'd need $\log_2((2^{256})^2 - 1) \approx 512$ bits and, thus, require $512/255 \approx 3$ native limbs.  However, each limb cannot consume all $255$ bits of the native field element because we need space to perform arithmetic on the limbs themselves while constraining the foreign field multiplication.  Therefore, we need to choose a limb size that leaves space for performing these computations.

Later in this document (see the section entitled "Choosing the limb configuration") we determine the optimal number of limbs that reduces the number of rows and gates required to constrain foreign field multiplication.  This results in $\ell = 88$ bits as our optimal limb size.  In the section about intermediate products we place some upperbounds on the number of bits required when constraining foreign field multiplication with limbs of size $\ell$ thereby proving that the computations can fit within the native field size.

Observe that by combining the naïve approach above with a limb size of $88$ bits, we would require $512/88 \approx 6$ limbs for representing foreign field elements.  Each limb is stored in a witness cell (a native field element).  However, since each limb is necessarily smaller than the native field element size, it must be copied to the range-check gate to constrain its value.  Since Kimchi only supports 7 copyable witness cells per row, this means that only one foreign field element can be stored per row.  This means a single foreign field multiplication would consume at least 4 rows (just for the operands, quotient and remainder).  This is not ideal because we want to limit the number of rows for improved performance.

### Chinese remainder theorem

Fortunately, we can do much better than this, however, by leveraging the chinese remainder theorem (CRT for short) as we will now show.  The idea is to reduce the number of bits required by constraining our multiplication modulo two coprime moduli: $2^t$ and $n$.  By constraining the multiplication with both moduli the CRT guarantees that the constraints hold with the bigger modulus $2^t \cdot n$.

The advantage of this approach is that constraining with the native modulus $n$ is very fast, allowing us to speed up our non-native computations.  This practically reduces the costs to constraining with limbs over $2^t$, where $t$ is something much smaller than $512$.

For this to work, we must select a value for $t$ that is big enough.  That is, we select $t$ such that $2^t \cdot n = M > f^2 - 1$. By doing so we reduce the number of bits required for our use cases to $264$. With $88$ bit limbs this means that each foreign field element only consumes $3$ witness elements, and that means the foreign field multiplication gate now only consumes $2$ rows. The section entitled [Binary modulus choice](#binary-modulus-choice) describes this in more detail.

### Overall approach

Bringing it all together, our approach is to verify that

$$\tag{A.1}
\begin{align}
a \cdot b = q \cdot f + r
\end{align}
$$

over $\mathbb{Z^+}$.  In order to do this efficiently we use the CRT, which means that the equation holds mod $M = 2^t \cdot n$.  For the equation to hold over the integers we must also check that each side of the equation is less than $2^t \cdot n$.

The overall steps are therefore

1. Apply CRT to equation (A.1)
    * Check validity with binary modulus $\mod 2^t$
    * Check validity with native modulus $\mod n$
2. Check each side of equation (1) is less than $M$
    * $a \cdot b < 2^t \cdot n$
    * $q \cdot f + r < 2^t \cdot n$

This then implies that

$$
a \cdot b = c \mod f.
$$

where $c = r$. That is, $a$ multiplied with $b$ is equal to $c$ where $a,b,c \in \mathbb{F_f}$.  

#### More steps

Within our overall approach, aside from the CRT, we also use a number of other strategies to reduce the number and degree of constraints.

* Avoiding borrows and carries
* Intermediate product elimination
* Combining multiplications

## Appendix B: Parameter selection

This section describes important parameters that we require and how they are computed.

* *Native field modulus* $n$
* *Foreign field modulus* $f$
* *Binary modulus* $2^t$
* *CRT modulus* $2^t \cdot n$
* *Limb size* in bits $\ell$

### Binary modulus choice

Under the hood, we constrain $a \cdot b = q \cdot f + r \mod 2^t \cdot n$.  Since this is a multiplication in the foreign field $f$, the maximum size of $q$ and $r$ are $f - 1$, so this means

$$
\begin{aligned}
a \cdot b &\le (f - 1) \cdot f + (f - 1) \\
&\le f^2 - 1.
\end{aligned}
$$

Therefore, we need the modulus $2^t \cdot n$ such that

$$
2^t \cdot n > f^2 - 1,
$$

which is the same as saying, given $f$ and $n$, we must select $t$ such that

$$
\begin{aligned}
2^t \cdot n &\ge f^2 \\
t &\ge 2\log_2(f) - \log_2(n).
\end{aligned}
$$

Thus, we have a lower bound on $t$.

Instead of dynamically selecting $t$ for every $n$ and $f$ combination, we fix a $t$ that will work for the different selections of $n$ and $f$ relevant to our use cases.

To guide us, from above we know that

$$
t_{min} = 2\log_2(f) - \log_2(n)
$$

and we know the field moduli for our immediate use cases.

```
vesta     = 2^254 + 45560315531506369815346746415080538113 (255 bits)
pallas    = 2^254 + 45560315531419706090280762371685220353 (255 bits)
secp256k1 = 2^256 - 2^32 - 2^9 - 2^8 - 2^7 - 2^6 - 2^4 - 1 (256 bits)
```

So we can create a table

| $n$         | $f$             | $t_{min}$ |
| ----------- | --------------- | --- |
| `vesta`     | `secp256k1`     | 258 |
| `pallas`    | `secp256k1`     | 258 |
| `vesta`     | `pallas`        | 255 |
| `pallas`    | `vesta`         | 255 |

and know that to cover our use cases we need $t \ge 258$.

Next, given our native modulus $n$ and $t$, we can compute the *maximum foreign field modulus supported*.  Actually, we compute the maximum supported bit length of the foreign field modulus $f=2^m$.

$$
\begin{aligned}
2^t \cdot n &\ge f^2 \\
&\ge (2^m)^2 = 2^{2m} \\
t + \log_2(n) &> \log_2(2^{2m}) = 2m
\end{aligned}
$$

So we get

$$
m_{max} = \frac{t + \log_2(n)}{2}.
$$

With $t=258, n=255$ we have 

$$
\begin{aligned}
m_{max} &= \frac{258 + 255}{2} = 256.5,
\end{aligned}
$$

which is not enough space to handle anything larger than 256 bit moduli.  Instead, we will use $t=264$, giving $m_{max} = 259$ bits.

The above formula is useful for checking the maximum number of bits supported of the foreign field modulus, but it is not useful for computing the maximum foreign field modulus itself (because $2^{m_{max}}$ is too coarse). For these checks, we can compute our maximum foreign field modulus more precisely with

$$
max_{mod} = \lfloor \sqrt{2^t \cdot n} \rfloor.
$$

The max prime foreign field modulus satisfying the above inequality for both Pallas and Vesta is `926336713898529563388567880069503262826888842373627227613104999999999999999607`.

### Limb configuration

Choosing the right limb size and the right number of limbs is a careful balance between the number of constraints (i.e. the polynomial degree) and the witness length (i.e. the number of rows).  Because one limiting factor that we have in Kimchi is the 12-bit maximum for range check lookups, we could be tempted to introduce 12-bit limbs.  However, this would mean having many more limbs, which would consume more witness elements and require significantly more rows.  It would also increase the polynomial degree by increasing the number of constraints required for the *intermediate products* (more on this later).

We need to find a balance between the number of limbs and the size of the limbs.  The limb configuration is dependent on the value of $t$ and our maximum foreign modulus (as described in the previous section).  The larger the maximum foreign modulus, the more witness rows we will require to constrain the computation.  In particular, each limb needs to be constrained by the range check gate and, thus, must be in a copyable (i.e. permuteable) witness cell.  We have at most 7 copyable cells per row and gates can operate on at most 2 rows, meaning that we have an upperbound of at most 14 limbs per gate (or 7 limbs per row).

As stated above, we want the foreign field modulus to fit in as few rows as possible and we need to constrain operands $a, b$, the quotient $q$ and remainder $r$. Each of these will require cells for each limb.  Thus, the number of cells required for these is

$$
cells = 4 \cdot limbs
$$

It is highly advantageous for performance to constrain foreign field multiplication with the minimal number of gates. This not only helps limit the number of rows, but also to keep the gate selector polynomial small. Because of all of this, we aim to constrain foreign field multiplication with a single gate (spanning at most $2$ rows). As mentioned above, we have a maximum of 14 permuteable cells per gate, so we can compute the maximum number of limbs that fit within a single gate like this.

$$
\begin{aligned}
limbs_{max} &= \lfloor cells/4  \rfloor \\
      &= \lfloor 14/4 \rfloor \\
      &= 3 \\
\end{aligned}
$$

Thus, the maximum number of limbs possible in a single gate configuration is 3.

Using $limbs_{max}=3$ and $t=264$ that covers our use cases (see the previous section), we are finally able to derive our limbs size

$$
\begin{aligned}
\ell &= \frac{t}{limbs_{max}} \\
&= 264/3 \\
&= 88
\end{aligned}
$$

Therefore, our limb configuration is:

  * Limb size $\ell = 88$ bits
  * Number of limbs is $3$

## Appendix C: Borrows

When we constrain $a \cdot b - q \cdot f = r \mod 2^t$ we want to be as efficient as possible.

Observe that the expansion of $a \cdot b - q \cdot f$ into limbs would also have subtractions between limbs, requiring our constraints to account for borrowing.  Dealing with this would create undesired complexity and increase the degree of our constraints.

In order to avoid the possibility of subtractions we instead use $a \cdot b + q \cdot f'$ where

$$
\begin{aligned}
f' &= -f \mod 2^t \\
   &= 2^t - f
\end{aligned}
$$

The negated modulus $f'$ becomes part of our gate coefficients and is not constrained because it is publicly auditable.

Using the substitution of the negated modulus, we now must constrain $a \cdot b + q \cdot f' = r \mod 2^t$.

> Observe that $f' < 2^t$ since $f < 2^t$ and that $f' > f$ when $f < 2^{t - 1}$.

### Subtractions

When unconstrained, computing $u_0 = p_0 + 2^{\ell} \cdot p_{10} - (r_0 + 2^{\ell} \cdot r_1)$ could require borrowing.  Fortunately, we have the constraint that the $2\ell$ least significant bits of $u_0$ are `0` (i.e. $u_0 = 2^{2\ell} \cdot v_0$), which means borrowing cannot happen.

Borrowing is prevented because when the the $2\ell$ least significant bits of the result are `0` it is not possible for the minuend to be less than the subtrahend.  We can prove this by contradiction.

Let

* $x = p_0 + 2^{\ell} \cdot p_{10}$
* $y = r_0 + 2^{\ell} \cdot r_1$
* the $2\ell$ least significant bits of $x - y$ be `0`

Suppose that borrowing occurs, that is, that $x < y$.  Recall that the length of $x$ is at most $2\ell + 2$ bits.  Therefore, since $x < y$ the top two bits of $x$ must be zero and so we have

$$
x - y = x_{2\ell} - y,
$$

where $x_{2\ell}$ denotes the $2\ell$ least significant bits of $x$.

Recall also that the length of $y$ is $2\ell$ bits.  We know this because limbs of the result are each constrained to be in $[0, 2^{\ell})$.  So the result of this subtraction is $2\ell$ bits.   Since the $2\ell$ least significant bits of the subtraction are `0` this means that

$$
\begin{aligned}
x - y  &  = 0 \\
x &= y,
\end{aligned}
$$

which is a contradiction with $x < y$.

## Appendix D: Gate constraints

This section explains how we expand our constraints into limbs and then eliminate a number of extra terms.

### Intermediate products

We must constrain $a \cdot b + q \cdot f' = r \mod 2^t$ on the limbs, rather than as a whole.  As described above, each foreign field element $x$ is split into three 88-bit limbs: $x_0, x_1, x_2$, where $x_0$ contains the least significant bits and $x_2$ contains the most significant bits and so on.

Expanding the right-hand side into limbs we have

$$
\begin{aligned}
&(a_0 + a_1 \cdot 2^{\ell} + a_2 \cdot 2^{2\ell}) \cdot (b_0 + b_1 \cdot 2^{\ell} + b_2 \cdot 2^{2\ell}) +  (q_0 + q_1 \cdot 2^{\ell} + q_2 \cdot 2^{2\ell}) \cdot (f'_0 + f'_1 \cdot 2^{\ell} + f'_2 \cdot 2^{2\ell}) \\
&=\\
&~~~~~ a_0 \cdot b_0 + a_0 \cdot b_1 \cdot 2^{\ell} + a_0 \cdot b_2 \cdot 2^{2\ell} \\
&~~~~ + a_1 \cdot b_0 \cdot 2^{\ell} + a_1 \cdot b_1 \cdot 2^{2\ell} + a_1 \cdot b_2 \cdot 2^{3\ell} \\
&~~~~ + a_2 \cdot b_0 \cdot 2^{2\ell} + a_2 \cdot b_1 \cdot 2^{3\ell} + a_2 \cdot b_2 \cdot 2^{4\ell} \\
&+ \\
&~~~~~ q_0 \cdot f'_0 + q_0 \cdot f'_1 \cdot 2^{\ell} + q_0 \cdot f'_2 \cdot 2^{2\ell} \\
&~~~~ + q_1 \cdot f'_0 \cdot 2^{\ell} + q_1 \cdot f'_1 \cdot 2^{2\ell} + q_1 \cdot f'_2 \cdot 2^{3\ell} \\
&~~~~ + q_2 \cdot f'_0 \cdot 2^{2\ell} + q_2 \cdot f'_1 \cdot 2^{3\ell} + q_2 \cdot f'_2 \cdot 2^{4\ell} \\
&= \\
&a_0 \cdot b_0 + q_0 \cdot f'_0 \\
&+ 2^{\ell} \cdot (a_0 \cdot b_1 + a_1 \cdot b_0 + q_0 \cdot f'_1 + q_1 \cdot f'_0) \\
&+ 2^{2\ell} \cdot (a_0 \cdot b_2 + a_2 \cdot b_0 + q_0 \cdot f'_2 + q_2 \cdot f'_0 + a_1 \cdot b_1 + q_1 \cdot f'_1) \\
&+ 2^{3\ell} \cdot (a_1 \cdot b_2 + a_2 \cdot b_1 + q_1 \cdot f'_2 + q_2 \cdot f'_1) \\
&+ 2^{4\ell} \cdot (a_2 \cdot b_2 + q_2 \cdot f'_2) \\
\end{aligned}
$$

Since $t = 3\ell$, the terms scaled by $2^{3\ell}$ and $2^{4\ell}$ are a multiple of the binary modulus and, thus, congruent to zero $\mod 2^t$. They can be eliminated and we don't need to compute them.  So we are left with 3 *intermediate products* that we call $p_0, p_1, p_2$:

| Term  | Scale       | Product                                                  |
| ----- | ----------- | -------------------------------------------------------- |
| $p_0$ | $1$         | $a_0b_0 + q_0f'_0$                                       |
| $p_1$ | $2^{\ell}$  | $a_0b_1 + a_1b_0 + q_0f'_1 + q_1f'_0$                    |
| $p_2$ | $2^{2\ell}$ | $a_0b_2 + a_2b_0 + q_0f'_2 + q_2f'_0 + a_1b_1 + q_1f'_1$ |

So far, we have introduced these checked computations to our constraints

> 1. Computation of $p_0, p_1, p_2$

### Constraining $\mod 2^t$

Let's call $p := ab + qf' \mod 2^t$. Remember that our goal is to constrain that $p - r = 0 \mod 2^t$ (recall that any more significant bits than the 264th are ignored in $\mod 2^t$). Decomposing that claim into limbs, that means

$$\tag{D.1}
\begin{align}
2^{2\ell}(p_2 - r_2) + 2^{\ell}(p_1 - r_1) + p_0 - r_0 = 0 \mod 2^t.
\end{align}
$$

We face two challenges

*  Since $p_0, p_1, p_2$ are at least $2^{\ell}$ bits each, the right side of the equation above does not fit in $\mathbb{F}_n$
*  The subtraction of the remainder's limbs $r_0$ and $r_1$ could require borrowing

For the moment, let's not worry about the possibility of borrows and instead focus on the first problem.

### Combining multiplications

The first problem is that our native field is too small to constrain $2^{2\ell}(p_2 - r_2) + 2^{\ell}(p_1 - r_1) + p_0 - r_0 = 0 \mod 2^t$. We could split this up by multiplying $a \cdot b$ and $q \cdot f'$ separately and create constraints that carefully track borrows and carries between limbs. However, a more efficient approach is combined the whole computation together and accumulate all the carries and borrows in order to reduce their number.

The trick is to assume a space large enough to hold the computation, view the outcome in binary and then split it up into parts that fit in the native modulus.

To this end, it helps to know how many bits these intermediate products require. On the left side of the equation, $p_0$  is at most $2\ell + 1$ bits. We can compute this by substituting the maximum possible binary values (all bits set to 1) into $p_0 = a_0 \cdot b_0 + q_0 \cdot f'_0$ like this

$$
\begin{aligned}
\mathsf{maxbits}(p_0) &= \log_2(\underbrace{(2^{\ell} - 1)}_{a_{0}} \underbrace{(2^{\ell} - 1)}_{b_{0}} + \underbrace{(2^{\ell} - 1)}_{q_{0}} \underbrace{(2^{\ell} - 1)}_{f'_{0}}) \\
&= \log_2(2(2^{2\ell} - 2^{\ell + 1} + 1)) \\
&= \log_2(2^{2\ell + 1} - 2^{\ell + 2} + 2).
\end{aligned}
$$

So $p_0$ fits in $2\ell + 1$ bits.  Similarly, $p_1$ needs at most $2\ell + 2$ bits and $p_2$ is at most $2\ell + 3$ bits.


| Term     | $p_0$       | $p_1$       | $p_2$       |
| -------- | ----------- | ----------- | ----------- |
| **Bits** | $2\ell + 1$ | $2\ell + 2$ | $2\ell + 3$ |

The diagram below shows the right hand side of the zero-sum equality from equation (2).  That is, the value $p - r$. Let's look at how the different bits of $p_0, p_1, p_2, r_0, r_1$ and $r_2$ impact it.

```text
0             L             2L            3L            4L
|-------------|-------------|-------------|-------------|-------------|
                            :
|--------------p0-----------:-| 2L + 1
                            :
              |-------------:-p1------------| 2L + 2
                    p10➚    :        p11➚
                            |----------------p2-------------| 2L + 3
                            :
|-----r0------|             :
                            :
              |-----r1------|
                            :
                            |-----r2------|
\__________________________/ \______________________________/
             ≈ h0                           ≈ h1
```

Within our native field modulus we can fit up to $2\ell + \delta < \log_2(n)$ bits, for small values of $\delta$ (but sufficient for our case).  Thus, we can only constrain approximately half of $p - r$ at a time. In the diagram above the vertical line at 2L bisects $p - r$ into two $\approx2\ell$ bit values: $h_0$ and $h_1$ (the exact definition of these values follow).  Our goal is to constrain $h_0$ and $h_1$.

### Computing the zero-sum halves

Now we can derive how to compute $h_0$ and $h_1$ from $p$ and $r$.

The direct approach would be to bisect both $p_0$ and $p_1$ and then define $h_0$ as just the sum of the $2\ell$ lower bits of $p_0$ and $p_1$ minus $r_0$ and $r_1$.  Similarly $h_1$ would be just the sum of upper bits of $p_0, p_1$ and $p_2$ minus $r_2$.  However, each bisection requires constraints for the decomposition and range checks for the two halves.  Thus, we would like to avoid bisections as they are expensive.

Ideally, if our $p$'s lined up on the $2\ell$ boundary, we would not need to bisect at all.  However, we are unlucky and it seems like we must bisect both $p_0$ and $p_1$.  Fortunately, we can at least avoid bisecting $p_0$ by allowing it to be summed into $h_0$ like this

$$
h_0 = p_0 + 2^{\ell}\cdot p_{10} - r_0 - 2^{\ell}\cdot r_1
$$

In particular, this is the only place where the middle and low limbs of the remainder are used in constraints. For space reasons, this leads us to provide the witness values of the remainder's limbs in _compact_ form, meaning, as $r_{01}$ and $r_2$ such that $r_{01} = r_{0} + 2^\ell \cdot r_{1}$. Then, the exact limbs of the result will be obtained from the corresponding multi range check cells.

Note that $h_0$ is actually greater than $2\ell$ bits in length.  This may not only be because it contains $p_0$ whose length is $2\ell + 1$, but also because adding $p_{10}$ may cause an overflow.  The maximum length of $h_0$ is computed by substituting in the maximum possible binary value of $2^{\ell} - 1$ for the added terms and $0$ for the subtracted terms of the above equation.

$$
\begin{aligned}
\mathsf{maxbits}(h_0) &= \log_2(\underbrace{(2^{\ell} - 1)(2^{\ell} - 1) + (2^{\ell} - 1)(2^{\ell} - 1)}_{p_0} + 2^{\ell} \cdot \underbrace{(2^{\ell} - 1)}_{p_{10}}) \\
&= \log_2(2^{2\ell + 1} - 2^{\ell + 2} + 2 + 2^{2\ell} - 2^\ell) \\
&= \log_2( 3\cdot 2^{2\ell} - 5 \cdot 2^\ell +2 ) \\
\end{aligned}
$$

which is $2\ell + 2$ bits.

>This computation assumes correct sizes values for $r_0$ and $r_1$, which we assure by range checks on the limbs.

Next, we compute $h_1$ as

$$
h_1 = p_{11} + p_2 - r_2
$$

The maximum size of $h_1$ is computed as

$$
\begin{aligned}
\mathsf{maxbits}(h_1) &= \mathsf{maxbits}(p_{11} + p_2)
\end{aligned}
$$

In order to obtain the maximum value of $p_{11}$, we define $p_{11} := \frac{p_1}{2^\ell}$. Since the maximum value of $p_1$ was $2^{2\ell+2}-2^{\ell+3}+4$, then the maximum value of $p_{11}$ is $2^{\ell+2}-8$. For $p_2$, the maximum value was $6\cdot 2^{2\ell} - 12 \cdot 2^\ell + 6$, and thus:

$$
\begin{aligned}
\mathsf{maxbits}(h_1) &= log_2(\underbrace{2^{\ell+2}-8}_{p_{11}} + \underbrace{6\cdot 2^{2\ell} - 12 \cdot 2^\ell + 6}_{p_2}) \\
&= \log_2(6\cdot 2^{2\ell} - 8 \cdot 2^\ell - 2) \\
\end{aligned}
$$

which is $2\ell + 3$ bits.

| Term     | $h_0$       | $h_1$       |
| -------- | ----------- | ----------- |
| **Bits** | $2\ell + 2$ | $2\ell + 3$ |

Thus far we have the following constraints
> 2. Composition of $p_{10}$ and $p_{11}$ result in $p_1$
> 3. Range check $p_{11} \in [0, 2^{\ell + 2})$
> 4. Range check $p_{10} \in [0, 2^{\ell})$

For the next step we would like to constrain $h_0$ and $h_1$ to zero.  Unfortunately, we are not able to do this!

* Firstly, as defined $h_0$ may not be zero because we have not bisected it precisely at $2\ell$ bits, but have allowed it to contain the full $2\ell + 2$ bits.  Recall that these two additional bits are because $p_0$ is at most $2\ell + 1$ bits long, but also because adding $p_{10}$ increases it to $2\ell + 2$.  These two additional bits are not part of the first $2\ell$ bits of $p - r$ and, thus, are not necessarily zero.  That is, they are added from the second $2\ell$ bits (i.e. $h_1$).

* Secondly, whilst the highest $\ell + 3$ bits of $p - r$ would wrap to zero $\mod 2^t$, when placed into the smaller $2\ell + 3$ bit $h_1$ in the native field, this wrapping does not happen.  Thus, $h_1$'s $\ell + 3$ highest bits may be nonzero.

We can deal with this non-zero issue by computing carry witness values.

### Computing carry witnesses values

Instead of constraining $h_0$ and $h_1$ to zero, there must be satisfying witness $v_0$ and $v_1$ such that the following constraints hold.
> 5. There exists $v_0$ such that $h_0 = v_0 \cdot 2^{2\ell}$
> 6. There exists $v_1$ such that $h_1 = v_1 \cdot 2^{\ell} - v_0$

Here $v_0$ is the last two bits of $h_0$'s $2\ell + 2$ bits, i.e., the result of adding the highest bit of $p_0$ and any possible carry bit from the operation of $h_0$. Similarly, $v_1$ corresponds to the highest $\ell + 3$ bits of $h_1$.  It looks like this

```text
0             L             2L            3L            4L
|-------------|-------------|-------------|-------------|-------------|
                            :
|--------------h0-----------:--| 2L + 2
                            : ↖v0
                            :-------------h1-------------| 2L + 3
                            :              \____________/
                            :                  v1➚
```


Remember we only need to prove the first $3\ell$ bits of $p - r$ are zero, since everything is $\mod 2^t$ and  $t = 3\ell$.  It may not be clear how this approach proves the $3\ell$ bits are indeed zero because within $h_0$ and $h_1$ there are bits that are nonzero.  The key observation is that these values are multiples of $2^t$ and thus would vanish $\mod 2^t$

By making the argument with $v_0$ and $v_1$ we are proving that $h_0$ is something where the $2\ell$ least significant bits are all zeros and that $h_1 + v_0$ is something where the $\ell$ are also zeros.  Any nonzero bits after $3\ell$ do not matter, since everything is $\mod 2^t$.

All that remains is to range check $v_0$ and $v_1$
> 7. Range check $v_0 \in [0, 2^2)$
> 8. Range check $v_1 \in [0, 2^{\ell + 3})$

### Native constraint

Until now we have constrained the equation $\mod 2^t$, but remember that our application of the CRT means that we must also constrain the equation $\mod n$. We are leveraging the fact that if the identity holds for all moduli in $\mathcal{M} = \{n, 2^t\}$, then it holds for $\mathtt{lcm} (\mathcal{M}) = 2^t \cdot n = M$.

Thus, we must check $a \cdot b - q \cdot f - r \equiv 0 \mod n$, which is over $\mathbb{F}_n$.

This gives us equality $\mod 2^t \cdot n$ as long as the multiplicands are coprime. That is, as long as $\mathsf{gcd}(2^t, n) = 1$. Since the native modulus $n$ is prime, this is true.

Thus, to perform this check is simple.  We compute

$$
\begin{aligned}
a_n &= a \mod n \\
b_n &= b \mod n \\
q_n &= q \mod n \\
r_n &= r \mod n \\
f_n &= f \mod n
\end{aligned}
$$

using our native field modulus with constraints like this

$$
\begin{aligned}
a_n &= 2^{2\ell} \cdot a_2 + 2^{\ell} \cdot a_1 + a_0 \\
b_n &= 2^{2\ell} \cdot b_2 + 2^{\ell} \cdot b_1 + b_0 \\
q_n &= 2^{2\ell} \cdot q_2 + 2^{\ell} \cdot q_1 + q_0 \\
r_n & = 2^{2\ell} \cdot r_2 + 2^{\ell} \cdot r_1 + r_0 \\
f_n &= 2^{2\ell} \cdot f_2 + 2^{\ell} \cdot f_1 + f_0 \\
\end{aligned}
$$

and then constrain

$$
a_n \cdot b_n - q_n \cdot f_n - r_n = 0 \mod n.
$$

Note that we do not use the negated foreign field modulus here.

This requires a single constraint of the form

> 9. $a_n \cdot b_n - q_n \cdot f_n = r_n$

with all of the terms expanded into the limbs according the the above equations.  The values $a_n, b_n, q_n, f_n$ and $r_n$ do not need to be in the witness.

## Appendix E: Range Checks

Range checks have a dominant cost, let's see how many we have.

Range check (3) requires two range checks for $p_{11} = p_{111} \cdot 2^\ell + p_{110}$
 * a) $p_{110} \in [0, 2^\ell)$
 * b) $p_{111} \in [0, 2^2)$

Range check (8) requires a decomposition check that is merged in (6), together with seven 12-bit lookups, three 2-bit checks, and one 1-bit check.
 * a) $v_{1,0} \in [0, 2^{12})$
 * b) $v_{1,12} \in [0, 2^{12})$
 * c) $v_{1,24} \in [0, 2^{12})$
 * d) $v_{1,36} \in [0, 2^{12})$
 * e) $v_{1,48} \in [0, 2^{12})$
 * f) $v_{1,60} \in [0, 2^{12})$
 * g) $v_{1,72} \in [0, 2^{12})$
 * h) $v_{1,84} \in [0, 2^{2})$
 * i) $v_{1,86} \in [0, 2^{2})$
 * j) $v_{1,88} \in [0, 2^{2})$
 * k) $v_{1,90} \in \{0, 1\}$

 Range checking the bound

The range checks on $p_0, p_1$ and $p_2$ follow from the range checks on $a,b$ and $q$.

So far, we have 3.a, 3.b, 4, 8.a, 8.b, 8.c, 8.d, 8.e, 8.f, 8.g, 8.h, 8.i, 8.j, 8.k

| Range check | Gate type(s)     | Witness     | Rows        |
| ----------- | ---------------- | ----------- | ----------- |
| 7           | crumb check      | $v_0$       | within gate |
| 3.a         | crumb check      | $p_{111}$   | within gate |
| 8.a         | 12-bit plookup   | $v_{1,0}$   | within gate |
| 8.b         | 12-bit plookup   | $v_{1,12}$  | within gate |
| 8.c         | 12-bit plookup   | $v_{1,24}$  | within gate |
| 8.d         | 12-bit plookup   | $v_{1,36}$  | within gate |
| 8.e         | 12-bit plookup   | $v_{1,48}$  | within gate |
| 8.f         | 12-bit plookup   | $v_{1,60}$  | within gate |
| 8.g         | 12-bit plookup   | $v_{1,72}$  | within gate |
| 8.h         | crumb-check      | $v_{1,84}$  | within gate |
| 8.i         | crumb-check      | $v_{1,86}$  | within gate |
| 8.j         | crumb-check      | $v_{1,88}$  | within gate |
| 8.k         | boolean-check    | $v_{1,90}$  | within gate |
| 4           | limb range check | $p_{10}$    | 1.33        |
| 4           | limb range check | $p_{110}$   | 1.33        |


### CRT-related

Until now we have constrained that equation $a \cdot b = q \cdot f + r$  holds modulo $2^t$ and modulo $n$, so by the CRT it holds modulo $M = 2^t \cdot n$.  Remember that we must prove our equation over the positive integers, rather than $\mod M$.  By doing so, we assure that our solution is not equal to some $q' \cdot M + r'$ where $q', r' \in F_{M}$, in which case $q$ or $r$ would be invalid.

First, let's consider the right hand side $q \cdot f + r$.  We have

$$
q \cdot f + r < 2^t \cdot n
$$

Recall that we have parameterized $2^t \cdot n \ge f^2$, so if we can bound $q$ and $r$ such that

$$
q \cdot f + r < f^2
$$

then we have achieved our goal.  We know that $q$ must be less than $f$, so that is our first check, leaving us with

$$
\begin{aligned}
(f - 1) \cdot f + r &< f^2 \\
r &< f^2 - (f - 1) \cdot f = f
\end{aligned}
$$

Therefore, to check $q \cdot f + r < 2^t \cdot n$, we need to check
* $q < f$
* $r < f$

This should come at no surprise, since that is how we parameterized $2^t \cdot n$ earlier on.  Note that by checking $q < f$ we assure correctness, while checking $r < f$ assures our solution is unique (sometimes referred to as canonical).

Next, we must perform the same checks for the left hand side (i.e., $a \cdot b < 2^t \cdot n$).  Since $a$ and $b$ must be less than the foreign field modulus $f$, this means checking
* $a < f$
* $b < f$

So we have

$$
\begin{aligned}
a \cdot b &\le (f - 1) \cdot (f - 1) = f^2 - 2f + 1 \\
\end{aligned}
$$

Since $2^t \cdot n \ge f^2$ we have

$$
\begin{aligned}
&f^2 - 2f + 1 < f^2 \le 2^t \cdot n \\
&\implies
a \cdot b < 2^t \cdot n
\end{aligned}
$$

## Appendix F: Bound checks


### Full bound checks

Here we show how to define a gadget to perform full bound checks, as in the *upper bound check* method described in the [Foreign Field Addition RFC](https://github.com/o1-labs/proof-systems/blob/master/book/src/rfcs/ffadd.md#upper-bound-check).

> Note that our `ForeignFieldMul` design obtains a correct result in the class of $r+k\cdot f$, meaning that it becomes the canonical $r$ modulo $f$. But when this gadget is being used in situations where only the canonical element is valid, then a _full bound check_ needs to be carried out in order to make sure that $r < f$. For usual values of $f$ such as secp256k1, only two possible values of the result could be output ($r$, and $r+f$ with $q-1$). But if smaller values of the foreign field were provided, then $k$ could take plenty of values.

The full bound check is meant to constrain that $0 \le x < f$ over the positive integers, so

$$
\begin{aligned}
2^t \le x &+ 2^t < f + 2^t \\
2^t - f \le x &+ 2^t - f < 2^t \\
\end{aligned}
$$

Remember $f' = 2^t - f$ is our negated foreign field modulus.  Thus, we have

$$
\begin{aligned}
f' \le x &+ f' < 2^t \\
\end{aligned}
$$

So to check $x < t$ we just need to compute $x' = x + f'$ and check $f' \le x' < 2^t$

Observe that

$$
0 \le x \implies  f' \le x'
$$

and that

$$
x' < 2^t \implies x < f
$$

So we only need to check

- $0 \le x$
- $x' < 2^t$

The first check is not as trivial as it seems, since "negative" limbs could wrap around and lead to fake results. Meaning, one cannot just assume that $x\in\mathbb{Z}^+$ just because we are working over finite fields. In order to do this, one needs to perform a multi range check on the original limbs of $x$. That is to ensure that each limb is at most $\ell$ limbs. And thus, no wraparound could happen with our $n$. 

Observe that the second check constrains that $x < 2^t$, since $f \le 2^{259} < 2^t$ and thus

$$
\begin{aligned}
x' &< 2^t \\
x + f' &< 2^t \\
x &< 2^t - (2^t - f) = f\\
x &< 2^t
\end{aligned}
$$

Therefore, to constrain $x < f$ we need constraints for

- $0 \le x$
- $x' = x + f'$
- $x' < 2^t$

This is done with 1 `ForeignFieldAdd` (followed by a `Zero` row) to obtain the $x'$ term, and 2 multi-range-checks. 

### Bounds in Addition

In our foreign field addition design the operands $a$ and $b$ do not need to be less than $f$. The field overflow bit $\mathcal{o}$ for foreign field addition is at most 1.  That is, $a + b = \mathcal{o} \cdot f + r$, where $r$ is allowed to be greater than $f$.  Therefore,

$$
(f + a) + (f + b) = 1 \cdot f + (f + a + b)
$$

These can be chained along $k$ times as desired.  The final result

$$
r = (\underbrace{f + \cdots + f}_{k} + a_1 + b_1 + \cdots a_k + b_k)
$$

Since the bit length of $r$ increases logarithmically with the number of additions, in Kimchi we must only check that the final $r$ in the chain is less than $f$ to constrain the entire chain.

> **Security note:** In order to defer the $r < f$ check to the end of any chain of additions, it is extremely important to consider the potential impact of wraparound in $\mathbb{F_n}$.  That is, we need to consider whether the addition of a large chain of elements greater than the foreign field modulus could wrap around.  If this could happen then the $r < f$ check could fail to detect an invalid witness.  Below we will show that this is not possible in Kimchi.
>
> Recall that our foreign field elements are comprised of 3 limbs of 88-bits each that are each represented as native field elements in our proof system.  In order to wrap around and circumvent the $r < f$ check, the highest limb would need to wrap around.  This means that an attacker would need to perform about $k \approx n/2^{\ell}$ additions of elements greater than then foreign field modulus.  Since Kimchi's native moduli (Pallas and Vesta) are 255-bits, the attacker would need to provide a witness for about $k \approx 2^{167}$ additions.  This length of witness is greater than Kimchi's maximum circuit (resp. witness) length.  Thus, it is not possible for the attacker to generate a false proof by causing wraparound with a large chain of additions.

In summary, for foreign field addition in Kimchi it is sufficient to only bound check the last result $r'$ in a chain of additions (and subtractions)

- Compute bound $r' = r + f'$ with addition gate (2 rows)
- Range check $r' < 2^t$ (4 rows)

### Bounds in Multiplication

In foreign field multiplication, the situation is unfortunately different, and we must check that each of $a, b, q$ and $r$ are less than a given length. We cannot adopt the strategy from foreign field addition where the operands are allowed to be arbitrarily greater because the bit length of $r$ would increases linearly with the number of multiplications. That is,

$$
(a_1 + f) \cdot (a_2 + f) = 1 \cdot f + \underbrace{f^2 + (a_1 + a_2 - 1) \cdot f + a_1 \cdot a_2}_{r}
$$

and after a chain of $k$ multiplication we have

$$
r = f^k + \ldots + a_1 \cdots a_k
$$

where $r > f^k$ quickly overflows our CRT modulus $2^t \cdot n$.  For example, assuming our maximum foreign modulus of $f = 2^{259}$ and either of Kimchi's native moduli (i.e. Pallas or Vesta), $f^k > 2^t \cdot n$ for $k > 2$.  That is, an overflow is possible for a chain of greater than 1 foreign field multiplication. Thus, we must check $a, b, q$ and $r$ are less than $2^t$ for each multiplication.

Fortunately, in many situations the input operands may already be checked either as inputs or as outputs of previous operations, so they may not be required for each multiplication operation. This means multi range checks for the original limbs of `a` and `b` can be reused across steps of a larger circuit, and also its bound checks.

Thus, the $q$ and $r$ checks (and corresponding $q'$ and $r'$) are our main focus because they must be done for every multiplication. Naïvely, this would take:

- Range check $q < 2^t$ (4 rows)
- Range check $r < 2^t$ (4 rows)
- Compute bound $q' = q + f'$ with addition gate (2 rows)
- Compute bound $r' = r + f'$ with addition gate (2 rows)
- Range check $q' < 2^t$ (4 rows)
- Range check $r' < 2^t$ (4 rows)

This costs 20 rows per multiplication. In a subsequent section, we will explain how to avoid redundant checks to reduce it further.

#### High limb checks

Given that all the limbs of $a, b, q, r$ are checked to be correct (at most $\ell$ bits each), the CRT bound can be satisfied if we just check further that their high limbs are correctly upper bounded like:

$$x'_2 = x_2 + 2^\ell - f_2 - 1 < 2^\ell$$

which is equivalent to saying that 

$$x_2 \leq f_2$$

This saves us $2.66$ range-check rows per term above. Moreover, it prevents us from computing the full bounds, saving us a considerable number of rows in terms of `ForeignFieldAdd` gadgets.

Either way, the high limbs of the input terms can be checked somewhere else, and they only contribute to at most $1.33\cdot 2$ rows per `ForeignFieldMul` (as opposed to 8). Plus, the high bounds computation itself can be carried out in one single `Generic` gate, taking $1$ row as opposed to $2\cdot 2$ rows before (using foreign field addition).

Similarly, the approach benefits the elements which are "newly" created within the `ForeignFieldMul` gate. That is, the quotient and remainder. 

#### Bounding the quotient

Given that we have sufficient space left in the gate, we can compute the actual high limb bound of one extra term within the `ForeignFieldMul` (instead of using a `Generic` for it). Since the quotient is an internal witness and is not meant to be used externally, we will include the high bound of $q$ inside (saving us $0.5$ rows). 

$$
q'_{2} = q_2 + 2^{\ell} - f_2 - 1
$$

Then, this value can be wired to a single-limb range check, using only $1.33$ rows. Because the constraints of the gate were using one multi range check gadget to constrain $p_{10}$ and $p_{110}$, it makes sense to use that remaining space to constrain $q'_2$ as well. 

#### Remainder compact limbs

Due to the limited number of permutable cells per gate, we do not have enough cells for copy constraining $q'$ and $r'$ (or $q$ and $r$) to their respective range check gadgets. To address this issue, we must decompose $r$ into 2 limbs instead of 3. This is doable because the bottom part of $r$ is always used together in the constraints (this is not doable in $q$ however):

$$
r = r_{01} + 2^{2\ell} \cdot r_2
$$

Then, $r_2$ is stored in a permutable cell and can be used by a _half_ `Generic` gate to compute 

$$
r'_2 = r_2 + 2^{2\ell} - f_2 - 1
$$

Note that $r'_2$ must be range checked by a `multi-range-check` gadget. Then, the exteral checks structure will keep track of it and make "dense" multi range check with triples of remainders coming from a number of `ForeignFieldMul` from a larger circuit. 

- Store a copy of the limbs $r_{01}$ and $r_2$ in its witness
- Compact-range check that each limb is $\ell$ bits each
- Decompose it into $r_0, r_1, r_2$ in the compact range check gadget
- Compute $r'_2$ using a `Generic` gate
- Push the bound to a list of external checks to be multi range checked together.


### Inlining compact full bound check inside a gate

This section explains an optimization that could be carried out if in the future we wanted to inline a compact-limb full bound check inside another gate. That is, checking that $x<f$ fully, inside one gate. By doing this we can save 2 rows per element (those from the `ForeignFieldAdd` gadget).

Doing this doesn't require adding a lot more witness data because the operands for the bound computations $x' = x + f'$ would already be present in the witness of the multiplication gate. We would only need to store the bounds $x'$ in permutable witness cells so that they may be copied to multi-range-check gates to check they are each less than $2^t$.

To constrain $x + f' = x'$, the equation we use is

$$
x + 2^t = \mathcal{o} \cdot f + x',
$$

where $x$ is the original value, $\mathcal{o}=1$ is the field overflow bit and $x'$ is the remainder and our desired addition result (e.g. the bound).  Rearranging things we get

$$
x + 2^t - f = x',
$$

which is just

$$
x + f' = x',
$$

Recall from the [Borrows](#appendix-c-borrows) section that $f'$ is often larger than $f$. At first this seems like it could be a problem because in multiplication each operation must be less than $f$. However, this is because the maximum size of the multiplication was quadratic in the size of $f$ (we use the CRT, which requires the bound that $a \cdot b < 2^t \cdot n$). However, for addition the result is much smaller and we do not require the CRT nor the assumption that the operands are smaller than $f$. Thus, we have plenty of space in $\ell$-bit limbs to perform our addition.

So, the equation we need to constrain is

$$
x + f' = x'.
$$

We can expand the left hand side into the 2 limb format in order to obtain 2 intermediate sums

$$
\begin{aligned}
s_{01} = x_{01} + f_{01}' \\
s_2 = x_2 + f'_2 \\
\end{aligned}
$$

where $x_{01}$ and $f'_{01}$ are defined like this

$$
\begin{aligned}
x_{01} = x_0 + 2^{\ell} \cdot x_1 \\
f'_{01} = f'_0 + 2^{\ell} \cdot f'_1 \\
\end{aligned}
$$

and $x$ and $f'$ are defined like this

$$
\begin{aligned}
x = x_{01} + 2^{2\ell} \cdot x_2 \\
f' = f'_{01} + 2^{2\ell} \cdot f'_2 \\
\end{aligned}
$$

Going back to our intermediate sums, the maximum bit length of sum $s_{01}$ is computed from the maximum bit lengths of $x_{01}$ and $f'_{01}$

$$
\underbrace{(2^{\ell} - 1) + 2^{\ell} \cdot (2^{\ell} - 1)}_{x_{01}} + \underbrace{(2^{\ell} - 1) + 2^{\ell} \cdot (2^{\ell} - 1)}_{f'_{01}} = 2^{2\ell+ 1} - 2,
$$

which means $s_{01}$ is at most $2\ell + 1$ bits long.

Similarly, since $x_2$ and $f'_2$ are less than $2^{\ell}$, the max value of $s_2$ is

$$
(2^{\ell} - 1) + (2^{\ell} - 1) = 2^{\ell + 1} - 2,
$$

which means $s_2$ is at most $\ell + 1$ bits long.

Thus, we must constrain

$$
s_{01} + 2^{2\ell} \cdot s_2 - x'_{01} - 2^{2\ell} \cdot x'_2 = 0 \mod 2^t.
$$

The accumulation of this into parts looks like this.

```text
0             L             2L            3L=t          4L
|-------------|-------------|-------------|-------------|-------------|
                            :
|------------s01------------:-| 2L + 1
                            : ↖w01
                            |------s2-----:-| L + 1
                            :               ↖w2
                            :
|------------x'01-----------|
                            :
                            |------x'2----|
                            :
\____________________________/
             ≈ z01           \_____________/
                                   ≈ z2
```

The two parts are computed with

$$
\begin{aligned}
z_{01} &= s_{01} - x'_{01} \\
z_2 &= s_2 - x'_2.
\end{aligned}
$$

Therefore, there are two carry bits $w_{01}$ and $w_2$ such that

$$
\begin{aligned}
z_{01} &= 2^{2\ell} \cdot w_{01} \\
z_2 + w_{01} &= 2^{\ell} \cdot w_2
\end{aligned}
$$

In this scheme $x'_{01}, x'_2, w_{01}$ and $w_2$ are witness data, whereas $s_{01}$ and $s_2$ are formed from a constrained computation of witness data $x_{01}, x_2$ and constraint system public parameter $f'$.  Note that due to carrying, witness $x'_{01}$ and $x'_2$ can be different than the values $s_{01}$ and $s_2$ computed from the limbs.

Thus, each bound addition $x + f'$ requires the following witness data

- $x_{01}, x_2$
- $x'_{01}, x'_2$
- $w_{01}, w_2$

where $f'$ is baked into the gate coefficients.  The following constraints are needed

- $2^{2\ell} \cdot w_{01} = s_{01} - x'_{01}$
- $2^{\ell} \cdot w_2 = s_2 + w_{01} - x'_2$
- $x'_{01} \in [0, 2^{2\ell})$
- $x'_2 \in [0, 2^{\ell})$
- $w_{01} \in [0, 2)$
- $w_2 \in [0, 2)$

Suppose that due to the limited number of copyable witness cells per gate, we were currently only performing this optimization for $q$.

The witness data is

- $q_0, q_1, q_2$
- $q'_{01}, q'_2$
- $q'_{carry01}, q'_{carry2}$

The checks are

1. $q_0 \in [0, 2^{\ell})$
2. $q_1 \in [0, 2^{\ell})$
3. $q'_0 = q_0 + f'_0$
4. $q'_1 = q_1 + f'_1$
5. $s_{01} = q'_0 + 2^{\ell} \cdot q'_1$
6. $q'_{01} \in [0, 2^{2\ell})$
7. $q'_{01} = q'_0 + 2^{\ell} \cdot q'_1$
8. $q'_{carry01} \in [0, 2)$
9. $2^{2\ell} \cdot q'_{carry01} = s_{01} - q'_{01}$
10. $q_2 \in [0, 2^{\ell})$
11. $s_2 = q_2 + f'_2$
12. $q'_{carry2} \in [0, 2)$
13. $2^{\ell} \cdot q'_{carry2} = s_2 + w_{01} - q'_2$


Checks (1) - (5) assure that $s_{01}$ is at most $2\ell + 1$ bits.  Whereas checks (10) - (11) assure that $s_2$ is at most $\ell + 1$ bits.  Altogether they are comprise a single `multi-range-check` of $q_0, q_1$ and $q_2$.  However, as noted above, we do not have enough copyable cells to output $q_1, q_2$ and $q_3$ to the `multi-range-check` gadget.  Therefore, we adopt a strategy where the 2 limbs $q'_{01}$ and $q'_2$ are output to the `multi-range-check` gadget where the decomposition of $q'_0$ and $q'_2$ into $q'_{01} = p_0 + 2^{\ell} \cdot p_1$ is constrained and then $q'_0, q'_1$ and $q'_2$ are range checked.

Although $q_1, q_2$ and $q_3$ are not range checked directly, this is safe because, as shown in the "Bound checks" section, range-checking that $q' \in [0, 2^t)$ also constrains that $q \in [0, 2^t)$.  Therefore, the updated checks are

1. $q_0 \in [0, 2^{\ell})$ `multi-range-check`
2. $q_1 \in [0, 2^{\ell})$ `multi-range-check`
3. $q'_0 = q_0 + f'_0$ `ForeignFieldMul`
4. $q'_1 = q_1 + f'_1$ `ForeignFieldMul`
5. $s_{01} = q'_0 + 2^{\ell} \cdot q'_1$ `ForeignFieldMul`
6. $q'_{01} = q'_0 + 2^{\ell} \cdot q'_1$  `multi-range-check`
7. $q'_{carry01} \in [0, 2)$ `ForeignFieldMul`
8. $2^{2\ell} \cdot q'_{carry01} = s_{01} - q'_{01}$ `ForeignFieldMul`
9.  $q_2 \in [0, 2^{\ell})$  `multi-range-check`
10. $s_2 = q_2 + f'_2$ `ForeignFieldMul`
11. $q'_{carry2} \in [0, 2)$ `ForeignFieldMul`
12. $2^{\ell} \cdot q'_{carry2} = s_2 + q'_{carry01} - q'_2$ `ForeignFieldMul`

Note that we don't need to range-check $q'_{01}$ is at most $2\ell + 1$ bits because it is already implicitly constrained by the `multi-range-check` gadget constraining that $q'_0, q'_1$ and $q'_2$ are each at most $\ell$ bits and that $q'_{01} = q'_0 + 2^{\ell} \cdot q'_1$.  Furthermore, since constraining the decomposition is already part of the `multi-range-check` gadget, we do not need to do it here also.

To simplify things further, we can combine some of these checks.  Recall our checked computations for the intermediate sums

$$
\begin{aligned}
s_{01} &= q_{01} + f'_{01} \\
s_2 &= q_2 + f'_2 \\
\end{aligned}
$$

where $q_{01} = q_0 + 2^{\ell} \cdot q_1$ and $f'_{01} = f'_0 + 2^{\ell} \cdot f'_1$.  These do not need to be separate constraints, but are instead part of existing ones.

Checks (10) and (11) can be combined into a single constraint $2^{\ell} \cdot q'_{carry2} = (q_2 + f'_2) + q'_{carry01} - q'_2$.  Similarly, checks (3) - (5) and (8) can be combined into $2^{2\ell} \cdot q'_{carry01} = q_{01} + f'_{01} - q'_{01}$ with $q_{01}$ and $f'_{01}$ further expanded.  The reduced constraints are

1. $q_0 \in [0, 2^{\ell})$ `multi-range-check`
2. $q_1 \in [0, 2^{\ell})$ `multi-range-check`
3. $q'_{01} = q'_0 + 2^{\ell} \cdot q'_1$  `multi-range-check`
4. $q'_{carry01} \in [0, 2)$ `ForeignFieldMul`
5. $2^{2\ell} \cdot q'_{carry01} = s_{01} - q'_{01}$ `ForeignFieldMul`
6. $q_2 \in [0, 2^{\ell})$  `multi-range-check`
7. $q'_{carry2} \in [0, 2)$ `ForeignFieldMul`
8. $2^{\ell} \cdot q'_{carry2} = s_2 + w_{01} - q'_2$ `ForeignFieldMul`


1. $q_0 \in [0, 2^{\ell})$ `multi-range-check`
2. $q_1 \in [0, 2^{\ell})$ `multi-range-check`
3. $q'_{01} = q'_0 + 2^{\ell} \cdot q'_1$  `multi-range-check`
4. $q'_{carry01} \in [0, 2)$ `ForeignFieldMul`
5. $2^{2\ell} \cdot q'_{carry01} = s_{01} - q'_{01}$ `ForeignFieldMul`
6. $q_2 \in [0, 2^{\ell})$  `multi-range-check`
7. $q'_2 = s_2 + w_{01}$ `ForeignFieldMul`

In other words, we have eliminated constraint (7) and removed $q'_{carry2}$ from the witness.

Since we already needed to range-check $q$ or $q'$, the total number of new constraints added is 4: 3 added to to `ForeignFieldMul` and 1 added to `multi-range-check` gadget for constraining the decomposition of $q'_{01}$.

This saves 2 rows per multiplication.

#### Second carry bit would always be zero

Finally, there is one more optimization that we will exploit.  This optimization relies on the observation that for bound addition the second carry bit $q'_{carry2}$ is always zero. This this may be obscure, so we will prove it by contradiction.  To simplify our work we rename some variables by letting $x_0 = q_{01}$ and $x_1 = q_2$.  Thus, $q'_{carry2}$ being non-zero corresponds to a carry in $x_1 + f'_1$.

> **Proof:** To get a carry in the highest limbs $x_1 + f'_1$ during bound addition, we need
>
> $$
> 2^{\ell} < x_1 + \phi_0 + f'_1 \le 2^{\ell} - 1 + \phi_0 + f'_1
> $$
>
> where $2^{\ell} - 1$ is the maximum possible size of $x_1$ (before it overflows) and $\phi_0$ is the overflow bit from the addition of the least significant limbs $x_0$ and $f'_0$.  This means
>
> $$
> 2^{\ell} - \phi_0 - f'_1 < x_1 < 2^{\ell}
> $$
>
> We cannot allow $x$ to overflow the foreign field, so we also have
>
> $$
> x_1 < (f - x_0)/2^{2\ell}
> $$
>
> Thus,
>
> $$
> 2^{\ell} - \phi_0  - f'_1 < (f - x_0)/2^{2\ell} = f/2^{2\ell} - x_0/2^{2\ell}
> $$
>
> Since $x_0/2^{2\ell} = \phi_0$ we have
>
> $$
> 2^{\ell} - \phi_0 - f'_1 < f/2^{2\ell} - \phi_0
> $$
>
> so
>
> $$
> 2^{\ell} - f'_1 < f/2^{2\ell}
> $$
>
> Notice that $f/2^{2\ell} = f_1$.  Now we have
>
> $$
> 2^{\ell} - f'_1 < f_1 \\
> \Longleftrightarrow \\
> f'_1 > 2^{\ell} - f_1
> $$
>
> However, this is a contradiction with the definition of our negated foreign field modulus limb $f'_1 = 2^{\ell} - f_1$. $\blacksquare$

We have proven that $q'_{carry2}$ is always zero, so that allows use to simplify our constraints.  We now have


## Appendix G: Costs

### Range constrain $a$

If the $a$ operand has not been constrained to $[0, f)$ by any previous foreign field operations, then we constrain it like this
- Range check $a \in [0, 2^t)$ (4 rows) `multi-range-check`
- Compute high bound $a'_2 = a_2 + 2^\ell - f_2 - 1$ (0.5 rows) `Generic`
- Limb range check $a'_2 \in [0, 2^\ell)$ (1.33 rows)  `multi-range-check`

### Range constrain $b$

If the $b$ operand has not been constrained to $[0, f)$ by any previous foreign field operations, then we constrain it like this
- Range check $b \in [0, 2^t)$ (4 rows) `multi-range-check`
- Compute high bound $b'_2 = b_2 + 2^\ell - f_2 - 1$ (0.5 rows) `Generic`
- Limb range check $b'_2 \in [0, 2^\ell)$ (1.33 rows)  `multi-range-check`

### Range constrain $q$

The quotient $q$ is constrained to $[0, ~f)$ for each multiplication as part of the multiplication gate
- Range check $q \in [0, 2^t)$ (4 rows) `multi-range-check`
- Compute high bound $q'_2 = q_2 + 2^\ell - f_2 - 1$ (implicit by storing `q'_2 in witness)
- Limb range check $q'_2 \in [0, 2^\ell)$ (1.33 rows) (part of `ForeignFieldMul` MRC)

### Range constrain $r$

The remainder $r$ is constrained to $[0, ~f)$ for each multiplication using an external gneric gate.
- Decomposition of $r_0, r_1, r_2$ and $r<2^t$ performed by `multi-range-check`
- Compute high bound $r'_2 = r_2 + 2^\ell - f_2 - 1$ (0.5 rows) `Generic`
- Limb range check $r'_2 \in [0, 2^\ell)$ (1.33 rows)  `multi-range-check`

### Compute intermediate products

Compute and constrain the intermediate products $p_0, p_1$ and $p_2$ as:

- $p_0 = a_0 \cdot b_0 + q_0 \cdot f'_0$ `ForeignFieldMul`
- $p_1 = a_0 \cdot b_1 + a_1 \cdot b_0 + q_0 \cdot f'_1 + q_1 \cdot f'_0$ `ForeignFieldMul`
- $p_2 = a_0 \cdot b_2 + a_2 \cdot b_0 + a_1 \cdot b_1 + q_0 \cdot f'_2 + q_2 \cdot f'_0 + q_1 \cdot f'_1$ `ForeignFieldMul`

where each of them is about $2\ell$-length elements.

### Native modulus checked computations

Compute and constrain the native modulus values, which are used to check the constraints modulo $n$ in order to apply the CRT

- $a_n = 2^{2\ell} \cdot a_2 + 2^{\ell} \cdot a_1 + a_0 \mod n$
- $b_n = 2^{2\ell} \cdot b_2 + 2^{\ell} \cdot b_1 + b_0 \mod n$
- $q_n = 2^{2\ell} \cdot q_2 + 2^{\ell} \cdot q_1 + q_0 \mod n$
- $r_n = 2^{2\ell} \cdot r_2 + r_{01} \mod n$
- $f_n = 2^{2\ell} \cdot f_2 + 2^{\ell} \cdot f_1 + f_0 \mod n$

Actually, there is no need to store in the coefficients the limbs of $f$, but only $(f'_0, f'_1, f'_2)$ and $f_2$. Then the native constrain uses the negated limbs of `f` instead.

### Decompose middle intermediate product

Check that $p_1 = 2^{\ell} \cdot p_{11} + p_{10}$:

- $p_1 = 2^\ell \cdot p_{11} + p_{10}$ `ForeignFieldMul`
- Range check $p_{10} \in [0, 2^\ell)$ `multi-range-check`
- Range check $p_{11} \in [0, 2^{\ell+2})$
    - $p_{11} = p_{111} \cdot 2^\ell + p_{110}$  `ForeignFieldMul`
    - Range check $p_{110} \in [0, 2^\ell)$ `multi-range-check`
    - Range check $p_{111} \in [0, 2^2)$ with a degree-4 constraint `ForeignFieldMul`

### Zero sum for multiplication

Now we have to constrain the zero sum

$$
(p_0 - r_0) + 2^{88}(p_1 - r_1) + 2^{176}(p_2 - r_2) = 0 \mod 2^t
$$

We constrain the first and the second halves as

- $v_0 \cdot 2^{2\ell} = p_0 + 2^\ell \cdot p_{10} - r_{01}$ `ForeignFieldMul`
- $v_1 \cdot 2^{\ell} = (p_{111} \cdot 2^\ell + p_{110}) + p_2 - r_2 + v_0$ `ForeignFieldMul`

And some more range checks. In order to save one space in permutable cells, and taking advantage of the many empty cells in the layout, we can decompose $v_1$ into smaller parts, and avoid the limb check through the range check gadget.

- Check that $v_0 \in [0, 2^2)$ with a degree-4 constraint `ForeignFieldMul`
- Check that $v_1 \in [0, 2^{\ell + 3})$
    - Check $v_{1,0} \in [0, 2^{12})$ lookup in `ForeignFieldMul`
    - Check $v_{1,12} \in [0, 2^{12})$ lookup in `ForeignFieldMul`
    - Check $v_{1,24} \in [0, 2^{12})$ lookup in `ForeignFieldMul`
    - Check $v_{1,36} \in [0, 2^{12})$ lookup in `ForeignFieldMul`
    - Check $v_{1,48} \in [0, 2^{12})$ lookup in `ForeignFieldMul`
    - Check $v_{1,60} \in [0, 2^{12})$ lookup in `ForeignFieldMul`
    - Check $v_{1,72} \in [0, 2^{12})$ lookup in `ForeignFieldMul`
    - Check $v_{1,84} \in [0, 2^2)$ crumb in `ForeignFieldMul`
    - Check $v_{1,86} \in [0, 2^2)$ crumb in `ForeignFieldMul`
    - Check $v_{1,88} \in [0, 2^2)$ crumb in `ForeignFieldMul`
    - Check $v_{1,90} \in \{0,1\}$ boolean in `ForeignFieldMul`
- Compose all parts of $v_1$

> Before, we used to have 1 limb of 88 bits and 1 scaled lookup that $v_{11}$ was at most 3 bits long. This was done by first checking $v_{11} \in [0, 2^{12})$ with a 12-bit plookup. This means there can be no higher bits set beyond the 12-bits of $v_{11}$.  Next, we scale $v_{11}$ by $2^9$ in order to move the highest $12 - 3 = 9$ bits beyond the $12$th bit. Finally, we perform a 12-bit plookup on the resulting value. That is, we had
>
>- Check $v_{11} \in [0, 2^{12})$ with a 12-bit plookup (to prevent any overflow)
>- Check $\mathsf{scaled}_{v_{11}} = 2^9 \cdot v_{11}$
>- Check $\mathsf{scaled}_{v_{11}}$ is a 12-bit value with a 12-bit plookup
>
> Kimchi's plookup implementation is extremely flexible and supports optional scaling of the lookup target value as part of the lookup operation. Thus, we do not require two witness elements, two lookup columns, nor the $\mathsf{scaled}_{v_{11}} = 2^9 \cdot v_{11}$ custom constraint.  Instead we can just store $v_{11}$ in the witness and define this column as a "joint lookup" comprised of one 12-bit plookup on the original cell value and another 12-bit plookup on the cell value scaled by $2^9$, thus, yielding a 3-bit check.  This eliminates one plookup column and reduces the total number of constraints.

### Native modulus constraint

Using the checked native modulus computations we constrain that

$$
a_n \cdot b_n + q_n \cdot f'_n - r_n = 0 \mod n.
$$

### Decompose the higher quotient bound

Check that $q'_{01} = q'_0 + 2^{\ell} \cdot q'_1$.

Done by (3) above with the  `multi-range-check` on $q'$
- $q'_{01} = q'_0 + 2^{\ell} \cdot q'_1$
- Range check $q'_0 \in [0, 2^\ell)$
- Range check $q'_1 \in [0, 2^\ell)$


## Appendix H: Layout

Based on the constraints above, we need the following 11 values copied from the range check gates.

```
a0, a1, a2, b0, b1, b2, r01, r2, q0, q1, q2
```
Since we need 11 copied values for the constraints, they must span 2 rows.

The $q$ high bound limb $q'_2$ must be in a copyable cell so it can be range-checked.  Similarly, the limbs of the operands $a$, $b$, quotient $q$ and the result $r$ must all be in copyable cells. This leaves only 2 remaining copyable cells and, therefore, we cannot compute and output $r'_2$. It must be deferred to an external `Generic` gate with the $r_2$ cell copied as an argument.

NB: the $f_2$ and $f'$ values are publicly visible in the gate coefficients.

|            | Curr                | Next                |
| ---------- | ------------------- | ------------------- |
| **Column** | `ForeignFieldMul`   | `Zero`              |
| 0          | $a_0$ (copy)        | $r_{01}$ (copy)     |
| 1          | $a_1$ (copy)        | $r_2$ (copy)        |
| 2          | $a_2$ (copy)        | $q_0$ (copy)        |
| 3          | $b_0$ (copy)        | $q_1$ (copy)        |
| 4          | $b_1$ (copy)        | $q_2$ (copy)        |
| 5          | $b_2$ (copy)        | $q'_2$  (copy)      |
| 6          | $p_{10}$ (copy)     | $p_{110}$ (copy)    |
| 7          | $v_{1,0}$ (lookup)  | $p_{111}$ (dummy)   |
| 8          | $v_{1,12}$ (lookup) | $v_{1,48}$ (lookup) |
| 9          | $v_{1,24}$ (lookup) | $v_{1,60}$ (lookup) |
| 10         | $v_{1,36}$ (lookup) | $v_{1,72}$ (lookup) |
| 11         | $v_{1,84}$          | $v_0$               |
| 12         | $v_{1,86}$          |                     |
| 13         | $v_{1,88}$          |                     |
| 14         | $v_{1,90}$          |                     |

## Appendix I: Checks

In total we require the following checks

1. $p_{111} \in [0, 2^2)$
2. $p_{110} \in [0, 2^{\ell})$ `multi-range-check`
3. $p_{10} \in [0, 2^{\ell})$ `multi-range-check`
4. $v_{1,0} \in [0, 2^{12})$ `Lookup`
5. $v_{1,12} \in [0, 2^{12})$ `Lookup`
6. $v_{1,24} \in [0, 2^{12})$ `Lookup`
7. $v_{1,36} \in [0, 2^{12})$ `Lookup`
8. $v_{1,48} \in [0, 2^{12})$ `Lookup`
9. $v_{1,60} \in [0, 2^{12})$ `Lookup`
10. $v_{1,72} \in [0, 2^{12})$ `Lookup`
11. $v_{1,84} \in [0, 2^2)$ 
12. $v_{1,86} \in [0, 2^2)$ 
13. $v_{1,88} \in [0, 2^2)$ 
14. $v_{1,90} \in \{0, 1\}$ 
15. $v_0 \in [0, 2^2)$
16. $p_{11} = 2^{\ell} \cdot p_{111} + p_{110}$
17. $p_1 = 2^{\ell} \cdot p_{11} + p_{10}$
18. $2^{2\ell} \cdot v_0 = p_0 + 2^{\ell} \cdot p_{10} - r_{01}$
19. $v_1 = v_{1,0} + v_{1,12}*2^{12} + v_{1,24}*2^{24} + v_{1,36}*2^{36} + v_{1,48}*2^{48} + v_{1,60}*2^{60} + v_{1,72}*2^{72} + v_{1,84}*2^{84} + v_{1,86}*2^{86} + v_{1,88}*2^{88} + v_{1,90}*2^{90}$
20. $2^{\ell} \cdot v_1 = v_0 + p_{11} + p_2 - r_2$
21. $a_n \cdot b_n + q_n \cdot f'_n = r_n$
22. $q'_2 = q_2 + 2^\ell - f_2 - 1$
23. $q'_2 \in [0, 2^\ell]$ `multi-range-check`

## Appendix J: Constraints

These checks can be condensed into the minimal number of constraints as follows.

Note that (4-10) are carried out by the lookup argument.

First, we have the range-check corresponding to (1) as a degree-4 constraint

**C1:** $p_{111} \cdot (p_{111} - 1) \cdot (p_{111} - 2) \cdot (p_{111} - 3)$

Now we have the range-check corresponding to (15) as another degree-4 constraint

**C2:** $v_0 \cdot (v_0 - 1) \cdot (v_0 - 2) \cdot (v_0 - 3)$

Next (16) and (17) can be combined into

**C3:** $2^{\ell} \cdot (2^{\ell} \cdot p_{111} + p_{110}) + p_{10} = p_1$

Next we have check (18)

**C4:** $2^{2\ell} \cdot v_0 = p_0 + 2^{\ell} \cdot p_{10} - r_{01}$

Next, for our use of the CRT, we must constrain that $a \cdot b = q \cdot f + r \mod n$.  Thus, check (21) is

**C5:** $a_n \cdot b_n + q_n \cdot f'_n = r_n$

**C6:** (11) $v_{1,84}$ (2-bit check)

$v_{1,84} \cdot (v_{1,84} - 1) \cdot (v_{1,84} - 2) \cdot (v_{1,84} - 3)$

**C7:** (12) $v_{1,86}$ (2-bit check)

$v_{1,86} \cdot (v_{1,86} - 1) \cdot (v_{1,86} - 2) \cdot (v_{1,86} - 3)$

**C8:** (13) $v_{1,88}$ (2-bit check)

$v_{1,88} \cdot (v_{1,88} - 1) \cdot (v_{1,88} - 2) \cdot (v_{1,88} - 3)$

**C9:** (14) $v_{1,90}$ (1-bit check)

$v_{1,90} \cdot (v_{1,90} - 1) $

Now checks (19) and (20) can be combined into

**C10:**  $2^{\ell} \cdot (v_{1,0} + v_{1,12}*2^{12} + v_{1,24}*2^{24} + v_{1,36}*2^{36} + v_{1,48}*2^{48} + v_{1,60}*2^{60} + v_{1,72}*2^{72} + v_{1,84}*2^{84} + v_{1,86}*2^{86} + v_{1,88}*2^{88} + v_{1,90}*2^{90}) = p_2 + p_{11} + v_0 - r_2$

Check (22) is correctly computed

**C11:** $q'_2 - q_2 + 2^\ell + f_2 + 1$.

**MRC:** `multi-range-check` $q'_2, p_{10}, p_{110}$

**MRC:** `1.33 multi-range-check` $r_2`

**MRC:** `compact-multi-range-check` $r_01, r_2$

**MRC:** `multi-range-check` $q_0, q_1, q_2$

## Appendix K: External checks

The following checks must be done with other gates to assure the soundness of the foreign field multiplication

- Range check input
    - `multi-range-check` $a$
    - `multi-range-check` $b$
- Compute and constrain high bounds of inputs $a'_2$ and $b'_2$
    - `Generic` $a_2 + (2^\ell - f_2 -1) = a'_2$ and $b_2 + (2^\ell - f_2 -1) = b'_2$
- If $r$ needs to be canonical ($<f$ compute a full bound check: 
    - `ForeignFieldAdd` $r + 2^t = 1 \cdot f + r'$
    - `multi-range-check` $r'$

Copy constraints must connect the above witness cells to their respective input cells within the corresponding external check gates witnesses.

## Appendix L: Soundness

This long section describes the soundness proof behind the proposed `ForeignFieldMul` gate.

Given witnesses $a$, $b$, $q$, $r$ each represented as 3 limbs $(x_0, x_1, x_2)$ by

$$x = x_0 + 2^\ell x_1 + 2^{2\ell} x_2,$$ 

we want to prove that, assuming a certain list of constraints,

$$\tag{L.1}
\begin{equation}
a b = q f + r
\end{equation}
$$

holds as an equation over the integers.

### `ForeignFieldMul` without range checks

Our derivation in this section will use the constraints of the FFMul gate, except for any assumptions about the sizes of $a$, $b$, $q$ and $r$. The goal is to work out exactly what range checks we need on these values.

The first constraint we use is checking $(L.1)$ modulo the native modulus $n$ (where we expand each term into its limbs). This single constraint proves that there is an $\varepsilon \in\mathbb{Z}$ such that

$$\tag{L.2a}
a b - q f - r = \varepsilon n
$$

Our strategy now is to also constrain the LHS modulo $2^{3\ell}$ and then use CRT-type arguments. So, let's expand  $a b - q f - r$ into limbs, but collect all terms higher than $2^{3\ell}$ into a single term $w$.

$$
\begin{aligned}
& a b - q f - r = \\
&  (a_0 b_0 + q_0 f'_0 - r_0) \\
&+ 2^{\ell} (a_0 b_1 + a_1 b_0 + q_0 f'_1 + q_1 f'_0 - r_1) \\
&+ 2^{2\ell} (a_0 b_2 + a_2 b_0 + q_0 f'_2 + q_2 f'_0 + a_1 b_1 + q_1 f'_1 - r_2) \\
&+ 2^{3\ell} w
\end{aligned}
$$

This equation holds over the integers. Like in the RFC, to abbreviate formulas we define $p_i$ such that the equation becomes

$$\tag{L.2b}
a b - q f - r = (p_0 - r_0) + 2^{\ell} (p_1 - r_1) + 2^{2\ell} (p_2 - r_2) + 2^{3\ell} w
$$

Next, we split $p_1 = p_{01} + 2^\ell p_{11}$ where we add range checks to prove that $p_{01}$ has at most $\ell$ bits and $p_{11}$ has at most $\ell + 2$ bits. The split itself is a constraint modulo $n$, showing that

$$\tag{L.3}
p_1 = p_{01} + 2^\ell p_{11} + \beta n
$$

for some $\beta \in\mathbb{Z}$.

Expanding $p_1$ using the last equation in $(2')$, we get

$$\tag{L.2c}
a b - q f - r = (p_0 + 2^\ell p_{01} - r_0 - 2^\ell r_1) + 2^{2\ell} (p_2 + p_{11} - r_2) + 2^{3\ell} w + 2^\ell \beta n 
$$

Now we will start constraining this equation limb-wise. We add a constraint for the bottom 2 limbs, namely

$$\tag{L.4}
p_0 + 2^\ell p_{01} - r_0 - 2^\ell r_1 = 2^{2\ell} v_0 + \alpha n
$$

for some $\alpha \in\mathbb{Z}$, where $v_0$ is some carry value and we prove that $v_0$ has at most 2 bits. Plugging this into $(L.2c)$, we eliminate the first bracketed term and get

$$\tag{L.2d}
a b - q f - r = 2^{2\ell} (p_2 + p_{11} - r_2 + v_0) + 2^{3\ell} w + (\alpha + 2^\ell \beta) n
$$

Next, we add a constraint for the high limb. For this we introduce a carry value $v_{1}$ which is range-checked to have at most $\ell + 3$ bits. We add the following constraint:

$$\tag{L.5}
p_2 + p_{11} - r_2 + v_0 = 2^{\ell} v_{1} + \gamma n 
$$

for some $\gamma \in\mathbb{Z}$. Plugging this into $(L.2d)$, the $2^\ell$ term is eliminated and we have proved that

$$\tag{L.2e}
a b - q f - r = 2^{3\ell} (w + v_{1})  + (\alpha + 2^\ell \beta + 2^{2\ell} \gamma) n $$

Now, for a brief moment let's look at this equation modulo $n$. Since the LHS is equal to $\varepsilon n$ by $(L.2a: native)$, we have

$$
2^{3\ell} (w + v_{1})  = 0 \mod{n}
$$

Because $2^{3\ell}$ and $n$ are co-prime, we can multiply with $2^{-3\ell}$ to see that

$$
w + v_{1}  = 0 \mod{n}
$$

So let's write

$$
w + v_{1}  = \delta n
$$

(This was the CRT-type argument.)

In summary, by $(L.2e)$ and **without any assumptions on the sizes of $a$, $b$, $q$, $r$, our constraints prove that**

$$\tag{L.6}
a b - q f - r = (\alpha + 2^\ell \beta + 2^{2\ell} \gamma + 2^{3\ell}\delta) n
$$

It only remains to introduce range checks sufficient to imply that $\alpha = \beta = \gamma = \delta = 0$. Or, looked at it differently, it suffices to make one of those values non-zero to find a counter-example to the soundness of the `ForeignFieldMul` gate.

### Conditions to make $\alpha = \beta = \gamma = \delta = 0$

Let's recall where possible non-zero values on the RHS may come from:

* **$\alpha$ can be non-zero if $\text{(L.4: bottom limbs)}$ overflows $n$.**
* **$\beta$ can be non-zero if $\text{(L.3: split)}$ overflows $n$.**
* **$\gamma$ can be non-zero if $\text{(L.5: high limb)}$ overflows $n$.**

To prove $\alpha = 0$ it's enough to show  
$$\tag{L.7a}
-n < p_0 + 2^\ell p_{01} - r_0 - 2^\ell r_1 - 2^{2\ell} v_0 < n$$

To prove $\beta = 0$ it's enough to show  
$$\tag{L.7b}
-n < p_1 - p_{01} + 2^\ell p_{11} < n
$$

To prove $\gamma = 0$ it's enough to show  
$$\tag{L.7c}
-n < p_2 + p_{11} - r_2 + v_0 - 2^{\ell} v_{1} < n$$

If $\alpha = \beta = \gamma = 0$, then by $(6)$ we know that

$$\tag{L.8}
a b - q f - r = \delta 2^{3\ell} n
$$

* **$\delta$ can be non-zero if $\text{(L.8: final)}$ overflows $2^{3\ell} n$.**

To prove $\delta = 0$ it's enough to show $\alpha = \beta = \gamma = 0$ and
 
$$\tag{L.9}
-2^{3\ell} n < a b - q f - r < 2^{3\ell} n
$$

#### Range-checking $a$, $b$, $q$, $r$

Recall the following range checks which we already had to assume so far:

* $v_0$ $\in [0,2^2)$
* $p_{01}$ $\in [0,2^\ell)$
* $p_{11}$ $\in [0,2^{\ell + 2})$
* $v_{1}$ $\in [0,2^{\ell + 3})$

Now, let's assume in addition that all of the following limb values are ranged-checked to be at most $\ell$ bits:

* $a_0$, $a_1$, $a_2$ $\in [0,2^\ell)$
* $b_0$, $b_1$, $b_2$ $\in [0,2^\ell)$
* $q_0$, $q_1$, $r_2$ $\in [0,2^\ell)$
* $r_0 + 2^\ell r_1 \in [0,2^{2\ell})$
* $r_2$ $\in [0,2^\ell)$

Note that $r$ is treated a bit differently than the other values: We don't require a range-check of the individual limbs $r_0$ and $r_1$, but only of the combined "compact limb" $r_0 + 2^\ell r_1$. This works because our constraints use only the combined form.

This is enough to prove $(L.7a)$, $(L.7b)$ and $(L.7c)$.

> (L.7a)

First, we put estimates on $p_0 = a_0 b_0 + q_0 f'_0$:

$$0 \le a_0 b_0 + q_0 f'_0 < 2^{2\ell} + 2^{2\ell} = 2^{2\ell + 1} $$

Using that we get $(L.7a)$, upper bound:

$$
\begin{align*}
& p_0 + 2^\ell p_{01} - (r_0 + 2^\ell r_1) - 2^{2\ell} v_0 < \\
& 2^{2\ell + 1} + 2^{2\ell} - 0 - 0 < \\
& 2^{2\ell + 2} < n 
\end{align*}
$$

$(L.7a)$, lower bound:

$$
\begin{align}
& -p_0 - 2^\ell p_{01} + (r_0 + 2^\ell r_1) + 2^{2\ell} v_0 < \\
& -0 - 0 + 2^{2\ell} + 3 \cdot 2^{2\ell} = \\
& 2^{2\ell + 2} < n
\end{align}
$$

> (L.7b)

Estimate $p_1 = a_0 b_1 + a_1 b_0 + q_0 f'_1 + q_1 f'_0$:

$$0 \le a_0 b_1 + a_1 b_0 + q_0 f'_1 + q_1 f'_0 < 4 \cdot 2^{2\ell} = 2^{2\ell + 2} $$

$(L.7b)$, upper bound:

$$
\begin{align*}
& p_1 - p_{01} + 2^\ell p_{11} < \\
& 2^{2\ell + 2} - 0 + 2^{2\ell + 2} = \\
& 2^{2\ell + 3} < n 
\end{align*}
$$

$(L.7b)$, lower bound:

$$
\begin{align*}
& -p_1 + p_{01} - 2^\ell p_{11} < \\
& -0 + 2^\ell - 0 = \\
& 2^\ell < n 
\end{align*}
$$

> (L.7c)

Estimate $p_2 = a_0 b_2 + a_1 b_1 + a_2 b_0 + q_0 f'_2 + q_1 f'_1 + q_2 f'_0$:

$$0 \le p_2 < 6 \cdot 2^{2\ell} $$

$(L.7c)$, upper bound:

$$
\begin{align}
& p_2 + p_{11} - r_2 + v_0 - 2^{\ell} v_{1} < \\
& 6 \cdot 2^{2\ell} + 2^{\ell + 2} - 0 + 2^2 - 0 < \\
& 2^{2\ell + 3} < n 
\end{align}
$$

$(L.7c)$, lower bound:

$$
\begin{align}
& -p_2 - p_{11} + r_2 - v_0 + 2^{\ell} v_{1} < \\
& -0 - 0 + 2^\ell - 0 +  2^{2\ell + 3} = \\
& 2^{2\ell + 4} < n 
\end{align}
$$

> Final bound

We will now discuss how to prove the final bound

$$\tag{L.9}
-2^{3\ell} n < a b - q f - r < 2^{3\ell} n
$$

To prove $(L.9)$, it's not enough to have $\ell$-bit range checks on all the limbs, because for example $ab = (2^{3\ell} - 1)^2 > 2^{3\ell} n$. The RFC chose $\ell$ and $3$ specifically such that
$$f^2 < 2^{3\ell} n$$
for all $f$ that we care about, so that the estimate $(L.9)$ works if $a$, $b$, $q$, $r$ < $f$ (i.e., if they are foreign field elements in canonical representation).

However, for efficiency we want to relax the $< f$ requirement a bit, because it would require a foreign field addition plus 3 range checks per variable. We can get an almost equivalent estimate by only range-checking the highest limb of each variable and a native addition.

Concretely, for a variable $x = (x_0, x_1, x_2)$ we define the _high-limb bounds check_ as the following two constraints:

* $x_2 + (2^\ell - f_2) = z \mod n$
* $z \in [0,2^\ell)$

We might write this succinctly as "$x_2 + (2^\ell - f_2) \in [0,2^\ell)$" but in that case we must not forget that "$+$" is finite field addition.

For a field element $x_2 \in [0, n)$ the high-limb bounds check implies that

$$x_2 \in [0, f_2) \cup [n - 2^\ell + f_2, n).$$

So this allows $x_2$ to be a large number close to $n$. If, in addition, $x_2$ is range-checked to be at most $\ell$ bits, the high interval is excluded:

$$x_2 \in [0, 2^\ell) \quad\text{and}\quad x_2 + (2^\ell - f_2) \in [0,2^\ell) \quad\Longrightarrow\quad x_2 \in [0, f_2)$$

For $x = (x_0, x_1, x_2)$, the 3-limb range check plus high-limb bounds check imply $x < f + 2^{2\ell}$, as follows:

$$
x = 2^{2\ell} x_2 + 2^\ell x_1 + x_0  < 2^{2\ell} f_2 + 2^{2\ell} \le f + 2^{2\ell}
$$

This is only slightly weaker than $x < f$ and is enough for us to prove $(L.9)$.

We add the high-limb bounds check to the 3-limb range-checks for $a$, $b$ and $q$:

* $a_2 + (2^\ell - f_2) \in [0,2^\ell)$
* $b_2 + (2^\ell - f_2) \in [0,2^\ell)$
* $q_2 + (2^\ell - f_2) \in [0,2^\ell)$

This implies that $a,b,q \in[0, f + 2^\ell)$. For $r$, we already know that $r \in [0, 2^{3\ell})$ and don't need additional assumptions.

We now get the following upper bound:

$$
ab - qr - f < ab < (f + 2^\ell)^2
$$

And lower bound:

$$
-ab + qr + f < qr + f < (f + 2^\ell)^2 + 2^{3\ell}
$$

Thus, we can prove $(L.9)$ if the foreign field modulus satisfies

$$
(f + 2^\ell)^2 + 2^{3\ell} < 2^{3\ell} n
$$

$$
f < \sqrt{ 2^{3\ell} (n - 1) } - 2^\ell
$$

For both Pasta moduli $n$, this still means that $f$ can be any prime up to 259 bits.

#### Necessity of range checks

In the last section we identified range conditions sufficient to make the FFmul gate sound. Ideally, we also want to be sure that they are _necessary_, and that we can't optimize our design further by removing some of them.

We'll analyze two different ideas to optimize the gate by removing conditions on $q$:

* **First idea**: Skip the check $q_2 \in [0, 2^\ell)$, so that $q_2$ is only constrained by $q_2 + (2^\ell - f_2) \in [0, 2^\ell)$
* **Second idea**: Do not range-check $q_0$ and $q_1$ individually, but only their combination into a compact limb: $q_0 + 2^\ell q_1 \in [0, 2^\ell)$ (where "$+$" is field addition)

The second idea allows us to borrow or carry values between $q_0$ and $q_1$. For example, we could represent $q = n$ by the limbs $(q_0, q_1, q_2) = (2^\ell, n-1, 0)$, because

$$(q_0 + 2^\ell q_1) \bmod{n} = 0 \in [0, 2^\ell)$$

In equations $\bmod{n}$, these limbs will be indistinguishable from $(2^\ell, -1, 0)$ which is a mathematically valid representation of $q = 0$. Indeed, it is easy to show that we satisfy all our constraints with $a = b = r = 0$ and $q = (2^\ell, n-1, 0)$, for typical values of $f$. In other words, the prover fools the verifer into believing that $q = n$ is a valid witness for

$$
0 \cdot 0 = qf + 0
$$

This breaks soundness if we consider $q$ one of the outputs of the `ForeignFieldMul` gate.

However, in most applications, we don't care about the value of $q$, and only care about $r$ as an output. With that in mind, we could interpret the `ForeignFieldMul` gate as a way to prove

$$\tag{L.10}
\exists q' \colon \quad ab = q' f + r
$$

given witnesses $a$, $b$ and $r$.

The example we just gave does **not** break soundness in this interpretation, because we only made $q$ have an invalid value, but $a$, $b$, $r$ are correct and the statement $(10)$ remains true.

In the remaining section, we aim for stronger counter-examples which provide an invalid $r$, and thus break soundness for $(10)$ as well.

#### Counter-example 1: Skipping $q_2 \in [0, 2^\ell)$

If we skip the check $q_2 \in [0, 2^\ell)$ and only use the bounds check $q_2 + (2^\ell - f_2) \in [0, 2^\ell)$, we potentially allow $q_2 \in [n - 2^\ell + f_2, n)$. The idea for exploiting this is that $q_2$ can be used to represent negative values of $q$ (modulo $n$).

> Representing negative $q$

In general, if $q = -|q|$ is negative, we can first split the abolute value $|q|$ into positive limbs $(|q|_0, |q|_1, |q|_2) \in [0, 2^\ell)^3$ as usual, and then represent $q$ as 

$$
q \cong (q_0, q_1, q_2) = (2^\ell - |q|_0, 2^\ell - 1 - |q|_1, -|q|_2 - 1)
$$

(Detail: If $|q|_0 = 0$ this doesn't work but then we just don't do the borrowing.)

In words, by borrowing from higher limbs we only need the highest limb $q_2 = -|q|_2 - 1$ to be negative. Now, $-|q|_2 - 1$ is indistinguishable in our constraints from $n-|q|_2 - 1$, and $q_2 = n-|q|_2 - 1$ is an allowed value for all $|q|_2 \in [0, 2^\ell - f_2)$. This means we can "represent" most negative $q \in (-2^{3\ell}, 0)$ by

$$
q_\text{fake} \cong (2^\ell - |q|_0, 2^\ell - |q|_1 + 1, n-|q|_2 - 1).
$$

Note that

$$
q_\text{fake} = q + 2^{2\ell} n
$$

> Attack on `ForeignFieldMul`

If we are able to use large negative $q$, the estimates for the final bound $(L.9)$ are no longer valid, which we can exploit. Given any $a$, $b$, we can find $q$, $r$ such that

$$\tag{L.9b}
ab = qf + r + 2^{3\ell} n
$$

This is always possible: we pick $q$ and $r$ to be the quotient and remainder of the modified divsion $(ab - 2^{3\ell} n) / f$. Concretely, $q = \lfloor\frac{ab - 2^{3\ell} n}{f}\rfloor$, which is a negative number (but big enough that we can always represent it by $q_\text{fake}$, if e.g. $2n < f$).

If we replace $q$ by $q_\text{fake} = q + 2^{2\ell}n$, we get a modified equation which still features the same $a$, $b$, $r$:

$$\tag{L.9c}
ab = (q_\text{fake} - 2^{2\ell} n)f + r + 2^{3\ell} n
$$

So, we want to fool the verifier into accepting $a$, $b$ and $r$ satisfying $(L.9c)$. $r$ is not the correct remainder of $ab / f$, because $2^{3\ell} n$ is not a multiple of $f$.

Since $q_\text{fake} = q \bmod n$, equation $\text{(L.2a: native)}$ is satisfied, and since we use the correct, $\ell$-bit values for $q_0$ and $q_1$, equations $\text{(L.3: split)}$ and $\text{(L.4: bottom limbs)}$ are as well (added multiples of $n$ like $2^{3\ell} n$ affect none of our constraints).

The only way our attack can sometimes fail is because $q_2$ is now "negative", the expression $p_2$ is no longer guaranteed to be positive and we might underflow 0 in $\text{(L.5: high limb)}$, which the range-check on the RHS doesn't allow:

$$\tag{L.5}
(ab)_2 + q_0f'_2 + q_1f'_1 + (- |q|_2 - 1)f'_0 + p_{11} - r_2 + v_0 = 2^{\ell} v_{1} \mod n 
$$

However, underflow caused by one negative value $(- |q|_2 - 1)f'_0$ against 5 positive values of similar size is the exception. Furthermore, there are important primes where $f'_0$ is an unusually small number:
* For Curve25519 we have $f'_0 = 19$
* For seqp256k1 we have $f'_0 = 2^{32} + 2^9 + 2^8 + 2^7 + 2^6 + 2^4 + 1$

In these cases, the negative $-|q|_2f'_0$ is small and should never cause underflow for "random" input values. In other words, the prover can use an incorrect output $r$ in almost every foreign field multiplication.

**Conclusion**: The full range check $q_2 \in [0,2^\ell)$ is necessary.

> Counter-example 2: Compact range-check on $q_0 + 2^\ell q_1$

TBC

