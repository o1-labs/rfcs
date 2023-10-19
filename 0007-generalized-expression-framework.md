# Generalized expression framework

## Summary

The goal of this RFC is to generalize the expression framework for the kimchi proof system, so that different columns may be used for different configurations -- for use by zkVMs and customizable kimchi -- and so that we can manipulate expressions to introduce intermediate columns (e.g. for folding).

## Motivation

The goal of this RFC is to generalize the design of the expression framework, which is used by kimchi to describe the polynomial constraints that must be satisfied by a given proof. This goal will be met when it is possible to describe constraints over a different set of columns and challenges than those hard-coded in the berkeley version of the kimchi proof system.

In particular, the requirements are that zkVM instructions can be encoded as an expression under the framework, without modifying or otherwise affecting the expressions used for the berkeley specialization of the kimchi framework.

We can measure progress towards this goal by executing an intermediate task. The 'cairo gates' defined in the kimchi proof system are currently unused in berkeley, but the selectors for these gates are [hard-coded in the list used for the berkeley expressions](https://github.com/o1-labs/proof-systems/blob/17041948eb2742244464d6749560a304213f4198/kimchi/src/circuits/gate.rs#L102); when this RFC is successfully implemented, it should be possible to define those constraints over a cairo-specific set of columns, which in turn will allow those variants to be removed.

We can also confirm progress by integrating a currently-unused gate (for example the ['generic boolean gate'](https://github.com/o1-labs/proof-systems/pull/940)), to demonstrate that it is possible to add and integrate a new gate without modifying the existing behaviour of the berkeley kimchi proof system.

This proposal is optimized for code re-use: the existing framework is familiar to the crypto team, and provides useful abstractions for creating and reasoning about proofs over different sets of polynomial constraints. In particular, this avoids the need to fork and maintain separate versions of the expression framework for each different proof system.

This RFC will result in our ability to implement 'customizable kimchi' -- a version of the kimchi proof system that accepts user-defined polynomial constraints -- as well as the ability to write proofs over arbitrary hard-coded polynomial constraints for the zkVM and related projects. The upcoming keccak RFC will also build upon this as its basis.

## Detailed design

Currently, the [expression framework](https://github.com/o1-labs/proof-systems/blob/master/kimchi/src/circuits/expr.rs) is specialized for those constraints that are required for the berkeley hard-fork's version of kimchi. This RFC proposes the following modifications:
* abstract over the `Environment` type in `Expr`, to allow for different execution environments which require different sets of columns and constants;
* abstract over the `Column` type in `Expr`, adding the corresponding helpers via a rust trait definition, which allow it to access the values in `Environment`;
* abstract over the `Constants` type in `Expr`, adding the corresponding helpers via a rust trait definition, which allow it to access the values in `Environment`;
* add helpers such as `map_column`, which allow an expression over one set of columns to be transformed into an expression over another set of columns.

The precise set of modifications should be defined by the state of the expression framework at the time of this RFC's implementation. However, the most important modifications are:
* transforming the definition of `get_column` into a trait instance, defined on `Environment` and parameterised by the abstracted input type `Column`;
* abstracting the printing helpers to support arbitrary `Column`s and `Constant`s, especially for the `latex` printer (which will be used to generate specifications), and the `ocaml` printer (which will be used to generate the code for scalar checks in pickles);
* `ConstantExpr` will need to be divided:
  - `ChallengeExpr`s should hold the challenges generated by the Fiat-Shamir transformation (or received from the verifier as part of an interactive execution);
  - `DomainConstantExpr`s should hold the constants that are derived from the choice of the proof's domain size, but *not* specific to a particular circuit or proof system;
  - `ConstraintConstantExpr`s should hold the constants that are derived from the proof system's specifics (for example the 'MDS' for poseidon in the berkeley configuration);
  - `ConstantExpr` should contain constant literals, wrappers around the other sub-types of `ConstantExpr`, and the results of arithmetic operations (add, mul, sub).

As part of these modifications, several of the 'helpers' for constructing the expressions required for the berkeley constraints will become over-specialized. The more general constructors should be added in their place, and the existing helpers should be moved to live alongside the berkeley trait implementations that they will require.

The berkeley specializations should be separated from the core expression framework; the types for the newly-abstract constraints should live together in one code file, alongside the trait implementations that make it usable by the expression framework.

The traits should be accompanied by user documentation, so that they can later be used when custom gates or constraints are to be created by users of kimchi / SnarkyJS / pickles.

**Still TBD:** Should we remove the leaky abstraction around `Argument` and `Alphas` as part of this work?

### Performance

Adding these extra layers of abstraction will result in slower compile times, and may also lead to a slight increase in binary executable size. The performance of the expression framework itself should not change; the proving performance should be benchmarked before and after to confirm that this is the case.

## Test plan and functional requirements

1. Testing goals and objectives: 
    * The proofs generated by the system after this abstraction must be compatible with the proofs accepted by the Mina berkeley RC using the version before this RFC.
    * The performance of the system must be unchanged.
    * The expression framework can be used with a distinct constraint definition, over distinct challenges and columns.
2. Testing approach: 
    * Compile Mina with the versions from before and after this change, running a local network with both versions connected, to confirm that they are still able to communicate.
    * Run a test benchmark with both versions of the proof system, ensuring that the performance change is negligible after these changes.
    * Create and execute a test of a new custom constraint (e.g. the 'generic boolean' gate), ensuring that the main primitives of the expression framework still allow for its evaluation and use.
3. Testing scope: 
    * The existing properties of the proof system are preserved.
    * The expression framework is able to be used for new specializations of the proof system.

## Drawbacks
[drawbacks]: #drawbacks

The additional overhead of compilation will cause a slow part of the Mina daemon's compilation to become even slower.

This also adds more 'generic parameters' to track and reason about at the Kimchi level, which both makes it harder for developers to reason about, and results in a potentially exponential blow-up in configurations that need to be tested.

This also increases the size of the code to perform the same operations, primarily due to the additional boilerplate to define and implement the abstraction.

## Rationale and alternatives

### Why is this design the best in the space of possible designs?

This allows us to re-use the existing code and institutional knowledge as much as we can, while still giving us the flexibility of expanding the proof system in the near-future directions that we have planned.

### What other designs have been considered and what is the rationale for not choosing them?

The most appealing, tried, and tested alternative is to fork the kimchi repository -- or just the expression framework -- to allow for a different set of columns or constants. However, this results in a large amount of duplicated code, especially in some of the more complex helpers, and isn't compatible with a generalized 'customizable kimchi/pickles' proof system.

### What is the impact of not doing this?

This change (or the alternative above) is required to support different proof systems (zkVM, customizable kimchi, etc.). We cannot support them without an effective way to represent their polynomial constraints.

## Prior art

There is some existing work in [this PR](https://github.com/o1-labs/proof-systems/pull/1103) (stale), which validates the hypothesis that it is practical and performant to make a subset of these changes. That work also highlights the additional complexity and boilerplate required to make this work as intended.

## Unresolved questions

### What parts of the design do you expect to resolve through the RFC process before this RFC gets merged?

See TBD in the design description around `Alphas` and `Argument`.

### What parts of the design do you expect to resolve through the implementation of this feature before merge?

Specific implementation details of the abstraction, which will be guided by the limitations of the rust type system and trait mechanism.

### What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

Other proposed modifications to the expression framework; naming or structure of the columns for the berkeley-compatible kimchi; definitions of berkeley constraints.