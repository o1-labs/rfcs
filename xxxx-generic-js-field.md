# Make native Field in SnarkyJS configurable

## Summary

Design the native field abstraction that will unblock a version of the zk-state bridge work using an implementation that aligns with SnarkyJS developer experience.

## Motivation

Abstracting over the native field unlocks efficient bridging logic to interact with Mina's blockchain proof. Bridging is critical for the Mina ecosystem: It enables increased interoperability and enhances the Mina ecosystem's value proposition.

## Detailed design

The plan is to expose a JS function, here called `instantiateWithCustomField()`, that takes as input a field modulus (and potentially other Field parameters) and returns all the methods SnarkyJS normally provides at the top level to write provable code.

```ts
// custom field modulus (e.g. bn128 base field)
const modulus = 0x30644e72e131a029b85045b68181585d97816a916871ca8d3c208c16d87cfd47n

// get circuit-writing modules
let { Field_, Bool_, Circuit, Provable } = instantiateWithCustomField(modulus);
class Field extends Field_ {}
class Bool extends Bool_ {}

// write a circuit
class Main extends Circuit {
  @circuitMain
  static main(y: Field, @public_ x: Field) {
    let y3 = y.square().mul(y);
    y3.assertEquals(x);
  }
}
```

### OCaml side: instantiating Snarky and the SnarkyJS OCaml bindings

An important step of `instantiateWithCustomField()` has to happen inside OCaml: That of instantiating snarky-ml with a custom backend.
The plan is to not use the usual bindings layer to expose individual functions, as done via [bindings.js](https://github.com/o1-labs/snarkyjs-bindings/blob/main/kimchi/js/bindings.js) for the Pasta fields, but instead have a single external function that returns an object containing all methods we need to instantiate a "snarky backend".

The plan is best described using this mock OCaml code:

```ocaml
module Js = Js_of_ocaml.Js

(* interfaces that snarky expects and returns *)
module type Field_intf = sig
  type t

  val add : t -> t -> t

  val sub : t -> t -> t

  val mul : t -> t -> t

  val zero : t

  val one : t
end

module type Backend_intf = sig
  module Field : Field_intf
end

module type Snarky_intf = sig
  module Field : Field_intf

  module Constraint : sig
    type t

    val r1cs : Field.t -> Field.t -> Field.t -> t

    val equal : Field.t -> Field.t -> t
  end

  val assert_ : Constraint.t -> unit
end

module Snarky (Backend : Backend_intf) = struct
  module Field = Backend.Field

  module Constraint = struct
    type t = unit

    let r1cs _x _y _z : t = ()

    let equal _x _y : t = ()
  end

  let assert_ (_c : Constraint.t) : unit = ()
end

(* generic native field module, reimplemented in JS *)
type bigint = Js.Unsafe.any Js.t

type field = bigint ref

type js_field_module =
  < add : (field -> field -> field) Js.readonly_prop
  ; sub : (field -> field -> field) Js.readonly_prop
  ; mul : (field -> field -> field) Js.readonly_prop
  ; zero : field Js.readonly_prop
  ; one : field Js.readonly_prop >
  Js.t

external create_js_field : field -> js_field_module = "create_field"

(* make existing snarky-bindings a functor from snarky *)
module Snarky_bindings (Impl : Snarky_intf) = struct
  module Field' = struct
    let assert_equal x y = Impl.assert_ (Impl.Constraint.equal x y)

    let assert_mul x y z = Impl.assert_ (Impl.Constraint.r1cs x y z)
  end
end

(* instantiating snarky and snarky-bindings with JS field module, and return something usable in JS *)
let create_field (modulus : field) =
  (* call generic JS field constructor *)
  let js_field = create_js_field modulus in
  (* instantiate backend *)
  let module Backend = struct
    module Field = struct
      type t = field

      let add = js_field##.add

      let sub = js_field##.sub

      let mul = js_field##.mul

      let zero = js_field##.zero

      let one = js_field##.one
    end
  end in
  (* instantiate snarky *)
  let module Snarky' = Snarky (Backend) in
  (* instantiate snarky-bindings *)
  let module Snarky_bindings' = Snarky_bindings (Snarky') in
  (* return for JS use *)
  object%js
    method assertEqual = Snarky_bindings'.Field'.assert_equal

    method assertMul = Snarky_bindings'.Field'.assert_mul
  end
```

### Rust side: instantiating core Kimchi methods

TODO - does Rust enable us to dynamically instantiate traits (or whatever) as function return values?
If not, we might have to create Kimchi bindings for a _fixed number of supported provers_ (e.g. Pasta/IPA, BN128/KZG) and in a JS wrapper dynamically pick one of them that matches the field configuration given to `instantiateWithCustomField()`.

## Test plan and functional requirements

TODO

## Drawbacks

It's work.

## Rationale and alternatives

An alternative could be to not build this to be part of the released version of SnarkyJS, but instead create a custom fork/branch where we simply change everything to be hard-coded to a different Field we need. That could be less work in the very short term, but would create huge tech debt to keep this fork up to date with mainline SnarkyJS, and also would eschew other use cases and benefits to the wider exosystem of having a configurable Field in SnarkyJS.

## Prior art

TODO

## Unresolved questions

* What is a better name for `instantiateWithCustomField()`??
