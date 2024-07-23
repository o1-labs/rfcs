# Offchain State API RFC: Name Service zkApp (DRAFT)
## Summary

The Exploration team at O1 Labs has a mandate to develop expertise in the o1js SDK and disseminate that knowledge through the user bases of our developer community and partners.  This RFC outlines an approach to build an app using the new Offchain State API, with the goals of providing a reference implementation of the feature, identifying blockers, pitfalls, and opportunities to improve developer experience, and inspiring the community to experiment on their own with offchain state.


## Motivation

Please see [RFC](https://www.notion.so/o1labs/Concurrent-State-Updates-Example-zkApp-4c557ef2724b401e83f58193d4fb2a84) for additional details.

We have identified concurrent updates to application state as a challenge that many zk app developers struggle to overcome.  The off-chain state API is a solution developed by O1 Labs to address the problem, but since its release, we have received questions about usage and have yet to see widespread adoption.  Building a sample app will give developers more confidence about the correct usage of the API, as well as develop our internal expertise about the feature, which will lead to better community support.

The app we decided to build is a namespace resolver.  This is an excellent example for a few reasons:
It is a well-known use case in the community.  The success of ENS means that most developers will have an intuitive understanding of the functional requirements for a namespace resolver and can spend their time understanding the implementation rather than the product spec.
It avoids ethical and legal gray areas as well as excessive financial risk.
Outside of the direct need for off-chain state, the app's functionality is simple, thus highlighting only the specific feature we intend to show off.


## Detailed design

Offchain State API offers `OffchainState.Map`, which can store key-value pairs and `OffchainState.Field`, which can store a Field or any other type of data that can be represented as a Field. This design utilises `OffchainState.Map` to store domain to record mapping and `OffchainState.Field` to store the premium for registering a domain.

### Data Types

```ts
class NameRecord extends Struct({
    owner: PublicKey,
    data: ...
}) {}

class Name extends PackedString() {}
```

### Events

### Reducer

### Smart Contract
##### Off-chain State
```ts 
names: OffchainState.Map(Name, NameRecord)
premium: OffchainState.Field(UInt64)
```
#### User Facing Methods
##### Register name
```ts
/** 
 * Sender transfers {this.premium} to the contract and sets themselves as the owner of the name in the state
 * Fails if
 *   - sender cannot afford premium
 *   - name is already owned
 *   - name does not meet criteria
 * 
 * @emits NameRegistrationEvent
 */
@method async register_name(name: Name)
```
##### Set record
```ts
/**
 * Sets the domain record for a given name
 * Fails if
 *   - the name is not owned by sender
 * 
 * @emits RecordSetEvent
 */
@method async set_record(name: Name, record: NameRecord)
```
##### Transfer ownership
```ts
/**
 * Transfer ownerhsip of a name record to a new PublicKey
 * Fails if
 *   - the name is not owned by sender
 * 
 * @emits NameRegistrationTransferEvent
 */
@method async transfer_name_ownership(name: Name, new_owner: PublicKey)
```
##### Owner of
```ts
@method.returns(PublicKey) async owner_of(name: Name): PublicKey
```
##### Resolve
```ts
@method.returns(NameRecord) async resolve_name(name: Name): NameRecord
```
##### Premium rate
```ts
@method.returns(UInt64) async premium_rate(): UInt64
```
##### Get admin
```ts
@method.returns(PublicKey) async get_admin(): PublicKey
```
##### Is Paused
```ts
@method.returns(Bool) async is_paused(): Bool
```


#### Admin Functions

##### Set premium rate
```ts
/**
 * 
 * 
 * @emits PremiumChangedEvent
 */
@method async set_premium(new_premimum: UInt64)
```
##### Toggle Pause 
```ts
/**
 * 
 * 
 * @emits PauseToggleEvent
 */
@method async toggle_pause()
```
##### Change admin
```ts
/**
 * 
 * 
 * @emits AdminChangeEvent
 */
@method async change_admin(new_admin: PublicKey)
```

### Settletment Engine

Offchain State API uses actions under the hood so the state is lagging like as in a default action-reducer scheme. An agent needs to create settlement proofs and send the proofs to blockchain regularly.The settle() method is not permissioned therefore any other actor can settle the latest state updates.

[Coby] What is a settlement proof?  Can you expand on the type of this?  IMO a visual aid is often useful in a document like this to help understand the flow.

[Coby] Can you add technical detail about where the data will actually be stored for the off chain state?  Will a single computer store it?  Or will we be able to use some decentralized layer like Celestia?


### User Interface

- Register Domain Page
- My Domains Page


## Test plan and functional requirements

1. Unit Testing
2. Deploying and testing the zkapp on LocalBlockchain
3. Deploying and testing the zkapp on Lightnet

[Coby] Can we list out specific functional and non-functional requirements?

[Coby] My suggestion would be for functional requirements: A user can register a domain to resolve to their Mina address if that domain is not already registered.  A user can resolve the owner of any domain and see either the owner, or that it is available.  A user can see all domains owned by a given Mina address.  When a user registers a domain, they pay a premium of Mina to the admin.

[Coby] My suggestion would be for NFRs: Total system throughput is at least 100 actions per block.  The system can handle unlimited action inputs and guarantee eventual consistency of the state.  The system can support storage of every name possible up to 31 characters (1-Field packed string length, or set some other arbitrary length).  The system can lookup a domain owner in constant time (without verifying state).  State can be verified in logarithmic time.  Actions can be processed in constant time.

## Drawbacks

no drawbacks

## Rationale and alternatives

Various other zkapps would utilize Offchain State API. Domain system zkapp is a good one since it doesn't have any dependencies. The domain system zkapp utilises offchain mapping and field. Thus, shows off the all existing Offchain State API functionality that currently exists.


## Prior art

There is no zkapp exist to show off the new Offchain State API. The [existing unit test](https://github.com/o1-labs/o1js/blob/main/src/lib/mina/actions/offchain-contract.unit-test.ts) demonstrates how to use the new OffchainState API. 




## Unresolved questions

