# Fungible Token Standard on Mina

## Summary

Fungible tokens are a pervasive feature that is used for many applications, such as bridges, decentralised exchanges, layer 2 solutions, and others. Establishing a standard for creating fungible tokens on Mina will reduce the efforts for builders to create their own tokens, and allow exchanges, explorers, and other third party applications to integrate fungible tokens in a uniform way.

## Motivation

Fungible tokens are a common feature on programmable blockchain systems. Establishing a standard will help developers to create their own fungible tokens and to interface their application with tokens that others have created.

Beyond standardising an API, we want to have a standard _implementation_ of fungible tokens that people should use when deploying a token. The reason for that is that in the off-chain execution model of Mina, integrating with a `SmartContract` does require having access to the code of that contract, and running it in your own environment. If each token were to use a different contract, that would put a significant burden on wallets or other applications that want to interface with arbitrary tokens.

In order to allow for some flexibility without changing the token contract itself, we separate out administrative actions like minting new tokens to a special admin contract. The token contract will call methods from the admin contract when trying to mint new tokens, and the admin contract can implement arbitrary logic to determine whether the minting should be permitted. That way, we can move all the custom code to the admin contract, which applications that only deal with token transfers will not need to call. Thus, a wallet, for instance, will be able to integrate against the standard token contract, and be compatible with any token that does not change that token contract.

We use Mina's [custom token feature](https://github.com/MinaProtocol/MIPs/blob/main/MIPS/mip-zkapps.md#custom-tokens) to store account balances directly in the Mina ledger, as opposed to storing balances in the state of the token contract itself. That way, we do not rely on any particular off-chain storage solution. Furthermore, this helps us to allow for multiple transactions involving a particular token within the same block, which would be more difficult to achieve if balances were part of the contract state.

The standard implementation supports the common features associated with fungible tokens: minting and burning, transferring, and viewing account balances. One-to-one transfers can be initiated by calling a `@method` of the contract defining the token (the _token owner contract_). More complicated transactions can be formed by constructing multiple `AccountUpdate`s and have those be _approved_ by the token owner contract. In particular, this design allows interoperability in the sense that arbitrary third party `SmartContract`s can use fungible tokens, by integrating against the one standard implementation.

## Detailed design

