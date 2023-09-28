# Folding

## Summary

This RFC proposes the addition of folding support to kimchi, folding is a technique which basically allows
to verify two proofs for the price of one, enabling the development of efficient IVC.

## Motivation

The ZkVM currently supports running a fixed number of cycles, to support an arbitrary number of cycles we will
need some kind of recursion that allows us to combine blocks of cycles in a way that will allow constant size
proofs.
Pickles is able to do it, but not while fulfilling the performance requirements for the ZkVM.
Folding is a less powerful alternative, but powerful enough while offering superior performance, especially in
the specific scenario where the ZkVM has to operate.
It is not a goal to replace pickles, just provide an alternative for cases like where folding can be a better
option.

## Detailed design

### Overview of folding

The general idea of folding is a protocol which starts with 2 instance-witness pairs, the prover has access to
everything, while the verifier only to the instance.
At the end of the protocol, a new instance-witness pair will be created, with the property that verifying it implies
verifying the 2 original pairs.
That way, the prover only has to do 1 actual proof, and the verifier only has to verify 1 proof, the new pair can
become an input of the protocol too, thus allowing to fold an arbitrary number of instance-witness pairs into a
single one.

### Folding in kimchi

For kimchi, the witness is the same witness we already have: a list of columns/polynomials. And the instance is list of the
commitments to those polynomials, plus the public inputs.
Additionally, to make folding work we need to add 2 things that will absorb errors generated during the folding, one is just
a field element $u$, the other is an extra witness column $E$ with its corresponding commitment $\overline{E}$ in the instance.
We will call this new kind of instance a relaxed kimchi instance (equivalent to the committed relaxed R1CS instance in Nova,
but we already had commitments so the only change is the relaxation).
A normal kimchi instance-witness pair can be trivially converted into relaxed kimchi by just setting $U=1$ and $E=[0;N]$
The verifier part of the protocol is relatively simple, get some commitment for the errors generated, make a challenge, send
the challenge to the prover and take a linear combination of the instances using the challenge.
The prover is doing the same with the witness after computing the error and committing to them.

### Implementation

The implementation will just be a few functions, working with relaxed instances and witnesses:

```rust
struct Instance;
struct Witness;
struct Challenge;
struct Error(Evals);

fn compute_error(a: &Instance,b: &Instance) -> Error;
fn fold_instance(a: Instance, b: Instance, error: Error) -> (Instance, Challenge);
fn fold_witness(a: Instance,b: Instance) -> (Instance, Challenge);
```

Additionally, it may be necessary to create some transformation that will add the errors to the kimchi equation so that the
verification will work again.
Given that another RFC will make the degree fixed to 2, it should be possible to make a function that will map expressions
to the their relaxed version and apply it to each expression used.

### Transformation

We can think that all kimchi expressions will ultimately aggregate into a multivariate polynomial, thanks to the quadricization
RFC we can additionally assume it to be of degree 2 or less.
For each term we will have to do a change to the expression and add to the error, we have 3 cases for constant, degree 1 and
degree 2 terms that will require different changes to the expression and contribute in a different way to the error.

#### Expression changes

- Constant: Will be multiplied by $u^2$
- Degree 1: Will be multiplied by $u$
- Degree 2: Unchanged.

#### Error contribution

In general, when taking the linear combination of a term, you will get 3 parts, each multiplied by $1$, $r$ or $r^2$. The
part multiplied by $r$ is the error contribution.
Here $u_1$ and $u_2$ are the scalar from each of the original instances that become the $u = u_1 * ru_2$ of the new folded
instance, same notation is use when needed for witness columns.
Also, $S$ will represent a selector, which can be multiplying in the term, but does not affect the degree.

- Constant: For some term $P$, the contribution will be $r2u_1u_2P$, we can likely add all terms together and multiply by
  $r2u_1u_2$ at the end.
- Degree 1: For some term $P$, the contribution will be $r(u_1P_2 + u_2P_1)$, similar optimization as above should be possible.
  here $P_1$ for example just means that for computing the expression we use a column from the first instance (the one with $u_1$).
- Degree 2: For some term $P$, that contains $ab$ from the witness, the contribution will be $rS(a_1b_2 + a_2b_1)$

Given that they all contain $r$, it can be omitted and multiplied to the entire $E$ at the end.

## Test plan and functional requirements

1. Testing goals and objectives:
    - The only requirement would be for it to work, it doesn't need backwards compatibility given that it will be first used
    in the ZkVM.
2. Testing approach:
    - Given that it is new and not a replacement for pickles, we are likely limited to unit test it.
    - When the ZkVM is ready, just running it with folding would be an useful test.

## Drawbacks

There isn't much not already mentioned about the comparison with pickles.

## Rationale and alternatives

The alternative would be pickles, which as explain at the start, wouldn't meet the performance requirement for the ZkVM.

## Unresolved questions

- We should probably add the equations here to some detail.
- It is probably better to explain in detail the process to convert the kimchi expressions in their relaxed version.
