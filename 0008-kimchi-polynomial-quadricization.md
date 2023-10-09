# Kimchi polynomial quadricization

## Summary

This RFC proposes a protocol for converting kimchi polynomial constraints (as defined by the expression framework) to polynomials of degree at most 2.

## Motivation

This RFC provides a mechanism for reducing an arbitrary-degree polynomial constraint to a polynomial with degree of at most 2, by inserting additional intermediate columns. This is essential for supporting a [Nova-style](https://eprint.iacr.org/2021/370) folding scheme with reasonable efficiency, since it depends on distributing an error term across the 'cross-terms' of a multiplication.

This will form part of a general scheme in kimchi to provide 'folding' as a component of proofs. In particular, this will enable arbitrarily long circuits to be run against the bn128 curve backend, where we do not currently have a scheme for using recursion.

The exit condition for this work is functionality in kimchi's expression framework that 'flattens' an expression into an expression of degree 2, synthesizing auxiliary columns to facilitate this as it does so. This work will be structured such that it can be used in the zkVM proposed for the OP zk RFP.

## Detailed design

In Nova-style folding, multiple witnesses are combined using the observation that

```math
(a_0 + r a_1) * (b_0 + r b_1) = (a_0 * b_0) + r^2 (a_1 * b_1) + r (a_0 * b_1 + a_1 * b_0)
```

We call the final term the 'cross-term', which accounts for the 'error' in the multiplication, while the other terms correspond to the 2 multiplication results that were desired, combined with some random challenge `r`.

The cross-term gets more complex as the degree of expressions increase, by fixing the degree to be at most 2 we make folding simple by having a smaller degree and also by not having to handle different degrees.

Considering this example, we have 3 columns a, b and c. We also have the next expression over them:

```math
a*b*c + a*b + a 
```

we can lower the degree by introducing an additional column ab, that is constrained to be $bc = b*c$ with a degree 1 constraint,
now the expression can be rewritten as the next expression where the degree is lowered by 1:

```math
a*bc + a*b + a 
```

Notice how we could also have chosen ab and it could have even been more efficient if also used for the second term.

For more complex expressions like this one:

```math
a*b*c + a*a*b + b*b*a + a*a
```

More columns will need to be added, and the choice is no longer trivial, we could do it with 3 columns $ac$, $aa$, and $bb$.
Or we could also use a single column $ab$ that would work for all the terms

To achieve an optimal quadratic expression we would do the next changes:

- Most expressions are combined at the gates level, but the gates never combine into a single expression, thus we have to flatten
in an optimal way through several expressions. The first change would be to identify the expressions we care about and extract them from
the prover into a location in the codebase were we can reuse them without duplication.
- Later at some point, could be just once at constraint system creation, we will run some algorithm to decide which extra columns to create,
or which sub expressions to extract into a column in a way that optimizes for the lowest number of extra columns, the result will be some
struct that records which sub expressions were extracted.
- We will create a function that takes an expression of any degree and transforms it into an equivalent quadratic expression making use of the
optimal flattening we created previously.
- What remains would be to apply this function in the prover and other places were we need to flatten the expressions.

The next example would likely require the generalized expression framework

```rust
fn reduce(self: E<F, Column>, flattening: &OptimalFlattening) -> E<F, WithAuxiliaries<Column>> {
}
```

## Test plan and functional requirements

1. Testing goals and objectives:

    - The flattened expressions should be equivalent to the original expressions.
    - The flattening algorithm should be optimal in terms of the number of columns added.

2. Testing approach:

    - For equivalence we can run evaluate the same expression twice with only one being flattened and check the equality of the results.
    - The optimal flattening is in some way impossible to check without just duplicating the problem, it should be enough to measure it
    and just check that it looks reasonable to us.

## Drawbacks

The main drawback of this is that we had a reason for high degree gates, they provided improvements that will be in some way lost, but given
that this is optional it will be a question of choosing the best option for each, folding in particular will benefit from it.
Proving time may increase, the size of proofs will increase, pickles recursive circuit (that won't likely be used together with this) will increase,
the folding-based IVC circuit we will have to build will grow for each row we add, but it isn't trivial to know if it will outweigh the reduction
due to the smaller degree.

## Rationale and alternatives

The alternative is not doing it, with the problems mentioned at the start.
In a more general way we could use some PCD construction instead of folding-based IVC, pickles wouldn't work with the new curve without changing KZG for a commitment like the one we currently use in kimchi but decided to change by KZG due to the EVM running costs.
We could make pickles support KZG, but that would require to implement some complex and potentially expensive cryptography inside of pickles.

## Prior art

The most similar to a prior art would pickles itself, which we can not use for VM mainly due to performance issues.

## Unresolved questions

- The specific algorithm to find an optimal flattening, it may be a problem without an efficient solution, but given that the degrees we find in practice are relatively small we can probably run even an exponential algorithm without any issue.
