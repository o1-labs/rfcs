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

## Design

The support for account deletion relies on adding features at the levels of
transaction logic(s) and storage (ledgers). The latter is a prerequisite of the
former.


### Storage

Within the protocol-related codebase, the lowest-level support lies in having a
working and sound `remove` functions for the different ledger implementations
(`Database`, `Any_ledger`, `Null_ledger`, `Syncable_ledger`) and in masking
merkle trees.

#### Merkle trees

Current implementation has an index of the next location to fill on insertion,
that is, the leftmost empty slot of the tree.

Now, on removal, there are (at least) 2 options:
    1. Symbolically mark the location as removed and make this knowledge
       available at data structure level such that, upon new insertions, these
       available locations are looked at before inserting on the leftmost
       available location.

    2. Switch the value currently indexed on the frontier of the tree with the
       location that will be removed, a little bit like one does when
       implementing a heap data structure.

Each option would correctly implement the needed support


### Transaction logic

<!-- Locations to track
 !--
 !-- user command snark https://github.com/MinaProtocol/mina/blob/develop/src/lib/transaction_logic/mina_transaction_logic.ml#L1007
 !-- zkapp command snark https://github.com/MinaProtocol/mina/blob/develop/src/lib/transaction_logic/zkapp_command_logic.ml#L1232
 !-- sparse ledger https://github.com/MinaProtocol/mina/blob/develop/src/lib/mina_base/sparse_ledger_base.ml#L85
 !-- mask ledger https://github.com/MinaProtocol/mina/blob/develop/src/lib/merkle_mask/masking_merkle_tree.ml#L973
 !-- db ledger https://github.com/MinaProtocol/mina/blob/develop/src/lib/merkle_ledger/database.ml#L559 -->

## API

The API used by `o1js` is not expected to change much but for some details.

- *accounts update* : we propose to implement account deletion as a specific
  kind of account updates so that users of this API can specify whether to
  delete a given account during a transaction.

  In its simplest form, this entails the addition of Boolean argument at the API
  level stating whether to delete or not. We propose to add a dedicated, more
  explicit, algebraic datatype of the form:

  ```ocaml
  type account_update = Delete | Update
  ```

- *recipient* :
