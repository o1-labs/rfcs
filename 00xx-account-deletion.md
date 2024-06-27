# Account deletion for consumable accounts

## Summary

The capacity to delete accounts is a prerequisite to implement *consumable
accounts* in order to propose the *zkPromise* abstraction in `o1js`.

## Motivation ##

Account deletion is a key protocol feature with the goal to enable the so-called
*zkPromise* abstraction in `o1js`.

Account deletion will allow to create as many accounts as you want, **then**
receive the deposit back when you no longer need them. This is key for zkApps
developers who want to trigger actions on other accounts in a deadlock-safe way
so that they can create a cross-account smart contracts & applications.

## Detailed Design

The support for account deletion relies on adding features at the levels of
transaction logic(s) and storage (ledgers). The latter is a prerequisite of the
former.

### Storage

<!-- sparse ledger https://github.com/MinaProtocol/mina/blob/develop/src/lib/mina_base/sparse_ledger_base.ml#L85
 !-- mask ledger https://github.com/MinaProtocol/mina/blob/develop/src/lib/merkle_mask/masking_merkle_tree.ml#L973
 !-- db ledger https://github.com/MinaProtocol/mina/blob/develop/src/lib/merkle_ledger/database.ml#L559 -->

Within the protocol-related codebase, the lowest-level support lies in having a
working and sound `remove` functions for the different ledger implementations
(`Database`, `Any_ledger`, `Null_ledger`, `Syncable_ledger`) and in masking
merkle trees.

#### <a name="merkle_trees"></a> Merkle trees

