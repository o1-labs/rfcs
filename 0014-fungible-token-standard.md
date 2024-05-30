# Fungible Token Standard on Mina

## Summary

Fungible tokens are a pervasive feature that is used for many applications, such as bridges, decentralised exchanges, layer 2 solutions, and others. Establishing a standard for creating fungible tokens on Mina will reduce the efforts for builders to create their own tokens, and allow exchanges, explorers, and other third party applications to integrate fungible tokens in a uniform way.

## Motivation

Fungible tokens are a common feature on programmable blockchain systems. Establishing a standard will help developers to create their own fungible tokens and to interface their application with tokens that others have created.

Beyond standardising an API, we want to have a standard _implementation_ of fungible tokens that people should use when deploying a token. The reason for that is that in the off-chain execution model of Mina, integrating with a `SmartContract` does require having access to the code of that contract, and running it in your own environment. If each token were to use a different contract, that would put a significant burden on wallets or other applications that want to interface with arbitrary tokens.

In order to allow for some flexibility without changing the token contract itself, we separate out administrative actions like minting new tokens to a special admin contract. The token contract will call methods from the admin contract when trying to mint new tokens, and the admin contract can implement arbitrary logic to determine whether the minting should be permitted. That way, we can move all the custom code to the admin contract, which applications that only deal with token transfers will not need to call. Thus, a wallet, for instance, will be able to integrate against the standard token contract, and be compatible with any token that does not change that token contract.

We use Mina's [custom token feature](https://github.com/MinaProtocol/MIPs/blob/main/MIPS/mip-zkapps.md#custom-tokens) to store account balances directly in the Mina ledger, as opposed to storing balances in the state of the token contract itself. That way, we do not rely on any particular off-chain storage solution. Furthermore, this helps us to allow for multiple transactions involving a particular token within the same block, which would be more difficult to achieve if balances were part of the contract state.

The standard implementation supports the common features associated with fungible tokens: minting and burning, transferring, and viewing account balances. One-to-one transfers can be initiated by calling a `@method` of the contract defining the token (the _token owner contract_). More complicated transactions can be formed by constructing multiple `AccountUpdate`s and have those be _approved_ by the token owner contract. In particular, this design allows interoperability in the sense that arbitrary third party `SmartContract`s can use fungible tokens.

## Detailed design

Fungible tokens on Mina will use the custom token feature of Mina defined in [MIP4](https://github.com/MinaProtocol/MIPs/blob/main/MIPS/mip-zkapps.md#custom-tokens). In Mina, a new class of custom tokens can be introduced by writing and deploying a `SmartContract` that defines the token. This is called the _token owner contract_. When the token owner contract is deployed, a new token id is created, and accounts with that token id will hold the custom token instead of MINA.

The token owner contract can change balances of accounts with the custom token, so it can mint, burn, and move tokens. When an `AccountUpdate` that has not been created in a method of the owner contract tries to modify the balance of an account with the custom token, it needs to be approved by the owner contract. That way, the owner contract can enforce rules that all transactions with the token must satisfy (conservation of tokens being a very common example). The [`TokenContract` class](https://github.com/o1-labs/o1js/pull/1384) defines methods for token transfer, as well as for approving a whole forest of account updates. The API that we define mirrors those methods, and the reference implementation is built using the `TokenContract`.

The API consists of the following TypeScript interfaces:

### Transferable
This interface consists of a single function, `transfer`, which sends a specified amount of tokens from one account to another.

```TypeScript

interface Transferable {
  transfer(from: PublicKey | AccountUpdate,
           to: PublicKey | AccountUpdate,
           amount: UInt64): void;
}
```

### Approvable
Transactions that are either more complicated than a transfer from one account to another (such as sending tokens from one account to many accounts) or that are constructed from third-party contracts can be done using the `Approvable` interface. The workflow is to construct the individual `AccountUpdate` values, and then have them be approved by the owner contract. The interface defines three functions to cover the cases of individual `AccountUpdate`s or `AccountUpdateTree`s, an array of either of these types, or a whole `AccountUpdateForest`.

```TypeScript
interface Approvable {
  approveAccountUpdate(accountUpdate: AccountUpdate | AccountUpdateTree): void;
  approveAccountUpdates(accountUpdates: (AccountUpdate | AccountUpdateTree)[]): void;
  approveBase(forest: AccountUpdateForest): void;
}
```

### Administrative Interfaces
For administrative actions, such as changing the total supply or minting and burning tokens, we define the `Mintable` and `Burnable` interfaces:

```TypeScript
interface Mintable {
  totalSupply: State<UInt64>;
  circulatingSupply: State<UInt64>;
  mint: (to: PublicKey, amount: UInt64) => AccountUpdate;
  setTotalSupply: (amount: UInt64) => void;
}

interface Burnable {
  burn: (from: PublicKey, amount: UInt64) => AccountUpdate;
}
```

### Viewable
For retrieving information about the token, such as the total or circulating supply, account balances, etc., we define the `Viewable` interface.

```TypeScript
interface Viewable {
  getAccountOf: (address: PublicKey) => ReturnType<typeof Account>;
  getBalanceOf: (address: PublicKey, options: ViewableOptions) => UInt64;
  getTotalSupply: (options: ViewableOptions) => UInt64;
  getCirculatingSupply: (options: ViewableOptions) => UInt64;
  getDecimals: () => UInt64;
}
```

### Reference Implementation

A reference implementation of a fungible token implementing the above interfaces can be found at https://github.com/MinaFoundation/mip-token-standard.

## Test plan and functional requirements

We test the reference implementation, using unit tests against `Mina.LocalBlockchain`. As of now, the test suite covers the main functionality: minting tokens, and transferring them, including transfers between third-party contracts.

We shall extend the test suite to also cover cases that must fail (transactions that have not been approved, transactions that implicitly mint or burn tokens, etc.). Furthermore, the functions from `Viewable` are not yet covered by the tests.

## Drawbacks

Currently, it is only possible to pass token permissions for one kind of token down an `AccountUpdate` tree. That limitation makes it somewhat cumbersome to implement solutions involving multiple custom tokens (such as a decentralized exchange). Removing that limitation in a future upgrade to `o1js` and Mina would make development around custom tokens easier.

## Prior art

This standard has evolved from a discussion and proposal on MinaResearch, see https://forums.minaprotocol.com/t/draft-fungible-token-standard-zkapps/6142

There is a lot of prior art on fungible tokens. The most prominent standard is ERC-20 on Ethereum. Cardano's native tokens choose a different point in the design space, in being very lightweight, but less customisable.

## Unresolved questions