Fungible tokens on Mina will use the custom token feature of Mina defined in [MIP4](https://github.com/MinaProtocol/MIPs/blob/main/MIPS/mip-zkapps.md#custom-tokens). In Mina, a new class of custom tokens can be introduced by writing and deploying a `SmartContract` that defines the token. This is called the _token owner contract_. When the token owner contract is deployed, a new token id is created, and accounts with that token id will hold the custom token instead of MINA.

The token owner contract can change balances of accounts with the custom token, so it can mint, burn, and move tokens. When an `AccountUpdate` that has not been created in a method of the owner contract tries to modify an account with the custom token, it needs to be approved by the owner contract. That way, the owner contract can enforce rules that all transactions with the token must satisfy (conservation of tokens being a very common example). The [`TokenContract` class](https://github.com/o1-labs/o1js/pull/1384) defines methods for token transfer, as well as for approving a whole forest of account updates. The standard implementation extends the `TokenContract` and makes use of these features.

The fungible token contract provides an interface consisting of the following methods:

### User-Facing Functionality

#### Transfer of Tokens
```TypeScript
@method async transfer(from: PublicKey, to: PublicKey, amount: UInt64)
```
Transfers the specified `amount` from account `from` to account `to`.

Fails when the token is paused (see [Pausing and Resuming Transfers](#pausing-and-resuming-transfers)).

Emits a `TransferEvent` (see [Events](#events)).

#### Approving Account Updates
```TypeScript
@method async approveBase(updates: AccountUpdateForest)
```
Approves all the account updates in `updates`, provided the following holds:

1. Amongst all the account updates, the total balance of the token is preserved
2. The account permissions for `receive` and `access` of accounts for the token are not changed from their default values. This is to ensure that accounts can receive tokens that are minted via the reducer (see [Actions and Reducers](#actions-and-reducers)). Without this check, a user could change their permissions for a token account with pending minted tokens and effectively halt the reducer.

Fails when the token is paused (see [Pausing and Resuming Transfers](#pausing-and-resuming-transfers)).

Note that this method does _not_ emit an event. Creating an appropriate and meaningful event would require a deeper inspection of the account update forest, as well as a more general event data type.

#### Burning Tokens
```TypeScript
 @method.returns(AccountUpdate) async burn(from: PublicKey, amount: UInt64): Promise<AccountUpdate>
```

Destroys a number of tokens specified by `amount` from the token account for the public key `from`.

Dispatches an action to update the circulating supply (see [Actions and Reducers](#actions-and-reducers)).

Fails when the token is paused (see [Pausing and Resuming Transfers](#pausing-and-resuming-transfers)).

Emits a `BurnEvent` (see [Events](#events)).

#### Query Balances
```TypeScript
@method.returns(UInt64) async getBalanceOf(address: PublicKey): Promise<UInt64>
```

Returns the balance of the token account for the public key `address`.

### Privileged Administrative functions

In this section, we list methods that require some sort of special privileges, like minting new tokens. The rules for when it is permissible to mint new tokens will be different for different tokens. A simple rule could be to require a signature from one or more special keys. There could also be a total limit on the number of tokens in existence, or it could be forbidden to mint new tokens at all.

In order to allow flexibility in granting permisssions, the methods in this section will call to a token admin contract, which is set during deployment of the token contract. That admin contract can grant or deny the right to mint tokens, pause/resume transfers, or change the admin contract itself. By using a third contract, the permissions can be changed without changing the token contract itself -- which is important for integration.

#### The Admin Contract
```TypeScript
export type FungibleTokenAdminBase = SmartContract & {
  canMint(accountUpdate: AccountUpdate): Promise<Bool>
  canChangeAdmin(admin: PublicKey): Promise<Bool>
  canPause(): Promise<Bool>
  canResume(): Promise<Bool>
}
```

An admin contract needs to provide the methods defined in `FungibleTokenAdminBase`, thus implementing that interface. Each of the methods will be called by the token contract to check permissions, and will return a `Bool` value to grant or deny permission.

An example implementation that allows a priviledged key to perform any kind of administrative action is provided with the standard implementation.

#### Deploying the Contract
```TypeScript
interface FungibleTokenDeployProps extends Exclude<DeployArgs, undefined> {
  /** Address of the contract controlling permissions for administrative actions */
  admin: PublicKey
  /** The token symbol. */
  symbol: string
  /** A source code reference, which is placed within the `zkappUri` of the contract account.
   * Typically a link to a file on github. */
  src: string
  /** Number of decimals in a unit */
  decimals: UInt8
}
```

Deploys the token contract. The token admin contract is assumed to be already deployed, at the address given by `admin`. The token symbol, number of digits, and reference to the source code are to be provided.

#### Changing the Admin Contract
```TypeScript
@method async setAdmin(admin: PublicKey)
```

#### Minting Tokens
```TypeScript
@method.returns(AccountUpdate) async mint(recipient: PublicKey, amount: UInt64): Promise<AccountUpdate>
```

Creates `amount` new tokens in the token account of `recipient`.

Requires `canMint()` of the admin contract to return `Bool(true)`.

#### Pausing and Resuming Transfers
```TypeScript
@method async pause()
@method async resume()
```

After `pause()` has been successfully called, users will not be able to move or burn tokens, until `resume()` has been called successfully.

Those methods call `canPause()` and `canResume()` of the admin contract, respectively, and only succeed on `Bool(true)`.

### Events

Minting, burning, or transferring tokens (via the `transfer()` method, not via approving account updates) will emit the following events:

```TypeScript
export class MintEvent extends Struct({
  recipient: PublicKey,
  amount: UInt64,
}) {}

export class BurnEvent extends Struct({
  from: PublicKey,
  amount: UInt64,
}) {}

export class TransferEvent extends Struct({
  from: PublicKey,
  to: PublicKey,
  amount: UInt64,
}) {}
```

### Actions and Reducers

The token contract keeps track of the current circulation of tokens. As it is not currently possible to have multiple transactions in one block that modify the same part of the contract state, we use actions and reducers instead.

Minting and burning tokens will dispatch an action to modify the circulating supply. These actions can be reduced by calling

```TypeScript
@method.returns(UInt64) async getCirculating(): Promise<UInt64>
```

This method will collect the actions that were dispatched by `mint()` and `burn()`, and update the circulating supply in the contract state. It will also return the current circulating supply.

### Standard Implementation

The standard implementation can be found at https://github.com/MinaFoundation/mip-token-standard.

## Test plan and functional requirements

We test the implementation, using unit tests against `Mina.LocalBlockchain`. As of now, the test suite covers the main functionality: minting tokens, and transferring them, including transfers between third-party contracts. It also covers transactions that should fail (because of lacking authorisation, or because token number is not conserved).

An extension of the test suite is always desirable.

## Drawbacks

Currently, it is only possible to pass token permissions for one kind of token down an `AccountUpdate` tree. That limitation makes it somewhat cumbersome to implement solutions involving multiple custom tokens (such as a decentralized exchange). Removing that limitation in a future upgrade to `o1js` and Mina would make development around custom tokens easier.

## Prior art

This standard has evolved from a discussion and proposal on MinaResearch, see https://forums.minaprotocol.com/t/draft-fungible-token-standard-zkapps/6142

There is a lot of prior art on fungible tokens. The most prominent standard is ERC-20 on Ethereum. Cardano's native tokens choose a different point in the design space, in being very lightweight, but less customisable.

## Unresolved questions
