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
so that I can create a cross-account smart contract & application.

## Design

Within the protocol-related codebase, the lowest-level support lies in having a
working `remove` functions for the different ledger implementations (`database`,
`any_ledger`, `null_ledger`) and in masking merkle trees.

## API
