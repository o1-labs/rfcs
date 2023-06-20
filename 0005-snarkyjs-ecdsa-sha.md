# RFC title

[title]: #title

Exposing ECDSA and SHA3/Keccak to SnarkyJS

## Summary

[summary]: #summary

Foreign cryptography primitives such as ECDSA and SHA3 are widely used outside of Mina. For example, Ethereum uses ECDSA over secp256k1 for signatures - in order to "communicate" with the outside world and other blockchains, SnarkyJS (and therefor Mina) needs to support these primitives as well. This RFC describes how we will leverage the custom gates implemented by the crypto team and expose them to SnarkyJS, making them accessible to smart contact developers.

## Motivation

[motivation]: #motivation

The initial [Ethereum Crypto Primitive PRD](https://www.notion.so/minaprotocol/Ethereum-Primitives-Support-PRD-d89af720e1c94f7b90166709432e7bd5) describes the importance of establishing cryptographic compatibility with Ethereum. The [ECDSA PRD](https://www.notion.so/minaprotocol/ECDSA-ver-gadget-PoC-PRD-9458c38adf204d6b922deb8eed1ac193) and the
[Keccak PRD](https://www.notion.so/minaprotocol/Keccak-gadget-PoC-PRD-59b024bce9d5441c8a00a0fcc9b356ae) then go into detail and describe two of the most important building blocks to achieve Ethereum compatibility in a cryptographic sense.

This RFC describes the steps needed to expose both ECDSA and SHA3/Keccak to SnarkyJS, enabling SnarkyJS developer to use these primitives in their applications and smart contracts.

It is important to mention that this RFC only considers the work required to _expose_ already existing cryptographic primitives (gates and gadgets), which have previously been implemented by the crypto team, from the OCaml bindings layer to SnarkyJS - it is not required to implement additional cryptographic primitives. Changes will only impact [snarkyjs-bindings](https://github.com/o1-labs/snarkyjs-bindings) and [SnarkyJS](https://github.com/o1-labs/snarkyjs) itself.

Once completed, SnarkyJS users will be able to leverage ECDSA and SHA3/Keccak to build applications that integrate with Ethereum and other use cases that require the use of said cryptographic primitives.

## Detailed design

[detailed-design]: #detailed-design

### SHA3/Keccak

The Keccak and SHA3 gadget has been implemented by the crypto team, enabling us to leverage the already existing bindings layer to and from OCaml. This design allows us to simply integrate the new gadgets into SnarkyJS by simply exposing them in `ocaml/lib/snarky_js_bindings_lib.ml`.

For Keccak/SHA3, the implementation exposes two ready-to-go functions.

```ocaml
(** Gagdet for NIST SHA-3 function for output lengths 224/256/384/512 *)
val nist_sha3 :
    (module Snarky_backendless.Snark_intf.Run with type field = 'f)
  -> int
  -> 'f Snarky_backendless.Cvar.t list
  -> 'f Snarky_backendless.Cvar.t array

(** Gadget for Keccak hash function for the parameters used in Ethereum *)
val eth_keccak :
    (module Snarky_backendless.Snark_intf.Run with type field = 'f)
  -> 'f Snarky_backendless.Cvar.t list
  -> 'f Snarky_backendless.Cvar.t array

```

These two functions will be imported into the bindings layer and exposed via a new sub-module as part of the already existing `Snarky` module.

```ocaml

  module Sha = struct
    let create message nist length =
      let message_array = Array.to_list message in
      if Js.to_bool nist then
        Kimchi_gadgets.Keccak.nist_sha3 (module Impl) length message_array
      else Kimchi_gadgets.Keccak.eth_keccak (module Impl) message_array
  end

```

In order to reduce the bindings surface, Keccak and SHA3 will be exposed via a `create` function, which behaves like a factory pattern.
By calling this function inside SnarkyJS, we can define and expose all possible variants of SHA3(224/256/384/512) and Keccak without the need to have an individual function in the bindings layer for each variant.

In SnarkyJS, these new primitives will be declared as followed:

```ts
function buildSha(length: 224 | 256 | 385 | 512, nist: boolean) {
  return {
    hash(message: Field[]) {
      let payload = [0, ...message.map((f) => f.value)];
      return Snarky.sha.create(payload, nist, length).map(Field);
    },
  };
}

const Sha3_224 = buildSha(224, true);
const Sha3_256 = buildSha(256, true);
const Sha3_385 = buildSha(385, true);
const Sha3_512 = buildSha(512, true);
const Keccak = buildSha(256, false);
```

Another alternative to the factory pattern above could be to supply the developer with a single function that takes a range of parameters, so that the developer can choose their flavour of SHA3/Keccak on their own.

```ts
function SHA3(
  length: 224 | 256 | 385 | 512,
  nist?: boolean = true
): { hash(xs: Field[]) };
```

Developers can then import these functions into their project via

```ts
import { Sha3_224, Sha3_256, Sha3_385, Sha3_512, Keccak } from "snarkyjs";

Sha3_224.hash(xs);
// ..
```

### Alternative Approach

Another possibility is to combine all hash functions, including Poseidon, under a shared namespace `Hash`. Developers will then be able to use these functions by calling `Hash.[hash_name].hash(xs)`. However, this would not be equivalent to the existing `Poseidon` API.

However, the existing `Poseidon` API could be deprecated in favour of the new `Hash` namespace.

```ts
const Hash = {
  Poseidon: {
    hash: (xs: any) => "digest",
  },

  // ..

  SHA3_224: {
    hash: (xs: any) => "digest",
  },

  Keccak: {
    hash: (xs: any) => "digest",
  },
};
```

Developers would then be able to simply import the `Hash` namespace into their project and use all available hash functions.

```ts
import { Hash } from "snarkyjs";

Hash.Poseidon.hash(xs);
Hash.Keccak.hash(xs);
// ..

// deprecated in favor of Hash.Poseidon.hash
Poseidon.hash(xs);
```

It is important to mention that we have a lot of freedom in the the implementation details of the API, as the gates and gadgets are already implemented in OCaml and just need to exposed and wrapped by a developer friendly API.

One goal of the API is to design an easy to use interface for developers as well as maintaining consistency with already existing primitives (Poseidon in this case, Schnorr in the case of ECDSA).
The usage of both Keccak/SHA3 and Poseidon is the same: The user passes an array of Field elements into the hash function and the output is returned. One important detail is that Keccak/SHA3 only accepts an array of Field element that each are no larger than 1 byte. We will also provide additional helper functions that allow conversion between Field element arrays and hexadecimal encoded strings.

```ts
function fieldBytesFromHex(xs: Field[]): string;
function hexToFieldBytes(hex: string): Field[];
```

Overall, exposing new gadgets and gates follow a strict pattern that has been used all over in the SnarkyJS bindings layer. As an example, the [Poseidon](https://github.com/o1-labs/snarkyjs-bindings/blob/main/ocaml/lib/snarky_js_bindings_lib.ml#L386) implementation behaves similarly. From the point of view of SnarkyJS, these gadgets are just another set of function calls to OCaml.

## ECDSA

**TODO** this will be a seperate PR stacked on top of this one

**Evergreen, wide-sweeping Details**

**Ephemeral details**

## Test plan and functional requirements

[test-plan-and-functional-requirements]: #test-plan-and-functional-requirements

## SHA3/Keccak

In order to test the implementation of SHA3 and Keccak in SnarkyJS, we will follow the testing approach we already apply to other gadgets and gates.
This includes testing the out-of- and in-snark variants using our testing framework, as well as adding a SHA3 and Keccak regression test. The regression tests will also include a set of predetermined digests to make sure that the algorithm doesn't unexpectedly change over time (similar to the tests implemented for the OCaml gadget).

In addition to that, we should provide a dedicated integration test that handles SHA3/Keccak hashing within a smart contract (proving enabled). This will allow us to not only provide developers with an example, but also ensure that SHA3 and Keccak proofs can be generated.

## Drawbacks

[drawbacks]: #drawbacks

Compared to Poseidon, hashing with SHA3 and Keccak is expensive. This should be made clear to the developer to avoid inefficient circuits. Additionally, it is important to educate developers of when to use SHA3/Keccak and when to use Poseidon. Additionally, the API should be secure.
Adding new primitives, especially cryptographic primitives, always includes risks such as the possibility of not constraining the algorithm and input enough to provide the developer with a safe API that is required to build secure applications. However, adding these primitives to SnarkyJS enables developers to explore a new range of important use cases.

## Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

Keccak and SHA3 could not be exposed to SnarkyJS at all. However, this would essentially render these primitives useless since they were specifically designed to be used by developers with SnarkyJS. By adding these primitives, SnarkyJS will become an even more powerful zero-knowledge SDK that enables developers to explore a wide range of use cases.

## Prior art

[prior-art]: #prior-art

Exposing gates and gadgets from the OCaml layer to SnarkyJS is nothing new - the same procedure has been applied to other primitives such as Poseidon, Field and Elliptic Curve operations.

## Unresolved questions

[unresolved-questions]: #unresolved-questions

No unresolved questions.