The current implementation of in-memory ledgers use a fixed-depth [Merkle
tree](https://en.wikipedia.org/wiki/Merkle_tree).

For insertion, the current implementation keeps track of the "fill frontier",
that is, the leftmost empty slot of the tree. Since removal is not possible, the
leaves between the leftmost leaf of the Merkle tree and the rightmost-filled
leaf are *all* filled with data (i.e., non-empty acccounts).

Now, on removal, there are (at least) 2 options:

1. Symbolically mark the location as removed and make this knowledge available
   at data structure level such that, upon new insertions, these available
   locations are looked at before inserting on the leftmost available
   location. This solution extends the datatype of current implementation with a
   structure tracking freed locations.

2. Switch the value currently indexed on the frontier of the tree with the
   location that will be removed, a little bit like one does when implementing a
   heap data structure. This solution keeps the insertion of new data to the
   leftmost available locations and thus solution wouldn't change the current
   datatype.

We will only discuss option 1 here and delay the discussion about option 2 in
[the alternative section](#alternative).


#### Example scenario

To illustrate option 1, let us assume we have the following Merkle tree of depth
2, with an empty free list. This Merkle tree has one location marked with `x` -
this marks the empty account.


```
    H(H(A,B),H(C,x))               free = []
      /       \
   H(A,B)     H(C,x)
   /  \       /  \
  A    B     C    x
```

The free list contains a set of locations bound to the empty account located
between the leftmost leaf and righmost non-empty leaf.  The idea is to recycle
these locations upon further addition of data, as we will see shortly.


The insertion of new data `D` results in the following Merkle tree.

```
    H(H(A,B),H(C,D))               free = []
      /       \
   H(A,B)     H(C,D)
   /  \       /  \
  A    B     C    D
```

Upon removal of `C`, the structure would evolve as follows, with `x` (the empty
account) marking the freed location.

In this example, locations will be represented as lists of directions for the
sake of readability. In practice, though, they will be represented by other,
usually isomorphic to the list of directions, means, e.g., an integer encoding
the position of the leaf.

The free list now states that the location determined by
sequence of directions `[Right; Left]` is available for reuse for new
insertions.

```
    H(H(A,B),H(C,D))       free = [[Right; Left]]
      /       \
   H(A,B)     H(x,D)
   /  \       /  \
  A    B     x    D

```

Thus removal incurs a cost proportional to the height of the tree since we have
to recompute the hashes on the Merkle path from the removed data.

Now adding data `E` would result in

```
    H(H(A,B),H(E,D))               free = []
      /       \
   H(A,B)     H(E,D)
   /  \       /  \
  A    B     E    D
```

##### Merklizing the free list

<!-- The practical usefulness of this section is in discussion  -->

The view of the free list shown in the example above is a simplification.

The data contained in the free list needs to be Merklized as well.  This means,
assuming a hash function $H$ and locations $loc_1, ..., loc_n$ being inserted in
the free list in that order, that the actual data in the free list would be:

$$
free = [ (loc_n, h_n = H(loc_n, h_{n-1})), ..., (loc_2, h_2 = H(loc_2, h_1)); (loc_1, h_1 = H(loc_1, nil))]
$$

In turn, the ledger state is the Merklized pair of the ledger itself (as a
Merkle tree) and its free list.


#### On-disk ledger

The `Database` module implements an on-disk ledger that backs up the in-memory data structure.

Deletion support relies on the same techniques as in-memory ledgers.

There are some further details to be taken care of, such as:

- updating Merkle paths on removal
- updating the `all_accounts` function to avoid iterating over addresses that
  have been removed


### Transaction logic and snark

The support for deletion at the ledger level now needs to be lifted within the
transaction logic and snark for account updates.
The key idea here is to handle removal as a specific kind of account update for
a given account.

Concretely, the basic support is provided by adding a deletion flag on account
updates, i.e., extend the `Account_update.Body.Stable.t` type with an additional
record field `delete_account`.

This turns the current definition into
```ocaml
      (* This will update the version of the correspond Mina_wire_types definition too).
      type t = Mina_wire_types.Mina_base.Account_update.Body.V2.t =
        { public_key : Public_key.Compressed.Stable.V1.t
        ; token_id : Token_id.Stable.V2.t
        ; update : Update.Stable.V1.t
        ; balance_change :
            (Amount.Stable.V1.t, Sgn.Stable.V1.t) Signed_poly.Stable.V1.t
        ; increment_nonce : bool
        ; events : Events'.Stable.V1.t
        ; actions : Events'.Stable.V1.t
        ; call_data : Pickles.Backend.Tick.Field.Stable.V1.t
        ; preconditions : Preconditions.Stable.V1.t
        ; use_full_commitment : bool
        ; implicit_account_creation_fee : bool
        ; may_use_token : May_use_token.Stable.V1.t
        ; authorization_kind : Authorization_kind.Stable.V1.t
         (* if true, this account update will delete the account *)
        ; delete_account: bool;
        }
```

This extension therefore provides a version 2 of the current versioned type (`V2`).

This simple change trickles down into updating the transitive closure of all
type definitions depending on `Account_update`, including
`Mina_wire_types.Mina_transaction`.

Deleting an account also provokes a payment of the initial account
creation fee back, and its MINA balance, to an address.

Clearly, account deletion should not be permitted except for authorized keys, e.g., the
contract that create the consumable account. For that, we need to check this
with respect to the `permissions` field of `Account_update.Update.t`.


At the snark level, removing an existing account is handled as setting the
location to the empty account using the already existing `set_account`.

Now similar to account creationg, one needs to check the resulting balance.


<!-- user command snark https://github.com/MinaProtocol/mina/blob/develop/src/lib/transaction_logic/mina_transaction_logic.ml#L1007
 !-- zkapp command snark https://github.com/MinaProtocol/mina/blob/develop/src/lib/transaction_logic/zkapp_command_logic.ml#L1232 -->

### Updating the archive node

The addition of a field in `Account_update` will need to reflected in table
`zkapp_account_update_body` with the addition of a `delete_account` column.

The migration of the current database needs to set a `false` value in this new
column for all already registered rows.



### Other expected code changes
#### Ledgers

#### Removal

We propose to support removing elements in ledgers through  2 functions, that we specify below:

- `val remove_location: t -> location -> unit`: remove the account found at the
  given `location` from the ledger, don't do anything if `location` is occupied
  by the empty account
- `val remove_account: t -> account -> unit`: remove `account` from the ledger,
  don't do anything if `account` does not exist.

While both are not needed, since one can usually be easily derived from the
other, we argue that it is nicer to have these 2 functions be provided in the
interface.

#### Changes to `last_filled`

The current ledger interface also exposes the following function:
```ocaml
(** for account locations in the ledger, the last (rightmost) filled
    location
*)
val last_filled : t -> Location.t option
```

The usage of this function relies on the implicit invariant that data are never
removed, thus the frontier location, as computed by `last_filled` is directly
related to the next location we can allocate.

This assumption breaks upon the introduction of the remove function we sketched
before.

There are two uses of this function

- [sparse_ledger](https://github.com/MinaProtocol/mina/blob/develop/src/lib/mina_base/sparse_ledger_base.ml#L39)
  Now that we would have two sources of free locations, we propose to integrate the computation done
  here in the `Ledger` interface.
- [util](https://github.com/MinaProtocol/mina/blob/develop/src/lib/merkle_ledger/util.ml#L168)

#### General protocol changes

The update to `Account_update.t`, though seemingly straightforward has a trickle
down effect, due to the impact of versioning.

All types dependent on type `Account_update.t` will need updating.  This single
change is relatively massive so that adding the extra field should really be a
single task to help reviewers.


#### o1js

The API used by `o1js` is not expected to change much but for some details.

- *accounts update* : we propose to implement account deletion as a specific
  kind of account updates so that users of this API can specify whether to
  delete a given account during a transaction.

  In its simplest form, this entails the addition of Boolean argument at the API
  level stating whether to delete or not. <!-- We propose to add a dedicated, more
   !-- explicit, algebraic datatype of the form:
   !--
   !-- ```ocaml
   !-- type account_update = Delete | Update
   !-- ``` -->

- *recipient* : upon deletion the interface should allow to specify who should
  the recipient of the account creation fee.




## Test plan and functional requirements

- **Testing goals and objectives**:
  - Check the correctness of the account deletion feature;
  - Validate the absence of impact of the feature on the existing functionalities.
- **Testing approach**:
  - Unit tests for the ledger implementations;
  - Integration tests for the extension of transaction logics.
- **Testing scope**:
  - Testing sequences of intertwined addition/removal commands;
  - Testing simple high-level account updates;
  - Testing from `o1js` typical use-cases scenarios.
- **Testing requirements**:
  - Test in various local setups to simulate different development environments.
- **Testing resources**:
  - Simulated environments resembling typical local development setups.


## Drawbacks

The main drawback of implementing account deletion is that it needs a hard fork
to be activated on-chain.

On the surface, the changes might seem trivial (adding a removal function,
adding a field in a data structure) but they they actually have a widespread
impact on the codebase (50+ files are touched just for adding the aforementioned
field addition so that the our codebase compiles). We thus need to be extra
careful both in the coding *and* the reviewing phases.

## <a name="alternatives"></a>Rationale and alternatives

### Merkle trees

For the sake of completeness, let us consider another option for implementing
removal at the level of Merkle trees, which aims at maintaining the fact that
any new insertions happens on the leftmost available location, and that all
leaves between the leftmost one and the rightmost non-empty one are filled.

In this case, removing `C` would not update any free list -- since this solution
does not need this concept. But it would change the location of `D`.

```
    H(H(A,B),H(D,x))
      /       \
   H(A,B)     H(D,x)
   /  \       /   \
  A    B     D     x
```

Since the ledger implementation also tracks mapping from account to location as
well as location to accounts, both these mappings would need updating.

Now adding data `E` would result in

```
    H(H(A,B),H(D,E))
      /       \
   H(A,B)     H(D,E)
   /  \       /  \
  A    B     D    E
```

##### Implementation choice

We opt to implement the tracking of freed locations as illustrate in [this section](#merkle_trees).

While this choice implies adding a structure in the masking ledger
implementation to track freed locations, it offers 2 advantages:

* the cost of removal is slightly better since we do not need to update location
   <-> account mappings;
*  account locations in this representation are fixed for the lifetime of the
  account: proof of inclusion would not change but for a single element of the
  Merkle path.


## Prior art

## Unresolved questions

- Merklization of ledger + free list and interface for interacting with this data structure


<!-- ## Resources - TB removed
 !--
 !--
 !--  Here are some things I am thinking about, maybe they help you inform the RFC:
 !--
 !--  - easy API - we need to express account deletion via an easy API in the sdk
 !--  - ideally, this should also be efficient (maybe a single "instruction")? How would this look like as a transaction?
 !--  - will we be able to assert that an account has been deleted? eg look up existing accounts, or put a precondition on it that says "account must be deleted"?
 !--  - recover the original account creation fee to a predetermined address or creator
 !--  - What requirements does an account have to fulfill in order to be deleted?
 !--  - Regarding consumable accounts as a whole, how will they look like? Will they be their own specific type of account or a full account as we know currently? -->
