# Exposing ECDSA and SHA3/Keccak to SnarkyJS

## Summary

Cryptographic primitives such as ECDSA and SHA3 are widely used outside of Mina. For example, Ethereum uses ECDSA over secp256k1 for signatures - in order to "communicate" with the outside world and other blockchains, SnarkyJS (and, therefore Mina) needs to support these primitives as well. This RFC describes how we will leverage the custom gates implemented by the crypto team and expose them to SnarkyJS, making them accessible to smart contract developers.

## Motivation

The initial [Ethereum Crypto Primitive PRD](https://www.notion.so/minaprotocol/Ethereum-Primitives-Support-PRD-d89af720e1c94f7b90166709432e7bd5) describes the importance of establishing cryptographic compatibility with Ethereum. The [ECDSA PRD](https://www.notion.so/minaprotocol/ECDSA-ver-gadget-PoC-PRD-9458c38adf204d6b922deb8eed1ac193) and the
[Keccak PRD](https://www.notion.so/minaprotocol/Keccak-gadget-PoC-PRD-59b024bce9d5441c8a00a0fcc9b356ae) then go into detail and describe two of the most important building blocks to achieve Ethereum compatibility in a cryptographic sense.

This RFC describes the steps needed to expose both ECDSA and SHA3/Keccak to SnarkyJS, enabling SnarkyJS developer to use these primitives in their applications and smart contracts.

It is important to mention that this RFC only considers the work required to _expose_ already existing cryptographic primitives (gates and gadgets), which have previously been implemented by the crypto team, from the OCaml bindings layer to SnarkyJS - it is not required to implement additional cryptographic primitives. Changes will only impact [snarkyjs-bindings](https://github.com/o1-labs/snarkyjs-bindings) and [SnarkyJS](https://github.com/o1-labs/snarkyjs) itself.

Once completed, SnarkyJS users will be able to leverage ECDSA and SHA3/Keccak to build applications that integrate with Ethereum and other use cases that require the use of said cryptographic primitives.

## Detailed design

### SHA3/Keccak

The Keccak and SHA3 gadget has been implemented by the crypto team ([minaprotocol/mina PR#13196](https://github.com/MinaProtocol/mina/pull/13196)), enabling us to leverage the already existing bindings layer to and from OCaml. This design allows us to simply integrate the new gadgets into SnarkyJS by simply exposing them in `ocaml/lib/snarky_js_bindings_lib.ml`.

For Keccak/SHA3, the implementation exposes two ready-to-go functions.

```ocaml
(** Gagdet for NIST SHA-3 function for output lengths 224/256/384/512 *)
val nist_sha3 :
    (module Snarky_backendless.Snark_intf.Run with type field = 'f)
  -> int
  -> 'f Snarky_backendless.Cvar.t list
  -> 'f Snarky_backendless.Cvar.t array

(** Gadget for Keccak hash function for the parameters used in Ethereum *)
val ethereum :
    (module Snarky_backendless.Snark_intf.Run with type field = 'f)
  -> 'f Snarky_backendless.Cvar.t list
  -> 'f Snarky_backendless.Cvar.t array

```

These two functions will be imported into the bindings layer and exposed via a new sub-module as part of the already existing `Snarky` module.

```ocaml

  module Sha = struct
    let create message nist_version length =
      let message_array = Array.to_list message in
      if Js.to_bool nist_version then
        Kimchi_gadgets.Keccak.nist_sha3 (module Impl) length message_array
      else Kimchi_gadgets.Keccak.ethereum (module Impl) message_array
  end

```

In order to reduce the bindings surface, Keccak and SHA3 will be exposed via a `create` function, which behaves like a factory pattern.
By calling this function inside SnarkyJS, we can define and expose all possible variants of SHA3(224/256/384/512) and Keccak without the need to have an individual function in the bindings layer for each variant.

In order to differentiate Poseidon, which works over native Field elements, and SHA3 and Keccak which works over byte-sized Field elements, it will be beneficial to introduce a new type `UInt8` (a Field element that is exactly a byte) to draw a clean line between both hash functions.

In SnarkyJS, these new primitives will be declared as followed:

```ts
function buildSha(length: 224 | 256 | 385 | 512, nist: boolean) {
  return {
    hash(message: UInt8[]) {
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

_Note_: Currently, if `nist` is set to `false`, we only support an output length of 256.

```ts
function SHA3(length: 224 | 256 | 385 | 512 | { nist: 256 }): {
  hash(xs: UInt8[]);
};
```

Developers can then import these functions into their project via

```ts
import { Sha3_224, Sha3_256, Sha3_385, Sha3_512, Keccak } from "snarkyjs";

Sha3_224.hash(xs);
// ..
```

#### Alternative Approach

Another possibility is to combine all hash functions, including Poseidon, under a shared namespace `Hash`. Developers will then be able to use these functions by calling `Hash.[hash_name].hash(xs)`. However, this would not be equivalent to the existing `Poseidon` API.

However, the existing `Poseidon` API could be deprecated in favour of the new `Hash` namespace.

```ts
const Hash = {
  Poseidon: {
    hash: (xs: UInt8[]) => "digest",
  },

  // ..

  SHA3_224: {
    hash: (xs: UInt8[]) => "digest",
  },

  Keccak: {
    hash: (xs: UInt8[]) => "digest",
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
class UInt8 {
  // constraints an array of Field elements to be at most 1 byte per Field
  static fromFields(xs: Field[]): UInt8[];
  // conversion from a hex-string to an array of UInt8 elements
  fromHex(hex: string): UInt8[];
  // conversion of UInt8 elements to a hex-string
  static toHex(xs: UInt8[]): string;
}
```

Additionally, we will add a function

```ts
ForeignField.fromBytes(bytes: UInt8[]): ForeignField;
```

to the `ForeignField` implementation that constraints the output of Keccak to a foreign field, which will later be an important constraint for ECDSA.

Overall, exposing new gadgets and gates follow a strict pattern that has been used all over in the SnarkyJS bindings layer. As an example, the [Poseidon](https://github.com/o1-labs/snarkyjs-bindings/blob/main/ocaml/lib/snarky_js_bindings_lib.ml#L386) implementation behaves similarly. From the point of view of SnarkyJS, these gadgets are just another set of function calls to OCaml.

## ECDSA

Similar to SHA3 and Keccak, the gadget for ECDSA has been implemented by the crypto team and is available through exposing it in the bindings layer. However, designing and implementing a safe API for ECDSA is not as straight forward as it is for SHA3 and Keccak.

The main verification step of ECDSA is implemented via a `verify` gadget in OCaml.

```ocaml
let verify (type f) (module Circuit : Snark_intf.Run with type field = f)
    (base_checks : f Foreign_field.External_checks.t)
    (scalar_checks : f Foreign_field.External_checks.t)
    (curve : f Curve_params.InCircuit.t) (pubkey : f Affine.t)
    ?(use_precomputed_gen_doubles = true) ?(scalar_mul_bit_length = 0)
    ?(doubles : f Affine.t array option)
    (signature :
      f Foreign_field.Element.Standard.t * f Foreign_field.Element.Standard.t )
    (msg_hash : f Foreign_field.Element.Standard.t)
```

The function takes the following arguments:

- `base_check` := Context to track required base field external checks
- `scalar_checks` := Context to track required scalar field external checks
- `curve` := Elliptic curve parameters - for now, the goal is to make ECDSA over secp256k1 available. However, ECDSA accepts different curve parameters as well.
- `pubkey` := Public key of signer
- `doubles` := Optional powers of $2^i$ of the `pubkey`, $0 <= i < n$ where $n$ is `curve.order_bit_length`
- `signature` := ECDSA signature (r, s) s.t. r, s $\in [1, n)$
- `msg_hash` := Message hash s.t. msg_hash \in Fn - this will be the output of `Keccak`

However, it is important to mention that the OCaml API has been designed with the assumption in mind that some of the inputs already satisfy a set of preconditions. These preconditions will not be checked internally, but are a requirement. Additionally, verifying ECDSA signatures is an expensive task and requires a lot of constraints. By moving the precondition checks of the inputs outside of the actual verification step (requiring them as preconditions), it allows us to optimize constraints but also provides a bigger API surface to cover in order for us to provide a safe API developers can use.

The input preconditions are:

- `pubkey` is on the curve and not O (`Ec_group.is_on_curve` gadget)
- `pubkey` is in the subgroup (nP = O) (`Ec_group.check_subgroup` gadget)
- `pubkey` is bounds checked (`multi-range-check` gadgets)
- `r, s` $\in [1, n)$ (`signature_scalar_check` gadget)
- `msg_hash` $\in Fn$ (`bytes_to_foreign_field_element` gadget)

Each of these preconditions requires expensive checks so its important to choose wisely when and how to check these inputs. Its important to provide a safe API as well as giving experienced developers enough space to optimize their applications by avoiding double constraining of inputs.

By building on top of the [`ForeignField`](https://github.com/o1-labs/snarkyjs/pull/985) implementation, we can leverage the modular approach taken by it to implement an ECDSA API which provides developers with a powerful modular interface. This approach allows us to potentially enable multiple versions of ECDSA with different curves, within the same API. We will also implement a `ForeignCurve` API which relies on the non-native elliptic curve primitives and gadgets introduce in the original [ECDAS PR](https://github.com/MinaProtocol/mina/pull/13279) so developers can not only profit from a modular ECDSA API, but can also access non-native elliptic curves directly.

Similar to [`ForeignField`](https://github.com/o1-labs/snarkyjs/pull/985), the API of `ForeignCurve` will make use of a modular approach.

```ts
type CurveParams = {
  /**
   * Base field modulus
   */
  modulus: bigint;
  /**
   * Scalar field modulus = group order
   */
  order: bigint;
  /**
   * The `a` parameter in the curve equation y^2 = x^3 + ax + b
   */
  a: bigint;
  /**
   * The `b` parameter in the curve equation y^2 = x^3 + ax + b
   */
  b: bigint;
  /**
   * Generator point
   */
  gen: AffineBigint;
};
```

The `ForeignCurve` API will accept a set of `CurveParams`, which can then be used to create a `ForeignCurve` and serve as the foundation for ECDSA.

```ts
function createForeignCurve(curve: CurveParams) {
  class BaseField extends createForeignField(curve.modulus) {}
  class ScalarField extends createForeignField(curve.order) {}

  class ForeignCurve implements Affine {
    x: BaseField;
    y: BaseField;

    constructor() {}

    add() {}

    static BaseField = BaseField;
    static ScalarField = ScalarField;
  }

  return ForeignCurve;
}
```

TODO ECDSA API preview

## Test plan and functional requirements

### SHA3/Keccak

In order to test the implementation of SHA3 and Keccak in SnarkyJS, we will follow the testing approach we already apply to other gadgets and gates.
This includes testing the out-of- and in-snark variants using our testing framework, as well as adding a SHA3 and Keccak regression test. The regression tests will also include a set of predetermined digests to make sure that the algorithm doesn't unexpectedly change over time (similar to the tests implemented for the OCaml gadget). We will include a range of edge cases in the tests (e.g. empty input, zero, etc).

In addition to that, we should provide a dedicated integration test that handles SHA3/Keccak hashing within a smart contract (proving enabled). This will allow us to not only provide developers with an example, but also ensure that SHA3 and Keccak proofs can be generated.

### ECDSA

ECDSA testing will follow a similar approach to SHA3 and Keccak. We will utilize the regression test framework in SnarkyJS to make sure no unintended changes are introduced and backwards compatibility is guaranteed. Similar to the original ECDSA gadget tests, we will also test a set of pre-defined ECDSA signatures (e.g. Ethereum transaction signatures).
The implementation will also be tested in provable code (a smart contract) to make sure proofs can be generated correctly while also providing developers with an example of how to use ECDSA within smart contracts.

## Drawbacks

Compared to Poseidon, hashing with SHA3 and Keccak is expensive. This should be made clear to the developer to avoid inefficient circuits. Additionally, it is important to educate developers of when to use SHA3/Keccak and when to use Poseidon. Additionally, the API should be secure.

It should be mentioned that developers should ideally use Poseidon for everything that does not explicitly require SHA3/Keccak (e.g. a Merkle Tree in SnarkyJS, checksums of Field elements and provable structures `Struct`, etc.) and only use SHA3/Keccak if it is really required (e.g. interacting with Ethereum, verifying Ethereum signatures, etc.).

Especially in the case of ECDSA, verifying a signature is very expensive. It is important to provide both a safe API as well as avoiding double constraining of inputs. The developer needs to be educated that ECDSA is not the default signature scheme and should only be used in special cases and the API should reflect this difference.

Adding new primitives, especially cryptographic primitives, always includes risks such as the possibility of not constraining the algorithm and input enough to provide the developer with a safe API that is required to build secure applications. However, adding these primitives to SnarkyJS enables developers to explore a new range of important use cases.

## Rationale and alternatives

ECDSA, Keccak and SHA3 could not be exposed to SnarkyJS at all. However, this would essentially render these primitives useless since they were specifically designed to be used by developers with SnarkyJS. By adding these primitives, SnarkyJS will become an even more powerful zero-knowledge SDK that enables developers to explore a wide range of use cases. Besides that, not adding these primitives would essentially block the ecosystem from interacting with other chains, mainly Ethereum.

## Prior art

Exposing gates and gadgets from the OCaml layer to SnarkyJS is nothing new - the same procedure has been applied to other primitives such as Poseidon, Field, and Elliptic Curve operations.

## Unresolved questions

No unresolved questions.
