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

For code links, we will use the head of branch `develop` at the time of writing, i.e.,
[commit 4495af5](https://github.com/MinaProtocol/mina/tree/4495af5caea5e1bb2f98f92592c065f93a586ade).


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

##### <a name="free_list repr"></a>Representation of the free list

The view of the free list shown in the example above is a simplification.

The contents of this list need be easily shareable, e.g., to synchronize a
bootstrapping node. In order to achieve that, the data structure representing
the free list needs to be *ordered*.

The ordering can be derived from the timestamp the locations are freed during
the lifetime of the chain. This would result in using a *stack* (for a LIFO
allocation) or a *queue* (for a FIFO allocation). However, this choice has some
drawbacks (see [alternative](#free_list_repr alt)), most importantly that a
newcomer node cannot have any knowledge of the past, so that the free list would
need to be fully communicated to it.

**Imposing an ordering** from the structure of the data that we insert in the
free list has better properties. Heaps, `OCaml` sets (which rely on a comparison
function), or even ordered lists are implementation choices that would work in
this context.

We thus propose to use a **max-heap** (like
[bheap](https://opam.ocaml.org/packages/bheap/) or
[CCHeap](https://c-cube.github.io/ocaml-containers/last/containers/CCHeap/index.html))
for this implementation, that is, the free list is an ordered set of locations,
from which we always allocate the biggest one.

**Deallocation at fill frontier** shows why we need to choose to reallocate the
maximal element of the free list.

When freeing the location at the fill frontier, we also need to check whether the preceeding location in the order of
Merkle tree leaves is free or not to compute the new frontier pointer.

For example if we have data `[A, B, x, E]` in the tree, the frontier would be at
index `3` with a free list `[2]`. Removing `E` should result in a frontier at
index `1` and an empty free list. The max-heap data structure handles that
scenario very gracefully in worst-case O(n log n), where $n$ is the number of
elements in the free list.  A stack structure could incur an *O(n²)*
worst-case cost. The min-heap would not be adequate.


**The main benefit** of having this ordered data structure is that location
allocation does not depend on when a location has been freed anymore. Thus, when
syncing ledgers, the free list can be recomputed while retrieving the ledger by
adding any empty location within the fill frontier to the currently built free list heap.

There is a single caveat: the comparison function for locations needs to be
*total* for this choice to work. The implementations of `Location` in the
codebase all have this property.

#### On-disk ledger

The
[`Database`](https://github.com/MinaProtocol/mina/blob/4495af5caea5e1bb2f98f92592c065f93a586ade/src/lib/merkle_ledger/database.ml)
module implements an on-disk ledger that backs up the in-memory data structure.

Deletion support relies on the same techniques as in-memory ledgers, that is, a
representation of a ledger as Merkle tree *and* an additional free list.

The main culprit is the representation of the free list on-disk. Here we will simply store
the data on the KV database as follows:
- the free list is identified by the key `free_list` (like
  [`last_account_location`](https://github.com/MinaProtocol/mina/blob/4495af5caea5e1bb2f98f92592c065f93a586ade/src/lib/merkle_ledger/database.ml#L235));
- store the serialized encoding of the heap in the database.

**Space requirements**. We argue that this is okay to have this simple scheme in
terms of space use.  Indeed, in the worst case today, the ledger is full and we
do need to store it in the database.  The requirement of storing the free list
alongside the ledger have the same worst-case scenario. When deleting an account
one is trading the storage of an account for the storage of a location, which is
usually smaller (a single integer instead of a complex data structure).
Therefore storing free locations does not add more storage requirements than
today.

The serialization of and array-based heap (ike `bheap`) of integers should be rather efficient.

**Allocation cost**. Allocating locations from the free list now means
1. read `free_list`
2. deserialize the heap value;
3. get the needed free locations;
4. write the accounts to the free locations;
5. reserialize the new value of the heap.

**Simple improvement on batch allocations**. Furthermore, we will implement a
decoding function where one can specify how many locations we ideally want to
retrieve (up to `n`) to handle functions `set_location_batch`.  This function
will return a list of free locations together with its size (so that we do not
need to call `List.length`) in order to know if we need further allocation
starting from the fill frontier index.  This function avoids
deserializing/serializing multiple times when we know in advance we will do
multiple allocations.

There are some further details to be taken care of, such as:

- patch function [`allocate`](https://github.com/MinaProtocol/mina/blob/4495af5caea5e1bb2f98f92592c065f93a586ade/src/lib/merkle_ledger/database.ml#L267) to handle the free list as well
- updating Merkle paths on removal
- updating the `all_accounts` function to avoid iterating over addresses that
  have been removed

Other strategies are discussed [here](#free_list_db alt).


#### Syncing ledgers

The current implementation of [syncable
ledger](https://github.com/MinaProtocol/mina/blob/4495af5caea5e1bb2f98f92592c065f93a586ade/src/lib/syncable_ledger/syncable_ledger.ml)
assumes the allocated addresses are contiguous (see [this
line](https://github.com/MinaProtocol/mina/blob/4495af5caea5e1bb2f98f92592c065f93a586ade/src/lib/syncable_ledger/syncable_ledger.ml#L301))
when sending data.

A ledger supporting account deletion can have unfilled leaves, which breaks this assumption.
Thus, we need to devise a suitable strategy to transfer this data.

We propose to maintain the current syncing protocol with the following simple
extension: send multiple chunks of contiguous intervals of allocated addresses
instead of a single one that assumes no unallocated address as the result of [this code block](https://github.com/MinaProtocol/mina/blob/4495af5caea5e1bb2f98f92592c065f93a586ade/src/lib/syncable_ledger/syncable_ledger.ml#L297).

On the other side of the synchronization,
[add_content](https://github.com/MinaProtocol/mina/blob/4495af5caea5e1bb2f98f92592c065f93a586ade/src/lib/syncable_ledger/syncable_ledger.ml#L408)
needs to be handle this correctly and this in turn relies on
[set_all_accounts_rooted_at_exn](https://github.com/MinaProtocol/mina/blob/4495af5caea5e1bb2f98f92592c065f93a586ade/src/lib/syncable_ledger/syncable_ledger.ml#L419).
The change here will mainly be to call ``set_all_accounts_rooted_at_exn` as many
times as there are chunks in the data.



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

#### Effect of deleting an account

Deleting an account $a₀$ results in two actions:
1. Actual removal of account $a₀$ of the storage layer;
2. Return of balance of MINA tokens from $a₉$ and initial creation fee to a specified $a₁$ account.

Handling other tokens is the responsability of the smart contract(s) dealing with these tokens.

At the snark level, deletion should thus be equivalent to proving two things
1. The location $l$ of account $a₀$ is now  equal to the empty account.
   A simple implementation is to call [`set_account`](https://github.com/MinaProtocol/mina/blob/4495af5caea5e1bb2f98f92592c065f93a586ade/src/lib/transaction_snark/transaction_snark.ml#L1531) to location $l$ with the empty account;
2. The change in MINA of $a₁$ should be equal to
   - the initial creation fee;
   - augmented by any remaining MINA balance.


**Permissions**. Furthermore, account deletion should not be permitted except
for some authorized participants, e.g., the contract that created the consumable
account. We will extend the [permissions
type](https://github.com/MinaProtocol/mina/blob/4495af5caea5e1bb2f98f92592c065f93a586ade/src/lib/mina_base/permissions.ml#L343)
with a dedicated `delete` field.  The handling of permissions will be similar as
the one for [nonce
update](https://github.com/MinaProtocol/mina/blob/4495af5caea5e1bb2f98f92592c065f93a586ade/src/lib/transaction_logic/zkapp_command_logic.ml#L1687)
for the computation of `has_permission` and the updated `local_state`. This
updated local state will also depend on extending [`Failure.t`](https://github.com/MinaProtocol/mina/blob/4495af5caea5e1bb2f98f92592c065f93a586ade/src/lib/mina_base/transaction_status.ml#L9) with an extra variant `Update_not_permitted_delete`.




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

We propose to support removing elements in ledgers through 2 functions, that we
specify below, using types definde in the ledger interface
[here](ttps://github.com/MinaProtocol/mina/blob/4495af5caea5e1bb2f98f92592c065f93a586ade/src/lib/merkle_ledger/intf.ml#L278)

- `val remove_location: t -> Location.t -> unit`: remove the account found at the
  given `location` from the ledger, don't do anything if `location` is occupied
  by the empty account
- `val remove_account: t -> account_id -> unit`: remove the account that has identifier `account_id` from the ledger,
  don't do anything if this account does not exist.

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

### Implementation steps

The implementation of deletable accounts can be broken down in 3 steps
- Ledger changes to get basic deletion working + unit testing;
- Add ledger synchronization and test it: upon completing this step, the set of
  changes implemented could even be part of soft-fork.
- Transition support:
  - Extend account updates : upon completion, the SDK team can start working on
    their end to see if the API works for them, though no effect can be
    observed;
  - Add actual support in the logic and the snark

### Testing

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


### <a name="free_list repr_alt"></a>Representation of free list


An alternative [this data representation](#free_list_repr) is to use a stack
representation. Whenever a location is freed, it is thus added on top of the
stack.

The data contained in the free list needs to be Merklized as well, in order to
be easily transmitted during node bootstrapping.  This means, assuming a hash
function $H$ and locations $loc_1, ..., loc_n$ being inserted in the free list
in that order, that the actual data in the free list would be:

$$
free = [ (loc_n, h_n = H(loc_n, h_{n-1})), ..., (loc_2, h_2 = H(loc_2, h_1)); (loc_1, h_1 = H(loc_1, nil))]
$$

In turn, the ledger state is the Merklized pair of the ledger itself (as a
Merkle tree) and its free list.

The main drawback with the choice described in this alternative is that we
*need* to transfer the data during synchronization since the order in which the
elements are found in the list cannot be inferred from the ledger.


### <a name="free_list_db alt"></a>Free list in database

There are other possible representations of the free list in key-value store.

#### No storage

For one, to follow the strategy which allocates the biggest available address
within contained within the fill frontier first, we could just scan from the
fill frontier index downto the leftmost leaf, and allocate the first available slot.

This takes O(n) time.

To save scanning when the ledger is full within the fill frontier, we could just
store how many free locations there are in the database. However, an update to
the database would probably require a full scan anyway to update this value so
that it does not seem as appealing.

#### Other storage strategies

One could devise a number of alternate strategies which would try to amortize
the cost of deserializing the heap, without trying to improve cost of storing
the heap.

The database would keep $n$ indices with dedicated keys $loc_{i}$ and an index
of the current one.  Once these are all used, we would deserialize the heap
once, fill the $n$ $loc_{i}$ keys with the value and reserialize the resulting
heap. One drawback of this option is that finding the right $n$ would need
specific testing.


## Prior art

## Unresolved questions

- Merklization of ledger + free list and interface for interacting with this data structure
- Upon account deletion, do we want to be able to return the creation fee to one
  account and the balance to another?

  Is account deletion only possible on
  accounts with a MINA balance of 0, so that we only deal with returning the
  creation fee in the transaction logic?

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
